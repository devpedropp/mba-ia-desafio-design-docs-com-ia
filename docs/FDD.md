### FDD: Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-13
Responsável: Larissa (Tech Lead)

---

### 1. Contexto e motivação técnica

O problema técnico é que uma chamada HTTP síncrona dentro da transação de mudança de status de pedido travaria mudanças de status de outros pedidos caso o cliente estivesse lento, e não haveria como decidir se a transação deveria sofrer rollback em caso de indisponibilidade do cliente externo. A transação de mudança de status hoje já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos do pedido.

A feature se encaixa no projeto existente como um novo módulo, seguindo o mesmo padrão de módulo já usado na base de código: cada domínio é organizado em `src/modules` com controller, service, repository, routes e schemas. O worker de processamento roda como uma entry point separada (`src/worker.ts`), com a lógica de processamento em um arquivo dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`), e usa o mesmo banco (mesma `DATABASE_URL`) através de uma instância própria de `PrismaClient`, por rodar em outro processo Node.

Atores
- `OrderService`, no método `changeStatus`: produtor do evento, dentro da transação de mudança de status
- Worker de webhooks, processo separado: consome a outbox e realiza as chamadas HTTP externas
- Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo): destinatários das notificações
- Usuário com papel ADMIN: responsável pelo reprocessamento manual de eventos em dead letter, com log de auditoria de quem executou

Limites de escopo
- Comunicação é exclusivamente outbound: o sistema envia webhooks para os clientes, os clientes não enviam para o sistema
- Esta entrega é single-worker; a garantia de ordering por pedido só vale enquanto houver um único worker rodando

---

### 2. Objetivos técnicos

- Garantir atomicidade entre a mudança de status do pedido e o registro do evento: se a transação principal commitou, o evento foi registrado; se ela sofreu rollback, o evento some junto (invariante transacional do padrão outbox)
- Entregar o evento ao cliente em até 10 segundos, considerando que o worker faz polling da outbox a cada 2 segundos (meta de latência aceita como "tempo real" pelos clientes)
- Garantir entrega at-least-once, com deduplicação do lado do cliente viabilizada por um `X-Event-Id` único gerado quando o evento entra na outbox
- Timeout de 10 segundos por tentativa de chamada HTTP do worker ao cliente
- Preservar ordering de eventos por pedido enquanto houver um único worker processando a outbox em ordem de `created_at`; essa garantia deixa de existir se a arquitetura evoluir para múltiplos workers em paralelo

---

### 3. Escopo e exclusões

**Incluído**
- Cadastro de webhook (endpoint POST): url, secret gerada pelo sistema e devolvida na criação, lista de status que o cliente quer receber, customer_id (no body ou no path, não vem do JWT)
- Edição (PATCH), remoção (DELETE) e listagem dos webhooks de um customer (GET)
- Filtro de eventos por webhook: lista de status que cada endpoint quer ouvir
- Histórico de entregas: `GET /webhooks/:id/deliveries`
- Endpoint administrativo de replay manual de dead letter: `POST /admin/webhooks/dead-letter/:id/replay`, restrito a papel ADMIN
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação de `changeStatus` do `OrderService`, recebendo o `tx` da transação atual
- Worker dedicado, processo separado da API, com polling de 2 segundos
- Tabela `webhook_outbox`, com índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`
- Tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp
- Assinatura HMAC-SHA256 do payload, secret única por endpoint de webhook, suporte a rotação de secret com grace period de 24 horas

**Excluído**
- Recebimento de webhooks de terceiros (a integração é somente outbound)
- Notificação por email quando um webhook falha repetidamente
- Rate limiting de envio para o cliente
- Dashboard visual para o cliente acompanhar seus webhooks
- Garantia de ordering global de eventos entre múltiplos workers concorrentes
- Garantia de entrega exactly-once
- Arquivamento de eventos já entregues na outbox após 30 dias

---

### 4. Fluxos detalhados e diagramas

