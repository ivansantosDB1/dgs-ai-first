# Exercício 3.2 — Observabilidade por Exceção e Melhoria Contínua

> **Trilha AI First — DGS / DB1 Global Software** · Cenário 3 (Governança e Validação)
> **Papel:** Delivery Manager · **Autor:** Ivan Ramos dos Santos · **Repositório:** `db1/novatech-assistant`
> **Ferramentas:** Claude (chat) — plano · Claude Cowork — template de relatório semanal.
> **Entregável:** plano de observabilidade por exceção · sistema de notificação (Teams) · template de relatório semanal.

---

## Princípio orientador (decisão de Delivery)

**O time não monitora dashboard diariamente.** O sistema vigia sozinho e só interrompe alguém **quando um indicador sai do range esperado**. Cada exceção vira um **card priorizado (P1/P2/P3)** postado num **canal centralizado no Microsoft Teams**, com **SLA de resposta de 4h a 48h** conforme a prioridade. Em paralelo, um **relatório semanal de tendência** dá à NovaTech a visão de "melhorou ou piorou" sem exigir que ninguém acompanhe métricas no dia a dia.

Dois mecanismos complementares:
- **Alerta por exceção** (reativo, urgente): "me avise quando sair do padrão".
- **Relatório semanal** (proativo, executivo): "me mostre a tendência".

---

## Parte 1 — Métricas coletadas (4 dimensões)

| Dimensão | Métrica | Por quê importa |
|----------|---------|-----------------|
| **Uso** | Perguntas/dia · perguntas/atendente · tempo médio de resposta percebido | Mede adoção; queda brusca pode indicar bot quebrado ou abandonado |
| **Qualidade** | % de feedback negativo (👎) · % de escalações para humano (HITL acionado) · % de respostas bloqueadas por falta de fonte | Mede se o usuário confia na resposta; é o sinal mais ligado ao "não impactar o usuário" do 3.1 |
| **Técnicas** | Latência (p50/p95) · taxa de erro (5xx, timeout) · disponibilidade do bot e do AI Search | Saúde da infraestrutura |
| **Conteúdo** | Documentos mais consultados · **perguntas sem resposta** (sem cobertura na base) · temas com mais feedback negativo | Aponta lacunas documentais — alimenta diretamente a melhoria contínua |

> As métricas de **qualidade** e **conteúdo** são as que diferenciam observabilidade de IA de monitoramento de software comum. Medir só latência/uptime (técnicas) é o erro clássico — aqui elas são uma dimensão entre quatro.

---

## Parte 2 — Sistema de notificação por exceção (Teams)

### Arquitetura do fluxo

```
Métricas (logs/telemetria)  →  Avaliador de thresholds (job periódico)
        │                                  │
        │                          fora do range?
        │                                  │ sim
        └──────────────►  Gera CARD priorizado  ──►  Canal Teams central
                                                        "NovaTech Assistant — Alertas"
                                                              │
                                                    Time recebe + responde dentro do SLA
```

O avaliador roda em janelas curtas (ex: a cada 15 min para P1, a cada 1h para P2/P3). Se nenhum threshold é violado, **ninguém é notificado** — esse é o ponto do monitoramento por exceção.

### Ranges e thresholds → prioridade → SLA

| Indicador | Range normal (verde) | Atenção (amarelo) | Exceção que dispara card | Prioridade | SLA de resposta |
|-----------|----------------------|-------------------|--------------------------|:----------:|:---------------:|
| Resposta alucinada sobre tema sensível chegou ao atendente (carga perigosa, frete, SLA) | 0 | — | qualquer ocorrência (≥1) | **P1** | **4h** |
| Dado pessoal do atendente exposto em log/resposta | 0 | — | qualquer ocorrência (≥1) | **P1** | **4h** |
| Disponibilidade do bot | ≥ 99% | 97–99% | < 97% em 1h | **P1** | **4h** |
| % feedback negativo (👎) | ≤ 10% | 10–15% | **> 15% em 24h** | **P2** | **24h** |
| Taxa de erro técnico (5xx/timeout) | ≤ 2% | 2–5% | > 5% em 1h | **P2** | **24h** |
| % de respostas bloqueadas por falta de fonte | ≤ 5% | 5–10% | > 10% em 24h | **P2** | **24h** |
| Latência p95 | ≤ 5s | 5–8s | > 8s sustentado por 1h | **P3** | **48h** |
| Perguntas sem resposta (sem cobertura) | ≤ 8%/sem | 8–15% | > 15% na semana | **P3** | **48h** |
| Queda de uso (perguntas/dia) | dentro de ±30% da média | — | queda > 50% vs média semanal | **P3** | **48h** |

> Os números são **propostas de partida** e devem ser calibrados com 1–2 semanas de dados reais. Os thresholds de P1 (alucinação sensível, dado pessoal) são **coerentes com os triggers de rollback do 3.1** — o alerta P1 e o trigger de rollback compartilham os mesmos sinais críticos.

### Definição de prioridade, card e SLA

