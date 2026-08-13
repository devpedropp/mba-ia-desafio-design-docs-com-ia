### RFC: Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | 2026-08-13 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |

---

### Resumo executivo (TL;DR)

Propomos um sistema de webhooks outbound para notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) em tempo real quando o status de um pedido muda, eliminando o polling atual em `GET /orders`. A entrega é assíncrona via padrão outbox: o evento é gravado na mesma transação SQL da mudança de status, e um worker dedicado, em processo separado, lê essa outbox e envia as chamadas HTTP assinadas com HMAC-SHA256. Falhas de entrega são reprocessadas com retry e backoff exponencial antes de caírem em uma dead letter queue reprocessável manualmente. A solução reaproveita a infraestrutura já existente (MySQL, Prisma, padrões de módulo, erro e logging do projeto), evitando subir infraestrutura nova.

---

### Contexto e problema

Três clientes B2B pediram formalmente para serem notificados quando o status dos pedidos deles muda, em vez de continuarem consultando `GET /orders` repetidamente. Esse polling está deixando a integração deles lenta e cara, e a Atlas Comercial já sinalizou risco de migrar para um concorrente se isso não for entregue até o fim do trimestre. Os clientes consideram "tempo real" qualquer notificação entregue em menos de 10 segundos.

O ponto de partida técnico é o método `changeStatus` do service de orders, que hoje já executa uma transação relativamente pesada: atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`. Qualquer solução precisa notificar os clientes sem acoplar a latência ou a disponibilidade deles a essa transação crítica.

---

### Proposta técnica

A proposta é um módulo de webhooks outbound, seguindo o padrão de módulo já usado no projeto (controller, service, repository, routes, schemas), com os seguintes elementos:

- **Padrão outbox em MySQL**: dentro da mesma transação que muda o status do pedido, uma função (`publishWebhookEvent`) insere um evento numa tabela `webhook_outbox` para cada webhook do customer inscrito naquele status. Se a transação principal commitar, o evento existe; se ela sofrer rollback, o evento some junto. Não há inconsistência possível entre o estado do pedido e o evento gerado.
- **Worker dedicado**: um processo Node separado da API (`src/worker.ts`), usando sua própria instância de `PrismaClient` sobre o mesmo banco, faz polling da outbox a cada 2 segundos, processa os eventos pendentes em lote pequeno e envia as chamadas HTTP para os clientes.
- **Entrega assinada**: cada chamada é assinada com HMAC-SHA256, usando uma secret única por endpoint de webhook (não uma secret global), com suporte a rotação e um grace period de 24 horas para o cliente migrar.
- **Resiliência**: timeout de 10 segundos por tentativa; em caso de falha, retry com backoff exponencial (1m/5m/30m/2h/12h) por até 5 tentativas; falhas permanentes vão para uma tabela `webhook_dead_letter`, reprocessável manualmente por um endpoint restrito a papel ADMIN, com log de auditoria.
- **Garantia de entrega**: at-least-once, com um identificador único por evento para o cliente deduplicar do lado dele.
- **CRUD de configuração**: os clientes cadastram, editam, removem e listam seus próprios webhooks pela API, escolhendo quais status de pedido querem receber, e consultam o histórico das últimas entregas.

O detalhamento de contratos, payloads, matriz de erros e critérios de aceite técnicos está no FDD.

---

### Alternativas consideradas

**Alternativa 1: Notificação síncrona dentro da transação de mudança de status**
Chamar o endpoint do cliente diretamente dentro do `changeStatus`, antes do commit.
Trade-off que levou ao descarte: a transação já é pesada, e um cliente lento travaria mudanças de status de outros pedidos; além disso, não haveria uma decisão clara sobre dar rollback na mudança de status por causa de uma falha de rede externa ao sistema.

**Alternativa 2: Fila ou mensageria dedicada (ex: Redis Streams)**
Desacoplar produção e consumo dos eventos usando uma peça de infraestrutura de mensageria dedicada, em vez de uma tabela no banco relacional já existente.
Trade-off que levou ao descarte: o time é pequeno, e subir uma nova infraestrutura (por exemplo, um cluster Redis) foi considerado overengineering para o problema atual. O padrão outbox no MySQL já existente resolve sem essa complexidade operacional adicional.

**Alternativa 3: Worker reativo via listener/trigger de banco, em vez de polling**
Usar um mecanismo equivalente ao NOTIFY/LISTEN do Postgres para o worker ser avisado imediatamente de novos eventos.
Trade-off que levou ao descarte: MySQL não tem um listener nativo equivalente. Usar um trigger de banco para notificar um processo externo exigiria soluções improvisadas (escrever em arquivo, bater num endpoint), consideradas inadequadas. O polling de 2 segundos já atende com folga ao requisito de latência abaixo de 10 segundos.

---

### Questões em aberto

- **Notificação de falha ao cliente** (ex: email quando o webhook dele falha repetidamente): levantada como necessidade real, mas adiada para uma fase futura, após medirmos o impacto desta primeira entrega.
- **Rate limiting de envio**: se um cliente tiver muitos pedidos mudando de status em um curto intervalo, hoje ele pode receber várias chamadas simultâneas. Decidimos não implementar rate limiting nesta entrega; vamos observar em produção e decidir depois se isso vira um problema real.
- **Escalabilidade para múltiplos workers**: a solução atual garante ordering de eventos por pedido apenas enquanto houver um único worker processando a outbox. Se for necessário escalar para múltiplos workers no futuro, a forma exata (particionamento por pedido, lock pessimista) ainda não foi definida.
- **Arquivamento de eventos entregues**: eventos já entregues na outbox vão se acumular ao longo do tempo. Mencionamos a necessidade de arquivar ou expurgar esses registros periodicamente, mas isso está fora do escopo de definição desta entrega.

---

### Impacto e riscos

- **Ponto crítico do sistema**: a proposta altera o `changeStatus` do service de orders, um caminho já sensível (mexe em estoque e histórico do pedido). A mitigação é manter a inserção do evento dentro da mesma transação, com rollback automático se qualquer etapa falhar.
- **Novo processo em produção**: o worker precisa rodar separado da API e ser observado; se ele parar, a fila de eventos pendentes na outbox cresce sem que ninguém perceba imediatamente.
- **Superfície de segurança nova**: o sistema passa a expor dados de pedidos para sistemas externos via HTTP. Mitigado com HMAC-SHA256, secret única por endpoint, HTTPS obrigatório e uma revisão de segurança dedicada da Sofia (mínimo 2 dias úteis) antes do deploy.
- **Indisponibilidade prolongada de um cliente**: pode esgotar as tentativas de retry. Mitigado pela dead letter queue e pelo endpoint de replay manual.
- **Entrega duplicada**: a garantia é at-least-once, não exactly-once; a responsabilidade de deduplicar fica com o cliente, usando o identificador único do evento, e isso precisa estar bem documentado no portal do desenvolvedor.
- **Risco comercial**: o prazo estimado (três sprints, incluindo a revisão de segurança) está diretamente ligado à retenção da Atlas Comercial, que sinalizou risco de migração para um concorrente.

---

### Decisões relacionadas

- [ADR-001: Padrão Outbox em MySQL para entrega de webhooks, em vez de fila dedicada](adrs/ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md)
- [ADR-002: Worker Dedicado com Polling Periódico da Outbox, em vez de Listener Reativo](adrs/ADR-002-worker-dedicado-polling-vs-listener-reativo.md)
- [ADR-003: Retry com Backoff Exponencial e Dead Letter Queue para Entrega de Webhooks](adrs/ADR-003-retry-backoff-exponencial-dead-letter-queue.md)
- [ADR-004: Garantia de entrega at-least-once com deduplicação pelo cliente](adrs/ADR-004-garantia-entrega-at-least-once.md)
- [ADR-005: Assinatura HMAC-SHA256 com Secret por Endpoint de Webhook e Rotação com Grace Period](adrs/ADR-005-hmac-secret-por-endpoint-rotacao.md)
- [ADR-006: Snapshot do Payload no Momento da Inserção do Evento](adrs/ADR-006-snapshot-payload-insercao-evento.md)
