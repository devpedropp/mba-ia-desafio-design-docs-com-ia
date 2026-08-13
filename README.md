# Da Reunião ao Documento: Design Docs Gerados por IA

> O enunciado original do desafio está preservado em [`README-ENUNCIADO.md`](./README-ENUNCIADO.md). Este README documenta o meu processo de produção.

## Sobre o desafio

O desafio parte de uma situação bem comum: uma decisão técnica inteira (arquitetura, trade-offs, segurança, prazos) foi fechada numa reunião de ~55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, e a única coisa que sobrou disso foi a transcrição bruta da call. A tarefa é transformar essa transcrição, cruzada com o código existente do Order Management System, num pacote de documentação de design (PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade) que seja bom o suficiente para o time de engenharia começar a implementar a feature de Webhooks de Notificação de Pedidos sem precisar reouvir a call.

A regra que guiou todo o trabalho foi a rastreabilidade: nada podia entrar em um documento sem uma origem identificável, seja um trecho literal da transcrição, seja um arquivo real do código-fonte. Usei a IA (Claude Code) como ferramenta principal de produção, mas o papel de decidir o que entra, o que fica de fora, e de auditar cada afirmação contra a fonte foi meu, inclusive quando isso significou jogar fora e reescrever um documento inteiro depois de perceber que ele tinha conteúdo inventado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code (Sonnet 5)** | Agente principal de produção. Leu `TRANSCRICAO.md` e o código-fonte, gerou PRD, FDD, RFC e o Tracker, e conduziu toda a auditoria de rastreabilidade (cruzar cada afirmação com a transcrição ou o código antes de aceitá-la). |
| **Plugin `adrs-management` (marketplace `fullcycle-claude-marketplace`)** | Conjunto de subagentes especializados (`adr-generator`, entre outros) usado para gerar 6 dos 7 ADRs no formato MADR a partir de "potential ADRs" com a evidência da reunião, já que o fluxo padrão do plugin espera minerar decisões do histórico de git de um código já existente — o que não se aplica aqui, pois a feature ainda não foi implementada. O 7º ADR (reuso de padrões do projeto) foi escrito diretamente no mesmo formato, numa correção posterior. |
| **Prompts do curso** (`prompt-prd-feature.md`, `prompt-fdd.md`) | Usados como esqueleto de formato e processo de entrevista para PRD e FDD, adaptados para rodar em modo "batch" sobre a transcrição em vez de entrevista interativa turno a turno. |

## Workflow adotado

Não segui a ordem sugerida no enunciado (ADRs → RFC → FDD → PRD). Comecei pelo documento mais alto nível e fui descendo, porque foi assim que a conversa naturalmente evoluiu comigo pedindo um documento de cada vez e revisando o resultado antes de pedir o próximo:

1. **Exploração de contexto**: pedi para a IA ler `TRANSCRICAO.md` por completo e mapear a estrutura do código existente (módulos, padrões de erro, middleware de auth, logger) antes de gerar qualquer documento.
2. **PRD** (`docs/PRD.md`): gerado a partir do prompt de entrevista do curso, rodado em lote sobre a transcrição (sem fazer as perguntas uma a uma, já que as respostas já estavam todas na call).
3. **FDD** (`docs/FDD.md`): gerado a partir do prompt de FDD do curso, cruzando transcrição + PRD + código. Passou por uma correção grande (ver seção de iterações).
4. **RFC** (`docs/RFC.md`): escrito à mão pelo Claude Code seguindo a estrutura pedida no requisito 2 do enunciado (não havia um prompt de curso específico para RFC), condensando as decisões já registradas em PRD/FDD para o nível de arquitetura.
5. **ADRs** (`docs/adrs/ADR-001..007`): as 4 decisões citadas no RFC, mais 2 adicionais (HMAC/rotação de secret e snapshot do payload), geradas via o plugin `adrs-management`, com um passo manual de renumeração e cross-linking depois. Um 7º ADR (reuso de padrões existentes do projeto) foi adicionado numa correção posterior, ao validar a entrega contra os critérios de aceite.
6. **Tracker** (`docs/TRACKER.md`): mantido de forma incremental, junto de cada documento — sempre que um documento era gerado ou corrigido, o Tracker era atualizado na mesma resposta, nunca como um passo separado no fim.
7. **README** (este arquivo): escrito por último, depois que todo o pacote já estava fechado.

A interação com a IA foi sempre em turnos curtos e dirigidos: um pedido específico por vez ("gere o FDD", "exclua do FDD o que não tem lastro na transcrição", "gere as ADRs e mais 2 adicionais"), nunca um pedido genérico de "gere toda a documentação". Isso deu espaço para revisar cada entrega antes de avançar para a próxima.