**Fluxo principal**
- Mudança de status do pedido é feita dentro do service de orders, no método `changeStatus`
- A transação atual já atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`
- Dentro da mesma transação SQL, é chamada a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`
- A função insere uma linha em `webhook_outbox` para cada webhook ativo do customer que está inscrito no novo status
- Se nenhum webhook do customer quiser aquele status, nada é inserido (filtro aplicado na inserção, não no envio)
- Se a transação principal commitar, o evento foi registrado; se ela sofrer rollback, o evento some junto
- O worker, em processo separado, roda em loop de polling a cada 2 segundos, buscando os eventos pendentes mais antigos em lote pequeno, processando, e marcando como entregue
- Para cada evento, o worker envia a chamada HTTP para a `url` cadastrada, assinada com HMAC-SHA256, com timeout de 10 segundos
- Cliente lento que não responde em 10 segundos é tratado como falha e marcado para retry

**Fluxos alternativos e exceções**
- Retry: em caso de falha, o worker tenta novamente com backoff exponencial (1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas); após 5 tentativas, considera falha permanente
- Dead letter: o evento que esgota as 5 tentativas é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp
- Replay manual: um usuário ADMIN chama o endpoint de replay; o sistema recoloca o evento na outbox como pendente; o endpoint loga quem fez o replay, para auditoria
- Rotação de secret: o cliente pede uma nova secret pela API; a secret antiga fica válida por 24 horas em paralelo, para o cliente migrar os sistemas dele; depois disso, a antiga morre

**Diagramas (opcional)**

Estados do campo de status da `webhook_outbox`, conforme definidos em reunião: pendente, processando, falhou, entregue. Um evento que falha em todas as 5 tentativas de retry deixa de ser tratado pela outbox e passa a existir apenas como registro na tabela `webhook_dead_letter`; um replay manual recoloca esse evento como pendente na outbox novamente.

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

Os status codes indicados abaixo não foram citados literalmente na reunião (a call definiu apenas o comportamento qualitativo, ex: "erro de validação", "acesso negado"). Eles seguem a convenção REST já estabelecida no código existente (`src/shared/errors/http-errors.ts`: `BadRequestError`/`ValidationError` → 400, `UnauthorizedError` → 401, `ForbiddenError` → 403, `NotFoundError` → 404, `ConflictError` → 409), reaproveitada aqui por consistência com o restante da API.

**Cadastro de webhook**
- Tipo: http_endpoint
- Assinatura/Rota: endpoint POST para cadastro de webhook (rota exata não foi definida em reunião)
- Método: POST
- Semântica de status/headers:
  - `201 Created` — resposta de sucesso inclui a secret gerada pelo sistema, devolvida apenas nesta criação
  - `400 Bad Request` (`WEBHOOK_INVALID_URL`) — URL cadastrada sem HTTPS é recusada com erro de validação
  - `401 Unauthorized` — token ausente ou inválido, seguindo o padrão já usado pelos demais endpoints autenticados

**Campos discutidos em reunião**

Requisição
- `url`: endereço do webhook do cliente, deve ser HTTPS
- `statuses`: lista de status de pedido que o webhook quer receber
- `customerId`: informado no body ou no path (não vem do JWT do usuário autenticado)

Resposta
- `secret`: gerada pelo sistema e devolvida somente na criação
- `active`: estado ativo do cadastro (campo confirmado como parte da tabela de configuração, junto de `url`, `secret` e `customerId`)

---

**Edição, remoção e listagem de webhooks**
- Tipo: http_endpoint
- Assinatura/Rota: PATCH para editar, DELETE para remover, GET para listar os webhooks de um customer (rotas exatas não foram detalhadas em reunião além do verbo e do recurso)
- Método: PATCH / DELETE / GET
- Semântica de status:
  - `200 OK` — PATCH altera a configuração existente e retorna o webhook atualizado; GET lista os webhooks cadastrados de um customer
  - `204 No Content` — DELETE remove o cadastro
  - `404 Not Found` (`WEBHOOK_NOT_FOUND`) — webhook inexistente (PATCH/DELETE)

---

**Rotação de secret**
- Tipo: http_endpoint
- Assinatura/Rota: endpoint dedicado, acessível pela API, para o cliente pedir uma nova secret (rota exata não foi definida em reunião)
- Método: POST
- Semântica de status:
  - `200 OK` — gera uma nova secret; a secret anterior permanece válida por 24 horas em paralelo, depois deixa de ser aceita
  - `404 Not Found` (`WEBHOOK_NOT_FOUND`) — webhook inexistente

---

