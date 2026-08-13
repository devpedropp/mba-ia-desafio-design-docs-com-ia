# ADR-004: Garantia de entrega at-least-once com deduplicação pelo cliente

**Status:** Aceito
**Data:** 2026-08-13
**Related ADRs:** [ADR-001](./ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md), [ADR-003](./ADR-003-retry-backoff-exponencial-dead-letter-queue.md), [ADR-005](./ADR-005-hmac-secret-por-endpoint-rotacao.md), [ADR-006](./ADR-006-snapshot-payload-insercao-evento.md)

## Contexto

Ao entregar eventos de webhook, o sistema precisa lidar com falhas de rede, timeouts e reentregas decorrentes de retries. Garantir entrega exactly-once exigiria coordenação complexa entre provedor e cliente (por exemplo, confirmação em duas fases), o que aumenta significativamente a complexidade de implementação e operação em ambos os lados. Diante disso, foi necessário decidir qual garantia de entrega o sistema ofereceria aos clientes integrados e como comunicar essa garantia de forma que os clientes pudessem lidar com eventos potencialmente duplicados.

A decisão foi tomada em reunião técnica de 2026-08-13, antes de qualquer implementação em código, e está registrada na transcrição da reunião e nos documentos de design (RFC e FDD) do projeto.

Como a decisão define um contrato público — o comportamento de entrega que qualquer sistema cliente integrado precisará tratar —, ela precisa ser explícita e estável desde antes da primeira implementação, para que a documentação de integração, o portal do desenvolvedor e o próprio design da outbox de eventos sejam construídos já considerando essa garantia.

## Fatores da Decisão

- Simplicidade de implementação e operação do lado do provedor foi priorizada em relação a garantias de entrega mais fortes.
- Necessidade de alinhar o comportamento do sistema a padrões já consolidados no mercado de webhooks (Stripe, GitHub).
- O contrato de entrega afeta diretamente a experiência de integração de todos os clientes externos, exigindo clareza na documentação pública.
- A responsabilidade de deduplicação transferida ao cliente precisa ser sustentável e não gerar atrito recorrente na integração.
- Baixa complexidade do lado do provedor evita a necessidade de protocolos de confirmação em duas fases entre os sistemas.
- Qualquer pessoa desenhando a integração ou o portal do desenvolvedor precisa saber, desde o início, que duplicatas são esperadas.

## Alternativas Consideradas

1. Entrega at-least-once com identificador único de evento (`X-Event-Id`) para deduplicação pelo cliente
2. Entrega exactly-once com coordenação entre provedor e cliente
3. Entrega at-least-once sem identificador único de evento

## Decisão

Opção escolhida: "Entrega at-least-once com identificador único de evento (`X-Event-Id`)", porque oferece simplicidade de implementação e operação do lado do provedor, sem exigir protocolos de confirmação em duas fases, e segue um padrão já validado por provedores de webhook estabelecidos no mercado, como Stripe e GitHub. Cada evento recebe um identificador único gerado no momento em que entra na outbox, permitindo que o cliente identifique e descarte duplicatas recebidas em reentregas.

[NEEDS INPUT: definir se o `X-Event-Id` deve ter mecanismo de expiração ou janela de deduplicação recomendada para o cliente implementar.]

## Prós e Contras das Opções

### Entrega at-least-once com `X-Event-Id` (escolhida)

- Prós: baixa complexidade de implementação e operação do lado do provedor.
- Prós: segue padrão de mercado já validado por provedores como Stripe e GitHub, reduzindo atrito de aprendizado para integradores.
- Prós: fornece ao cliente um meio claro de identificar duplicatas.
- Contras: transfere a responsabilidade de deduplicação para os sistemas dos clientes, exigindo documentação clara e podendo gerar atrito de integração.

### Entrega exactly-once

- Prós: elimina a possibilidade de o cliente receber eventos duplicados.
- Contras: exige coordenação complexa entre provedor e cliente (por exemplo, confirmação em duas fases).
- Contras: aumenta significativamente a complexidade de implementação e operação do provedor.
- Contras: não é o padrão adotado pelos provedores de webhook de mercado consultados como referência.

### At-least-once sem identificador de evento

- Contras: não oferece ao cliente nenhum meio de identificar e descartar duplicatas.
- Contras: rejeitada implicitamente ao se optar pelo uso do `X-Event-Id`; não foi considerada uma alternativa viável.

## Consequências

Todo cliente integrado precisa implementar sua própria lógica de deduplicação usando o `X-Event-Id`, o que deve ser documentado de forma explícita no portal do desenvolvedor e em qualquer material de integração. Essa responsabilidade compartilhada é um contrato público que deve permanecer estável, já que mudanças futuras podem impactar diretamente os clientes já integrados.

Se clientes relatarem problemas recorrentes causados por duplicatas, a decisão poderá precisar ser revisitada. Por ora, a responsabilidade de comunicar essa garantia de forma clara aos clientes foi assumida pela equipe de produto.

[NEEDS INPUT: definir como o time irá medir se a taxa de eventos duplicados está gerando problemas reais para os clientes.]

Esta decisão está diretamente acoplada a duas outras decisões arquiteturais ainda não formalizadas como ADR: o padrão de outbox em MySQL, de onde se origina o evento e o identificador único, e a estratégia de retry com backoff exponencial e dead letter queue, que é a principal fonte de reentregas capazes de gerar duplicatas. Alterações futuras em qualquer uma dessas duas decisões podem afetar diretamente a frequência de duplicatas observada pelos clientes.

## Referências

- `TRANSCRICAO.md:146-158`
- `docs/RFC.md` (seções "Proposta técnica" e "Impacto e riscos")
- `docs/FDD.md` (seção 2 "Objetivos técnicos" e invariantes na seção 6)
