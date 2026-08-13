# ADR-002: Worker Dedicado com Polling Periódico da Outbox, em vez de Listener Reativo

**Status:** Aceita
**Data:** 13-08-2026
**Related ADRs:** [ADR-001](./ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md), [ADR-003](./ADR-003-retry-backoff-exponencial-dead-letter-queue.md)

## Contexto

A feature de webhooks de notificação de pedidos usa o padrão outbox: dentro da mesma transação que muda o status de um pedido, um evento é gravado na tabela `webhook_outbox`. É necessário um mecanismo separado para consumir esses eventos pendentes e realizar as entregas HTTP aos clientes, sem acoplar esse processamento ao ciclo de vida da API.

A questão central é como esse consumo é disparado: de forma reativa, sendo notificado assim que um evento é gravado, ou de forma periódica, verificando a tabela em intervalos regulares. O banco de dados do projeto é MySQL, que não possui um mecanismo nativo de notificação equivalente ao NOTIFY/LISTEN do PostgreSQL, o que restringe as opções reativas disponíveis sem introduzir soluções improvisadas ou infraestrutura adicional.

O requisito de negócio aceito para esta entrega é entregar o evento ao cliente em até 10 segundos, tratado pelos clientes como "tempo real".

## Fatores de Decisão

- Requisito de latência de entrega em até 10 segundos, aceito como suficiente pelos clientes.
- MySQL não possui mecanismo nativo de notificação equivalente ao NOTIFY/LISTEN do PostgreSQL.
- Time pequeno, com preferência por evitar infraestrutura adicional e soluções improvisadas de notificação.
- Necessidade de o worker sobreviver a reinícios da API, exigindo isolamento entre os dois processos.
- Preservação da garantia de ordering de eventos por pedido, hoje dependente de um único worker processando a outbox.

## Alternativas Consideradas

- Worker dedicado, em processo separado da API, com polling periódico da outbox a cada 2 segundos.
- Worker reativo via trigger/listener de banco, equivalente ao NOTIFY/LISTEN do PostgreSQL.
- Processamento embutido no mesmo processo da API, sem um worker dedicado.

## Decisão

Opção escolhida: worker dedicado em processo separado (`src/worker.ts`), com sua própria instância de `PrismaClient` sobre o mesmo banco, fazendo polling da outbox a cada 2 segundos e processando os eventos pendentes mais antigos em lote pequeno.

O mecanismo reativo via trigger/listener foi descartado porque o MySQL não oferece um equivalente nativo ao NOTIFY/LISTEN do PostgreSQL; notificar um processo externo a partir de um trigger exigiria soluções improvisadas (escrita em arquivo, chamada a um endpoint), consideradas inadequadas para o projeto. O processamento embutido no processo da API foi descartado porque o worker seria derrubado a cada reinício da API, comprometendo a confiabilidade da entrega. O intervalo de polling de 2 segundos atende com folga o requisito de latência de até 10 segundos.

## Prós e Contras das Opções

### Worker dedicado com polling periódico (escolhida)

- Boa, porque é simples de implementar e não depende de recursos específicos do banco.
- Boa, porque é portável entre bancos relacionais, sem depender de extensões do MySQL.
- Boa, porque isola o worker de reinícios da API.
- Ruim, porque introduz uma latência mínima inerente ao intervalo de polling.
- Ruim, porque consome ciclos de consulta ao banco de forma constante, mesmo sem eventos pendentes.

### Worker reativo via trigger/listener de banco

- Boa, porque poderia notificar o worker de novos eventos de forma praticamente imediata.
- Boa, porque reduziria consultas ao banco quando não há eventos pendentes.
- Ruim, porque o MySQL não possui mecanismo nativo equivalente ao NOTIFY/LISTEN do PostgreSQL.
- Ruim, porque notificar um processo externo a partir de um trigger exigiria soluções improvisadas, consideradas inadequadas.

### Processamento embutido no processo da API

- Boa, porque elimina a necessidade de operar um processo adicional.
- Ruim, porque o processamento seria interrompido a cada reinício da API.
- Ruim, porque acopla o ciclo de vida do processamento de webhooks ao ciclo de vida da API.

## Consequências

Um processo adicional (`npm run worker`) passa a rodar em produção, separado da API, e precisa ser operado e observado de forma independente. Quem for depurar problemas de entrega de webhook precisa saber que existe um worker fazendo polling, e não um mecanismo push.

A garantia de ordering de eventos por pedido depende de a arquitetura permanecer single-worker, processando a outbox em ordem de criação; isso está documentado como limitação conhecida, não como garantia permanente. [NECESSITA INPUT: em que ponto o intervalo de polling de 2 segundos deixaria de ser suficiente, exigindo revisão da abordagem?]

Se for necessário escalar para múltiplos workers no futuro, a garantia atual de ordering por pedido deixa de valer sem uma estratégia adicional. [NECESSITA INPUT: qual estratégia (particionamento por pedido, lock pessimista) será adotada caso seja necessário escalar para múltiplos workers?]

## Referências

- `TRANSCRICAO.md:58-88`
- `docs/RFC.md:51-53`
- `docs/RFC.md:61`
- `docs/FDD.md:62-71`
- `docs/FDD.md:277-283`
