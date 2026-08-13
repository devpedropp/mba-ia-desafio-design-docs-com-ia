# ADR-006: Snapshot do Payload no Momento da Inserção do Evento

**Status:** Aceita
**Data:** 13-08-2026
**Related ADRs:** [ADR-001](./ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md), [ADR-004](./ADR-004-garantia-entrega-at-least-once.md)

## Contexto e Declaração do Problema

Cada mudança de status de um pedido gera um evento de webhook, que é primeiro persistido na `webhook_outbox` (dentro da mesma transação da mudança de status) e só depois entregue de forma assíncrona por um worker dedicado. Como a entrega usa retry com backoff exponencial (1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, cobrindo uma janela de até aproximadamente 15 horas), pode haver um intervalo considerável entre o momento em que o evento é criado e o momento em que ele efetivamente chega ao cliente, seja por instabilidade do endpoint do cliente, seja por indisponibilidade temporária de rede.

Isso levanta a pergunta de quando o payload do evento (os dados do pedido enviados ao cliente) deve ser montado: no instante da inserção do evento na outbox, capturando um retrato ("snapshot") do pedido naquele momento, ou apenas no instante do envio pelo worker, buscando o estado mais atual do pedido na hora da entrega. A resposta afeta diretamente o que o cliente do webhook pode assumir sobre o conteúdo de cada notificação recebida, especialmente em cenários de retry, onde o pedido pode ter avançado para outros status entre a criação do evento e sua entrega efetiva.

A decisão foi tomada ao final de uma reunião técnica de definição da feature de webhooks, como dúvida de última hora levantada por um engenheiro sobre a modelagem do evento na outbox, e confirmada rapidamente pela tech lead e por outro engenheiro presente, sem levantar objeções. Por se tratar de uma feature ainda não implementada, não há código-fonte associado; a evidência desta decisão vem da transcrição da reunião e do documento de design funcional (FDD) da feature.

## Direcionadores da Decisão

- O cliente deve sempre receber o estado do pedido exatamente como ele era no momento da mudança de status que gerou aquele evento, não um estado potencialmente diferente capturado depois.
- Retries podem atrasar a entrega em minutos ou horas; o pedido pode mudar de estado novamente nesse intervalo, o que criaria payloads inconsistentes com o evento original se renderizados tardiamente.
- A imutabilidade do payload após a inserção sustenta a garantia de idempotência do cliente via `X-Event-Id`: reentregas do mesmo evento devem carregar sempre o mesmo conteúdo.
- Baixa complexidade de implementação: o pedido já está carregado na mesma transação em que o evento é inserido, então a renderização acontece uma única vez, nesse ponto.
- Custo de manter dados potencialmente redundantes ou desatualizados em relação ao estado atual do pedido, armazenados dentro da própria outbox.

## Opções Consideradas

1. Snapshot do payload renderizado no momento da inserção do evento na outbox (escolhida)
2. Armazenar apenas o `order_id` na outbox e renderizar o payload completo no momento do envio pelo worker

## Decisão

Opção escolhida: "Snapshot do payload na inserção", porque garante que o conteúdo entregue ao cliente reflita fielmente o estado do pedido no instante exato da mudança de status que originou o evento, independentemente de quanto tempo o worker leve para efetivar a entrega (por causa de retries). Renderizar o payload apenas no envio introduziria uma janela de inconsistência caso o pedido mudasse de estado entre a inserção do evento e sua entrega, comportamento considerado indesejável pelo time. A decisão também tem o benefício colateral de reforçar a semântica de reentrega idêntica exigida pela garantia de entrega at-least-once com deduplicação via `X-Event-Id`.

Essa escolha foi favorecida também por sua baixa complexidade de implementação: como o pedido já precisa estar carregado na mesma transação em que o evento é inserido na outbox (para a mudança de status), a renderização do payload pode reaproveitar esses dados sem consultas adicionais. A decisão foi formalizada como invariante do sistema no documento de design funcional da feature, reforçando que o payload de um evento é imutável a partir de sua inserção.

## Prós e Contras das Opções

### Snapshot do payload na inserção (escolhida)

- Prós: garante consistência semântica entre o evento e o estado do pedido que o originou, mesmo com atraso na entrega
- Prós: payload imutável após a inserção, o que sustenta a deduplicação por `X-Event-Id` em reentregas
- Prós: implementação simples, já que o pedido está carregado na mesma transação da inserção
- Contras: mantém dados potencialmente redundantes ou desatualizados em relação ao estado atual do pedido dentro da outbox

### Renderização tardia no momento do envio

- Prós: evitaria armazenar dados redundantes na outbox, guardando apenas a referência (`order_id`)
- Contras: se o pedido mudasse de estado entre a inserção do evento e o envio, o payload entregue refletiria um estado diferente daquele que gerou a notificação original
- Contras: comportamento considerado inconsistente e de difícil depuração pelo time
- Contras: enfraquece a garantia de idempotência, já que reentregas do mesmo `X-Event-Id` poderiam retornar conteúdos diferentes entre tentativas

## Consequências

O payload de cada evento passa a ser imutável a partir do momento em que é inserido na outbox: qualquer mudança posterior no pedido não é refletida em eventos já criados, apenas em eventos futuros. Essa imutabilidade é o que permite ao cliente tratar reentregas do mesmo `X-Event-Id` como idênticas em conteúdo, sustentando a semântica de entrega at-least-once com deduplicação, e é consistente com a invariante de que, se a transação de mudança de status for revertida, o evento correspondente também não existe.

Como efeito colateral, quem for depurar uma notificação com dados aparentemente desatualizados em relação ao pedido atual precisa entender que esse comportamento é esperado por design — o payload reflete o momento do evento, não o estado corrente do pedido. Isso deve ser documentado para o time de suporte e desenvolvimento, para evitar que divergências entre o payload entregue e o estado atual do pedido sejam tratadas como bug.

A outbox retém, portanto, dados potencialmente redundantes em relação ao estado corrente do pedido enquanto o evento permanecer armazenado (incluindo em caso de falha definitiva, quando o evento é movido para `webhook_dead_letter` junto com seu payload original). Esse custo de armazenamento foi considerado aceitável dado o limite de 64KB por payload de evento definido para a feature.

[NEEDS INPUT: Existe algum campo do pedido que deveria refletir o estado mais atual no momento do envio, mesmo com o restante do payload em snapshot (por exemplo, dados de rastreio ou contato que mudam com frequência)? Não foi discutido na reunião.]

## Referências

- `TRANSCRICAO.md` (linhas 308-314) — discussão que originou a decisão de snapshot na inserção
- `docs/FDD.md` (seção 6, "Erros, exceções e fallback" — Invariantes) — registro formal da invariante de imutabilidade do payload após a inserção
