# SPEC — Sprint Orchestrator Symmetry v1.4

**Projeto:** CalibraCV
**Data de fechamento da spec:** 2026-05-14
**Versão:** orchestrator-symmetry-v1.4
**MER de referência:** v37
**Ambiente alvo:** pré-produção, base operacional limpa (sem usuários ativos, sem dados históricos relevantes)
**Executor:** Antigravity (com acesso direto ao Supabase para migrations)
**Dependência:** sprint cleanup v3.3 (paralelismo controlado por SUB-PR descrito em §0.5)

---

## §0 Diretrizes da sprint

### §0.1 Convenção nominal

- Spec, narrativa e UI em PT-BR puro, sem estrangeirismos.
- Identificadores técnicos (rotas, arquivos, funções, parâmetros) em inglês.
- "Telemetria" e "instrumentação" aceitos como termos técnicos consagrados em PT-BR.
- A entidade `function_orchestrator_items` será renomeada dentro desta sprint para `function_orchestrator_role_items` (§2.6). Daqui em diante, o nome novo é o canônico.

### §0.2 Diretrizes de execução

1. **TodoWrite item-por-item antes de edição.** Para cada §X.Y, criar item na todo list e marcar concluído somente após validação contra o critério da §7.2. Validação de conformidade é DURANTE a execução, não depois.
2. **Validação ground truth obrigatória.** Antes de submeter PR, rodar os blocos de evidência da §7.1 contra o banco e o código real. Revisão estática multi-AI não substitui.
3. **Spec é fotografia.** Sem narrativa histórica, mapeamento de fixes acumulados, ou referência a versões anteriores no corpo.
4. **PT-BR puro em narrativa e UI.** Nomes técnicos em inglês.
5. **Sem novos canais externos ou controles operacionais.** Esta sprint não introduz SMS, e-mail externo, push, webhook externo, budget guard novo, kill-switch novo, rate limiter novo, monitor externo novo ou alerta novo — qualquer adição desta natureza requer autorização expressa do product owner. O cache in-process adicionado em §4.1 segue precedente da cleanup v3.3 (D-PS-68 v3.3) e é memoização, NÃO rate limiter — ver D-PS-86 desta sprint.
6. **Simetria skill↔role.** Toda decisão considera paridade. Assimetrias justificadas em D-PS específica. A janela de simetrização é durante a sprint que toca o objeto — não há ciclo posterior dedicado.
7. **Paralelismo controlado por SUB-PR (não por sprint inteira).** Ver §0.5.

### §0.3 Escopo

Esta sprint entrega valor em **três frentes paralelas** consumindo a mesma infraestrutura de dados sobre o pipeline de curadoria de vagas (`function_orchestrator_role_items`, `function_orchestrator_skill_items`, `function_orchestrator_runs`):

**Frente A — Infraestrutura de dados (pré-requisito das demais)**

- Tabela paralela `function_orchestrator_skill_items` espelhando `function_orchestrator_role_items` semanticamente, com enum próprio refletindo os paths reais do pipeline de skill (§2.1)
- 5 contadores agregados de skill em `function_orchestrator_runs` (§2.2)
- Refator da função de descoberta de skills (`discoverAndLinkSkills` interna + wrapper `safeDiscoverAndLinkSkills`, conforme E0) expandindo retorno void → `DiscoverResultDetailed` item-por-item (§3.1). **Caller já está em `await` — refator é apenas captura do retorno, não conversão de fire-and-forget (D-PS-97)**
- INSERT de `function_orchestrator_skill_items` em todos os call sites cobrindo fluxos A, B, C (§3.2)
- Atualização dos finalizers de run para acumular contadores de skill via RPC SQL (§3.3)
- Rename de `function_orchestrator_items` para `function_orchestrator_role_items`, incluindo atualização do corpo de funções que referenciam o nome antigo — em especial `fn_recompute_jcr_confidence_median` (§2.6, §3.4)
- Auditoria sistemática de todas as funções/triggers/scripts que referenciam `function_orchestrator_items` por nome para atualização no rename (§6.2)
- Evidence pré-PR obrigatório (E0 em §6.1) confirmando nome real da função de descoberta no codebase + ground truth do aggregator atual antes do SUB-PR 5 — D-PS-88

**Frente B — Dashboard global (consumidor materializado via `dashboard_daily_summary`)**

- Expansão de `lib/admin/dashboard-day-aggregator.ts` para coletar métricas FORI/FOSI/FOR contadores no `aggregateDayData()` (§3.6)
- Campos novos materializados no JSONB de `dashboard_daily_summary.data` paralelos aos pré-existentes role-only — `operational.o7_skill`, `operational.o8_skill`, `operational.o9_skill`, `operational.items_processed` (D-PS-92)
- Backfill obrigatório via `/api/admin/backfill-dashboard-summary` para regenerar 26 dias rolling com a família nova preenchida
- Refator UI dos painéis 2.7 (O7 — drift), 2.8 (O8 — Caminho de resolução), 2.9 (O9 — Saúde do pipeline) para exibir séries paralelas role+skill (§5.2)
- Ajuste do card 2.1 KPI com novo indicador "Items processados pelo pipeline" (role + skill) — D-PS-94

**Frente C — Aba Limiares (consumidor live + Redis padrão `pg.Pool`)**

- Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` **migra de cache in-process Map para padrão Limiares com `pg.Pool` + Redis TTL 300s** preservando método HTTP POST, nome canônico, contrato pré-existente. **Cache in-process Map TTL 30s da cleanup v3.3 (D-PS-68) é REMOVIDO desta sprint (D-PS-86 desta sprint substitui)** (§4.1)
- Queries novas em `lib/admin/limiares/queries.ts`: distribuição de confidence em FORI, distribuição em FOSI, agregação sobre `opus_arbitration_outcomes` — alimentam o impact-preview e o enriquecimento dos painéis Limiares (§3.5)
- Enriquecimento dos painéis Limiares 1 (Hard Gate) e 8 (creation_confidence) com séries via FOSI/FORI como subproduto natural do trabalho de Frente C (D-PS-91)
- Payload expandido do `impact-preview` com `current_impact` + `proposed_impact` + `histogram` + `sample_status` + seletor de janela [7d / 30d / 90d], **preservando campos pré-existentes da cleanup v3.3** (`affected_count`, `projected_event_cost_usd`, `cost_is_fallback`) onde aplicável — D-PS-85
- Integração do payload expandido ao componente `ImpactPreview` no modal §5.1.6 — adição à caixa qualitativa pré-existente, não substituição (§5.1)
- Extração de `EditModal` + `ImpactPreview` da `app/admin/pipeline-config/page.tsx` para arquivos próprios em `components/admin/pipeline-config/` como pré-trabalho do SUB-PR 8 (D-PS-93)

**Não inclui (out-of-scope cravado):**

- Trigger novo para `confidence_median` em `job_canonical_skills` — `fn_jps_recompute_jcs` já popula essa coluna a partir de fonte robusta (`job_posting_skills` curated). Ver D-PS-74.
- Mudança de fonte de `confidence_median` em qualquer dos lados — assimetria de fontes role/skill está documentada em D-PS-49 da cleanup v3.3 e é INTENCIONAL. Ver D-PS-75.
- Pipeline CV (input 1) — D-PS-43 da cleanup v3.3.
- Cobertura do path quarantined (legacy `mapSkillsToCanonical`) — skills extraídas em vagas quarantenadas permanecem invisíveis ao orchestrator. Ver D-PS-83.
- Simetria estrutural ampla dos contadores outcome-oriented de FOR (8 colunas role-only pré-existentes) — esta sprint adiciona apenas contadores stage-oriented de skill. Ver D-PS-84. D-PS-50 da cleanup v3.3 já documentou recorte parcial sobre `canonical_created`/`canonical_promoted`; esta sprint amplia o reconhecimento para os 8 contadores pré-existentes.
- Adição de `bulk_run_id` para agregação multi-lote — dependência da tabela `bulk_curation_progress` que será criada no Sub-PR3 do benchmark v14 (adiado). Ver LK-PS-07.
- Cobertura das 4 chaves de `confidence` no endpoint `impact-preview` — simular impacto exige recalcular `confidence_median` em massa. Frente futura: D-PS-80.
- Painel admin dedicado "Skills extraídas por run" — opcional pós-MVP, vira sprint subsequente caso volume operacional justifique.
- Painéis novos em `/admin/dashboard?tab=limiares` — esta sprint enriquece 2 painéis existentes (1 e 8), não cria novos.
- Painéis Limiares 2, 4, 5, 6, 7, 9, 10 — sem interseção significativa com FORI/FOSI/FOR; ficam intocados.
- Backfill histórico em `function_orchestrator_skill_items` — base limpa em pré-produção; histórico vai sendo populado conforme uso natural.
- Substituição ou deprecação dos 8 estimators pré-existentes (`pipeline-impact-estimators.ts` da cleanup v3.3) — preservados e consumidos pelo endpoint refatorado para os campos `affected_count` + `projected_event_cost_usd` (D-PS-85).
- Renomeação dos campos pré-existentes role-only `o7`/`o8`/`o9` para `o7_role`/`o8_role`/`o9_role` — manteria simetria nominal completa mas quebra retrocompat de consumidores. Frente futura caso a renomeação se mostre necessária.

### §0.4 Mapa de simetria desta sprint

| § | Conteúdo | Frente | Simetria | Mecanismo |
|---|---|---|---|---|
| §2.1 | `function_orchestrator_skill_items` | A | parcial — espelha estrutura mas enum de stage reflete paths reais de skill | tabela paralela, schema espelhado, enum próprio (D-PS-79) |
| §2.2 | 5 colunas `skills_*` em `function_orchestrator_runs` | A | stage-oriented; 8 colunas role pré-existentes são outcome-oriented (D-PS-84) | colunas paralelas, semântica explicitada via COMMENT |
| §2.6 | rename `function_orchestrator_items` → `_role_items` | A | simetria nominal | rename + atualização mecânica de corpos de funções dependentes |
| §3.1 | refator função de descoberta retornando `DiscoverResultDetailed` | A | par com tipo de retorno equivalente em path de role | instrumentação de todos paths internos; caller já em await (D-PS-97) |
| §3.2 | `insertFOSkillItem` em todos os call sites identificados | A | par com `insertFORoleItem` (renomeado em §3.4) | cobertura A/B/C; quarantined path explicitamente fora |
| §3.3 | acumuladores em 3 finalizers via RPC SQL | A | par com acumulação de role; RPC obrigatória por LK-CBO-31 | RPC `count_skill_items_by_status` |
| §3.5 | queries novas em `lib/admin/limiares/queries.ts` | C | par role+skill via UNION ALL nas queries de distribuição de confidence | `pg.Pool` direto + Redis TTL 300s padrão Limiares |
| §3.6 | expansão de `aggregateDayData` em `dashboard-day-aggregator.ts` | B | par role+skill via campos paralelos `o7_skill`/`o8_skill`/`o9_skill` no JSONB | sub-função `aggregatePipelineOrchestratorMetrics` chamada de dentro de `aggregateDayData` |
| §4.1 | endpoint `pipeline-config/[key]/impact-preview` no padrão Limiares | C | par role+skill via parâmetro `key` | `pg.Pool` direto + Redis TTL 300s; cache in-process Map TTL 30s REMOVIDO (D-PS-86 substitui D-PS-68 cleanup v3.3); reutiliza mapa de `pipeline-config-tooltips.ts` + estimators pré-existentes de `pipeline-impact-estimators.ts` |
| §5.1 | integração no modal §5.1.6 com extração de `EditModal` + `ImpactPreview` | C | visualização adicional consolidada agnostic role/skill | extração para arquivos próprios em `components/admin/pipeline-config/` + ampliação do componente para consumir payload novo |
| §5.2 | refator UI dos painéis 2.7/2.8/2.9 do dashboard global | B | bifurcação em séries paralelas role+skill | mesmo componente parametrizado por `entityType`; pré-existentes role-only mantidos para retrocompat |
| §5.2 | KPI novo "Items processados pelo pipeline" no card 2.1 | B | exibe somente o total agregado role+skill no header; breakdown role/skill no tooltip | adição ao card pré-existente, sem refator do layout |

**Assimetrias intencionais preservadas (referência apenas, não tocadas nesta sprint):**

- Fontes de `confidence_median` divergem entre role e skill — D-PS-49 da cleanup v3.3.
- 8 contadores outcome-oriented em FOR são role-only por convenção pré-existente — D-PS-50 da cleanup v3.3 + D-PS-84 desta sprint.
- `fallback_ratio` é role-only por construção — D-PS-51 da cleanup v3.3.
- 4 chaves de confidence (`{role,skill}.confidence.{lookback_days,min_count}`) ficam fora do endpoint `impact-preview` — D-PS-80 desta sprint.
- Pré-existentes `data.operational.o7`/`o8`/`o9` no JSONB permanecem role-only por retrocompat; campos novos `o7_skill`/`o8_skill`/`o9_skill` são paralelos. Renomeação para `o7_role`/`o8_role`/`o9_role` para simetria nominal completa fica como frente futura (out-of-scope).
- Estimators `auto_assign_family.min_similarity` / `min_score` pré-existentes da cleanup v3.3 cobrem `affected_count` + `projected_event_cost_usd` mas não estão em IMPACT_SOURCES desta sprint (D-PS-41 cleanup v3.3 — famílias só role).

### §0.5 Coordenação com cleanup v3.3 — paralelismo controlado β-light + 10 SUB-PRs

A cleanup v3.3 foi fechada em 2026-05-14. Esta sprint orchestrator-symmetry é sequencial após a cleanup v3.3 ter sido mergeada, mas o paralelismo entre SUB-PRs desta sprint e a janela final de aplicação da cleanup é granular **por SUB-PR**, não por sprint inteira.

Justificativa: precedentes de paralelismo descontrolado (SMS Zenvia Sub-PR19, opus-budget-guard.ts Sub-PR14a) ensinaram que paralelismo amplo aumenta área de atenção operacional do product owner. β-light limita SUB-PRs paralelos a frentes cirurgicamente delimitadas onde a área de overlap com a cleanup v3.3 é demonstravelmente zero.

**Classificação por SUB-PR (10 total — Opção 2 cross-cutting cravada pelo product owner em 2026-05-14):**

| SUB-PR | Frente | Conteúdo | Depende de | Paralelo com cleanup v3.3? |
|---|---|---|---|---|
| 1 | A | §2.1 (tabela `_skill_items` com coluna `needs_review`) | — | sim — SQL puro, zero impacto TS |
| 2 | A | §2.2 (colunas em runs) + §2.3 (validador) + §2.4 (RPC `count_skill_items_by_status`) | — | sim — SQL puro |
| 3 | A | §6.1 (auditoria scripts + E0 obrigatório) + §6.2 (auditoria funções) — documentos | — | sim — sem código, apenas relatórios |
| 4 | A | §2.6 (rename) + §3.4 (atualização de call sites TS + corpos de funções SQL) | cleanup v3.3 mergeada + SUB-PR 3 | não — toca arquivos compartilhados com cleanup |
| 5 | A | §3.1 (refator função de descoberta — Cenário A ou B conforme E0; caller já em await) | SUB-PR 4 + evidence E0 da §6.1 | não — toca pipeline compartilhado |
| 6 | A | §3.2 (insertFOSkillItem em call sites) + §3.3 (3 finalizers) | SUB-PR 5 | não — dependência interna |
| 7 | C | §3.5 (queries novas em `lib/admin/limiares/queries.ts`) + §4.1 (refator endpoint `impact-preview` para padrão Limiares `pg.Pool` + Redis; remoção do cache in-process) + enriquecimento dos painéis Limiares 1 e 8 | cleanup v3.3 mergeada + SUB-PR 6 | não — depende de FOSI populada (SUB-PR 6) e do schema pós-cleanup |
| 8 | C | §5.1 (extração de `EditModal` + `ImpactPreview` da `page.tsx` + integração payload novo no componente extraído) | cleanup v3.3 mergeada + SUB-PR 7 | não — toca componente do cleanup |
| 9 | B | §3.6 (expansão de `aggregateDayData` + sub-função `aggregatePipelineOrchestratorMetrics` cobrindo `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` no JSONB) + backfill obrigatório via `/api/admin/backfill-dashboard-summary` | SUB-PR 6 (FOSI populada) | não — depende de FOSI |
| 10 | B | §5.2 (refator UI dos painéis 2.7 O7 / 2.8 O8 / 2.9 O9 para séries paralelas role+skill) + ajuste KPI 2.1 com novo card "Items processados pelo pipeline" | SUB-PR 9 (campos materializados no JSONB) | não — toca UI de painéis pré-existentes |

**Resultado:**

- **Fase 1 (paralela com cleanup v3.3):** SUB-PRs 1, 2, 3 — ~1,5 dia em paralelo
- **Fase 2 (sequencial após cleanup v3.3):** SUB-PR 4 → SUB-PR 5 → SUB-PR 6 — Frente A completa (~3,5 dias)
- **Fase 3 (paralela entre frentes B e C, após SUB-PR 6):** SUB-PRs 7 + 9 em paralelo (~3 dias máximo de janela)
- **Fase 4 (sequencial dentro de cada frente):** SUB-PR 8 (depois de 7) + SUB-PR 10 (depois de 9) — em paralelo entre si (~2 dias máximo de janela)

Janela total estimada: **12-14 dias úteis**. Detalhamento de estimativas em §8.

---

## §1 Estado pós-implantação

Ao final desta sprint:

1. Toda skill processada pelo pipeline de vagas (fluxos A, B, C) em caminho curated deixa rastro item-por-item em `function_orchestrator_skill_items`, com estágio do pipeline real, confidence, canonical_skill_id resolvido, status final e sinal `needs_review` propagado do skill-type-guard upstream. Skills em caminho quarantined permanecem invisíveis (D-PS-83).
2. Contadores agregados de skill populam `function_orchestrator_runs` em 5 colunas stage-oriented (`skills_extracted`, `skills_reused`, `skills_pending_created`, `skills_gate_rejected`, `skills_failed`).
3. Tabela `function_orchestrator_items` foi renomeada para `function_orchestrator_role_items` em toda a base de código, schema e corpos de funções dependentes (incluindo `fn_recompute_jcr_confidence_median`, `reset_taxonomy_core` e demais funções identificadas na §6.2).
4. Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` migrou de cache in-process Map para **padrão Limiares com `pg.Pool` direto + Redis TTL 300s** (D-PS-86 substitui D-PS-68 da cleanup v3.3). Retorna telemetria empírica para 22 chaves operacionais (20 cobertas pelo IMPACT_SOURCES + 2 `auto_assign_family.*` role-only cobertas só pelos estimators pré-existentes). Para 4 chaves de confidence (out-of-scope desta sprint), endpoint retorna `sample_status='unsupported_in_v1'` mantendo HTTP 200. Campos pré-existentes da cleanup v3.3 (`affected_count`, `projected_event_cost_usd`, `cost_is_fallback`) preservados onde aplicável (8 chaves cobertas por estimators).
5. `EditModal` e `ImpactPreview`, antes inline em `app/admin/pipeline-config/page.tsx`, foram extraídos para arquivos próprios em `components/admin/pipeline-config/EditModal.tsx` e `components/admin/pipeline-config/ImpactPreview.tsx` (D-PS-93). Modal exibe dois blocos lado a lado na seção "Impacto estimado": à esquerda a caixa qualitativa pré-existente (painéis afetados + texto explicativo + horizonte de acompanhamento + custo Opus para 8 chaves cobertas por estimator), à direita o componente `ImpactPreview` expandido (tabela de contagem + histograma de distribuição + seletor de janela 7d/30d/90d).
6. `lib/admin/limiares/queries.ts` ganhou queries novas alimentando o impact-preview: distribuição de confidence em FORI, distribuição em FOSI, agregação sobre `opus_arbitration_outcomes`. Como subproduto natural, painéis Limiares 1 (Hard Gate) e 8 (creation_confidence) foram enriquecidos com séries via FOSI/FORI (D-PS-91). Painéis 2, 3, 4, 5, 6, 7, 9, 10 mantidos sem mudança.
7. `lib/admin/dashboard-day-aggregator.ts` ganhou sub-função `aggregatePipelineOrchestratorMetrics` chamada de dentro de `aggregateDayData()`. Família nova de campos no JSONB de `dashboard_daily_summary.data`: `operational.o7_skill`, `operational.o8_skill`, `operational.o9_skill`, `operational.items_processed` (D-PS-92). Pré-existentes `operational.o7`/`o8`/`o9` role-only mantidos intactos por retrocompat. Backfill via `/api/admin/backfill-dashboard-summary` regenerou os 26 dias rolling com a família nova preenchida.
8. Painéis 2.7 (O7 — Funções Novas / drift), 2.8 (O8 — Caminho de resolução), 2.9 (O9 — Saúde do pipeline) do dashboard global passaram a exibir séries paralelas role+skill: painel 2.7 ganha bloco "Habilidades Novas" ao lado de "Funções Novas"; painel 2.8 ganha barra stacked paralela para skill ao lado da barra de role; painel 2.9 ganha bloco de status skill ao lado do status role. Card 2.1 ganha indicador novo "Items processados pelo pipeline" agregando role + skill no header com breakdown no tooltip (D-PS-94).
9. D-PS-33 da cleanup v3.3 é resolvida por esta sprint via expansão do endpoint pré-existente + extração de componentes + enriquecimento de painéis Limiares. D-PS-40 da cleanup é resolvida. D-PS-43, D-PS-49, D-PS-50, D-PS-51, D-PS-64, D-PS-66 da cleanup são reafirmadas sem modificação. D-PS-65, D-PS-67, D-PS-69 da cleanup são reconhecidas como precedentes operacionais. **D-PS-68 da cleanup v3.3 é REVOGADA** (cache in-process Map TTL 30s substituído pelo padrão Limiares `pg.Pool` + Redis).
10. `confidence_median` em `job_canonical_skills` continua sendo populado por `fn_jps_recompute_jcs` (sem mudança). `confidence_median` em `job_canonical_roles` continua sendo populado por `fn_recompute_jcr_confidence_median`, cujo corpo é atualizado **mecanicamente** apenas para referenciar `function_orchestrator_role_items` pós-rename — sem mudança de fonte, lógica, parâmetros ou semântica do cálculo.
11. A terminologia `projected_event_cost_usd` (D-PS-67 da cleanup v3.3 — "evento único", NÃO recorrência mensal) é preservada no payload do endpoint refatorado. UI mantém o rótulo canônico "Custo Opus projetado (evento único)" na caixa qualitativa quando o estimator pré-existente atender a chave editada (D-PS-87).

---

## §2 Migrations SQL

### §2.1 — `function_orchestrator_skill_items` (tabela nova)

**Arquivo:** `01_function_orchestrator_skill_items.sql`
**Atende:** S-ORCH-1 (rastro item-por-item para skills).
**SUB-PR:** 1.

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS function_orchestrator_skill_items (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id uuid NOT NULL REFERENCES function_orchestrator_runs(id) ON DELETE CASCADE,
  job_posting_id uuid NOT NULL REFERENCES job_postings(id) ON DELETE CASCADE,

  -- Origem
  raw_skill_name text,

  -- Resolução
  canonical_skill_id uuid REFERENCES job_canonical_skills(id) ON DELETE SET NULL,
  canonical_status text,

  -- Trajetória pelo pipeline (paths reais de skill — ver D-PS-79)
  skill_pipeline_stage text,
  skill_confidence numeric,
  status text NOT NULL,

  -- Erros
  error_type text,
  error_detail text,

  -- Dict match (rastro de qual alias casou)
  dict_match boolean NOT NULL DEFAULT false,
  dict_match_source jsonb,

  -- Sinal de revisão (paridade com `needsReview` que skill-type-guard.ts §3.15 da cleanup v3.3
  -- propaga end-to-end via D-PS-65 + D-PS-69). Campo opcional, NULL quando guard não foi aplicado
  -- ou retornou needsReview=false. Quando true, item entra na fila de revisão Opus correspondente
  -- ao canonical_skill_id (se houver) — sem ramificação adicional aqui.
  needs_review boolean NOT NULL DEFAULT false,

  -- Temporal
  processed_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT NOW(),

  CONSTRAINT foski_canonical_status_check
    CHECK (canonical_status IS NULL OR canonical_status IN ('active', 'pending')),

  -- Enum de stage reflete os paths reais de discoverAndLinkSkills + hard gate upstream.
  -- Difere intencionalmente do enum de role (D-PS-79).
  CONSTRAINT foski_pipeline_stage_check
    CHECK (skill_pipeline_stage IS NULL OR skill_pipeline_stage IN (
      'slug_match',           -- lookup direto em job_canonical_skills.slug
      'alias_match',           -- lookup via lookup_canonical_skill_by_normalized_alias em taxonomy_relations
      'llm_new',               -- skill nova, INSERT em job_canonical_skills com status='pending'
      'race_recovered',        -- INSERT colidiu, resolve_active_canonical_by_slug recuperou
      'gate_rejected',         -- rejeitada pelo hard gate ANTES de discoverAndLinkSkills (skill.hard_gate.min_confidence)
      'fallback_error'         -- erro durante extração ou persistência
    )),

  CONSTRAINT foski_status_check
    CHECK (status IN ('success', 'reused', 'created_pending', 'gate_rejected', 'failed')),

  CONSTRAINT foski_error_type_check
    CHECK (error_type IS NULL OR error_type IN (
      'llm_timeout', 'batch_output_parse_error', 'item_output_parse_error',
      'payload_parse_error', 'parser_bug', 'db_error', 'rpc_error'
    ))
);

