# Governança de Specs — Modelo SDD · Projeto NovaTech Assistant

> **DB1 Global Software** · Padrão de Governança de Spec Driven Development
> **Autor:** Ivan Ramos dos Santos — Gerente de Projetos, DB1 Global Software
> **Repositório:** `db1/novatech-assistant` · **Cenário 2 — Estruturação do Trabalho**
> **Exercício 2.1/2.2 (DM):** Entregável — documento de governança + board de tracking (Cowork) + processo de change management.

> ⚠️ **Nota sobre o Anexo C:** A estrutura de diretórios abaixo foi ancorada nos paths conhecidos do repositório (`/docs/adr/`, `/skills/{foundation,domain,artifact}/`, `/src/functions/query/`). A pasta de specs adota `/specs/<modulo>/`. **Confronte com o Anexo C oficial** — se ele definir outro path para specs, ajuste as Partes 1, 2 e 5.

---

## Sumário

1. Modelo de Governança de Specs (papéis, estrutura, versionamento, rastreabilidade)
2. Board de Tracking
3. Processo de Change Management
4. Critérios de Governança (indicadores)
5. (Publicação executiva — ver HTML)

---

# PARTE 1 — Modelo de Governança de Specs

No SDD da NovaTech, a spec é um **contrato executável**, não documento passivo. Três artefatos por módulo:

- **`requirements.md`** — o QUE precisa ser construído (requisitos + critérios de aceite verificáveis).
- **`plan.md`** — COMO será construído (arquitetura, contratos de API, stack, decisões técnicas).
- **`tasks.md`** — decomposição em unidades atômicas executáveis por devs e agentes.

Cada transição (requirements → plan → tasks) é um **checkpoint humano** (os validation gates do Exercício 2.1).

## 1.1 Papéis e Responsabilidades — Matriz RACI

Legenda: **R** = Responsável (executa) · **A** = Aprova (accountable, único) · **C** = Consultado · **I** = Informado.

| Atividade | Product Specialist | Delivery Manager | Tech Lead | Desenvolvedor | QA | Agentes IA (Claude/Copilot) |
|-----------|:------------------:|:----------------:|:---------:|:-------------:|:--:|:---------------------------:|
| **Criar Requirements** | R | I | C | I | C | R (rascunho assistido) |
| **Revisar Requirements** | C | I | C | I | **R** (testabilidade) | — |
| **Aprovar Requirements** | C | I | **A** | I | C | — |
| **Criar Plan** | C | I | R | C | C | R (rascunho assistido) |
| **Revisar Plan** | I | I | C | **R** (sênior) | C | — |
| **Aprovar Plan** | I | I | **A** | I | I | — |
| **Criar Tasks** | I | I | C | **R** | I | R (decomposição via Copilot) |
| **Revisar Tasks** | I | I | **R** | C | I | — |
| **Aprovar Tasks** | I | C | **A** | I | I | — |
| **Governar o processo** | I | **R/A** | C | I | C | — |

Observações de design da RACI:
- **Agentes de IA nunca aprovam** — só geram rascunhos (R compartilhado na criação). Aprovação é sempre humana, coerente com os validation gates.
- **Tech Lead é o aprovador único** das três transições técnicas (evita aprovação difusa).
- **QA entra cedo** (revisa requirements pela testabilidade), não só no fim.
- **Delivery Manager governa o processo** (board, prazos, change management), não o conteúdo técnico.

## 1.2 Estrutura das Specs no Repositório

