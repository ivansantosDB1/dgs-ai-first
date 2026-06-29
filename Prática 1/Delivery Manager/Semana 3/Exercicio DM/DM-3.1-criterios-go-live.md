# Exercício 3.1 — Critérios de Go-Live com Harness de Governança

> **Trilha AI First — DGS / DB1 Global Software** · Cenário 3 (Governança e Validação)
> **Papel:** Delivery Manager · **Autor:** Ivan Ramos dos Santos · **Repositório:** `db1/novatech-assistant`
> **Ferramentas:** Claude (chat) — critérios · Claude Cowork — dashboard de readiness.
> **Entregável:** critérios de go-live (5 camadas) · dashboard de readiness · plano de rollback.

---

## Princípio orientador (decisão de Delivery)

**O usuário final (atendente) não pode receber resposta alucinada nem ter dado sensível exposto.** Como eliminar alucinação na origem não é viável em 2 semanas (o sistema tem 12% de erro hoje), o go-live não exige "alucinação zero" — exige **"zero alucinação ou risco de segurança que CHEGA ao usuário sem contenção"**. A contenção vem de structured output (toda resposta carrega fonte + confiança), verificação determinística e HITL para baixa confiança em temas sensíveis. Critério controlável fecha; meta absoluta nunca fecharia.

> ⚠️ **Sobre "premissas iniciais do projeto":** não há lista formal de premissas/critérios de aceite disponível. Os critérios abaixo foram **derivados** dos artefatos rastreáveis — guardrails do Cenário 2 (DEVE/NÃO DEVE/QUANDO EM DÚVIDA), ADR-0002 (context budget), ADR-0003 (contradições), `source_document` do AGENTS.md — e dos problemas relatados no cenário (12% de erro, texto livre sem fonte, módulo que logou dado do atendente). Itens marcados **[dedução]** são premissas plausíveis que você deve confirmar; itens marcados **[artefato]** vêm de algo já decidido.

**Legenda:** 🔴 **Bloqueante** (sem isso não vai ao ar) · 🟡 **Desejável** (pode ir ao ar sem, com risco residual aceito).

---

## Critérios de go-live pelas 5 camadas do harness

### Camada 1 — Tool Orchestration (coordenação de ferramentas/agentes)

| # | Critério | Tipo | Origem |
|---|----------|------|--------|
| O1 | Pipeline RAG (ingestão → Azure AI Search → query endpoint → bot Teams) funcional ponta a ponta em staging com os 5 atendentes-piloto | 🔴 | [artefato] |
| O2 | Endpoint degrada com elegância se o AI Search ficar indisponível (retorna "não consegui consultar a base agora", não erro cru nem resposta sem busca) | 🔴 | [dedução] proteção do usuário |
| O3 | Timeout configurado por etapa (busca, geração) para evitar resposta pendurada | 🟡 | [dedução] |

### Camada 2 — Verification Loops (verificação automática de outputs)

| # | Critério | Tipo | Origem |
|---|----------|------|--------|
| V1 | **Toda resposta passa por verificação de `source_document`**: a fonte citada existe na lista de documentos válidos (POL-001, PROC-042, etc.). Resposta sem fonte válida é **bloqueada**, não logada | 🔴 | [artefato] AGENTS.md + regra de corte do Dev 3.1 |
| V2 | Verificação de contradição de versões: resposta que cita PROC-042 não mistura v1 e v2 (ADR-0003) | 🟡 | [artefato] ADR-0003 |
| V3 | Resposta para pergunta sem cobertura na base retorna "não encontrei essa informação", não inventa (anti-alucinação do Anexo B) | 🔴 | [dedução] proteção do usuário |

### Camada 3 — Context & Memory

| # | Critério | Tipo | Origem |
|---|----------|------|--------|
| C1 | Context budget respeitado conforme **ADR-0002** (~system + chunks por query; histórico limitado a 3 turnos no bot) — não reinventar, usar o que o Cenário 1 definiu | 🔴 | [artefato] ADR-0002 |
| C2 | Memória do bot não vaza contexto entre conversas de atendentes diferentes | 🔴 | [dedução] segurança/privacidade |

### Camada 4 — Guardrails (limites + HITL)

| # | Critério | Tipo | Origem |
|---|----------|------|--------|
| G1 | **Guardrails DEVE/NÃO DEVE/QUANDO EM DÚVIDA do Cenário 2 estão ativos e não regrediram** (invariantes): não inventa tier (só Gold/Silver/Standard), não trata carga perigosa como devolvível, cita fonte | 🔴 | [artefato] guardrails C2 |
| G2 | **Structured output obrigatório:** resposta segue schema com `answer`, `source_document`, `confidence_score`. Resposta fora do schema é rejeitada | 🔴 | [artefato] conceito-chave da fase |
| G3 | **Ponto de HITL (obrigatório):** resposta com `confidence_score` abaixo do limiar **E** tema sensível (carga perigosa, valor de frete, SLA contratual) **não vai direto ao atendente** — é roteada para revisão de um atendente sênior/supervisor antes de ser exibida | 🔴 | [artefato] exemplo do enunciado |
| G4 | Nenhum dado pessoal do atendente (e-mail, ID) é logado ou exposto — corrige o problema do módulo de feedback | 🔴 | [artefato] incidente do cenário |