CREATE INDEX IF NOT EXISTS idx_foski_run_id
  ON function_orchestrator_skill_items (run_id);

CREATE INDEX IF NOT EXISTS idx_foski_job_posting_id
  ON function_orchestrator_skill_items (job_posting_id);

CREATE INDEX IF NOT EXISTS idx_foski_canonical_skill_id
  ON function_orchestrator_skill_items (canonical_skill_id)
  WHERE canonical_skill_id IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_foski_status
  ON function_orchestrator_skill_items (status);

CREATE INDEX IF NOT EXISTS idx_foski_skill_confidence
  ON function_orchestrator_skill_items (skill_confidence)
  WHERE skill_confidence IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_foski_pipeline_stage
  ON function_orchestrator_skill_items (skill_pipeline_stage);

CREATE INDEX IF NOT EXISTS idx_foski_needs_review
  ON function_orchestrator_skill_items (needs_review)
  WHERE needs_review = true;

COMMENT ON TABLE function_orchestrator_skill_items IS
'Rastro item-por-item de skills processadas pelo caminho curated do pipeline de curadoria de vagas (fluxos A, B, C). Skills processadas no caminho quarantined via legacy mapSkillsToCanonical NÃO aparecem aqui (D-PS-83 da sprint orchestrator). Enum skill_pipeline_stage reflete os 6 paths reais de discoverAndLinkSkills + hard gate upstream, e difere intencionalmente do enum role (D-PS-79). NÃO alimenta job_canonical_skills.confidence_median — esse cálculo permanece em fn_jps_recompute_jcs a partir de job_posting_skills curated (D-PS-49 da cleanup v3.3). Coluna needs_review propaga sinal de D-PS-65/D-PS-69 da cleanup v3.3 (skill-type-guard.ts).';

COMMENT ON COLUMN function_orchestrator_skill_items.needs_review IS
'Sinal upstream do skill-type-guard.ts (§3.15 da cleanup v3.3, D-PS-65) e dos resolutores de alias (D-PS-69). True indica que algum item foi normalizado (ex: alias PT-BR resolvido para canônico EN minúsculo) — drift detectado do prompt LLM ou input fora de template. Conectividade com fila Opus humana sobre canonical_skill_id é responsabilidade do circuito de revisão pré-existente.';

COMMIT;
```

**Notas:**
- Schema **estruturalmente idêntico** ao de `function_orchestrator_role_items` (após rename de §2.6), exceto:
  - Campo `title_original` → `raw_skill_name` (mais explicativo)
  - Enum `skill_pipeline_stage` próprio (6 valores que refletem paths de skill, não os 6 de role)
  - Coluna nova `needs_review` para propagar sinal de D-PS-65/D-PS-69 da cleanup v3.3 (não existe equivalente em role, porque role não tem `skill_type` — assimetria documentada em D-PS-65 da cleanup v3.3)
- Enum `status` é simétrico ao de role (5 valores: success, reused, created_pending, gate_rejected, failed). Diferença com role: skill tem `gate_rejected` explícito; role não tem (role nunca é rejeitada pelo gate — vai para fallback_error).
- **NÃO há trigger novo** para popular `job_canonical_skills.confidence_median`. Ver D-PS-74.

---

### §2.2 — `function_orchestrator_runs` ADD colunas de skill

**Arquivo:** `02_function_orchestrator_runs_skill_columns.sql`
**Atende:** S-ORCH-2.
**SUB-PR:** 2.

```sql
BEGIN;

ALTER TABLE function_orchestrator_runs
  ADD COLUMN IF NOT EXISTS skills_extracted INT NOT NULL DEFAULT 0;

ALTER TABLE function_orchestrator_runs
  ADD COLUMN IF NOT EXISTS skills_reused INT NOT NULL DEFAULT 0;

ALTER TABLE function_orchestrator_runs
  ADD COLUMN IF NOT EXISTS skills_pending_created INT NOT NULL DEFAULT 0;

ALTER TABLE function_orchestrator_runs
  ADD COLUMN IF NOT EXISTS skills_gate_rejected INT NOT NULL DEFAULT 0;

ALTER TABLE function_orchestrator_runs
  ADD COLUMN IF NOT EXISTS skills_failed INT NOT NULL DEFAULT 0;

COMMENT ON COLUMN function_orchestrator_runs.skills_extracted IS
'Count total de skills processadas no run, agregando todos os status (reused + created_pending + gate_rejected + failed). Stage-oriented — NÃO equivalente a `curated` ou `total` de role (que são outcome-oriented). Ver D-PS-84.';

COMMENT ON COLUMN function_orchestrator_runs.skills_reused IS
'Skills resolvidas para canonicals já existentes (slug_match + alias_match + race_recovered).';

COMMENT ON COLUMN function_orchestrator_runs.skills_pending_created IS
'Skills que geraram criação de novo canonical em status=pending (path llm_new).';

COMMENT ON COLUMN function_orchestrator_runs.skills_gate_rejected IS
'Skills rejeitadas pelo hard gate (skill.hard_gate.min_confidence) ANTES de discoverAndLinkSkills ser chamada.';

COMMENT ON COLUMN function_orchestrator_runs.skills_failed IS
'Skills com erro em algum estágio do pipeline (fallback_error).';

COMMIT;
```

---

### §2.3 — Validação de schema simétrico entre `_role_items` e `_skill_items`

**Arquivo:** `03_validate_orchestrator_symmetry.sql`
**Atende:** gate operacional antes do SUB-PR 4.
**SUB-PR:** 2.

**Decisão sobre validação de `pipeline_stage`:** os enums de stage role e skill **divergem intencionalmente** (D-PS-79). O validador NÃO compara stage — apenas `status` e `error_type`, que são genuinamente simétricos.

**Decisão sobre nome da tabela role:** este validador roda no SUB-PR 2, **antes** do rename do SUB-PR 4. Portanto referencia o nome antigo `function_orchestrator_items`. Validador pós-rename é coberto por S10 da §7.2.

```sql
BEGIN;

DO $$
DECLARE
  v_role_status_enum text;
  v_skill_status_enum text;
  v_role_error_enum text;
  v_skill_error_enum text;
BEGIN
  -- Status enum
  SELECT pg_get_constraintdef(oid) INTO v_role_status_enum
  FROM pg_constraint
  WHERE conname = 'function_orchestrator_items_status_check';

  SELECT pg_get_constraintdef(oid) INTO v_skill_status_enum
  FROM pg_constraint
  WHERE conname = 'foski_status_check';

  IF v_role_status_enum IS NULL THEN
    RAISE EXCEPTION 'Constraint function_orchestrator_items_status_check ausente — rename ainda não rodou?';
  END IF;
  IF v_skill_status_enum IS NULL THEN
    RAISE EXCEPTION 'Constraint foski_status_check ausente — §2.1 não rodou?';
  END IF;
  -- Status enums devem ser SEMANTICAMENTE iguais (mesmos 5 valores).
  -- Compara via extração dos valores entre parênteses.
  IF regexp_replace(v_role_status_enum, '[^a-z_,]', '', 'g')
     != regexp_replace(v_skill_status_enum, '[^a-z_,]', '', 'g') THEN
    RAISE EXCEPTION 'Enum status divergente. Role: %, Skill: %', v_role_status_enum, v_skill_status_enum;
  END IF;

  -- Error_type enum
  SELECT pg_get_constraintdef(oid) INTO v_role_error_enum
  FROM pg_constraint
  WHERE conname = 'function_orchestrator_items_error_type_check';

  SELECT pg_get_constraintdef(oid) INTO v_skill_error_enum
  FROM pg_constraint
  WHERE conname = 'foski_error_type_check';

  -- Error_type permite divergência (skill pode ter rpc_error adicional para race recovery)
  -- então apenas RAISE NOTICE para visibilidade, sem abortar.
  IF v_role_error_enum IS NOT NULL AND v_skill_error_enum IS NOT NULL THEN
    IF v_role_error_enum != v_skill_error_enum THEN
      RAISE NOTICE 'Enum error_type divergente (esperado — skill tem rpc_error adicional). Role: %, Skill: %',
        v_role_error_enum, v_skill_error_enum;
    END IF;
  END IF;

  RAISE NOTICE 'Validação de simetria OK.';
END $$;

COMMIT;
```

---

### §2.4 — RPC `count_skill_items_by_status` (obrigatória para finalizers)

**Arquivo:** `04_rpc_count_skill_items_by_status.sql`
**Atende:** §3.3 — evita truncamento de paginação Supabase 1000 rows (LK-CBO-31).
**SUB-PR:** 2.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION count_skill_items_by_status(p_run_id uuid)
RETURNS TABLE(status text, c bigint)
LANGUAGE sql
STABLE
AS $$
  SELECT status, COUNT(*) AS c
  FROM function_orchestrator_skill_items
  WHERE run_id = p_run_id
  GROUP BY status;
$$;

COMMENT ON FUNCTION count_skill_items_by_status(uuid) IS
'Agregação por status para finalizers de run. Obrigatória para evitar truncamento de paginação Supabase em runs com >1000 skills (LK-CBO-31). Substitui .from("function_orchestrator_skill_items").select("status").eq("run_id",runId) no caller TS.';

COMMIT;
```

---

### §2.5 — (reservado para uso futuro)

---

### §2.6 — Rename `function_orchestrator_items` → `function_orchestrator_role_items`

**Arquivo:** `06_rename_function_orchestrator_items.sql`
**Atende:** S-ORCH-6.
**SUB-PR:** 4 (sequencial após cleanup v3.3 + SUB-PR 3).
**Pré-requisito:** SUB-PR 3 finalizado — `AUDIT-rename-orchestrator-<data>.md` lista exata de objetos a renomear/atualizar.

```sql
BEGIN;

-- 1. RENOMEAR TABELA
ALTER TABLE function_orchestrator_items
  RENAME TO function_orchestrator_role_items;

-- 2. RENOMEAR CONSTRAINTS
ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_pkey
  TO function_orchestrator_role_items_pkey;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_status_check
  TO function_orchestrator_role_items_status_check;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_pipeline_stage_check
  TO function_orchestrator_role_items_pipeline_stage_check;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_canonical_status_check
  TO function_orchestrator_role_items_canonical_status_check;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_error_type_check
  TO function_orchestrator_role_items_error_type_check;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_action_required_check
  TO function_orchestrator_role_items_action_required_check;

-- FKs (lista exata vem da §6.2)
-- Padrão esperado: function_orchestrator_items_run_id_fkey → function_orchestrator_role_items_run_id_fkey
-- Padrão esperado: function_orchestrator_items_job_posting_id_fkey → function_orchestrator_role_items_job_posting_id_fkey
-- Padrão esperado: function_orchestrator_items_canonical_role_id_fkey → function_orchestrator_role_items_canonical_role_id_fkey

-- 3. RENOMEAR INDICES (lista exata vem da §6.2 query E1)
-- Exemplos esperados (validar nomes reais):
ALTER INDEX IF EXISTS function_orchestrator_items_run_id_idx
  RENAME TO function_orchestrator_role_items_run_id_idx;

ALTER INDEX IF EXISTS function_orchestrator_items_job_posting_id_idx
  RENAME TO function_orchestrator_role_items_job_posting_id_idx;

ALTER INDEX IF EXISTS function_orchestrator_items_canonical_role_id_idx
  RENAME TO function_orchestrator_role_items_canonical_role_id_idx;

ALTER INDEX IF EXISTS function_orchestrator_items_status_idx
  RENAME TO function_orchestrator_role_items_status_idx;

ALTER INDEX IF EXISTS function_orchestrator_items_pipeline_stage_idx
  RENAME TO function_orchestrator_role_items_pipeline_stage_idx;

-- 4. RENOMEAR TRIGGERS
ALTER TRIGGER trg_foi_jcr_confidence_insert
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_insert;

ALTER TRIGGER trg_foi_jcr_confidence_update
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_update;

ALTER TRIGGER trg_foi_jcr_confidence_delete
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_delete;

-- 5. ATUALIZAR CORPOS DE FUNÇÕES QUE REFERENCIAM O NOME ANTIGO
-- Ground truth confirmou que fn_recompute_jcr_confidence_median usa
-- FROM function_orchestrator_items foi. Sem REPLACE, quebra pós-rename.
-- IMPORTANTE: este CREATE OR REPLACE é MECÂNICO — apenas troca o identificador
-- da tabela. Lógica, parâmetros, fonte e semântica são idênticos ao corpo
-- atual da função. D-PS-49 da cleanup v3.3 cravou que "sprint orchestrator
-- NÃO TOCA o circuito de confidence_median". Esta atualização é parte do
-- rename, não toque do circuito.

CREATE OR REPLACE FUNCTION fn_recompute_jcr_confidence_median(p_canonical_role_id uuid)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
  v_lookback_days INT;
  v_min_count INT;
BEGIN
  IF p_canonical_role_id IS NULL THEN RETURN; END IF;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.confidence.lookback_days'), 120)
    INTO v_lookback_days;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.confidence.min_count'), 5)
    INTO v_min_count;

  WITH new_median AS (
    SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY fori.confidence) AS m
    FROM function_orchestrator_role_items fori           -- renomeado de function_orchestrator_items
    JOIN function_orchestrator_runs fo ON fo.id = fori.run_id
    WHERE fori.canonical_role_id = p_canonical_role_id
      AND fori.confidence IS NOT NULL
      AND fo.started_at >= NOW() - make_interval(days => v_lookback_days)
    HAVING COUNT(*) >= v_min_count
  )
  UPDATE job_canonical_roles jcr
  SET confidence_median = nm.m
  FROM new_median nm
  WHERE jcr.id = p_canonical_role_id
    AND jcr.status IN ('active', 'pending');
END;
$$;

-- Demais funções identificadas em §6.2 que referenciam 'function_orchestrator_items' por nome
-- devem ser atualizadas no mesmo bloco. Lista mínima conhecida via ground truth:
--   - reset_taxonomy_core (DELETE FROM function_orchestrator_items)
--   - maintenance_phase_1 (a verificar via §6.2)
--   - process_opus_create_new (a verificar via §6.2)
--   - quaisquer outras detectadas em §6.2 com query 'SELECT proname FROM pg_proc WHERE prosrc LIKE %function_orchestrator_items%'

-- Para CADA função adicional encontrada na §6.2, adicionar bloco
-- CREATE OR REPLACE FUNCTION ... com o nome novo substituído no corpo.
-- A regra de "atualização mecânica do identificador, sem toque do circuito" vale
-- para todas elas.

COMMIT;
```

**Notas críticas:**

- **Lista exata de objetos a renomear vem da §6.2.** Esta migration é um TEMPLATE — o conteúdo final depende do que a auditoria retornar. SUB-PR 4 só inicia após SUB-PR 3 fechado.
- **Funções de trigger `fn_jcr_confidence_median_insert/update/delete` NÃO precisam ser renomeadas** (ground truth confirma que NÃO referenciam `function_orchestrator_items` por nome — apenas chamam `fn_recompute_jcr_confidence_median(canonical_role_id)`).
- **Janela atual:** ground truth confirmou 0 rows em produção nos últimos 30 dias. Rename é de baixíssimo risco.
- **Pós-migration:** todos os call sites TS precisam ser atualizados na mesma sprint (§3.4). Migration sozinha quebra a aplicação.
- **D-PS-49 da cleanup v3.3 reafirmada:** atualização do identificador `function_orchestrator_items → function_orchestrator_role_items` no corpo de funções é mecânica de rename. Lógica, parâmetros lidos de `pipeline_config`, semântica de cálculo (`PERCENTILE_CONT(0.5)`), filtros de `status IN ('active', 'pending')`, fonte de dados (`function_orchestrator_role_items.confidence`) — tudo permanece idêntico.

---

### §2.7 — (reservado para uso futuro)

---

## §3 TypeScript backend

### §3.1 — Refator da função de descoberta de skills + conversão do caller

**Arquivo da função:** `lib/pipeline/ingest-job-and-discover-skills.ts` (confirmar nome real via E0 — ver §6.1)
**Arquivo do caller:** `lib/pipeline/persist-curation.ts` (ou caminho equivalente — confirmar em E0)
**Atende:** S-ORCH-3.
**SUB-PR:** 5.
**Pré-requisito obrigatório:** evidence E0 da §6.1 — confirma nome real da função no codebase (`discoverAndLinkSkills` vs `safeDiscoverAndLinkSkills`) e sua estrutura atual (retorna `DiscoverResult` ou void; é wrapper ou função única) — ver D-PS-88.

#### §3.1.0 — Estado atual: dois cenários possíveis (resolução via E0) + caller já em await (D-PS-97)

A cleanup v3.3 (D-PS-40) referencia `safeDiscoverAndLinkSkills` retornando void. Sessões anteriores de spec referenciavam `discoverAndLinkSkills` retornando `result { skills_reused, skills_pending_created }`. Antes de iniciar o refator, evidence E0 da §6.1 confirma qual cenário vige no codebase atual:

**Cenário A — função única `discoverAndLinkSkills` já retorna `result`:**
- Refator desta sprint EXPANDE o tipo de retorno para `DiscoverResultDetailed` (estrutura nova abaixo)

**Cenário B — função `safeDiscoverAndLinkSkills` é wrapper externo void que internamente chama `discoverAndLinkSkills` (que pode ou não retornar `result`):**
- Refator desta sprint EXPANDE retorno em AMBAS as camadas: função interna devolve `DiscoverResultDetailed`; wrapper passa a retornar o mesmo

**Diretriz crítica (D-PS-97 — DV-5 cravado em 2026-05-14):** ground truth via E0d revelou que o caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ EM `await safeDiscoverAndLinkSkills(supabase, normalized);` — não é mais fire-and-forget. O comentário inline `M60 — await both; emit events on failure` indica que houve refator anterior (sprint M60) que converteu o caller para await mas a D-PS-64 da cleanup v3.3 não foi atualizada. **Logo, esta sprint NÃO converte caller de fire-and-forget para await — caller já está em await. Refator é exclusivamente: (a) expandir tipo de retorno do wrapper de `Promise<void>` para `Promise<DiscoverResultDetailed>`; (b) usar o valor retornado no caller para empurrar para `insertFOSkillItemsBatch`.**

Ambos os cenários convergem para o mesmo resultado funcional. A diferença é apenas qual arquivo recebe a expansão de tipo de retorno e em quantas camadas. O ground truth via E0 determina isso antes da implementação começar.

**Para fins desta spec, daqui em diante uso "função de descoberta" quando o nome real não importa, e cito ambos os nomes (`discoverAndLinkSkills` / `safeDiscoverAndLinkSkills`) quando o nome importa para o reader entender o cenário.**

#### §3.1.1 — Expandir tipo de retorno

**Mudança 1 — exportar tipos da função de descoberta (arquivo de implementação):**

```typescript
// Tipos novos (export do arquivo da função de descoberta)

export type SkillProcessingDetail = {
  raw_skill_name: string;
  canonical_skill_id: string | null;
  canonical_status: 'active' | 'pending' | null;
  skill_pipeline_stage:
    | 'slug_match'
    | 'alias_match'
    | 'llm_new'
    | 'race_recovered'
    | 'gate_rejected'
    | 'fallback_error';
  skill_confidence: number | null;
  status:
    | 'success'
    | 'reused'
    | 'created_pending'
    | 'gate_rejected'
    | 'failed';
  error_type?:
    | 'llm_timeout'
    | 'batch_output_parse_error'
    | 'item_output_parse_error'
    | 'payload_parse_error'
    | 'parser_bug'
    | 'db_error'
    | 'rpc_error';
  error_detail?: string;
  dict_match: boolean;
  dict_match_source?: object;
  needs_review: boolean;       // Propagado de skill-type-guard.ts (D-PS-65/D-PS-69 cleanup v3.3)
  processed_at: string;
};

export type DiscoverResultDetailed = {
  aggregate: {
    extracted: number;
    reused: number;
    pending_created: number;
    gate_rejected: number;
    failed: number;
  };
  items: SkillProcessingDetail[];
};
```

**Mudança 2 — instrumentar paths internos da função:**

Cada path interno passa a empurrar um `SkillProcessingDetail` no array `items`. O sinal `needs_review` vem do skill-type-guard pré-existente (§3.15 da cleanup v3.3): quando o LLM emite alias PT-BR ou variante de caixa, o guard normaliza E marca `needsReview=true` em `LLMSkillItem.needsReview` (D-PS-69). Esse sinal **propaga end-to-end** até `function_orchestrator_skill_items.needs_review` por meio do tipo `RawSkill` consumido pela função de descoberta.

- Path `slug_match` (lookup em `job_canonical_skills.slug` retornou hit):
  - `status='reused'`, `skill_pipeline_stage='slug_match'`, `canonical_status` lido do registro encontrado, `dict_match=false`, `needs_review` copiado do input
- Path `alias_match` (RPC `lookup_canonical_skill_by_normalized_alias` retornou hit):
  - `status='reused'`, `skill_pipeline_stage='alias_match'`, `dict_match=true`, `dict_match_source={ source: 'taxonomy_relations', matched_term: <normalized> }`, `needs_review` copiado do input
- Path `llm_new` (INSERT bem-sucedido com `status='pending'`):
  - `status='created_pending'`, `skill_pipeline_stage='llm_new'`, `canonical_status='pending'`, `dict_match=false`, `needs_review` copiado do input
- Path `race_recovered` (INSERT falhou, RPC `resolve_active_canonical_by_slug` recuperou):
  - `status='reused'`, `skill_pipeline_stage='race_recovered'`, `dict_match=false`, `needs_review` copiado do input
- Path `fallback_error` (catch externo capturou exceção catastrófica):
  - `status='failed'`, `skill_pipeline_stage='fallback_error'`, `error_type` mapeado via `classifyError(err)`, `error_detail=err.message`, `needs_review=false`

`aggregate` é derivado do array `items` ao final (consistência garantida por construção):

```typescript
const aggregate = {
  extracted: items.length,
  reused: items.filter(i => i.status === 'reused').length,
  pending_created: items.filter(i => i.status === 'created_pending').length,
  gate_rejected: items.filter(i => i.status === 'gate_rejected').length,
  failed: items.filter(i => i.status === 'failed').length,
};

return { aggregate, items };
```

**Nota sobre `gate_rejected`:** o hard gate de skill (`skill.hard_gate.min_confidence`) é aplicado **antes** da função de descoberta ser chamada, em `persist-curation.ts` (§3.2 da cleanup v3.3). Skills que não passam no gate **não chegam** nesta função. Portanto, o caller também precisa registrar essas rejeições — ver Mudança 3 abaixo. Itens com `status='gate_rejected'` na tabela `function_orchestrator_skill_items` são inseridos pelo caller, não pela função.

#### §3.1.2 — Caller: capturar o retorno expandido (DV-5 cravado — caller já está em await)

**Mudança 3 — caller passa a USAR o retorno do wrapper, não converter de fire-and-forget para await:**

Estado atual confirmado por E0d: `lib/pipeline/persist-curation/persist-fn.ts:410` já tem `await safeDiscoverAndLinkSkills(supabase, normalized);` — retorno descartado porque a função era `Promise<void>`. O comentário inline `M60 — await both; emit events on failure` documenta que o caller foi convertido para await em sprint anterior (M60). A D-PS-64 da cleanup v3.3 referencia comportamento fire-and-forget desatualizado.

Logo, o refator desta sprint é exclusivamente:
- (a) expandir tipo de retorno do wrapper de `Promise<void>` para `Promise<DiscoverResultDetailed>`
- (b) usar o valor retornado no caller para empurrar para `insertFOSkillItemsBatch`

**Cenário A (função única `discoverAndLinkSkills`):**

