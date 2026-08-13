# ADR-001: Padrão Outbox em MySQL para entrega de webhooks, em vez de fila dedicada

**Status:** Aceito
**Data:** 13-08-2026
**Related ADRs:** [ADR-002](./ADR-002-worker-dedicado-polling-vs-listener-reativo.md), [ADR-003](./ADR-003-retry-backoff-exponencial-dead-letter-queue.md)

---

## Contexto e Declaração do Problema

O sistema precisa notificar clientes B2B externos sobre mudanças de status de pedido através de webhooks outbound. A transação que realiza a mudança de status já é pesada: ela atualiza o pedido, insere um registro de histórico de status e decrementa o estoque dos produtos envolvidos. Chamar o endpoint HTTP do cliente de forma síncrona dentro dessa transação faria com que um cliente lento ou indisponível travasse mudanças de status de outros pedidos, sem que houvesse uma decisão clara sobre executar rollback em caso de falha de rede externa ao sistema.

O time, em reunião técnica realizada em 13-08-2026, decidiu que a entrega de eventos não poderia ser síncrona e avaliou como desacoplar a produção do evento (dentro da transação de mudança de status) do seu consumo (a chamada HTTP ao cliente). A decisão precisava equilibrar atomicidade transacional, simplicidade operacional e o tamanho reduzido do time responsável por manter a solução.

Esta é uma decisão de arquitetura tomada durante a fase de design da feature de webhooks, antes de qualquer implementação em código. A evidência desta decisão está registrada na transcrição da reunião técnica e nos documentos de design (RFC e FDD) já produzidos pelo time. Por se tratar de decisão pré-implementação, não há histórico de commits ou código-fonte associado a ela; toda a evidência disponível vem de documentação de design.

## Motivadores da Decisão

- Garantir atomicidade entre a mudança de status do pedido e o registro do evento de notificação: se a transação principal for confirmada, o evento existe; se sofrer rollback, o evento desaparece junto.
- Evitar que um cliente externo lento ou indisponível bloqueie a transação crítica de mudança de status de outros pedidos.
- Time pequeno, sem capacidade operacional dedicada para provisionar e manter uma nova peça de infraestrutura de mensageria.
- Reaproveitar o MySQL já provisionado no projeto reduz a complexidade de infraestrutura, ainda que exija um processo worker novo.
- Qualquer pessoa que altere a mudança de status de pedido ou integrações externas precisa compreender o padrão adotado.
- Aceitar, de forma consciente, menor throughput e menor capacidade de escala horizontal em troca de simplicidade operacional.

## Opções Consideradas

1. **Padrão outbox em tabela dedicada no MySQL já existente**: o evento de notificação é inserido dentro da mesma transação SQL da mudança de status, e um worker separado lê a tabela e realiza a entrega.

2. **Notificação síncrona dentro da transação de mudança de status**: o endpoint HTTP do cliente é chamado diretamente dentro do método que altera o status, antes do commit da transação.

3. **Fila ou mensageria dedicada (ex.: Redis Streams)**: produção e consumo dos eventos são desacoplados por uma peça de infraestrutura de mensageria separada do banco relacional já usado pelo projeto.

## Resultado da Decisão

Opção escolhida: padrão outbox no MySQL, porque garante atomicidade transacional entre a mudança de status do pedido e o registro do evento sem exigir uma nova peça de infraestrutura, o que é adequado ao tamanho e à capacidade operacional atual do time. O evento é inserido numa tabela de outbox dentro da mesma transação SQL da mudança de status; um worker separado lê essa tabela e realiza as chamadas HTTP aos clientes.

O time reconheceu explicitamente o trade-off aceito: a solução escolhida tem menor throughput e menor capacidade de escala horizontal do que uma fila dedicada. [NECESSITA DE INFORMAÇÃO: não foi definido um volume de eventos específico que justificaria revisitar esta decisão em favor de uma fila dedicada no futuro]

## Prós e Contras das Opções

### Padrão outbox em tabela dedicada no MySQL

- Prós: garante atomicidade entre a mudança de status e o registro do evento (invariante transacional do padrão outbox)
- Prós: reaproveita infraestrutura já provisionada, sem novo componente operacional de mensageria
- Prós: baixa complexidade de infraestrutura, compatível com a capacidade operacional de um time pequeno
- Contras: menor throughput e capacidade de escala horizontal em comparação a uma fila dedicada
- Contras: exige um processo worker novo, rodando separado da API, como dependência operacional adicional
- Contras: [NECESSITA DE INFORMAÇÃO: o comportamento da solução caso o projeto migre de MySQL para outro banco de dados no futuro não foi avaliado]

### Notificação síncrona dentro da transação de mudança de status

- Prós: simplicidade de implementação, sem componente adicional
- Prós: entrega imediata, sem necessidade de worker separado
- Contras: cliente lento ou indisponível trava a mudança de status de outros pedidos
- Contras: sem decisão clara sobre rollback da transação principal em caso de falha de rede externa

### Fila ou mensageria dedicada (ex.: Redis Streams)

- Prós: melhor desacoplamento entre produção e consumo de eventos
- Prós: maior throughput e capacidade de escala horizontal
- Contras: exige subir e operar uma nova peça de infraestrutura
- Contras: considerado overengineering para o estágio atual do time e do produto

## Consequências

A decisão introduz uma nova tabela de outbox e um processo worker dedicado, separado da API, como peças permanentes da arquitetura de comunicação assíncrona com clientes externos. Essa é a decisão-base de toda a feature de webhooks: as demais decisões de design ainda em aberto, como o mecanismo de consumo da outbox pelo worker (polling versus algum mecanismo reativo) e a estratégia de retry e dead letter para falhas de entrega, dependem diretamente desta escolha e devem ser tratadas em ADRs próprias.

Como consequência operacional, qualquer pessoa que trabalhe na mudança de status de pedidos ou em integrações externas precisa entender o padrão outbox e sua relação transacional com o serviço de pedidos, que é um ponto crítico do sistema.

Como consequência futura, o time reconheceu que a solução pode precisar ser revisitada em favor de uma fila dedicada caso o volume de eventos cresça significativamente. [NECESSITA DE INFORMAÇÃO: critério objetivo de volume ou throughput que dispararia essa revisão ainda não foi definido]

## Referências

- `TRANSCRICAO.md:30-58`
- `docs/RFC.md:41-53` (seção "Alternativas consideradas")
- `docs/FDD.md:9-16` (seção "Contexto e motivação técnica")
- `docs/FDD.md:37-63` (seção "Escopo e exclusões")
- `docs/adrs/potential-adrs/done/WEBHOOKS/outbox-pattern-mysql-vs-fila-dedicada.md` (potential ADR de origem)