| Prioridade | Significado | SLA de resposta | Quem recebe / responde | Ação esperada |
|:----------:|-------------|:---------------:|------------------------|---------------|
| **P1 — Crítico** | Usuário final impactado ou risco de segurança | **4h** | Plantão (Tech Lead) + DM marcados (@) no card | Investigar imediato; pode acionar rollback (3.1) |
| **P2 — Alto** | Qualidade degradando, sem impacto direto ainda | **24h** | Time de dev + QA | Investigar causa; corrigir ou criar plano |
| **P3 — Médio** | Tendência a observar, lacuna de conteúdo | **48h** | DM triagem → encaminha (PS p/ conteúdo, Dev p/ técnico) | Priorizar no backlog |

### Conteúdo do card no Teams (template)

Cada card postado no canal traz:
- **Título:** `[P1] Resposta alucinada sobre carga perigosa chegou ao atendente`
- **Disparado por:** indicador + valor medido vs threshold (ex: "feedback negativo 18% > 15% nas últimas 24h")
- **Janela:** período em que a exceção ocorreu
- **SLA:** prazo de resposta e horário-limite (ex: "responder até 14h de hoje")
- **Responsável:** @menção conforme prioridade
- **Link:** para o log/conversa que gerou o alerta
- **Status:** Aberto → Em análise → Resolvido (atualizável no próprio card)

> Implementação compatível com o stack: **Teams Incoming Webhook / Workflows** recebendo um Adaptive Card postado pelo job de avaliação (Azure Function agendada lendo a telemetria). Não requer ferramenta nova.

---

## Parte 3 — Feedback loop (atendente → correção no assistente)

O 👎 do atendente (Ob2 do 3.1) não morre numa métrica — ele entra num ciclo de correção:

```
Atendente dá 👎  →  agrega no indicador de qualidade
        │
        ├─ se cruza threshold → card P2 no Teams (SLA 24h)
        │
        ▼
Triagem (DM/QA): qual a causa-raiz?
        │
        ├─ Lacuna documental  → PS cria/atualiza doc → reindexação no AI Search
        ├─ Documento desatualizado/contraditório → atualiza fonte (ADR-0003) → reindexação
        ├─ Prompt mal calibrado → ajuste de prompt  →  REGRESSÃO antes de publicar
        └─ Chunk errado recuperado → ajuste no pipeline de RAG
        │
        ▼
ANTES de aplicar: regression testing
  - respostas que estavam corretas continuam corretas
  - guardrails do Cenário 2 (DEVE/NÃO DEVE) NÃO regridem
        │
        ▼
HITL: mudança de prompt/doc que afeta tema sensível precisa de aprovação (Tech Lead) antes de produção
        │
        ▼
Publica correção  →  monitora se o indicador volta ao range
```

Esse loop preserva os **guardrails do Cenário 2 como invariantes**: nenhuma correção pode, ao consertar um caso, fazer o assistente voltar a inventar tier ou tratar carga perigosa como devolvível. Toda mudança passa por regressão antes de ir ao ar.

---

## Parte 4 — Template de relatório semanal (Cowork)

> Uma página. Um executivo da NovaTech entende em 2 minutos se o assistente **melhorou ou piorou**. A coluna "Tendência" (▲ melhor · ▬ estável · ▼ pior) é o coração do relatório.

---

### Relatório Semanal — Assistente NovaTech
**Semana:** [dd/mm – dd/mm] · **Responsável:** Ivan Ramos dos Santos (DM) · **Status geral:** 🟢 / 🟡 / 🔴

**1. Resumo executivo (1 frase)**
> Ex: "O assistente atendeu 1.240 perguntas com 91% de feedback positivo — qualidade estável e uso em crescimento; 1 alerta P1 resolvido dentro do SLA."

**2. KPIs principais**

| KPI | Esta semana | Semana anterior | Tendência | Meta |
|-----|:-----------:|:---------------:|:---------:|:----:|
| Perguntas atendidas | — | — | ▲ ▬ ▼ | — |
| % feedback positivo | — | — | ▲ ▬ ▼ | ≥ 85% |
| % escalações (HITL) | — | — | ▲ ▬ ▼ | ≤ 15% |
| Latência p95 | — | — | ▲ ▬ ▼ | ≤ 5s |
| Perguntas sem resposta | — | — | ▲ ▬ ▼ | ≤ 8% |

**3. Alertas da semana (cards)**

| Prioridade | Alerta | Respondido no SLA? | Status |
|:----------:|--------|:------------------:|--------|
| P1 / P2 / P3 | — | Sim / Não | Resolvido / Em análise |

**4. Top 3 lacunas de conteúdo** (perguntas mais frequentes sem resposta → o que documentar)
1. — 2. — 3. —

**5. Ações planejadas para a próxima semana**
- —

---

## Conexão com os cenários anteriores

O plano consome o que já existe em vez de reinventar: o botão de feedback e os logs auditáveis vêm dos critérios de go-live do **3.1**; os thresholds de P1 espelham os **triggers de rollback** do 3.1; os **guardrails do Cenário 2** são invariantes que o feedback loop não pode regredir; o tratamento de fonte desatualizada/contraditória segue a **ADR-0003**; e a reindexação respeita o pipeline de RAG do **Cenário 1**. O canal de notificação usa o **Teams**, já no stack do projeto — sem ferramenta nova.
