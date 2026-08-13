### PRD: Order Management API Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-13
Responsável: Marcos (Product Manager)

---

### Resumo

Sistema de webhooks outbound que notifica clientes B2B integrados via API em tempo real (abaixo de 10 segundos) sempre que o status de um pedido muda, eliminando a necessidade de polling constante em `GET /orders`. A entrega é assíncrona via padrão outbox no MySQL, processada por um worker dedicado, com garantias de atomicidade, retry com backoff exponencial, dead letter queue e assinatura HMAC-SHA256 dos eventos.

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados via API (Atlas Comercial, MaxDistribuição, Nova Cargo)
- Usuários internos com papel ADMIN (operação de reprocessamento de falhas)

Cenários de uso chave
- Cliente cadastra um endpoint de webhook e passa a ser notificado automaticamente quando o status de um pedido dele muda
- Cliente consulta o histórico de entregas de webhooks para diagnosticar problemas de integração
- Operador ADMIN reprocessa manualmente eventos que falharam definitivamente (dead letter)

Onde essa feature será implantada
- Sistema existente (Order Management API). A feature adiciona um novo módulo `src/modules/webhooks` seguindo o padrão de módulos já usado no projeto, além de um novo processo worker (`src/worker.ts`) que roda separado da API.

Problemas priorizados
- Clientes B2B fazem polling recorrente em `GET /orders` para detectar mudança de status, tornando a integração lenta e cara para eles. Prioridade alta.
- Risco concreto de churn: a Atlas Comercial sinalizou que pode migrar para um concorrente caso a feature não seja entregue até o fim do trimestre. Prioridade alta.

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Notificar clientes B2B em tempo real sobre mudança de status de pedido | Latência entre a mudança de status e a entrega do webhook       | Abaixo de 10 segundos     |
| Entregar a feature a tempo de reter os clientes B2B estratégicos       | Prazo de entrega                                                | Fim de novembro (3 sprints, incluindo revisão de segurança) |

---

### Escopo