## Prompts customizados

**1. Prompt inicial, combinando o prompt de PRD do curso com a exigência específica do Tracker deste desafio** (adaptação do material do curso ao formato de rastreabilidade pedido no enunciado):

```text
leia TRANSCRICAO.md e gere um arquivo PRD seguindo ../prompts/prompt-prd-feature.md,
para todo link, menção ou itens usados nos documentos gerados faça a anotação no
arquivo docs/TRACKER.md, no seguinte formato: Produza o arquivo docs/TRACKER.md, uma
tabela markdown que mapeia cada item registrado nos seus documentos à origem na
transcrição ou no código.

Formato obrigatório da tabela:
ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização

Onde:
- ID: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
- Documento: arquivo onde o item aparece
- Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off...
- Conteúdo (resumo): descrição de uma linha do item
- Fonte: TRANSCRICAO ou CODIGO
- Localização: para TRANSCRICAO, timestamp + falante (ex: [09:17] Diego).
  Para CODIGO, caminho do arquivo.
```

**2. Prompt para geração do RFC, aplicado depois de revisar o primeiro FDD gerado** (curto, porém foi possível devido ao contexto da transcrição e uso do FDD gerado):

```text
Produza o arquivo docs/RFC.md com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão. O RFC opera em nível de arquitetura: apresente a abordagem escolhida, as alternativas que foram debatidas e as questões deixadas em aberto. Faça um documento entre 2 e 4 páginas. O detalhamento de implementação fica no docs/FDD.md. Inclua os seguintes pontos:

  Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
  Resumo executivo (TL;DR) da proposta
  Contexto e problema
  Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
  Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
  Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
  Impacto e riscos
  Decisões relacionadas (links para os ADRs correspondentes)
  O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD. E atualize docs/TRACKER.md
```


## Iterações e ajustes

O processo levou **7 ciclos principais** de geração/correção até fechar o pacote:

1. **PRD — 1 ciclo, sem correção relevante.** Saiu bem alinhado à transcrição já na primeira geração, provavelmente porque o prompt de entrevista do curso já força a captura estruturada de objetivo, escopo, riscos etc.

2. **FDD — 2 ciclos, correção grande.** A primeira versão do FDD tinha vários problemas típicos de alucinação sutil que só apareceram numa auditoria linha a linha contra a transcrição:
   - Inventou um prefixo de rota `/api/v1` (existe no código, mas nunca foi dito na reunião).
   - Inventou números de versão de dependências (Node ≥20, Prisma 5.22.0 etc.) puxados do `package.json`, não da call.
   - Atribuiu códigos HTTP numéricos (200/201/400/403/404) a cada endpoint — nenhum código HTTP foi citado literalmente na reunião, só comportamentos qualitativos ("recusamos com erro de validação").
   - Criou um mecanismo de dupla assinatura HMAC durante a rotação de secret, plausível tecnicamente mas nunca discutido.
   - Criou um risco inteiro ("worker travado sem reinício automático") com mitigações inventadas.
   
   A correção foi reescrever o FDD inteiro mantendo só o que tinha citação direta ou paráfrase fiel na transcrição, e ajustei também o `docs/TRACKER.md` para remover todas as linhas `Fonte = CODIGO` desse documento (o item [Hipótese] deixou de existir; ou tinha origem rastreável, ou saía do documento).

3. **RFC — 1 ciclo, sem correção relevante.** Escrito já em cima do PRD/FDD corrigidos, então herdou o rigor de rastreabilidade que eu já tinha estabelecido.

4. **ADRs — 2 ciclos.** No primeiro ciclo, os 6 ADRs foram gerados corretamente em conteúdo, mas saíram com numeração placeholder (`ADR-XXX`) e organizados na estrutura padrão do plugin (`docs/adrs/generated/WEBHOOKS/...`), que não batia com a convenção já definida em `docs/adrs/README.md` para este projeto. Precisei renumerar manualmente, adicionar os links cruzados de `Related ADRs` entre as decisões relacionadas, achatar a estrutura de pastas e apagar o andaime intermediário do pipeline (`generated/`, `potential-adrs/`).

5. **Convenção de nome dos ADRs — 1 ciclo.** Depois de entregues, pedi a troca do padrão de nome de `000N-titulo.md` para `ADR-NNN-titulo-em-kebab-case.md`, o que exigiu renomear os 6 arquivos e atualizar todos os pontos que referenciavam o nome antigo: os próprios links internos entre ADRs, a seção "Decisões relacionadas" do RFC, os caminhos no Tracker e o exemplo de convenção no `docs/adrs/README.md`.

