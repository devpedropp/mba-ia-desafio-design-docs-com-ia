# ADR-005: Assinatura HMAC-SHA256 com Secret por Endpoint de Webhook e Rotação com Grace Period

**Status:** Aceito
**Data:** 2026-08-13
**Related ADRs:** [ADR-004](./ADR-004-garantia-entrega-at-least-once.md)

## Contexto e Problema

Toda chamada de webhook enviada ao cliente precisa ser autenticável, de forma que o cliente consiga verificar que a requisição realmente partiu da plataforma e não foi forjada por terceiros. Esse mecanismo de assinatura é a base de confiança de toda a integração externa: qualquer sistema de terceiros que consome webhooks depende dele para decidir se aceita ou rejeita uma notificação recebida. A decisão define que cada requisição de webhook é assinada com HMAC-SHA256 sobre o corpo da requisição, enviada no header `X-Signature`, usando uma secret específica daquele endpoint de webhook (não uma secret global compartilhada entre todos os clientes).

A secret é rotacionável: o cliente pode solicitar uma nova secret pela API, e a secret anterior permanece válida por 24 horas em paralelo à nova, dando tempo para o cliente migrar seus sistemas antes que a secret antiga deixe de ser aceita. Esse período de convivência entre as duas secrets (grace period) foi desenhado especificamente para evitar que a rotação de secret vire um evento de downtime para o cliente, já que ele precisa atualizar seus próprios sistemas de validação antes que a secret antiga expire.

A alternativa de usar uma secret global da plataforma foi explicitamente descartada na reunião de definição técnica, reforçada por um incidente relatado pelo próprio time: um cliente já vazou uma secret em log de aplicação no passado, o que evidenciou o risco concreto de um vazamento único comprometer múltiplos clientes ao mesmo tempo caso a secret fosse compartilhada entre todos os endpoints da plataforma.

## Fatores da Decisão

- Define o modelo de confiança de toda a integração externa; qualquer sistema de terceiros que valide webhooks depende diretamente desse esquema.
- Precisa isolar o raio de impacto de um vazamento de secret a um único cliente/endpoint, em vez de expor todos os clientes simultaneamente.
- Precisa permitir que o cliente migre para uma nova secret sem downtime na validação de webhooks.
- Já existe precedente real de vazamento de secret em log de aplicação de um cliente, reforçando a necessidade de isolamento por endpoint.
- A rotação com duas secrets simultaneamente válidas adiciona complexidade operacional que precisa ser aceita em troca da migração sem interrupção.

## Opções Consideradas

- Secret específica por endpoint de webhook, com rotação e grace period de 24 horas
- Secret global da plataforma, compartilhada entre todos os endpoints
- Rotação imediata, invalidando a secret antiga no momento da troca (sem grace period)

## Decisão

Opção escolhida: **secret específica por endpoint de webhook, com rotação e grace period de 24 horas**, porque isola o raio de impacto de um eventual vazamento a um único cliente/endpoint, em vez de comprometer toda a base de clientes de uma vez — risco considerado inaceitável à luz do incidente de vazamento já relatado. O grace period de 24 horas foi adotado especificamente para evitar downtime na validação de webhooks do cliente durante a migração para a nova secret, aceitando em troca a complexidade adicional de manter duas secrets válidas simultaneamente por endpoint durante essa janela.

[NEEDS INPUT: O mecanismo de armazenamento seguro das secrets — hashing, criptografia em repouso, ou ambos — não foi detalhado na reunião e precisa ser definido antes da implementação.]

## Prós e Contras das Opções

### Secret por endpoint com rotação e grace period de 24h (escolhida)

- Prós: isola o raio de impacto de um vazamento a um único cliente/endpoint
- Prós: permite migração de secret sem downtime para o cliente
- Prós: resposta direta a um incidente real já ocorrido (vazamento em log de aplicação)
- Contras: aumenta a superfície de dados sensíveis a armazenar (uma secret por endpoint cadastrado, em vez de uma única global)
- Contras: adiciona complexidade operacional ao exigir aceitar duas secrets válidas simultaneamente durante o grace period

### Secret global da plataforma (descartada)

- Prós: mais simples de implementar e operar (uma única secret para gerenciar)
- Prós: menor overhead de armazenamento e ciclo de vida
- Contras: um único vazamento compromete todos os clientes simultaneamente
- Contras: nenhum isolamento do impacto de uma secret vazada

### Rotação imediata sem grace period (rejeitada implicitamente)

- Prós: fluxo de rotação mais simples (apenas uma secret válida a qualquer momento)
- Contras: força o cliente a atualizar seus sistemas instantaneamente, sob risco de falha na validação de webhooks durante a transição
- Contras: foi implicitamente descartada ao se optar pelo grace period de 24 horas, justamente para evitar esse risco

## Consequências

Qualquer pessoa que altere o fluxo de cadastro ou rotação de webhook precisa entender esse modelo de secret por endpoint antes de modificá-lo, já que ele é a base do modelo de confiança de toda a integração externa. A necessidade de armazenar e gerenciar o ciclo de vida de duas secrets por endpoint durante o grace period de 24 horas passa a ser uma responsabilidade operacional permanente do sistema de webhooks, e não uma exceção pontual do fluxo de rotação.

Esta decisão está marcada para revisão de segurança dedicada, com no mínimo 2 dias úteis reservados, antes do lançamento em produção — reconhecida na própria reunião como necessária dado que a feature ainda não foi implementada e que o esquema de assinatura é uma superfície de segurança nova para a plataforma.

[NEEDS INPUT: Resultado da revisão de segurança dedicada (mínimo 2 dias úteis) ainda não realizada; pode alterar detalhes desta decisão antes do lançamento em produção.]

[NEEDS INPUT: Se o grace period de 24 horas deve ser configurável por cliente ou permanece fixo para todos — não foi definido na reunião.]

Esta decisão se relaciona com a definição de garantia de entrega at-least-once com deduplicação pelo lado do cliente, outro contrato de segurança/confiabilidade da entrega de webhooks ainda em fase de potencial ADR.

## Referências

- `TRANSCRICAO.md:118-144` — discussão de segurança: HMAC-SHA256, secret por endpoint, rotação, TLS obrigatório, limite de payload
- `docs/RFC.md` — seções "Proposta técnica" e "Impacto e riscos"
- `docs/FDD.md` — seção 5 ("Contratos públicos") e seção 10 ("Riscos e mitigação")