**Histórico de entregas**
- Tipo: http_endpoint
- Assinatura/Rota: `GET /webhooks/:id/deliveries`
- Método: GET
- Semântica de status:
  - `200 OK` — retorna os últimos 100 webhooks enviados para aquele cadastro, com sucesso ou falha, payload, resposta recebida e tempo de resposta
  - `404 Not Found` (`WEBHOOK_NOT_FOUND`) — webhook inexistente

---

**Replay manual de dead letter**
- Tipo: http_endpoint
- Assinatura/Rota: `POST /admin/webhooks/dead-letter/:id/replay`
- Método: POST
- Semântica de status/headers:
  - `200 OK` — recoloca o evento na outbox como pendente; a ação é logada, registrando quem executou o replay, para auditoria
  - `403 Forbidden` — requer papel ADMIN; usuário autenticado sem esse papel não pode acessar
  - `404 Not Found` (`WEBHOOK_NOT_FOUND`) — registro de dead letter inexistente

---

**Função interna de publicação de evento**
- Tipo: function
- Assinatura/Rota: `publishWebhookEvent(tx, order, fromStatus, toStatus)`
- Método: n/a (função pura chamada dentro da transação de `changeStatus`, recebendo o `tx` da transação atual, sem precisar injetar o repository inteiro)
- Semântica: insere o evento na outbox para os webhooks do customer inscritos em `toStatus`, dentro da mesma transação SQL da mudança de status

---

**Chamada de entrega do webhook (worker → cliente)**
- Tipo: http_endpoint (chamada outbound feita pelo worker)
- Assinatura/Rota: `POST` para a `url` cadastrada pelo cliente
- Método: POST
- Semântica de headers:
  - `X-Event-Id` — UUID gerado quando o evento entra na outbox, único por evento, usado pelo cliente para deduplicar
  - `X-Signature` — HMAC-SHA256 do payload, calculado com a secret do endpoint
  - `X-Timestamp` — timestamp do envio, para o cliente conseguir detectar replay attack se quiser
  - `X-Webhook-Id` — id do cadastro de webhook, para o cliente que tem vários conseguir saber qual cadastro caiu naquele envio
  - `Content-Type: application/json`
- Limites:
  - Payload: máximo de 64KB; se o evento ultrapassar esse tamanho, não é enviado e gera erro
  - Timeout: 10 segundos por tentativa

**Exemplo de requisição (corpo enviado ao cliente)**
```json
{
  "event_id": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-13T12:00:03.000Z",
  "order_id": "7c8d9e0f-1a2b-4c3d-8e4f-5a6b7c8d9e0f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "b2f0a6b0-1e0a-4a3e-9c1a-2f9a1a2b3c4d",
  "total_cents": 15990
}
```
O payload não inclui os items do pedido; se o cliente quiser detalhes, ele consulta `GET /orders/:id` depois.

---

### 6. Erros, exceções e fallback

Matriz de erros previstos e tratamentos

| Condição | Tratamento | Notas |
| --- | --- | --- |
| URL cadastrada sem HTTPS | Recusado com erro de validação, código `WEBHOOK_INVALID_URL` | Validação no schema Zod, não é decisão arquitetural |
| Payload do evento maior que 64KB | Evento não é enviado, gera erro | Se chegou nesse tamanho, "tem algo errado" |
| Timeout de 10s na chamada HTTP outbound | Tratado como falha, segue para retry | |
| Erro de rede ou resposta de falha do cliente | Tratado como falha, segue para retry | |
| 5ª tentativa de retry falha | Evento movido para `webhook_dead_letter`, com payload, motivo e timestamp | |
| Replay de dead letter por usuário sem papel ADMIN | Acesso negado | Mexer na fila de entrega de notificação não é ação de operador comum |
| Falha ao inserir evento na outbox dentro da transação de `changeStatus` | Toda a transação (mudança de status incluída) sofre rollback | Garantia essencial do padrão outbox; se ficasse fora da transação, perderia a garantia toda |

Códigos de erro do módulo, seguindo o padrão já existente de `AppError` com prefixo `WEBHOOK_`: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, entre outros.

Estratégias de resiliência
- Timeout: 10 segundos por tentativa de chamada HTTP outbound
- Retries: backoff exponencial, 5 tentativas (1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas), cobrindo uma janela de até aproximadamente 15 horas

Política de fallback
- Não há envio por email quando o webhook falha definitivamente (ficou definido como fora de escopo desta fase)
- O fallback é o registro em `webhook_dead_letter`, com reprocessamento manual via endpoint administrativo