```typescript
// ANTES (caller já em await; retorno result possivelmente lido para aggregate counts):
const discoverResult = await discoverAndLinkSkills(supabase, jobPostingId, skillsPassedGate);
// discoverResult tinha shape { skills_reused, skills_pending_created } — usado parcialmente

// DEPOIS (caller usa retorno expandido):
let skillDetail: DiscoverResultDetailed;
try {
  skillDetail = await discoverAndLinkSkills(supabase, jobPostingId, skillsPassedGate);
} catch (err) {
  // Manter comportamento non-blocking — erro de discovery não bloqueia persistência da vaga.
  // Mas registrar como item failed para visibilidade no orchestrator.
  console.error('[persist-curation] discoverAndLinkSkills failed (non-blocking):', err);
  skillDetail = {
    aggregate: { extracted: 0, reused: 0, pending_created: 0, gate_rejected: 0, failed: skillsPassedGate.length },
    items: skillsPassedGate.map(s => ({
      raw_skill_name: s.label,
      canonical_skill_id: null,
      canonical_status: null,
      skill_pipeline_stage: 'fallback_error' as const,
      skill_confidence: s.confidence,
      status: 'failed' as const,
      error_type: classifyError(err),
      error_detail: err instanceof Error ? err.message : String(err),
      dict_match: false,
      needs_review: s.needsReview ?? false,
      processed_at: new Date().toISOString(),
    })),
  };
}

// Adicionar items das skills rejeitadas pelo hard gate ao detail
const gateRejectedItems: SkillProcessingDetail[] = skillsRejectedByGate.map(s => ({
  raw_skill_name: s.label,
  canonical_skill_id: null,
  canonical_status: null,
  skill_pipeline_stage: 'gate_rejected' as const,
  skill_confidence: s.confidence,
  status: 'gate_rejected' as const,
  dict_match: false,
  needs_review: s.needsReview ?? false,
  processed_at: new Date().toISOString(),
}));

const allItems = [...skillDetail.items, ...gateRejectedItems];

// Persistir no orchestrator (ver §3.2)
await insertFOSkillItemsBatch(supabase, runId, jobPostingId, allItems);
```

**Cenário B (wrapper `safeDiscoverAndLinkSkills` + função interna `discoverAndLinkSkills`) — confirmado em produção por E0d:**

Refator em 2 camadas:

1. **Função interna `discoverAndLinkSkills`** continua sendo responsável pela lógica; agora retorna `DiscoverResultDetailed` (Mudança 1 + Mudança 2 acima).
2. **Wrapper `safeDiscoverAndLinkSkills`** deixa de ser `Promise<void>` e passa a propagar o `DiscoverResultDetailed` da função interna. Pseudocódigo (em `lib/pipeline/persist-curation/skill-mapper.ts`):

```typescript
// ANTES: assinatura void
// export async function safeDiscoverAndLinkSkills(
//   supabase: SupabaseClient,
//   normalized: NormalizedJobPosting,
// ): Promise<void> { ... }

// DEPOIS: assinatura tipada com DiscoverResultDetailed
export async function safeDiscoverAndLinkSkills(
  supabase: SupabaseClient,
  normalized: NormalizedJobPosting,
): Promise<DiscoverResultDetailed> {
  try {
    return await discoverAndLinkSkills(supabase, normalized.jobPostingId, normalized.skills);
  } catch (err) {
    console.error('[safeDiscoverAndLinkSkills] failed (non-blocking):', err);
    return {
      aggregate: { extracted: 0, reused: 0, pending_created: 0, gate_rejected: 0, failed: normalized.skills.length },
      items: normalized.skills.map(s => ({
        raw_skill_name: s.label,
        canonical_skill_id: null,
        canonical_status: null,
        skill_pipeline_stage: 'fallback_error' as const,
        skill_confidence: s.confidence,
        status: 'failed' as const,
        error_type: classifyError(err),
        error_detail: err instanceof Error ? err.message : String(err),
        dict_match: false,
        needs_review: s.needsReview ?? false,
        processed_at: new Date().toISOString(),
      })),
    };
  }
}
```

E em `lib/pipeline/persist-curation/persist-fn.ts:410`:

```typescript
// ANTES (M60 — caller já em await mas retorno descartado):
await safeDiscoverAndLinkSkills(supabase, normalized);

// DEPOIS (caller captura retorno):
const skillDetail = await safeDiscoverAndLinkSkills(supabase, normalized);

// Coletar items rejeitados pelo hard gate (vindos de outro ponto do persist-fn)
const gateRejectedItems: SkillProcessingDetail[] = skillsRejectedByGate.map(s => ({ /* ... */ }));

const allItems = [...skillDetail.items, ...gateRejectedItems];

// Persistir no orchestrator (ver §3.2)
await insertFOSkillItemsBatch(supabase, runId, jobPostingId, allItems);
```

`try/catch` no caller é REDUNDANTE (o wrapper já trata), mas pode ser mantido como defesa em profundidade.

**Decisão de spec:** Cenário B preserva 2 camadas (encapsulamento de erro fica no wrapper, lógica fica na função interna). Cenário A é mais enxuto. Ambos são tecnicamente válidos. Implementador segue o que o codebase já tem — não há reorganização de camadas embutida nesta sprint.

**Helpers a validar antes do PR (§6.1 amplia):**

- `classifyError(err)` — provável função similar em `lib/pipeline/batch-processor/error-handlers.ts`. Se não existir, criar mapeamento simples baseado em `err.code` e `err.message`.
- Como `skillsRejectedByGate` está estruturado em `persist-curation/persist-fn.ts` hoje — precisa carregar `label` + `confidence` + `needsReview` para reconstruir o item de orchestrator.
- Como o tipo `RawSkill` é declarado (precisa incluir `needsReview?: boolean` para propagar D-PS-69 da cleanup v3.3). Se ainda não inclui, ampliar declaração em `lib/pipeline/types.ts`.

---

### §3.2 — `insertFOSkillItem` em call sites

**Atende:** S-ORCH-4.
**SUB-PR:** 6.

**Função utilitária (em arquivo novo `lib/pipeline/_shared/insert-fo-skill-item.ts`):**

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';
import type { SkillProcessingDetail } from '@/lib/pipeline/ingest-job-and-discover-skills';

export async function insertFOSkillItem(
  supabase: SupabaseClient,
  runId: string,
  jobPostingId: string,
  detail: SkillProcessingDetail,
): Promise<void> {
  const { error } = await supabase
    .from('function_orchestrator_skill_items')
    .insert({
      run_id: runId,
      job_posting_id: jobPostingId,
      raw_skill_name: detail.raw_skill_name,
      canonical_skill_id: detail.canonical_skill_id,
      canonical_status: detail.canonical_status,
      skill_pipeline_stage: detail.skill_pipeline_stage,
      skill_confidence: detail.skill_confidence,
      status: detail.status,
      error_type: detail.error_type,
      error_detail: detail.error_detail,
      dict_match: detail.dict_match,
      dict_match_source: detail.dict_match_source ?? null,
      needs_review: detail.needs_review,
      processed_at: detail.processed_at,
    });

  if (error) {
    // Logging estruturado para conformidade com LK-CBO-30
    console.error('[FOSI] insert_failed', {
      run_id: runId,
      job_posting_id: jobPostingId,
      raw_skill_name: detail.raw_skill_name,
      error_code: error.code,
      error_message: error.message,
    });
    // Incrementar logging_failures em function_orchestrator_runs (campo pré-existente)
    await supabase.rpc('increment_run_logging_failures', { p_run_id: runId }).catch(() => {});
  }
}

export async function insertFOSkillItemsBatch(
  supabase: SupabaseClient,
  runId: string,
  jobPostingId: string,
  items: SkillProcessingDetail[],
): Promise<void> {
  if (items.length === 0) return;

  const rows = items.map(detail => ({
    run_id: runId,
    job_posting_id: jobPostingId,
    raw_skill_name: detail.raw_skill_name,
    canonical_skill_id: detail.canonical_skill_id,
    canonical_status: detail.canonical_status,
    skill_pipeline_stage: detail.skill_pipeline_stage,
    skill_confidence: detail.skill_confidence,
    status: detail.status,
    error_type: detail.error_type,
    error_detail: detail.error_detail,
    dict_match: detail.dict_match,
    dict_match_source: detail.dict_match_source ?? null,
    needs_review: detail.needs_review,
    processed_at: detail.processed_at,
  }));

  const { error } = await supabase
    .from('function_orchestrator_skill_items')
    .insert(rows);

  if (error) {
    console.error('[FOSI] batch_insert_failed', {
      run_id: runId,
      job_posting_id: jobPostingId,
      count: rows.length,
      error_code: error.code,
      error_message: error.message,
    });
    await supabase.rpc('increment_run_logging_failures', { p_run_id: runId }).catch(() => {});
  }
}
```

**Call sites a modificar — lista exata conforme §6.1 (ground truth via grep):**

| # | Arquivo | Mudança | Confirmação via §6.1 |
|---|---|---|---|
| 1 | `lib/pipeline/persist-curation.ts` (caminho curated) | após `await discoverAndLinkSkills` (ou `safeDiscoverAndLinkSkills`, conforme E0), chamar `insertFOSkillItemsBatch(supabase, runId, jobPostingId, allItems)` — onde `allItems = skillDetail.items + gateRejectedItems` | confirmar linha exata pré-PR |
| 2 | `lib/pipeline/batch-processor/process-item.ts` (path quarantine) | NÃO altera — caminho quarantined usa legacy `mapSkillsToCanonical` e está fora de escopo (D-PS-83) | confirmar via §6.1 que esse path não invoca `discoverAndLinkSkills` |
| 3 | `lib/pipeline/batch-processor/error-handlers.ts` (parse error) | se o erro ocorre antes do gate, registrar items com `status='failed'` e `skill_pipeline_stage='fallback_error'` via `insertFOSkillItemsBatch` | confirmar formato de entrada do error handler |
| 4 | `lib/analysis/curate-flow-b.ts` (fluxo B) | confirmar se invoca `persist-curation.ts` (transitivo) ou tem caminho próprio de skill — se próprio, adicionar callsite explícito | §6.1 |
| 5 | `app/api/admin/curate-job-postings/_handlers/persist-results.ts` (admin manual, fluxo A) | mesma lógica de §3.1 Mudança 3 — captura `DiscoverResultDetailed`, adiciona `gateRejectedItems`, chama `insertFOSkillItemsBatch` | §6.1 |
| 6 | `scripts/test-flow-c.ts` ou equivalente do Fluxo C | confirmar se persistência de skill é coberta — se sim, callsite explícito | §6.1 |

**Pré-flight check (§7.1 E2):** após implementar, rodar:
```bash
grep -rn 'insertFOSkillItem' lib/ app/ scripts/ --include='*.ts' --include='*.tsx'
```
Esperado: 1 hit no arquivo do helper + N hits nos call sites identificados.

---

### §3.3 — Atualizar acumuladores nos 3 finalizers (via RPC SQL — obrigatório)

**Atende:** S-ORCH-5.
**SUB-PR:** 6.

**Decisão arquitetural cravada — LK-CBO-31:** Supabase JS truncate paginação em 1000 rows por default. Para runs com >1000 skills (cenário plausível em lotes de 100+ vagas com média de 10+ skills/vaga), `.from('function_orchestrator_skill_items').select('status').eq('run_id', runId)` retorna apenas 1000 rows e o agregado fica até 10× menor que o real — falha de correctness, não de performance. **Solução obrigatória:** RPC SQL `count_skill_items_by_status` definida em §2.4.

**Padrão a aplicar nos 3 finalizers:**

```typescript
// Substitui o padrão paginado por chamada à RPC
const { data: skillCounters, error: countError } = await supabase
  .rpc('count_skill_items_by_status', { p_run_id: runId });

if (countError) {
  console.error('[finalizer] count_skill_items_by_status failed', { run_id: runId, error: countError });
  // Continuar com zeros — finalizer não bloqueia por erro de telemetria
}

const counts = {
  skills_reused: 0,
  skills_pending_created: 0,
  skills_gate_rejected: 0,
  skills_failed: 0,
};

(skillCounters ?? []).forEach((row: { status: string; c: number }) => {
  if (row.status === 'reused') counts.skills_reused = Number(row.c);
  if (row.status === 'created_pending') counts.skills_pending_created = Number(row.c);
  if (row.status === 'gate_rejected') counts.skills_gate_rejected = Number(row.c);
  if (row.status === 'failed') counts.skills_failed = Number(row.c);
});

const skillsExtracted =
  counts.skills_reused +
  counts.skills_pending_created +
  counts.skills_gate_rejected +
  counts.skills_failed;

await supabase
  .from('function_orchestrator_runs')
  .update({
    /* ...campos role pré-existentes... */
    skills_extracted: skillsExtracted,
    skills_reused: counts.skills_reused,
    skills_pending_created: counts.skills_pending_created,
    skills_gate_rejected: counts.skills_gate_rejected,
    skills_failed: counts.skills_failed,
  })
  .eq('id', runId);
```

**Os 3 finalizers a atualizar — lista exata via §6.1:**

| # | Finalizer | Padrão | Confirmação via §6.1 |
|---|---|---|---|
| 1 | `lib/pipeline/curate-job-postings.ts` — função `finalizeFORun` | aplicar padrão acima ao final do UPDATE de role | confirmar linha do UPDATE em `function_orchestrator_runs` |
| 2 | `lib/analysis/curate-flow-b.ts` — finalizers do Fluxo B (provavelmente 2 sub-paths: run normal + run vazio) | aplicar padrão; sub-path de run vazio mantém contadores em 0 (naturalmente) | confirmar quantos sub-paths existem |
| 3 | `app/api/admin/curate-job-postings/route.ts` — UPDATE incremental por chunk no Fluxo A admin manual | **caso especial** — acumulação incremental por chunk, não agregação no final | ver §3.3.1 abaixo |

#### §3.3.1 — Caso especial: UPDATE incremental por chunk no Fluxo A

No Fluxo A admin manual, `route.ts` faz UPDATE incremental dos counters em FOR após cada chunk processado. Aqui a RPC `count_skill_items_by_status` é chamada **a cada chunk** com o `runId` correspondente — naturalmente retorna o agregado correto (acumulado até aquele ponto, já que FOSI nunca diminui). O UPDATE sobrescreve com o agregado real:

```typescript
// Após processar chunk N do Fluxo A
const { data: skillCounters } = await supabase
  .rpc('count_skill_items_by_status', { p_run_id: runId });
// Mesmo mapeamento da §3.3
// UPDATE com os valores absolutos (não incrementais — sobrescreve)
```

Vantagem: idempotente. Se um chunk reprocessar, o agregado continua correto.

---

### §3.4 — Atualização de call sites pós-rename

**Atende:** §2.6 rename.
**SUB-PR:** 4 (junto com §2.6).

**Lista exaustiva — a confirmar via grep antes do PR (§6.1):**

```bash
grep -rn 'function_orchestrator_items' app/ lib/ scripts/ \
  --include='*.ts' --include='*.tsx'
```

**Arquivos esperados (lista mínima conhecida via ground truth):**

- `lib/pipeline/curate-job-postings.ts`
- `lib/analysis/insert-jobs.ts`
- `lib/analysis/curate-flow-b.ts`
- `lib/pipeline/batch-processor/process-item.ts`
- `lib/pipeline/batch-processor/error-handlers.ts`
- `app/api/admin/curate-job-postings/_handlers/fetch-chunk.ts`
- `app/api/admin/curate-job-postings/_handlers/persist-results.ts`
- `app/api/admin/curate-job-postings/route.ts`
- `scripts/test-flow-c.ts`

**Operação:** find/replace em todos os arquivos. Mudança puramente de identificador de tabela — sem mudança de lógica.

**Pré-flight check (§7.1 E2):** rodar `grep -rn 'function_orchestrator_items' app/ lib/ scripts/ --include='*.ts' --include='*.tsx'` antes do merge — deve retornar 0 referências.

**Atualização do tipo TypeScript:** `lib/pipeline/types.ts` exporta `PipelineStage` (ou similar) com os 6 valores do enum role. Sem mudança de valores, mas o type alias ou comentário pode referenciar o nome novo da tabela.

---

### §3.5 — Queries novas em `lib/admin/limiares/queries.ts` (Frente C)

**Atende:** S-ORCH-8 (alimentação do endpoint `impact-preview` no padrão Limiares + enriquecimento dos painéis Limiares 1 e 8).
**SUB-PR:** 7.

A aba Limiares usa `pg.Pool` direto (não Supabase client) para queries com `WIDTH_BUCKET`, CTEs e `UNION ALL`. Cache Redis com TTL 60s (online) ou 300s (historical), allowlist de intervalos `{'1 day','7 days','30 days'}` via `INTERVAL_SQL`, `Promise.allSettled` para resilience.

Esta sprint AMPLIA `lib/admin/limiares/queries.ts` com queries novas alimentando 2 frentes:

1. **Endpoint `impact-preview`** (consumidor principal) — 4 queries cobrindo as fontes não presentes em queries pré-existentes da aba Limiares
2. **Enriquecimento dos painéis Limiares 1 e 8** — séries paralelas FOSI/FORI lado a lado com séries pré-existentes baseadas em `events` / `job_canonical_*`

#### §3.5.1 — `confidenceDistributionFORI(intervalSql)`

Distribuição de confidence em `function_orchestrator_role_items` na janela. Consumida pela chave `role.hard_gate.min_confidence` no impact-preview.

```sql
WITH role_items_window AS (
  SELECT fori.confidence
  FROM function_orchestrator_role_items fori
  JOIN function_orchestrator_runs r ON r.id = fori.run_id
  WHERE r.started_at >= NOW() - $1::INTERVAL
    AND fori.confidence IS NOT NULL
)
SELECT
  WIDTH_BUCKET(confidence, 0.0, 1.0, 10) AS bucket,
  COUNT(*) AS cnt,
  MIN(confidence) AS bucket_min,
  MAX(confidence) AS bucket_max
FROM role_items_window
GROUP BY 1 ORDER BY 1;

-- Retorno: 10 buckets + counts. Endpoint deriva sample_size = SUM(cnt) e universe_size de uma query separada.
```

#### §3.5.2 — `confidenceDistributionFOSI(intervalSql)`

Distribuição de confidence em `function_orchestrator_skill_items` na janela. Consumida pela chave `skill.hard_gate.min_confidence` no impact-preview. Também alimenta a série complementar no painel Limiares 1.

```sql
WITH skill_items_window AS (
  SELECT fosi.skill_confidence
  FROM function_orchestrator_skill_items fosi
  JOIN function_orchestrator_runs r ON r.id = fosi.run_id
  WHERE r.started_at >= NOW() - $1::INTERVAL
    AND fosi.skill_confidence IS NOT NULL
)
SELECT
  WIDTH_BUCKET(skill_confidence, 0.0, 1.0, 10) AS bucket,
  COUNT(*) AS cnt,
  MIN(skill_confidence) AS bucket_min,
  MAX(skill_confidence) AS bucket_max
FROM skill_items_window
GROUP BY 1 ORDER BY 1;
```

#### §3.5.3 — `opusReviewCooldownDistribution(intervalSql, entityType)`

Agregação sobre `opus_arbitration_outcomes` separando por `entity_type` ('role' ou 'skill'). Consumida pelas chaves `{role,skill}.opus_review.cooldown_days` no impact-preview.

```sql
-- Distribuição de dias desde a última arbitragem por canonical
WITH last_arb AS (
  SELECT
    canonical_id,
    entity_type,
    MAX(decided_at) AS last_arbitration_at
  FROM opus_arbitration_outcomes
  WHERE decided_at >= NOW() - $1::INTERVAL
    AND entity_type = $2  -- 'role' ou 'skill'
  GROUP BY canonical_id, entity_type
)
SELECT
  WIDTH_BUCKET(
    EXTRACT(DAY FROM NOW() - last_arbitration_at)::INT,
    0, 365, 10
  ) AS bucket,
  COUNT(*) AS cnt,
  MIN(EXTRACT(DAY FROM NOW() - last_arbitration_at)::INT) AS bucket_min_days,
  MAX(EXTRACT(DAY FROM NOW() - last_arbitration_at)::INT) AS bucket_max_days
FROM last_arb
GROUP BY 1 ORDER BY 1;
```

#### §3.5.4 — `skillPathDistribution(intervalSql)` (enriquecimento painel Limiares 8)

Distribuição de `skill_pipeline_stage` em FOSI separando o universo por path. Alimenta refonte do painel 8 com séries por path em vez de só `creation_confidence`.

```sql
SELECT
  fosi.skill_pipeline_stage AS path,
  WIDTH_BUCKET(fosi.skill_confidence, 0.0, 1.0, 10) AS bucket,
  COUNT(*) AS cnt
FROM function_orchestrator_skill_items fosi
JOIN function_orchestrator_runs r ON r.id = fosi.run_id
WHERE r.started_at >= NOW() - $1::INTERVAL
  AND fosi.skill_confidence IS NOT NULL
  AND fosi.skill_pipeline_stage IN ('slug_match', 'alias_match', 'llm_new', 'race_recovered')
GROUP BY 1, 2
ORDER BY 1, 2;

-- Retorno: linhas (path, bucket, cnt) para 4 paths × 10 buckets.
-- Painel renderiza como séries empilhadas ou linhas paralelas.
```

#### §3.5.5 — Queries reusadas (sem mudança nesta sprint)

As demais 14 chaves do IMPACT_SOURCES reutilizam queries que JÁ existem em `lib/admin/limiares/queries.ts` ou em outros pontos do projeto. Auditoria via §6.1 confirma a interseção exata:

| Chave do IMPACT_SOURCES | Reuso de query existente |
|---|---|
| `{role,skill}.promotion.auto_min_confidence` | Lógica do painel 2 (banda por confidence_median) — query já existe |
| `{role,skill}.promotion.min_vacancies` + `min_distinct_employers` | Lógica do painel 4 (filtro vacancy_count + distinct_sources_count) — query já existe |
| `{role,skill}.promotion.lookback_days` | Lógica do painel 10 (events `*_promoted_dynamic`) — query já existe |
| `{role,skill}.merge_candidate.*` | Lógica do painel 3 (`canonical_*_merge_candidates`) — query já existe |
| `{role,skill}.retirement.gap_days` | Lógica do painel 6 (`latest_posted_at` vs `gap_days`) — query já existe |

Endpoint `impact-preview` invoca essas queries existentes adaptadas para receber `new_value` como parâmetro de comparação.

#### §3.5.6 — Helpers de cálculo derivado

Função utilitária para derivar `current_impact` + `proposed_impact` a partir de uma distribuição agregada (10 buckets) + `current_value` + `new_value`:

```typescript
// lib/admin/limiares/helpers/derive-impact-from-distribution.ts (novo)

export function deriveImpactFromDistribution(args: {
  buckets: Array<{ bucket: number; cnt: number; bucket_min: number; bucket_max: number }>;
  current_value: number;
  new_value: number;
}): {
  current: { items_total: number; items_passing: number; items_rejecting: number; pass_rate: number };
  proposed: { items_total: number; items_passing: number; items_rejecting: number; pass_rate: number; delta_passing: number; delta_rejecting: number };
} {
  const total = args.buckets.reduce((acc, b) => acc + b.cnt, 0);

  const sumPassingAbove = (threshold: number) =>
    args.buckets
      .filter(b => b.bucket_min >= threshold)
      .reduce((acc, b) => acc + b.cnt, 0);

  const currentPassing = sumPassingAbove(args.current_value);
  const proposedPassing = sumPassingAbove(args.new_value);

  return {
    current: {
      items_total: total,
      items_passing: currentPassing,
      items_rejecting: total - currentPassing,
      pass_rate: total > 0 ? currentPassing / total : 0,
    },
    proposed: {
      items_total: total,
      items_passing: proposedPassing,
      items_rejecting: total - proposedPassing,
      pass_rate: total > 0 ? proposedPassing / total : 0,
      delta_passing: proposedPassing - currentPassing,
      delta_rejecting: (total - proposedPassing) - (total - currentPassing),
    },
  };
}
```

#### §3.5.7 — Enriquecimento do painel Limiares 1 (Hard Gate) — D-PS-91

Painel 1 hoje conta `events.skill_filtered_hard_gate` por dia (1 barra em 24h, 7 ou 30 em janelas maiores). Esta sprint adiciona série complementar via `function_orchestrator_skill_items.status='gate_rejected'`:

```sql
-- Série complementar via FOSI (fonte mais robusta capturada no ponto da decisão)
SELECT
  date_trunc('day', fosi.created_at) AS day,
  COUNT(*) AS cnt_fosi
FROM function_orchestrator_skill_items fosi
WHERE fosi.status = 'gate_rejected'
  AND fosi.created_at >= NOW() - $1::INTERVAL