Incluso
- CRUD de configuração de webhook por customer (criar, editar, remover, listar)
- Filtro de eventos por status do pedido em cada webhook cadastrado
- Histórico de entregas de webhooks por endpoint (`GET /webhooks/:id/deliveries`)
- Endpoint administrativo de reprocessamento manual de dead letter
- Publicação de evento na outbox dentro da mesma transação de mudança de status do pedido
- Worker dedicado, em processo separado, com polling da outbox
- Retry com backoff exponencial e dead letter queue para falhas permanentes
- Assinatura HMAC-SHA256 dos payloads e rotação de secret por endpoint
- Payload e headers padronizados (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`)

Fora de escopo
- Notificação por email quando um webhook falha repetidamente (avaliado para fase futura)
- Rate limiting de envio para o cliente (ponto em aberto, será observado após o lançamento)
- Dashboard visual para o cliente acompanhar seus webhooks (projeto separado do time de frontend)
- Garantia de ordering global de eventos entre múltiplos workers (limitação conhecida; ordering só é garantida por `order_id` e enquanto houver um único worker)
- Garantia de entrega exactly-once (a entrega é at-least-once, com deduplicação sob responsabilidade do cliente)
- Arquivamento ou purga de eventos já entregues na outbox (tratado como assunto futuro, fora desta feature)

---

### Requisitos funcionais

#### FR-001 Cadastro de webhook
Cliente cadastra um endpoint de webhook para um customer via API autenticada.

**Fluxo principal**
- Cliente envia `POST /webhooks` autenticado com JWT, informando `customer_id`, `url` e a lista de status de pedido que deseja receber
- Sistema gera a secret do webhook e a devolve na resposta da criação
- Sistema persiste o cadastro com estado ativo

**Fluxos alternativos e exceções**
- Endpoint pode ser cadastrado por qualquer usuário autenticado (não exige papel ADMIN)

**Erros previstos**
- URL informada não usa HTTPS
- Dados obrigatórios ausentes ou inválidos (customer_id, url, status)

**Prioridade:** alta

---

#### FR-002 Edição de webhook
Cliente atualiza a configuração de um webhook já cadastrado.

**Fluxo principal**
- Cliente envia `PATCH /webhooks/:id` com os campos a alterar (url, lista de status, estado ativo)
- Sistema valida e persiste as alterações

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Webhook não encontrado
- Nova URL informada não usa HTTPS

**Prioridade:** alta

---

#### FR-003 Remoção de webhook
Cliente remove um webhook cadastrado.

**Fluxo principal**
- Cliente envia `DELETE /webhooks/:id`
- Sistema remove o cadastro

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Webhook não encontrado

**Prioridade:** media

---

#### FR-004 Listagem de webhooks de um customer
Cliente consulta os webhooks cadastrados para um customer.

**Fluxo principal**
- Cliente envia `GET /webhooks` informando o customer
- Sistema retorna a lista de webhooks cadastrados, sem expor a secret completa

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Nenhum erro específico além de autenticação inválida

**Prioridade:** media

---

#### FR-005 Publicação de evento de mudança de status na outbox
Quando o status de um pedido muda, o sistema registra um evento na outbox para cada webhook interessado naquele status, dentro da mesma transação da mudança de status.

**Fluxo principal**
- `OrderService.changeStatus` executa a transação que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`
- Dentro da mesma transação, o sistema chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`
- A função verifica quais webhooks ativos do customer estão inscritos no novo status e insere uma linha na `webhook_outbox` para cada um, com o payload já renderizado (snapshot)
- Se nenhum webhook do customer estiver inscrito naquele status, nenhuma linha é inserida

**Fluxos alternativos e exceções**
- Se a transação de mudança de status sofrer rollback, o evento inserido na outbox também é revertido

**Erros previstos**
- Falha ao inserir na outbox deve provocar rollback de toda a transação de mudança de status

**Prioridade:** alta

---

#### FR-006 Worker de processamento da outbox
Processo separado que lê eventos pendentes da outbox e envia as chamadas HTTP assinadas para os endpoints dos clientes.

**Fluxo principal**
- Worker roda em loop de polling a cada 2 segundos
- Busca os eventos pendentes mais antigos em lote pequeno
- Para cada evento, monta o payload JSON e assina com HMAC-SHA256 usando a secret do endpoint
- Envia requisição HTTP POST com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`
- Marca o evento como entregue em caso de sucesso

**Fluxos alternativos e exceções**
- Chamada que não responde em até 10 segundos é tratada como falha e segue para o fluxo de retry
- Worker roda como processo Node.js separado da API (entry point próprio), usando uma instância própria de `PrismaClient` conectada ao mesmo banco

**Erros previstos**
- Timeout de resposta do cliente
- Erro de rede ou status HTTP de falha retornado pelo cliente

**Prioridade:** alta

---

#### FR-007 Retry com backoff exponencial e dead letter
Eventos que falham no envio são reenviados com backoff crescente até um limite de tentativas, após o qual são movidos para a dead letter queue.

**Fluxo principal**
- Em caso de falha, o worker agenda nova tentativa com backoff exponencial: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas
- Após 5 tentativas falhas, o evento é considerado falha permanente
- O evento é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Falha persistente de entrega após esgotar as 5 tentativas

**Prioridade:** alta

---

#### FR-008 Reprocessamento manual de dead letter
Usuário ADMIN reprocessa manualmente um evento que caiu em dead letter.

**Fluxo principal**
- ADMIN envia `POST /admin/webhooks/dead-letter/:id/replay`
- Sistema valida que o usuário autenticado tem papel ADMIN
- Sistema recoloca o evento na outbox como pendente
- Sistema registra log de auditoria informando quem executou o replay

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Usuário sem papel ADMIN tentando acessar o endpoint
- Registro de dead letter não encontrado

**Prioridade:** media

---

#### FR-009 Histórico de entregas de webhook
Cliente consulta o histórico de tentativas de entrega de um webhook.

**Fluxo principal**
- Cliente envia `GET /webhooks/:id/deliveries`
- Sistema retorna os últimos registros de entrega, incluindo sucesso ou falha, payload enviado, resposta recebida e tempo de resposta

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Webhook não encontrado

**Prioridade:** media

---

#### FR-010 Rotação de secret do webhook
Cliente solicita a geração de uma nova secret para um webhook cadastrado, sem interromper a validação de eventos já em trânsito.

**Fluxo principal**
- Cliente solicita rotação de secret via API
- Sistema gera nova secret e passa a assinar novos eventos com ela
- Secret antiga permanece válida em paralelo por 24 horas
- Após 24 horas, a secret antiga deixa de ser aceita

**Fluxos alternativos e exceções**
- Nenhuma variação adicional registrada na reunião

**Erros previstos**
- Nenhum erro específico além de autenticação inválida

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência de entrega do webhook abaixo de 10 segundos, considerando o polling do worker a cada 2 segundos
- Timeout de 10 segundos por tentativa de chamada HTTP do worker ao endpoint do cliente
- [Hipótese] p95 menor que 150 ms para os endpoints síncronos de CRUD de webhook, aplicando o padrão de performance já usado no projeto. Não foi discutido explicitamente na reunião.

Disponibilidade
- [Hipótese] 99.9% de uptime mensal para os endpoints de API voltados aos clientes externos. Não foi discutido explicitamente na reunião; aplicado como padrão para sistemas voltados a cliente externo.

Segurança e autorização
- Cada requisição de webhook é assinada com HMAC-SHA256 sobre o corpo da requisição, enviada no header `X-Signature`
- Cada endpoint de webhook cadastrado tem uma secret única, não compartilhada entre customers
- Secret é rotacionável via API, com grace period de 24 horas em que a secret antiga permanece válida
- URL do webhook deve ser HTTPS; cadastro com HTTP é rejeitado na validação
- Endpoints de CRUD de configuração de webhook exigem apenas autenticação normal (qualquer papel)
- Endpoint de reprocessamento manual de dead letter exige papel ADMIN

Observabilidade
- Logs estruturados via Pino, reaproveitando o padrão de logging já existente no projeto
- Log de auditoria registrando qual usuário executou o reprocessamento manual de um evento em dead letter
- [Hipótese] Tracing distribuído ponta a ponta não foi discutido na reunião; aplicar apenas se já existir como padrão no restante do projeto.

Confiabilidade e integridade de dados
- Inserção do evento na `webhook_outbox` ocorre dentro da mesma transação SQL da mudança de status do pedido; rollback da transação principal remove o evento junto
- Garantia de entrega at-least-once; o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar usando `X-Event-Id`
- Payload do evento é um snapshot renderizado no momento da inserção na outbox, não recalculado no momento do envio

Compatibilidade e portabilidade
- Payload em JSON, com `Content-Type: application/json`
- Limite de 64KB por payload de evento; eventos que ultrapassam o limite não são enviados e geram erro

Compliance
- Trilha de auditoria do reprocessamento manual de dead letter, registrando quem executou o replay

Acessibilidade no frontend consumidor
- Não aplicável nesta fase: a feature expõe apenas endpoints de API, sem interface visual própria (dashboard do cliente está fora de escopo)

---

### Arquitetura e abordagem

Abordagem
- Padrão outbox implementado no MySQL já usado pelo projeto, com um worker dedicado processando os eventos pendentes via polling. Evita subir infraestrutura adicional (como Redis Streams) para um time pequeno.

Componentes
- Módulo `src/modules/webhooks` (controller, service, repository, routes, schemas), seguindo o mesmo padrão dos demais módulos do projeto (orders, products, customers, users)
- Tabela `webhook_outbox` no MySQL, com índice em status e `created_at`
- Tabela `webhook_dead_letter` separada, armazenando payload, motivo da falha e timestamp dos eventos com falha permanente
- Worker em processo separado (`src/worker.ts`), com a lógica de processamento em um arquivo próprio do módulo (ex: `webhook.worker.ts` ou `webhook.processor.ts`), rodando via script `npm run worker`
- Instância própria de `PrismaClient` no processo do worker, conectada ao mesmo banco de dados da API
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação de `OrderService.changeStatus`

Integrações
- `OrderService.changeStatus` insere eventos na outbox como parte da mesma transação de mudança de status
- Worker realiza chamadas HTTP outbound, assinadas com HMAC-SHA256, para os endpoints de webhook cadastrados pelos clientes B2B

### Decisões e trade-offs

#### Decisão: Outbox pattern em MySQL em vez de fila dedicada (ex: Redis Streams)
- **Justificativa:** time pequeno, evita subir infraestrutura adicional; garante atomicidade reaproveitando a transação SQL já existente na mudança de status
- **Trade-off:** throughput e latência ficam limitados ao ciclo de polling; escalar horizontalmente o processamento é mais complexo no futuro

#### Decisão: Worker em polling a cada 2 segundos, em vez de listener reativo
- **Justificativa:** MySQL não tem um mecanismo nativo equivalente ao NOTIFY/LISTEN do Postgres; alternativas via trigger exigiriam soluções improvisadas; polling de 2s já atende ao requisito de latência abaixo de 10s
- **Trade-off:** latência mínima de até 2 segundos mesmo no melhor caso, e uso constante de ciclos de polling

#### Decisão: Worker roda como processo separado da API
- **Justificativa:** evita que o worker seja derrubado junto com reinícios da API
- **Trade-off:** mais um processo para operar em produção, exigindo instância própria de `PrismaClient`

#### Decisão: Retry com 5 tentativas e backoff exponencial (1m/5m/30m/2h/12h)
- **Justificativa:** cobre janelas de indisponibilidade de cliente de até aproximadamente 15 horas, cenário já observado com um cliente em manutenção planejada de 2 horas; 3 tentativas foi considerado agressivo demais
- **Trade-off:** eventos com falha ficam pendentes por até ~15 horas antes de serem movidos para dead letter

#### Decisão: Dead letter em tabela separada (`webhook_dead_letter`), em vez de status "failed" na própria outbox
- **Justificativa:** mantém a leitura da outbox principal limpa e enxuta; serve como evidência para debug e reprocessamento
- **Trade-off:** mais uma tabela para manter sincronizada com a outbox

#### Decisão: Reprocessamento de dead letter é manual, via endpoint administrativo com papel ADMIN
- **Justificativa:** mexer na fila de entrega de notificações não é uma ação de operador comum; precisa de controle de auditoria
- **Trade-off:** não há reprocessamento automático; sempre exige intervenção humana

#### Decisão: Entrega garantida at-least-once, com deduplicação via `X-Event-Id` do lado do cliente
- **Justificativa:** garantir exactly-once exigiria coordenação complexa entre os dois lados; é o padrão adotado por provedores de mercado como Stripe e GitHub
- **Trade-off:** a responsabilidade de deduplicar eventos recai sobre o cliente

#### Decisão: Payload do evento é um snapshot renderizado no momento da inserção na outbox
- **Justificativa:** garante que o evento reflita o estado do pedido no momento exato da mudança de status, mesmo que o pedido mude novamente depois
- **Trade-off:** nenhum trade-off relevante destacado na reunião

#### Decisão: Identificadores em UUID para as novas tabelas (outbox e dead letter)
- **Justificativa:** segue o padrão já usado em todo o restante do projeto, que usa UUID como identificador
- **Trade-off:** nenhum trade-off relevante destacado na reunião

#### Decisão: Filtro de eventos por status é aplicado no momento da inserção na outbox, não no momento do envio
- **Justificativa:** economiza linhas na tabela, evitando registrar eventos que nenhum webhook do customer deseja receber
- **Trade-off:** nenhum trade-off relevante destacado na reunião

#### Decisão: Reuso máximo dos padrões já existentes no projeto
- **Justificativa:** o módulo de webhooks reaproveita `AppError`, Pino, o middleware de erro centralizado, o padrão de módulos (`src/modules`), o padrão de schemas Zod e o prefixo de código de erro `WEBHOOK_`, mantendo consistência com o restante da base de código
- **Trade-off:** nenhum trade-off relevante destacado na reunião

---

### Dependências

#### Organizacional: Revisão de segurança dedicada
A engenheira de segurança (Sofia) precisa reservar pelo menos 2 dias úteis para revisar o código de geração de secret e assinatura HMAC antes do deploy em produção.

#### Organizacional: Documentação para os clientes no portal do desenvolvedor
O Product Manager (Marcos) precisa documentar no portal do desenvolvedor o formato do payload, os headers enviados e a necessidade de deduplicação por `X-Event-Id`, para orientar a integração dos clientes B2B.

#### Comercial: Confirmação de prazo com a Atlas Comercial
O Product Manager precisa confirmar com a Atlas Comercial o prazo estimado de entrega (fim de novembro, três sprints), já que foi o cliente que sinalizou risco de migração para concorrente.

#### Técnica: Nova entry point e script de execução do worker
É necessário criar um novo ponto de entrada (`src/worker.ts`) e um script (`npm run worker`) para rodar o worker como processo separado da API, conectando-se ao mesmo banco de dados.

---

### Riscos e mitigação

#### Cliente fica indisponível por período prolongado e esgota todas as tentativas de retry
- **Probabilidade:** media
- **Impacto:** cliente deixa de receber notificações de mudança de status pelo canal de webhook, precisando recorrer a outro meio (como polling manual) até a intervenção de um operador
- **Mitigação:**
  - Backoff exponencial com 5 tentativas cobrindo até aproximadamente 15 horas de indisponibilidade
  - Persistência do evento em dead letter com payload e motivo da falha para investigação
  - Endpoint administrativo de reprocessamento manual
- **Plano de contingência:** cliente aciona o suporte; operador ADMIN reprocessa o evento manualmente via replay do dead letter

#### Perda da garantia de ordering ao escalar para múltiplos workers no futuro
- **Probabilidade:** baixa (cenário atual é single-worker)
- **Impacto:** cliente pode receber eventos de um mesmo pedido fora da ordem em que ocorreram
- **Mitigação:**
  - Manter operação com um único worker enquanto a garantia de ordering for necessária
  - Documentar explicitamente a limitação conhecida (ordering garantida apenas por `order_id` e apenas em modo single-worker)
- **Plano de contingência:** caso seja necessário escalar para múltiplos workers, implementar particionamento por `order_id` ou lock pessimista antes de escalar

#### Vazamento de secret de um endpoint de webhook
- **Probabilidade:** media (já houve um incidente anterior de vazamento de secret em log de aplicação de um cliente)
- **Impacto:** possibilidade de forjar assinaturas HMAC válidas para eventos daquele endpoint específico
- **Mitigação:**
  - Secret única por endpoint de webhook, nunca uma secret global da plataforma
  - Suporte a rotação de secret via API
  - Grace period de 24 horas permitindo migração sem interrupção do cliente
- **Plano de contingência:** cliente solicita rotação imediata da secret comprometida; a secret antiga é invalidada ao fim do grace period

#### Sobrecarga do cliente com rajadas de eventos simultâneos
- **Probabilidade:** baixa
- **Impacto:** cliente pode receber muitas chamadas simultâneas se vários pedidos dele mudarem de status em um curto intervalo, já que não há rate limiting de saída implementado
- **Mitigação:**
  - Ponto em aberto registrado para observação após o lançamento em produção
- **Plano de contingência:** implementar rate limiting de saída caso o problema seja observado em produção

#### Entrega duplicada de eventos por conta da garantia at-least-once
- **Probabilidade:** media
- **Impacto:** cliente pode processar o mesmo evento mais de uma vez caso não implemente deduplicação do lado dele
- **Mitigação:**
  - Envio de `X-Event-Id` único (UUID) por evento para permitir deduplicação do lado do cliente
  - Documentação clara no portal do desenvolvedor sobre a necessidade de deduplicação
- **Plano de contingência:** nenhuma ação adicional do lado do provedor; a responsabilidade de deduplicar é documentada como sendo do cliente

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Toda mudança de status de pedido que corresponda a um status assinado por pelo menos um webhook ativo do customer gera uma linha na `webhook_outbox` dentro da mesma transação da mudança de status
- Se a transação de mudança de status sofrer rollback, nenhum evento correspondente permanece na `webhook_outbox`
- O worker entrega o evento ao endpoint do cliente em até 10 segundos após o commit da transação, no cenário sem falhas do cliente
- Toda requisição de webhook enviada contém os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`
- O payload enviado ao cliente não inclui a lista de itens do pedido
- Eventos que falham nas 5 tentativas de retry (1m/5m/30m/2h/12h) são movidos para `webhook_dead_letter` com payload, motivo da falha e timestamp
- `POST /admin/webhooks/dead-letter/:id/replay` exige papel ADMIN e registra log de auditoria de quem executou o replay
- Cadastro de webhook com URL que não usa HTTPS é rejeitado com erro de validação
- Payload de evento maior que 64KB não é enviado e gera erro
- Rotação de secret mantém a secret antiga válida por 24 horas em paralelo com a nova
- `GET /webhooks/:id/deliveries` retorna histórico com status de sucesso ou falha, payload enviado, resposta recebida e tempo de resposta
- Erros do módulo de webhooks seguem o padrão `AppError` com códigos prefixados por `WEBHOOK_`

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a lógica de retry, cálculo de backoff e geração/verificação de assinatura HMAC
- Testes de integração cobrindo o fluxo completo: mudança de status, inserção na outbox, processamento pelo worker e chamada HTTP simulada ao endpoint do cliente
- Teste de integração garantindo atomicidade: rollback da transação de mudança de status não deixa evento na outbox
- Teste de segurança dedicado para geração de secret e verificação de HMAC, com revisão da engenheira de segurança
- Teste de autorização garantindo que o endpoint de replay de dead letter exige papel ADMIN

Estratégia de validação
- Revisão de segurança dedicada da engenheira de segurança (mínimo 2 dias úteis), focada em HMAC e geração de secret, antes do deploy
- Sessão de revisão de design entre a Tech Lead e os engenheiros do time antes do início da implementação
- QA guiado por roteiro cobrindo cadastro, edição, remoção, filtro de eventos, histórico de entregas e replay de dead letter