Invariantes
- Se a transação de mudança de status commitar, o evento correspondente foi registrado na outbox; se ela sofrer rollback, o evento some junto
- Todo evento tem um `X-Event-Id` (UUID) único, gerado quando o evento entra na outbox
- O payload do evento é renderizado no momento da inserção na outbox (snapshot); se o pedido mudar depois, o evento já enviado continua refletindo o estado de quando o status mudou
- A entrega é garantida at-least-once; o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo `X-Event-Id`

---

### 7. Observabilidade

**Métricas**
- Não foram discutidas métricas específicas na reunião.

**Logs**
- Logs estruturados via Pino, já usado em todo o projeto; nenhuma ferramenta nova de logging é introduzida
- O endpoint de replay de dead letter loga quem executou a ação, para fins de auditoria

**Tracing**
- Não foi discutido na reunião.

**Dashboards e alertas**
- Não foram discutidos na reunião.

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| MySQL | Não especificada em reunião | Outbox roda no mesmo banco MySQL já usado pelo projeto; alternativa como Redis Streams foi descartada para não subir infraestrutura nova |
| Prisma / PrismaClient | Não especificada em reunião | Worker usa uma instância própria de `PrismaClient`, mesma `DATABASE_URL`, mesmo banco, por rodar em processo Node separado |
| Zod | Não especificada em reunião | Usado para validação de schema, por exemplo a exigência de URL HTTPS no cadastro do webhook |
| Pino | Não especificada em reunião | Logger já usado em todo o projeto, reaproveitado sem alterações |

**Garantias de compatibilidade**
- Erros do módulo seguem o padrão `AppError` já existente, com códigos prefixados por `WEBHOOK_`; o middleware de erro centralizado já trata `AppError`, Zod e Prisma, e não precisa ser alterado

---

### 9. Integração com o sistema existente

Esta seção mapeia os pontos concretos de integração no código-fonte já existente. Diferente das demais seções deste documento, o conteúdo abaixo vem da leitura do código-base (`Fonte = CODIGO` no Tracker), não da transcrição da reunião — com exceção do primeiro item, que é dual (mencionado na reunião e confirmado no código).

- **`src/modules/orders/order.service.ts`** (método `changeStatus`) — ponto de integração central. A chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` precisa ser inserida dentro do bloco `this.prisma.$transaction(async (tx) => {...})` já existente nesse método, depois do `tx.orderStatusHistory.create(...)` e antes do `return refreshed!`, reaproveitando o mesmo `tx` (`Prisma.TransactionClient`) que a transação de mudança de status já usa. Esse é o único ponto do service de orders que precisa ser alterado.
- **`src/shared/errors/app-error.ts`** e **`src/shared/errors/http-errors.ts`** — as novas classes de erro do módulo de webhooks (ex: `WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookSecretRequiredError`) devem estender as classes já existentes (`AppError`, `NotFoundError`, `BadRequestError`, `ConflictError`), seguindo o mesmo padrão de construtor `(message, statusCode, errorCode, details?)` já usado por `InsufficientStockError` e `InvalidStatusTransitionError` no módulo de orders, apenas trocando os códigos de erro para o prefixo `WEBHOOK_`.
- **`src/middlewares/error.middleware.ts`** — o middleware de erro centralizado já trata qualquer instância de `AppError`, formatando a resposta como `{ error: { code, message, details } }`. Como as novas classes de erro do módulo de webhooks estendem `AppError`, esse middleware não precisa de nenhuma alteração.
- **`src/middlewares/auth.middleware.ts`** (função `requireRole`) — o endpoint de replay de dead letter deve usar `requireRole('ADMIN')`, o mesmo middleware de autorização por papel já usado em outras rotas administrativas do projeto, em vez de implementar uma verificação de papel própria.
- **`src/shared/logger/index.ts`** — tanto o módulo de webhooks quanto o novo processo worker (`src/worker.ts`) devem importar o logger Pino já configurado aqui (`import { logger } from '../shared/logger/index.js'`), em vez de instanciar um novo logger, para manter a configuração de redação de campos sensíveis já existente.
- **`src/shared/http/response.ts`** (função `paginated`) — a listagem de webhooks (`GET /webhooks`) e o histórico de entregas (`GET /webhooks/:id/deliveries`) devem reaproveitar o mesmo helper de paginação já usado por `GET /orders`, para manter o formato de resposta (`data` + `pagination`) consistente com o resto da API.
- **`src/routes/index.ts`** e **`src/app.ts`** (função `buildApiRouter` / `buildControllers`) — o novo router de webhooks precisa ser registrado nesses dois arquivos seguindo exatamente o mesmo padrão de injeção de dependências (`repository` → `service` → `controller` → `router`) já usado para `orders`, `products` e `customers`.

---

### 10. Critérios de aceite técnicos

- Ao mudar o status de um pedido para um valor inscrito por um webhook ativo do customer, uma linha correspondente é inserida em `webhook_outbox` dentro da mesma transação de `changeStatus`
- Se a transação de `changeStatus` sofrer rollback, nenhuma linha correspondente permanece na outbox
- Em um cenário sem falhas do cliente, o evento é entregue em até 10 segundos
- Toda chamada de entrega contém os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`
- A assinatura em `X-Signature` é validável com HMAC-SHA256 usando a secret do endpoint
- O payload enviado ao cliente não contém a lista de itens do pedido
- Após 5 tentativas de retry falhas, respeitando os intervalos de backoff (1m/5m/30m/2h/12h), o evento é movido para `webhook_dead_letter` com payload, motivo da falha e timestamp
- O endpoint de replay de dead letter exige papel ADMIN e loga quem executou a ação
- Cadastro de webhook com URL que não usa HTTPS é recusado com erro de validação
- Evento com payload superior a 64KB não é enviado ao cliente
- Após rotação de secret, a secret anterior continua válida por 24 horas e deixa de ser aceita depois disso
- O histórico de entregas retorna os últimos webhooks enviados, com status de sucesso ou falha, payload, resposta recebida e tempo de resposta