GROUP BY 1
ORDER BY 1;
```

UI do painel 1 ganha 2 séries paralelas (events legado + FOSI) com tooltip explicando a divergência quando houver.

#### §3.5.8 — Enriquecimento do painel Limiares 8 (creation_confidence distribuição) — D-PS-91

Painel 8 hoje renderiza histograma de `creation_confidence` em `job_canonical_skills` + `job_canonical_roles` (UNION ALL). Esta sprint adiciona séries por path via `skillPathDistribution` (§3.5.4) e equivalente para role (`rolePathDistribution` — análoga adaptada ao enum de role).

UI ramifica em modo "Por entidade" (skill / role — comportamento atual) e modo novo "Por path" (slug_match / alias_match / llm_new / race_recovered). Toggle dentro do painel.

---

### §3.6 — Expansão de `lib/admin/dashboard-day-aggregator.ts` (Frente B)

**Atende:** materialização de métricas FORI/FOSI/FOR no JSONB de `dashboard_daily_summary.data` para uso genérico em painéis do dashboard global.
**SUB-PR:** 9.
**Pré-requisito:** SUB-PR 6 finalizado — FOSI populada e contadores `skills_*` em FOR escrevendo.

#### §3.6.1 — Sub-função `aggregatePipelineOrchestratorMetrics`

Nova função em `lib/admin/dashboard-day-aggregator.ts` (ou módulo importado por ele) chamada de dentro de `aggregateDayData(supabase, summaryDate)`. Coleta dados de FORI + FOSI + contadores `skills_*` em FOR para o dia (`summaryDate` ± 23:59:59 UTC).

```typescript
// lib/admin/dashboard-day-aggregator.ts (expansão dentro de aggregateDayData)

async function aggregatePipelineOrchestratorMetrics(
  supabase: SupabaseClient,
  summaryDate: string,  // YYYY-MM-DD
): Promise<PipelineOrchestratorMetrics> {
  const dayStart = `${summaryDate}T00:00:00Z`;
  const dayEnd = `${summaryDate}T23:59:59Z`;

  // 1. Counts por pipeline_stage em FORI
  const { data: foriStages } = await supabase.rpc('count_role_items_by_stage_in_window', {
    p_start: dayStart, p_end: dayEnd
  });
  // RPC nova em §3.6.2

  // 2. Counts por pipeline_stage em FOSI
  const { data: fosiStages } = await supabase.rpc('count_skill_items_by_stage_in_window', {
    p_start: dayStart, p_end: dayEnd
  });
  // RPC nova em §3.6.2

  // 3. Counts por status em FORI / FOSI (analógo aos contadores existentes em FOR)
  // Reusa pattern de count_skill_items_by_status (§2.4) adaptada com filtro de janela

  // 4. Items processados (volume bruto)
  const roleItemsTotal = Object.values(foriStages).reduce((a, b) => a + b, 0);
  const skillItemsTotal = Object.values(fosiStages).reduce((a, b) => a + b, 0);

  // 5. Cálculo de O7-skill (drift = canonicals_novos_skill / items_processed_skill)
  const { data: skillCanonicalsCreated } = await supabase
    .from('events')
    .select('id', { count: 'exact', head: true })
    .eq('event_name', 'skill_canonical_created')
    .gte('created_at', dayStart)
    .lte('created_at', dayEnd);
  const o7SkillDrift = skillItemsTotal > 0 ? (skillCanonicalsCreated.count / skillItemsTotal) * 100 : 0;

  // 6. Cálculo de O9-skill (saúde — fallback_error rate + sem_canonical)
  const fallbackErrorCount = fosiStages['fallback_error'] || 0;
  const semCanonicalCount = await countSkillItemsWithoutCanonical(supabase, dayStart, dayEnd);
  // helper auxiliar

  return {
    role: {
      items_by_stage: foriStages,
      items_total: roleItemsTotal,
      o7_drift_percent: /* análogo para role, se ainda não existir em outras sub-funções */,
      // ... demais campos
    },
    skill: {
      items_by_stage: fosiStages,
      items_total: skillItemsTotal,
      o7_canonicals_novos: skillCanonicalsCreated.count,
      o7_drift_percent: o7SkillDrift,
      o9_fallback_error: fallbackErrorCount,
      o9_sem_canonical: semCanonicalCount,
      needs_review_count: await countNeedsReview(supabase, dayStart, dayEnd),
    },
    items_processed_total: roleItemsTotal + skillItemsTotal,
  };
}
```

E `aggregateDayData()` passa a incorporar:

```typescript
// dentro de aggregateDayData(supabase, summaryDate)
const pipelineMetrics = await aggregatePipelineOrchestratorMetrics(supabase, summaryDate);

return {
  // ... famílias pré-existentes (ai, resources, communications, auth, operational, errors, accumulators, O7/O8/O9 role)
  operational: {
    ...operationalPreExisting,  // o7, o8, o9 role-only preservados
    o7_skill: pipelineMetrics.skill.o7_drift_percent,
    o8_skill: pipelineMetrics.skill.items_by_stage,
    o9_skill: {
      fallback_error: pipelineMetrics.skill.o9_fallback_error,
      sem_canonical: pipelineMetrics.skill.o9_sem_canonical,
      status_summary: deriveO9Status(pipelineMetrics.skill),
    },
    items_processed: {
      role: pipelineMetrics.role.items_total,
      skill: pipelineMetrics.skill.items_total,
      total: pipelineMetrics.items_processed_total,
    },
  },
  // ... resto da família operational e outras famílias
};
```

#### §3.6.2 — RPCs novas em SQL

Para evitar truncamento Supabase em paginação de >1000 rows (LK-CBO-31), criar 2 RPCs SQL análogas a `count_skill_items_by_status` (§2.4):

```sql
-- count_role_items_by_stage_in_window
CREATE OR REPLACE FUNCTION count_role_items_by_stage_in_window(
  p_start timestamptz,
  p_end timestamptz
)
RETURNS TABLE(stage text, c bigint)
LANGUAGE sql STABLE
AS $$
  SELECT fori.pipeline_stage AS stage, COUNT(*) AS c
  FROM function_orchestrator_role_items fori
  JOIN function_orchestrator_runs r ON r.id = fori.run_id
  WHERE r.started_at >= p_start
    AND r.started_at <= p_end
  GROUP BY fori.pipeline_stage;
$$;

-- count_skill_items_by_stage_in_window
CREATE OR REPLACE FUNCTION count_skill_items_by_stage_in_window(
  p_start timestamptz,
  p_end timestamptz
)
RETURNS TABLE(stage text, c bigint)
LANGUAGE sql STABLE
AS $$
  SELECT fosi.skill_pipeline_stage AS stage, COUNT(*) AS c
  FROM function_orchestrator_skill_items fosi
  JOIN function_orchestrator_runs r ON r.id = fosi.run_id
  WHERE r.started_at >= p_start
    AND r.started_at <= p_end
  GROUP BY fosi.skill_pipeline_stage;
$$;
```

Estas RPCs entram como migrations 07 e 08 dentro do SUB-PR 9.

#### §3.6.3 — Backfill obrigatório pós-deploy

Após deploy do aggregator expandido, rodar `/api/admin/backfill-dashboard-summary` para os 26 dias rolling, regenerando o JSONB de cada row de `dashboard_daily_summary` com os campos novos preenchidos. Endpoint já existe (cleanup do projeto pré-existente) e é idempotente — reusa `aggregateDayData()`.

**Restrição operacional:** O backfill só faz sentido depois que FOSI tem dados (SUB-PR 6 implementado em produção). Para os 26 dias retroativos:

- Para dias ANTES da implementação do SUB-PR 6, FOSI estará vazia → `o8_skill`, `o9_skill.fallback_error`, `items_processed.skill` serão 0
- Para dias APÓS o SUB-PR 6, todos os campos terão dados reais

Isso é aceitável em pré-produção pré-MVP. UI dos painéis 2.7/2.8/2.9 trata 0 como "sem dados", não como sinal de problema.

#### §3.6.4 — Campos pré-existentes role-only mantidos intactos

D-PS-92 cravado: `data.operational.o7`, `data.operational.o8`, `data.operational.o9` (todos role-only) **NÃO são tocados** por esta sprint. UI dos painéis 2.7/2.8/2.9 continua lendo os campos pré-existentes para a série role + os novos campos `o7_skill`/`o8_skill`/`o9_skill` para a série skill. Sem refator dos consumidores pré-existentes.

Renomeação dos pré-existentes para `o7_role`/`o8_role`/`o9_role` (simetria nominal completa) fica documentada como frente futura em LK-PS-13 (sem urgência).

---

## §4 Endpoints API

### §4.1 — Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` no padrão Limiares

**Atende:** S-ORCH-8 (fechamento de D-PS-33 da cleanup v3.3 via expansão aditiva, não substituição). Opção β cravada pelo product owner em 2026-05-14, refinada para padrão Limiares (Opção 2 cross-cutting) em 2026-05-14.
**SUB-PR:** 7.
**Pré-requisito:** cleanup v3.3 mergeada — endpoint `POST /[key]/impact-preview` já existe em `app/api/admin/pipeline-config/[key]/impact-preview/route.ts` (§4.1.5 da cleanup v3.3), com handler que despacha para os 8 estimators de `lib/admin/pipeline-impact-estimators.ts` (§3.16 da cleanup v3.3), retornando `affected_count` + `projected_event_cost_usd` para chaves cobertas, com cache in-process Map TTL 30s.

#### §4.1.1 — Decisão arquitetural: migração para padrão Limiares (D-PS-85 + D-PS-86)

A cleanup v3.3 cravou:

- Método HTTP: **POST** (preserva semântica de envio de novo valor + flags de cache control no body)
- Nome canônico: **`impact-preview`** (não `impact`)
- Cache in-process `Map<string, CacheEntry>` TTL 30s + eviction 200 entradas (D-PS-68 cleanup v3.3) — **REVOGADA por esta sprint**
- Terminologia `projected_event_cost_usd` (D-PS-67 cleanup v3.3 — evento único, NÃO recorrência mensal) — **preservada**
- 8 estimators cobertos em `pipeline-impact-estimators.ts` para 4 famílias de chaves — **preservados**

Esta sprint orchestrator **migra o endpoint para o padrão arquitetônico da aba Limiares** sem renomear ou trocar método, preservando todos os campos pré-existentes do payload e adicionando 4 novos blocos:

1. **Conexão via `pg.Pool` direta** (não Supabase client) — necessário para `WIDTH_BUCKET`, CTEs, agregações que PostgREST não expressa
2. **Cache via Redis TTL 300s** (mesmo padrão `limiares:historical:*`) — substitui o cache in-process Map TTL 30s
3. **Allowlist de intervalos** `[7d, 30d, 90d]` validada em tempo de execução (defesa contra SQL injection via cast)
4. **`Promise.allSettled`** para tolerância: estimator pode falhar sem derrubar a resposta IMPACT_SOURCES, e vice-versa

Cobertura expandida: **22 chaves operacionais com algum payload preenchido**:

- 6 chaves com payload completo (estimator + IMPACT_SOURCES): `hard_gate.min_confidence` role+skill, `promotion.auto_min_confidence` role+skill, `promotion.min_vacancies` role+skill
- 14 chaves com payload parcial via IMPACT_SOURCES (sem estimator): `promotion.lookback_days` role+skill, `promotion.min_distinct_employers` role+skill, todas as 6 `merge_candidate.*`, ambas as `opus_review.cooldown_days`, ambas as `retirement.gap_days`
- 2 chaves com payload parcial via estimators (sem IMPACT_SOURCES): `role.auto_assign_family.min_similarity`, `role.auto_assign_family.min_score` — assimetria D-PS-41 cleanup v3.3 mantida (famílias só para role)
- 4 chaves de confidence retornam `sample_status='unsupported_in_v1'` (D-PS-80)

Total operacional no banco: 26 chaves (24 originais + 2 `auto_assign_family.*` da cleanup v3.3). Aritmética: 22 com payload + 4 confidence = 26 operacionais. + 2 sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) filtradas no UI antes de chegar ao endpoint. Total no banco = 28.

#### §4.1.2 — Contrato

**Request:**

```
POST /api/admin/pipeline-config/[key]/impact-preview
Content-Type: application/json

{
  "new_value": "0.75",
  "days": 30,
  "skip_cache": false
}
```

- `key` (path param) — chave de `pipeline_config` (uma das 26 operacionais)
- `new_value` (body) — valor proposto (string, validado conforme tipo da chave)
- `days` (body, opcional) — janela retroativa; valores aceitos: `7`, `30`, `90`; default `30`
- `skip_cache` (body, opcional) — flag pré-existente da cleanup v3.3 preservada

**Auth:** admin (`requireAdmin`).

**Validação de janela:** allowlist `INTERVAL_SQL` adaptada do padrão Limiares:

```typescript
const VALID_WINDOWS = { 7: '7 days', 30: '30 days', 90: '90 days' } as const;
const days = body.days ?? 30;

if (!(days in VALID_WINDOWS)) {
  return NextResponse.json(
    { error: 'invalid_window', accepted: [7, 30, 90] },
    { status: 400 }
  );
}

const intervalSql = VALID_WINDOWS[days as keyof typeof VALID_WINDOWS];
// intervalSql é usado como literal seguro porque vem de allowlist
```

#### §4.1.3 — Payload de resposta — caso normal (chave coberta por estimator + IMPACT_SOURCES, amostra suficiente)

```json
{
  "key": "skill.hard_gate.min_confidence",
  "current_value": "0.70",
  "new_value": "0.75",
  "window_days": 30,
  "source": "function_orchestrator_skill_items",

  "affected_count": 1756,
  "projected_event_cost_usd": 0.43,
  "cost_is_fallback": false,

  "sample_size": 12450,
  "sample_status": "sufficient",
  "current_impact": {
    "items_total": 12450,
    "items_passing": 9876,
    "items_rejecting": 2574,
    "pass_rate": 0.793
  },
  "proposed_impact": {
    "items_total": 12450,
    "items_passing": 8120,
    "items_rejecting": 4330,
    "pass_rate": 0.652,
    "delta_passing": -1756,
    "delta_rejecting": 1756
  },
  "histogram": {
    "buckets": [
      { "range": [0.50, 0.55], "count": 423 },
      { "range": [0.55, 0.60], "count": 612 },
      { "range": [0.60, 0.65], "count": 859 },
      { "range": [0.65, 0.70], "count": 1241 },
      { "range": [0.70, 0.75], "count": 1756 },
      { "range": [0.75, 0.80], "count": 1893 },
      { "range": [0.80, 0.85], "count": 1845 },
      { "range": [0.85, 0.90], "count": 1981 },
      { "range": [0.90, 0.95], "count": 1840 },
      { "range": [0.95, 1.00], "count": 1840 }
    ],
    "bucket_count": 10
  }
}
```

**Campos pré-existentes da cleanup v3.3 (preservados sem mudança):** `key`, `current_value`, `new_value`, `affected_count`, `projected_event_cost_usd`, `cost_is_fallback`.
**Campos novos desta sprint:** `window_days`, `source`, `sample_size`, `sample_status`, `current_impact`, `proposed_impact`, `histogram`.

#### §4.1.4 — Payload — chave coberta apenas por IMPACT_SOURCES (sem estimator pré-existente)

Para as 14 chaves cobertas por IMPACT_SOURCES mas NÃO cobertas pelos 8 estimators pré-existentes (cooldowns, lookback_days de merge_candidate/opus_review/promotion, distinct_employers, retirement.gap_days), o endpoint retorna campos novos da sprint e marca campos pré-existentes como nulos:

```json
{
  "key": "skill.opus_review.cooldown_days",
  "current_value": "90",
  "new_value": "60",
  "window_days": 30,
  "source": "opus_arbitration_outcomes",

  "affected_count": null,
  "projected_event_cost_usd": null,
  "cost_is_fallback": null,

  "sample_size": 8,
  "sample_status": "insufficient",
  "sample_threshold": {
    "absolute_min": 30,
    "relative_min_pct_of_universe": 5
  },
  "current_impact": {
    "items_total": 8,
    "items_inside_cooldown": 5,
    "items_outside_cooldown": 3
  },
  "proposed_impact": {
    "items_total": 8,
    "items_inside_cooldown": 3,
    "items_outside_cooldown": 5,
    "delta_inside": -2,
    "delta_outside": 2
  }
  /* histogram OMITIDO quando sample_status === 'insufficient' */
}
```

UI exibe nestes casos a tabela de contagem desta sprint, sem linha de "Custo Opus projetado" (esconde quando `projected_event_cost_usd === null` — alinhado à diretriz DC-2 / D-PS-67 da cleanup v3.3).

#### §4.1.5 — Payload — chave coberta apenas por estimator (`auto_assign_family.*` role-only)

Para `role.auto_assign_family.min_similarity` e `role.auto_assign_family.min_score` — cobertas por estimator pré-existente mas fora de IMPACT_SOURCES por D-PS-41 cleanup v3.3 (assimetria de famílias só role):

```json
{
  "key": "role.auto_assign_family.min_similarity",
  "current_value": "0.75",
  "new_value": "0.80",
  "window_days": 30,
  "source": "estimator_only",

  "affected_count": 23,
  "projected_event_cost_usd": 0.12,
  "cost_is_fallback": false,

  "sample_size": null,
  "sample_status": "estimator_only",
  "current_impact": null,
  "proposed_impact": null
  /* histogram OMITIDO */
}
```

UI exibe linha "Custo Opus projetado (evento único)" da caixa qualitativa pré-existente + placeholder no bloco quantitativo: "Análise quantitativa não disponível para esta chave nesta versão — caixa qualitativa ao lado orienta o impacto."

#### §4.1.6 — Payload — chave de confidence (out-of-scope v1 desta sprint)

```json
{
  "key": "skill.confidence.min_count",
  "current_value": "3",
  "new_value": "4",
  "window_days": 30,
  "source": "unsupported_in_v1",

  "affected_count": null,
  "projected_event_cost_usd": null,
  "cost_is_fallback": null,

  "sample_status": "unsupported_in_v1",
  "unsupported_reason": "Chaves de confidence (lookback_days, min_count) requerem recálculo de confidence_median em massa, incompatível com endpoint síncrono. Frente futura: sprint dedicada de circuito de confidence.",
  "current_impact": null,
  "proposed_impact": null
  /* histogram OMITIDO */
}
```

HTTP **200** (não 404) — preserva semântica do endpoint v3.3 que sempre retorna 200 com campos nulos quando a chave não está coberta. UI ramifica por `sample_status === 'unsupported_in_v1'`.

#### §4.1.7 — Payload — chave de sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`)

Comportamento já cravado pela cleanup v3.3 (tela admin pipeline-config filtra essas chaves via §5.1.0). Caso o endpoint seja chamado diretamente (via curl ou outro caller), retorna 200 com `sample_status='unsupported_in_v1'` e `unsupported_reason='Chave de sistema, não exposta na tela /admin/pipeline-config'`.

#### §4.1.8 — Lógica do `sample_status`

```typescript
function classifySample(
  itemCount: number,
  universeSize: number,
  keyClass: 'covered' | 'estimator_only' | 'unsupported_in_v1',
): 'sufficient' | 'insufficient' | 'estimator_only' | 'unsupported_in_v1' {
  if (keyClass === 'unsupported_in_v1') return 'unsupported_in_v1';
  if (keyClass === 'estimator_only') return 'estimator_only';

  const ABSOLUTE_MIN = 30;
  const RELATIVE_MIN_PCT = 0.05;

  if (itemCount >= ABSOLUTE_MIN) return 'sufficient';
  if (itemCount >= universeSize * RELATIVE_MIN_PCT) return 'sufficient';
  return 'insufficient';
}
```

Critério: N >= 30 itens **OU** N >= 5% do universo total → suficiente. Critério OR (não AND). Histograma é omitido quando insuficiente ou estimator_only; tabela de contagem é exibida quando `current_impact` existe. Edição NÃO é bloqueada em nenhum caso.

#### §4.1.9 — Mapeamento chave → fonte de dados (20 chaves cobertas pelo IMPACT_SOURCES)

Implementação reutiliza o mapa de `pipeline-config-tooltips.ts` (§3.14 da cleanup v3.3) como fonte canônica das chaves. Para cada chave, define a fonte de dados:

```typescript
// lib/admin/pipeline-config-impact-sources.ts (novo)
// IMPORTAR mapa de chaves de pipeline-config-tooltips.ts para evitar divergência

import { PIPELINE_CONFIG_TOOLTIPS } from './pipeline-config-tooltips';

export type ImpactSource = {
  table: string;
  confidence_column?: string;
  column?: string;
  custom_query?: string;
  reused_limiares_query?: string;  // referência a query existente em lib/admin/limiares/queries.ts
};

export const IMPACT_SOURCES: Record<string, ImpactSource> = {
  // ─── HARD GATE (2) — queries novas em §3.5 ───
  'role.hard_gate.min_confidence': {
    table: 'function_orchestrator_role_items',
    confidence_column: 'confidence',
    custom_query: 'confidenceDistributionFORI',
  },
  'skill.hard_gate.min_confidence': {
    table: 'function_orchestrator_skill_items',
    confidence_column: 'skill_confidence',
    custom_query: 'confidenceDistributionFOSI',
  },

  // ─── PROMOTION (8) — reusam queries do painel Limiares 2, 4, 10 ───
  'role.promotion.auto_min_confidence': {
    table: 'job_canonical_roles',
    column: 'confidence_median',
    reused_limiares_query: 'panel2_promotion_band_role',
  },
  'skill.promotion.auto_min_confidence': {
    table: 'job_canonical_skills',
    column: 'confidence_median',
    reused_limiares_query: 'panel2_promotion_band_skill',
  },
  'role.promotion.lookback_days': {
    table: 'events',
    custom_query: 'canonical_promoted_gap_days_role',
  },
  'skill.promotion.lookback_days': {
    table: 'events',
    custom_query: 'canonical_promoted_gap_days_skill',
  },
  'role.promotion.min_distinct_employers': {
    table: 'job_canonical_roles',
    column: 'distinct_sources_count',
    reused_limiares_query: 'panel4_gate_cumulativo_role',
  },
  'skill.promotion.min_distinct_employers': {
    table: 'job_canonical_skills',
    column: 'distinct_sources_count',
    reused_limiares_query: 'panel4_gate_cumulativo_skill',
  },
  'role.promotion.min_vacancies': {
    table: 'job_canonical_roles',
    column: 'vacancy_count',
    reused_limiares_query: 'panel4_gate_cumulativo_role',
  },
  'skill.promotion.min_vacancies': {
    table: 'job_canonical_skills',
    column: 'vacancy_count',
    reused_limiares_query: 'panel4_gate_cumulativo_skill',
  },

  // ─── MERGE_CANDIDATE (6) — reusam queries do painel Limiares 3 ───
  'role.merge_candidate.cosine_threshold': {
    table: 'canonical_role_merge_candidates',
    column: 'similarity',
    reused_limiares_query: 'panel3_merge_candidates_role',
  },
  'skill.merge_candidate.cosine_threshold': {
    table: 'canonical_skill_merge_candidates',
    column: 'similarity',
    reused_limiares_query: 'panel3_merge_candidates_skill',
  },
  'role.merge_candidate.lookback_days': {
    table: 'canonical_role_merge_candidates',
    custom_query: 'merge_candidate_age_role',
  },
  'skill.merge_candidate.lookback_days': {
    table: 'canonical_skill_merge_candidates',
    custom_query: 'merge_candidate_age_skill',
  },
  'role.merge_candidate.opus_review_cooldown_days': {
    table: 'opus_arbitration_outcomes',
    custom_query: 'opus_cooldown_merge_role',
  },
  'skill.merge_candidate.opus_review_cooldown_days': {
    table: 'opus_arbitration_outcomes',
    custom_query: 'opus_cooldown_merge_skill',
  },

  // ─── OPUS_REVIEW (2) — queries novas §3.5.3 ───
  'role.opus_review.cooldown_days': {
    table: 'opus_arbitration_outcomes',
    custom_query: 'opusReviewCooldownDistribution_role',
  },
  'skill.opus_review.cooldown_days': {
    table: 'opus_arbitration_outcomes',
    custom_query: 'opusReviewCooldownDistribution_skill',
  },

  // ─── RETIREMENT (2) — reusam query do painel Limiares 6 ───
  'role.retirement.gap_days': {
    table: 'job_canonical_roles',
    custom_query: 'retirement_gap_days_role',
    reused_limiares_query: 'panel6_aposentadoria_role',
  },
  'skill.retirement.gap_days': {
    table: 'job_canonical_skills',
    custom_query: 'retirement_gap_days_skill',
    reused_limiares_query: 'panel6_aposentadoria_skill',
  },
};

// Validação de cobertura: IMPACT_SOURCES deve cobrir exatamente as 20 chaves
// (26 operacionais − 4 confidence − 2 auto_assign_family). Gate de CI:
// teste TypeScript valida que Object.keys(IMPACT_SOURCES) é exatamente esse subset.
// Build quebra se divergir.
```

#### §4.1.10 — Handler do endpoint (estrutura cravada)

