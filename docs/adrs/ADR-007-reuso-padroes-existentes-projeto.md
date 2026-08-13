# ADR-007: Reuso dos Padrões Existentes do Projeto no Módulo de Webhooks

**Status:** Aceito
**Data:** 13-08-2026
**Related ADRs:** [ADR-001](./ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md)

---

## Contexto

O módulo de webhooks é uma feature nova dentro de uma base de código já estabelecida, com convenções claras de organização de módulo, tratamento de erro e logging. O time decidiu, em reunião técnica, que o módulo de webhooks não deveria introduzir nenhuma convenção nova nesses três aspectos, e sim reaproveitar integralmente o que já existe no restante do projeto.

Concretamente, o padrão de módulo já usado no projeto organiza cada domínio em `src/modules/{dominio}` com `controller`, `service`, `repository`, `routes` e `schemas` separados — visível hoje no módulo de pedidos (`src/modules/orders/`, com `order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts` e `order.schemas.ts`). O tratamento de erro já segue uma classe base `AppError` (`src/shared/errors/app-error.ts`), com subclasses específicas por domínio que carregam um código de erro em maiúsculas (por exemplo `InsufficientStockError` e `InvalidStatusTransitionError`, ambas em `src/shared/errors/http-errors.ts`, com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`). O middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata qualquer `AppError`, além de erros do Zod e do Prisma, formatando a resposta HTTP de forma padronizada. O logging estruturado já é feito via Pino, através de uma instância única compartilhada (`src/shared/logger/index.ts`).

## Fatores da Decisão

- Consistência de manutenção: um desenvolvedor que já conhece o padrão de módulos do projeto deve conseguir navegar o módulo de webhooks sem aprender uma convenção nova.
- O middleware de erro centralizado já cobre `AppError`, Zod e Prisma; criar um mecanismo de erro paralelo duplicaria lógica já existente e testada.
- O logger Pino já está configurado com redação de campos sensíveis; instanciar um logger novo abriria mão dessa configuração sem necessidade.
- Baixo custo de adoção: seguir o padrão existente não exige nenhuma decisão de design adicional para essas três dimensões (estrutura de módulo, erros, logging).
- Time pequeno, com preferência explícita por minimizar superfícies de código a manter.

## Alternativas Consideradas

1. **Reaproveitar integralmente os padrões existentes** (estrutura de módulo, `AppError`, códigos de erro prefixados, middleware de erro centralizado, logger Pino compartilhado) — escolhida.
2. **Criar convenções próprias para o módulo de webhooks** (por exemplo, um formato de erro específico ou um logger dedicado), sob o argumento de que webhooks têm necessidades diferentes de auditoria e observabilidade.

## Decisão

Opção escolhida: reuso máximo dos padrões já existentes no projeto. O módulo de webhooks segue a mesma estrutura de módulo (`controller`, `service`, `repository`, `routes`, `schemas`) já usada por `orders`, `products`, `customers` e `users`; os erros do módulo estendem `AppError` com códigos prefixados por `WEBHOOK_` (por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo exatamente o mesmo padrão de `InsufficientStockError`/`InvalidStatusTransitionError`; o middleware de erro centralizado não precisa de nenhuma alteração; e tanto o módulo quanto o worker separado reaproveitam a instância compartilhada do logger Pino, sem introduzir uma ferramenta de logging nova.

A alternativa de criar convenções próprias foi descartada porque nenhuma necessidade específica de webhooks (auditoria do replay de dead letter, por exemplo) exige um mecanismo de erro ou logging diferente do que já existe — a auditoria do replay é resolvida com um log estruturado comum via Pino, não com uma ferramenta dedicada.

## Prós e Contras das Opções

### Reaproveitar padrões existentes (escolhida)

- Prós: nenhuma curva de aprendizado adicional para quem já conhece a base de código
- Prós: reaproveita o middleware de erro centralizado já testado, sem duplicar lógica de formatação de resposta
- Prós: preserva a configuração de redação de dados sensíveis já existente no logger
- Contras: qualquer limitação do padrão atual (por exemplo, os campos suportados por `AppError`) também se aplica ao módulo de webhooks

### Convenções próprias para o módulo de webhooks

- Prós: liberdade para desenhar um formato de erro ou logging otimizado especificamente para observabilidade de entregas de webhook
- Contras: duplica lógica já existente e testada no middleware de erro centralizado
- Contras: aumenta a superfície de manutenção e a curva de aprendizado para quem já conhece o resto do projeto
- Contras: nenhuma necessidade concreta foi identificada na reunião que justificasse essa duplicação

## Consequências

O módulo de webhooks (`src/modules/webhooks/`, ainda não criado) deve seguir, desde o primeiro commit, a mesma estrutura interna de `src/modules/orders/`. Qualquer erro lançado pelo módulo deve estender `AppError` e usar um código prefixado por `WEBHOOK_`, e nenhuma alteração é necessária em `src/middlewares/error.middleware.ts` para que esses erros sejam tratados corretamente. O worker (`src/worker.ts`, ainda não criado) deve importar o logger compartilhado de `src/shared/logger/index.ts` em vez de instanciar um novo, e deve seguir o mesmo padrão de configuração de conexão já usado em `src/config/database.ts`, mas com uma instância própria de `PrismaClient` por rodar em processo separado da API.

Como consequência de manutenção, revisões de código do módulo de webhooks podem ser feitas por qualquer pessoa já familiarizada com os módulos existentes do projeto, sem necessidade de documentação adicional sobre convenções específicas de erro ou logging.

## Referências

- `TRANSCRICAO.md`, linhas 160-180 (padrão de módulo, classes de erro, prefixo `WEBHOOK_`)
- `TRANSCRICAO.md`, linhas 174-180 (reuso de Pino e do middleware de erro centralizado)
- `TRANSCRICAO.md`, linha 180 (decisão consolidada: "reuso máximo do que já existe")
- `src/modules/orders/order.service.ts`, `order.controller.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts` — padrão de módulo de referência
- `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — classe base de erro e exemplos de subclasses com código prefixado
- `src/middlewares/error.middleware.ts` — middleware de erro centralizado a ser reaproveitado sem alteração
- `src/shared/logger/index.ts` — instância compartilhada do logger Pino
