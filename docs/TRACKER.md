# Tracker

Tabela de rastreabilidade que mapeia cada item registrado nos documentos gerados (`docs/PRD.md`, `docs/FDD.md`, `docs/RFC.md`, `docs/adrs/*.md`) à sua origem, seja na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte existente. Itens marcados como `[Hipótese]` no PRD (defaults não discutidos na reunião) não têm linha correspondente aqui, pois por definição não têm origem em TRANSCRICAO ou CODIGO. O `docs/FDD.md` foi revisado para conter apenas itens com menção literal ou parafraseada em `TRANSCRICAO.md`, exceto na seção 9 ("Integração com o sistema existente") e nos status codes HTTP dos contratos da seção 5, que são explicitamente `Fonte = CODIGO` (convenção REST já usada no projeto, não discutida na reunião) — ambos adicionados depois para atender ao critério de aceite do enunciado que exige referência a caminhos reais de código. O `docs/RFC.md` é derivado inteiramente da reunião, sem referências a CODIGO. Os 6 primeiros ADRs em `docs/adrs/` foram gerados com o plugin `adrs-management` a partir de "potential ADRs" redigidos manualmente com evidência de `TRANSCRICAO.md` e dos demais documentos (a feature ainda não foi implementada em código, então não há evidência de CODIGO possível para essas decisões). O `ADR-007` foi escrito diretamente no mesmo formato numa correção posterior, e por cobrir a decisão de reuso dos padrões existentes do projeto, referencia arquivos reais do código-base (`Fonte = CODIGO`), diferente dos demais.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Público-alvo: clientes B2B Atlas Comercial, MaxDistribuição, Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Feature roda no sistema existente, com novo módulo `src/modules/webhooks` e worker separado | TRANSCRICAO | [09:11] Larissa |
| PRD-CTX-03 | docs/PRD.md | Contexto | Feature roda no sistema existente, com novo módulo `src/modules/webhooks` e worker separado | TRANSCRICAO | [09:27] Bruno |
| PRD-PROB-01 | docs/PRD.md | Requisito Funcional (origem do problema) | Clientes fazem polling recorrente em `GET /orders`, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Restrição | Risco de churn: Atlas pode migrar para concorrente se não entregar até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-GOAL-01 | docs/PRD.md | Requisito Não Funcional | Meta de latência: abaixo de 10 segundos é considerado "tempo real" pelo cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-GOAL-02 | docs/PRD.md | Restrição | Meta de prazo: fim de novembro, estimado em 3 sprints incluindo revisão da Sofia | TRANSCRICAO | [09:45]-[09:47] Marcos, Larissa |
| PRD-SCOPE-IN-01 | docs/PRD.md | Requisito Funcional | CRUD de configuração de webhook (criar, editar, remover, listar) | TRANSCRICAO | [09:31]-[09:33] Marcos, Bruno |
| PRD-SCOPE-IN-02 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status por webhook | TRANSCRICAO | [09:33]-[09:34] Marcos, Diego, Bruno |
| PRD-SCOPE-IN-03 | docs/PRD.md | Requisito Funcional | Histórico de entregas de webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-SCOPE-IN-04 | docs/PRD.md | Requisito Funcional | Endpoint admin de replay de dead letter | TRANSCRICAO | [09:18], [09:35] Diego, Larissa |
| PRD-SCOPE-IN-05 | docs/PRD.md | Decisão | Outbox pattern integrado à transação de `changeStatus` | TRANSCRICAO | [09:06], [09:40]-[09:41] Diego, Bruno |
| PRD-SCOPE-IN-06 | docs/PRD.md | Decisão | Worker dedicado com polling | TRANSCRICAO | [09:09], [09:11] Diego, Larissa |
| PRD-SCOPE-IN-07 | docs/PRD.md | Requisito Funcional | Retry com backoff exponencial e DLQ | TRANSCRICAO | [09:15]-[09:18] Diego |
| PRD-SCOPE-IN-08 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 e rotação de secret | TRANSCRICAO | [09:20]-[09:22] Sofia |
| PRD-SCOPE-IN-09 | docs/PRD.md | Requisito Funcional | Payload e headers padronizados do webhook | TRANSCRICAO | [09:43]-[09:45] Diego, Sofia |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Notificação por email em falha repetida fora de escopo (fase futura) | TRANSCRICAO | [09:37]-[09:38] Larissa, Marcos |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Rate limiting de envio fora de escopo, será observado depois | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Dashboard visual para o cliente fora de escopo | TRANSCRICAO | [09:39]-[09:40] Marcos, Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Ordering global entre múltiplos workers não é garantida | TRANSCRICAO | [09:12]-[09:13] Diego, Larissa |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Trade-off | Exactly-once não é garantido, apenas at-least-once | TRANSCRICAO | [09:24]-[09:25] Diego |
| PRD-SCOPE-OUT-06 | docs/PRD.md | Restrição | Arquivamento de eventos entregues após 30 dias fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | FR-001 Cadastro de webhook (POST), secret gerada e devolvida na criação | TRANSCRICAO | [09:31]-[09:32] Marcos, Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | FR-002 Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | FR-003 Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | FR-004 Listagem de webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | FR-005 Publicação de evento na outbox na mesma transação de `changeStatus`, com filtro na inserção | TRANSCRICAO | [09:40]-[09:41], [09:34] Bruno |
| PRD-FR-05-CODE | docs/PRD.md | Requisito Funcional | Método `changeStatus` do `OrderService` onde o evento de outbox deve ser inserido | CODIGO | src/modules/orders/order.service.ts |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | FR-006 Worker de processamento com polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-FR-06-HEADERS | docs/PRD.md | Requisito Funcional | Headers enviados na chamada do worker (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id) | TRANSCRICAO | [09:44]-[09:45] Diego, Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | FR-007 Retry com backoff exponencial (1m/5m/30m/2h/12h), 5 tentativas, depois DLQ | TRANSCRICAO | [09:15]-[09:18] Diego, Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | FR-008 Replay manual de dead letter via endpoint admin, exige ADMIN e log de auditoria | TRANSCRICAO | [09:18], [09:35]-[09:36] Diego, Larissa, Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | FR-009 Histórico de entregas (GET /webhooks/:id/deliveries) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | FR-010 Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-PERF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-PERF-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de chamada HTTP do worker | TRANSCRICAO | [09:42] Diego, Sofia |
| PRD-NFR-SEC-01 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 do payload no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-SEC-02 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint de webhook, não global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-SEC-03 | docs/PRD.md | Requisito Não Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-SEC-04 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório, URL deve ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-SEC-05 | docs/PRD.md | Requisito Não Funcional | CRUD de configuração exige apenas autenticação normal | TRANSCRICAO | [09:36]-[09:37] Sofia, Marcos |
| PRD-NFR-SEC-06 | docs/PRD.md | Requisito Não Funcional | Replay de DLQ exige papel ADMIN | TRANSCRICAO | [09:36] Sofia, Larissa |
| PRD-NFR-SEC-06-CODE | docs/PRD.md | Requisito Não Funcional | Middleware `requireRole` reaproveitado para exigir ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-NFR-OBS-01 | docs/PRD.md | Requisito Não Funcional | Logs estruturados via Pino, reaproveitando padrão existente | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-OBS-01-CODE | docs/PRD.md | Requisito Não Funcional | Implementação atual do logger Pino usada como padrão a reaproveitar | CODIGO | src/shared/logger/index.ts |
| PRD-NFR-OBS-02 | docs/PRD.md | Requisito Não Funcional | Log de auditoria de quem executou o replay de dead letter | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-REL-01 | docs/PRD.md | Restrição | Inserção do evento na outbox deve estar na mesma transação da mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| PRD-NFR-REL-01-CODE | docs/PRD.md | Restrição | Transação atual em `changeStatus` que precisa incluir a inserção na outbox | CODIGO | src/modules/orders/order.service.ts |
| PRD-NFR-REL-02 | docs/PRD.md | Trade-off | Garantia at-least-once, dedup por X-Event-Id do lado do cliente | TRANSCRICAO | [09:24]-[09:26] Diego, Sofia, Marcos |
| PRD-NFR-REL-03 | docs/PRD.md | Decisão | Payload é snapshot renderizado no momento da inserção na outbox | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego, Bruno |
| PRD-NFR-COMPAT-01 | docs/PRD.md | Requisito Não Funcional | Payload em JSON, Content-Type application/json | TRANSCRICAO | [09:44] Diego |
| PRD-NFR-COMPAT-02 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB por payload, erro se ultrapassar | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego, Larissa |
| PRD-NFR-COMPLI-01 | docs/PRD.md | Requisito Não Funcional | Trilha de auditoria do replay manual de dead letter | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-A11Y-01 | docs/PRD.md | Restrição | Não aplicável nesta fase: sem frontend próprio, dashboard fora de escopo | TRANSCRICAO | [09:39]-[09:40] Larissa, Marcos |
| PRD-ARCH-01 | docs/PRD.md | Decisão | Módulo `src/modules/webhooks` segue padrão de módulo do projeto (controller, service, repository, routes, schemas) | TRANSCRICAO | [09:27]-[09:28] Bruno, Diego |
| PRD-ARCH-01-CODE | docs/PRD.md | Decisão | Estrutura de módulo existente usada como referência de padrão (controller/service/repository/routes/schemas) | CODIGO | src/modules/orders/ |
| PRD-ARCH-02 | docs/PRD.md | Decisão | Worker separado `src/worker.ts`, com lógica em `webhook.worker.ts`/`webhook.processor.ts` | TRANSCRICAO | [09:11], [09:28] Larissa, Bruno |
| PRD-ARCH-02-CODE | docs/PRD.md | Decisão | Entry point de API existente usado como referência para a nova entry point do worker | CODIGO | src/server.ts |
| PRD-ARCH-03 | docs/PRD.md | Decisão | Worker usa instância própria de PrismaClient, mesmo banco | TRANSCRICAO | [09:29]-[09:30] Diego, Bruno |
| PRD-ARCH-03-CODE | docs/PRD.md | Decisão | Configuração atual de conexão Prisma reaproveitada pelo worker | CODIGO | src/config/database.ts |
| PRD-ARCH-04 | docs/PRD.md | Decisão | Tabela `webhook_outbox` com índice em status e created_at | TRANSCRICAO | [09:07]-[09:08] Diego |
| PRD-ARCH-05 | docs/PRD.md | Decisão | Tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| PRD-ARCH-06 | docs/PRD.md | Decisão | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação | TRANSCRICAO | [09:41] Bruno, Diego |
| PRD-ARCH-07 | docs/PRD.md | Decisão | Erros do módulo seguem `AppError` com prefixo `WEBHOOK_` | TRANSCRICAO | [09:28]-[09:29] Bruno |
| PRD-ARCH-07-CODE | docs/PRD.md | Decisão | Classe `AppError` e classes de erro HTTP existentes usadas como padrão a seguir | CODIGO | src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox em MySQL em vez de fila dedicada (Redis Streams) | TRANSCRICAO | [09:06]-[09:07] Diego, Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker em polling de 2s em vez de listener reativo | TRANSCRICAO | [09:09] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | Worker roda como processo separado da API | TRANSCRICAO | [09:11] Diego, Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão | Retry de 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15]-[09:17] Diego, Bruno, Larissa |
| PRD-DEC-05 | docs/PRD.md | Decisão | DLQ em tabela separada em vez de status "failed" na outbox | TRANSCRICAO | [09:18] Diego, Bruno |
| PRD-DEC-06 | docs/PRD.md | Decisão | Replay de DLQ manual, via endpoint admin com papel ADMIN | TRANSCRICAO | [09:18], [09:35]-[09:36] Diego, Sofia |
| PRD-DEC-07 | docs/PRD.md | Decisão | At-least-once com dedup por X-Event-Id, não exactly-once | TRANSCRICAO | [09:24]-[09:26] Diego |
| PRD-DEC-08 | docs/PRD.md | Decisão | Payload é snapshot renderizado na inserção | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego, Bruno |
| PRD-DEC-09 | docs/PRD.md | Decisão | IDs em UUID nas novas tabelas | TRANSCRICAO | [09:50]-[09:51] Diego, Larissa |
| PRD-DEC-10 | docs/PRD.md | Decisão | Filtro de eventos aplicado na inserção da outbox, não no envio | TRANSCRICAO | [09:33]-[09:34] Marcos, Diego, Bruno |
| PRD-DEC-11 | docs/PRD.md | Decisão | Reuso máximo dos padrões já existentes no projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (mínimo 2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação de integração no portal do desenvolvedor pelo Marcos | TRANSCRICAO | [09:26], [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas Comercial | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Nova entry point e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| PRD-DEP-04-CODE | docs/PRD.md | Dependência | Scripts npm existentes usados como referência para o novo script `worker` | CODIGO | package.json |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente indisponível por período prolongado esgota tentativas de retry | TRANSCRICAO | [09:14]-[09:17] Larissa, Diego, Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Perda de ordering ao escalar para múltiplos workers no futuro | TRANSCRICAO | [09:12]-[09:13] Diego, Bruno, Larissa |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret de um endpoint de webhook | TRANSCRICAO | [09:21]-[09:22] Sofia, Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Sobrecarga do cliente com rajadas de eventos simultâneos | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| PRD-RISK-05 | docs/PRD.md | Risco | Entrega duplicada de eventos por conta do at-least-once | TRANSCRICAO | [09:24]-[09:26] Diego, Sofia, Marcos |
| PRD-AC-01 | docs/PRD.md | Requisito Funcional | Critério: evento gerado na outbox na mesma transação da mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| PRD-AC-02 | docs/PRD.md | Requisito Funcional | Critério: rollback da transação não deixa evento na outbox | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| PRD-AC-03 | docs/PRD.md | Requisito Não Funcional | Critério: entrega em até 10s após commit | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-04 | docs/PRD.md | Requisito Funcional | Critério: headers obrigatórios em toda requisição de webhook | TRANSCRICAO | [09:44]-[09:45] Diego, Sofia |
| PRD-AC-05 | docs/PRD.md | Requisito Funcional | Critério: payload não inclui items do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-AC-06 | docs/PRD.md | Requisito Funcional | Critério: eventos com falha nas 5 tentativas vão para dead letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-AC-07 | docs/PRD.md | Requisito Funcional | Critério: replay de DLQ exige ADMIN e loga auditoria | TRANSCRICAO | [09:35]-[09:36] Larissa, Sofia |
| PRD-AC-08 | docs/PRD.md | Requisito Não Funcional | Critério: cadastro com URL não HTTPS é rejeitado | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-09 | docs/PRD.md | Requisito Não Funcional | Critério: payload maior que 64KB não é enviado e gera erro | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego |
| PRD-AC-10 | docs/PRD.md | Requisito Não Funcional | Critério: rotação de secret mantém a antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-11 | docs/PRD.md | Requisito Funcional | Critério: histórico de entregas retorna status, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-12 | docs/PRD.md | Requisito Não Funcional | Critério: erros do módulo seguem AppError com prefixo WEBHOOK_ | TRANSCRICAO | [09:28]-[09:29] Bruno |
| PRD-TEST-01 | docs/PRD.md | Decisão | Testes unitários de retry, backoff e HMAC | TRANSCRICAO | [09:15]-[09:20] Diego, Sofia |
| PRD-TEST-02 | docs/PRD.md | Decisão | Teste de integração do fluxo completo outbox → worker → envio | TRANSCRICAO | [09:06]-[09:09] Diego |
| PRD-TEST-03 | docs/PRD.md | Decisão | Teste de atomicidade (rollback não deixa evento na outbox) | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| PRD-TEST-04 | docs/PRD.md | Decisão | Revisão de segurança dedicada a HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-05 | docs/PRD.md | Decisão | Sessão de revisão de design entre Larissa, Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| FDD-CTX-01 | docs/FDD.md | Restrição | Chamada HTTP síncrona dentro da transação de mudança de status bloquearia outros pedidos | TRANSCRICAO | [09:04] Bruno |
| FDD-CTX-02 | docs/FDD.md | Decisão | Novo módulo segue padrão existente: cada domínio em `src/modules` com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| FDD-CTX-03 | docs/FDD.md | Decisão | Worker como entry point separada, PrismaClient próprio, mesma DATABASE_URL, mesmo banco, outro processo Node | TRANSCRICAO | [09:11] Larissa, [09:28]-[09:30] Bruno, Diego |
| FDD-CTX-04 | docs/FDD.md | Requisito Funcional | Mudança de status é feita no método `changeStatus` do service de orders (`OrderService`) | TRANSCRICAO | [09:40]-[09:41] Bruno |
| FDD-CTX-05 | docs/FDD.md | Requisito Funcional | Clientes B2B são os destinatários das notificações de webhook | TRANSCRICAO | [09:00] Marcos |
| FDD-CTX-06 | docs/FDD.md | Requisito Funcional | Usuário ADMIN é responsável pelo reprocessamento manual, com log de auditoria | TRANSCRICAO | [09:35]-[09:36] Larissa, Sofia |
| FDD-CTX-07 | docs/FDD.md | Restrição | Comunicação é exclusivamente outbound, sistema não recebe webhooks de terceiros | TRANSCRICAO | [09:02]-[09:03] Sofia, Marcos |
| FDD-CTX-08 | docs/FDD.md | Restrição | Escopo desta entrega é single-worker | TRANSCRICAO | [09:12]-[09:13] Diego |
| FDD-OBJ-01 | docs/FDD.md | Requisito Não Funcional | Objetivo técnico: atomicidade entre mudança de status e registro do evento (outbox) | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-02 | docs/FDD.md | Requisito Não Funcional | Objetivo técnico: entrega em até 10s com polling de 2s | TRANSCRICAO | [09:02], [09:09] Marcos, Diego |
| FDD-OBJ-03 | docs/FDD.md | Requisito Não Funcional | Objetivo técnico: at-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:24]-[09:25] Diego |
| FDD-OBJ-04 | docs/FDD.md | Requisito Não Funcional | Objetivo técnico: timeout de 10s por tentativa HTTP outbound | TRANSCRICAO | [09:42] Diego |
| FDD-OBJ-05 | docs/FDD.md | Requisito Não Funcional | Objetivo técnico: ordering por pedido preservada em single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-SCOPE-IN-01 | docs/FDD.md | Requisito Funcional | Cadastro de webhook: url, secret gerada e devolvida na criação, lista de status, customer_id | TRANSCRICAO | [09:31]-[09:33] Marcos, Bruno |
| FDD-SCOPE-IN-02 | docs/FDD.md | Requisito Funcional | Edição (PATCH), remoção (DELETE) e listagem (GET) dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-SCOPE-IN-03 | docs/FDD.md | Requisito Funcional | Filtro de eventos por status em cada webhook | TRANSCRICAO | [09:33]-[09:34] Marcos, Diego, Bruno |
| FDD-SCOPE-IN-04 | docs/FDD.md | Requisito Funcional | Histórico de entregas (`GET /webhooks/:id/deliveries`) | TRANSCRICAO | [09:34] Marcos |
| FDD-SCOPE-IN-05 | docs/FDD.md | Requisito Funcional | Endpoint admin de replay de dead letter (`POST /admin/webhooks/dead-letter/:id/replay`) | TRANSCRICAO | [09:18], [09:35] Diego, Larissa |
| FDD-SCOPE-IN-06 | docs/FDD.md | Decisão | `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação de `changeStatus` | TRANSCRICAO | [09:41] Bruno, Diego |
| FDD-SCOPE-IN-07 | docs/FDD.md | Decisão | Worker dedicado, processo separado, polling de 2 segundos | TRANSCRICAO | [09:09], [09:11] Diego, Larissa |
| FDD-SCOPE-IN-08 | docs/FDD.md | Decisão | Tabela `webhook_outbox` com índice em status e created_at | TRANSCRICAO | [09:07]-[09:08] Diego |
| FDD-SCOPE-IN-09 | docs/FDD.md | Decisão | Tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| FDD-SCOPE-IN-10 | docs/FDD.md | Requisito Não Funcional | HMAC-SHA256 e rotação de secret com grace period de 24h | TRANSCRICAO | [09:20]-[09:22] Sofia |
| FDD-SCOPE-EXCL-01 | docs/FDD.md | Restrição | Recebimento de webhooks de terceiros fora de escopo | TRANSCRICAO | [09:02]-[09:03] Sofia, Marcos |
| FDD-SCOPE-EXCL-02 | docs/FDD.md | Restrição | Email de falha, rate limiting e dashboard fora de escopo | TRANSCRICAO | [09:37]-[09:40] Larissa, Marcos, Diego |
| FDD-SCOPE-EXCL-03 | docs/FDD.md | Restrição | Ordering global multi-worker e exactly-once fora de escopo | TRANSCRICAO | [09:12]-[09:13], [09:24]-[09:25] Diego |
| FDD-SCOPE-EXCL-04 | docs/FDD.md | Restrição | Arquivamento de eventos entregues fora de escopo desta feature | TRANSCRICAO | [09:08] Diego |
| FDD-FLOW-01 | docs/FDD.md | Requisito Funcional | Mudança de status feita no método `changeStatus` do service de orders | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Requisito Funcional | Transação atual já atualiza orders, insere history e ajusta stock | TRANSCRICAO | [09:04] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Requisito Funcional | `publishWebhookEvent` chamada dentro da mesma transação | TRANSCRICAO | [09:41] Bruno, Diego |
| FDD-FLOW-04 | docs/FDD.md | Decisão | Filtro por status aplicado na inserção da outbox, não no envio | TRANSCRICAO | [09:33]-[09:34] Marcos, Diego, Bruno |
| FDD-FLOW-05 | docs/FDD.md | Restrição | Rollback da transação principal remove o evento junto | TRANSCRICAO | [09:06] Diego |
| FDD-FLOW-06 | docs/FDD.md | Requisito Funcional | Worker faz polling em lote pequeno a cada 2s | TRANSCRICAO | [09:08]-[09:09] Diego |
| FDD-FLOW-07 | docs/FDD.md | Requisito Funcional | Worker assina com HMAC e usa timeout de 10s por chamada | TRANSCRICAO | [09:20] Sofia, [09:42] Diego |
| FDD-FLOW-08 | docs/FDD.md | Requisito Funcional | Retry com backoff em caso de falha | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-FLOW-09 | docs/FDD.md | Requisito Funcional | Evento movido para dead letter após 5 tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-10 | docs/FDD.md | Requisito Funcional | Replay manual recoloca evento como pendente, com log de auditoria | TRANSCRICAO | [09:18], [09:35]-[09:36] Diego, Larissa, Sofia |
| FDD-FLOW-11 | docs/FDD.md | Requisito Não Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | Cadastro de webhook: secret gerada e devolvida apenas na criação | TRANSCRICAO | [09:31]-[09:32] Marcos, Larissa |
| FDD-CONTRATO-01-CAMPOS | docs/FDD.md | Requisito Funcional | Campos discutidos: url, statuses, customerId (body/path), active | TRANSCRICAO | [09:21], [09:31]-[09:33] Sofia, Marcos, Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH edita, DELETE remove, GET lista os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | Endpoint para o cliente pedir nova secret pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | `GET /webhooks/:id/deliveries`: últimos 100 envios, sucesso/falha, payload, resposta, tempo de resposta | TRANSCRICAO | [09:34]-[09:35] Marcos |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | `POST /admin/webhooks/dead-letter/:id/replay`, exige ADMIN | TRANSCRICAO | [09:18], [09:35] Diego, Larissa |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | Assinatura de `publishWebhookEvent(tx, order, fromStatus, toStatus)`, função pura recebendo o tx | TRANSCRICAO | [09:41] Bruno, Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | Headers de entrega: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44]-[09:45] Diego, Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Requisito Funcional | Campos do payload do evento (event_id, event_type, timestamp, order_id, etc.) | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Requisito Funcional | Payload não inclui items; cliente consulta GET /orders/:id depois | TRANSCRICAO | [09:43]-[09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Requisito Não Funcional | Limite de 64KB e timeout de 10s como limites do contrato | TRANSCRICAO | [09:23]-[09:24], [09:42] Sofia, Diego |
| FDD-ERR-01 | docs/FDD.md | Requisito Não Funcional | URL sem HTTPS recusada com erro de validação, código WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia, [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Requisito Não Funcional | Payload maior que 64KB não é enviado, gera erro | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego, Larissa |
| FDD-ERR-03 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s tratado como falha, segue para retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-04 | docs/FDD.md | Requisito Funcional | Cliente offline: retry com backoff em vez de falha imediata | TRANSCRICAO | [09:14]-[09:15] Larissa, Diego |
| FDD-ERR-05 | docs/FDD.md | Requisito Funcional | 5ª tentativa falha move evento para dead letter | TRANSCRICAO | [09:15]-[09:18] Diego |
| FDD-ERR-06 | docs/FDD.md | Requisito Não Funcional | Replay sem papel ADMIN é acesso negado | TRANSCRICAO | [09:35]-[09:36] Larissa, Sofia |
| FDD-ERR-07 | docs/FDD.md | Restrição | Falha ao inserir na outbox provoca rollback de toda a transação | TRANSCRICAO | [09:06], [09:40]-[09:41] Diego, Bruno |
| FDD-ERR-08 | docs/FDD.md | Requisito Não Funcional | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, padrão AppError prefixo WEBHOOK_ | TRANSCRICAO | [09:28] Bruno |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Estratégia de retry com backoff exponencial fixo (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-FALLBACK-01 | docs/FDD.md | Restrição | Sem fallback automático de canal (email fora de escopo) | TRANSCRICAO | [09:37]-[09:38] Larissa, Marcos |
| FDD-INV-01 | docs/FDD.md | Restrição | Invariante de atomicidade outbox/transação | TRANSCRICAO | [09:06], [09:40]-[09:41] Diego, Bruno |
| FDD-INV-02 | docs/FDD.md | Restrição | Invariante: X-Event-Id único e imutável por evento | TRANSCRICAO | [09:25] Diego |
| FDD-INV-03 | docs/FDD.md | Restrição | Invariante: payload é snapshot imutável após a inserção | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego, Bruno |
| FDD-INV-04 | docs/FDD.md | Restrição | Invariante: entrega é at-least-once, nunca exactly-once | TRANSCRICAO | [09:24]-[09:26] Diego |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Logs estruturados via Pino, já usado em todo o projeto | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Log de auditoria do replay manual de dead letter | TRANSCRICAO | [09:36] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Outbox roda no mesmo MySQL já usado pelo projeto, sem nova infraestrutura | TRANSCRICAO | [09:07] Diego |
| FDD-DEP-02 | docs/FDD.md | Dependência | Worker usa PrismaClient próprio, mesma DATABASE_URL, mesmo banco | TRANSCRICAO | [09:11] Bruno, [09:29]-[09:30] Diego, Bruno |
| FDD-DEP-03 | docs/FDD.md | Dependência | Validação via schema Zod (ex: URL HTTPS) | TRANSCRICAO | [09:23] Sofia |
| FDD-DEP-04 | docs/FDD.md | Dependência | Logger Pino já usado no projeto inteiro | TRANSCRICAO | [09:29] Bruno |
| FDD-COMPAT-01 | docs/FDD.md | Decisão | Erros seguem AppError com prefixo WEBHOOK_; middleware de erro centralizado não precisa mudar | TRANSCRICAO | [09:28]-[09:29] Bruno |
| FDD-AC-01 | docs/FDD.md | Requisito Funcional | Critério: evento inserido na outbox na mesma transação da mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno, Diego |
| FDD-AC-02 | docs/FDD.md | Requisito Funcional | Critério: rollback não deixa evento na outbox | TRANSCRICAO | [09:06], [09:40]-[09:41] Diego, Bruno |
| FDD-AC-03 | docs/FDD.md | Requisito Não Funcional | Critério: entrega em até 10s sem falhas do cliente | TRANSCRICAO | [09:02] Marcos |
| FDD-AC-04 | docs/FDD.md | Requisito Funcional | Critério: headers obrigatórios presentes em toda entrega | TRANSCRICAO | [09:44]-[09:45] Diego, Sofia |
| FDD-AC-05 | docs/FDD.md | Requisito Funcional | Critério: payload não contém items do pedido | TRANSCRICAO | [09:43] Diego |
| FDD-AC-06 | docs/FDD.md | Requisito Funcional | Critério: 5 falhas consecutivas movem evento para dead letter respeitando backoff | TRANSCRICAO | [09:15]-[09:18] Diego |
| FDD-AC-07 | docs/FDD.md | Requisito Não Funcional | Critério: replay exige ADMIN e loga auditoria | TRANSCRICAO | [09:35]-[09:36] Larissa, Sofia |
| FDD-AC-08 | docs/FDD.md | Requisito Não Funcional | Critério: URL não HTTPS é recusada | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-09 | docs/FDD.md | Requisito Não Funcional | Critério: payload maior que 64KB não é enviado | TRANSCRICAO | [09:23]-[09:24] Sofia, Diego |
| FDD-AC-10 | docs/FDD.md | Requisito Não Funcional | Critério: rotação mantém secret antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-11 | docs/FDD.md | Requisito Funcional | Critério: histórico de entregas retorna status, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-RISK-01 | docs/FDD.md | Risco | Cliente indisponível por período prolongado esgota retries | TRANSCRICAO | [09:14]-[09:17] Larissa, Diego, Marcos |
| FDD-RISK-02 | docs/FDD.md | Risco | Perda de ordering ao escalar para múltiplos workers | TRANSCRICAO | [09:12]-[09:13] Diego, Bruno, Larissa |
| FDD-RISK-03 | docs/FDD.md | Risco | Vazamento de secret de um endpoint de webhook | TRANSCRICAO | [09:21]-[09:22] Sofia, Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Entrega duplicada por conta do at-least-once | TRANSCRICAO | [09:24]-[09:26] Diego, Sofia, Marcos |
| FDD-STATUS-01 | docs/FDD.md | Requisito Não Funcional | Status codes 201/400/401 do cadastro de webhook, convenção REST do projeto | CODIGO | src/shared/errors/http-errors.ts |
| FDD-STATUS-02 | docs/FDD.md | Requisito Não Funcional | Status codes 200/204/404 de edição, remoção e listagem de webhook | CODIGO | src/shared/errors/http-errors.ts |
| FDD-STATUS-03 | docs/FDD.md | Requisito Não Funcional | Status codes 200/404 da rotação de secret e do histórico de entregas | CODIGO | src/shared/errors/http-errors.ts |
| FDD-STATUS-04 | docs/FDD.md | Requisito Não Funcional | Status codes 200/403/404 do replay manual de dead letter | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-01 | docs/FDD.md | Decisão | `publishWebhookEvent` deve ser chamada dentro do `$transaction` de `changeStatus`, após `orderStatusHistory.create` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Decisão | Novas classes de erro do módulo devem estender `AppError`/`NotFoundError`/`BadRequestError`/`ConflictError` | CODIGO | src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Decisão | Middleware de erro centralizado já trata `AppError`, não precisa de alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Decisão | Endpoint de replay deve usar `requireRole('ADMIN')` já existente | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Decisão | Módulo e worker devem reaproveitar o logger Pino já configurado, sem instanciar um novo | CODIGO | src/shared/logger/index.ts |
| FDD-INT-06 | docs/FDD.md | Decisão | Listagem de webhooks e histórico de entregas devem reaproveitar o helper `paginated` | CODIGO | src/shared/http/response.ts |
| FDD-INT-07 | docs/FDD.md | Decisão | Router de webhooks deve ser registrado seguindo o padrão de injeção já usado pelos demais módulos | CODIGO | src/routes/index.ts, src/app.ts |
| RFC-META-01 | docs/RFC.md | Decisão | Autora do RFC é quem abre o doc de design; revisores são os demais participantes da reunião | TRANSCRICAO | [09:50] Larissa; participantes listados no cabeçalho de TRANSCRICAO.md |
| RFC-CTX-01 | docs/RFC.md | Requisito Funcional (origem do problema) | Pedido formal de 3 clientes B2B, polling caro, ameaça de churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Requisito Não Funcional | "Tempo real" definido pelo cliente como abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Ponto de partida técnico é o `changeStatus`, transação já pesada | TRANSCRICAO | [09:04] Bruno |
| RFC-PROP-01 | docs/RFC.md | Decisão | Padrão outbox: `publishWebhookEvent` inserido na mesma transação de `changeStatus` | TRANSCRICAO | [09:06], [09:41] Diego, Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker dedicado (`src/worker.ts`), PrismaClient próprio, polling de 2s | TRANSCRICAO | [09:09], [09:11], [09:29]-[09:30] Diego, Larissa, Bruno |
| RFC-PROP-03 | docs/RFC.md | Requisito Não Funcional | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:20]-[09:22] Sofia |
| RFC-PROP-04 | docs/RFC.md | Requisito Funcional | Timeout 10s, retry com backoff (5 tentativas), DLQ, replay admin | TRANSCRICAO | [09:15]-[09:18], [09:35]-[09:36], [09:42] Diego, Larissa, Sofia |
| RFC-PROP-05 | docs/RFC.md | Trade-off | Garantia at-least-once, dedup fica a cargo do cliente | TRANSCRICAO | [09:24]-[09:26] Diego |
| RFC-PROP-06 | docs/RFC.md | Requisito Funcional | CRUD de configuração de webhook e histórico de entregas | TRANSCRICAO | [09:31]-[09:34] Marcos, Bruno |
| RFC-ALT-01 | docs/RFC.md | Decisão | Alternativa descartada: notificação síncrona dentro da transação de `changeStatus` | TRANSCRICAO | [09:03]-[09:05] Larissa, Bruno |
| RFC-ALT-02 | docs/RFC.md | Decisão | Alternativa descartada: fila/mensageria dedicada (ex: Redis Streams) | TRANSCRICAO | [09:06]-[09:07] Larissa, Diego |
| RFC-ALT-03 | docs/RFC.md | Decisão | Alternativa descartada: worker reativo via listener/trigger de banco, em favor de polling | TRANSCRICAO | [09:08]-[09:09] Bruno, Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Notificação de falha ao cliente (ex: email) adiada para fase futura | TRANSCRICAO | [09:37]-[09:38] Marcos, Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Rate limiting de envio: observar em produção e decidir depois | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Escalabilidade para múltiplos workers não definida, tratada como problema do futuro | TRANSCRICAO | [09:12]-[09:13] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Arquivamento de eventos entregues fora do escopo de definição desta entrega | TRANSCRICAO | [09:08] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Restrição | Proposta altera o `changeStatus`, ponto crítico do sistema | TRANSCRICAO | [09:40]-[09:41] Bruno |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Novo processo (worker) precisa rodar separado e ser observado | TRANSCRICAO | [09:11] Larissa, Diego |
| RFC-IMPACT-03 | docs/RFC.md | Risco | Superfície de segurança nova; revisão dedicada da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| RFC-IMPACT-04 | docs/RFC.md | Risco | Indisponibilidade prolongada de cliente pode esgotar retries | TRANSCRICAO | [09:14]-[09:17] Larissa, Diego, Marcos |
| RFC-IMPACT-05 | docs/RFC.md | Risco | Entrega duplicada (at-least-once); dedup é responsabilidade do cliente | TRANSCRICAO | [09:24]-[09:26] Diego, Sofia, Marcos |
| RFC-IMPACT-06 | docs/RFC.md | Restrição | Risco comercial: prazo de 3 sprints ligado à retenção da Atlas Comercial | TRANSCRICAO | [09:00] Marcos, [09:45]-[09:47] Marcos, Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md | Decisão | Padrão outbox em MySQL para entrega de webhooks, em vez de fila/mensageria dedicada | TRANSCRICAO | [09:03]-[09:08] Larissa, Bruno, Diego |
| ADR-002 | docs/adrs/ADR-002-worker-dedicado-polling-vs-listener-reativo.md | Decisão | Worker dedicado com polling de 2s, em vez de listener/trigger reativo de banco | TRANSCRICAO | [09:08]-[09:11] Bruno, Diego, Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-dead-letter-queue.md | Decisão | Retry com backoff exponencial (5 tentativas) e dead letter queue em tabela separada | TRANSCRICAO | [09:14]-[09:18] Larissa, Diego, Bruno, Marcos |
| ADR-004 | docs/adrs/ADR-004-garantia-entrega-at-least-once.md | Decisão | Garantia de entrega at-least-once com deduplicação pelo cliente via X-Event-Id | TRANSCRICAO | [09:24]-[09:26] Diego, Bruno, Sofia, Marcos |
| ADR-005 | docs/adrs/ADR-005-hmac-secret-por-endpoint-rotacao.md | Decisão | HMAC-SHA256 com secret única por endpoint de webhook e rotação com grace period de 24h | TRANSCRICAO | [09:18]-[09:22], [09:32] Sofia, Bruno, Diego |
| ADR-006 | docs/adrs/ADR-006-snapshot-payload-insercao-evento.md | Decisão | Snapshot do payload no momento da inserção do evento, em vez de renderização no envio | TRANSCRICAO | [09:51]-[09:52] Bruno, Larissa, Diego |
| ADR-007 | docs/adrs/ADR-007-reuso-padroes-existentes-projeto.md | Decisão | Reuso máximo dos padrões existentes do projeto (módulo, AppError, prefixo WEBHOOK_, Pino, error middleware) | TRANSCRICAO | [09:27]-[09:30] Bruno, Diego, Larissa |
| ADR-007-CODE | docs/adrs/ADR-007-reuso-padroes-existentes-projeto.md | Decisão | Padrão de módulo, classe AppError, subclasses de erro e middleware de erro reaproveitados como referência | CODIGO | src/modules/orders/order.service.ts, src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts, src/middlewares/error.middleware.ts, src/shared/logger/index.ts |