```typescript
// app/api/admin/pipeline-config/[key]/impact-preview/route.ts
// Estrutura cravada na cleanup v3.3 + migração para padrão Limiares desta sprint

import { Pool } from 'pg';
import { redis } from '@/lib/admin/limiares/_shared/redis-cache';
import { IMPACT_SOURCES } from '@/lib/admin/pipeline-config-impact-sources';
import { ESTIMATORS } from '@/lib/admin/pipeline-impact-estimators';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 5,
  idleTimeoutMillis: 10000,
});

const VALID_WINDOWS = { 7: '7 days', 30: '30 days', 90: '90 days' } as const;
const REDIS_TTL = 300; // 300s historical Limiares padrão

export async function POST(req: NextRequest, { params }: { params: { key: string } }) {
  const { key } = params;
  const body = await req.json();
  const { new_value, days = 30, skip_cache = false } = body;

  if (!(days in VALID_WINDOWS)) {
    return NextResponse.json({ error: 'invalid_window', accepted: [7, 30, 90] }, { status: 400 });
  }
  const intervalSql = VALID_WINDOWS[days as keyof typeof VALID_WINDOWS];

  // 1. Cache Redis lookup (D-PS-86 padrão Limiares)
  const cacheKey = `limiares:impact_preview:${key}:${new_value}:${days}`;
  if (!skip_cache) {
    try {
      const cached = await redis.get(cacheKey);
      if (cached) return NextResponse.json(JSON.parse(cached));
    } catch (err) {
      // Fall-through silencioso para query direta (padrão Limiares)
      console.warn('[impact-preview] redis miss', err);
    }
  }

  // 2. Buscar current_value
  const currentValueRow = await pool.query(
    'SELECT value FROM pipeline_config WHERE key = $1', [key]
  );
  const current_value = currentValueRow.rows[0]?.value ?? null;

  // 3. Promise.allSettled — paralelo, tolerante a falhas individuais
  const [estimatorResult, impactResult] = await Promise.allSettled([
    callEstimator(key, current_value, new_value, days),
    callImpactSource(key, current_value, new_value, intervalSql),
  ]);

  // 4. Compor payload mesclado (mesmo da v1.3, agora com source resolution e estimator_only)
  const payload = composePayload({
    key,
    current_value,
    new_value,
    days,
    estimatorResult,
    impactResult,
  });

  // 5. Cache write
  if (!skip_cache) {
    try {
      await redis.setex(cacheKey, REDIS_TTL, JSON.stringify(payload));
    } catch (err) {
      console.warn('[impact-preview] redis write fail', err);
    }
  }

  return NextResponse.json(payload);
}

async function callEstimator(key: string, currentValue: any, newValue: any, days: number) {
  const estimator = ESTIMATORS[key];
  if (!estimator) return null;
  // Estimators pré-existentes da cleanup v3.3 usam Supabase client; preservados como estão.
  return await estimator(supabase, currentValue, newValue, days);
}

async function callImpactSource(key: string, currentValue: any, newValue: any, intervalSql: string) {
  if (key.match(/\.confidence\.(lookback_days|min_count)$/)) {
    return { unsupported_in_v1: true };
  }
  const source = IMPACT_SOURCES[key];
  if (!source) return null;

  // Dispatcha para a query do padrão Limiares
  if (source.custom_query) {
    return await runCustomQueryViaPool(pool, source.custom_query, currentValue, newValue, intervalSql);
  }
  if (source.reused_limiares_query) {
    return await runReusedLimiaresQuery(pool, source.reused_limiares_query, currentValue, newValue, intervalSql);
  }
  if (source.confidence_column || source.column) {
    return await runColumnDistributionQuery(pool, source, currentValue, newValue, intervalSql);
  }
  return null;
}
```

#### §4.1.11 — Cache Redis (D-PS-86 — substitui D-PS-68 da cleanup v3.3)

O cache in-process `Map<string, CacheEntry>` TTL 30s da cleanup v3.3 (D-PS-68) é **REMOVIDO desta sprint**. Substitui pelo padrão Redis da aba Limiares:

- TTL 300s (mesmo `limiares:historical:*`)
- Chave: `limiares:impact_preview:{key}:{new_value}:{days}`
- Fall-through silencioso para query direta em caso de erro Redis (padrão Limiares)
- Não é rate limiter (não bloqueia, não tem quota, não expõe headers)

Vantagens vs cache in-process:

- Compartilhado entre instâncias serverless Vercel (não evapora a cada deploy)
- TTL 10× maior (300s vs 30s) reduz queries em sessão típica de admin
- Coerência arquitetônica com a aba Limiares — admin tem comportamento consistente entre `impact-preview` e os 10 painéis Limiares
- Elimina possibilidade de bug "cache stale após restart silencioso"

#### §4.1.12 — Custom queries (em `lib/admin/pipeline-config-impact-custom-queries.ts` — arquivo novo)

Queries para chaves não cobertas por reuso de queries Limiares existentes. Todas via `pg.Pool` direto seguindo o padrão Limiares:

- `canonical_promoted_gap_days_role` / `canonical_promoted_gap_days_skill`
- `merge_candidate_age_role` / `merge_candidate_age_skill`
- `opus_cooldown_merge_role` / `opus_cooldown_merge_skill`

Definições completas no arquivo do helper. Padrão de cada query: agregar uma coluna temporal/numérica da tabela alvo filtrada por `intervalSql` (janela retroativa) e calcular `current_impact` (filtro pelo valor atual) e `proposed_impact` (filtro pelo valor proposto).

---

## §5 UI

### §5.1 — Integração no modal de edição + extração de componentes (DV-4 opção a)

**Atende:** S-ORCH-9.
**SUB-PR:** 8 (sequencial após cleanup v3.3 + SUB-PR 7).

**Pré-requisito:** sprint cleanup v3.3 já mergeada — modal de edição existe **inline em `app/admin/pipeline-config/page.tsx`** (não como componente separado). Função local `EditModal(...)` + função local `ImpactPreview(...)` ambas dentro do mesmo arquivo `page.tsx`. Cleanup v3.3 entregou o ImpactPreview inline consumindo `affected_count` + `projected_event_cost_usd` do endpoint pré-existente.

#### §5.1.0 — Pré-trabalho obrigatório: extração de componentes inline para arquivos próprios (D-PS-93)

Estado atual confirmado via grep no SUB-PR 3 (§6.1.6): tanto `EditModal` quanto `ImpactPreview` estão inline em `app/admin/pipeline-config/page.tsx`. Antes da integração do payload novo, **extrair ambos para arquivos próprios**:

1. Criar `components/admin/pipeline-config/EditModal.tsx` — recebe `configKey`, `currentValue`, `onSave`, `onCancel` como props
2. Criar `components/admin/pipeline-config/ImpactPreview.tsx` — recebe `configKey`, `currentValue`, `newValue` como props
3. Atualizar `app/admin/pipeline-config/page.tsx` para importar e usar os componentes extraídos
4. Migrar types compartilhados para `components/admin/pipeline-config/types.ts` (ou similar)

Custo estimado: ~1-2 horas. Beneficios:

- `page.tsx` que hoje contém tabela de chaves + botões + modal + ImpactPreview fica enxuto
- `ImpactPreview`, que nesta sprint vai virar componente complexo (histograma SVG, tabela com 3 estados, seletor de janela, render condicional por sample_status), fica em arquivo dedicado fácil de evoluir e testar
- Coerência arquitetônica com o resto do projeto (`components/admin/<feature>/` padrão)

**Notas operacionais:**

- Preservar fielmente o comportamento atual do `EditModal` (estados de loading, validação de input, criticality_level handling, confirmação textual "PUBLICAR" para criticidade high)
- Preservar fielmente o comportamento atual do `ImpactPreview` quanto a campos pré-existentes (`affected_count`, `projected_event_cost_usd`, `cost_is_fallback`) — apenas adicionar consumo dos campos novos do payload

#### §5.1.1 — Decisão arquitetural cravada — adição lateral, não substituição (D-PS-72)

O modal de edição mantém a estrutura cravada pela cleanup v3.3 (D-PS-33 cleanup) — caixa qualitativa pré-existente preservada — e adiciona campos novos ao componente `ImpactPreview` que (após extração) consome o endpoint refatorado em §4.1. O layout passa a ter **dois blocos lado a lado** dentro da seção "Impacto estimado":

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ Impacto estimado                                                              │
│                                                                               │
│ ┌─────────────────────────────────┐  ┌─────────────────────────────────────┐ │
│ │ [BLOCO QUALITATIVO]              │  │ [BLOCO QUANTITATIVO — EXPANDIDO]    │ │
│ │ (pré-existente da v3.3 cleanup)  │  │ (ImpactPreview com payload novo)    │ │
│ │                                  │  │                                     │ │
│ │ Painéis afetados:                │  │ Janela: 7d ─── 30d ═══ 90d ───      │ │
│ │ • Painel 1 — Hard Gate           │  │                                     │ │
│ │ • Painel 8 — Rejeitadas          │  │              Atual    Proposto      │ │
│ │                                  │  │ Items        12.450   12.450        │ │
│ │ Observação esperada:             │  │ Aprovados    9.876    8.120         │ │
│ │ Subir o piso aumenta o volume    │  │ Rejeitados   2.574    4.330         │ │
│ │ rejeitado nas primeiras janelas. │  │ Δ                     -1.756        │ │
│ │                                  │  │                                     │ │
│ │ Horizonte de acompanhamento:     │  │ Distribuição de confidence (30d):   │ │
│ │ 24-48h                           │  │ [histograma SVG inline]             │ │
│ │                                  │  │                                     │ │
│ │ Custo Opus projetado             │  │                                     │ │
│ │ (evento único): $0,43            │  │                                     │ │
│ │                                  │  │                                     │ │
│ └─────────────────────────────────┘  └─────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Notas sobre o layout final:**

- **Bloco qualitativo (esquerda):** preservado da cleanup v3.3. Inclui painéis afetados, observação esperada, horizonte de acompanhamento E linha "Custo Opus projetado (evento único)" quando o estimator pré-existente atender a chave (8 das 20 chaves). Para chaves cobertas só por IMPACT_SOURCES e não por estimator, a linha de custo é OMITIDA do bloco qualitativo (alinhado a D-PS-67 cleanup v3.3 + D-PS-87 desta sprint).
- **Bloco quantitativo (direita):** EXPANDIDO desta sprint. Tabela de contagem (current vs proposto + delta) + histograma + seletor de janela. Para chaves só com estimator (sem IMPACT_SOURCES), bloco quantitativo exibe placeholder "Análise quantitativa não disponível para esta chave nesta versão" — mas é caso teórico improvável dado o desenho.

#### §5.1.2 — Componente `ImpactPreview.tsx` — expansão sobre o pré-existente da v3.3

**Componente alvo:** `components/admin/pipeline-config/EditModal.tsx` — layout grid já existe da cleanup v3.3.
**Componente expandido:** `components/admin/pipeline-config/ImpactPreview.tsx` — já existe da cleanup v3.3, consumindo apenas `affected_count` + `projected_event_cost_usd`. Esta sprint AMPLIA o componente para consumir também `current_impact` + `proposed_impact` + `histogram` + `sample_status` + janela [7d/30d/90d].

```tsx
type Props = {
  configKey: string;
  currentValue: string;
  newValue: string;
};

const VALID_WINDOWS = [7, 30, 90] as const;

export function ImpactPreview({ configKey, currentValue, newValue }: Props) {
  const [selectedDays, setSelectedDays] = useState<7 | 30 | 90>(30);
  const debouncedNewValue = useDebounce(newValue, 500);

  // Endpoint POST pré-existente da cleanup v3.3, agora retorna payload expandido
  const { data, isLoading, error } = useSWR(
    debouncedNewValue && debouncedNewValue !== currentValue
      ? [`/api/admin/pipeline-config/${encodeURIComponent(configKey)}/impact-preview`,
         { method: 'POST', body: JSON.stringify({ new_value: debouncedNewValue, days: selectedDays }) }]
      : null,
    swrPOSTFetcher,
    { revalidateOnFocus: false }
  );

  if (!newValue || newValue === currentValue) {
    return <div className="text-sm text-muted">Digite um valor novo para ver o impacto quantitativo.</div>;
  }

  if (error?.status === 400) {
    return <div className="text-sm text-red-600">Janela inválida.</div>;
  }

  if (isLoading) {
    return <div className="text-sm">Calculando impacto...</div>;
  }

  if (!data) return null;

  // Caso 1: chave de confidence (out-of-scope v1 desta sprint)
  if (data.sample_status === 'unsupported_in_v1') {
    return (
      <div className="text-sm text-muted">
        Análise quantitativa não disponível para esta chave nesta versão.
        Edição não é bloqueada — use a caixa qualitativa ao lado para orientação.
      </div>
    );
  }

  return (
    <div className="space-y-3">
      {/* Seletor de janela DENTRO do componente */}
      <div className="flex gap-2 text-xs">
        <span className="text-muted">Janela:</span>
        {VALID_WINDOWS.map(d => (
          <button
            key={d}
            type="button"
            onClick={() => setSelectedDays(d)}
            className={selectedDays === d ? 'font-bold underline' : 'text-muted'}
          >
            {d}d
          </button>
        ))}
      </div>

      {/* Tabela de contagem — SEMPRE exibida quando current_impact existe */}
      {data.current_impact && data.proposed_impact && (
        <ImpactTable
          current={data.current_impact}
          proposed={data.proposed_impact}
        />
      )}

      {/* Badge de amostra insuficiente — substitui histograma */}
      {data.sample_status === 'insufficient' && (
        <div className="border-l-4 border-amber-500 bg-amber-50 p-2 text-xs">
          <strong>Amostra insuficiente para distribuição confiável.</strong>{' '}
          Apenas {data.sample_size} {data.sample_size === 1 ? 'item' : 'items'} nesta janela
          (mínimo: {data.sample_threshold?.absolute_min ?? 30} absoluto ou{' '}
          {data.sample_threshold?.relative_min_pct_of_universe ?? 5}% do universo).
          Considere janela maior ou aguarde mais dados.
        </div>
      )}

      {/* Histograma — apenas quando amostra suficiente */}
      {data.sample_status === 'sufficient' && data.histogram && (
        <ImpactHistogram
          buckets={data.histogram.buckets}
          currentValue={Number(currentValue)}
          newValue={Number(debouncedNewValue)}
        />
      )}

      <div className="text-xs text-muted">
        Análise sobre janela de {data.window_days} dias. Fonte: {data.source}.
      </div>
    </div>
  );
}
```

#### §5.1.3 — Caixa qualitativa pré-existente — linha de custo Opus

A caixa qualitativa preservada da cleanup v3.3 já renderiza linha de custo Opus quando o estimator pré-existente atende a chave editada. Esta sprint NÃO altera essa linha — terminologia `projected_event_cost_usd` + rótulo "Custo Opus projetado (evento único)" + nota de fallback quando `cost_is_fallback=true` continuam exatamente como a cleanup v3.3 entregou. D-PS-87 desta sprint reafirma sem modificar.

Quando a chave editada está coberta apenas por IMPACT_SOURCES (sem estimator pré-existente), a caixa qualitativa **omite** a linha de custo (campos `affected_count` e `projected_event_cost_usd` retornam null no payload — ver §4.1.4). Bloco qualitativo continua mostrando painéis afetados + observação esperada + horizonte de acompanhamento.

#### §5.1.4 — Detalhes técnicos do componente expandido

- `ImpactTable`: render simples de current/proposed + % e delta absoluto. Quando `current_impact` tem `items_inside_cooldown` em vez de `items_passing` (chaves cooldown), labels adaptam para "Dentro do cooldown" / "Fora do cooldown".
- `ImpactHistogram`: SVG inline manual com 10 buckets + 2 linhas verticais (atual e proposto). Sem dependência de chart lib — escopo limitado não justifica peso extra.
- Tooltip nos buckets ao hover (ex: "0.70-0.75: 1.245 items, 92% aprovados") — entrega no MVP.
- Debounce de 500ms no `newValue` — preservado da cleanup v3.3, mesmo valor já em uso no componente original.
- AbortController em request inflight — preservado da cleanup v3.3.
- Cache via SWR sem revalidação on focus.
- Botão de salvar do modal **NÃO** é bloqueado em nenhum cenário — admin tem autonomia para decidir mesmo com sinal incompleto.

---

### §5.2 — Refator UI dos painéis 2.1, 2.7, 2.8, 2.9 (Frente B parte UI)

**Atende:** consumo dos campos novos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` materializados em `dashboard_daily_summary.data` pela expansão de §3.6.
**SUB-PR:** 10 (sequencial após SUB-PR 9 — campos materializados precisam estar populados).

**Pré-requisito:** SUB-PR 9 mergeado e backfill rodado — `data.operational.o7_skill`, `data.operational.o8_skill`, `data.operational.o9_skill`, `data.operational.items_processed` preenchidos em todos os 26 dias rolling.

#### §5.2.1 — Decisão arquitetural cravada — adição lateral, não substituição (D-PS-93)

Os 3 painéis O7/O8/O9 hoje (`components/admin/dashboard/OperationalTab.tsx` ou similar — confirmar via E0e em §6.1) consomem exclusivamente `data.operational.o7`, `data.operational.o8`, `data.operational.o9` (todos role-only). Refator desta sprint **não toca** esses campos pré-existentes (D-PS-92), apenas adiciona consumo dos campos novos paralelos:

| Painel | Campo pré-existente (mantido) | Campo novo (consumido lateralmente) |
|---|---|---|
| 2.1 KPI "Habilidades Canônicas" | `live.operational.skillsActive` + delta `skills_active_added` | adicional `data.operational.items_processed` (D-PS-94) |
| 2.7 O7 (drift) | `data.operational.o7.{canonicals_novos, vagas_curadas, drift_percent}` | `data.operational.o7_skill.{canonicals_novos, items_processed, drift_percent}` |
| 2.8 O8 (Caminho de resolução) | `data.operational.o8.{camada0, camada1, llm_direct, quarantined, total}` | `data.operational.o8_skill.{slug_match, alias_match, llm_new, race_recovered, gate_rejected, fallback_error, total}` |
| 2.9 O9 (Saúde) | `data.operational.o9.{quarantined, disobeyed, sem_canonico, total}` | `data.operational.o9_skill.{fallback_error, sem_canonical, status_summary, total}` |

Critério de design (cravado): cada painel ramifica em **dois blocos lado a lado ou empilhados** (decisão por painel, conforme espaço visual disponível em cada layout), com header indicando "Funções" e "Habilidades" no respectivo bloco. Sem toggle, sem opção de view — ambas as séries são sempre visíveis. Justificativa: a paridade role↔skill é exatamente o que admin precisa enxergar para validar a saúde simétrica do pipeline pós-MVP.

#### §5.2.2 — Painel 2.1 — KPI novo "Items processados pelo pipeline" (D-PS-94)

A linha 2.1 hoje tem 5 KPIs (Vagas curadas, Saldo Fantastic Jobs, Uploads, Créditos consumidos, Habilidades Canônicas).

Decisão: **adicionar tooltip no KPI "Habilidades Canônicas" pré-existente** mostrando `items_processed.skill` no período (não criar 6º card que quebraria o layout 5-colunas pré-existente). Header do KPI mantido como está. Tooltip aparece on-hover com texto:

```
Itens processados pelo pipeline na janela
─────────────────────────────────────────
Funções:    23.450 items   (FORI)
Habilidades: 18.230 items   (FOSI)
Total:       41.680 items
```

Dados consumidos de `data.operational.items_processed.{role, skill, total}`. Em janelas 7d/30d, somatório cross-day dos rows correspondentes. Em janela 24h, o row do dia atual (ou estimativa em tempo real via dashboard-events — coerente com o que o KPI principal já faz).

Refator: ~30 min — adição de `<Tooltip>` ao `<KpiCard>` existente.

#### §5.2.3 — Painel 2.7 — Bloco "Habilidades Novas" ao lado de "Funções Novas"

Layout atual (CSS pré-existente): big number text-3xl colorido centralizado + 2 MetricRows laterais (`canonicals_novos`, `vagas_curadas`).

Layout novo: dois containers verticais lado a lado em `grid grid-cols-2`. Cada container exibe o mesmo conjunto (big number drift_percent + 2 MetricRows) para sua entidade respectiva:

```
┌────────────────────────────┬────────────────────────────┐
│ Funções Novas              │ Habilidades Novas          │
│                            │                            │
│        14%                 │        9%                  │
│   (drift_percent)          │    (drift_percent)         │
│                            │                            │
│  ▸ canonicals novos: 142   │  ▸ canonicals novos: 87    │
│  ▸ vagas curadas: 1.014    │  ▸ items processados: 953  │
└────────────────────────────┴────────────────────────────┘
```

Cores: ambos os blocos usam a mesma palette pré-existente (Verde < 10%, Âmbar 10-20%, Vermelho > 20%). Pode haver divergência entre os dois (role verde + skill âmbar é cenário possível e informativo).

**Diferenças semânticas a documentar visualmente:**

- Funções: drift = `canonicals_novos / vagas_curadas` (denominador é vaga inteira)
- Habilidades: drift = `canonicals_novos / items_processados` (denominador é cada skill individual; vaga única produz N items FOSI)

Tooltip no header de cada container explica essa diferença de denominador para o admin não confundir as duas escalas.

#### §5.2.4 — Painel 2.8 — Duas barras horizontais stacked paralelas

Layout atual: 1 barra horizontal stacked com 4 camadas (Camada 0 verde, Camada 1 azul, LLM amarelo, Quarentena vermelho) + tabela detalhada abaixo + botão "▸ detalhe" abrindo modal de hint distribution.

Layout novo: 2 barras stacked empilhadas verticalmente (mais legível que side-by-side dado o detalhe da legenda):

```
Funções (FORI):
[█████████████████████░░░░░░░░░░░░░░░░░] 60% Camada 0  15% C1  20% LLM  5% Q