6. **Reintrodução de conteúdo de código no FDD — 1 ciclo, tensão entre dois requisitos.** Ao escrever este README, reli o enunciado original com atenção e percebi que a correção do item 2 tinha ido longe demais: o enunciado exige explicitamente uma seção "Integração com o sistema existente" no FDD, com pelo menos 4 caminhos reais de código, e status codes nos contratos — e eu tinha removido as duas coisas do FDD ao aplicar a regra "só o que está na transcrição" de forma literal demais. A correção foi reintroduzir esse conteúdo, mas deixando explícito no próprio texto do FDD e no Tracker que ele tem `Fonte = CODIGO` (ou é convenção REST do projeto), em vez de misturá-lo com o que veio da reunião. Esse foi o momento mais claro de que "não alucinar" e "atender todos os critérios do enunciado" às vezes puxam em direções opostas, e que a solução é marcar a origem de cada afirmação com precisão, não simplesmente cortar tudo que é incerto.

7. **Auditoria final contra os critérios de aceite — 1 ciclo.** Pedi para validar a entrega inteira, item por item, contra a checklist de critérios de aceite do enunciado. Isso revelou 3 gaps concretos que a correção do item 6 não tinha coberto por completo: (a) a seção "Contratos públicos" do FDD tinha só 1 endpoint com exemplo de payload, e o critério exige pelo menos 4 com request **e** response; (b) nenhum dos 6 ADRs referenciava um arquivo real e já existente do código (o único arquivo citado, `src/worker.ts`, ainda não existe); (c) uma linha do Tracker tinha `Fonte = "TRANSCRICAO; CODIGO"`, um valor combinado fora do enum estrito de duas opções. A correção foi: adicionar exemplos de request/response em 4 contratos do FDD; escrever o `ADR-007` (reuso dos padrões existentes do projeto), que cobre a 6ª decisão do enunciado que ainda faltava e referencia arquivos reais como `app-error.ts` e o `error.middleware.ts`; corrigir a linha do Tracker; e padronizar os cabeçalhos de seção dos 6 ADRs anteriores para os nomes literais do critério (`Alternativas Consideradas`, `Decisão`).

Em todos os casos, o `docs/TRACKER.md` foi o instrumento que expôs os problemas: qualquer linha para a qual eu não conseguia apontar um timestamp real da transcrição ou um caminho real de arquivo era sinal de conteúdo inventado.

## Como navegar a entrega

Ordem sugerida de leitura:

1. **[`README.md`](./README.md)** — este arquivo, processo de produção.
2. **[`docs/PRD.md`](./docs/PRD.md)** — por que a feature existe, para quem, e o que ela precisa fazer.
3. **[`docs/RFC.md`](./docs/RFC.md)** — a proposta técnica de solução, alternativas descartadas e questões em aberto.
4. **[`docs/adrs/`](./docs/adrs/)** — as 7 decisões arquiteturais fechadas, cada uma em um arquivo curto:
   - [`ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md`](./docs/adrs/ADR-001-outbox-pattern-mysql-vs-fila-dedicada.md)
   - [`ADR-002-worker-dedicado-polling-vs-listener-reativo.md`](./docs/adrs/ADR-002-worker-dedicado-polling-vs-listener-reativo.md)
   - [`ADR-003-retry-backoff-exponencial-dead-letter-queue.md`](./docs/adrs/ADR-003-retry-backoff-exponencial-dead-letter-queue.md)
   - [`ADR-004-garantia-entrega-at-least-once.md`](./docs/adrs/ADR-004-garantia-entrega-at-least-once.md)
   - [`ADR-005-hmac-secret-por-endpoint-rotacao.md`](./docs/adrs/ADR-005-hmac-secret-por-endpoint-rotacao.md)
   - [`ADR-006-snapshot-payload-insercao-evento.md`](./docs/adrs/ADR-006-snapshot-payload-insercao-evento.md)
   - [`ADR-007-reuso-padroes-existentes-projeto.md`](./docs/adrs/ADR-007-reuso-padroes-existentes-projeto.md)
5. **[`docs/FDD.md`](./docs/FDD.md)** — como implementar: fluxos, contratos, matriz de erros, observabilidade.
6. **[`docs/TRACKER.md`](./docs/TRACKER.md)** — use como referência cruzada a qualquer momento: toda linha dos outros documentos aponta pra cá, e toda linha daqui aponta de volta para `[hh:mm] Falante` em `TRANSCRICAO.md` ou para um caminho real em `src/`/`prisma/`.

Fontes originais, para conferência:

- [`TRANSCRICAO.md`](./TRANSCRICAO.md) — transcrição da reunião, não alterada.
- [`README-ENUNCIADO.md`](./README-ENUNCIADO.md) — enunciado original do desafio.
