# ADR-003: Retry com Backoff Exponencial e Dead Letter Queue para Entrega de Webhooks

**Status:** Proposto
**Data:** 2026-08-13
**Related ADRs:** [ADR-001](./ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md), [ADR-002](./ADR-002-worker-dedicado-polling-vs-listener-reativo.md), [ADR-004](./ADR-004-garantia-entrega-at-least-once.md)

---

## Contexto

O sistema de webhooks outbound entrega notificações de mudança de status de pedido a clientes
B2B de forma assíncrona: um worker dedicado faz polling da outbox e envia chamadas HTTP com
timeout de 10 segundos para o endpoint cadastrado pelo cliente. Quando uma entrega falha — por
timeout, erro de rede ou resposta de falha do cliente —, o sistema precisa de uma estratégia de
recuperação que equilibre a chance de entrega bem-sucedida em caso de indisponibilidade
temporária do cliente com o tempo até uma falha permanente ser sinalizada e ficar visível e
recuperável pela operação de suporte.

Essa decisão foi calibrada por um incidente real relatado em reunião técnica: um cliente já
apresentou indisponibilidade de 2 horas durante uma manutenção planejada, o que descartou
abordagens mais agressivas de número de tentativas por encerrarem o retry antes do cliente
voltar a responder. Ao mesmo tempo, um retry indefinido foi discutido e descartado
implicitamente, por deixar eventos pendurados sem limite de tempo caso o cliente desaparecesse
permanentemente da integração.

Decisão tomada em reunião técnica de 2026-08-13, ainda não implementada em código; a evidência
disponível vem da transcrição da reunião e dos documentos de design (RFC e FDD), não de
histórico de commits ou código-fonte.

## Fatores da Decisão

- Garantir entrega bem-sucedida mesmo diante de indisponibilidades temporárias de clientes, cobrindo o caso real observado de 2 horas de manutenção planejada.
- Evitar retry indefinido, que deixaria eventos pendurados sem limite de tempo caso o cliente não voltasse a responder.
- Manter a leitura e operação da outbox principal limpa, sem misturar eventos ativos com falhas permanentes.
- Prover uma trilha de auditoria e uma superfície dedicada para debug e reprocessamento de falhas permanentes.
- Minimizar a complexidade operacional adicional, mantendo baixa complexidade de implementação.
- Deixar claro para quem dá suporte onde investigar e como reprocessar manualmente uma falha permanente.

## Alternativas Consideradas

- 5 tentativas com backoff exponencial (1m/5m/30m/2h/12h) e falha permanente registrada em tabela `webhook_dead_letter` separada
- 3 tentativas com backoff exponencial e falha permanente registrada em tabela `webhook_dead_letter` separada
- Status "failed" na própria tabela de outbox, sem tabela dedicada de dead letter

## Decisão

Opção escolhida: **5 tentativas com backoff exponencial (1m/5m/30m/2h/12h), com falha
permanente movida para uma tabela dedicada `webhook_dead_letter`**, porque essa janela de
aproximadamente 15 horas cobre com folga o pior caso de indisponibilidade já observado em
produção (2 horas de manutenção planejada) sem deixar eventos pendurados indefinidamente.

A tabela dedicada, por sua vez, mantém a leitura da outbox principal limpa enquanto preserva
payload, motivo da falha e timestamp como evidência para debug, com reprocessamento manual
restrito a um endpoint administrativo de papel ADMIN. Essa combinação foi preferida a um retry
indefinido (que nunca sinalizaria falha permanente) e a um número menor de tentativas (que
teria descartado eventos prematuramente frente ao incidente real já observado).

## Prós e Contras das Opções

### 5 tentativas + tabela `webhook_dead_letter` separada (escolhida)

- Bom, porque cobre com folga o pior caso de indisponibilidade já observado em produção (2 horas)
- Bom, porque mantém a tabela de outbox principal limpa para leitura e operação
- Bom, porque cria uma trilha de auditoria dedicada (payload, motivo, timestamp) para debug e reprocessamento
- Ruim, porque aumenta a superfície de dados a manter (uma tabela adicional)

### 3 tentativas + tabela `webhook_dead_letter` separada

- Bom, porque sinaliza a falha permanente mais rapidamente
- Bom, porque reduz o tempo em que um evento permanece pendente de retentativa
- Ruim, porque é insuficiente frente ao caso real de indisponibilidade de 2 horas, encerrando as tentativas antes do cliente voltar
- Ruim, porque aumenta falsos positivos de falha permanente para indisponibilidades temporárias legítimas

### Status "failed" na própria tabela de outbox

- Bom, porque evita criar e manter uma tabela adicional no schema
- Bom, porque reduz a complexidade de schema do módulo
- Ruim, porque mistura eventos ativos e falhas permanentes na mesma tabela, poluindo a leitura da outbox principal
- Ruim, porque não oferece uma trilha de auditoria específica e isolada para debug e reprocessamento

## Consequências

O tempo até uma falha ser definitivamente sinalizada passa a ser de aproximadamente 15 horas,
o que é aceitável frente ao cenário observado, mas significa que qualquer pessoa dando suporte
a um cliente com problema de recebimento de webhook precisa saber onde investigar
(`webhook_dead_letter`) e como reprocessar manualmente via endpoint administrativo restrito a
papel ADMIN.

Os parâmetros escolhidos — 5 tentativas e os intervalos específicos de backoff — foram
calibrados por um incidente real relatado na reunião, não apenas por convenção teórica.
Mudanças futuras nesses números devem levar esse histórico em conta, e não apenas ajustes
arbitrários feitos sem revisitar o incidente que os motivou.

Como o reprocessamento de eventos em dead letter é manual, a operação passa a depender de um
operador com papel ADMIN disponível para agir quando uma indisponibilidade de cliente se
prolonga além da janela de retry automático, o que é uma dependência operacional a ser
considerada no suporte ao produto.

[NECESSITA INPUT: definir se o reprocessamento manual de eventos em dead letter deve, no
futuro, ganhar algum mecanismo de retry automático adicional]

[NECESSITA INPUT: definir se os intervalos de backoff (1m/5m/30m/2h/12h) devem ser
configuráveis por ambiente ou por cliente, em vez de fixos no código]

## Referências

- `TRANSCRICAO.md`, linhas 90-116 (discussão completa de retry, backoff, número de tentativas e decisão de dead letter em tabela separada)
- `docs/RFC.md`, seção "Proposta técnica" (item de resiliência)
- `docs/RFC.md`, seção "Impacto e riscos" (indisponibilidade prolongada de cliente)
- `docs/FDD.md`, seção 6 ("Erros, exceções e fallback")
- `docs/FDD.md`, seção 10 ("Riscos e mitigação")