Habilidades (FOSI):
[████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 22% Slug  35% Alias  30% LLM  10% Race  3% Erro
```

**Mapeamento de paths skill (do enum FOSI):**

- `slug_match` (verde) — equivalente conceitual de Camada 0 (cache hit por slug exato)
- `alias_match` (azul) — equivalente conceitual de Camada 1 (resolução via dicionário de aliases)
- `llm_new` (amarelo) — LLM direto (canonical criado novo)
- `race_recovered` (cinza) — recuperação de race condition (não conta como problema)
- `gate_rejected` (laranja escuro) — não exibido na barra principal (faz parte do "sem canônico", aparece no 2.9)
- `fallback_error` (vermelho) — erro

Tabela detalhada abaixo das duas barras: linhas paralelas role/skill com contagens absolutas + percentuais.

KPI principal continua sendo "% Camada 0 (Funções)" e ganha par "% Slug match (Habilidades)" — verde ≥ 30%, âmbar 15-30%, vermelho < 15% (faixas mais brandas que role porque skill tem cardinalidade maior).

Botão "▸ detalhe da Camada 2 (Funções)" pré-existente é preservado. **Não criar** botão equivalente para skills nesta sprint (drilldown de hints LLM é específico do path role hoje; modal análogo para skill é frente futura).

#### §5.2.5 — Painel 2.9 — Dois badges de status lado a lado

Layout atual: badge texto "OK / Atenção / Crítico" + 3 MetricRows (Quarentena, Disobeyed, Sem canônico).

Layout novo: dois containers verticais com mesma estrutura:

```
┌────────────────────────────┬────────────────────────────┐
│ Funções        [OK]        │ Habilidades   [ATENÇÃO]    │
│                            │                            │
│ ▸ Quarentena LLM:    0.8%  │ ▸ Falha pipeline:    2.1%  │
│ ▸ LLM desobedeceu:   0.2%  │ ▸ Sem canônico:      0.0%  │
│ ▸ Sem canônico:      0     │                            │
└────────────────────────────┴────────────────────────────┘
```

Faixas para skill (cravadas):

- Falha pipeline (`fallback_error_rate`): Verde < 2%, Âmbar 2-5%, Vermelho ≥ 5%
- Sem canônico (`sem_canonical`): deve ser sempre 0 (estado terminal inválido) — qualquer valor > 0 é Vermelho

Worst-of-two define cor do badge skill (análogo ao worst-of-three pré-existente do role).

#### §5.2.6 — Comportamento histórico

Em janelas 7d/30d (`days=7` ou `days=30`), todos os 4 painéis somam cross-day os campos respectivos do JSONB. Em janela 24h, lê o row do dia atual (mais recente).

Para dias anteriores ao SUB-PR 6 (FOSI vazia), valores skill aparecem como 0 (não como erro). Tooltip nos containers skill em janelas que cobrem esses dias informa: "Habilidades só passaram a ser instrumentadas em <data SUB-PR 6 deploy>. Valores antes dessa data refletem zero por design, não problema operacional."

#### §5.2.7 — Layout responsivo

Em viewport < 768px (mobile/tablet), os 4 painéis colapsam de grid 2-colunas para empilhamento vertical (role em cima, skill embaixo) preservando legibilidade. Componente atual da `OperationalTab` provavelmente já tem grid responsivo — adaptação trivial.

#### §5.2.8 — Resumo de mudanças por painel

| Painel | Esforço | Risco visual |
|---|---|---|
| 2.1 (tooltip) | ~30 min | Mínimo — adição de tooltip ao card existente |
| 2.7 (refator container) | ~2h | Baixo — replica estrutura existente em grid 2-col |
| 2.8 (segunda barra + legenda) | ~3h | Médio — nova legenda com 6 estados; testar contraste e text-overflow |
| 2.9 (refator container) | ~1.5h | Baixo — replica estrutura existente em grid 2-col |

Esforço total: ~7h coding + ~3h testes visuais + ajustes = 1.5 dia. Estimativa SUB-PR 10 = 2 dias com folga para revisão.

---

## §6 Auditorias operacionais

### §6.1 — Auditoria de scripts ad-hoc + grep de call sites + verificação de helpers

**Atende:** §0.3 escopo + §3.1 (helpers) + §3.2 (call sites) + §3.4 (call sites pós-rename) + E0 ground truth pré-PR.
**SUB-PR:** 3.

#### §6.1.0 — Evidence E0 obrigatório ANTES do SUB-PR 5 e SUB-PR 9 (D-PS-88)

**E0 cobre 5 sub-evidences (E0a–E0e):**

- E0a–E0d resolvem a ambiguidade do nome da função de descoberta (`discoverAndLinkSkills` vs `safeDiscoverAndLinkSkills`) — ver §3.1.0. **Conforme ground truth de Claude Code em 2026-05-14: Cenário B confirmado** — `safeDiscoverAndLinkSkills` em `lib/pipeline/persist-curation/skill-mapper.ts:37` é wrapper void de `discoverAndLinkSkills` em `lib/pipeline/ingest-job-and-discover-skills.ts:48` (retorna `Promise<DiscoverResult>`). Caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ em `await safeDiscoverAndLinkSkills(supabase, normalized);` (M60 refator). Evidence E0 da implementação atual confirma essa fotografia e detecta drift caso o codebase tenha mudado entre a spec e o SUB-PR 5.

- E0e é auditoria nova adicionada nesta sprint v1.4 (Opção 2): mapa do `aggregateDayData()` atual em `lib/admin/dashboard-day-aggregator.ts` para entender estrutura JSONB pré-existente antes de expandir (SUB-PR 9).

```bash
# ─── E0a a E0d — função de descoberta ───

# E0a — qual nome existe?
grep -rn 'discoverAndLinkSkills\|safeDiscoverAndLinkSkills' lib/ app/ scripts/ \
  --include='*.ts' --include='*.tsx'

# E0b — qual arquivo declara cada nome (procura "export ... function ... <name>")?
grep -rn 'export.*function.*\(discoverAndLinkSkills\|safeDiscoverAndLinkSkills\)' lib/ \
  --include='*.ts'

# E0c — qual nome o caller em persist-curation/persist-fn.ts chama?
grep -n 'discoverAndLinkSkills\|safeDiscoverAndLinkSkills' \
  lib/pipeline/persist-curation/persist-fn.ts

# E0d — wrapper ou função única? Procura por chain interno
grep -rn 'await discoverAndLinkSkills' lib/ --include='*.ts'

# ─── E0e — aggregator atual (novo nesta sprint v1.4) ───

# E0e1 — onde está aggregateDayData declarado e como?
grep -rn 'export.*function aggregateDayData' lib/ --include='*.ts'

# E0e2 — quais famílias compõem o JSONB hoje? Procura por keys de retorno
grep -n "^\s*\(operational\|ai\|resources\|communications\|auth\|errors\|accumulators\)\s*:" \
  lib/admin/dashboard-day-aggregator.ts

# E0e3 — quem chama aggregateDayData (CRON + backfill)?
grep -rn 'aggregateDayData' app/ lib/ scripts/ --include='*.ts'

# E0e4 — quais sub-funções já existem para a família operational (o7/o8/o9 pré-existentes)?
grep -rn 'o7\|o8\|o9\|drift_percent\|camada0\|camada1' \
  lib/admin/dashboard-day-aggregator.ts
```

**Saídas esperadas (registrar no `AUDIT-orchestrator-symmetry-<data>.md`):**

- E0a deve retornar ≥1 hit. Se zero, há renomeação intermediária que precisa de investigação adicional.
- E0b identifica qual arquivo é o "owner" da implementação.
- E0c identifica qual nome o caller atual usa — esse é o nome que será modificado. Resultado esperado: `await safeDiscoverAndLinkSkills(supabase, normalized);` em `lib/pipeline/persist-curation/persist-fn.ts:410`.
- E0d, se retornar hits **dentro de um arquivo que contém `safeDiscoverAndLinkSkills`**, confirma Cenário B (wrapper externo + função interna — esperado).
- E0e1 confirma a localização do aggregator.
- E0e2 lista famílias existentes no JSONB — onde os campos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` desta sprint devem ser inseridos (dentro de `operational`).
- E0e3 confirma quem chama o aggregator — pelo menos o endpoint de backfill e o CRON noturno.
- E0e4 mapeia sub-funções pré-existentes para o7/o8/o9 role; SUB-PR 9 cria sub-função `aggregatePipelineOrchestratorMetrics` SEM tocar nessas.

**Gate de bloqueio:** SUB-PR 5 só pode iniciar após E0a–E0d ser registrado, com decisão explícita entre Cenário A e Cenário B (ver §3.1.0). SUB-PR 9 só pode iniciar após E0e ser registrado, com mapa das famílias pré-existentes. Implementador documenta no PR descritor: "E0a–E0d → Cenário B confirmado" + "E0e → aggregator mapeado, sub-função nova confirmada não-disruptiva".

#### §6.1.1 — Procedimento geral de auditoria

```bash
# 1. Mapa geral de referências
grep -rn 'function_orchestrator' . \
  --include="*.ts" --include="*.tsx" --include="*.sql" --include="*.py" \
  --exclude-dir=node_modules --exclude-dir=.next \
  > /tmp/audit-orchestrator.txt

# 2. Call sites da função de descoberta (já coberto por E0)
# Saída do E0a/E0b/E0c já registrada acima

# 3. Verificação de helpers do refator §3.1
grep -rn 'classifyError\|lookup_canonical_skill_by_normalized_alias\|resolve_active_canonical_by_slug' \
  lib/ app/ --include='*.ts' >> /tmp/audit-orchestrator.txt

# 4. Verificação de mapSkillsToCanonical (path legacy quarantined)
grep -rn 'mapSkillsToCanonical' lib/ app/ scripts/ \
  --include='*.ts' --include='*.tsx' >> /tmp/audit-orchestrator.txt

# 5. Verificação de skills passadas/rejeitadas pelo gate
grep -rn 'skill.hard_gate' lib/ app/ --include='*.ts' >> /tmp/audit-orchestrator.txt

# 6. Verificação do skill-type-guard pré-existente (D-PS-65 cleanup v3.3)
grep -rn 'skill-type-guard\|assertSkillType\|needsReview' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 7. Verificação dos estimators pré-existentes (D-PS-67 cleanup v3.3)
grep -rn 'pipeline-impact-estimators\|projected_event_cost_usd' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 8. Verificação do cache pré-existente (D-PS-68 cleanup v3.3 — REVOGADA por D-PS-86 desta sprint)
# Confirmar localização atual do cache in-process inline em route.ts:18-46 para REMOÇÃO no SUB-PR 7
grep -rn 'impactPreviewCache\|Map<string,.*CacheEntry>\|pipeline-impact-preview-cache' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 9. Auditoria EditModal + ImpactPreview inline (DV-4 — pré-requisito SUB-PR 8)
# Confirmar que ambos estão INLINE em page.tsx (não como componentes separados)
grep -rn 'EditModal\|ImpactPreview' app/admin/pipeline-config/ components/admin/pipeline-config/ \
  --include='*.tsx' --include='*.ts' >> /tmp/audit-orchestrator.txt

# 10. Auditoria padrão Limiares pré-existente (pré-requisito SUB-PR 7)
# Confirmar localização de pg.Pool, redis-cache e queries pré-existentes
grep -rn 'pg.Pool\|new Pool\|redis-cache\|INTERVAL_SQL\|limiares' lib/admin/limiares/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt

# 11. Auditoria aggregateDayData pré-existente (pré-requisito SUB-PR 9)
# Confirmar estrutura das famílias do JSONB
grep -rn 'aggregateDayData\|dashboard-day-aggregator\|dashboard_daily_summary' lib/admin/ app/api/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt
```

**Decisão por categoria (entrega em `AUDIT-orchestrator-symmetry-<data>.md`):**

| Categoria | Decisão |
|---|---|
| CRONs em `app/api/cron/` que escrevem em FO_items | adicionar escrita simétrica em FO_skill_items se invocam função de descoberta; ignorar se apenas tocam FOR (que ganha colunas via §2.2) |
| Scripts em `scripts/` | se for backfill histórico, considerar script-pair; se for teste/diagnóstico, ajustar tabela nova |
| Migrations antigas (`/migrations/*.sql`) | ignorar (fotografias históricas) |
| Testes em `tests/` | atualizar fixtures e expects para refletir nome novo da tabela e tabela nova `function_orchestrator_skill_items` |
| Path quarantined (`mapSkillsToCanonical`) | confirmar que NÃO é tocado nesta sprint — D-PS-83 |
| Helpers `classifyError`, `lookup_canonical_skill_by_normalized_alias`, `resolve_active_canonical_by_slug` | confirmar existência; se ausente, escalar para PO antes de SUB-PR 5 |
| Skill-type-guard | confirmar que `needsReview` está sendo propagado no tipo `RawSkill` consumido pela função de descoberta; se não, ampliar tipo no SUB-PR 5 |
| Estimators pré-existentes | confirmar import path canônico de `pipeline-impact-estimators.ts` para reaproveitamento no endpoint refatorado (§4.1) |
| Cache in-process pré-existente | confirmar localização inline em `route.ts:18-46` (não em arquivo separado, conforme DV-1) para REMOÇÃO no SUB-PR 7 e migração para Redis (D-PS-86) |
| EditModal + ImpactPreview inline | confirmar que ambos estão inline em `app/admin/pipeline-config/page.tsx` (DV-4); extração para `components/admin/pipeline-config/EditModal.tsx` e `ImpactPreview.tsx` é pré-trabalho do SUB-PR 8 (D-PS-93) |
| Padrão Limiares (pg.Pool + Redis) | confirmar localização dos helpers compartilhados em `lib/admin/limiares/_shared/` (redis-cache, INTERVAL_SQL allowlist, pool config); reaproveitar no SUB-PR 7 |
| `aggregateDayData` em `dashboard-day-aggregator.ts` | confirmar estrutura das famílias pré-existentes no JSONB (`operational`, `ai`, `resources`, `communications`, `auth`, `errors`, `accumulators`); confirmar que `operational` é onde campos `o7`/`o8`/`o9` role pré-existentes residem — onde os novos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` devem ser inseridos por simetria (SUB-PR 9) |

**Entrega:** documento `AUDIT-orchestrator-symmetry-<data>.md` no PR final, com cada hit e decisão tomada.

---

### §6.2 — Auditoria de funções/triggers que referenciam `function_orchestrator_items` por NOME

**Atende:** §2.6 rename — toda função/trigger que tem `function_orchestrator_items` no corpo (SQL `FROM`, `DELETE FROM`, etc.) precisa ser atualizada na mesma migration.
**SUB-PR:** 3 (paralelo).

**Ground truth já confirmado (resultado parcial — sprints v14 e anteriores):**

| Objeto | Tipo | Referência a `function_orchestrator_items` | Ação no rename |
|---|---|---|---|
| `trg_foi_jcr_confidence_insert` | Trigger em FOI | implícita (tabela hospedeira) | renomear para `trg_fori_jcr_confidence_insert` |
| `trg_foi_jcr_confidence_update` | Trigger em FOI | implícita | renomear para `trg_fori_jcr_confidence_update` |
| `trg_foi_jcr_confidence_delete` | Trigger em FOI | implícita | renomear para `trg_fori_jcr_confidence_delete` |
| `fn_jcr_confidence_median_insert` | Função de trigger | NÃO referencia por nome | nenhuma ação |
| `fn_jcr_confidence_median_update` | Função de trigger | NÃO referencia por nome | nenhuma ação |
| `fn_jcr_confidence_median_delete` | Função de trigger | NÃO referencia por nome | nenhuma ação |
| `fn_recompute_jcr_confidence_median` | Função escritora | **SIM** — `FROM function_orchestrator_items foi` | atualizar corpo (CREATE OR REPLACE — modelo cravado em §2.6) |
| `reset_taxonomy_core` | Função reset | **SIM** — `DELETE FROM function_orchestrator_items` | atualizar corpo |
| `fn_promote_canonical_on_threshold` | Função promoção | a verificar | rodar query abaixo |
| `catchup_pending_promotions` | Função catchup | a verificar | rodar query abaixo |
| `maintenance_phase_1` | Função maintenance | a verificar | rodar query abaixo |
| `process_opus_create_new` | Função Opus | a verificar | rodar query abaixo |

**Query de verificação adicional (executar no SUB-PR 3):**

```sql
SELECT proname, prosrc
FROM pg_proc
WHERE prosrc LIKE '%function_orchestrator_items%';
```

Para cada função retornada com `function_orchestrator_items` no corpo, gerar bloco `CREATE OR REPLACE FUNCTION` na §2.6 substituindo por `function_orchestrator_role_items`.

**Validação simétrica de fontes de `confidence_median` (já documentada — D-PS-49 cleanup v3.3):**

- Role: `fn_recompute_jcr_confidence_median` lê de `function_orchestrator_role_items` (pós-rename), com `lookback_days=120` e `min_count=5`.
- Skill: `fn_jps_recompute_jcs` lê de `job_posting_skills` curated, com `lookback_days=120` e `min_count=3`.
- Esta sprint NÃO toca esse circuito (D-PS-75).

**Entrega:** documento `AUDIT-rename-orchestrator-<data>.md` listando todas as funções afetadas com seu corpo atualizado, anexado ao SUB-PR 4.

---

## §7 Evidence e smoke tests

### §7.1 — Blocos de evidence pré e pós-aplicação

#### E0 — Pré-SUB-PR 5: ground truth do nome da função de descoberta (D-PS-88)

Ver §6.1.0 — bloco completo de queries `bash` cobrindo E0a/E0b/E0c/E0d. Gate de bloqueio: SUB-PR 5 só inicia após decisão entre Cenário A e Cenário B ser registrada explicitamente no PR descritor.

#### E1 — Pré-rename: ground truth dos nomes de objetos existentes

```sql
-- Indices da tabela atual
SELECT indexname FROM pg_indexes
WHERE tablename = 'function_orchestrator_items';

-- Triggers
SELECT tgname FROM pg_trigger
WHERE tgrelid = 'function_orchestrator_items'::regclass
  AND NOT tgisinternal;

-- Constraints
SELECT conname FROM pg_constraint
WHERE conrelid = 'function_orchestrator_items'::regclass;

-- Funções que referenciam por nome
SELECT proname FROM pg_proc
WHERE prosrc LIKE '%function_orchestrator_items%';
```

#### E2 — Pós-rename: zero referências TS ao nome antigo

```bash
grep -rn 'function_orchestrator_items' app/ lib/ scripts/ \
  --include="*.ts" --include="*.tsx"
```

**Esperado:** 0 linhas.

#### E3 — Pós-rename: zero referências SQL em corpos de função ao nome antigo

```sql
SELECT proname FROM pg_proc
WHERE prosrc LIKE '%function_orchestrator_items%'
  AND prosrc NOT LIKE '%function_orchestrator_role_items%';
```

**Esperado:** 0 linhas.

#### E4 — Pós-aplicação: tabela `function_orchestrator_skill_items` criada

```sql
SELECT table_name FROM information_schema.tables
WHERE table_name = 'function_orchestrator_skill_items';
-- Esperado: 1 linha.

SELECT indexname FROM pg_indexes
WHERE tablename = 'function_orchestrator_skill_items';
-- Esperado: 7 índices (run_id, job_posting_id, canonical_skill_id, status, skill_confidence, pipeline_stage, needs_review).
```

#### E5 — Pós-aplicação: colunas de skill em `function_orchestrator_runs`

```sql
SELECT column_name, data_type, column_default
FROM information_schema.columns
WHERE table_name = 'function_orchestrator_runs'
  AND column_name LIKE 'skills_%'
ORDER BY column_name;
-- Esperado: 5 linhas, todas integer com default 0.
```

#### E6 — RPC `count_skill_items_by_status` criada

```sql
SELECT proname FROM pg_proc WHERE proname = 'count_skill_items_by_status';
-- Esperado: 1 linha.

-- Smoke da RPC com runId inexistente
SELECT * FROM count_skill_items_by_status('00000000-0000-0000-0000-000000000000'::uuid);
-- Esperado: 0 linhas (sem erro).
```

#### E7 — Endpoint expandido `impact-preview` responde corretamente (POST)

```bash
# 1. Chave coberta por estimator + IMPACT_SOURCES, amostra suficiente (caso normal §4.1.3):
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"0.75","days":30}'
# Esperado: 200, sample_status='sufficient', affected_count + projected_event_cost_usd + current_impact + proposed_impact + histogram presentes.

# 2. Chave coberta APENAS por IMPACT_SOURCES, amostra insuficiente (caso §4.1.4):
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.opus_review.cooldown_days/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"60","days":7}'
# Esperado: 200, sample_status='insufficient', affected_count=null + projected_event_cost_usd=null + current_impact presente + histogram OMITIDO.

# 3. Chave de confidence (out-of-scope v1 — caso §4.1.6):
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.confidence.min_count/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"4"}'
# Esperado: 200, sample_status='unsupported_in_v1', unsupported_reason texto, todos os campos quantitativos null.

# 4. Chave de sistema:
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/CURATE_PIPELINE_ENABLED/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"true"}'
# Esperado: 200, sample_status='unsupported_in_v1', unsupported_reason='Chave de sistema, não exposta na tela /admin/pipeline-config'.

# 5. Janela inválida:
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"0.75","days":14}'
# Esperado: 400, body { error: 'invalid_window', accepted: [7,30,90] }.

# 6. Missing new_value:
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{}'
# Esperado: 400.

# 7. Cache hit (consecutivo): segunda chamada idêntica em <30s deve retornar payload do cache.
# Inspecionar logs para presença de [pipeline-impact-cache] HIT key=... new_value=...
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"0.75","days":30}'
# (executar 2× consecutivos)
# Esperado: 2× HTTP 200; 2ª chamada lê do cache.

# 8. Bypass de cache via skip_cache:
curl -X POST "https://app.calibracv.com/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview" \
  -H "Content-Type: application/json" \
  -d '{"new_value":"0.75","days":30,"skip_cache":true}'
# Esperado: cache MISS forçado mesmo após hit anterior.
```

#### E8 — Cobertura IMPACT_SOURCES = chaves não-sistema de pipeline_config menos as 4 de confidence

```sql
-- 20 chaves cobertas pelo IMPACT_SOURCES nesta sprint:
SELECT key FROM pipeline_config
WHERE key NOT IN ('CURATE_PIPELINE_ENABLED', 'QUARANTINE_EXPIRY_DAYS')
  AND key NOT LIKE '%.confidence.lookback_days'
  AND key NOT LIKE '%.confidence.min_count'
ORDER BY key;
-- Esperado: 20 linhas.

-- 4 chaves de confidence fora do escopo desta sprint (frente futura D-PS-80):
SELECT key FROM pipeline_config
WHERE key LIKE '%.confidence.lookback_days' OR key LIKE '%.confidence.min_count'
ORDER BY key;
-- Esperado: 4 linhas.

-- 2 chaves de sistema:
SELECT key FROM pipeline_config
WHERE key IN ('CURATE_PIPELINE_ENABLED', 'QUARANTINE_EXPIRY_DAYS')
ORDER BY key;
-- Esperado: 2 linhas.

-- Total: 20 + 4 + 2 = 26.
-- Se total no banco for diferente, divergência de cobertura.
```

#### E9 — Cobertura de estimators pré-existentes (cleanup v3.3) preservada

```bash
# Inspecionar pipeline-impact-estimators.ts: confirmar que os 8 estimators originais
# (hard_gate.min_confidence role/skill, promotion.auto_min_confidence role/skill,
# promotion.min_vacancies role/skill, auto_assign_family.min_similarity, auto_assign_family.min_score)
# continuam ativos e exportados.
grep -E 'export.*role\.|export.*skill\.|auto_assign_family' lib/admin/pipeline-impact-estimators.ts
# Esperado: 8 hits cobrindo as 8 chaves originais.
```

#### E10 — Cobertura de skill-type-guard pré-existente (cleanup v3.3) preservada

```bash
# Confirmar que assertSkillType + needsReview continuam exportados de skill-type-guard.ts
# e usados nos 4 pontos de plumbing (D-PS-65 cleanup v3.3).
grep -rn 'assertSkillType\|skill-type-guard' lib/pipeline/ --include='*.ts'
# Esperado: ≥1 hit no arquivo do guard + N hits nos consumidores
# (normalize-result.ts, skill-mapper.ts, ingest-job-and-discover-skills.ts, canonical-skills.ts).
```

### §7.2 — Smoke tests funcionais

#### S1 — Fluxo A (admin manual JSON) escreve em `function_orchestrator_skill_items`

1. Subir JSON de teste via `/admin/ingestor` com 1 vaga contendo 3 skills variadas (uma com slug existente, uma com alias existente, uma genuinamente nova).
2. Disparar curate.
3. Verificar:

```sql
SELECT raw_skill_name, status, skill_pipeline_stage, canonical_status, needs_review
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id_de_teste>';
-- Esperado: 3 linhas, com stages 'slug_match', 'alias_match', 'llm_new' respectivamente.
-- needs_review: false em casos comuns; true se algum alias de skill_type foi normalizado (D-PS-69 cleanup v3.3).
```

#### S2 — Fluxo B (LinkedIn busca) escreve em `function_orchestrator_skill_items`

Similar S1, disparado via execução do CRON ou API automática.

#### S3 — Fluxo C (1:1 vaga colada) escreve em `function_orchestrator_skill_items`

Similar S1, via UI do usuário colando uma vaga.

#### S4 — Hard gate rejeita skills com confidence baixa e registra em FOSI

1. Definir `skill.hard_gate.min_confidence=0.80` em `pipeline_config`.
2. Subir vaga com 5 skills, sendo 2 com confidence < 0.80.
3. Verificar:

```sql
SELECT raw_skill_name, status, skill_pipeline_stage
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id>'
  AND status = 'gate_rejected';
-- Esperado: 2 linhas, ambas com skill_pipeline_stage='gate_rejected'.
```

#### S5 — Counters em `function_orchestrator_runs` batem com agregação real (RPC)

```sql
SELECT
  r.id AS run_id,
  r.skills_extracted,
  r.skills_reused,
  r.skills_pending_created,
  r.skills_gate_rejected,
  r.skills_failed,
  (SELECT SUM(c) FROM count_skill_items_by_status(r.id)) AS real_total,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'reused') AS real_reused,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'created_pending') AS real_pending,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'gate_rejected') AS real_gate,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'failed') AS real_failed
FROM function_orchestrator_runs r
WHERE r.status = 'success'
  AND r.finished_at > NOW() - INTERVAL '24 hours';