```
db1/novatech-assistant/
├── docs/
│   └── adr/                          # ADRs do Cenário 1 (decisões herdadas)
│       ├── ADR-0001-modelo-llm.md
│       ├── ADR-0002-context-budget.md
│       ├── ADR-0003-documentos-contraditorios.md
│       └── ADR-0004-build-vs-buy.md
├── specs/                            # ⚠️ confirmar path no Anexo C
│   ├── ingestao-documentos/
│   │   ├── requirements.md
│   │   ├── plan.md
│   │   └── tasks.md
│   ├── query-endpoint/
│   │   ├── requirements.md
│   │   ├── plan.md
│   │   └── tasks.md
│   ├── feedback-api/
│   │   ├── requirements.md
│   │   ├── plan.md
│   │   └── tasks.md
│   ├── bot-teams/
│   │   ├── requirements.md
│   │   ├── plan.md
│   │   └── tasks.md
│   └── painel-web/
│       ├── requirements.md
│       ├── plan.md
│       └── tasks.md
├── src/
│   └── functions/query/              # implementação (Anexo C)
└── skills/{foundation,domain,artifact}/
```

**Convenção de nomenclatura:** pasta por módulo em `kebab-case`; sempre os três arquivos fixos (`requirements.md`, `plan.md`, `tasks.md`). Versão aprovada marcada por **tag git** `spec/<modulo>/<artefato>/vN` (ex: `spec/query-endpoint/requirements/v2`).

**Estrutura mínima obrigatória de cada arquivo:**

`requirements.md`: cabeçalho (módulo, versão, status, autor, data, links para ADRs) · objetivo · escopo / fora-de-escopo · requisitos funcionais com **ID** (`REQ-QE-01`) · critérios de aceite verificáveis por requisito · glossário de domínio aplicável.

`plan.md`: cabeçalho + link para o `requirements.md` que origina · arquitetura · contratos de API (request/response) · stack e libs (TypeScript, Zod, Azure Functions v4, pino) · context budget aplicável (ADR-0002) · decisões técnicas (links para ADRs) · riscos técnicos.

`tasks.md`: cabeçalho + link para o `plan.md` · lista de tasks atômicas, cada uma com **ID** (`TASK-QE-001`), descrição, dependências, critério de aceite, requisito de origem (`REQ-QE-0x`).

**Relacionamento entre os três:** cada `plan.md` referencia o `requirements.md` que o origina; cada task em `tasks.md` aponta o requisito de origem. Isso fecha a cadeia REQ → PLAN → TASK por ID.

**Exemplo (módulo Query Endpoint):** `REQ-QE-03` ("toda resposta deve citar `source_document`") → no `plan.md` vira o contrato de resposta com campo obrigatório `source_document` → em `tasks.md` gera `TASK-QE-014` ("implementar serialização da resposta com `source_document` validado por Zod"), com critério de aceite "resposta sem `source_document` retorna erro 500 logado".

## 1.3 Versionamento

- **Estratégia:** versionamento semântico simplificado por artefato — `vMAJOR.MINOR`. MINOR = ajuste sem mudar contrato; MAJOR = muda contrato/escopo aprovado.
- **Versão aprovada:** marcada por tag git (`spec/<modulo>/<artefato>/vN`) no commit aprovado pelo Tech Lead. O `status` no cabeçalho passa a `Aprovada`.
- **Controle de histórico:** git é a fonte da verdade; cada versão aprovada tem tag imutável. O cabeçalho do arquivo mantém um bloco `## Histórico` com data, versão, autor e resumo da mudança.
- **Registro de alterações:** toda alteração pós-aprovação entra pelo change management (Parte 3) e gera entrada no histórico + link ao PR.
- **Critério de nova versão:** nova MAJOR quando muda um requisito aprovado, contrato de API ou escopo; nova MINOR para esclarecimento que não quebra contrato.
- **Arquivamento:** versões antigas **não são apagadas** (coerente com ADR-0003 — documentos obsoletos marcados, não excluídos). A tag git preserva; o cabeçalho da versão superada recebe `status: Superada por vN`.

## 1.4 Rastreabilidade ponta a ponta

Cadeia: **Requirements → Plan → Tasks → Desenvolvimento → Testes → Deploy.**