### Camada 5 — Observability

| # | Critério | Tipo | Origem |
|---|----------|------|--------|
| Ob1 | Toda resposta é logada com pergunta, fonte citada, confiança e se passou por HITL (com pino, sem dado pessoal) | 🔴 | [dedução] sem isso não há como auditar pós-go-live |
| Ob2 | Botão de feedback (👍/👎) disponível ao atendente no bot | 🔴 | [dedução] alimenta o 3.2 |
| Ob3 | Dashboard mínimo de produção (taxa de erro, % feedback negativo, % HITL acionado) | 🟡 | [dedução] |

---

## Resumo — o que trava o go-live

São **bloqueantes** (🔴): O1, O2, V1, V3, C1, C2, G1, G2, G3, G4, Ob1, Ob2. Em uma frase: o assistente só vai ao ar se **toda resposta tiver fonte verificada, seguir o schema, preservar os guardrails do C2, conter baixa-confiança sensível via HITL, não vazar dado do atendente, e for auditável**. Os 🟡 (O3, V2, V3→já bloqueante, Ob3) podem ficar para a primeira semana pós-go-live com risco residual explicitamente aceito.

---

## Dashboard de Readiness do Go-Live (Cowork)

> Status: 🟢 Pronto · 🟡 Em andamento / risco · 🔴 Não iniciado / bloqueado. Data-alvo relativa à demo (D = dia da demo, D-14 = hoje).

| Critério | Camada | Tipo | Status | Responsável | Data-alvo |
|----------|--------|------|--------|-------------|-----------|
| O1 — pipeline ponta a ponta | Orchestration | 🔴 Bloq. | 🟢 Pronto | Tech Lead | D-10 |
| O2 — degradação elegante | Orchestration | 🔴 Bloq. | 🟡 Em andamento | Dev Sênior | D-7 |
| V1 — verificação de fonte | Verification | 🔴 Bloq. | 🟡 Em andamento | Dev Sênior | D-7 |
| V3 — sem cobertura → não inventa | Verification | 🔴 Bloq. | 🟡 Em andamento | Dev Pleno | D-7 |
| C1 — context budget (ADR-0002) | Context | 🔴 Bloq. | 🟢 Pronto | Tech Lead | D-10 |
| C2 — sem vazamento entre conversas | Context | 🔴 Bloq. | 🔴 A validar | QA | D-5 |
| G1 — guardrails C2 não regridem | Guardrails | 🔴 Bloq. | 🟡 Em regressão | QA | D-5 |
| G2 — structured output | Guardrails | 🔴 Bloq. | 🟡 Em andamento | Dev Sênior | D-6 |
| G3 — HITL baixa confiança sensível | Guardrails | 🔴 Bloq. | 🔴 Não iniciado | Tech Lead + DM | D-4 |
| G4 — sem logar dado do atendente | Guardrails | 🔴 Bloq. | 🟡 Corrigindo | Dev Pleno | D-6 |
| Ob1 — log auditável | Observability | 🔴 Bloq. | 🟡 Em andamento | Dev Sênior | D-5 |
| Ob2 — botão de feedback | Observability | 🔴 Bloq. | 🟢 Pronto | Dev Pleno | D-8 |
| O3 — timeout por etapa | Orchestration | 🟡 Desej. | 🟡 Em andamento | Dev Pleno | D-3 |
| V2 — contradição de versões | Verification | 🟡 Desej. | 🔴 Não iniciado | Dev Sênior | pós-go-live |
| Ob3 — dashboard de produção | Observability | 🟡 Desej. | 🔴 Não iniciado | DM | D-2 |

**Leitura para a diretoria:** 3 bloqueantes ainda vermelhos (C2, G3, Ob1 em risco) — o go-live depende de fechá-los até D-4. O HITL (G3) é o de maior atenção: não iniciado e é o que protege o usuário no pior caso (carga perigosa de baixa confiança).

---

## Plano de Rollback

| Elemento | Definição |
|----------|-----------|
| **Trigger 1 (automático)** | Taxa de erro verificável (resposta bloqueada por falta de fonte ou fora do schema) **> 10% em 1h** → alerta imediato ao plantão |
| **Trigger 2 (qualidade)** | **% de feedback negativo > 25% em 24h**, ou qualquer resposta alucinada sobre tema sensível (carga perigosa) que tenha chegado ao atendente | 
| **Trigger 3 (segurança)** | Qualquer evidência de dado pessoal do atendente em log/resposta → rollback imediato, sem espera |
| **Quem decide** | Plantão técnico (Tech Lead) aciona; **Delivery Manager autoriza** o desligamento e comunica a NovaTech. Trigger 3 (segurança) o plantão executa sem aguardar autorização |
| **Ação** | Desligar o bot no Teams (feature flag / desabilitar app em staging→prod) e voltar os atendentes ao processo manual de consulta documental. O pipeline de ingestão continua rodando; só a interface com o usuário é desativada |
| **Comunicação** | DM avisa os 5 atendentes-piloto e o patrocinador da NovaTech em até 30 min; registro do incidente para a retro |

> O rollback é **parcial e reversível por design**: desliga só a camada que toca o usuário (bot), preserva o que foi indexado, e tem dono claro por tipo de trigger. Segurança não espera reunião.