-- Esperado: para cada linha, counters batem com agregação real.
```

#### S6 — Validação de correctness sob volume >1000 (LK-CBO-31)

1. Inserir manualmente 1.500 rows em FOSI vinculadas a um único `run_id` de teste.
2. Rodar finalizer manualmente (chamar a função TS do finalizer apontando para o run de teste).
3. Verificar que `function_orchestrator_runs.skills_extracted = 1500` (não 1000).
4. Sem a RPC, esperado: skills_extracted ≤ 1000 (bug de truncamento).

#### S7 — Modal de edição exibe impacto expandido após digitar valor novo (chave coberta por estimator + IMPACT_SOURCES)

1. Abrir `/admin/pipeline-config`.
2. Clicar em "Editar" em `skill.hard_gate.min_confidence`.
3. Verificar que a caixa qualitativa pré-existente (painéis afetados + linha "Custo Opus projetado (evento único)") está visível.
4. Digitar 0.75. Aguardar 500ms.
5. Verificar que **ao lado** da caixa qualitativa aparece o ImpactPreview com tabela + histograma + seletor 7d/30d/90d.
6. Verificar que linha "Custo Opus projetado" da caixa qualitativa atualiza com `projected_event_cost_usd` do payload.
7. Trocar para 7d. Verificar nova query disparada (POST `impact-preview` com `days: 7`).
8. Trocar para 90d. Verificar nova query.

#### S8 — Modal exibe impacto expandido para chave coberta APENAS por IMPACT_SOURCES (sem estimator)

1. Editar `skill.opus_review.cooldown_days` em ambiente com dados suficientes de arbitragem.
2. Digitar 60. Aguardar.
3. ImpactPreview exibe tabela "Dentro do cooldown" / "Fora do cooldown" + histograma + seletor de janela.
4. Caixa qualitativa pré-existente **NÃO** mostra linha de "Custo Opus projetado" (campo nulo no payload).
5. Botão "Salvar" do modal continua habilitado.

#### S9 — Modal lida com amostra insuficiente

1. Editar `skill.opus_review.cooldown_days` em ambiente sem dados suficientes de arbitragem.
2. Digitar 60. Aguardar.
3. ImpactPreview exibe tabela + badge âmbar "Amostra insuficiente".
4. Histograma omitido.
5. Botão "Salvar" do modal continua habilitado.

#### S10 — Modal lida com chave fora de escopo v1 (4 chaves de confidence)

1. Editar `skill.confidence.min_count`.
2. Digitar 4. Aguardar.
3. ImpactPreview exibe mensagem "Análise quantitativa não disponível para esta chave nesta versão. Edição não é bloqueada — use a caixa qualitativa ao lado para orientação."
4. Caixa qualitativa pré-existente continua orientando.
5. Botão "Salvar" continua habilitado.

#### S11 — Modal preserva linha de custo Opus para chaves de `auto_assign_family.*` (estimator sem IMPACT_SOURCES)

1. Editar `role.auto_assign_family.min_similarity` (estimator pré-existente, fora de IMPACT_SOURCES — D-PS-41 cleanup v3.3).
2. Digitar valor novo. Aguardar.
3. Caixa qualitativa pré-existente mostra "Custo Opus projetado (evento único)" com `projected_event_cost_usd`.
4. ImpactPreview (lado direito) exibe placeholder "Análise quantitativa não disponível para esta chave nesta versão" — payload tem `current_impact=null`.
5. Botão "Salvar" continua habilitado.

#### S12 — Pós-rename: trigger antigo (`trg_foi_jcr_confidence_*`) deixou de existir; novo (`trg_fori_*`) existe

```sql
SELECT tgname FROM pg_trigger
WHERE NOT tgisinternal
  AND tgname LIKE 'trg_fo%_jcr_confidence_%';
-- Esperado: 3 linhas, todas com prefixo 'trg_fori_'.
```

#### S13 — Pós-rename: `fn_recompute_jcr_confidence_median` ainda funciona após atualização mecânica do identificador

1. Inserir 5 rows em `function_orchestrator_role_items` (renomeado) para um canonical de teste, com `confidence` entre 0.7 e 0.95.
2. Verificar:

```sql
SELECT confidence_median FROM job_canonical_roles
WHERE id = '<canonical_id_de_teste>';
-- Esperado: valor entre 0.7 e 0.95 (mediana dos 5 inseridos).
-- Confirma que apenas o identificador da tabela mudou no corpo da função — lógica preservada.
```

#### S14 — Path quarantined permanece invisível ao orchestrator (D-PS-83)

1. Forçar uma vaga a entrar em quarentena (mecanismo de quarantine).
2. Confirmar que `mapSkillsToCanonical` foi chamada (não a função de descoberta).
3. Confirmar que `function_orchestrator_skill_items` **NÃO** tem rows para essa vaga.
4. Confirmar que isso é comportamento esperado (D-PS-83).

#### S15 — Propagação de `needs_review` end-to-end (D-PS-65 + D-PS-69 cleanup v3.3)

1. Forçar LLM a emitir um alias PT-BR (`'tecnica'`) em uma skill via input controlado em ambiente de teste.
2. Verificar que `skill-type-guard.ts` normaliza para `'technical'` e marca `needsReview=true`.
3. Verificar que o item correspondente em `function_orchestrator_skill_items` tem `needs_review=true`:

```sql
SELECT raw_skill_name, skill_pipeline_stage, needs_review
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id_de_teste>'
  AND needs_review = true;
-- Esperado: ≥1 linha, com raw_skill_name correspondente à skill que teve skill_type normalizado.
```

#### S16 — Cache in-process não bloqueia requests subsequentes (D-PS-86)

1. Disparar 5 requests POST consecutivas (mesma key + new_value + days) em <30s.
2. Confirmar que todas retornam 200.
3. Confirmar via logs que apenas a primeira fez query ao DB (demais leram do cache).
4. Aguardar >30s. Disparar 1 request com mesma combinação.
5. Confirmar via logs que essa request fez query ao DB (cache expirou).
6. Confirmar que não há header de rate limit no response (não é rate limiter — D-PS-86).

---

## §8 Sub-PRs breakdown

10 SUB-PRs distribuídos em 3 frentes (Opção 2 cross-cutting):

| SUB-PR | Frente | Conteúdo | Depende de | Paralelo com cleanup v3.3? | Estimativa |
|---|---|---|---|---|---|
| 1 | A | §2.1 (tabela `_skill_items` com coluna `needs_review`) | — | sim | 0.5 dia |
| 2 | A | §2.2 (colunas em runs) + §2.3 (validador) + §2.4 (RPC count) | — | sim | 0.5 dia |
| 3 | A | §6.1 + §6.2 (auditorias — documentos) + E0a–E0e obrigatórios registrados | — | sim | 0.5 dia |
| 4 | A | §2.6 (rename + atualização mecânica de corpos de funções) + §3.4 (call sites TS) | cleanup v3.3 mergeada + SUB-PR 3 | não | 1.5 dia |
| 5 | A | §3.1 (refator função de descoberta — Cenário B confirmado por E0d; caller já em await — DV-5) | SUB-PR 4 + E0a–E0d registrado em §6.1.0 | não | 1 dia |
| 6 | A | §3.2 (insertFOSkillItem em call sites + propagação needs_review) + §3.3 (3 finalizers) | SUB-PR 5 | não | 1 dia |
| 7 | C | §3.5 (queries novas em `lib/admin/limiares/queries.ts` + enriquecimento painéis Limiares 1, 8) + §4.1 (refator endpoint `impact-preview` para padrão Limiares — `pg.Pool` + Redis TTL 300s; REMOÇÃO do cache in-process) | cleanup v3.3 mergeada + SUB-PR 6 | não | 2.5-3 dias |
| 8 | C | §5.1.0 (extração de `EditModal` + `ImpactPreview` para arquivos próprios — DV-4 opção a) + §5.1.1 a §5.1.7 (integração payload novo nos componentes extraídos) | cleanup v3.3 mergeada + SUB-PR 7 | não | 1.5 dia |
| 9 | B | §3.6 (expansão de `aggregateDayData` + sub-função `aggregatePipelineOrchestratorMetrics` + RPCs novas `count_role_items_by_stage_in_window` / `count_skill_items_by_stage_in_window`) + backfill obrigatório via `/api/admin/backfill-dashboard-summary` | SUB-PR 6 + E0e registrado | não | 2 dias |
| 10 | B | §5.2 (refator UI dos painéis 2.7 O7 + 2.8 O8 + 2.9 O9 para séries paralelas role+skill + ajuste KPI 2.1 com tooltip "Items processados pelo pipeline") | SUB-PR 9 | não | 2 dias |

**Total estimado: 12-14 dias úteis** (Opção 2 cravada em 2026-05-14).

**Fases:**

- **Fase 1 (paralela com cleanup v3.3):** SUB-PRs 1, 2, 3 — ~1.5 dia em paralelo
- **Fase 2 (sequencial Frente A após cleanup v3.3):** SUB-PR 4 → SUB-PR 5 → SUB-PR 6 — ~3.5 dias
- **Fase 3 (paralela entre Frente B e Frente C, ambas após Frente A):**
  - Frente B caminho: SUB-PR 9 → SUB-PR 10 — ~4 dias
  - Frente C caminho: SUB-PR 7 → SUB-PR 8 — ~4-4.5 dias
- **Janela crítica total:** ~9-10 dias se Fase 3 for fielmente paralela; ~13-14 dias se Fase 3 for sequencial pessoa-única

Janela mais provável dado contexto operacional (uma equipe de implementação executando sequencialmente dentro de cada frente, com algum paralelismo cirúrgico):

```
D+0     ┃ SUB-PRs 1, 2, 3 (paralelo com cleanup v3.3 ainda em revisão)
D+1.5   ┃ cleanup v3.3 mergeada → libera Fase 2
D+1.5   ┃ SUB-PR 4 (rename)
D+3     ┃ SUB-PR 5 (refator função de descoberta)
D+4     ┃ SUB-PR 6 (insertFOSkillItem + finalizers)
D+5     ┃ Fase 3 abre — paralelo entre B e C
D+5     ┃ SUB-PR 7 começa (Frente C — endpoint)
D+5     ┃ SUB-PR 9 começa (Frente B — aggregator)
D+7.5   ┃ SUB-PR 7 fecha → libera SUB-PR 8
D+7     ┃ SUB-PR 9 fecha → libera SUB-PR 10
D+8.5   ┃ SUB-PR 8 (extração componentes + integração) começa
D+7     ┃ SUB-PR 10 (refator UI painéis dashboard global) começa
D+10    ┃ SUB-PR 8 fecha
D+9     ┃ SUB-PR 10 fecha
D+10    ┃ Sprint completa
```

Estimativa otimista: 10 dias úteis. Estimativa conservadora (com revisão multi-AI entre cada SUB-PR e tempo para fixes): 12-14 dias úteis.

---

## §9 D-PS, S-ORCH, LK-PS, decisões herdadas

### Decisões de produto (D-PS) desta sprint

D-PS desta sprint numeradas a partir de D-PS-70 para evitar colisão com D-PS-01 a D-PS-69 cravadas na cleanup v3.3 (que mantêm seus números originais quando referenciadas).

- **D-PS-70 (rename `function_orchestrator_items` na mesma sprint):** janela de zero rows em produção pré-MVP torna o rename de baixíssimo risco. Postergar criaria dívida permanente. Custo mapeado em §2.6 + §3.4 + §6.2. Atualização do corpo de funções dependentes (§2.6) é mecânica de rename — D-PS-49 da cleanup v3.3 cravou que "sprint orchestrator NÃO TOCA o circuito de confidence_median"; trocar identificador da tabela no `FROM` da função não conta como toque.

- **D-PS-71 (cobertura do endpoint refatorado = 22 chaves com payload):** o endpoint `impact-preview` no padrão Limiares cobre **22 chaves operacionais com algum payload preenchido** (26 operacionais menos 4 de confidence). Decomposição: 6 chaves com payload completo (estimator + IMPACT_SOURCES); 14 chaves com payload via IMPACT_SOURCES (sem estimator); 2 chaves com payload via estimators apenas (`auto_assign_family.*` role-only — assimetria D-PS-41 cleanup v3.3); 4 chaves de confidence retornam `sample_status='unsupported_in_v1'` (D-PS-80). Total banco operacional: 26 (24 originais com simetria role/skill + 2 `auto_assign_family.*` adicionadas pela cleanup v3.3) + 2 sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS` — filtradas no UI por §5.1.0 cleanup v3.3) = 28 total. **Correção DV-2 cravada em 2026-05-14:** v1.3 referenciava "24 operacionais" desatualizado; 26 é o número correto após cleanup v3.3.

- **D-PS-72 (visualização consolidada quantitativa — sem toggle, adição lateral):** dentro do ImpactPreview, tabela + histograma lado a lado, sem comutador. Mostrar ambos sempre = decisão tomada uma única vez. Esta caixa é **adição** lateral à caixa qualitativa pré-existente da cleanup v3.3 — não substitui. Caixa qualitativa preserva linha de "Custo Opus projetado (evento único)" quando estimator pré-existente atende a chave (8 chaves); para as 12 chaves restantes da cobertura desta sprint, linha de custo é omitida.

- **D-PS-73 (paralelismo β-light controlado por SUB-PR):** apenas SUB-PRs 1, 2, 3 paralelos com cleanup v3.3. Demais sequenciais. Justificativa: precedentes de paralelismo descontrolado (SMS Zenvia Sub-PR19, opus-budget-guard.ts Sub-PR14a) ensinaram que paralelismo amplo aumenta área de atenção do PO. β-light limita paralelos a frentes cirurgicamente delimitadas.

- **D-PS-74 (trigger `confidence_median` em JCS NÃO criado nesta sprint):** ground truth confirmou que `fn_jps_recompute_jcs` já popula `job_canonical_skills.confidence_median` a partir de `job_posting_skills` curated. Criar trigger novo a partir de `function_orchestrator_skill_items` seria duplicação E conflito com a fonte estabelecida. Sprint orchestrator NÃO toca esse circuito (alinhado a D-PS-49 da cleanup v3.3).

- **D-PS-75 (assimetria de fontes de `confidence_median` mantida — referencia D-PS-49 da cleanup v3.3):** a tabela `function_orchestrator_skill_items` criada nesta sprint poderia ALIMENTAR `job_canonical_skills.confidence_median` para simetria total de fonte com role. A decisão arquitetural cravada em D-PS-49 da cleanup v3.3 é manter skill lendo de `job_posting_skills` curated — a robustez do sinal pós-curadoria é trade-off intencional. Esta sprint reafirma sem modificar.

- **D-PS-76 (janela do endpoint expandido):** janela fixa default 30 dias + seletor discreto `[7d, 30d, 90d]` dentro do componente `ImpactPreview` (não no modal externo). Valores arbitrários rejeitados com HTTP 400. Sem opção "todo histórico" — exclui período pré-rename e dados de migração que poluiriam o sinal.

- **D-PS-77 (threshold de amostra suficiente — critério OR):** N >= 30 itens OU N >= 5% do universo total → suficiente. Caso contrário, insuficiente. Critério OR (não AND) — permite reconhecer caso onde N é pequeno mas é 100% do universo (e.g., chave nova com 5 arbitragens totais; 5/5 = 100% e é o melhor sinal possível).

- **D-PS-78 (comportamento sob amostra insuficiente):** tabela de contagem SEMPRE exibida quando `current_impact` existe. Histograma OMITIDO quando sample_status='insufficient'. Badge âmbar informa o motivo. Edição NÃO bloqueada — admin tem autonomia.

- **D-PS-79 (enum `skill_pipeline_stage` reflete paths reais de skill — divergente do enum role):** ground truth do banco confirmou que `function_orchestrator_items.pipeline_stage` (role) tem 6 valores (`deterministic`, `cache_hit_layer_0`, `dict_match_layer_1`, `suggested_role_layer_2`, `llm_pure_layer_3`, `fallback_error`) que refletem a arquitetura de 4 camadas pré-LLM do pipeline de role + override pós-LLM determinístico. A função de descoberta de skills (caminho real de skill) tem 5 paths conceituais distintos (`slug_match`, `alias_match`, `llm_new`, `race_recovered`, `fallback_error`) + 1 path injetado pelo caller (`gate_rejected`). Mapeamento:

  | Path de role | Equivalente em skill? | Justificativa |
  |---|---|---|
  | `cache_hit_layer_0` | NÃO | Cache de SHA-256 opera sobre descrição da vaga inteira; função de descoberta de skills opera por skill individual |
  | `dict_match_layer_1` | SIM (`slug_match` + `alias_match`) | Skills têm 2 sub-paths conceitualmente equivalentes |
  | `suggested_role_layer_2` | NÃO | Skills não têm sistema de famílias/domains; sem sugestão por canônico próximo |
  | `llm_pure_layer_3` | SIM (`llm_new`) | Skill nova cria pending após não-matches |
  | `deterministic` | NÃO | Não há multi-skill split nem Hard Rules de skill (override pós-LLM) |
  | `fallback_error` | SIM | Equivalente direto |

  Paths exclusivos de skill: `race_recovered` (recuperação via `resolve_active_canonical_by_slug` após colisão de slug — D-PS-66 cleanup v3.3), `gate_rejected` (rejeição pelo hard gate de skill antes da função de descoberta). Validador §2.3 NÃO compara enums de stage — apenas `status` e `error_type`.

- **D-PS-80 (4 chaves de `confidence` fora do escopo do endpoint expandido v1):** `{role,skill}.confidence.{lookback_days, min_count}` controlam o circuito interno de cálculo de `confidence_median`. Simular impacto de mudança nessas chaves requer recalcular agregação de `PERCENTILE_CONT(0.5)` para cada canonical afetado dentro da nova janela ou novo `min_count` — operação O(n) sobre cada canonical com volume relevante, inadequada para endpoint síncrono. Frente futura: sprint dedicada de "calibração de circuito de confidence" com cache de pré-cálculo. Endpoint retorna HTTP 200 com `sample_status='unsupported_in_v1'` para essas 4 chaves (paridade com o tratamento que o endpoint pré-existente da cleanup v3.3 já dava a chaves não cobertas pelos 8 estimators).

- **D-PS-81 (RPC SQL obrigatória para finalizers — LK-CBO-31):** Supabase JS trunca paginação em 1000 rows por default. Para runs com >1000 skills, agregar status via `.from('function_orchestrator_skill_items').select('status').eq('run_id', runId)` retorna até 10× menor que real — bug silencioso de correctness. RPC `count_skill_items_by_status` (§2.4) é obrigatória nos 3 finalizers. Conformidade com LK-CBO-31.

- **D-PS-82 (logging estruturado em `insertFOSkillItem` — LK-CBO-30):** falhas de INSERT em FOSI logam estrutura `{run_id, job_posting_id, raw_skill_name, error_code, error_message}` e incrementam `logging_failures` em `function_orchestrator_runs` (campo pré-existente). Conformidade com LK-CBO-30 (error logging estruturado obrigatório em upserts). Comportamento permanece non-blocking — pipeline produtivo não é interrompido por falha de telemetria.

- **D-PS-83 (path quarantined fora de escopo desta sprint):** D-PS-64 da cleanup v3.3 cravou que a função de descoberta (`discoverAndLinkSkills` ou wrapper `safeDiscoverAndLinkSkills` conforme E0) é o único path em caminho curated; `mapSkillsToCanonical` permanece em caminho quarantined com semântica mínima (sem creation, sem CBO link, sem confidence_median bootstrap). Esta sprint NÃO instrumenta `mapSkillsToCanonical` — skills processadas no caminho quarantined permanecem **invisíveis** ao orchestrator. Decisão: aceitar gap. Razão: caminho quarantined é exceção operacional (vagas em quarentena) e a sprint orchestrator é arquitetura de simetria do caminho curated.

- **D-PS-84 (assimetria estrutural ampla dos contadores em FOR):** ground truth confirmou que `function_orchestrator_runs` tem 8 contadores **outcome-oriented** pré-existentes role-only (`total`, `curated`, `curated_fallback`, `low_quality`, `failed`, `pending_review`, `canonical_created`, `canonical_promoted`). Esta sprint adiciona 5 contadores **stage-oriented** de skill (`skills_extracted`, `_reused`, `_pending_created`, `_gate_rejected`, `_failed`). As duas semânticas são distintas: outcome-oriented agrega resultados finais por vaga (uma vaga → uma linha); stage-oriented agrega items pelo estágio em que cada skill foi resolvida. Defender simetria estrutural plena exigiria refatorar contadores role para também serem stage-oriented — fora de escopo. Esta sprint reconhece a assimetria como dívida arquitetural e documenta a divergência semântica via COMMENT em `skills_extracted`. **Amplia o recorte da D-PS-50 da cleanup v3.3** (que cobre 2 dos 8 contadores — `canonical_created` e `canonical_promoted`) para reconhecer os 8 contadores integrais pré-existentes.