- **Vínculo entre artefatos:** por ID. `REQ-xx` → `plan` (seção que o atende) → `TASK-xxx` (campo "requisito de origem") → branch/PR (nome `feat/<modulo>-<task-id>`) → teste (nomeado com o `REQ`/`TASK`) → release.
- **Dependências:** declaradas no campo `dependências` de cada task e no board (Parte 2).
- **Registro de decisões:** decisões arquiteturais viram ADR em `/docs/adr/`; o `plan.md` linka. Decisões menores ficam no histórico do arquivo.
- **Relação tarefas ↔ PRs ↔ testes ↔ entregas:** convenção de nome amarra tudo — PR referencia `TASK-id` no título; commit segue `feat(query): TASK-QE-014 ...`; teste cita o `REQ`; a release lista os `REQ` entregues. Resultado: dá para partir de qualquer requisito e chegar ao deploy (e vice-versa).

---

# PARTE 2 — Board de Tracking

> Gerado no **Claude Cowork**. Compatível com Azure DevOps Boards, Jira, GitHub Projects e Planner (colunas = campos; status = swimlanes ou coluna "Status").

**Status obrigatórios (ciclo de vida da spec):**
`Rascunho` → `Em Revisão` → `Aprovada` → `Em Implementação` → `Validada`

**Colunas do board:** Módulo · Tipo da Spec · Responsável · Status Atual · Última Alteração · Próxima Aprovação · Dependências · Riscos.

| Módulo | Tipo | Responsável | Status | Última alteração | Próxima aprovação | Dependências | Riscos |
|--------|------|-------------|--------|------------------|-------------------|--------------|--------|
| Pipeline de Ingestão | requirements | Product Specialist | Aprovada | — | Plan (Tech Lead) | ADR-0004 (chunking) | Tabelas complexas quebram chunking |
| Pipeline de Ingestão | plan | Tech Lead | Em Revisão | — | Plan (Tech Lead) | requirements aprovado | OCR de docs escaneados |
| API de Busca (Query Endpoint) | requirements | Product Specialist | Aprovada | — | Plan (Tech Lead) | ADR-0002 (context budget) | Context overflow em multi-domínio |
| API de Busca (Query Endpoint) | plan | Tech Lead | Em Revisão | — | Tasks (Tech Lead) | requirements aprovado | Latência do Azure AI Search |
| API de Feedback | requirements | Product Specialist | Em Revisão | — | Requirements (Tech Lead) | query-endpoint (consome resposta) | Baixa adesão dos atendentes |
| Bot do Teams | requirements | Product Specialist | Rascunho | — | Requirements (Tech Lead) | query-endpoint + feedback-api | Limite de histórico (3 turnos, ADR-0002) |
| Painel Web (Métricas) | requirements | Product Specialist | Rascunho | — | Requirements (Tech Lead) | feedback-api (dados) | Escopo de métricas indefinido |

> O board inicial mostra cada módulo na fase real esperada no início do Cenário 2: os dois módulos centrais (ingestão e query) já com requirements aprovados e plan em revisão; os demais em estágios anteriores, refletindo as dependências.

**Regras de movimentação:**
- Só sai de `Em Revisão` para `Aprovada` com o gate humano correspondente (Tech Lead) registrado.
- `Em Implementação` exige a versão da spec **tagueada** (Parte 1.3).
- `Validada` exige Gate 4 (QA) do Exercício 2.1 passado.

---

# PARTE 3 — Processo de Change Management

Aplica-se a alterações de spec **após o início da implementação** (antes disso, é simples iteração de rascunho). Toda mudança entra por um **Spec Change Request (SCR)** registrado como issue no repositório, com ID `SCR-<nº>`.

**Fluxo geral:** solicitação (SCR) → avaliação de impacto → aprovação → atualização dos artefatos → tratamento das tasks em andamento → comunicação.