---

### 11. Riscos e mitigação

### Cliente fica indisponível por período prolongado e esgota todas as tentativas de retry

- **Probabilidade:** média
- **Impacto:** cliente deixa de receber notificações pelo canal de webhook durante a indisponibilidade e após o esgotamento das tentativas; já houve caso de cliente com indisponibilidade de duas horas em manutenção planejada
- **Mitigação:**
    - Backoff exponencial com 5 tentativas, cobrindo até aproximadamente 15 horas
    - Persistência do evento em `webhook_dead_letter` com payload e motivo da falha
    - Endpoint administrativo de reprocessamento manual
- **Plano de contingência:** operador ADMIN reprocessa o evento manualmente via replay do dead letter

### Perda de ordering ao escalar para múltiplos workers no futuro

- **Probabilidade:** baixa (arquitetura atual é single-worker)
- **Impacto:** cliente pode receber eventos de um mesmo pedido fora da ordem em que ocorreram, caso a arquitetura evolua para múltiplos workers em paralelo
- **Mitigação:**
    - Manter operação com um único worker enquanto a garantia de ordering for necessária
    - Documentar a limitação: ordering garantida apenas por pedido e apenas em modo single-worker
- **Plano de contingência:** se for necessário escalar, particionar por pedido ou usar lock pessimista; tratado como problema do futuro, não desta entrega

### Vazamento de secret de um endpoint de webhook

- **Probabilidade:** média (já houve um incidente anterior de vazamento de secret em log de aplicação de um cliente)
- **Impacto:** se vazar a secret de um endpoint, compromete apenas aquele endpoint, não a plataforma inteira
- **Mitigação:**
    - Secret única por endpoint de webhook, nunca uma secret global da plataforma
    - Suporte a rotação de secret via API
    - Grace period de 24 horas para o cliente migrar sem interrupção
- **Plano de contingência:** cliente solicita rotação imediata da secret comprometida; a secret antiga é invalidada ao fim do grace period

### Entrega duplicada de eventos por conta da garantia at-least-once

- **Probabilidade:** média
- **Impacto:** cliente pode processar o mesmo evento mais de uma vez caso não implemente deduplicação do lado dele; isso joga a responsabilidade de dedup para o cliente
- **Mitigação:**
    - Envio de `X-Event-Id` único (UUID) por evento para o cliente deduplicar
    - Documentação clara sobre a necessidade de deduplicação, a ser publicada no portal do desenvolvedor
- **Plano de contingência:** nenhuma ação adicional do lado do provedor; a responsabilidade de deduplicar é do cliente, seguindo o mesmo padrão adotado por provedores como Stripe e GitHub