- **D-PS-85 (endpoint `impact-preview` migrado para padrão Limiares — não endpoint novo, não cache in-process):** cravado pelo product owner em 2026-05-14, refinado em 2026-05-14 para Opção 2 cross-cutting. O endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` da cleanup v3.3 é MIGRADO por esta sprint para o **padrão arquitetônico da aba Limiares** mantendo método HTTP POST, nome canônico, terminologia `projected_event_cost_usd`. Mudanças arquiteturais: (a) conexão via `pg.Pool` direto em vez de Supabase client; (b) cache Redis TTL 300s em vez de Map in-process TTL 30s; (c) allowlist de intervalos `[7d, 30d, 90d]`; (d) `Promise.allSettled` para tolerância a falha individual de estimator vs IMPACT_SOURCES. Esta sprint adiciona blocos novos ao payload (`current_impact`, `proposed_impact`, `histogram`, `sample_status`, `window_days`, `source`, `unsupported_reason`) para cobertura de 20 chaves via IMPACT_SOURCES. Os 8 estimators pré-existentes da cleanup v3.3 continuam ativos e seus campos pré-existentes (`affected_count`, `projected_event_cost_usd`, `cost_is_fallback`) permanecem no payload. Sem fragmentação em endpoints separados, sem deprecação. Justificativa: padrão Limiares é necessário para queries com `WIDTH_BUCKET`, CTEs, `UNION ALL` (PostgREST não expressa); compartilha helpers e disciplina de cache com a aba Limiares; coerência arquitetônica do ecossistema de painéis de calibração.

- **D-PS-86 (cache in-process Map TTL 30s REVOGADO — substituído pelo padrão Redis Limiares):** **D-PS-68 da cleanup v3.3 é REVOGADA por esta sprint.** O cache `Map<string, CacheEntry>` com TTL 30s + eviction simples ao passar de 200 entradas (inline em `app/api/admin/pipeline-config/[key]/impact-preview/route.ts:18-46` — DV-1 cravado em 2026-05-14: o helper file `lib/admin/pipeline-impact-preview-cache.ts` referenciado por LK-PS-10 da v1.3 NÃO EXISTE; o cache está inline no route.ts) é **REMOVIDO desta sprint**. Substituído pelo padrão Redis da aba Limiares: TTL 300s (`limiares:impact_preview:{key}:{new_value}:{days}`), fall-through silencioso em caso de erro Redis (padrão Limiares). Vantagens: (1) compartilhado entre instâncias serverless Vercel (não evapora a cada deploy/restart); (2) TTL 10× maior reduz queries em sessão típica de admin; (3) coerência arquitetônica com os 10 painéis Limiares; (4) elimina bug latente "cache stale após restart silencioso". Continua sendo memoização, NÃO rate limiter (não bloqueia, não tem quota, não expõe headers). Não viola diretriz "sem novos controles operacionais sem autorização" — substituição de mecanismo equivalente, não adição de controle.

- **D-PS-87 (terminologia `projected_event_cost_usd` preservada — D-PS-67 cleanup v3.3):** o campo `projected_event_cost_usd` do payload mantém semântica de "evento único, NÃO recorrência mensal" cravada pela cleanup v3.3. UI continua usando o rótulo canônico "Custo Opus projetado (evento único)" na caixa qualitativa pré-existente quando estimator pré-existente atende a chave. Para chaves cobertas apenas por IMPACT_SOURCES (sem estimator), o campo retorna null e a UI omite a linha — alinhado a D-PS-67 cleanup v3.3 e diretriz DC-2 (chaves não mapeadas escondem linhas dinâmicas).

- **D-PS-88 (ground truth condicional do nome da função de descoberta):** o nome real da função de descoberta de skills no codebase atual (`discoverAndLinkSkills` vs `safeDiscoverAndLinkSkills`) foi resolvido via evidence E0a–E0d antes desta sprint. **Resultado cravado em 2026-05-14:** Cenário B confirmado — `safeDiscoverAndLinkSkills` (em `lib/pipeline/persist-curation/skill-mapper.ts:37`) é wrapper void que internamente chama `discoverAndLinkSkills` (em `lib/pipeline/ingest-job-and-discover-skills.ts:48`, retorna `Promise<DiscoverResult>`). Caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ EM `await safeDiscoverAndLinkSkills(supabase, normalized);` (M60 refator). Re-execução de E0a–E0d no SUB-PR 3 desta sprint confirma a fotografia atual e detecta drift caso o codebase tenha mudado.

- **D-PS-89 (escopo cross-cutting expandido — Opção 2 cravada):** cravado pelo product owner em 2026-05-14 após análise do ecossistema completo de painéis admin (8 abas, 50+ painéis, padrões `dashboard_daily_summary` + `pg.Pool`/Redis Limiares). Esta sprint passa a entregar valor em 3 frentes paralelas consumindo a mesma infraestrutura de dados FORI/FOSI/FOR contadores: Frente A (infraestrutura, SUB-PRs 1-6, pré-requisito); Frente B (dashboard global materializado, SUB-PRs 9-10 — `aggregateDayData` expandido + refator UI dos painéis 2.7/2.8/2.9 + ajuste KPI 2.1); Frente C (aba Limiares, SUB-PRs 7-8 — endpoint impact-preview no padrão Limiares + extração de componentes + enriquecimento dos painéis Limiares 1 e 8). Justificativa: alavancar trabalho de infraestrutura desta sprint para enriquecer máximo de superfícies de visibilidade do pipeline, dado que admin vai calibrar `pipeline_config` em produção e quanto mais simetria role↔skill ele enxerga, mais seguro é o controle online da ferramenta. Janela ampliada: 12-14 dias úteis vs ~8.5 dias da Opção 1.

- **D-PS-90 (Frente C — queries Limiares novas):** §3.5 amplia `lib/admin/limiares/queries.ts` com 4 queries que alimentam o endpoint `impact-preview` e os painéis Limiares 1 e 8 enriquecidos: `confidenceDistributionFORI`, `confidenceDistributionFOSI`, `opusReviewCooldownDistribution(entityType)`, `skillPathDistribution`. Todas via `pg.Pool` direto seguindo padrão Limiares. Demais 14 chaves do IMPACT_SOURCES reusam queries pré-existentes da aba Limiares (painéis 2, 3, 4, 6, 10 — auditoria via §6.1 confirma reuso vs custom query).

- **D-PS-91 (enriquecimento dos painéis Limiares 1 e 8 como subproduto natural):** painel 1 (Hard Gate) ganha série complementar via FOSI `status='gate_rejected'` paralela à série pré-existente baseada em `events.skill_filtered_hard_gate`. Painel 8 (creation_confidence distribuição) ganha modo "Por path" via `skillPathDistribution` (4 paths slug_match/alias_match/llm_new/race_recovered) paralelo ao modo pré-existente "Por entidade" (skill/role). Toggle dentro do painel. Justificativa: trabalho de queries em §3.5 + dados FORI/FOSI disponíveis pós-Frente A já habilitam esses enriquecimentos sem custo adicional significativo. Painéis Limiares 2, 3, 4, 5, 6, 7, 9, 10 sem interseção significativa — mantidos sem mudança.

- **D-PS-92 (Frente B — campos novos paralelos no JSONB de `dashboard_daily_summary`):** §3.6 expande `aggregateDayData` via sub-função `aggregatePipelineOrchestratorMetrics` que adiciona campos novos à família `operational`: `o7_skill` (drift de habilidades), `o8_skill` (path distribution skill), `o9_skill` (saúde skill), `items_processed` (volume role+skill). **Campos pré-existentes role-only `operational.o7`/`o8`/`o9` NÃO são tocados** — adição paralela, sem refator. UI dos painéis 2.7/2.8/2.9 consome ambos os conjuntos. Renomeação dos pré-existentes para `o7_role`/`o8_role`/`o9_role` (simetria nominal completa) fica como frente futura LK-PS-13 — sem urgência.

- **D-PS-93 (extração de `EditModal` + `ImpactPreview` para arquivos próprios — pré-trabalho SUB-PR 8 — DV-4 opção a cravada):** §5.1.0 cravado em 2026-05-14 após confirmação por DV-4 de que `EditModal` e `ImpactPreview` estão **inline em `app/admin/pipeline-config/page.tsx`** (não como componentes separados em `components/admin/pipeline-config/`). SUB-PR 8 começa extraindo ambos para arquivos próprios em `components/admin/pipeline-config/EditModal.tsx` e `components/admin/pipeline-config/ImpactPreview.tsx` antes de integrar o payload novo. Custo: ~1-2h. Justificativa: `ImpactPreview` nesta sprint vira componente complexo (histograma SVG, tabela 3-estados, seletor de janela, render condicional por sample_status); manter inline pioraria a manutenibilidade da `page.tsx`. Padrão arquitetônico do projeto é `components/admin/<feature>/`.

- **D-PS-94 (Frente B — KPI novo "Items processados pelo pipeline" no card 2.1):** §5.2.2 cravado. Card pré-existente "Habilidades Canônicas" ganha **tooltip** mostrando volume FORI + FOSI no período (não criar 6º card que quebraria o layout 5-colunas). Dados de `data.operational.items_processed.{role, skill, total}`. Justificativa: indicador mais visível ao admin do volume real de items que estão sendo processados (paridade direta com vagas curadas mas em granularidade de item). Layout do card preservado — só adiciona tooltip on-hover.

- **D-PS-95 (correção DV-2 — 26 operacionais total):** v1.3 referenciava "24 operacionais" desatualizado. **Correção cravada em 2026-05-14:** após cleanup v3.3 são 26 operacionais (24 originais + 2 `role.auto_assign_family.*` adicionadas pela cleanup §2.37). Aritmética final do IMPACT_SOURCES preservada: 20 chaves cobertas = 26 operacionais − 4 confidence − 2 `auto_assign_family` (out por D-PS-41 cleanup v3.3). + 2 sistema = 28 total no banco.

- **D-PS-96 (correção DV-3 — coluna real em `function_orchestrator_runs`):** v1.3 LK-PS-07 referenciava coluna inexistente `job_run_id` em `function_orchestrator_runs`. **Correção cravada em 2026-05-14:** a coluna real é `session_id` (já presente no schema). LK-PS-07 atualizado para refletir o nome correto. Vinculação ao `job_runs` é feita via JOIN por `session_id` quando aplicável.

- **D-PS-97 (correção DV-5 — caller já está em `await`):** v1.3 referenciava conversão do caller de fire-and-forget para await com base em D-PS-64 da cleanup v3.3. **Correção cravada em 2026-05-14:** ground truth de E0d revelou que o caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ EM `await safeDiscoverAndLinkSkills(supabase, normalized);` — refator anterior (sprint M60, comentário inline `M60 — await both; emit events on failure`) converteu o caller mas a D-PS-64 não foi atualizada. Refator desta sprint sobre o caller é exclusivamente: (a) expandir tipo do wrapper de `Promise<void>` para `Promise<DiscoverResultDetailed>`; (b) usar o valor retornado para empurrar para `insertFOSkillItemsBatch`. Não converter fire-and-forget para await — caller já está em await.

### Janela / threshold / formato — decisões cravadas

Vide D-PS-76, D-PS-77, D-PS-78 acima.

### Decisões herdadas da cleanup v3.3 que esta sprint resolve, reafirma, utiliza ou REVOGA

- **D-PS-33 (cleanup v3.3) — RESOLVIDA via expansão aditiva do endpoint pré-existente** por §4.1 + §5.1. Esta sprint adiciona blocos novos ao payload do `POST /[key]/impact-preview` pré-existente — não substitui o endpoint, não substitui a caixa qualitativa do modal §5.1.6.
- **D-PS-40 (cleanup v3.3) — RESOLVIDA** por §§2–§3. Função de descoberta passa a retornar `DiscoverResultDetailed`; tabela `function_orchestrator_skill_items` criada; 5 contadores `skills_*` em `function_orchestrator_runs`.
- **D-PS-43 (cleanup v3.3) — REAFIRMADA.** Pipeline CV continua fora do orchestrator.
- **D-PS-49 (cleanup v3.3) — REAFIRMADA.** Assimetria de fontes de `confidence_median` mantida intencional. Atualização do identificador da tabela em `fn_recompute_jcr_confidence_median` (§2.6) é mecânica de rename, não toque do circuito.
- **D-PS-50 (cleanup v3.3) — REAFIRMADA + AMPLIADA por D-PS-84.** `canonical_created` e `canonical_promoted` permanecem role-only; D-PS-84 amplia para reconhecer os 8 contadores outcome-oriented pré-existentes.
- **D-PS-51 (cleanup v3.3) — REAFIRMADA.** `fallback_ratio` continua role-only por construção. Métrica equivalente skill é calculável on-the-fly via `skills_failed / NULLIF(skills_extracted, 0)`.
- **D-PS-64 (cleanup v3.3) — REAFIRMADA.** Função de descoberta é único path do caminho curated; `mapSkillsToCanonical` permanece em quarantined sem instrumentação.
- **D-PS-65 (cleanup v3.3) — UTILIZADA.** `skill-type-guard.ts` é precedente do canal de propagação de `needs_review`. Esta sprint adiciona coluna `needs_review` em FOSI (§2.1) e propaga o sinal end-to-end via tipo `RawSkill` → `SkillProcessingDetail` → INSERT em `function_orchestrator_skill_items`.
- **D-PS-66 (cleanup v3.3) — UTILIZADA.** Race recovery via `resolve_active_canonical_by_slug` (CTE recursiva). FOSI registra esse path com `skill_pipeline_stage='race_recovered'` e `status='reused'`.
- **D-PS-67 (cleanup v3.3) — UTILIZADA + REAFIRMADA em D-PS-87.** Terminologia `projected_event_cost_usd` (evento único, NÃO /mês) preservada no payload expandido.
- **D-PS-68 (cleanup v3.3) — REVOGADA por D-PS-86 desta sprint.** Cache in-process Map TTL 30s + eviction 200 entradas é REMOVIDO. Substituído pelo padrão Redis da aba Limiares (TTL 300s). Justificativa: coerência arquitetônica com o ecossistema de painéis Limiares + compartilhamento entre instâncias serverless + eliminação de bug latente "cache stale após restart silencioso".
- **D-PS-69 (cleanup v3.3) — UTILIZADA.** Aliases resolvidos com `needsReview=true` propagam para FOSI via D-PS-65 (utilizada).

### Simetrias planejadas (S-ORCH)

- **S-ORCH-1:** `function_orchestrator_skill_items` espelha `function_orchestrator_role_items` estruturalmente, com enum próprio de stage (D-PS-79) e coluna nova `needs_review` para propagar sinal upstream (D-PS-65 cleanup v3.3). §2.1.
- **S-ORCH-2:** `function_orchestrator_runs` ganha 5 colunas `skills_*` stage-oriented. §2.2. Assimetria com 8 colunas role-only outcome-oriented documentada em D-PS-84.
- **S-ORCH-3:** função de descoberta (Cenário B confirmado — `safeDiscoverAndLinkSkills` wrapper de `discoverAndLinkSkills`) retorna `DiscoverResultDetailed`. Caller já em await; refator é só captura do retorno (D-PS-97). §3.1.
- **S-ORCH-4:** `insertFOSkillItem` em call sites identificados. Path quarantined explicitamente fora (D-PS-83). §3.2.
- **S-ORCH-5:** 3 finalizers atualizados via RPC SQL (correctness vs LK-CBO-31). §3.3.
- **S-ORCH-6:** rename `_items` → `_role_items` fecha simetria nominal. §2.6.
- **S-ORCH-7:** NÃO se aplica. D-PS-74 explica que simetria funcional de `confidence_median` em JCS já existe via `fn_jps_recompute_jcs`. Simetria estrutural de fonte é deliberadamente NÃO buscada (D-PS-75).
- **S-ORCH-8:** endpoint `POST /[key]/impact-preview` no padrão Limiares cobre 22 chaves com payload (correção DV-2): 20 via IMPACT_SOURCES + 2 via estimators apenas (`auto_assign_family.*` role) + 4 via `unsupported_in_v1` (confidence). §4.1 + D-PS-71. Cache in-process REVOGADO (D-PS-86); padrão Redis Limiares TTL 300s.
- **S-ORCH-9:** UI consolidada quantitativa ampliada dentro do componente `ImpactPreview` extraído de `page.tsx` para `components/admin/pipeline-config/` (D-PS-93). Adição de tabela + histograma + seletor de janela ao lado da caixa qualitativa pré-existente. §5.1 + D-PS-72.
- **S-ORCH-10 (Frente B — NOVO desta sprint v1.4):** dashboard global ganha simetria role↔skill nos painéis 2.7 (O7 drift), 2.8 (O8 caminho de resolução), 2.9 (O9 saúde do pipeline). Materialização via expansão de `aggregateDayData` em §3.6; consumo na UI via §5.2. Pré-existente role-only mantido (D-PS-92).
- **S-ORCH-11 (Frente B — NOVO):** card 2.1 ganha tooltip "Items processados pelo pipeline" agregando FORI + FOSI no período (D-PS-94). §5.2.2.
- **S-ORCH-12 (Frente C — NOVO):** painéis Limiares 1 (Hard Gate) e 8 (creation_confidence distribuição) enriquecidos com séries via FOSI/FORI como subproduto natural das queries novas de §3.5 (D-PS-91). §3.5.7 e §3.5.8.

### Links (LK-PS)

- **LK-PS-04:** §3.1 reutiliza helpers de `lib/pipeline/canonical-skills.ts` e RPCs `lookup_canonical_skill_by_normalized_alias` + `resolve_active_canonical_by_slug` (pré-existentes da cleanup v3.3).
- **LK-PS-05:** §4.1 endpoint no padrão Limiares depende de schema pós-cleanup v3.3 — colunas `criticality_level`, `description`, `updated_at`, `updated_by` em `pipeline_config` + arquivos `pipeline-impact-estimators.ts`. Resolução temporal: SUB-PR 7 só inicia após cleanup v3.3 mergeada. **DV-1 cravado em 2026-05-14:** o arquivo `lib/admin/pipeline-impact-preview-cache.ts` referenciado por LK-PS-10 da v1.3 NÃO EXISTE — cache estava inline em `route.ts:18-46`. Esta sprint REMOVE esse cache inline (D-PS-86) e substitui pelo padrão Redis Limiares.
- **LK-PS-06:** §2.6 atualização mecânica do identificador em `fn_recompute_jcr_confidence_median` confirmada via ground truth (corpo da função referencia `function_orchestrator_items` no `FROM`). Lógica, parâmetros, fonte e semântica preservados — apenas o nome da tabela muda.
- **LK-PS-07:** agregação cross-bulk (múltiplos runs do mesmo evento operacional do PO, ex: curadoria de 10.150 vagas em 10+ lotes) é frente futura. Será habilitada quando `bulk_curation_progress` for criada no Sub-PR3 do benchmark v14 adiado. Esta sprint NÃO endereça esse vínculo — `function_orchestrator_runs.session_id` (coluna real existente — **correção DV-3 cravada em 2026-05-14**: v1.3 referenciava `job_run_id` que NÃO existe) continua sendo o vínculo entre run e contexto. Quando `bulk_curation_progress` chegar, frente futura pode adicionar `bulk_run_id NULL` em FOR via ALTER TABLE aditivo.
- **LK-PS-08:** IMPACT_SOURCES importa keys de `PIPELINE_CONFIG_TOOLTIPS` (§3.14 da cleanup v3.3) para evitar duplicação de mapa estático. Gate de CI: teste TypeScript valida que `Object.keys(IMPACT_SOURCES)` é exatamente igual a `{Object.keys(PIPELINE_CONFIG_TOOLTIPS) \ {chaves de confidence + chaves de sistema + chaves de auto_assign_family}}`. Build quebra se divergir.
- **LK-PS-09:** §4.1 endpoint no padrão Limiares importa e USA os 8 estimators pré-existentes de `lib/admin/pipeline-impact-estimators.ts` (§3.16 da cleanup v3.3 + D-PS-67 cleanup v3.3). Esta sprint NÃO modifica os estimators, apenas envolve em `Promise.allSettled` paralelo com IMPACT_SOURCES.
- **LK-PS-10:** §4.1 endpoint refatorado USA helpers compartilhados da aba Limiares de `lib/admin/limiares/_shared/` (redis-cache wrapper, INTERVAL_SQL allowlist, pool config). Cache in-process Map TTL 30s da cleanup v3.3 REMOVIDO (D-PS-86). Sem fragmentação de mecanismo de cache no projeto.
- **LK-PS-11:** §2.1 coluna `needs_review` em FOSI propaga o sinal `needsReview` exportado de `lib/pipeline/skill-type-guard.ts` (§3.15 da cleanup v3.3 — D-PS-65 cleanup v3.3). Tipo `RawSkill` consumido pela função de descoberta DEVE incluir `needsReview?: boolean` — confirmar via §6.1.
- **LK-PS-12 (Frente B — NOVO desta sprint v1.4):** §3.6 expande `aggregateDayData` em `lib/admin/dashboard-day-aggregator.ts`. Auditoria E0e (§6.1.0) mapeia famílias pré-existentes no JSONB antes da expansão. Campos novos materializados (`o7_skill`, `o8_skill`, `o9_skill`, `items_processed`) entram dentro da família `operational` por simetria com pré-existentes role-only (D-PS-92).
- **LK-PS-13 (Frente B — frente futura):** renomeação dos campos pré-existentes `data.operational.o7`/`o8`/`o9` para `o7_role`/`o8_role`/`o9_role` (simetria nominal completa com `o7_skill`/`o8_skill`/`o9_skill` desta sprint) fica como frente futura caso necessidade operacional justifique. Quebra retrocompat de consumidores — não há urgência.

---

## §10 Resumo executivo das mudanças vs v1.3

Esta seção é fotografia das mudanças material vs v1.3, não narrativa histórica. Existe para facilitar revisão por terceiros.

### Decisão arquitetônica central — Opção 2 cravada

Em 2026-05-14 o product owner cravou Opção 2 (cross-cutting completa) após análise do ecossistema de painéis admin: a sprint passa de 8 SUB-PRs / ~8.5 dias (v1.3, escopo restrito a Frente A + endpoint impact-preview) para **10 SUB-PRs / 12-14 dias** organizados em 3 frentes paralelas que consomem a MESMA infraestrutura de dados FORI/FOSI/FOR contadores. D-PS-89.

| Frente | SUB-PRs | Conteúdo |
|---|---|---|
| A — Infraestrutura de dados | 1, 2, 3, 4, 5, 6 | FOSI tabela, FOR contadores skill, função de descoberta detalhada, rename `_role_items`. Sem mudança vs v1.3 exceto correções DV-3 e DV-5. |
| B — Dashboard global materializado (NOVO) | 9, 10 | Expansão de `aggregateDayData` + campos novos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` no JSONB + refator UI dos painéis 2.7/2.8/2.9 + KPI 2.1 com tooltip de items processados. |
| C — Aba Limiares + impact-preview (REFATORADO) | 7, 8 | Endpoint impact-preview MIGRADO para padrão Limiares (`pg.Pool` + Redis TTL 300s) + queries novas em `lib/admin/limiares/queries.ts` + enriquecimento dos painéis Limiares 1 e 8 + extração de `EditModal` + `ImpactPreview` para arquivos próprios. |

### Correções identificadas por Claude Code via ground truth (DV-1 a DV-5)

| ID | O que estava errado na v1.3 | Correção |
|---|---|---|
| **DV-1** | LK-PS-10 referenciava `lib/admin/pipeline-impact-preview-cache.ts` que NÃO existe — cache estava inline em `route.ts:18-46`. | LK-PS-10 atualizado. Cache inline será REMOVIDO no SUB-PR 7 (substituído por Redis Limiares — D-PS-86). |
| **DV-2** | "24 operacionais" referenciado em múltiplos pontos da v1.3 — desatualizado. | Atualizado para "26 operacionais" (24 originais + 2 `role.auto_assign_family.*` adicionadas pela cleanup v3.3 §2.37). Aritmética IMPACT_SOURCES preservada: 20 cobertas = 26 − 4 confidence − 2 auto_assign_family. D-PS-95. |
| **DV-3** | LK-PS-07 referenciava `function_orchestrator_runs.job_run_id` — coluna NÃO existe. | LK-PS-07 corrigido para `session_id` (coluna real). D-PS-96. |
| **DV-4** | §5.1 referenciava `EditModal.tsx` como arquivo separado já existente — ambos `EditModal` e `ImpactPreview` estão INLINE em `app/admin/pipeline-config/page.tsx`. | §5.1.0 adicionado: extração de ambos para `components/admin/pipeline-config/` é pré-trabalho obrigatório do SUB-PR 8 (~1-2h). D-PS-93. |
| **DV-5** | §3.1.2 referenciava conversão do caller de fire-and-forget para await com base em D-PS-64 cleanup v3.3. | Ground truth via E0d revelou que caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ EM `await safeDiscoverAndLinkSkills(supabase, normalized);` (sprint M60 anterior converteu). Refator desta sprint sobre caller é exclusivamente: expandir tipo do wrapper + usar retorno. D-PS-97. |

### Resolução de cenário da função de descoberta (E0d ground truth)

Cenário B confirmado em produção: `safeDiscoverAndLinkSkills` em `lib/pipeline/persist-curation/skill-mapper.ts:37` é wrapper externo de `discoverAndLinkSkills` em `lib/pipeline/ingest-job-and-discover-skills.ts:48` (retorna `Promise<DiscoverResult>`). Refator da sprint:

1. Função interna `discoverAndLinkSkills` passa a retornar `DiscoverResultDetailed` (estrutura expandida item-por-item) em vez de `DiscoverResult` (aggregate only).
2. Wrapper `safeDiscoverAndLinkSkills` deixa de ser `Promise<void>` e passa a `Promise<DiscoverResultDetailed>`.
3. Caller em `persist-fn.ts:410` (já em await) passa a usar o retorno para empurrar para `insertFOSkillItemsBatch`.

D-PS-88 atualizada para refletir a resolução; E0a–E0d permanecem como gate de detecção de drift no SUB-PR 3.

### Revogação importante

**D-PS-68 da cleanup v3.3 — REVOGADA por D-PS-86 desta sprint.** Cache in-process Map TTL 30s + eviction 200 entradas é REMOVIDO. Substituído pelo padrão Redis da aba Limiares (TTL 300s, prefixo `limiares:impact_preview:*`, fall-through silencioso). Justificativa cravada por D-PS-86. Esta é a única revogação real de decisão da cleanup v3.3 por esta sprint — todas as demais D-PS herdadas são reafirmadas ou utilizadas.

### Adições estruturais ao corpo (vs v1.3)

- **§0.3 escopo** ampliado para 3 frentes (era restrito a Frente A + endpoint)
- **§0.4 mapa de simetria** ampliado com itens das frentes B e C (S-ORCH-10, 11, 12)
- **§0.5 paralelismo** reorganizado para 10 SUB-PRs em 4 fases (era 8 SUB-PRs em 2 fases)
- **§1 estado pós-implantação** ampliado com itens 6, 7, 8 (queries Limiares, aggregator, painéis dashboard global) + atualização do item 4 (endpoint migrado para padrão Limiares) + item 9 (D-PS-68 cleanup v3.3 REVOGADA)
- **§3.1 refator função de descoberta** atualizado para Cenário B confirmado e caller já em await (DV-5)
- **§3.5 NOVO — queries Limiares** (5 subseções: 4 queries novas + helper de cálculo + 2 enriquecimentos painéis)
- **§3.6 NOVO — expansão de `aggregateDayData`** (sub-função `aggregatePipelineOrchestratorMetrics`, 2 RPCs novas SQL, backfill obrigatório, campos paralelos no JSONB)
- **§4.1 endpoint REESCRITO** para padrão Limiares: `pg.Pool` direto, Redis TTL 300s, allowlist intervalos, `Promise.allSettled`, payload estruturado por estados (sufficient / insufficient / estimator_only / unsupported_in_v1 / sistema)
- **§5.1.0 NOVO — pré-trabalho de extração** de `EditModal` + `ImpactPreview` para arquivos próprios (DV-4)
- **§5.2 NOVO — refator UI** dos painéis 2.7 / 2.8 / 2.9 do dashboard global + ajuste KPI 2.1 com tooltip de items processados
- **§6.1.0 ampliado** com E0e (auditoria do aggregator) — pré-requisito do SUB-PR 9
- **§6.1.1 ampliado** com auditorias EditModal inline + padrão Limiares + aggregateDayData
- **§8 reorganizado** para 10 SUB-PRs em 4 fases com diagrama temporal D+0 a D+10
- **§9 D-PS** ampliado: D-PS-71 corrigida (DV-2); D-PS-85, 86, 87 reescritas (padrão Limiares + revogação D-PS-68); D-PS-88 atualizada (resolução E0d); D-PS-89 a D-PS-97 NOVAS (Opção 2, frentes B/C, correções DV)
- **§9 S-ORCH** ampliado: S-ORCH-10, 11, 12 NOVAS
- **§9 LK-PS** ampliado: LK-PS-05, 07, 10 corrigidas (correções DV-1, DV-3); LK-PS-12, 13 NOVAS
- **§9 herdadas** atualizada: D-PS-68 cleanup v3.3 marcada como REVOGADA

### Conteúdo da v1.3 preservado integralmente

- §2.1 schema completo de `function_orchestrator_skill_items` (+ coluna `needs_review`)
- §2.2 colunas `skills_*` em FOR
- §2.3 validador de simetria
- §2.4 RPC `count_skill_items_by_status`
- §2.6 rename + atualização mecânica do corpo de funções
- §3.1 refator da função de descoberta (estrutura cenários A/B preservada; cenário B agora confirmado)
- §3.2 helper `insertFOSkillItem` + propagação de `needs_review`
- §3.3 finalizers via RPC
- §3.4 call sites pós-rename
- §6.2 auditoria de funções com `function_orchestrator_items` no corpo
- §7 evidence E1-E10 + smokes S1-S16 (sem mudança nesta sprint — cobertura adequada para Frente A; cobertura para Frentes B e C ficará por testes funcionais ad-hoc do SUB-PR específico)

---

**Fim da SPEC-sprint-orchestrator-symmetry-v1.4.**