| Cenário | Quem solicita | Quem avalia impacto | Quem aprova | Como tratar tasks já iniciadas |
|---------|---------------|---------------------|-------------|--------------------------------|
| **Mudança de requisito** | Product Specialist | Tech Lead + QA | Tech Lead (+ DM se afeta prazo) | Pausar tasks afetadas; re-decompor após nova `requirements` |
| **Descoberta técnica** | Desenvolvedor / Tech Lead | Tech Lead | Tech Lead | Ajustar `plan`/`tasks`; tasks em curso continuam se não conflitam |
| **Correção de erro** | Qualquer papel | Tech Lead | Tech Lead | Hotfix vira task nova ligada ao `REQ` afetado; spec atualizada depois |
| **Mudança regulatória** | Compliance (NovaTech) / PS | Tech Lead + PS | Delivery Manager + Tech Lead | Reavaliar escopo; pode gerar MAJOR e bloquear deploy até conformidade |
| **Mudança de escopo** | Product Specialist / Patrocinador | Tech Lead + DM | Delivery Manager | Tasks fora do novo escopo são canceladas/arquivadas com rastro |

**Como registrar e atualizar os artefatos:**
- O SCR descreve a mudança, o motivo e os IDs afetados (`REQ`/`TASK`).
- **`requirements.md`:** nova versão (MAJOR se muda contrato/escopo); entrada no `## Histórico` linkando o SCR.
- **`plan.md`:** atualizado se a arquitetura/contrato muda; novo ADR se for decisão estrutural.
- **`tasks.md`:** tasks afetadas marcadas (`bloqueada`/`cancelada`/`a refazer`); novas tasks criadas com referência ao SCR.
- **Tasks já iniciadas:** se a mudança invalida o trabalho, a task é revertida/cancelada com nota no PR; se não conflita, segue e é revalidada no Gate 3.
- **Comunicação ao time:** o DM anuncia o SCR aprovado no canal do projeto (Teams), atualiza o board e marca os responsáveis impactados como `I` na RACI.

**Impacto em atividades em andamento:** o change management explicita que mudança pós-implementação **não é gratuita** — ela reabre gates já passados (a spec volta de `Em Implementação` para `Em Revisão` quando a mudança é MAJOR), o que protege o time de implementar sobre contrato instável.

---

# PARTE 4 — Critérios de Governança (Indicadores)

| Indicador | Como medir | O que protege |
|-----------|------------|---------------|
| **Tempo médio de aprovação de spec** | Média de horas úteis entre `Em Revisão` e `Aprovada` | Gargalo nos gates; saúde do fluxo (meta alinhada aos SLAs do Ex. 2.1) |
| **Alterações após aprovação (SCR/spec)** | Nº de SCRs por módulo após `Aprovada` | Qualidade do discovery e da spec inicial — muitos SCRs = requisito imaturo |
| **% de rastreabilidade REQ → Task** | Tasks com `requisito de origem` preenchido ÷ total de tasks | Garante que nada é implementado sem origem (evita task "fantasma") |
| **Tempo médio de processamento de mudança** | Média de horas entre abertura e fechamento do SCR | Agilidade do change management sem perder controle |
| **% de specs atualizadas** | Specs cujo arquivo bate com o código entregue ÷ total | Evita spec "morta" — mantém o contrato executável vivo |
| **Cobertura de aprovação humana** | % de transições com gate humano registrado | Garante que IA gera mas humano valida (princípio do AI First) |

Cada métrica liga a um risco concreto do projeto: rastreabilidade baixa indica risco de código sem requisito; muitos SCRs indicam discovery raso; spec desatualizada indica perda do valor de contrato executável.

---

## Apêndice — Conexão com o Cenário 1

Este modelo de governança **consome** as decisões da fase anterior, em vez de reinventá-las: os ADRs em `/docs/adr/` são referenciados pelos `plan.md`; o context budget da ADR-0002 entra na estrutura mínima do plan; o tratamento de contradições da ADR-0003 fundamenta a regra de "arquivar, não excluir" versões de spec; e os validation gates do Exercício 2.1 são os mesmos checkpoints que movem as specs entre os status do board.
