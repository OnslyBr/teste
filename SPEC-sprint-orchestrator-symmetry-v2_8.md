# SPEC — Sprint Orchestrator Symmetry v2.8

**Projeto:** CalibraCV
**Versão:** orchestrator-symmetry-v2.8
**MER de referência:** v37
**Ambiente alvo:** pré-produção, base operacional limpa (sem usuários ativos, sem dados históricos relevantes)
**Executor:** Antigravity (com acesso direto ao Supabase para migrations)
**Dependência:** sprint cleanup v3.4 mergeada em main + dev

---

## §0 Diretrizes da sprint

### §0.1 Convenção nominal

- Spec, narrativa e UI em PT-BR puro, sem estrangeirismos.
- Identificadores técnicos (rotas, arquivos, funções, parâmetros) em inglês.
- "Telemetria" e "instrumentação" aceitos como termos técnicos consagrados em PT-BR.
- A entidade `function_orchestrator_items` é renomeada dentro desta sprint para `function_orchestrator_role_items` (§2.6). Daqui em diante o nome novo é canônico.

### §0.2 Diretrizes de execução

1. **TodoWrite item-por-item antes de edição.** Para cada §X.Y, criar item na todo list e marcar concluído somente após validação contra o critério da §7.2. Validação de conformidade é DURANTE a execução, não depois.
2. **Validação ground truth obrigatória.** Antes de submeter PR, rodar os blocos de evidência da §7.1 contra o banco e o código real. Revisão estática multi-AI não substitui.
3. **Renomeação de parâmetro de função SQL exige DROP antes de CREATE OR REPLACE.** Renomear apenas o corpo da função preservando assinatura permite `CREATE OR REPLACE`; mudança de assinatura exige `DROP FUNCTION` + `CREATE`.
4. **Spec é fotografia.** Sem narrativa histórica, mapeamento de fixes acumulados, ou referência a versões anteriores no corpo.
5. **PT-BR puro em narrativa e UI.** Nomes técnicos em inglês.
6. **Sem novos canais externos ou controles operacionais.** Esta sprint não introduz SMS, e-mail externo, push, webhook externo, budget guard novo, kill-switch novo, rate limiter novo, monitor externo novo ou alerta novo — qualquer adição desta natureza requer autorização expressa do product owner.
7. **Simetria skill↔role.** Toda decisão considera paridade. Assimetrias justificadas em D-PS específica. A janela de simetrização é durante a sprint que toca o objeto — não há ciclo posterior dedicado.
8. **Triagem de feedback externo sem implementar.** Quando reviewer externo sinaliza achado no meio da sprint, ANALISAR SEM IMPLEMENTAR. Reportar como acatado/rejeitado/duplicata. Perguntar autorização antes de qualquer edit.
9. **Cross-check de código TS obrigatório antes de fechar spec.** Toda assinatura de tipo importado (`NormalizedCurationResult`, `LLMSkillItem`, `PersistOptions`, `RunCounters`, `BatchContext`, retorno de `requireAdmin`, etc), pattern de framework (Next.js 15 `params: Promise<...>` + `await params`, App Router conventions, SWR fetcher contract), uso de client Supabase (`adminCheck.supabase` sessão vs `adminCheck.admin` SERVICE_ROLE em tabelas com RLS sem policy permissiva) e helper externo referenciado sem JSDoc inline (`combineDistribution`, `aggregateLimiaresHistorical`, `swrPOSTFetcher`) precisa ser validado contra o código real do repositório antes de cravar na spec. Revisão de DB ground truth (§6.1 E0a-E0k) não cobre esses pontos — eles vivem no TS, não no banco.

10. **Handshake 4-way wire ↔ aggregator ↔ helper ↔ UI antes de fechar.** Para cada campo do payload emitido em wire spec (§4.1.5), validar que: (a) aggregator emite com o shape exato (mesmo nome, mesmo nesting, mesmo tipo); (b) helpers retornam o shape esperado pelo aggregator; (c) UI consumer lê o caminho exato com o tipo correto. Mismatch em qualquer ponta é bug — não é "comment para validar em E0", é falha estrutural de spec. Comments tipo "validar nome final em E0d antes do PR" não substituem o handshake durante a escrita. Specs que tocam payloads multi-camada devem ser escritas top-down TDD-style: wire spec primeiro (§4.1.5 — types literais), depois UI consumer (§5.1 — branches por campo), depois helpers (§4.4), e handler por último (§4.1.10 — apenas orquestra os 3 anteriores).

11. **Cross-check de imports via Glob durante a escrita.** Para cada `import X from 'path'` ou `import type X from 'path'` na spec, abrir o arquivo `path` no repositório real antes de cravar e confirmar: (a) o arquivo existe; (b) `X` é exportado nominal/default daquele arquivo; (c) o shape exportado bate com o uso na spec. Para hooks/utilities que a spec descreve como "pré-existentes da cleanup vX": validar via Glob/grep que existem; se não existirem, cravar deliverable explícito na §X.Y para criar o arquivo OU substituir por padrão inline. Comments do tipo "validar caminho em E0" são insuficientes — diretrizes #9, #10 e #11 são complementares e devem ser executadas DURANTE a escrita, não como auditoria pós-fato.

12. **Validação de ENUM SQL contra CHECK constraint pré-existente no DB obrigatória.** Antes de cravar `CREATE TYPE <enum>` em qualquer migration, validação obrigatória contra `pg_constraint` real do DB para tabelas envolvidas. Se a coluna já tem `CHECK (status IN (...))` ou constraint similar (legado de cleanup anterior ou de schema histórico), valores do ENUM devem ser **idênticos** ao conjunto da CHECK. Cross-check executado via query `SELECT conname, pg_get_constraintdef(oid) FROM pg_constraint WHERE conname LIKE '%status_check%' OR conname LIKE '%pipeline_stage_check%'`. Paritary, validar contra type TS real correspondente (ex: `lib/pipeline/types.ts` exports). Vetor de validação cego em todas as auditorias até v2.6 — bug L-1 (ENUM `role_item_status` com valores inventados `curated`/`curated_fallback` + omitindo valor real `fallback`) viveu 14 iterações por falta desta diretriz. Esta diretriz é especificamente sobre **valores literais** do ENUM (não sobre nome do tipo nem sobre tabela); §6.1 deve incluir um pré-step `E0k` para listar todas as CHECK constraints relevantes do DB antes de qualquer `CREATE TYPE` ser cravado em §2.x/§2.1/etc.

### §0.3 Escopo

Esta sprint entrega valor em **três frentes paralelas** consumindo a mesma infraestrutura de dados sobre o pipeline de curadoria de vagas (`function_orchestrator_role_items`, `function_orchestrator_skill_items`, `function_orchestrator_runs`):

**Frente A — Infraestrutura de dados (pré-requisito das demais)**

- Tabela paralela `function_orchestrator_skill_items` (FOSI) espelhando `function_orchestrator_role_items` (FORI) semanticamente, com enums próprios (`skill_pipeline_stage`, `skill_item_status`) refletindo os paths reais do pipeline de skill, coluna `canonical_status` paritária a FORI, `CHECK (skill_confidence >= 0 AND skill_confidence <= 1)` e `raw_item_index int` + `UNIQUE(run_id, job_posting_id, raw_item_index)` para idempotência de retry. FKs `run_id`/`job_posting_id` com `ON DELETE CASCADE` e `canonical_skill_id` com `ON DELETE SET NULL` (§2.1)
- Corretivos retroativos em FORI: `ADD CONSTRAINT chk_fori_confidence_range CHECK (confidence IS NULL OR (confidence >= 0 AND confidence <= 1))` + `ADD COLUMN raw_item_index int` + `CREATE UNIQUE INDEX uq_fori_run_posting_index` paritários; conversão `pipeline_stage` e `status` de `text + CHECK` para `CREATE TYPE role_pipeline_stage AS ENUM` + `CREATE TYPE role_item_status AS ENUM` (paridade total de tipos com FOSI); FKs retroativas `run_id`/`job_posting_id` com `ON DELETE CASCADE` e `canonical_role_id` com `ON DELETE SET NULL` (§2.x corretivos)
- 5 contadores agregados de skill em `function_orchestrator_runs` — stage-oriented (§2.2)
- Backfill obrigatório de `entity_type` em events legacy (`canonical_promoted_dynamic` 31 rows + `canonical_creation_blocked_low_confidence` 5 rows, todas sem `metadata->>'entity_type'` segundo ground truth) via FK lookup em `resource_id` cruzando `job_canonical_roles` e `job_canonical_skills`. Executa como **SUB-PR 0** (pré-step bloqueador de SUB-PR 2). Rows não-resolvidas → `entity_type='unknown'` + log de auditoria + exclusão da rename §2.y (§2.w novo)
- Migração de evolução de `event_names`: separação `role_*`/`skill_*` por construção via update determinístico baseado em `metadata->>'entity_type'` (populado pelo SUB-PR 0). Aplica a `canonical_promoted_dynamic` → `role_promoted_dynamic`/`skill_promoted_dynamic` e `canonical_creation_blocked_low_confidence` → `role_creation_blocked_low_confidence`/`skill_creation_blocked_low_confidence`. **Whitelist completa em `events.event_name`**: auditoria via §6.1 lista todos os `event_names` ativos no DB (77); CHECK constraint **nova** é criada via `ADD CONSTRAINT events_event_name_check` (NÃO `DROP/ADD` — constraint não existia previamente segundo ground truth) contendo lista exaustiva. Update de call sites TS que emitem os eventos antigos para emitir os nomes novos (§2.y)
- Triggers paritários `trg_jcr_emit_canonical_role_created` em `job_canonical_roles` e `trg_jcs_emit_canonical_skill_created` em `job_canonical_skills`, ambos `AFTER INSERT WHEN (pg_trigger_depth() = 0)` (guard contra loop de re-emissão) emitindo `canonical_role_created`/`canonical_skill_created` em `events` com `metadata = {canonical_id, label, entity_type}` (sem `created_via` — coluna não existe nas tabelas e cravar coluna vazia perpétua seria overengineering; rastreabilidade de origem fica como dívida explícita LK-PS-NEW caso justifique no futuro) (§2.z)
- Refator da função de descoberta de skills (`safeDiscoverAndLinkSkills` wrapper sobre `discoverAndLinkSkills` interna) expandindo retorno void → `DiscoverResultDetailed` item-por-item (§3.1)
- `insertFOSkillItem` em todos os call sites cobrindo fluxos A, B, C, com bulk path `.insert(array)` dada cardinalidade real skill 10-30 por job vs role 1-2 (D-PS desta sprint); leitura de `.error` + log estruturado `[batch] insertFOSkillItem failed:` em FOSI e retroativamente em FORI (paridade COM bug fix) (§3.2)
- Atualização dos finalizers de run para acumular contadores de skill via RPC SQL compartilhada (§3.3)
- Rename de `function_orchestrator_items` para `function_orchestrator_role_items`, incluindo atualização do corpo de funções dependentes — em especial `fn_recompute_jcr_confidence_median` (apenas diff mínimo de nome de tabela no FROM; `SET search_path` já existe; `CREATE OR REPLACE` preserva GRANTs) (§2.6)
- Helper `handleSkillItemError` paritário ao `handleItemError` pré-existente, em `lib/pipeline/batch-processor/error-handlers.ts`, com prefixo de log `[batch]` (convenção por módulo, não por entidade). **Cobertura paritária de callers**: auditoria E0h do §6.1 identifica todos os call sites de `handleItemError` no pipeline de role; SUB-PR 3 instala invocações paritárias de `handleSkillItemError` em call sites equivalentes do pipeline de skill (não apenas no catch do `safeDiscoverAndLinkSkills`) (§3.2)
- Cron `monthly-cleanup` ganha cleanup paritário de FOSI > 2 anos (B1 paritário; runs `function_orchestrator_runs` mantidos no cleanup pré-existente)
- Cleanup novo de orphan runs (universal, não Flow C-específico): runs `status='running' AND started_at < NOW() - 24h` → `status='error'`/`error_message='orphan_run_timeout'`; deletar `created_at < 90d`

**Frente B — Unificação no dashboard_daily_summary (storage materializado único)**

- Refator do shape JSONB de `dashboard_daily_summary.data`: aggregator passa a emitir família aninhada lowercase `data.operational.{o7, o8, o9}` (role pré-existentes — substituem `data.O7`/`data.O8`/`data.O9` top-level uppercase que existia antes desta sprint) + paralelos novos `data.operational.{o7_skill, o8_skill, o9_skill, items_processed}`. Refator coordenado entre aggregator (§3.6.2), backfill 26 dias e UI dos painéis 2.7/2.8/2.9 (§5.2) no mesmo SUB-PR para evitar janela de inconsistência
- Expansão de `lib/admin/dashboard-day-aggregator/aggregator.ts` para coletar métricas FORI/FOSI/FOR contadores via sub-função `aggregatePipelineOrchestratorMetrics`, com leitura de `.error` do Supabase + log estruturado `[dashboard-aggregator] orchestrator.<sub>_query_failed` antes de fallback (corrigindo dívida pré-existente paritariamente)
- Família nova `data.limiares.{panel_1, panel_2, panel_3, panel_4, panel_5, panel_6, panel_7, panel_8, panel_9, panel_10}` com snapshot diário dos 10 painéis da aba Limiares (inclusive painéis 3/5/9 que são listas detalhadas — listas serializadas em JSONB quando aplicável)
- Backfill obrigatório via `/api/admin/backfill-dashboard-summary` regenerando 26 dias rolling com a estrutura nova: `data.operational` reorganizado + `data.limiares` novo. Dias pré-existentes que tinham `data.O7/O8/O9` top-level são sobrescritos com `data.operational.o7/o8/o9` aninhado
- Refator UI dos painéis 2.7 (O7 — drift), 2.8 (O8 — Caminho de resolução), 2.9 (O9 — Saúde do pipeline) para exibir séries paralelas role+skill, lendo dos paths novos `data.operational.*` (§5.2)
- Ajuste do card 2.1 KPI com indicador "Items processados pelo pipeline" (role + skill no tooltip)
- 12 RPCs SQL novas (`limiares_panel_N_snapshot` para N de 1 a 10, `count_role_items_by_stage_in_window`, `count_skill_items_by_stage_in_window`) chamadas via Supabase client — encapsulam queries complexas que PostgREST não expressa diretamente (WIDTH_BUCKET, CTE, UNION ALL, PERCENTILE_CONT)
- Refator UI dos 10 painéis Limiares para consumir as 2 famílias novas no JSONB para janelas históricas (7d/30d) e RPCs SQL para janela 24h

**Frente C — impact-preview como consumidor do storage unificado**

- Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` migra para consumir `dashboard_daily_summary` em janelas históricas (7d/30d/90d) e invocar RPCs SQL via Supabase client para sample do dia corrente. Sem `pg.Pool`, sem Redis custom. Preserva campos pré-existentes do payload (`affected_count`, `projected_event_cost_usd`, `cost_window`, `cost_is_fallback`, `panels`); adiciona campos novos `window_days`, `source`, `sample_size`, `sample_status`, `current_impact`, `proposed_impact`, `histogram`
- 28 chaves de `pipeline_config` cobertas via estimator base da cleanup v3.4 §3.16 (F23 entregou cobertura 28/28); histograma `WIDTH_BUCKET` adicional para subset com `sample_size` suficiente
- Extração de `EditModal` + `ImpactPreview` da `app/admin/pipeline-config/page.tsx` para arquivos próprios em `components/admin/pipeline-config/`

**Não inclui (out-of-scope cravado):**

- Trigger novo para `confidence_median` em `job_canonical_skills` — `fn_recompute_jcs_confidence_median` (renomeada de `fn_jps_recompute_jcs` pela cleanup v3.4 F21) já popula essa coluna a partir de fonte robusta (`job_posting_skills` curated). Ver D-PS-74.
- Mudança de fonte de `confidence_median` em qualquer dos lados — assimetria de fontes role/skill é intencional. Ver D-PS-75.
- Pipeline CV (input 1) — D-PS-43 da cleanup v3.4.
- Cobertura do path quarantined (legacy `mapSkillsToCanonical`) — skills extraídas em vagas quarantenadas permanecem invisíveis ao orchestrator. Ver D-PS-83.
- Simetria estrutural ampla dos contadores outcome-oriented de FOR (8 colunas role-only pré-existentes) — esta sprint adiciona apenas contadores stage-oriented de skill. Ver D-PS-84.
- Adição de `bulk_run_id` para agregação multi-lote — dependência da tabela `bulk_curation_progress` do benchmark v14 adiado. Ver LK-PS-07.
- **Flow C visibility model** — sprint dedicada futura. Apenas o cleanup de orphan runs (universal) integra esta sprint. Migration em `job_postings` com `visibility` + `owner_profile_id`, atualização de `fn_recompute_jcr_confidence_median`/`fn_promote_role_on_threshold`/`fn_promote_skill_on_threshold` para filtrar `WHERE jp.visibility='public'`, filtros em queries de análise CV pública, análise condicional do dono para incluir vagas privadas, UI do modal de vaga 1:1 colada. Ver LK-PS-19.
- **Rate-limit em endpoints admin** — frente futura transversal a todos os ~88 painéis admin do dashboard global, não pontual em um endpoint. Avaliada com base em métricas reais pós-go-live com autorização expressa. Ver LK-PS-20.
- Renomeação dos campos pré-existentes role-only `o7`/`o8`/`o9` para `o7_role`/`o8_role`/`o9_role` — manteria simetria nominal completa mas quebra retrocompat de consumidores. Frente futura.

### §0.4 Mapa de simetria desta sprint

| § | Conteúdo | Frente | Simetria | Mecanismo |
|---|---|---|---|---|
| §2.1 | `function_orchestrator_skill_items` com CHECK 0..1 + `raw_item_index` + UNIQUE | A | parcial — espelha estrutura mas enum de stage reflete paths reais de skill | tabela paralela, schema espelhado, enum próprio (D-PS-79) |
| §2.x | corretivos retroativos em FORI: CHECK 0..1 + `raw_item_index` + UNIQUE | A | paridade total — fix retroativo no lado role baseado em ground truth Claude Code | `ALTER TABLE` + `CREATE UNIQUE INDEX` |
| §2.2 | 5 colunas `skills_*` em `function_orchestrator_runs` | A | stage-oriented; 8 colunas role pré-existentes são outcome-oriented (D-PS-84) | colunas paralelas, semântica explicitada via COMMENT |
| §2.6 | rename `function_orchestrator_items` → `_role_items` (a) + rewrite mecânico de `fn_recompute_jcr_confidence_median` (b) | A | simetria nominal | rename + diff mínimo no FROM da função (`SET search_path` já existe; `CREATE OR REPLACE` preserva GRANTs) |
| §3.1 | refator função de descoberta retornando `DiscoverResultDetailed` | A | par com tipo de retorno equivalente em path de role | instrumentação de todos paths internos; caller já em await |
| §3.2 | `insertFOSkillItem` (individual + bulk) em todos os call sites + `handleSkillItemError` | A | par com `insertFORoleItem` (renomeado em §3.4) + `handleItemError` pré-existente | cobertura A/B/C; quarantined path explicitamente fora; bulk via cardinalidade skill 10-30 vs role 1-2 (assimetria operacional documentada) |
| §3.2 | leitura de `.error` + log `[batch] insertFOSkillItem failed:` | A | paridade COM bug fix em FORI (retroativo) | adicionar `if (error) console.warn` em ambos os lados |
| §3.3 | acumuladores em 3 finalizers via RPC SQL compartilhada `count_skill_items_by_status` | A | par com acumulação de role | RPC obrigatória; UPDATE atômica com fields role+skill |
| §3.5 | RPCs SQL paritárias `limiares_panel_1..10_snapshot` + `count_*_items_by_stage_in_window` | B | par role+skill nas RPCs que retornam JSONB já estruturado | RPCs encapsulam `WIDTH_BUCKET`, CTE, UNION ALL; chamadas via Supabase client em endpoints |
| §3.6 | expansão de `aggregateDayData` materializando `data.operational.*_skill` e `data.limiares.panel_1..10` | B | par role+skill via campos paralelos no JSONB | sub-função `aggregatePipelineOrchestratorMetrics`; leitura de `.error` + log paritário em ambos os lados (corrige dívida pré-existente) |
| §4.1 | endpoint `pipeline-config/[key]/impact-preview` consumindo `dashboard_daily_summary` + RPCs | C | par role+skill via parâmetro `key` | Supabase client; sem `pg.Pool`; sem Redis custom |
| §4.x | refator endpoints `/api/admin/limiares/{online,historical}` via Supabase client + RPCs | B | par role+skill nas RPCs | `pg.Pool` eliminado; convergência arquitetônica completa com resto do dashboard global |
| §5.1 | integração no modal de edição com extração de `EditModal` + `ImpactPreview` | C | visualização adicional consolidada agnostic role/skill | extração para arquivos próprios + ampliação do componente para consumir payload novo |
| §5.2 | refator UI dos painéis 2.7/2.8/2.9 + 10 painéis Limiares | B | bifurcação em séries paralelas role+skill nos 3 painéis O7/O8/O9; 10 painéis Limiares consumem JSONB unificado | mesmo componente parametrizado por `entityType`; pré-existentes role-only mantidos para retrocompat |

**Assimetrias intencionais preservadas:**

- Fontes de `confidence_median` divergem entre role e skill — D-PS-49 da cleanup v3.4.
- 8 contadores outcome-oriented em FOR são role-only por convenção pré-existente — D-PS-50 da cleanup v3.4 + D-PS-84 desta sprint.
- `fallback_ratio` é role-only por construção — D-PS-51 da cleanup v3.4.
- Bulk path `insertFOSkillItem` é assimétrico (skill tem `.insert(array)`; role mantém individual) — assimetria operacional por cardinalidade real (skill 10-30 por job vs role 1-2). Ver D-PS desta sprint.
- Estimators `auto_assign_family.min_similarity` / `min_score` cobertos pelos estimators da cleanup v3.4 §3.16 mas estruturalmente role-only (D-PS-41 cleanup v3.4 — famílias só role).
- Pré-existentes `data.operational.o7`/`o8`/`o9` no JSONB permanecem role-only por retrocompat; campos novos `o7_skill`/`o8_skill`/`o9_skill` são paralelos.

### §0.5 Sequência de execução — 12 SUB-PRs

A sprint orchestrator-symmetry é sequencial após a cleanup v3.4 ter sido mergeada (main + dev). 12 SUB-PRs com dependências internas explícitas.

| SUB-PR | Frente | Conteúdo | Depende de |
|---|---|---|---|
| 0 | A | §2.w backfill `entity_type` em events legacy via FK lookup + auditoria de rows não-resolvidas | cleanup v3.4 mergeada |
| 1 | A | §2.1 (FOSI com CHECK + raw_item_index + UNIQUE + canonical_status + 2 ENUMs skill_*) + §2.2 (colunas skill em FOR) + §2.3 (validação simétrica) + auditoria scripts/funções/event_names/handleItemError callers (§6.1 E0a-E0k, §6.2) | cleanup v3.4 mergeada |
| 2 | A | §2.x corretivos retroativos em FORI (CHECK 0..1, raw_item_index + UNIQUE, ENUMs role_*, CASCADE) + §2.4 RPC `count_skill_items_by_status` + §2.6 rename FORI (5 funções reais) + §2.y migração event_names role_*/skill_* + §2.z triggers canonical_*_created + §3.4 call sites do rename + atualização emissores TS de events | SUB-PR 0 + SUB-PR 1 |
| 3 | A | §3.1 refator descoberta retornando `DiscoverResultDetailed` + §3.2 `insertFOSkillItem` individual e bulk (incluindo `canonical_status`) + paridade bug fix retroativo em FORI + `handleSkillItemError` em error-handlers.ts + **cobertura paritária de callers (E0h)** | SUB-PR 2 |
| 4 | A | §3.3 finalizers atualizados via RPC compartilhada (UPDATE atômica role+skill) | SUB-PR 3 |
| 5 | A | cron `monthly-cleanup` paritário FOSI + cleanup novo de orphan runs (universal) | SUB-PR 1 (pode rodar em paralelo com 2-4) |
| 6 | B | 12 RPCs SQL paritárias (`count_*_items_by_stage_in_window` + 10 `limiares_panel_N_snapshot` com nomes de events role_*/skill_*, schema correto pipeline_config_history, `detected_at` em merge_candidates, fallback gap_days 365/365) | SUB-PR 4 (FOSI populada e finalizers escrevendo) + SUB-PR 2 (event_names migrados) |
| 7 | B | §3.6 expansão `aggregateDayData` materializando `data.operational.{o7, o8, o9, o7_skill, o8_skill, o9_skill, items_processed}` (refator do shape de top-level uppercase para aninhado lowercase) + `data.limiares.panel_1..10` no JSONB + leitura de `.error` paritária em ambos os lados + backfill 26 dias rolling com shape novo | SUB-PR 6 |
| 8 | B | refator `/api/admin/limiares/historical` para consumir `dashboard_daily_summary` (Supabase client; sem pg.Pool; sem Redis) | SUB-PR 7 |
| 9 | B | refator `/api/admin/limiares/online` para Supabase client invocando 10 RPCs `limiares_panel_N_snapshot` com janela 24h (sem pg.Pool; sem Redis) | SUB-PR 6 |
| 10 | C | refator `impact-preview` para consumir `dashboard_daily_summary` + RPCs SQL para sample do dia corrente. Extração `EditModal`/`ImpactPreview` da page.tsx | SUB-PR 7 |
| 11 | B | refator UI dos painéis 2.1/2.7/2.8/2.9 (lendo dos paths novos `data.operational.*`) + 10 painéis Limiares para consumir JSONB unificado + **painel 9 refatorado para vocabulário `previous_value`/`changed_by`/`changed_at`** (alinhado ao schema). **Deploy coordenado com SUB-PR 7** para evitar janela de inconsistência | SUB-PR 7 (10 painéis Limiares + paths novos) + SUB-PR 9 (modo online 24h) + SUB-PR 10 (consumidor impact-preview) |

Janela total estimada: **14-17 dias úteis**. Detalhamento de estimativas em §8.

---

## §1 Estado pós-implantação

Ao final desta sprint:

1. Toda skill processada pelo pipeline de vagas (fluxos A, B, C) em caminho curated deixa rastro item-por-item em `function_orchestrator_skill_items`, com estágio do pipeline real (enum `skill_pipeline_stage`), confidence (com CHECK 0..1), `canonical_skill_id` resolvido, `canonical_status` paritário a FORI, status final (enum `skill_item_status`), sinal `needs_review` propagado do skill-type-guard upstream, e `raw_item_index` único por `(run_id, job_posting_id)` garantindo idempotência de retry. FKs `run_id` e `job_posting_id` com `ON DELETE CASCADE`; `canonical_skill_id` com `ON DELETE SET NULL`. Skills em caminho quarantined permanecem invisíveis (D-PS-83).
2. `function_orchestrator_role_items` ganha as mesmas garantias retroativamente: `CHECK (confidence IS NULL OR (confidence >= 0 AND confidence <= 1))` + `raw_item_index int` + `UNIQUE(run_id, job_posting_id, raw_item_index)`. Tipos `pipeline_stage` e `status` convertidos de `text + CHECK` para ENUMs nominais paritários (`role_pipeline_stage`, `role_item_status`). FKs `run_id` e `job_posting_id` ganham `ON DELETE CASCADE` retroativo; `canonical_role_id` ganha `ON DELETE SET NULL`. Caller TypeScript (`process-item.ts:199 + 206`) atualizado para passar `raw_item_index` explicitamente.
3. Contadores agregados de skill populam `function_orchestrator_runs` em 5 colunas stage-oriented (`skills_extracted`, `skills_reused`, `skills_pending_created`, `skills_gate_rejected`, `skills_failed`).
4. Tabela `function_orchestrator_items` foi renomeada para `function_orchestrator_role_items` em toda a base de código, schema e corpos de funções dependentes — `internal.reset_taxonomy_core`, `public.cleanup_batch_items`, `public.fn_recompute_jcr_confidence_median`, `public.merge_canonicals`, `public.release_quarantined_jobs_limited` — cada uma com rewrite mecânico via `CREATE OR REPLACE` preservando proconfig original específica (algumas têm `SET search_path TO 'public'` sem `pg_temp`, outras têm `public, pg_temp` — rewrite preserva exatamente o que cada uma tem).
5. Events `canonical_promoted_dynamic` e `canonical_creation_blocked_low_confidence` evoluíram para nomes discriminados por entidade: `role_promoted_dynamic` / `skill_promoted_dynamic` e `role_creation_blocked_low_confidence` / `skill_creation_blocked_low_confidence`. Update determinístico das rows existentes baseado em `metadata->>'entity_type'` (populado pelo SUB-PR 0 de backfill via FK lookup). Call sites TS que emitem esses eventos atualizados para emitir os nomes novos. `CHECK constraint` **nova** em `events.event_name` criada (não existia previamente segundo ground truth) com whitelist completa derivada da auditoria E0i do §6.1 — lista exaustiva de todos os `event_names` ativos no DB (77) incluindo os 4 nomes novos paritários desta sprint mais `canonical_role_created`/`canonical_skill_created`.
6. Triggers paritários `trg_jcr_emit_canonical_role_created` em `job_canonical_roles` e `trg_jcs_emit_canonical_skill_created` em `job_canonical_skills`, ambos `AFTER INSERT WHEN (pg_trigger_depth() = 0)` emitindo `canonical_role_created` / `canonical_skill_created` em `events` com `metadata = {canonical_id, label, entity_type}`. Coluna `created_via` NÃO é adicionada (base zera no deploy; coluna nasceria vazia perpétua sem caller que a populasse). Rastreabilidade de origem fica como dívida explícita LK-PS-NEW caso necessidade futura justifique. Aggregator passa a ler séries não-vazias para drift O7 role+skill.
7. `insertFOSkillItem` tem duas modalidades: individual (call sites com 1-2 skills por job) e bulk via `.insert(array)` (call sites com cardinalidade alta 10-30 skills por job). Helper `insertFOItem` lado role e `insertFOSkillItem` lado skill verificam `.error` retornado pelo Supabase e logam estruturadamente com prefixo `[batch]` quando falha (correção paritária da dívida pré-existente).
8. `handleSkillItemError` paritário ao `handleItemError` pré-existente em `lib/pipeline/batch-processor/error-handlers.ts`. Auditoria E0h (§6.1) lista todos os call sites de `handleItemError` no pipeline de role; SUB-PR 3 instala invocações paritárias de `handleSkillItemError` em call sites equivalentes do pipeline de skill (cobertura completa, não apenas no catch upstream do `safeDiscoverAndLinkSkills`). `insertEvent('persist_curation.skill_map_failed')` pré-existente mantido (auditoria global em `events`).
9. Cron `monthly-cleanup` ganha cleanup paritário de FOSI > 2 anos. Novo cleanup de orphan runs marca runs com `started_at < NOW() - 24h AND status='running'` como `status='error'`/`error_message='orphan_run_timeout'` e deleta runs `created_at < 90d` (retention).
10. `lib/admin/dashboard-day-aggregator/aggregator.ts` ganhou sub-função `aggregatePipelineOrchestratorMetrics` chamada de dentro de `aggregateDayData()`. JSONB de `dashboard_daily_summary.data` reorganizado: famílias `data.O7`/`data.O8`/`data.O9` (uppercase top-level pré-existente) migram para `data.operational.{o7, o8, o9}` (lowercase aninhado, paritário ao naming pattern das demais famílias `ai`/`resources`/`communications`/`auth`/`errors`/`accumulators`). Campos paralelos novos `data.operational.{o7_skill, o8_skill, o9_skill, items_processed}` adicionados. Família nova `data.limiares.{panel_1, ..., panel_10}` com snapshot diário dos 10 painéis Limiares. Backfill via `/api/admin/backfill-dashboard-summary` regenerou os 26 dias rolling com a estrutura nova. UI dos painéis 2.7/2.8/2.9 atualizada nos mesmos SUB-PRs do backfill para evitar janela de inconsistência (deploy coordenado).
11. Leitura de `.error` do Supabase + log estruturado `[dashboard-aggregator] orchestrator.<sub>_query_failed` paritário em ambos os lados (corrige dívida pré-existente de fetchers role-only que engoliam erros via `?? 0`).
12. 12 RPCs SQL novas chamadas via Supabase client encapsulam queries complexas que PostgREST não expressa: `count_role_items_by_stage_in_window`, `count_skill_items_by_stage_in_window`, `limiares_panel_1_snapshot`, ..., `limiares_panel_10_snapshot`. Cada RPC `limiares_panel_N_snapshot` recebe `p_interval text` (allowlist `'1 day'`, `'7 days'`, `'30 days'`) e retorna JSONB já estruturado para o painel N. RPCs são `LANGUAGE sql STABLE` com `SET search_path TO 'public', 'pg_temp'`.
13. Endpoints `/api/admin/limiares/historical` e `/api/admin/limiares/online` refatoradas: `pg.Pool` direto eliminado; Redis custom eliminado; endpoints consomem Supabase client. `historical` lê de `dashboard_daily_summary` cross-day para janelas 7d/30d. `online` invoca as 10 RPCs `limiares_panel_N_snapshot('1 day')` para janela 24h. Padrão HTTP `cache-control: max-age=60` paritário ao resto do dashboard global.
14. Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview` migra para Supabase client: lê de `dashboard_daily_summary` para janelas históricas (7d/30d/90d); invoca RPCs SQL para sample do dia corrente; sem `pg.Pool`, sem Redis custom. Preserva campos pré-existentes da cleanup v3.4 (`affected_count`, `projected_event_cost_usd`, `cost_window`, `cost_is_fallback`, `panels`). Adiciona campos novos `window_days`, `source`, `sample_size`, `sample_status`, `current_impact`, `proposed_impact`, `histogram`. 28 chaves de `pipeline_config` cobertas via `estimateImpact()` pré-existente da cleanup v3.4 §3.16; histograma `WIDTH_BUCKET` adicional para subset com `sample_size` suficiente.
15. `EditModal` e `ImpactPreview`, antes inline em `app/admin/pipeline-config/page.tsx`, foram extraídos para arquivos próprios em `components/admin/pipeline-config/EditModal.tsx` e `components/admin/pipeline-config/ImpactPreview.tsx`. Modal exibe dois blocos lado a lado na seção "Impacto estimado": à esquerda a caixa qualitativa pré-existente, à direita o componente `ImpactPreview` expandido (tabela de contagem + histograma de distribuição + seletor de janela 7d/30d/90d).
16. Painéis 2.7 (O7 — Funções Novas / drift), 2.8 (O8 — Caminho de resolução), 2.9 (O9 — Saúde do pipeline) do dashboard global passaram a exibir séries paralelas role+skill, lendo dos paths novos `data.operational.*`. Card 2.1 ganha indicador novo "Items processados pelo pipeline" agregando role + skill no header com breakdown no tooltip.
17. Worst-of-three / worst-of-N para classificação de saúde dos painéis O9 e seu equivalente skill é calculado no render layer da UI, não no aggregator. Aggregator materializa counters puros (`fallback_error`, `sem_canonical`, etc); UI aplica o algoritmo de traffic-light.
18. 10 painéis da aba Limiares passam a consumir `dashboard_daily_summary` para janelas históricas e RPCs SQL para janela 24h. Convergência arquitetônica: 100% dos painéis do dashboard global e aba Limiares passam a usar o mesmo storage materializado para histórico, com cache HTTP `max-age=60` paritário, e Supabase client + RPCs SQL para queries complexas no dia corrente. Zero `pg.Pool` direto em endpoints admin. Zero Redis custom em endpoints admin.
19. `confidence_median` em `job_canonical_skills` continua sendo populado por `fn_recompute_jcs_confidence_median` (renomeada de `fn_jps_recompute_jcs` pela cleanup v3.4 F21 — esta sprint apenas referencia o nome novo). `confidence_median` em `job_canonical_roles` continua sendo populado por `fn_recompute_jcr_confidence_median`, cujo corpo é atualizado mecanicamente apenas para referenciar `function_orchestrator_role_items` pós-rename — sem mudança de fonte, lógica, parâmetros ou semântica do cálculo.
20. A terminologia `projected_event_cost_usd` da cleanup v3.4 ("evento único", NÃO recorrência mensal) é preservada no payload do endpoint refatorado. UI mantém o rótulo canônico "Custo Opus projetado (evento único)" na caixa qualitativa quando o estimator atende a chave editada.

---

## §2 Migrations SQL

### §2.1 — `function_orchestrator_skill_items` (tabela nova)

**Arquivo:** `sub_pr_1_01_function_orchestrator_skill_items.sql`

**SUB-PR:** 1.

```sql
-- Enum stage-oriented refletindo paths reais do pipeline de skill (D-PS-79)
CREATE TYPE skill_pipeline_stage AS ENUM (
  'slug_match',      -- caminho 0: hit direto de slug exato em catálogo curado
  'alias_match',     -- caminho 1: hit via dicionário de aliases (label normalizado)
  'llm_new',         -- caminho 2: LLM propôs canonical novo passando hard gate
  'race_recovered',  -- recuperação de race condition (duplicação concorrente)
  'gate_rejected',   -- LLM propôs canonical novo mas hard gate barrou
  'fallback_error'   -- exceção upstream sem categorização clara — registro defensivo
);

-- Status final do item agrupando outcome
-- Inclui 'success' para paridade com FORI (que usa 'success' em process-item.ts:199 + 206)
CREATE TYPE skill_item_status AS ENUM (
  'success',         -- caminho feliz: skill resolveu para canonical_skill_id válido
  'reused',          -- skill reaproveitou canonical pré-existente
  'created_pending', -- canonical novo criado em status='pending' (aguardando promoção)
  'gate_rejected',   -- skill barrada pelo hard gate; canonical_skill_id = NULL
  'failed'           -- erro terminal no item; canonical_skill_id = NULL
);

CREATE TABLE function_orchestrator_skill_items (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id uuid NOT NULL REFERENCES function_orchestrator_runs(id) ON DELETE CASCADE,
  job_posting_id uuid NOT NULL REFERENCES job_postings(id) ON DELETE CASCADE,
  raw_item_index int,
  -- Identificação da skill original
  skill_raw_name text NOT NULL,
  -- J-1 fix v2.5: CHECK alinhado ao SkillType real ([VALIDADO via DB ground truth + LLM SYSTEM_PROMPT.ts:17-89 + lib/pipeline/types.ts:23 + lib/pipeline/skill-type-guard.ts:13])
  -- Versões anteriores desta spec tinham CHECK ('hard', 'soft', 'tool', 'cert') — inventado, nunca usado por LLM/DB/runtime.
  -- Rename coluna skill_raw_kind → skill_raw_type para paridade nominal com LLMSkillItem.type + job_canonical_skills.skill_type.
  skill_raw_type text NOT NULL CHECK (skill_raw_type IN ('technical', 'behavioral', 'hybrid')),
  needs_review boolean NOT NULL DEFAULT false,
  -- Resultado
  canonical_skill_id uuid REFERENCES job_canonical_skills(id) ON DELETE SET NULL,
  canonical_skill_proposed text,
  canonical_status text CHECK (canonical_status IS NULL OR canonical_status IN ('active', 'pending')),
  skill_confidence numeric(4,3),
  similarity_score numeric(4,3),
  reasoning text,
  pipeline_stage skill_pipeline_stage NOT NULL,
  status skill_item_status NOT NULL,
  error_type text,
  error_detail text,
  processed_at timestamptz NOT NULL DEFAULT NOW(),
  created_at timestamptz NOT NULL DEFAULT NOW(),
  -- Defesa em profundidade contra parser bug ou LLM retornando valor fora da faixa
  CONSTRAINT chk_foski_skill_confidence_range
    CHECK (skill_confidence IS NULL OR (skill_confidence >= 0 AND skill_confidence <= 1)),
  CONSTRAINT chk_foski_similarity_range
    CHECK (similarity_score IS NULL OR (similarity_score >= 0 AND similarity_score <= 1))
);

-- Idempotência de retry: 1 item por (run, vaga, posição-no-batch)
CREATE UNIQUE INDEX uq_foski_run_posting_index
  ON function_orchestrator_skill_items (run_id, job_posting_id, raw_item_index)
  WHERE raw_item_index IS NOT NULL;

-- Índices de consulta para queries Limiares + aggregator
CREATE INDEX idx_foski_run_id ON function_orchestrator_skill_items (run_id);
CREATE INDEX idx_foski_canonical_skill_id ON function_orchestrator_skill_items (canonical_skill_id);
CREATE INDEX idx_foski_created_at ON function_orchestrator_skill_items (created_at);
CREATE INDEX idx_foski_pipeline_stage ON function_orchestrator_skill_items (pipeline_stage);
CREATE INDEX idx_foski_status ON function_orchestrator_skill_items (status);

-- Trigger updated_at via convenção pré-existente (se aplicável)
-- (Nenhum trigger necessário neste momento; coluna updated_at não é parte do schema mínimo)

COMMENT ON TABLE function_orchestrator_skill_items IS
  'Telemetria item-por-item de skills processadas pelo pipeline de orchestrator. '
  'Paritária a function_orchestrator_role_items. Caminho quarantined fica fora (D-PS-83).';

COMMENT ON COLUMN function_orchestrator_skill_items.raw_item_index IS
  'Índice da skill no batch original processado pelo run para a vaga. '
  'Garante idempotência de retry junto com UNIQUE(run_id, job_posting_id, raw_item_index). '
  'NULL apenas em registros legados pré-instrumentação; populado obrigatoriamente em INSERTs novos.';

COMMENT ON COLUMN function_orchestrator_skill_items.needs_review IS
  'Sinal propagado do skill-type-guard upstream quando classificação automática '
  'do tipo (technical/behavioral/hybrid) tem baixa confiança. Não bloqueia processamento; '
  'serve para curadoria manual posterior.';  -- K-6 fix v2.6: prose tipo paritary ao SkillType real (J-1 fix); versões anteriores usavam (hard/soft/tool/cert)
```

**Critério de aceite (§7.2):** tabela criada com 5 índices + **3 CHECK constraints** + 1 UNIQUE index parcial; `pg_constraint` reporta `chk_foski_skill_confidence_range` E `chk_foski_similarity_range` ativos (J-9 fix v2.5 — paritário ao DDL acima que declara ambos); `pg_indexes` reporta `uq_foski_run_posting_index` com filtro `(raw_item_index IS NOT NULL)`; `pg_type` reporta `skill_pipeline_stage` com 6 valores (`slug_match`, `alias_match`, `llm_new`, `race_recovered`, `gate_rejected`, `fallback_error`) e `skill_item_status` com 5 valores (`success`, `reused`, `created_pending`, `gate_rejected`, `failed`); `information_schema.columns` reporta `pipeline_stage` e `status` da FOSI com `data_type='USER-DEFINED'` e `udt_name` igual ao enum correspondente (paritário ao critério análogo em §2.x para FORI). Coluna renomeada `skill_raw_type` (J-1 fix v2.5) com `CHECK IN ('technical', 'behavioral', 'hybrid')` — paritary ao SkillType real do código.

### §2.2 — `function_orchestrator_runs` ADD colunas de skill

**Arquivo:** `sub_pr_1_02_function_orchestrator_runs_add_skill_counters.sql`

**SUB-PR:** 1.

```sql
ALTER TABLE function_orchestrator_runs
  ADD COLUMN skills_extracted int NOT NULL DEFAULT 0,
  ADD COLUMN skills_reused int NOT NULL DEFAULT 0,
  ADD COLUMN skills_pending_created int NOT NULL DEFAULT 0,
  ADD COLUMN skills_gate_rejected int NOT NULL DEFAULT 0,
  ADD COLUMN skills_failed int NOT NULL DEFAULT 0;

COMMENT ON COLUMN function_orchestrator_runs.skills_extracted IS
  'Total de skills processadas pelo run independente de outcome. Stage-oriented (D-PS-84).';
COMMENT ON COLUMN function_orchestrator_runs.skills_reused IS
  'Skills que reaproveitaram canonical_skill_id pré-existente (slug_match + alias_match + race_recovered).';
COMMENT ON COLUMN function_orchestrator_runs.skills_pending_created IS
  'Skills que criaram canonical novo em status=pending (path llm_new passando hard gate).';
COMMENT ON COLUMN function_orchestrator_runs.skills_gate_rejected IS
  'Skills barradas pelo hard gate; canonical_skill_id ficou NULL em job_posting_skills.';
COMMENT ON COLUMN function_orchestrator_runs.skills_failed IS
  'Skills com erro terminal; canonical_skill_id ficou NULL em job_posting_skills.';
```

**Decisão de design (D-PS desta sprint):** os contadores skill são stage-oriented, não outcome-oriented como os 8 pré-existentes role (`total`, `curated`, `curated_fallback`, `low_quality`, `failed`, `pending_review`, `canonical_created`, `canonical_promoted`). **M-2 fix v2.8 — DB ground truth confirmado via psql:** as 8 colunas em `function_orchestrator_runs` são bare-named (sem prefixo `roles_`) com os nomes literais acima — assimetria histórica intencional entre nome de coluna e valores do ENUM `role_item_status` preservada por retrocompat (`curated`/`curated_fallback` são os nomes reais das COLUNAS apesar do ENUM agora ter `fallback`/etc — colunas e enum-values têm naming independente desde pré-cleanup-v3.4). Sem necessidade de auditoria adicional E0d sobre estes nomes — já confirmados. Justificativa stage-oriented: o pipeline de skill foi instrumentado retrospectivamente sobre paths já existentes em produção; mapeá-los para os 8 outcomes role exigiria reclassificação semântica que perde fidelidade ao path real. Os 5 contadores stage-oriented refletem 1:1 os valores do enum `skill_pipeline_stage` agregados por run.

**Critério de aceite (§7.2):** 5 colunas em `function_orchestrator_runs` com `DEFAULT 0 NOT NULL`; runs pré-existentes (se houver) automaticamente populadas com 0 sem necessidade de backfill.

### §2.w — Backfill obrigatório de `entity_type` em events legacy (SUB-PR 0)

**Arquivo:** `sub_pr_0_01_backfill_entity_type_legacy_events.sql`

**SUB-PR:** 0 (novo — pré-step bloqueador de SUB-PR 2).

**Motivação:** ground truth via §6.1 (E0i) confirma 36 rows legacy em `events` sem `metadata->>'entity_type'` — 31 `canonical_promoted_dynamic` + 5 `canonical_creation_blocked_low_confidence`. Sem `entity_type` populado, o pre-check de §2.y (`RAISE EXCEPTION se metadata->>'entity_type' NOT IN ('role', 'skill')`) dispara antes de qualquer UPDATE e a migração trava na origem. SUB-PR 0 resolve isso via FK lookup determinístico antes do SUB-PR 2 iniciar.

**Estratégia:** para cada row legacy sem `entity_type`, lookup `resource_id` em `job_canonical_roles` e `job_canonical_skills`. Se resolver em uma das duas → cravar `entity_type` correspondente em `metadata`. Se resolver em ambas (não esperado) → log de inconsistência + `entity_type='unknown'`. Se não resolver em nenhuma → `entity_type='unknown'` + exclusão da rename de §2.y (esses rows ficam com nome antigo permanentemente — auditoria explícita).

```sql
BEGIN;

-- 1. Pré-check: quantas rows legacy precisam de backfill?
DO $$
DECLARE
  v_pending int;
BEGIN
  SELECT COUNT(*) INTO v_pending
  FROM events
  WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
    AND (metadata IS NULL OR metadata->>'entity_type' IS NULL);
  RAISE NOTICE 'Backfill entity_type: % rows legacy detectadas', v_pending;
END$$;

-- L-4 fix v2.7: 2. AMBIGUITY DETECTOR ANTES DAS UPDATEs.
-- Em v2.6 e anteriores, o detector estava DEPOIS das UPDATEs — UPDATE #1 marcava ambíguas como 'role'
-- antes do detector conseguir flagar. Resultado: silent misclassification.
-- Agora: abort migration se houver QUALQUER row com resource_id em ambas as tabelas.
-- Justificativa: backfill é one-shot; ambiguidade real (mesma resource_id apontando para role E skill)
-- é situação anômala que merece investigação manual antes de prosseguir. Diretriz #20 paritary aqui:
-- ambiguidade não é tratável automaticamente sem perda de informação.
DO $$
DECLARE
  v_ambiguous int;
BEGIN
  SELECT COUNT(*) INTO v_ambiguous
  FROM events e
  WHERE e.event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
    AND (e.metadata IS NULL OR e.metadata->>'entity_type' IS NULL)
    AND EXISTS (SELECT 1 FROM job_canonical_roles r WHERE r.id = e.resource_id)
    AND EXISTS (SELECT 1 FROM job_canonical_skills s WHERE s.id = e.resource_id);
  IF v_ambiguous > 0 THEN
    RAISE EXCEPTION 'Backfill ABORTADO: % rows com resource_id apontando para job_canonical_roles E job_canonical_skills simultaneamente. Investigar manualmente antes de re-rodar migration. Query para inspeção: SELECT id, event_name, resource_id, created_at FROM events WHERE event_name IN (...) AND metadata->>''entity_type'' IS NULL AND EXISTS (SELECT 1 FROM job_canonical_roles r WHERE r.id = events.resource_id) AND EXISTS (SELECT 1 FROM job_canonical_skills s WHERE s.id = events.resource_id);', v_ambiguous;
  END IF;
END$$;

-- 3. Backfill via FK lookup (somente após confirmar zero ambiguidade)
-- Padrão esperado: events.resource_id é uuid apontando para job_canonical_roles.id OU job_canonical_skills.id
-- Validar nome real da coluna via §6.1 (resource_id, target_id, entity_id, ou similar).

UPDATE events e
SET metadata = COALESCE(e.metadata, '{}'::jsonb) || jsonb_build_object('entity_type', 'role')
WHERE e.event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
  AND (e.metadata IS NULL OR e.metadata->>'entity_type' IS NULL)
  AND EXISTS (SELECT 1 FROM job_canonical_roles r WHERE r.id = e.resource_id);

UPDATE events e
SET metadata = COALESCE(e.metadata, '{}'::jsonb) || jsonb_build_object('entity_type', 'skill')
WHERE e.event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
  AND (e.metadata IS NULL OR e.metadata->>'entity_type' IS NULL)
  AND EXISTS (SELECT 1 FROM job_canonical_skills s WHERE s.id = e.resource_id);

-- 4. Rows não-resolvidas → entity_type='unknown' (serão excluídas da rename de §2.y)
UPDATE events
SET metadata = COALESCE(metadata, '{}'::jsonb) || jsonb_build_object('entity_type', 'unknown')
WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
  AND (metadata IS NULL OR metadata->>'entity_type' IS NULL);

-- 5. Emit event de auditoria com agregação por destino
INSERT INTO events (event_name, metadata, created_at)
SELECT
  'entity_type_backfilled',
  jsonb_build_object(
    'sprint', 'orchestrator-symmetry-v2.8',
    'role_count', COUNT(*) FILTER (WHERE metadata->>'entity_type' = 'role'),
    'skill_count', COUNT(*) FILTER (WHERE metadata->>'entity_type' = 'skill'),
    'unknown_count', COUNT(*) FILTER (WHERE metadata->>'entity_type' = 'unknown')
  ),
  NOW()
FROM events
WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence');

-- 6. Validação pós-backfill
DO $$
DECLARE
  v_missing int;
BEGIN
  SELECT COUNT(*) INTO v_missing
  FROM events
  WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
    AND (metadata IS NULL OR metadata->>'entity_type' IS NULL);
  IF v_missing > 0 THEN
    RAISE EXCEPTION 'Backfill incompleto: % rows ainda sem entity_type', v_missing;
  END IF;
END$$;

COMMIT;
```

**Critério de aceite (§7.2):** `SELECT COUNT(*) FROM events WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence') AND metadata->>'entity_type' IS NULL` = 0; smoke S25 (backfill paritário cobrindo role + skill + auditoria) verde; event `entity_type_backfilled` emitido com contagens role/skill/unknown.

### §2.x — Corretivos retroativos em FORI (CHECK 0..1 + raw_item_index + UNIQUE + CASCADE + ENUMs)

**Arquivo:** `sub_pr_2_01_role_items_retroactive_constraints.sql`

**SUB-PR:** 2 (executa após rename de §2.6 — toca a tabela já com nome novo).

```sql
-- Pré-check obrigatório: não pode haver dado poluído antes de aplicar CHECK
DO $$
DECLARE
  v_count int;
BEGIN
  SELECT COUNT(*) INTO v_count
  FROM function_orchestrator_role_items
  WHERE confidence > 1 OR confidence < 0;
  IF v_count > 0 THEN
    RAISE EXCEPTION 'Migration abortada: % rows com confidence fora da faixa [0,1]', v_count;
  END IF;
END$$;

-- CHECK constraint paritária à do FOSI
ALTER TABLE function_orchestrator_role_items
  ADD CONSTRAINT chk_fori_confidence_range
  CHECK (confidence IS NULL OR (confidence >= 0 AND confidence <= 1));

-- Coluna raw_item_index para idempotência paritária
ALTER TABLE function_orchestrator_role_items
  ADD COLUMN raw_item_index int;

-- UNIQUE index parcial paritário
CREATE UNIQUE INDEX uq_fori_run_posting_index
  ON function_orchestrator_role_items (run_id, job_posting_id, raw_item_index)
  WHERE raw_item_index IS NOT NULL;

COMMENT ON COLUMN function_orchestrator_role_items.raw_item_index IS
  'Índice da role no batch original processado pelo run para a vaga. '
  'Garante idempotência de retry junto com UNIQUE(run_id, job_posting_id, raw_item_index). '
  'NULL em registros pré-instrumentação; populado obrigatoriamente em INSERTs novos via process-item.ts.';

-- Conversão pipeline_stage e status de text+CHECK para ENUM (paridade total com FOSI)
-- Valores atuais (ground truth): pipeline_stage TEM 6 valores; status TEM N valores.
-- Snapshot via SUB-PR 1 §6.1: SELECT DISTINCT pipeline_stage, status FROM function_orchestrator_role_items;
-- Os valores criados no ENUM abaixo são os ground-truth confirmados antes da migration.

CREATE TYPE role_pipeline_stage AS ENUM (
  'deterministic',
  'cache_hit_layer_0',
  'dict_match_layer_1',
  'suggested_role_layer_2',
  'llm_pure_layer_3',
  'fallback_error'
);

-- L-1 fix v2.7 — ENUM paritary ao DB CHECK constraint REAL (function_orchestrator_items_status_check):
-- DB ground truth: CHECK (status IN ('success', 'fallback', 'low_quality', 'pending_review', 'failed'))
-- TS ground truth (lib/pipeline/types.ts:167-172): ItemStatus = 'success' | 'fallback' | 'low_quality' | 'pending_review' | 'failed'
-- Versões anteriores desta spec tinham 'curated' e 'curated_fallback' INVENTADOS + omitiam 'fallback' REAL.
-- Bug viveu 14 iterações; detectado em v2.7 via DB cross-check (Diretriz §0.2.12).
CREATE TYPE role_item_status AS ENUM (
  'success',
  'fallback',
  'low_quality',
  'pending_review',
  'failed'
);

-- Drop CHECK constraints antigos (redundantes pós-ENUM)
ALTER TABLE function_orchestrator_role_items
  DROP CONSTRAINT IF EXISTS function_orchestrator_role_items_pipeline_stage_check;
ALTER TABLE function_orchestrator_role_items
  DROP CONSTRAINT IF EXISTS function_orchestrator_role_items_status_check;

-- ALTER COLUMN TYPE com USING cast
ALTER TABLE function_orchestrator_role_items
  ALTER COLUMN pipeline_stage TYPE role_pipeline_stage USING pipeline_stage::role_pipeline_stage;

ALTER TABLE function_orchestrator_role_items
  ALTER COLUMN status TYPE role_item_status USING status::role_item_status;

-- ON DELETE CASCADE retroativo em FKs (paridade total com FOSI)
-- Padrão FOSI: run_id e job_posting_id → CASCADE; canonical_skill_id → SET NULL
-- Paridade FORI: run_id e job_posting_id → CASCADE; canonical_role_id → SET NULL

-- Lista exata de FKs vem do §6.2 (ground truth via pg_constraint). Padrão esperado:
ALTER TABLE function_orchestrator_role_items
  DROP CONSTRAINT IF EXISTS function_orchestrator_role_items_run_id_fkey;
ALTER TABLE function_orchestrator_role_items
  ADD CONSTRAINT function_orchestrator_role_items_run_id_fkey
  FOREIGN KEY (run_id) REFERENCES function_orchestrator_runs(id) ON DELETE CASCADE;

ALTER TABLE function_orchestrator_role_items
  DROP CONSTRAINT IF EXISTS function_orchestrator_role_items_job_posting_id_fkey;
ALTER TABLE function_orchestrator_role_items
  ADD CONSTRAINT function_orchestrator_role_items_job_posting_id_fkey
  FOREIGN KEY (job_posting_id) REFERENCES job_postings(id) ON DELETE CASCADE;

ALTER TABLE function_orchestrator_role_items
  DROP CONSTRAINT IF EXISTS function_orchestrator_role_items_canonical_role_id_fkey;
ALTER TABLE function_orchestrator_role_items
  ADD CONSTRAINT function_orchestrator_role_items_canonical_role_id_fkey
  FOREIGN KEY (canonical_role_id) REFERENCES job_canonical_roles(id) ON DELETE SET NULL;
```

**Decisão de design (D-PS desta sprint):** conversão de FORI para ENUMs paritários estabelece paridade total de tipos com FOSI. Custo é único (migration única, valores ground-truth confirmados via §6.1) e o ganho é robustez (impossibilidade de valor fora do enum por construção). `CREATE TYPE ENUM` no PG é imutável após criação (adicionar valores requer `ALTER TYPE ADD VALUE`), o que é trade-off conhecido e aceito para esta sprint pré-MVP.

**Crítério de aceite (§7.2):** `pg_constraint` reporta `chk_fori_confidence_range` ativo; `pg_indexes` reporta `uq_fori_run_posting_index` com filtro parcial; `pg_type` reporta `role_pipeline_stage` e `role_item_status` com valores esperados; `information_schema.columns` reporta `pipeline_stage` e `status` com `data_type='USER-DEFINED'` e `udt_name` igual ao enum correspondente; `pg_constraint` reporta 3 FKs com `confdeltype='c'` (CASCADE) ou `confdeltype='n'` (SET NULL) conforme paridade FOSI; pré-check passou com 0 rows.

### §2.y — Migração de evolução de `event_names` para `role_*`/`skill_*`

**Arquivo:** `sub_pr_2_02_evolve_event_names_role_skill.sql`

**SUB-PR:** 2 (paralelo a §2.x corretivos retroativos; depende do SUB-PR 0 §2.w concluído).

Ground truth via SUB-PR 1 (§6.1) confirma:
- `canonical_promoted_dynamic` tem ~31 rows em `events`
- `canonical_creation_blocked_low_confidence` tem ~5 rows em `events`
- **Antes do SUB-PR 0 (§2.w)**: nenhuma row tinha `metadata->>'entity_type'` populado (36 rows órfãs)
- **Pós SUB-PR 0**: todas as 36 rows têm `entity_type` populado via FK lookup (`'role'`/`'skill'`/`'unknown'`)
- **`events_event_name_check` constraint NÃO EXISTE no DB** (ground truth via E0i §6.1) — `event_name` é text livre. Esta migration CRIA a constraint do zero com whitelist completa derivada de E0i (77 event_names ativos)

```sql
BEGIN;

-- 1. Pré-check: todas as rows dos 2 nomes antigos têm metadata.entity_type populado
-- pelo SUB-PR 0 (§2.w)? Aceita 'role', 'skill', 'unknown' — rows 'unknown' ficam intactas.
DO $$
DECLARE
  v_missing int;
BEGIN
  SELECT COUNT(*) INTO v_missing
  FROM events
  WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
    AND (metadata IS NULL OR metadata->>'entity_type' IS NULL);
  IF v_missing > 0 THEN
    RAISE EXCEPTION 'SUB-PR 0 (§2.w) não rodou: % rows ainda sem entity_type', v_missing;
  END IF;
END$$;

-- 2. Update determinístico baseado em metadata.entity_type
-- Rows com entity_type='unknown' (não resolvidas via FK lookup) ficam com nome antigo —
-- auditoria documental explícita do que não conseguiu ser discriminado.
UPDATE events
SET event_name = CASE metadata->>'entity_type'
  WHEN 'role' THEN 'role_promoted_dynamic'
  WHEN 'skill' THEN 'skill_promoted_dynamic'
  ELSE event_name  -- 'unknown' permanece com nome antigo
END
WHERE event_name = 'canonical_promoted_dynamic'
  AND metadata->>'entity_type' IN ('role', 'skill');

UPDATE events
SET event_name = CASE metadata->>'entity_type'
  WHEN 'role' THEN 'role_creation_blocked_low_confidence'
  WHEN 'skill' THEN 'skill_creation_blocked_low_confidence'
  ELSE event_name
END
WHERE event_name = 'canonical_creation_blocked_low_confidence'
  AND metadata->>'entity_type' IN ('role', 'skill');

-- 3. CRIAR CHECK constraint (nova — não existia previamente)
-- Lista exaustiva derivada da auditoria E0i (§6.1): SELECT DISTINCT event_name FROM events.
-- Ground truth: DB tem 77 event_names distintos + 6 novos paritários = 83 valores totais.
-- IMPLEMENTOR: colar lista LITERAL do artefato anexo `EVENT_NAMES-orchestrator-symmetry-v2.8.md`
-- produzido pelo SUB-PR 1 (entregável obrigatório). NÃO usar placeholders abaixo —
-- substituir a cláusula IN inteira pela lista literal de 83 valores antes da migration:

ALTER TABLE events ADD CONSTRAINT events_event_name_check
  CHECK (event_name IN (
    -- 4 nomes novos paritários desta sprint (substituem 2 antigos para rows com entity_type role/skill)
    'role_promoted_dynamic',
    'skill_promoted_dynamic',
    'role_creation_blocked_low_confidence',
    'skill_creation_blocked_low_confidence',
    -- 2 nomes novos paritários cravados pelos triggers de §2.z
    'canonical_role_created',
    'canonical_skill_created',
    -- NOTA H-8: os 77 demais event_names ativos no DB já incluem (a) 2 legacy mantidos para rows
    -- entity_type='unknown' (`canonical_promoted_dynamic`, `canonical_creation_blocked_low_confidence`) e
    -- (b) `entity_type_backfilled` emitido pelo SUB-PR 0 antes do snapshot E0i. Não duplicar abaixo.
    -- Os 77 demais event_names ativos no DB — placeholder abaixo é representativo;
    -- IMPLEMENTOR cola lista literal do artefato anexo EVENT_NAMES-orchestrator-symmetry-v2.8.md:
    'role_creation_blocked_missing_confidence',
    'skill_creation_blocked_missing_confidence',
    'role_creation_blocked_invalid_slug',
    'skill_creation_blocked_invalid_slug',
    'persist_curation.skill_map_failed',
    'catchup_pending_promotions_executed',
    'job_posting_expurged',
    'canonical_promoted_dynamic',                       -- legacy: já em E0i
    'canonical_creation_blocked_low_confidence',        -- legacy: já em E0i
    'entity_type_backfilled'                            -- auditoria SUB-PR 0: já em E0i
    -- + 67 nomes restantes via artefato anexo EVENT_NAMES-orchestrator-symmetry-v2.8.md
  ));

-- 4. Validação pós-migração
DO $$
DECLARE
  v_renamed_count int;
  v_remaining_unknown_count int;
BEGIN
  SELECT COUNT(*) INTO v_renamed_count
  FROM events
  WHERE event_name IN (
    'role_promoted_dynamic', 'skill_promoted_dynamic',
    'role_creation_blocked_low_confidence', 'skill_creation_blocked_low_confidence'
  );
  SELECT COUNT(*) INTO v_remaining_unknown_count
  FROM events
  WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence');
  RAISE NOTICE 'Migration: % rows renomeadas; % rows preservadas com nome antigo (entity_type=unknown)',
    v_renamed_count, v_remaining_unknown_count;
END$$;

COMMIT;
```

**Atualização paritária dos call sites TS que emitem os eventos antigos:**

Grep pré-PR via §6.1 procedimento geral identifica todos os pontos de emissão de `canonical_promoted_dynamic` e `canonical_creation_blocked_low_confidence` no código. Para cada um, atualizar para emitir o nome novo discriminado pela entidade que está sendo processada no momento da emissão:

- `lib/pipeline/canonical-promotion.ts` (ou similar) — emissor de `canonical_promoted_dynamic` passa a emitir `role_promoted_dynamic` ou `skill_promoted_dynamic` conforme entidade
- `lib/pipeline/hard-gate.ts` (ou similar) — emissor de `canonical_creation_blocked_low_confidence` passa a emitir `role_creation_blocked_low_confidence` ou `skill_creation_blocked_low_confidence` conforme entidade

`metadata->>'entity_type'` continua sendo populado normalmente (não removido — serve para auditoria/cross-check e para queries que precisem agregar sobre ambas entidades sem split).

**Critério de aceite (§7.2):** `pg_constraint` reporta `events_event_name_check` ativo com lista exaustiva derivada de E0i; smokes S19a/S19b (eventos role/skill emitidos com nomes novos pós call site update) verdes; rows pós-migration com entity_type populado têm nomes novos discriminados; rows com `entity_type='unknown'` permanecem com nome antigo + log de auditoria.

### §2.z — Triggers paritários `canonical_role_created` / `canonical_skill_created`

**Arquivo:** `sub_pr_2_03_triggers_canonical_role_skill_created.sql`

**SUB-PR:** 2 (paralelo a §2.x e §2.y).

**Motivação:** cleanup v3.4 cravou os nomes canônicos `canonical_role_created` e `canonical_skill_created` em D-PS-50 + LK-PS-22, mas o trigger de emissão real nunca foi criado. Ground truth via §6.1 (E0g) confirma: 0 rows desses nomes em `events`. Sem trigger, aggregator filtra eventos inexistentes → drift O7 sempre 0 em ambos os lados. Esta sprint cria os 2 triggers paritários por construção.

```sql
BEGIN;

-- Função do trigger lado ROLE — payload sem created_via (coluna não existe; ver D-PS-88)
CREATE OR REPLACE FUNCTION public.fn_emit_canonical_role_created()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $function$
BEGIN
  INSERT INTO events (event_name, metadata, created_at)
  VALUES (
    'canonical_role_created',
    jsonb_build_object(
      'canonical_id', NEW.id,
      'label', NEW.label,
      'entity_type', 'role'
    ),
    NOW()
  );
  RETURN NEW;
END;
$function$;

-- Função do trigger lado SKILL (paritária — payload sem created_via)
CREATE OR REPLACE FUNCTION public.fn_emit_canonical_skill_created()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $function$
BEGIN
  INSERT INTO events (event_name, metadata, created_at)
  VALUES (
    'canonical_skill_created',
    jsonb_build_object(
      'canonical_id', NEW.id,
      'label', NEW.label,
      'entity_type', 'skill'
    ),
    NOW()
  );
  RETURN NEW;
END;
$function$;

-- Triggers AFTER INSERT com WHEN guard (pg_trigger_depth() = 0) para prevenir loop
-- caso algum trigger downstream insira de volta em job_canonical_roles/skills
CREATE TRIGGER trg_jcr_emit_canonical_role_created
  AFTER INSERT ON job_canonical_roles
  FOR EACH ROW
  WHEN (pg_trigger_depth() = 0)
  EXECUTE FUNCTION public.fn_emit_canonical_role_created();

CREATE TRIGGER trg_jcs_emit_canonical_skill_created
  AFTER INSERT ON job_canonical_skills
  FOR EACH ROW
  WHEN (pg_trigger_depth() = 0)
  EXECUTE FUNCTION public.fn_emit_canonical_skill_created();

COMMENT ON TRIGGER trg_jcr_emit_canonical_role_created ON job_canonical_roles IS
  'Emite event canonical_role_created em events sempre que uma role canonical é criada. '
  'Usado pelo aggregator do dashboard global para calcular drift O7 role.';

COMMENT ON TRIGGER trg_jcs_emit_canonical_skill_created ON job_canonical_skills IS
  'Emite event canonical_skill_created em events sempre que uma skill canonical é criada. '
  'Usado pelo aggregator do dashboard global para calcular drift O7 skill.';

COMMIT;
```

**`created_via` removido do payload (D-PS-88 opção c):** ground truth via §6.1 (E0a expandido) confirma que a coluna `created_via` **não existe** em `job_canonical_roles` nem em `job_canonical_skills`. Adicionar coluna nova só para o trigger lê-la geraria coluna perpetuamente vazia (nenhum caller existente popularia o campo) — overengineering sem ganho real dado que a base começa zerada no deploy. Payload do event fica `{canonical_id, label, entity_type}`. Rastreabilidade de origem (LLM pipeline vs admin manual vs backfill) fica como dívida explícita LK-PS-NEW caso necessidade futura justifique adicionar a coluna em ambas as tabelas paritárias com callers populando-a.

**Critério de aceite (§7.2):** smokes S20a/S20b (inserção sintética de canonical role/skill emite eventos corretos) verdes; `pg_trigger` reporta ambos triggers ativos; `pg_get_functiondef` confirma `search_path TO 'public', 'pg_temp'` em ambas funções; após smokes S20a/S20b, `SELECT COUNT(*) FROM events WHERE event_name='canonical_role_created'` e idem para skill > 0.

### §2.3 — Validação de schema simétrico entre `_role_items` e `_skill_items`

**Arquivo:** `sub_pr_1_03_validate_symmetric_schema.sql`

**SUB-PR:** 1.

```sql
-- Validação: 18 colunas em function_orchestrator_skill_items
-- (id, run_id, job_posting_id, raw_item_index, skill_raw_name, skill_raw_type,
--  needs_review, canonical_skill_id, canonical_skill_proposed, skill_confidence,
--  similarity_score, reasoning, pipeline_stage, status, error_type, error_detail,
--  processed_at, created_at)
DO $$
DECLARE
  v_skill_cols int;
  v_role_cols int;
BEGIN
  SELECT COUNT(*) INTO v_skill_cols
  FROM information_schema.columns
  WHERE table_schema='public' AND table_name='function_orchestrator_skill_items';

  IF v_skill_cols < 18 THEN
    RAISE EXCEPTION 'FOSI tem apenas % colunas; esperado >= 18', v_skill_cols;
  END IF;

  -- Validar que FORI ganhou raw_item_index pós-migration §2.x corretivos
  SELECT COUNT(*) INTO v_role_cols
  FROM information_schema.columns
  WHERE table_schema='public'
    AND table_name='function_orchestrator_role_items'
    AND column_name='raw_item_index';
  IF v_role_cols <> 1 THEN
    RAISE EXCEPTION 'FORI não tem coluna raw_item_index esperada';
  END IF;

  -- Validar CHECK constraints paritárias
  IF NOT EXISTS (
    SELECT 1 FROM pg_constraint c
    JOIN pg_class t ON t.oid = c.conrelid
    WHERE t.relname = 'function_orchestrator_role_items'
      AND c.conname = 'chk_fori_confidence_range'
  ) THEN
    RAISE EXCEPTION 'FORI não tem CHECK constraint chk_fori_confidence_range';
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM pg_constraint c
    JOIN pg_class t ON t.oid = c.conrelid
    WHERE t.relname = 'function_orchestrator_skill_items'
      AND c.conname = 'chk_foski_skill_confidence_range'
  ) THEN
    RAISE EXCEPTION 'FOSI não tem CHECK constraint chk_foski_skill_confidence_range';
  END IF;
END$$;
```

**Critério de aceite (§7.2):** bloco DO roda sem RAISE EXCEPTION.

### §2.4 — RPC `count_skill_items_by_status` (obrigatória para finalizers)

**Arquivo:** `sub_pr_1_04_rpc_count_skill_items_by_status.sql`

**SUB-PR:** 2.

```sql
CREATE OR REPLACE FUNCTION count_skill_items_by_status(p_run_id uuid)
RETURNS TABLE(
  status text,
  c bigint
)
LANGUAGE sql
STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT status::text, COUNT(*) AS c
  FROM function_orchestrator_skill_items
  WHERE run_id = p_run_id
  GROUP BY status;
$$;

-- Sem SECURITY DEFINER: caller é service_role admin, paridade com count helpers similares
-- já existentes no projeto.

COMMENT ON FUNCTION count_skill_items_by_status(uuid) IS
  'Contagem agregada de itens FOSI por status para um run. Usada pelos 3 finalizers '
  'em §3.3 para popular as 5 colunas skills_* em function_orchestrator_runs via UPDATE atômica.';

GRANT EXECUTE ON FUNCTION count_skill_items_by_status(uuid) TO service_role, authenticated;
```

**Critério de aceite (§7.2):** `pg_get_function_arguments('count_skill_items_by_status'::regproc)` retorna `'p_run_id uuid'`; `pg_proc.proconfig` inclui `search_path=public, pg_temp`; `prosecdef = false`.

### §2.5 — (reservado para uso futuro)

### §2.6 — Rename `function_orchestrator_items` → `function_orchestrator_role_items`

**Arquivo:** `sub_pr_2_04_rename_function_orchestrator_items.sql`
**Atende:** S-ORCH-6.
**SUB-PR:** 2 (sequencial após SUB-PR 1).
**Pré-requisito:** SUB-PR 1 finalizado — `AUDIT-orchestrator-symmetry-v2.8-<data>.md` lista exata de objetos a renomear/atualizar via §6.2.

A migration é dividida conceitualmente em dois blocos sequenciais dentro de uma única transação: §2.6a faz o rename mecânico de objetos (tabela + constraints + índices + triggers); §2.6b faz o rewrite mecânico dos corpos de funções dependentes via `CREATE OR REPLACE` (preserva GRANTs amplas: PUBLIC, anon, authenticated, service_role).

#### §2.6a — Rename mecânico de objetos

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

-- FKs — nomes reais via ground truth §6.1. Padrão antigo: function_orchestrator_items_<col>_fkey
ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_run_id_fkey
  TO function_orchestrator_role_items_run_id_fkey;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_job_posting_id_fkey
  TO function_orchestrator_role_items_job_posting_id_fkey;

ALTER TABLE function_orchestrator_role_items
  RENAME CONSTRAINT function_orchestrator_items_canonical_role_id_fkey
  TO function_orchestrator_role_items_canonical_role_id_fkey;

-- 3. RENOMEAR ÍNDICES — nomes reais via ground truth (prefixo atual é fo_items_*, NÃO function_orchestrator_items_*)
-- Lista exata validada via §6.1 E0j (auditoria de índices fo_items_*):
--   SELECT indexname FROM pg_indexes
--   WHERE tablename='function_orchestrator_role_items' AND indexname LIKE 'fo_items_%';
-- DB real reporta 6 índices (lista de rename correta cravada abaixo):

ALTER INDEX fo_items_run_id_idx
  RENAME TO function_orchestrator_role_items_run_id_idx;

ALTER INDEX fo_items_job_posting_id_idx
  RENAME TO function_orchestrator_role_items_job_posting_id_idx;

ALTER INDEX fo_items_job_posting_created_idx
  RENAME TO function_orchestrator_role_items_job_posting_created_idx;

ALTER INDEX fo_items_canonical_role_id_idx
  RENAME TO function_orchestrator_role_items_canonical_role_id_idx;

ALTER INDEX fo_items_canonical_proposed_idx
  RENAME TO function_orchestrator_role_items_canonical_proposed_idx;

ALTER INDEX fo_items_status_idx
  RENAME TO function_orchestrator_role_items_status_idx;

-- NOTA: índices `fo_items_pipeline_stage_idx` e `fo_items_created_at_idx` alegados em
-- versões anteriores NÃO existem no DB. Removidos do rename.
-- Se essas colunas precisarem de índice no futuro, criar via CREATE INDEX em frente futura.

-- NÃO usar IF EXISTS aqui — se um índice esperado não existir, o ALTER deve falhar
-- para forçar atualização da spec via auditoria. IF EXISTS mascara incompletude.

-- 4. RENOMEAR TRIGGERS (verificar via pg_trigger antes — alguns podem não existir)
ALTER TRIGGER IF EXISTS trg_foi_jcr_confidence_insert
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_insert;

ALTER TRIGGER IF EXISTS trg_foi_jcr_confidence_update
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_update;

ALTER TRIGGER IF EXISTS trg_foi_jcr_confidence_delete
  ON function_orchestrator_role_items
  RENAME TO trg_fori_jcr_confidence_delete;

COMMIT;
```

#### §2.6b — Rewrite mecânico de funções dependentes

Ground truth via §6.2 (E0e) identifica todas as funções SQL que referenciam o nome antigo no corpo. Para CADA função identificada, aplicar `CREATE OR REPLACE` com o nome novo no FROM/JOIN/UPDATE/DELETE — sem mudança de lógica, parâmetros, semântica, fonte de dados ou filtros. **Apenas o identificador da tabela muda.**

`CREATE OR REPLACE` preserva GRANTs em todas as funções afetadas. `SET search_path TO 'public', 'pg_temp'` pré-existente é preservado (ground truth confirma que `fn_recompute_jcr_confidence_median` JÁ tem o set; não duplicar).

**Função 1: `fn_recompute_jcr_confidence_median`** (cravada por D-PS-49 da cleanup v3.4 como circuito intocado; rewrite mecânico do identificador é parte de rename, não toque do circuito):

```sql
CREATE OR REPLACE FUNCTION public.fn_recompute_jcr_confidence_median(
  p_canonical_role_id uuid
) RETURNS void
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $function$
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
    FROM function_orchestrator_role_items fori  -- ÚNICA MUDANÇA vs corpo atual: nome novo da tabela
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
$function$;
```

**Função 2 em diante:** ground truth via §6.1 (E0e) confirma 4 funções adicionais referenciando `function_orchestrator_items` por nome, distribuídas em 2 namespaces:

**Função 2: `internal.reset_taxonomy_core`** (namespace `internal`, não `public`):

```sql
-- Schema fonte: internal (não public)
-- proconfig original: validar via pg_proc.proconfig antes de aplicar — preservar exatamente
CREATE OR REPLACE FUNCTION internal.reset_taxonomy_core()
RETURNS void
LANGUAGE plpgsql
-- SET search_path original preservado (validar via §6.1 antes do PR)
AS $function$
BEGIN
  -- corpo idêntico ao atual, com UMA mudança: nome da tabela no DELETE
  DELETE FROM function_orchestrator_role_items;  -- ANTES: function_orchestrator_items
  -- demais DELETEs preservados
  -- ...
END;
$function$;
```

**Função 3: `public.cleanup_batch_items`** — **ATENÇÃO crítica:** ground truth via §6.1 confirma `proconfig=search_path=public` (sem `pg_temp`). Rewrite mecânico DEVE preservar exatamente essa config, NÃO adicionar `pg_temp` por padronização:

```sql
CREATE OR REPLACE FUNCTION public.cleanup_batch_items(...)
RETURNS void
LANGUAGE plpgsql
SET search_path TO 'public'  -- PRESERVAR exatamente — sem pg_temp por design dessa função
AS $function$
BEGIN
  -- corpo com nome novo da tabela
  DELETE FROM function_orchestrator_role_items WHERE ...;  -- ANTES: function_orchestrator_items
END;
$function$;
```

**Função 4: `public.merge_canonicals`**:

```sql
CREATE OR REPLACE FUNCTION public.merge_canonicals(...)
RETURNS ...
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'  -- ground truth via §6.1
AS $function$
BEGIN
  -- corpo com nome novo
  UPDATE function_orchestrator_role_items SET canonical_role_id = ... WHERE ...;
END;
$function$;
```

**Função 5: `public.release_quarantined_jobs_limited`**:

```sql
CREATE OR REPLACE FUNCTION public.release_quarantined_jobs_limited(...)
RETURNS ...
LANGUAGE plpgsql
-- SET search_path original via §6.1 ground truth
AS $function$
BEGIN
  -- corpo com nome novo
  DELETE FROM function_orchestrator_role_items WHERE ...;
END;
$function$;
```

**Validação obrigatória pós-rewrite (smoke S21 — §7.2):**

```sql
-- Para cada função reescrita, validar que proconfig original foi preservado
SELECT proname, pronamespace::regnamespace, prosrc, proconfig
FROM pg_proc
WHERE proname IN (
  'reset_taxonomy_core',
  'cleanup_batch_items',
  'fn_recompute_jcr_confidence_median',
  'merge_canonicals',
  'release_quarantined_jobs_limited'
);
-- Esperado: proconfig de cada função idêntico ao snapshot pré-rewrite via §6.1.
-- Cross-check com diff: cada função tem 'function_orchestrator_role_items' no body
-- (não mais 'function_orchestrator_items').
```

**Funções alegadas em sessões anteriores que NÃO existem no DB** (ground truth via §6.1 confirma ausência): `maintenance_phase_1`, `process_opus_create_new`. Removidas como alvos da migration.

**Notas críticas:**

- **Lista exata de objetos a renomear vem da §6.2.** A migration é um TEMPLATE — o conteúdo final depende do que a auditoria retornar. SUB-PR 2 só inicia após SUB-PR 1 fechado com auditoria publicada.
- **Funções de trigger `fn_jcr_confidence_median_insert/update/delete` NÃO precisam ser renomeadas** (ground truth confirma que NÃO referenciam `function_orchestrator_items` por nome — apenas chamam `fn_recompute_jcr_confidence_median(canonical_role_id)`).
- **Janela atual:** ground truth confirmou 0 rows de produção nos últimos 30 dias (ambiente pré-MVP). Rename é de baixíssimo risco.
- **Pós-migration:** todos os call sites TS precisam ser atualizados na mesma sprint (§3.4). Migration SQL sozinha quebra a aplicação.
- **D-PS-49 da cleanup v3.4 reafirmada:** rewrite do identificador `function_orchestrator_items → function_orchestrator_role_items` no corpo de funções é mecânica de rename. Lógica, parâmetros lidos de `pipeline_config`, semântica de cálculo (`PERCENTILE_CONT(0.5)`), filtros de `status IN ('active', 'pending')`, fonte de dados (`function_orchestrator_role_items.confidence`) — tudo permanece idêntico.

**Critério de aceite (§7.2):** `to_regclass('public.function_orchestrator_items')` retorna NULL; `to_regclass('public.function_orchestrator_role_items')` retorna nome válido; `pg_get_functiondef('fn_recompute_jcr_confidence_median'::regproc)` contém `FROM function_orchestrator_role_items fori` no body; `pg_proc.proconfig` da função inclui `search_path=public, pg_temp`; GRANTs preservados (validar via `pg_get_function_grants` ou `\df+` no psql).

### §2.7 — (reservado para uso futuro)

---

## §3 TypeScript backend

### §3.1 — Refator da função de descoberta de skills + conversão do caller

**Atende:** S-ORCH-3.
**SUB-PR:** 3.
**Pré-requisito obrigatório:** evidence E0a da §6.1 — confirma nome real da função no codebase (`discoverAndLinkSkills` vs `safeDiscoverAndLinkSkills`) e sua estrutura atual.

**Arquivos tocados:**

- `lib/pipeline/ingest-job-and-discover-skills.ts` — função interna `discoverAndLinkSkills`
- `lib/pipeline/persist-curation/skill-mapper.ts` — wrapper `safeDiscoverAndLinkSkills`
- `lib/pipeline/persist-curation/persist-fn.ts` — caller (já em `await`; refator é apenas captura do retorno expandido)
- `lib/pipeline/ingest-job-and-discover-skills.ts` — tipo `RawSkill` ampliado com `needsReview?: boolean`

#### §3.1.0 — Estado atual: Cenário B confirmado em produção + caller já em `await`

**J-25 fix v2.5 — Cenário A removido:** versões anteriores desta spec contemplavam 2 cenários possíveis (Cenário A: função única retorna `result`; Cenário B: wrapper externo void). Ground truth via E0a + Claude Code grep confirma: o codebase real está em **Cenário B** (`safeDiscoverAndLinkSkills` é wrapper externo de `discoverAndLinkSkills` interna). Cenário A descontinuado nesta spec; §3.1.3 abaixo trata apenas Cenário B.

**Refator desta sprint:** EXPANDE retorno em AMBAS as camadas: função interna `discoverAndLinkSkills` devolve `DiscoverResultDetailed`; wrapper `safeDiscoverAndLinkSkills` passa a retornar o mesmo. Caller em `lib/pipeline/persist-curation/persist-fn.ts:410` JÁ ESTÁ EM `await safeDiscoverAndLinkSkills(supabase, normalized);` (confirmado por E0a) — esta sprint NÃO converte caller de fire-and-forget para `await`; refator é exclusivamente: (a) expandir tipo de retorno do wrapper de `Promise<void>` para `Promise<DiscoverResultDetailed>`; (b) usar o valor retornado no caller para empurrar para `insertFOSkillItemsBatch`.

**Para fins desta spec, daqui em diante uso "função de descoberta" quando o nome real não importa, e cito ambos os nomes (`discoverAndLinkSkills` / `safeDiscoverAndLinkSkills`) quando o nome importa.**

**I-2 fix — Deliverable mandatório pré-§3.1.3 (`classifyError` helper):** ground truth via Claude Code grep confirma que `classifyError` NÃO existe; mas `ErrorType` JÁ EXISTE em `lib/pipeline/types.ts:174-185` com **10 valores** (`'llm_timeout' | 'llm_rate_limit' | 'llm_overloaded' | 'llm_billing' | 'llm_auth' | 'batch_output_parse_error' | 'item_output_parse_error' | 'payload_parse_error' | 'parser_bug' | 'db_error'`), e `lib/pipeline/batch-processor/error-handlers.ts:8` já faz `import type { ErrorType } from '../types';`.

**J-3 fix v2.5 — Estender (não redefinir) ErrorType:** o deliverable desta sprint é (a) adicionar `'rpc_error'` e `'unknown'` ao type existente em `lib/pipeline/types.ts` (12 valores totais); (b) criar `classifyError` em `lib/pipeline/batch-processor/error-handlers.ts` que mapeia para a união completa:

```typescript
// PASSO 1 — lib/pipeline/types.ts (arquivo pré-existente; ampliar export):
export type ErrorType =
  | 'llm_timeout'
  | 'llm_rate_limit'           // pré-existente
  | 'llm_overloaded'           // pré-existente
  | 'llm_billing'              // pré-existente
  | 'llm_auth'                 // pré-existente
  | 'batch_output_parse_error'
  | 'item_output_parse_error'
  | 'payload_parse_error'
  | 'parser_bug'
  | 'db_error'
  | 'rpc_error'                // J-3 fix v2.5: NOVO desta sprint
  | 'unknown';                 // J-3 fix v2.5: NOVO desta sprint (paritary fallback)

// PASSO 2 — lib/pipeline/batch-processor/error-handlers.ts (arquivo pré-existente; adicionar export desta função):
import type { ErrorType } from '../types';

/**
 * Mapeia err.code / err.message para um ErrorType padronizado.
 * Compartilhada entre handleItemError (role) e handleSkillItemError (skill) — D-PS paridade.
 * [VALIDAR EM E0d: confirmar que classifyError não existe pré-deploy; validar ordering das condições contra padrões reais de erros do código]
 */
export function classifyError(err: unknown): ErrorType {
  const msg = err instanceof Error ? err.message : String(err);
  const code = (err as { code?: string })?.code;

  // Erros LLM específicos antes de fallbacks genéricos (ordering importa)
  if (msg.includes('429') || msg.toLowerCase().includes('rate limit')) return 'llm_rate_limit';
  if (msg.includes('529') || msg.toLowerCase().includes('overloaded')) return 'llm_overloaded';
  if (msg.includes('402') || msg.toLowerCase().includes('billing')) return 'llm_billing';
  if (msg.includes('401') || msg.includes('403') || msg.toLowerCase().includes('auth')) return 'llm_auth';
  if (msg.includes('timeout') || code === 'ETIMEDOUT') return 'llm_timeout';

  // Erros de parser
  if (msg.includes('batch') && msg.includes('parse')) return 'batch_output_parse_error';
  if (msg.includes('item') && msg.includes('parse')) return 'item_output_parse_error';
  if (msg.includes('payload') && msg.includes('parse')) return 'payload_parse_error';
  if (msg.includes('Unexpected') || msg.includes('TypeError')) return 'parser_bug';

  // Erros DB / RPC
  if (code?.startsWith('PG') || msg.includes('relation') || msg.includes('column')) return 'db_error';
  if (msg.includes('RPC') || msg.includes('function')) return 'rpc_error';

  return 'unknown';
}
```

Critério: TSC limpo após este deliverable. Code blocks de §3.1.3 importam `classifyError` de `@/lib/pipeline/batch-processor/error-handlers` sem necessidade de mudanças adicionais.

#### §3.1.1 — Expandir tipo de retorno (Mudança 1)

Exportar tipos da função de descoberta:

```typescript
// Tipos novos em lib/pipeline/persist-curation/types.ts

export type SkillProcessingDetail = {
  raw_item_index: number;            // posição da skill no batch da vaga (idempotência via UNIQUE)
  skill_raw_name: string;
  skill_raw_type: 'technical' | 'behavioral' | 'hybrid';
  needs_review: boolean;             // propagado de skill-type-guard.ts (D-PS-65/D-PS-69 cleanup v3.4)
  canonical_skill_id: string | null;
  canonical_skill_proposed: string | null;
  canonical_status: 'active' | 'pending' | null;
  skill_confidence: number | null;   // validado contra [0,1] antes do INSERT
  similarity_score: number | null;
  reasoning: string | null;
  pipeline_stage:
    | 'slug_match'
    | 'alias_match'
    | 'llm_new'
    | 'race_recovered'
    | 'gate_rejected'
    | 'fallback_error';
  status:
    // G-13: 'success' permanece no enum (paridade com DB FOSI §2.1 e FORI). Nenhum path emite hoje,
    // mas type espelha enum DB para compatibilidade com paths futuros sem ALTER TYPE.
    | 'success'
    | 'reused'
    | 'created_pending'
    | 'gate_rejected'
    | 'failed';
  // J-23 fix v2.5: error_type referencia type ErrorType compartilhado (J-3 fix em lib/pipeline/types.ts),
  // não literal-only duplicado. DB coluna é text sem CHECK enforcing — runtime constraint é TS strict.
  // Mantém 12 valores post-J-3 extension (M-3 fix v2.8 — qualifier explícito: 10 pré-existentes em lib/pipeline/types.ts:174-184 + 'rpc_error' + 'unknown' adicionados pelo deliverable J-3 §3.1.0 desta sprint).
  error_type: ErrorType | null;
  error_detail: string | null;
  dict_match: boolean;
  dict_match_source?: { source: string; matched_term: string };
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

**G-4 fix — rename completo do tipo de retorno:** ground truth via Glob (`lib/pipeline/ingest-job-and-discover-skills.ts:59-63`) confirma que `discoverAndLinkSkills` retorna hoje `DiscoverResult` com counters `{skills_discovered, skills_reused, skills_pending_created, skills_blocked}`. Esta sprint promove para `DiscoverResultDetailed` com counters renomeados paritários aos `pipeline_stage` reais: `{extracted, reused, pending_created, gate_rejected, failed}`. **Rename completo**, não delta — `DiscoverResult` é descontinuado.

**Auditoria obrigatória pré-SUB-PR 3 (E0d ampliado):** rodar `grep -rn "DiscoverResult[^D]" lib/` para identificar todos os consumers do tipo antigo. Resultado documentado em `AUDIT-orchestrator-symmetry-v2.8-<data>.md`. Cada consumer atualizado no mesmo SUB-PR (não em sprint futura — quebraria TSC se separado). Mapeamento de campos:

| `DiscoverResult` antigo | `DiscoverResultDetailed.aggregate` novo |
|---|---|
| `skills_discovered` | `extracted` |
| `skills_reused` | `reused` |
| `skills_pending_created` | `pending_created` |
| `skills_blocked` | `gate_rejected` |
| (não existia) | `failed` (novo — paritary ao `pipeline_stage: 'fallback_error'`) |

Tipo `RawSkill` em `lib/pipeline/ingest-job-and-discover-skills.ts` ampliado para incluir `needsReview?: boolean` (propagação D-PS-69 da cleanup v3.4 end-to-end). NÃO mover para `types.ts` — declaração permanece onde o código real define hoje, evitando refator de callers:

```typescript
interface RawSkill {
  label: string;
  skill_type: SkillType;  // ground truth — código real usa skill_type, não kind
  confidence: number;
  needsReview?: boolean;  // NOVO — propagado de skill-type-guard.ts
}
```

#### §3.1.2 — Instrumentar paths internos da função de descoberta (Mudança 2)

Cada path interno passa a empurrar um `SkillProcessingDetail` no array `items`. O sinal `needs_review` vem do skill-type-guard pré-existente (cleanup v3.4): quando o LLM emite alias PT-BR ou variante de caixa, o guard normaliza E marca `needsReview=true` em `LLMSkillItem.needsReview`. Esse sinal **propaga end-to-end** até `function_orchestrator_skill_items.needs_review` por meio do tipo `RawSkill`.

**Path `slug_match`** (lookup em `job_canonical_skills.slug` retornou hit):
- `status='reused'`, `pipeline_stage='slug_match'`, `canonical_skill_id` lido do registro encontrado, `canonical_status` lido do registro, `dict_match=false`, `needs_review` copiado do input

**Path `alias_match`** (RPC `lookup_canonical_skill_by_normalized_alias` retornou hit):
- `status='reused'`, `pipeline_stage='alias_match'`, `dict_match=true`, `dict_match_source={ source: 'taxonomy_relations', matched_term: <normalized> }`, `needs_review` copiado do input

**Path `llm_new`** (INSERT bem-sucedido com `status='pending'`):
- `status='created_pending'`, `pipeline_stage='llm_new'`, `canonical_status='pending'`, `dict_match=false`, `needs_review` copiado do input

**Path `race_recovered`** (INSERT falhou por duplicação concorrente, RPC `resolve_active_canonical_by_slug` recuperou):
- `status='reused'`, `pipeline_stage='race_recovered'`, `dict_match=false`, `needs_review` copiado do input

**Path `fallback_error`** (catch externo capturou exceção catastrófica):
- `status='failed'`, `pipeline_stage='fallback_error'`, `error_type` mapeado via `classifyError(err)`, `error_detail=err.message`, `needs_review=false`

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

**Nota sobre `gate_rejected`:** o hard gate de skill (`skill.hard_gate.min_confidence`) é aplicado **antes** da função de descoberta ser chamada, em `persist-curation.ts` (cleanup v3.4). Skills que não passam no gate **não chegam** nesta função. Portanto, o caller também precisa registrar essas rejeições — ver Mudança 3 abaixo. Itens com `status='gate_rejected'` em `function_orchestrator_skill_items` são inseridos pelo caller, não pela função de descoberta.

**Validação de `skill_confidence`:** garantida pelo `CHECK chk_foski_skill_confidence_range` em §2.1 (`CHECK (skill_confidence IS NULL OR (skill_confidence >= 0 AND skill_confidence <= 1))`). DB rejeita valores fora da faixa no INSERT; não há helper TS adicional (`validateSkillConfidence` descartado por YAGNI — defesa em profundidade redundante).

#### §3.1.3 — Caller: capturar o retorno expandido (Mudança 3 — caller já está em `await`)

Estado atual confirmado por E0a: caller em `persist-fn.ts:410` já tem `await safeDiscoverAndLinkSkills(supabase, normalized);` — retorno descartado porque a função era `Promise<void>`. O refator desta sprint é exclusivamente:
- (a) expandir tipo de retorno do wrapper de `Promise<void>` para `Promise<DiscoverResultDetailed>`
- (b) usar o valor retornado no caller para empurrar para `insertFOSkillItemsBatch`

**Refator em 2 camadas** (J-25 fix v2.5: Cenário A descontinuado; produção confirmadamente está em Cenário B):

1. **Função interna `discoverAndLinkSkills`** continua sendo responsável pela lógica; agora retorna `DiscoverResultDetailed` (Mudanças 1 + 2 acima).
2. **Wrapper `safeDiscoverAndLinkSkills`** deixa de ser `Promise<void>` e passa a propagar o `DiscoverResultDetailed` da função interna:

```typescript
// ANTES: assinatura void
// export async function safeDiscoverAndLinkSkills(
//   supabase: SupabaseClient,
//   normalized: NormalizedJobPosting,
// ): Promise<void> { ... }

// DEPOIS: assinatura tipada com DiscoverResultDetailed
export async function safeDiscoverAndLinkSkills(
  supabase: SupabaseClient,
  normalized: NormalizedCurationResult,
): Promise<DiscoverResultDetailed> {
  // I-10 fix: empty-array guard cosmético paritary a skill-mapper.ts:41 — short-circuit antes de
  // qualquer mapping/RPC. discoverAndLinkSkills tem guard interno equivalente, mas preservar aqui
  // mantém paridade arquitetônica e evita 1 alocação de array vazio + 1 chamada noop.
  if (normalized.skillItems.length === 0) {
    return {
      aggregate: { extracted: 0, reused: 0, pending_created: 0, gate_rejected: 0, failed: 0 },
      items: [],
    };
  }

  // F-9 fix: preservar mapping LLMSkillItem → RawSkill que existe no skill-mapper.ts:43-54.
  // discoverAndLinkSkills espera RawSkill[] (label/skill_type), não LLMSkillItem[] (name/type).
  // J-26 fix v2.5: mapping explícito mantido — código real do skill-mapper.ts:43-54 usa spread `{...rest}`,
  // mas explicit mapping aqui facilita rastreabilidade (item.name → label; item.type → skill_type)
  // e evita propagar campos extras do LLMSkillItem se estrutura mudar futuramente. Drift estilístico aceitável.
  const rawSkills: RawSkill[] = normalized.skillItems.map(item => ({
    label: item.name,
    skill_type: item.type,
    confidence: item.confidence,
    needsReview: item.needsReview,
  }));

  try {
    return await discoverAndLinkSkills(supabase, normalized.id, rawSkills);
  } catch (err) {
    console.error('[safeDiscoverAndLinkSkills] failed (non-blocking):', err);
    // F-4 fix: insertEvent assinatura single-object (ground truth: lib/events/insert.ts:48-51).
    // Pattern paritary a skill-mapper.ts:22-29.
    await insertEvent(supabase, {
      event_name: 'persist_curation.skill_map_failed',
      resource_type: 'job_posting',
      resource_id: normalized.id,
      actor: 'system',  // J-8 fix v2.5: paritary a skill-mapper.ts:26+60 (cleanup v3.4 usa 'system')
      actor_id: null,
      metadata: {
        error: err instanceof Error ? err.message : String(err),
      },
    });
    return {
      aggregate: { extracted: 0, reused: 0, pending_created: 0, gate_rejected: 0, failed: rawSkills.length },
      items: rawSkills.map((s, idx) => ({
        raw_item_index: idx,
        skill_raw_name: s.label,
        skill_raw_type: s.skill_type,
        canonical_skill_id: null,
        canonical_skill_proposed: null,
        canonical_status: null,
        pipeline_stage: 'fallback_error' as const,
        skill_confidence: s.confidence,
        similarity_score: null,
        reasoning: null,
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

**F-10 fix — `gate_rejected` items via `DiscoverResultDetailed.items`:** o hard gate de skills vive dentro de `discoverAndLinkSkills` (`ingest-job-and-discover-skills.ts:81-100`) — ele emite evento + faz `continue` no loop, sem expor a lista de rejeitados ao caller. Em vez de hoist do gate (refator amplo), `discoverAndLinkSkills` passa a emitir items com `pipeline_stage: 'gate_rejected'` + `status: 'failed'` em `DiscoverResultDetailed.items` para cada skill barrada. Caller `persist-fn.ts` lê esses items normalmente em `skillDetail.items` (sem variável separada `skillsRejectedByGate`).

E em `lib/pipeline/persist-curation/persist-fn.ts:410`:

```typescript
// ANTES (caller já em await mas retorno descartado):
await safeDiscoverAndLinkSkills(supabase, normalized);

// DEPOIS (caller captura retorno; gate_rejected items vêm dentro de skillDetail.items — F-10 fix):
const skillDetail = await safeDiscoverAndLinkSkills(supabase, normalized);

// runId vem de PersistOptions (ampliado nesta sprint — ver abaixo); jobPostingId vem de normalized.id
if (options.runId && skillDetail.items.length > 0) {
  await insertFOSkillItemsBatch(supabase, options.runId, normalized.id, skillDetail.items);
}
```

**Ampliação de `PersistOptions`** em `lib/pipeline/persist-curation/types.ts` (preserva retrocompatibilidade — campo opcional):

```typescript
export interface PersistOptions {
  // ... campos pré-existentes preservados ...
  runId?: string;  // NOVO — threading do run_id do orchestrator para insertFOSkillItemsBatch
}
```

**Callers de `persistCuratedJob`** (I-11 fix — **4 sites** identificados pela auditoria de SUB-PR 1: `curate-flow-b`, `process-item`, `persist-results` + 1 site em `tests/`) passam a passar `runId` em `options` quando vier de um contexto com orchestrator ativo:

```typescript
// Caller A — fluxo A/B/C com orchestrator
await persistCuratedJob(supabase, normalized, { runId: ctx.runId, /* ... */ }, counters);

// Caller B — sem orchestrator (legado quarantined ou manual)
await persistCuratedJob(supabase, normalized, { /* sem runId */ }, counters);
```

Quando `options.runId` está `undefined`, `insertFOSkillItemsBatch` é pulado silenciosamente (sem warning) — caminho legado preservado, paritário ao comportamento atual onde FOSI não existia.

`try/catch` no caller é REDUNDANTE (o wrapper já trata), mas pode ser mantido como defesa em profundidade.

**Decisão de spec:** Cenário B preserva 2 camadas (encapsulamento de erro fica no wrapper, lógica fica na função interna). É o cenário cravado pelo código real do CalibraCV (E0a confirmou).

#### §3.1.4 — Helpers a validar antes do PR

- `classifyError(err)` — provável função similar em `lib/pipeline/batch-processor/error-handlers.ts`. Se não existir, criar mapeamento simples baseado em `err.code` e `err.message`. Compartilhada entre `handleItemError` (role) e `handleSkillItemError` (skill).
- `discoverAndLinkSkills` precisa emitir items com `pipeline_stage: 'gate_rejected'` + `status: 'failed'` em `DiscoverResultDetailed.items` para cada skill barrada pelo hard gate (ver F-10 fix em §3.1.3). Hoje o gate vive em `ingest-job-and-discover-skills.ts:81-100` e faz `continue` no loop sem expor a lista de rejeitados — implementor amplia o retorno para incluir esses items.
- Como `RawSkill` é declarado em `lib/pipeline/ingest-job-and-discover-skills.ts` (interface local, NÃO em `types.ts`) — precisa incluir `needsReview?: boolean` para propagar D-PS-69 da cleanup v3.4. Se ainda não inclui, ampliar declaração no local atual.

**Critério de aceite (§7.2):** TSC limpo; `discoverAndLinkSkills` retorna array com 1 entry por skill processada em cada path (incluindo `gate_rejected`); caller invoca `insertFOSkillItemsBatch` quando `skillDetail.items.length > 0`; smoke S15 (propagação `needs_review`) verde.

### §3.2 — `insertFOSkillItem` em call sites (individual + bulk + `handleSkillItemError`)

**SUB-PR:** 3.

**Arquivo novo:** `lib/pipeline/batch-processor/insert-fo-skill-item.ts`

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';
// J-24 fix v2.5: BatchContext NÃO importado — E-6 fix passou SupabaseClient + runId diretos como params.
// SkillProcessingDetail é definido em §3.1.1 em lib/pipeline/persist-curation/types.ts (pasta irmã).
import type { SkillProcessingDetail } from '../persist-curation/types';

// Path individual — call sites com 1 item por vez (rarissimamente usado para skill;
// preservado por paridade com insertFOItem role-side)
// E-6 fix paritário: SupabaseClient + runId diretos.
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
      raw_item_index: detail.raw_item_index,
      skill_raw_name: detail.skill_raw_name,
      skill_raw_type: detail.skill_raw_type,
      needs_review: detail.needs_review,
      canonical_skill_id: detail.canonical_skill_id,
      canonical_skill_proposed: detail.canonical_skill_proposed,
      canonical_status: detail.canonical_status,
      skill_confidence: detail.skill_confidence,
      similarity_score: detail.similarity_score,
      reasoning: detail.reasoning,
      pipeline_stage: detail.pipeline_stage,
      status: detail.status,
      error_type: detail.error_type,
      error_detail: detail.error_detail,
      processed_at: new Date().toISOString(),
    });
  if (error) {
    console.warn('[batch] insertFOSkillItem failed:', error.message);
  }
}

// Path bulk — caminho preferencial para skill dada cardinalidade real (10-30 skills/job)
// Assimetria operacional documentada em D-PS desta sprint
// E-6 fix: assinatura aceita SupabaseClient direto. runId já threaded via PersistOptions (CRITICAL-3);
// BatchContext era redundante neste caller (persistCuratedJob não tem ctx no escopo).
// Callers role-side (process-item.ts) que JÁ TÊM BatchContext passam ctx.supabase, ctx.runId — sem breaking change.
export async function insertFOSkillItemsBatch(
  supabase: SupabaseClient,
  runId: string,
  jobPostingId: string,
  details: SkillProcessingDetail[],
): Promise<void> {
  if (details.length === 0) return;
  const rows = details.map(d => ({
    run_id: runId,
    job_posting_id: jobPostingId,
    raw_item_index: d.raw_item_index,
    skill_raw_name: d.skill_raw_name,
    skill_raw_type: d.skill_raw_type,
    needs_review: d.needs_review,
    canonical_skill_id: d.canonical_skill_id,
    canonical_skill_proposed: d.canonical_skill_proposed,
    canonical_status: d.canonical_status,
    skill_confidence: d.skill_confidence,
    similarity_score: d.similarity_score,
    reasoning: d.reasoning,
    pipeline_stage: d.pipeline_stage,
    status: d.status,
    error_type: d.error_type,
    error_detail: d.error_detail,
    processed_at: new Date().toISOString(),
  }));
  const { error } = await supabase
    .from('function_orchestrator_skill_items')
    .insert(rows);
  if (error) {
    console.warn('[batch] insertFOSkillItemsBatch failed:', error.message);
  }
}
```

**Atualização paritária retroativa em `lib/pipeline/batch-processor/process-item.ts`** — `insertFOItem` (lado role) ganha leitura de `.error` + log estruturado paritário:

```typescript
async function insertFOItem(
  ctx: BatchContext, result: LLMItemResult,
  canonicalRoleId: string | null, status: string,
  pipelineStage: string, rawItemIndex: number,  // NOVO: índice obrigatório
  canonicalStatus?: string,
): Promise<void> {
  const validatedConfidence = validateCanonicalConfidence(result.canonical_confidence);
  const { error } = await ctx.supabase
    .from('function_orchestrator_role_items')  // NOME NOVO pós-rename §2.6
    .insert({
      run_id: ctx.runId,
      job_posting_id: result.id,
      raw_item_index: rawItemIndex,  // NOVO: passado pelo caller
      title_original: result.canonical_role,
      canonical_role_proposed: result.canonical_role,
      secondary_role_proposed: result.secondary_role || null,
      canonical_role_id: canonicalRoleId,
      canonical_status: canonicalStatus || null,
      skills_count: result.skills?.length ?? 0,  // F-5 fix: campo real em LLMItemResult (lib/pipeline/types.ts:37-53) é `skills`, não `skillItems`. skillItems existe em NormalizedCurationResult (CRITICAL-1); são tipos distintos
      confidence: validatedConfidence,
      reasoning: result.reasoning || null,
      pipeline_stage: pipelineStage,
      status,
      processed_at: new Date().toISOString(),
    });
  if (error) {
    console.warn('[batch] insertFOItem failed:', error.message);  // NOVO: paridade COM bug fix
  }
}
```

Caller de `insertFOItem` em `process-item.ts:199 + 206` atualizado para passar `rawItemIndex` (índice do item no batch processado).

**Helper `handleSkillItemError`** em `lib/pipeline/batch-processor/error-handlers.ts` (paritário ao `handleItemError` pré-existente). **E-6 fix:** assinatura aceita `SupabaseClient` + `runId` direto; callers role-side com `BatchContext` passam `ctx.supabase, ctx.runId`; callers sem `ctx` (path do `persistCuratedJob`) passam `supabase, options.runId`:

```typescript
export async function handleSkillItemError(
  supabase: SupabaseClient,
  runId: string,
  jobPostingId: string,
  skillRawName: string,
  skillRawKind: SkillType,  // I-3 fix: tipo unificado exportado de lib/pipeline/types.ts (ingest-job-and-discover-skills.ts apenas re-importa)
  rawItemIndex: number,
  errorType: string,
  errorDetail: string,
): Promise<void> {
  const { error: insertError } = await supabase
    .from('function_orchestrator_skill_items')
    .insert({
      run_id: runId,
      job_posting_id: jobPostingId,
      raw_item_index: rawItemIndex,
      skill_raw_name: skillRawName,
      skill_raw_type: skillRawKind,
      needs_review: false,
      canonical_skill_id: null,
      canonical_status: null,
      pipeline_stage: 'fallback_error',
      status: 'failed',
      error_type: errorType,
      error_detail: errorDetail.slice(0, 500),
      processed_at: new Date().toISOString(),
    });
  if (insertError) {
    console.warn('[batch] handleSkillItemError orchestrator INSERT failed:', insertError.message);
  }
}
```

**Cobertura paritária de callers (auditoria E0h do §6.1):**

Estado atual antes desta sprint: `handleItemError` (role) é invocado em **N call sites** do pipeline de role (lista exata derivada de E0h: grep `handleItemError` em `lib/pipeline/`). Versão skill cravada nesta sprint inicialmente cobre apenas 1 call site (catch upstream do `safeDiscoverAndLinkSkills` em §3.1.3). Para paridade real, SUB-PR 3 instala invocações de `handleSkillItemError` em **todos os call sites equivalentes** do pipeline de skill identificados via E0h.

Mapeamento paritário esperado (validar pela auditoria — pode haver call sites role sem equivalente skill por construção arquitetural):

| Call site role | Call site skill paritário esperado |
|---|---|
| `lib/pipeline/batch-processor/process-item.ts` catch principal | catch upstream do path de skill (já cravado em §3.1.3) |
| `lib/pipeline/batch-processor/error-handlers.ts` retry final | retry final no path de skill (instalar paritário) |
| `lib/pipeline/persist-curation/persist-fn.ts` catch de orchestrator | catch paritário no `safeDiscoverAndLinkSkills` |
| Outros call sites pré-existentes de `handleItemError` (via E0h) | instalar `handleSkillItemError` no equivalente skill |

**Justificativa de paridade necessária:** sem cobertura paritária de callers, falhas no pipeline de skill em pontos fora do catch upstream **não geram row de erro em FOSI** — admin não consegue diagnosticar via dashboard. Cobertura única no catch upstream cobre apenas falhas catastróficas de discovery, não falhas de instrumentação granular em outros pontos.

**Critério de aceite (§7.2):** TSC limpo; smoke `insertFOSkillItemsBatch` com 10 rows válidos popula 10 entries em FOSI com `raw_item_index` distintos; smoke de duplicação (mesma combinação `run_id+job_posting_id+raw_item_index`) retorna erro de UNIQUE constraint sem corromper estado; smoke S24 (auditoria de cobertura paritária `handleSkillItemError` vs `handleItemError`) verde — invocações em call sites equivalentes existem em ambos pipelines.

### §3.3 — Atualizar acumuladores nos 3 finalizers via RPC SQL compartilhada

**SUB-PR:** 4.

**3 finalizers a atualizar** (identificados via auditoria pré-existente da cleanup v3.4):

1. `lib/pipeline/curate-job-postings.ts:311-337` — `finalizeFORun()`
2. `lib/analysis/curate-flow-b.ts:157-178` — finalizer inline `finalizeEmptyRun()`
3. `app/api/admin/curate-job-postings/route.ts:107-127` — finalizer HTTP per-chunk

Cada um, após acumular contadores role pré-existentes, invoca a RPC `count_skill_items_by_status(runId)` e atualiza as 5 colunas skill na MESMA UPDATE atômica (sem writes separados que poderiam gerar torn write):

```typescript
// Padrão paritário aplicado nos 3 finalizers
const { data: skillCounts, error: rpcError } = await supabase
  .rpc('count_skill_items_by_status', { p_run_id: runId });

if (rpcError) {
  console.warn('[batch] count_skill_items_by_status RPC failed:', rpcError.message);
  // Política: continuar com NULL nos 5 campos skill_*, role finalization é source-of-truth
  // (não rollback do run inteiro só porque contadores skill falharam)
}

// Mapeamento status → coluna em FOR
const skillFields = {
  skills_extracted: 0,
  skills_reused: 0,
  skills_pending_created: 0,
  skills_gate_rejected: 0,
  skills_failed: 0,
};

(skillCounts ?? []).forEach((row: { status: string; c: number }) => {
  switch (row.status) {
    // F-11 fix: 'success' não é emitido por nenhum path (§3.1.2 path table não mapeia para esse valor); dropped.
    case 'reused':
      skillFields.skills_reused += row.c;
      break;
    case 'created_pending':
      skillFields.skills_pending_created += row.c;
      break;
    case 'gate_rejected':
      skillFields.skills_gate_rejected += row.c;
      break;
    case 'failed':
      skillFields.skills_failed += row.c;
      break;
  }
});
skillFields.skills_extracted = Object.values(skillFields).reduce((a, b) => a + b, 0);

// UPDATE atômica com fields role pré-existentes + 5 fields skill NOVOS
const { error: updateError } = await supabase
  .from('function_orchestrator_runs')
  .update({
    ...rolePreExistingFields,  // 8 contadores outcome-oriented role pré-existentes
    ...skillFields,            // 5 contadores stage-oriented skill NOVOS
    finished_at: new Date().toISOString(),
    status: 'success',
  })
  .eq('id', runId);

if (updateError) {
  console.warn('[batch] finalizer UPDATE failed:', updateError.message);
}
```

**Decisão de design:** RPC compartilhada nos 3 finalizers (não 3 forks). UPDATE atômica com fields role+skill no mesmo statement (evita torn write). Falha do RPC skill = continuar com NULL nos 5 campos + log estruturado, NÃO rollback (role finalization é source-of-truth para status). Finalizer 3 (read-modify-write) ganha idempotência grátis porque FOSI é append-only e o RPC retorna absoluto.

**Sobre `fetch-chunk.ts:91`** (write site adicional em FOR identificado pela auditoria): é status flip `queued→running`, mecânica de orquestração — NÃO finalizer. Fora do escopo desta sprint.

**Critério de aceite (§7.2):** smoke de run completo (Flow A admin) com 5 skills processadas (3 reused, 1 pending, 1 failed) atualiza FOR com `skills_extracted=5, skills_reused=3, skills_pending_created=1, skills_failed=1, skills_gate_rejected=0` em UPDATE única validada via `BEGIN; ... ROLLBACK;` (sem efeito permanente em pré-prod).

### §3.4 — Atualização de call sites pós-rename

**SUB-PR:** 2.

Grep global por `function_orchestrator_items` em `lib/`, `app/`, `scripts/`, `tests/` — substituir por `function_orchestrator_role_items` em referências de tabela. Listagem específica via auditoria §6.2.

**Sites identificados antecipadamente** (lista preliminar; auditoria §6.2 confirma cobertura):

- `lib/pipeline/batch-processor/process-item.ts:199 + 206` — INSERT em `function_orchestrator_items`
- `lib/pipeline/batch-processor/error-handlers.ts` — INSERT em `handleItemError` + `markBatchAsRetryable`
- `lib/pipeline/curate-job-postings.ts` — SELECT/finalizer
- `lib/analysis/curate-flow-b.ts` — finalizer
- `app/api/admin/curate-job-postings/route.ts` — finalizer
- `lib/analysis/insert-jobs.ts` — `createOrchestratorRun()` (criação de run, não tabela items)
- `lib/analysis/fetch-chunk.ts:91` — UPDATE status
- Testes em `tests/` — fixtures e expects

**Critério de aceite (§7.2):** TSC limpo; grep `function_orchestrator_items` retorna 0 hits em arquivos não-migration; testes verdes.

### §3.5 — RPCs SQL paritárias para queries Limiares e aggregator

**SUB-PR:** 6.

Esta sprint elimina o uso de `pg.Pool` direto em `lib/admin/limiares/queries.ts` substituindo as queries por RPCs SQL chamadas via Supabase client. Convergência arquitetônica: 100% dos endpoints admin passam a usar Supabase client para acesso ao DB.

**12 RPCs novas em SQL** — todas `LANGUAGE sql STABLE` com `SET search_path TO 'public', 'pg_temp'`, sem `SECURITY DEFINER`, sem GRANTs específicos além do default:

#### §3.5.1 — `count_role_items_by_stage_in_window(p_start, p_end)`

```sql
CREATE OR REPLACE FUNCTION count_role_items_by_stage_in_window(
  p_start timestamptz,
  p_end timestamptz
)
RETURNS TABLE(stage text, c bigint)
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT fori.pipeline_stage::text AS stage, COUNT(*) AS c
  FROM function_orchestrator_role_items fori
  JOIN function_orchestrator_runs r ON r.id = fori.run_id
  WHERE r.started_at >= p_start
    AND r.started_at < p_end  -- F-13 fix: half-open interval — p_end deve ser dayStart do dia seguinte (T00:00:00 do D+1)
  GROUP BY fori.pipeline_stage;
$$;
```

#### §3.5.2 — `count_skill_items_by_stage_in_window(p_start, p_end)`

```sql
CREATE OR REPLACE FUNCTION count_skill_items_by_stage_in_window(
  p_start timestamptz,
  p_end timestamptz
)
RETURNS TABLE(stage text, c bigint)
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT fosi.pipeline_stage::text AS stage, COUNT(*) AS c
  FROM function_orchestrator_skill_items fosi
  JOIN function_orchestrator_runs r ON r.id = fosi.run_id
  WHERE r.started_at >= p_start
    AND r.started_at < p_end  -- F-13 fix: half-open interval — p_end deve ser dayStart do dia seguinte (T00:00:00 do D+1)
  GROUP BY fosi.pipeline_stage;
$$;
```

#### §3.5.3 — `limiares_panel_1_snapshot(p_interval)` — Hard Gate

Conta eventos `*_creation_blocked_*` (entregues pela cleanup v3.4 F22) por dia + 3 séries paralelas: role (eventos), skill (eventos), skill FOSI (`status='gate_rejected'`).

```sql
CREATE OR REPLACE FUNCTION limiares_panel_1_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  -- Outer SELECT FROM role_events retorna 0 rows quando role_events vazio.
  -- Resolvido encapsulando cada série em CTE com SELECT escalar (sempre 1 row) e juntando
  -- via cross-join trivial garantindo 1 row final ainda que as 3 séries estejam vazias.
  WITH role_events AS (
    SELECT COALESCE(jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day), '[]'::jsonb) AS payload
    FROM (
      SELECT date_trunc('day', created_at) AS day, COUNT(*) AS cnt
      FROM events
      WHERE event_name IN (
        'role_creation_blocked_missing_confidence',
        'role_creation_blocked_low_confidence',
        'role_creation_blocked_invalid_slug'
      )
        AND created_at >= NOW() - p_interval::INTERVAL
      GROUP BY 1
    ) sub
  ),
  skill_events AS (
    SELECT COALESCE(jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day), '[]'::jsonb) AS payload
    FROM (
      SELECT date_trunc('day', created_at) AS day, COUNT(*) AS cnt
      FROM events
      WHERE event_name IN (
        'skill_creation_blocked_missing_confidence',
        'skill_creation_blocked_low_confidence',
        'skill_creation_blocked_invalid_slug'
      )
        AND created_at >= NOW() - p_interval::INTERVAL
      GROUP BY 1
    ) sub
  ),
  skill_fosi AS (
    -- JOIN com runs para paridade temporal com demais painéis
    -- (todos usam function_orchestrator_runs.started_at como referência de janela)
    SELECT COALESCE(jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day), '[]'::jsonb) AS payload
    FROM (
      SELECT date_trunc('day', r.started_at) AS day, COUNT(*) AS cnt
      FROM function_orchestrator_skill_items fosi
      JOIN function_orchestrator_runs r ON r.id = fosi.run_id
      WHERE fosi.status = 'gate_rejected'
        AND r.started_at >= NOW() - p_interval::INTERVAL
      GROUP BY 1
    ) sub
  )
  SELECT jsonb_build_object(
    'series_role_events', re.payload,
    'series_skill_events', se.payload,
    'series_skill_fosi', sf.payload
  )
  FROM role_events re, skill_events se, skill_fosi sf;
$$;
```

#### §3.5.4 — `limiares_panel_2_snapshot(p_interval)` — Promoção vs Zona Opus

UNION ALL `job_canonical_roles` + `job_canonical_skills` agrupando por banda de `confidence_median` × `entity_type` (paritário pós-cleanup v3.4 F17).

```sql
CREATE OR REPLACE FUNCTION limiares_panel_2_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  WITH unified AS (
    -- Guard GREATEST(LEAST(..., 10), 1): clamp bucket no intervalo [1, 10] (paridade com §3.5.7/§3.5.10).
    -- Valores aberrantes de confidence_median (< 0.0 ou >= 1.0) clamped em vez de retornar bucket 0/11
    -- que crasharia consumer bucketEdges[b.bucket - 1] em §4.4.
    SELECT 'role' AS entity, GREATEST(LEAST(WIDTH_BUCKET(confidence_median, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
    FROM job_canonical_roles
    WHERE confidence_median IS NOT NULL
      AND status IN ('active', 'pending')
    GROUP BY 1, 2
    UNION ALL
    SELECT 'skill' AS entity, GREATEST(LEAST(WIDTH_BUCKET(confidence_median, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
    FROM job_canonical_skills
    WHERE confidence_median IS NOT NULL
      AND status IN ('active', 'pending')
    GROUP BY 1, 2
  )
  SELECT jsonb_build_object(
    'distribution_role', COALESCE((SELECT jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket) FROM unified WHERE entity='role'), '[]'::jsonb),
    'distribution_skill', COALESCE((SELECT jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket) FROM unified WHERE entity='skill'), '[]'::jsonb)
  );
$$;
```

#### §3.5.5 — `limiares_panel_3_snapshot(p_interval)` — Merge Candidates

Snapshot diário lista até 100 candidatos por entidade (`canonical_role_merge_candidates` + `canonical_skill_merge_candidates`).

```sql
CREATE OR REPLACE FUNCTION limiares_panel_3_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  -- Ground truth via §6.1: tabelas canonical_*_merge_candidates têm coluna detected_at (não created_at).
  -- Fix paritário em ambas queries (role + skill).
  WITH role_candidates AS (
    SELECT jsonb_agg(
      jsonb_build_object('id', id, 'similarity', similarity, 'detected_at', detected_at)
      ORDER BY similarity DESC, id ASC
    ) AS rows
    FROM (
      SELECT id, similarity, detected_at
      FROM canonical_role_merge_candidates
      WHERE detected_at >= NOW() - p_interval::INTERVAL
      ORDER BY similarity DESC, id ASC
      LIMIT 100
    ) sub
  ),
  skill_candidates AS (
    SELECT jsonb_agg(
      jsonb_build_object('id', id, 'similarity', similarity, 'detected_at', detected_at)
      ORDER BY similarity DESC, id ASC
    ) AS rows
    FROM (
      SELECT id, similarity, detected_at
      FROM canonical_skill_merge_candidates
      WHERE detected_at >= NOW() - p_interval::INTERVAL
      ORDER BY similarity DESC, id ASC
      LIMIT 100
    ) sub
  )
  SELECT jsonb_build_object(
    'candidates_role', COALESCE((SELECT rows FROM role_candidates), '[]'::jsonb),
    'candidates_skill', COALESCE((SELECT rows FROM skill_candidates), '[]'::jsonb)
  );
$$;
```

#### §3.5.6 — `limiares_panel_4_snapshot(p_interval)` — Gate cumulativo

UNION ALL filtrando pending com volume + `entity_type`.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_4_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT jsonb_build_object(
    'pending_role', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'vacancy_count', vacancy_count,
        'distinct_sources_count', distinct_sources_count, 'confidence_median', confidence_median
      ) ORDER BY vacancy_count DESC), '[]'::jsonb)
      FROM (
        SELECT id, label, vacancy_count, distinct_sources_count, confidence_median
        FROM job_canonical_roles
        WHERE status = 'pending' AND vacancy_count > 0
        ORDER BY vacancy_count DESC
        LIMIT 200
      ) sub
    ),
    'pending_skill', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'vacancy_count', vacancy_count,
        'distinct_sources_count', distinct_sources_count, 'confidence_median', confidence_median
      ) ORDER BY vacancy_count DESC), '[]'::jsonb)
      FROM (
        SELECT id, label, vacancy_count, distinct_sources_count, confidence_median
        FROM job_canonical_skills
        WHERE status = 'pending' AND vacancy_count > 0
        ORDER BY vacancy_count DESC
        LIMIT 200
      ) sub
    )
  );
$$;
```

#### §3.5.7 — `limiares_panel_5_snapshot(p_interval)` — Pending stuck >30d

UNION ALL listando até 50 itens × `entity_type`.

**Nota:** `days_stuck` calculado como `(NOW()::date - created_at::date)::int` (diferença total de dias entre datas). `EXTRACT(DAY FROM NOW() - created_at)::int` retornaria apenas o componente "day" do interval Postgres (interval `'1 month 2 days'` retornaria 2, não 32) — armadilha conhecida em Postgres ao trabalhar com diferença entre timestamps.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_5_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT jsonb_build_object(
    'stuck_role', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'created_at', created_at,
        'days_stuck', (NOW()::date - created_at::date)::int
      ) ORDER BY created_at), '[]'::jsonb)
      FROM (
        SELECT id, label, created_at
        FROM job_canonical_roles
        WHERE status = 'pending' AND created_at < NOW() - INTERVAL '30 days'
        ORDER BY created_at
        LIMIT 50
      ) sub
    ),
    'stuck_skill', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'created_at', created_at,
        'days_stuck', (NOW()::date - created_at::date)::int
      ) ORDER BY created_at), '[]'::jsonb)
      FROM (
        SELECT id, label, created_at
        FROM job_canonical_skills
        WHERE status = 'pending' AND created_at < NOW() - INTERVAL '30 days'
        ORDER BY created_at
        LIMIT 50
      ) sub
    )
  );
$$;
```

#### §3.5.8 — `limiares_panel_6_snapshot(p_interval)` — Aposentadoria por gap

UNION ALL com `latest_posted_at < NOW() - gap_days × entity_type`.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_6_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  -- Ground truth via §6.1: pipeline_config tem role.retirement.gap_days=365 e skill.retirement.gap_days=365.
  -- Fallback alinhado paritariamente em 365 para refletir DB; cravado em D-PS-86.
  -- F-20 fix: ORDER BY + LIMIT em sub-SELECT para garantir "100 mais antigos" (não "100 arbitrários, então ordenados").
  WITH role_gap_days AS (
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.retirement.gap_days'), 365) AS d
  ),
  skill_gap_days AS (
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.retirement.gap_days'), 365) AS d
  )
  SELECT jsonb_build_object(
    'retire_role', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'latest_posted_at', latest_posted_at
      ) ORDER BY latest_posted_at), '[]'::jsonb)
      FROM (
        SELECT id, label, latest_posted_at
        FROM job_canonical_roles, role_gap_days
        WHERE status = 'active' AND latest_posted_at < NOW() - make_interval(days => role_gap_days.d)
        ORDER BY latest_posted_at ASC, id ASC
        LIMIT 100
      ) sub
    ),
    'retire_skill', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'id', id, 'label', label, 'latest_posted_at', latest_posted_at
      ) ORDER BY latest_posted_at), '[]'::jsonb)
      FROM (
        SELECT id, label, latest_posted_at
        FROM job_canonical_skills, skill_gap_days
        WHERE status = 'active' AND latest_posted_at < NOW() - make_interval(days => skill_gap_days.d)
        ORDER BY latest_posted_at ASC, id ASC
        LIMIT 100
      ) sub
    )
  );
$$;
```

#### §3.5.9 — `limiares_panel_7_snapshot(p_interval)` — `confidence_median_at_promotion`

UNION ALL agrupando por bucket `WIDTH_BUCKET × entity_type`.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_7_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT jsonb_build_object(
    'histogram_role', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket), '[]'::jsonb)
      FROM (
        SELECT GREATEST(LEAST(WIDTH_BUCKET(confidence_median_at_promotion, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
        FROM job_canonical_roles
        WHERE confidence_median_at_promotion IS NOT NULL
          AND promoted_at >= NOW() - p_interval::INTERVAL
        GROUP BY 1
      ) sub
    ),
    'histogram_skill', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket), '[]'::jsonb)
      FROM (
        SELECT GREATEST(LEAST(WIDTH_BUCKET(confidence_median_at_promotion, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
        FROM job_canonical_skills
        WHERE confidence_median_at_promotion IS NOT NULL
          AND promoted_at >= NOW() - p_interval::INTERVAL
        GROUP BY 1
      ) sub
    )
  );
$$;
```

#### §3.5.10 — `limiares_panel_8_snapshot(p_interval)` — `creation_confidence` distribuição

Histograma role + skill (UNION ALL pré-existente) + modo "Por path" via FOSI.

**Nota:** `WIDTH_BUCKET` envelopado por `GREATEST(LEAST(..., 10), 1)` para garantir que valores fora da faixa (`< 0.0` ou `>= 1.0`) sejam clamped no intervalo `[1, 10]` em vez de retornar bucket 0 ou 11. Sem isso, o consumidor TS `bucketEdges[b.bucket - 1]` (§4.4) crasharia com `undefined` em valores aberrantes. Guard paritário em §3.5.4 (panel_2 distribuição `confidence_median` role+skill) e §3.5.7 (panel_5 `gap_days` role+skill).

```sql
CREATE OR REPLACE FUNCTION limiares_panel_8_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  SELECT jsonb_build_object(
    'histogram_role', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket), '[]'::jsonb)
      FROM (
        SELECT GREATEST(LEAST(WIDTH_BUCKET(creation_confidence, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
        FROM job_canonical_roles
        WHERE creation_confidence IS NOT NULL
          AND created_at >= NOW() - p_interval::INTERVAL
        GROUP BY 1
      ) sub
    ),
    'histogram_skill', (
      SELECT COALESCE(jsonb_agg(jsonb_build_object('bucket', bucket, 'cnt', cnt) ORDER BY bucket), '[]'::jsonb)
      FROM (
        SELECT GREATEST(LEAST(WIDTH_BUCKET(creation_confidence, 0.0, 1.0, 10), 10), 1) AS bucket, COUNT(*) AS cnt
        FROM job_canonical_skills
        WHERE creation_confidence IS NOT NULL
          AND created_at >= NOW() - p_interval::INTERVAL
        GROUP BY 1
      ) sub
    ),
    'histogram_by_path_skill', (
      -- F-21 fix: filtro WHERE pipeline_stage IN ('slug_match', 'alias_match', 'llm_new', 'race_recovered')
      -- exclui intencionalmente 'gate_rejected' (não tem skill_confidence — barrado antes do scoring)
      -- e 'fallback_error' (skill_confidence pode estar presente mas é valor pré-erro, não confidence da decisão).
      -- Distribuição reflete apenas paths que produziram canonical (com ou sem reuse).
      SELECT COALESCE(jsonb_agg(jsonb_build_object(
        'path', path, 'bucket', bucket, 'cnt', cnt
      ) ORDER BY path, bucket), '[]'::jsonb)
      FROM (
        SELECT fosi.pipeline_stage::text AS path,
               GREATEST(LEAST(WIDTH_BUCKET(fosi.skill_confidence, 0.0, 1.0, 10), 10), 1) AS bucket,
               COUNT(*) AS cnt
        FROM function_orchestrator_skill_items fosi
        JOIN function_orchestrator_runs r ON r.id = fosi.run_id
        WHERE r.started_at >= NOW() - p_interval::INTERVAL
          AND fosi.skill_confidence IS NOT NULL
          AND fosi.pipeline_stage IN ('slug_match', 'alias_match', 'llm_new', 'race_recovered')
        GROUP BY 1, 2
      ) sub
    )
  );
$$;
```

#### §3.5.11 — `limiares_panel_9_snapshot(p_interval)` — Histórico pipeline_config

Audit trail. Lista até 100 mudanças mais recentes.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_9_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  -- Payload alinhado ao schema real (sem aliases). Nomes: previous_value, changed_by, changed_at.
  SELECT jsonb_build_object(
    'recent_changes', COALESCE(jsonb_agg(jsonb_build_object(
      'id', id, 'key', key, 'previous_value', previous_value, 'new_value', new_value,
      'changed_by', changed_by, 'reason', reason, 'changed_at', changed_at
    ) ORDER BY changed_at DESC), '[]'::jsonb)
  )
  FROM (
    SELECT id, key, previous_value, new_value, changed_by, reason, changed_at
    FROM pipeline_config_history
    WHERE changed_at >= NOW() - p_interval::INTERVAL
    ORDER BY changed_at DESC
    LIMIT 100
  ) sub;
$$;
```

**Vocabulário alinhado ao schema:** payload usa `previous_value`/`changed_by`/`changed_at` (nomes reais da tabela `pipeline_config_history`). RPC entrega vocabulary correto a partir de SUB-PR 6; aggregator §3.6.2 (SUB-PR 7) propaga para JSONB; UI da aba Limiares — painel 9 — consome esses nomes a partir de SUB-PR 11. **J-22 fix v2.5 — clarificação**: a referência em §0.5 a "painel 9 refatorado para vocabulário..." em SUB-PR 11 NÃO é refator independente — é consumo do vocabulary que entra via RPC em SUB-PR 6 + JSONB em SUB-PR 7. Sequencial, não duplicado. Sem aliases na RPC, sem mapeamento em runtime na UI; vocabulário consistente do DB até o componente React.

**Caso `actor` = `SYSTEM_USER_ID`:** ground truth via §6.1 (migration 40) confirma que `pipeline_config_history.changed_by` é `NOT NULL` por construção, populado via trigger com `SYSTEM_USER_ID` (UUID conhecido) quando a mudança vem de processo automático sem admin atrelado. UI da aba Limiares deve tratar `changed_by = SYSTEM_USER_ID` (lookup do UUID conhecido em runtime) exibindo `'sistema'` no painel 9. Smoke S26 (§7.2) valida que admin UI renderiza `changed_by = SYSTEM_USER_ID` como `'sistema'` em pelo menos 1 row sintética inserida com `changed_by=SYSTEM_USER_ID`.

#### §3.5.12 — `limiares_panel_10_snapshot(p_interval)` — Promoções vs Rejeições por dia

Agrupa events `*_promoted_dynamic` + `*_creation_blocked_*` por dia × entity_type.

```sql
CREATE OR REPLACE FUNCTION limiares_panel_10_snapshot(p_interval text)
RETURNS jsonb
LANGUAGE sql STABLE
SET search_path TO 'public', 'pg_temp'
AS $$
  -- F-22 fix: entity inference via CASE com whitelist explícita de event_names paritários.
  -- Antes: `CASE WHEN event_name LIKE 'role_%' THEN 'role' ELSE 'skill' END` matcharia futuros
  -- nomes role_check_*, role_audit_*, etc. com 'role' implicitamente, gerando ruído no painel.
  WITH promotions AS (
    SELECT
      date_trunc('day', created_at) AS day,
      CASE event_name
        WHEN 'role_promoted_dynamic' THEN 'role'
        WHEN 'skill_promoted_dynamic' THEN 'skill'
      END AS entity,
      COUNT(*) AS cnt
    FROM events
    WHERE event_name IN ('role_promoted_dynamic', 'skill_promoted_dynamic')
      AND created_at >= NOW() - p_interval::INTERVAL
    GROUP BY 1, 2
  ),
  rejections AS (
    SELECT
      date_trunc('day', created_at) AS day,
      CASE
        WHEN event_name LIKE 'role_creation_blocked_%' THEN 'role'
        WHEN event_name LIKE 'skill_creation_blocked_%' THEN 'skill'
      END AS entity,
      COUNT(*) AS cnt
    FROM events
    WHERE event_name IN (
      'role_creation_blocked_missing_confidence', 'role_creation_blocked_low_confidence', 'role_creation_blocked_invalid_slug',
      'skill_creation_blocked_missing_confidence', 'skill_creation_blocked_low_confidence', 'skill_creation_blocked_invalid_slug'
    )
      AND created_at >= NOW() - p_interval::INTERVAL
    GROUP BY 1, 2
  )
  SELECT jsonb_build_object(
    'promoted_role', COALESCE((SELECT jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day) FROM promotions WHERE entity='role'), '[]'::jsonb),
    'promoted_skill', COALESCE((SELECT jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day) FROM promotions WHERE entity='skill'), '[]'::jsonb),
    'rejected_role', COALESCE((SELECT jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day) FROM rejections WHERE entity='role'), '[]'::jsonb),
    'rejected_skill', COALESCE((SELECT jsonb_agg(jsonb_build_object('day', day, 'cnt', cnt) ORDER BY day) FROM rejections WHERE entity='skill'), '[]'::jsonb)
  );
$$;
```

**Validação de `p_interval`:** allowlist em cada caller TS (`'1 day'`, `'7 days'`, `'30 days'`, `'90 days'`). RPCs aceitam o texto bruto e fazem `p_interval::INTERVAL` cast no SQL — seguro porque o caller TS já validou contra allowlist antes de invocar.

**Critério de aceite (§7.2):** 12 RPCs criadas; cada uma retorna JSONB válido para `p_interval='7 days'` (smoke); `pg_proc.proconfig` inclui `search_path=public, pg_temp` para todas as 12.

### §3.6 — Expansão de `lib/admin/dashboard-day-aggregator/aggregator.ts`

**SUB-PR:** 7.

**Caminho real do arquivo:** `lib/admin/dashboard-day-aggregator/aggregator.ts` (módulo é um diretório com `aggregator.ts`, `index.ts` e `types.ts`; `index.ts` re-exporta).

#### §3.6.1 — Sub-função `aggregatePipelineOrchestratorMetrics`

Nova sub-função chamada de dentro de `aggregateDayData(supabase, summaryDate)`. Coleta:

- Contadores por `pipeline_stage` em FORI/FOSI via RPCs `count_*_items_by_stage_in_window` (§3.5.1, §3.5.2)
- Contadores por `status` em FOSI via padrão de `count_skill_items_by_status` adaptado com filtro de janela (sub-query inline)
- Items processados (volume bruto) por entidade
- O7-skill drift = `canonicals_skill_novos / items_processados_skill`
- O9-skill saúde — fallback_error rate + sem_canonical
- Snapshot dos 10 painéis Limiares via 10 RPCs `limiares_panel_N_snapshot('1 day')` para o dia em curso

**Anchors temporais:** items (FORI/FOSI) usam `runs.started_at` (run-day) consistentemente em todas as queries — RPCs `count_*_items_by_stage_in_window` e query inline `sem_canonical` JOIN com `function_orchestrator_runs` filtrando `r.started_at`. Events usam `events.created_at` (event-time) — não estão amarrados a runs, então é o único anchor possível. As 2 fontes são intencionalmente diferentes: cross-day para items é controlado por run-day; cross-day para events é controlado por event-time. Smoke S22 valida que items cujo run começou em D-1 mas processaram em D ficam atribuídos a D-1 (consistente cross-painel).

```typescript
// lib/admin/dashboard-day-aggregator/aggregator.ts (expansão)

// J-5 fix v2.5: type definido explicitamente — paritary ao return shape emitido pela função.
// Single source of truth; consumido por aggregateDayData (§3.6.2) sem duplicação.
// [VALIDAR EM E0d: confirmar que este nome não conflita com tipos pré-existentes do aggregator]
export type PipelineOrchestratorMetrics = {
  role: {
    items_by_stage: Record<string, number>;
    items_total: number;
  };
  skill: {
    items_by_stage: Record<string, number>;
    items_total: number;
  };
  o7_skill: {
    canonicals_novos: number;
    items_processed: number;
    drift_percent: number;
  };
  o9_skill: {
    fallback_error: number;
    sem_canonical: number;
    total: number;
  };
  items_processed_total: number;
};

async function aggregatePipelineOrchestratorMetrics(
  supabase: SupabaseClient,
  summaryDate: string,
): Promise<PipelineOrchestratorMetrics> {
  const dayStart = `${summaryDate}T00:00:00Z`;
  // F-13 fix: half-open interval — dayEnd é T00:00:00 do dia seguinte (não T23:59:59 do mesmo dia,
  // que excluiria runs em T23:59:59.500Z e mais). RPC usa `r.started_at < p_end` (strict less-than).
  const nextDay = new Date(`${summaryDate}T00:00:00Z`);
  nextDay.setUTCDate(nextDay.getUTCDate() + 1);
  const dayEnd = nextDay.toISOString();

  // RPCs de stage por janela
  const [foriRes, fosiRes] = await Promise.all([
    supabase.rpc('count_role_items_by_stage_in_window', { p_start: dayStart, p_end: dayEnd }),
    supabase.rpc('count_skill_items_by_stage_in_window', { p_start: dayStart, p_end: dayEnd }),
  ]);

  if (foriRes.error) {
    console.warn('[dashboard-aggregator] orchestrator.fori_stages_query_failed:', foriRes.error.message);
  }
  if (fosiRes.error) {
    console.warn('[dashboard-aggregator] orchestrator.fosi_stages_query_failed:', fosiRes.error.message);
  }

  const foriStages = Object.fromEntries(
    (foriRes.data ?? []).map((r: { stage: string; c: number }) => [r.stage, r.c])
  );
  const fosiStages = Object.fromEntries(
    (fosiRes.data ?? []).map((r: { stage: string; c: number }) => [r.stage, r.c])
  );

  const roleItemsTotal = Object.values(foriStages).reduce((a: number, b) => a + (b as number), 0);
  const skillItemsTotal = Object.values(fosiStages).reduce((a: number, b) => a + (b as number), 0);

  // Canonicals novos skill (paritário ao role)
  // Convenção de nome do evento: canonical_skill_created (paritário a canonical_role_created
  // já cravado pela cleanup v3.4 reconcile_canonical_role)
  const skillCanonicalsCreatedRes = await supabase
    .from('events')
    .select('id', { count: 'exact', head: true })
    .eq('event_name', 'canonical_skill_created')
    .gte('created_at', dayStart)
    .lt('created_at', dayEnd)  // F-13 fix: half-open interval;

  if (skillCanonicalsCreatedRes.error) {
    console.warn('[dashboard-aggregator] orchestrator.skill_canonicals_query_failed:',
                 skillCanonicalsCreatedRes.error.message);
  }

  const skillCanonicalsNovos = skillCanonicalsCreatedRes.count ?? 0;
  const o7SkillDrift = skillItemsTotal > 0 ? (skillCanonicalsNovos / skillItemsTotal) * 100 : 0;

  const fallbackErrorCount = fosiStages['fallback_error'] || 0;
  // Anchor temporal paritário às RPCs §3.5.1/§3.5.2: usar `runs.started_at` (run-day), não fosi.created_at.
  // Items cujo run começou em D-1 mas processaram em D ficam atribuídos ao dia D-1 (consistente cross-painel).
  const semCanonicalRes = await supabase
    .from('function_orchestrator_skill_items')
    .select('id, function_orchestrator_runs!inner(started_at)', { count: 'exact', head: true })
    .is('canonical_skill_id', null)
    .neq('pipeline_stage', 'gate_rejected')  // gate_rejected é estado intencional, não problema
    .gte('function_orchestrator_runs.started_at', dayStart)
    .lt('function_orchestrator_runs.started_at', dayEnd)  // F-13 fix: half-open interval;

  if (semCanonicalRes.error) {
    console.warn('[dashboard-aggregator] orchestrator.sem_canonical_query_failed:',
                 semCanonicalRes.error.message);
  }

  // F-7/F-8 fix: shape nested paritário ao consumer UI §5.2.1 (single source of truth).
  // Estrutura final do JSONB data.operational consumida diretamente por panel_2.7/2.8/2.9 sem mapeamento adicional.
  return {
    role: {
      items_by_stage: foriStages,
      items_total: roleItemsTotal,
    },
    skill: {
      items_by_stage: fosiStages,
      items_total: skillItemsTotal,
    },
    o7_skill: {
      canonicals_novos: skillCanonicalsNovos,
      items_processed: skillItemsTotal,
      drift_percent: o7SkillDrift,
    },
    o9_skill: {
      fallback_error: fallbackErrorCount,
      sem_canonical: semCanonicalRes.count ?? 0,
      total: fallbackErrorCount + (semCanonicalRes.count ?? 0),
    },
    items_processed_total: roleItemsTotal + skillItemsTotal,
  };
}

// Sub-função paritária para snapshot diário dos 10 painéis Limiares
async function aggregateLimiaresDailySnapshots(
  supabase: SupabaseClient,
  summaryDate: string,
): Promise<Record<string, unknown>> {
  // Cada RPC retorna JSONB já estruturado para o painel N
  const panels: Record<string, unknown> = {};

  const panelIds = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  const rpcResults = await Promise.all(
    panelIds.map(n =>
      supabase.rpc(`limiares_panel_${n}_snapshot`, { p_interval: '1 day' })
    )
  );

  rpcResults.forEach((res, idx) => {
    const panelN = panelIds[idx];
    if (res.error) {
      console.warn(`[dashboard-aggregator] limiares.panel_${panelN}_snapshot_failed:`, res.error.message);
      panels[`panel_${panelN}`] = null;
    } else {
      panels[`panel_${panelN}`] = res.data;
    }
  });

  return panels;
}
```

#### §3.6.2 — Refator do shape JSONB e incorporação em `aggregateDayData`

**Mudança estrutural do JSONB:** o aggregator pré-existente (`aggregator.ts:93-110`) emite famílias top-level **uppercase** `data.O7`, `data.O8`, `data.O9` (role-only) ao lado de `data.ai`, `data.resources`, `data.communications`, `data.auth`, `data.errors`, `data.accumulators` (lowercase, naming pattern padrão das demais famílias). Essa assimetria nominal é pré-existente a esta sprint.

Esta sprint reorganiza para naming pattern uniforme — todas as famílias em lowercase aninhado:
- `data.O7` → `data.operational.o7` (role pré-existente)
- `data.O8` → `data.operational.o8` (role pré-existente)
- `data.O9` → `data.operational.o9` (role pré-existente)
- + `data.operational.o7_skill` (novo, paritário)
- + `data.operational.o8_skill` (novo, paritário)
- + `data.operational.o9_skill` (novo, paritário)
- + `data.operational.items_processed` (novo, agregado role+skill)
- + `data.limiares.panel_1..10` (nova família, 10 painéis Limiares)

**Deploy coordenado obrigatório:** o refator do shape exige que aggregator + backfill + UI dos painéis 2.7/2.8/2.9 + UI do KPI 2.1 sejam aplicados no mesmo SUB-PR (SUB-PR 7 + SUB-PR 11 em sequência sem janela operacional entre eles). Janela de inconsistência: durante o backfill, dias regenerados terão `data.operational.*` e dias não-regenerados ainda terão `data.O7/O8/O9` top-level. UI atualizada lê `data.operational.*`; UI antiga lê `data.O7`. Estratégia: rodar backfill 26 dias rolling primeiro (rápido — minutos), depois UI deployed; janela total < 5 min em pré-produção. Em produção pós-MVP, considerar feature flag.

```typescript
// J-4 fix v2.5: tipo real é SummaryData (lib/admin/dashboard-day-aggregator/types.ts:9), não DayData.
// [VALIDAR EM E0d: confirmar export name + shape completo de SummaryData; spec assume fields paritários abaixo + alguns fields pré-existentes da cleanup v3.4 (analyses, credits_consumed, signups, etc.) preservados sem alteração]
import type { SummaryData } from '@/lib/admin/dashboard-day-aggregator/types';

async function aggregateDayData(
  supabase: SupabaseClient,
  summaryDate: string,
): Promise<SummaryData> {
  // Família operational pré-existente (Promise.all paritário — paridade fail-fast)
  const [analyses, ai, comms, curation, auth, misc] = await Promise.all([
    fetchAnalysesAndPayments(supabase, summaryDate),
    fetchAiData(supabase, summaryDate),
    fetchCommsData(supabase, summaryDate),
    fetchCurationData(supabase, summaryDate),   // retorna {o7, o8, o9} role-only pré-existentes
    fetchAuthData(supabase, summaryDate),
    fetchMiscData(supabase, summaryDate),
  ]);

  // NOVO: leitura de .error em queries pré-existentes (correção paritária da dívida)
  // Cada fetcher acima é atualizado internamente para ler .error e logar
  // [dashboard-aggregator] <fetcher>_query_failed antes do fallback 0.

  // Sub-funções NOVAS
  const [pipelineMetrics, limiaresSnapshots] = await Promise.all([
    aggregatePipelineOrchestratorMetrics(supabase, summaryDate),
    aggregateLimiaresDailySnapshots(supabase, summaryDate),
  ]);

  return {
    // Famílias pré-existentes da cleanup v3.4 preservadas (estrutura exata mantida — analyses/credits_consumed/signups/etc.)
    // [VALIDAR EM E0d: enumerar fields top-level pré-existentes de SummaryData; spec assume preservação 1:1 de todos exceto operational e limiares]
    ...analyses,           // analyses, credits_consumed, signups, etc. (paritary ao return pré-existente)
    ai, resources: misc.resources, communications: comms, auth, errors: misc.errors, accumulators: misc.accumulators,
    // FAMÍLIA operational REORGANIZADA: lowercase aninhado, incorporando role pré-existente + skill novo
    operational: {
      // Role pré-existentes (antes em data.O7/O8/O9 top-level; agora em data.operational.o7/o8/o9)
      o7: curation.o7,
      o8: curation.o8,
      o9: curation.o9,
      // Skill novos paralelos — F-7/F-8 fix: shape nested vindo direto do aggregator (single source of truth)
      o7_skill: pipelineMetrics.o7_skill,
      // G-3 fix: o8_skill com total paritary a o7_skill/o9_skill (UI §5.2.1 espera o8_skill.total)
      o8_skill: { ...pipelineMetrics.skill.items_by_stage, total: pipelineMetrics.skill.items_total },
      o9_skill: pipelineMetrics.o9_skill,
      items_processed: {
        role: pipelineMetrics.role.items_total,
        skill: pipelineMetrics.skill.items_total,
        total: pipelineMetrics.items_processed_total,
      },
    },
    // Família NOVA: snapshot diário dos 10 painéis Limiares
    limiares: limiaresSnapshots,
  };
}
```

**Validação pós-deploy crítica:**

```sql
-- Pós backfill 26 dias, validar que nenhum dia ainda tem o shape antigo top-level
SELECT summary_date, jsonb_object_keys(data) AS k
FROM dashboard_daily_summary
WHERE 'O7' IN (SELECT jsonb_object_keys(data));
-- Esperado: 0 rows. Se aparecer alguma, backfill não cobriu todos os dias.
```

#### §3.6.3 — Backfill obrigatório pós-deploy

Após deploy do aggregator expandido, rodar `/api/admin/backfill-dashboard-summary` para os 26 dias rolling. Endpoint pré-existente é idempotente e reusa `aggregateDayData()`.

Para dias ANTES da implementação do SUB-PR 4 (FOSI populada e finalizers escrevendo), FOSI estará vazia para aqueles dias → `data.operational.o8_skill = {}`, `data.operational.o9_skill.fallback_error = 0`, `data.operational.items_processed.skill = 0`. Para `data.limiares.panel_*`, snapshots dos painéis que dependem de FOSI (painéis 1 que tem série FOSI; painel 8 modo "Por path") terão dados parciais para dias anteriores. Aceitável em pré-produção pré-MVP. UI dos painéis trata zero como "sem dados", não como sinal de problema.

#### §3.6.4 — Campos role-only pré-existentes mantidos intactos

`data.operational.o7`, `data.operational.o8`, `data.operational.o9` (todos role-only) **NÃO são tocados** por esta sprint. UI dos painéis 2.7/2.8/2.9 consome ambos os conjuntos (role pré-existente + skill novo). Sem refator dos consumidores pré-existentes.

**Critério de aceite (§7.2):** após backfill, query `SELECT data->'operational'->'o7_skill', data->'limiares'->'panel_1' FROM dashboard_daily_summary WHERE summary_date = CURRENT_DATE - 1` retorna valores válidos (não NULL); `data->'operational'->'o7'` pré-existente preservado intacto; smoke de 1 dia de tráfego sintético popula campos esperados com valores não-zero.

---

## §4 Endpoints API

### §4.1 — Endpoint `POST /api/admin/pipeline-config/[key]/impact-preview`

**SUB-PR:** 10.

**Caminho:** `app/api/admin/pipeline-config/[key]/impact-preview/route.ts`.

**Pré-requisito:** cleanup v3.4 mergeada — endpoint pré-existente está em produção com:

- Método POST + body `{new_value: string, days?: number, skip_cache?: boolean}` (campos `days`/`skip_cache` foram pré-cravados pela cleanup v3.4 F11 + diretrizes operacionais)
- Cache key inclui `updated_at` do `pipeline_config` (cleanup v3.4 F11)
- 28 chaves cobertas via **estimators** (cleanup v3.4 F23 — Caminho B com simulação delta); **12 dessas chaves** (key_class: 'covered') recebem **histograma adicional** via `IMPACT_SOURCES` (`panel_path` populado) nesta sprint, **14 estimator_only** (M-1 fix v2.8 — paritary ao canonical breakdown §4.1.4: 4 `*.confidence.*` + 2 `auto_assign_family.*` role-only + 8 reclassificadas em K-1/K-4/K-5 v2.6 [`*.lookback_days` x2 + `*.merge_candidate.lookback_days` x2 + `*.merge_candidate.opus_review_cooldown_days` x2 + `*.opus_review.cooldown_days` x2]) ficam apenas com estimator (`panel_path: ''` por D-PS-41 ou drop K-1 v2.6, sem painel Limiares simétrico), e **2 system_key** (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) retornam payload de marcação operacional sem análise quantitativa
- Auth `requireAdmin(req)`
- Shape de payload: `{affected_count, affected_label, projected_event_cost_usd, cost_window, cost_is_fallback, panels}`
- Cache atual: in-process Map TTL 30s inline em `route.ts:18-46`

#### §4.1.1 — Mudanças desta sprint

1. Cache in-process Map TTL 30s é **REMOVIDO**. Endpoint passa a consumir `dashboard_daily_summary` para janelas históricas (`days ∈ {7, 30, 90}`) e invocar RPCs SQL via Supabase client para sample do dia corrente (janela `'1 day'`).
2. `pg.Pool` direto **NÃO é introduzido** nesta sprint — endpoint segue inteiramente via Supabase client + RPCs SQL.
3. Redis custom **NÃO é introduzido**. Cache via header HTTP `cache-control: max-age=60` paritário ao resto do dashboard global.
4. Payload preserva campos pré-existentes da cleanup v3.4 e adiciona campos novos para apoiar a UI expandida.

#### §4.1.2 — Contrato (preservado)

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

- `key` (path param) — chave de `pipeline_config` (uma das 28 cobertas pela cleanup v3.4 F23)
- `new_value` (body, obrigatório) — valor proposto (string, validado conforme tipo da chave)
- `days` (body, opcional) — janela retroativa; valores aceitos: `7`, `30`, `90`; default `30`
- `skip_cache` (body, opcional) — passa cabeçalho `cache-control: no-store` (não há mais cache em memória/Redis a invalidar; flag preservada por contrato)

**Auth:** admin (`requireAdmin(req)`).

**Validação de janela:** allowlist em runtime; valor inválido → HTTP 400 `{error: 'invalid_window', accepted: [7, 30, 90]}`.

#### §4.1.3 — Estratégia de dados

Para cada chamada:

1. **Resolução do `current_value`** — `SELECT value, updated_at FROM pipeline_config WHERE key = $key` via Supabase client.
2. **Histórico (janela)** — query `dashboard_daily_summary` cross-day. Os snapshots dos painéis Limiares relevantes para a chave editada (mapa `IMPACT_SOURCES` adaptado para apontar para `data.limiares.panel_N` em vez de queries dedicadas) fornecem a base estatística agregada.
3. **Dia corrente** — invocar RPCs SQL com `p_interval='1 day'` para o subset de painéis relevantes para a chave (mesmo mapa `IMPACT_SOURCES`).
4. **Estimator base** — invocar estimator pré-existente da cleanup v3.4 §3.16 (todas as 28 chaves cobertas) para `affected_count` + `projected_event_cost_usd`.
5. **Histograma e impacto detalhado** — derivado da combinação de histograma agregado + thresholds atual/proposto via helper `deriveImpactFromDistribution`.

#### §4.1.4 — Mapa `IMPACT_SOURCES` (chave → fonte no `dashboard_daily_summary`)

```typescript
// lib/admin/pipeline-config-impact-sources.ts

export type ImpactSource = {
  panel_path: string;                                    // path no JSONB de dashboard_daily_summary; '' para chaves cobertas só por estimator ou system_key
  histogram_path?: string;                               // path no JSONB para histograma (subset com sample_size suficiente)
  current_value_filter?: 'gte' | 'lte';                  // H-10 fix: drop 'eq' (YAGNI — nenhuma entry usa); restringido para bater com helper §4.4
  key_class: 'covered' | 'estimator_only' | 'system_key'; // classe da chave para sample_status; consumido por classifySample
  affected_label: string;                                // label legível para o admin (ex: "items podem ser barrados pelo gate")
  panels: string[];                                      // J-2 fix v2.5: lista de painéis UI afetados pela chave; metadado estático do entry (movido de ImpactEstimate da cleanup v3.4, que NÃO tem este campo — [VALIDADO via type real de @/lib/admin/pipeline-impact-estimators])
  system_key_reason?: string;                            // H-3 fix: razão explícita exibida na UI quando key_class==='system_key'; opcional para entries non-system_key
};

export const IMPACT_SOURCES: Record<string, ImpactSource> = {
  // HARD GATE (2)
  'role.hard_gate.min_confidence': {
    panel_path: 'data.limiares.panel_1.series_role_events',
    histogram_path: 'data.limiares.panel_8.histogram_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles podem ser barradas pelo gate',
    panels: ['Aba Limiares — Painel 1 (Hard Gate)', 'Aba Limiares — Painel 8 (creation_confidence)'],
  },
  'skill.hard_gate.min_confidence': {
    panel_path: 'data.limiares.panel_1.series_skill_fosi',
    histogram_path: 'data.limiares.panel_8.histogram_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills podem ser barradas pelo gate',
    panels: ['Aba Limiares — Painel 1 (Hard Gate)', 'Aba Limiares — Painel 8 (creation_confidence)'],
  },
  // PROMOTION (8)
  'role.promotion.auto_min_confidence': {
    panel_path: 'data.limiares.panel_2.distribution_role',
    histogram_path: 'data.limiares.panel_2.distribution_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles pendentes podem ser auto-promovidas',
    panels: ['Aba Limiares — Painel 2 (Promoção por confiança)'],
  },
  'skill.promotion.auto_min_confidence': {
    panel_path: 'data.limiares.panel_2.distribution_skill',
    histogram_path: 'data.limiares.panel_2.distribution_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills pendentes podem ser auto-promovidas',
    panels: ['Aba Limiares — Painel 2 (Promoção por confiança)'],
  },
  // K-5 fix v2.6: lookback_days reclassificada como 'estimator_only'.
  // Panel_10 retorna [{day, cnt}] (time-series) — não threshold-comparable contra current_value de dias.
  'role.promotion.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles entram em janela de avaliação para promoção',
    panels: ['Aba Limiares — Painel 10 (Promoções vs Rejeições)'],
  },
  'skill.promotion.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills entram em janela de avaliação para promoção',
    panels: ['Aba Limiares — Painel 10 (Promoções vs Rejeições)'],
  },
  'role.promotion.min_distinct_employers': {
    panel_path: 'data.limiares.panel_4.pending_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles podem promover por diversidade de empregadores',
    panels: ['Aba Limiares — Painel 4 (Promoção por diversidade)'],
  },
  'skill.promotion.min_distinct_employers': {
    panel_path: 'data.limiares.panel_4.pending_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills podem promover por diversidade de empregadores',
    panels: ['Aba Limiares — Painel 4 (Promoção por diversidade)'],
  },
  'role.promotion.min_vacancies': {
    panel_path: 'data.limiares.panel_4.pending_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles podem promover por volume de vagas',
    panels: ['Aba Limiares — Painel 4 (Promoção por volume)'],
  },
  'skill.promotion.min_vacancies': {
    panel_path: 'data.limiares.panel_4.pending_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills podem promover por volume de vagas',
    panels: ['Aba Limiares — Painel 4 (Promoção por volume)'],
  },
  // MERGE_CANDIDATE (6)
  'role.merge_candidate.cosine_threshold': {
    panel_path: 'data.limiares.panel_3.candidates_role',
    histogram_path: 'data.limiares.panel_3.candidates_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles podem entrar em janela de revisão de merge',
    panels: ['Aba Limiares — Painel 3 (Candidatos de merge)'],
  },
  'skill.merge_candidate.cosine_threshold': {
    panel_path: 'data.limiares.panel_3.candidates_skill',
    histogram_path: 'data.limiares.panel_3.candidates_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills podem entrar em janela de revisão de merge',
    panels: ['Aba Limiares — Painel 3 (Candidatos de merge)'],
  },
  // K-4 fix v2.6: merge_candidate.{lookback_days,opus_review_cooldown_days} reclassificadas como 'estimator_only'.
  // Panel_3 retorna [{id, similarity, detected_at}] — similarity é numérico bucketable, mas para cosine_threshold
  // (preservado como 'covered'); days-based keys precisariam de campo numérico de tempo no payload.
  'role.merge_candidate.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles entram em janela de avaliação de merge',
    panels: ['Aba Limiares — Painel 3 (Janela de merge)'],
  },
  'skill.merge_candidate.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills entram em janela de avaliação de merge',
    panels: ['Aba Limiares — Painel 3 (Janela de merge)'],
  },
  'role.merge_candidate.opus_review_cooldown_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles aguardam cooldown antes de nova revisão Opus',
    panels: ['Aba Limiares — Painel 3 (Cooldown Opus)', 'Aba Limiares — Painel 9 (Histórico arbitragens)'],
  },
  'skill.merge_candidate.opus_review_cooldown_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills aguardam cooldown antes de nova revisão Opus',
    panels: ['Aba Limiares — Painel 3 (Cooldown Opus)', 'Aba Limiares — Painel 9 (Histórico arbitragens)'],
  },
  // OPUS_REVIEW (2) — K-1 fix v2.6: reclassificadas como 'estimator_only' por drop de J-13/J-14.
  // Panel_9 retorna lista de history (recent_changes), sem campo numérico bucketable contra current_value.
  // Sprint futura (LK-PS-23) pode adicionar `days_since` ao panel_9 RPC para retornar 'covered' aqui.
  'role.opus_review.cooldown_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles aguardam cooldown antes de nova arbitragem',
    panels: ['Aba Limiares — Painel 9 (Histórico arbitragens)'],
  },
  'skill.opus_review.cooldown_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills aguardam cooldown antes de nova arbitragem',
    panels: ['Aba Limiares — Painel 9 (Histórico arbitragens)'],
  },
  // RETIREMENT (2)
  'role.retirement.gap_days': {
    panel_path: 'data.limiares.panel_6.retire_role',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'roles podem ser aposentadas por gap de vagas',
    panels: ['Aba Limiares — Painel 6 (Aposentadoria por gap)'],
  },
  'skill.retirement.gap_days': {
    panel_path: 'data.limiares.panel_6.retire_skill',
    current_value_filter: 'gte',
    key_class: 'covered',
    affected_label: 'skills podem ser aposentadas por gap de vagas',
    panels: ['Aba Limiares — Painel 6 (Aposentadoria por gap)'],
  },
  // CONFIDENCE (4) — cobertura via estimator base da v3.4 (sem histograma derivado)
  'role.confidence.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles recalculam confidence_median com janela ampliada',
    panels: ['Caixa qualitativa — recálculo de confidence_median'],
  },
  'skill.confidence.lookback_days': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills recalculam confidence_median com janela ampliada',
    panels: ['Caixa qualitativa — recálculo de confidence_median'],
  },
  'role.confidence.min_count': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'roles atingem min_count para median elegível',
    panels: ['Caixa qualitativa — recálculo de confidence_median'],
  },
  'skill.confidence.min_count': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'skills atingem min_count para median elegível',
    panels: ['Caixa qualitativa — recálculo de confidence_median'],
  },
  // AUTO_ASSIGN_FAMILY (2 role-only — assimetria D-PS-41 cleanup v3.4)
  'role.auto_assign_family.min_similarity': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'vagas podem reassociar a outra família',
    panels: ['Caixa qualitativa — recálculo de auto-assign de família'],
  },
  'role.auto_assign_family.min_score': {
    panel_path: '',
    key_class: 'estimator_only',
    affected_label: 'vagas podem reassociar por score ajustado',
    panels: ['Caixa qualitativa — recálculo de auto-assign de família'],
  },
  // SYSTEM_KEY (2) — H-3 fix: chaves operacionais/feature flags expostas só via API direta (filtradas da tela /admin/pipeline-config).
  // key_class:'system_key' + panel_path:'' garante que classifySample retorna 'system_key' (não 'unsupported').
  // composePayload usa source.system_key_reason para popular o campo paritário no payload (H-4 fix).
  'CURATE_PIPELINE_ENABLED': {
    panel_path: '',
    key_class: 'system_key',
    affected_label: 'pipeline de curadoria está habilitado',
    panels: [],
    system_key_reason: 'Chave de sistema (feature flag de kill-switch global do pipeline). Não exposta na tela /admin/pipeline-config; edição apenas via API direta ou super-admin.',
  },
  'QUARANTINE_EXPIRY_DAYS': {
    panel_path: '',
    key_class: 'system_key',
    affected_label: 'items aguardam TTL de quarentena',
    panels: [],
    system_key_reason: 'Chave de sistema (TTL operacional de quarentena). Não exposta na tela /admin/pipeline-config; edição apenas via API direta ou super-admin.',
  },
};

// Validação de cobertura: 28 entries em IMPACT_SOURCES desta sprint:
//   - 12 'covered' (HARD_GATE 2 + PROMOTION 4 [auto_min_confidence + min_distinct_employers + min_vacancies role+skill — drop lookback_days] + MERGE_CANDIDATE 2 [cosine_threshold role+skill — drop lookback/opus_cooldown] + RETIREMENT 2)
//   - 14 'estimator_only' (CONFIDENCE 4 role+skill {lookback_days, min_count} + AUTO_ASSIGN_FAMILY 2 role-only + 8 reclassificadas em K-1/K-4/K-5 v2.6 [*.lookback_days role+skill x2 + *.merge_candidate.lookback_days role+skill + *.merge_candidate.opus_review_cooldown_days role+skill + *.opus_review.cooldown_days role+skill])
//   - 2 'system_key' (CURATE_PIPELINE_ENABLED + QUARANTINE_EXPIRY_DAYS) — H-3 fix
// Build quebra se Object.keys(IMPACT_SOURCES).length !== 28
// cleanup v3.4 §3.16 entregou cobertura 28/28 via estimator base; esta sprint amplia metadata
// com key_class + affected_label + system_key_reason inline.
```

Espera-se `Object.keys(IMPACT_SOURCES).length === 28`. Comportamento por cenário de chave forçada via curl direto:
- **Chave NÃO em `pipeline_config`** (key inexistente no DB): handler retorna HTTP 404 (`key_not_found`) — `§4.1.10` lookup falha antes de chegar ao IMPACT_SOURCES
- **Chave em `pipeline_config` mas NÃO em IMPACT_SOURCES** (key registrada no DB mas ainda não cadastrada no registry da sprint): `source = IMPACT_SOURCES[key]` retorna `undefined` → `classifySample` cai no branch genérico → payload retorna HTTP 200 com `sample_status: 'unsupported'` + UI mostra mensagem "Análise quantitativa indisponível"

#### §4.1.5 — Payload de resposta — caso normal (sample suficiente)

**Single source of truth do payload** (TypeScript literal — aggregator, helpers, UI consumer e wire spec referenciam este tipo, não duplicam shape):

```typescript
// lib/admin/pipeline-config-impact-preview-types.ts (arquivo novo desta sprint)
// J-27 fix v2.5: convenção flat (sibling) — paritary aos demais arquivos lib/admin/pipeline-config-*.ts
// (pipeline-config-impact-sources.ts, pipeline-config-classify-sample.ts, etc).

export type SampleStatus =
  | 'sufficient'      // distribuição tem dados suficientes para histograma
  | 'insufficient'    // histograma omitido por falta de amostra (current_impact ainda exibido)
  | 'estimator_only'  // chave coberta apenas por estimator (panel_path: '' em IMPACT_SOURCES; ex: auto_assign_family.* role-only)
  | 'system_key'      // chave de sistema sem análise quantitativa (CURATE_PIPELINE_ENABLED, QUARANTINE_EXPIRY_DAYS)
  | 'unsupported';    // análise quantitativa indisponível na janela atual (estimator falhou OU impact-source falhou OU ambas as fontes vazias)

export type ImpactSeries = {
  items_total: number;
  items_passing: number;
  items_rejecting: number;
  pass_rate: number;
  // K-1 fix v2.6: items_inside_cooldown? REMOVIDO desta versão (feature J-13/J-14 dropped).
  // Não há helper code path que popule este campo + entries cooldown reclassificadas como 'estimator_only'.
  // Sprint futura (LK-PS-23) pode reintroduzir quando RPCs panel_3/9/10 emitirem campo days_since bucketable.
};

export type ProposedImpact = ImpactSeries & {
  delta_passing: number;     // proposedPassing - currentPassing
  delta_rejecting: number;   // currentPassing - proposedPassing (simétrico negativo)
};

export type HistogramBucket = {
  range: [number, number];   // [min, max] do bucket
  count: number;
};

export type Histogram = {
  buckets: HistogramBucket[];
  bucket_count: number;       // 10 fixo nesta versão (WIDTH_BUCKET em 10 buckets)
};

export type SampleThreshold = {
  absolute_min: number;       // 30 (cravado em classifySample)
  relative_min_pct: number;   // 0.05
};

export type ImpactPreviewPayload = {
  // Identidade
  key: string;
  current_value: string;
  new_value: string;
  window_days: 7 | 30 | 90;

  // Origem da análise quantitativa
  source: 'dashboard_daily_summary' | 'estimator_only' | 'system_key' | null;

  // Campos pré-existentes cleanup v3.4 (SEMPRE emitidos)
  affected_count: number | null;
  affected_label: string | null;
  projected_event_cost_usd: number | null;
  cost_window: '30d' | null;  // K-9 fix v2.6: type tightening — pipeline-impact-estimators.ts:23 emite literal '30d' ou null
  cost_is_fallback: boolean;
  panels: string[];

  // Campos novos desta sprint (emitidos conforme sample_status)
  sample_size: number;
  sample_status: SampleStatus;
  sample_threshold: SampleThreshold | null;  // null quando sample_status='system_key' ou 'unsupported'
  current_impact: ImpactSeries | null;        // null quando sample_status='unsupported' ou 'system_key'
  proposed_impact: ProposedImpact | null;     // null quando sample_status='unsupported' ou 'system_key'
  histogram: Histogram | null;                // null quando sample_status !== 'sufficient'

  // G-12 fix: campo opcional emitido apenas quando sample_status='system_key' (ver §4.1.8)
  system_key_reason?: string;
};
```

**Exemplo de payload — caso normal (`sample_status='sufficient'`):** (K-2 fix v2.6: `cost_window` literal `"30d"` paritary ao `ImpactEstimate.cost_window: '30d' | null` real do `pipeline-impact-estimators.ts:23` — versões anteriores usavam `"evento_unico"`, valor que nunca era produzido em runtime)

```json
{
  "key": "skill.hard_gate.min_confidence",
  "current_value": "0.70",
  "new_value": "0.75",
  "window_days": 30,
  "source": "dashboard_daily_summary",

  "affected_count": 1756,
  "affected_label": "items podem ser barrados pelo gate",
  "projected_event_cost_usd": 0.43,
  "cost_window": "30d",
  "cost_is_fallback": false,
  "panels": ["Aba Limiares — Painel 1 (Hard Gate)", "Aba Limiares — Painel 8 (creation_confidence)"],

  "sample_size": 12450,
  "sample_status": "sufficient",
  "sample_threshold": { "absolute_min": 30, "relative_min_pct": 0.05 },
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

**Campos pré-existentes da cleanup v3.4 (preservados sem mudança):** `key`, `current_value`, `new_value`, `affected_count`, `affected_label`, `projected_event_cost_usd`, `cost_window`, `cost_is_fallback`, `panels`.
**Campos novos desta sprint:** `window_days`, `source`, `sample_size`, `sample_status`, `sample_threshold`, `current_impact`, `proposed_impact`, `histogram`.

#### §4.1.6 — Payload — sample insuficiente

**L-2 fix v2.7:** exemplo usa `skill.merge_candidate.cosine_threshold` (chave `covered` cujo painel pode legitimamente produzir `sample_status='insufficient'` em ambientes com pouco volume de candidatos). Versões anteriores (v2.5/v2.6) usavam `skill.opus_review.cooldown_days` que pós-K-1 foi reclassificada como `estimator_only` — `classifySample` faz short-circuit `if (keyClass === 'estimator_only') return 'estimator_only'`, tornando `sample_status='insufficient'` fisicamente impossível para essa chave.

```json
{
  "key": "skill.merge_candidate.cosine_threshold",
  "current_value": "0.80",
  "new_value": "0.85",
  "window_days": 30,
  "source": "dashboard_daily_summary",

  "affected_count": 8,
  "affected_label": "pares de skill ficam acima/abaixo do threshold de merge",
  "projected_event_cost_usd": null,
  "cost_window": null,
  "cost_is_fallback": false,
  "panels": ["Aba Limiares — Painel 3 (Candidatos de merge)"],

  "sample_size": 8,
  "sample_status": "insufficient",
  "sample_threshold": {
    "absolute_min": 30,
    "relative_min_pct": 0.05
  },
  "current_impact": null,
  "proposed_impact": null
  /* histogram OMITIDO quando sample_status === 'insufficient' */
}
```

#### §4.1.7 — Payload — chave coberta apenas por estimator (`auto_assign_family.*` role-only)

Para `role.auto_assign_family.min_similarity` e `role.auto_assign_family.min_score` — cobertas por estimator pré-existente da cleanup v3.4 §3.16 mas **registradas em `IMPACT_SOURCES` com `panel_path: ''`** (D-PS-41 cleanup v3.4 — assimetria de famílias só role; sem painel Limiares 6 para skill consequentemente sem histograma derivado pelo `dashboard_daily_summary`):

```json
{
  "key": "role.auto_assign_family.min_similarity",
  "current_value": "0.75",
  "new_value": "0.80",
  "window_days": 30,
  "source": "estimator_only",

  "affected_count": 23,
  "affected_label": "vagas podem reassociar a outra família",
  "projected_event_cost_usd": 0.12,
  "cost_window": "30d",
  "cost_is_fallback": false,
  "panels": ["Caixa qualitativa — recálculo de auto-assign de família"],

  "sample_size": 0,
  "sample_status": "estimator_only",
  "sample_threshold": { "absolute_min": 30, "relative_min_pct": 0.05 },
  "current_impact": null,
  "proposed_impact": null,
  "histogram": null
}
```

UI exibe linha "Custo Opus projetado (evento único)" da caixa qualitativa pré-existente + placeholder no bloco quantitativo: "Análise quantitativa não disponível para esta chave nesta versão — caixa qualitativa ao lado orienta o impacto."

#### §4.1.8 — Payload — chave de sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`)

Comportamento cravado pela cleanup v3.4 (tela admin pipeline-config filtra essas chaves via §5.1.0 da cleanup). Caso o endpoint seja chamado diretamente (via curl ou outro caller bypassando a UI), retorna HTTP 200 com:

```json
{
  "key": "CURATE_PIPELINE_ENABLED",
  "current_value": "true",
  "new_value": "false",
  "window_days": 30,
  "source": "system_key",

  "affected_count": null,
  "affected_label": "pipeline de curadoria está habilitado",
  "projected_event_cost_usd": null,
  "cost_window": null,
  "cost_is_fallback": false,
  "panels": [],

  "sample_size": 0,
  "sample_status": "system_key",
  "sample_threshold": null,
  "system_key_reason": "Chave de sistema (feature flag de kill-switch global do pipeline). Não exposta na tela /admin/pipeline-config; edição apenas via API direta ou super-admin.",
  "current_impact": null,
  "proposed_impact": null,
  "histogram": null
}
```

HTTP **200** (não 404) — preserva semântica do endpoint que sempre retorna 200 com campos nulos quando a chave não é apta a análise quantitativa.

#### §4.1.9 — Lógica do `sample_status`

**Localização:** `lib/admin/pipeline-config-classify-sample.ts` (arquivo novo). Função pura testável isolada; consumida pelo aggregator §4.1.10 e por testes unitários. Retorna `SampleStatus` (tipo cravado em §4.1.5 — single source of truth).

```typescript
import type { SampleStatus } from '@/lib/admin/pipeline-config-impact-preview-types';

export const SAMPLE_THRESHOLD = {
  absolute_min: 30,
  relative_min_pct: 0.05,
} as const;

export function classifySample(
  itemCount: number,
  universeSize: number,
  keyClass: 'covered' | 'estimator_only' | 'system_key',
): SampleStatus {
  // Chaves classificadas a priori (independem de amostra)
  if (keyClass === 'system_key') return 'system_key';
  if (keyClass === 'estimator_only') return 'estimator_only';

  // Sem amostra → análise quantitativa indisponível (não é "insufficient" — insufficient implica QUE há amostra, só não suficiente)
  if (itemCount === 0 && universeSize === 0) return 'unsupported';

  // Critério OR — basta uma das 2 condições serem verdadeiras
  if (itemCount >= SAMPLE_THRESHOLD.absolute_min) return 'sufficient';
  if (universeSize > 0 && itemCount >= universeSize * SAMPLE_THRESHOLD.relative_min_pct) return 'sufficient';
  return 'insufficient';
}
```

**Critério OR (não AND)** — permite reconhecer caso onde N é pequeno mas é 100% do universo (ex: chave nova com 5 arbitragens totais; 5/5 = 100% e é o melhor sinal possível). Tabela de contagem SEMPRE exibida quando `current_impact` existe. Histograma omitido quando `sample_status` != `'sufficient'`. Edição NÃO bloqueada em nenhum caso — admin tem autonomia.

**5 estados do enum `SampleStatus`:**

| Estado | Origem | UI render |
|---|---|---|
| `'sufficient'` | classifySample com amostra >= threshold | Tabela completa + histograma |
| `'insufficient'` | classifySample com amostra < threshold mas > 0 | Tabela current_impact; histograma oculto |
| `'estimator_only'` | IMPACT_SOURCES[key].key_class === 'estimator_only' (panel_path: '' — ex: auto_assign_family.* role-only) | Mensagem "Análise quantitativa não disponível para esta chave nesta versão"; estimator base ainda exibido |
| `'system_key'` | IMPACT_SOURCES[key].key_class === 'system_key' (chaves operacionais como CURATE_PIPELINE_ENABLED) | Mensagem "Chave de sistema — análise quantitativa não aplicável" |
| `'unsupported'` | Estimator falhou OU impact-source falhou OU ambas as fontes retornaram vazio | Mensagem "Análise quantitativa indisponível para esta janela. Tente outra janela ou prossiga com julgamento qualitativo" + permitir editar |

#### §4.1.10 — Handler do endpoint

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { requireAdmin } from '@/lib/auth/require-admin';
import { IMPACT_SOURCES, type ImpactSource } from '@/lib/admin/pipeline-config-impact-sources';
import { estimateImpact, type ImpactEstimate } from '@/lib/admin/pipeline-impact-estimators';
import { deriveImpactFromDistribution } from '@/lib/admin/limiares/derive-impact-from-distribution';
import { classifySample, SAMPLE_THRESHOLD } from '@/lib/admin/pipeline-config-classify-sample';
import type {
  ImpactPreviewPayload,
  SampleStatus,
  Histogram,
} from '@/lib/admin/pipeline-config-impact-preview-types';
// ImpactSourceResult é o retorno do aggregateImpactSource — definido inline neste mesmo arquivo
type ImpactSourceResult = {
  source: 'dashboard_daily_summary';
  sample_size: number;
  sample_status: SampleStatus;
  current_impact: ImpactPreviewPayload['current_impact'];
  proposed_impact: ImpactPreviewPayload['proposed_impact'];
  histogram: Histogram;
};

const VALID_WINDOWS = { 7: '7 days', 30: '30 days', 90: '90 days' } as const;

export async function POST(req: NextRequest, { params }: { params: Promise<{ key: string }> }) {
  const adminCheck = await requireAdmin(req);
  if (!adminCheck.ok) return adminCheck.response;

  const supabase = adminCheck.admin; // SERVICE_ROLE — bypass RLS de dashboard_daily_summary (sem policy permissiva; sessão retornaria 0 rows silenciosamente)
  const { key } = await params;  // Next 15 mudou o contrato — params agora é Promise

  let body;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'invalid_json' }, { status: 400 });
  }

  const { new_value, days = 30, skip_cache = false } = body;

  if (typeof new_value !== 'string' || new_value.trim().length === 0 || new_value.length > 256) {
    return NextResponse.json({ error: 'new_value_required' }, { status: 400 });
  }
  if (!(days in VALID_WINDOWS)) {
    return NextResponse.json({ error: 'invalid_window', accepted: [7, 30, 90] }, { status: 400 });
  }

  // 1. Resolver current_value
  const { data: currentRow, error: currentErr } = await supabase
    .from('pipeline_config')
    .select('value, updated_at')
    .eq('key', key)
    .single();

  if (currentErr || !currentRow) {
    return NextResponse.json({ error: 'key_not_found' }, { status: 404 });
  }

  const current_value = currentRow.value;

  // E-1 fix: source hoisted para escopo do handler — usado tanto em callImpactSource
  // quanto em composePayload (afetado_label, current_value_filter). IMPACT_SOURCES é o
  // mapa estático cravado em §4.1.4; key foi validada implicitamente via callEstimator/key_not_found.
  const source = IMPACT_SOURCES[key];

  // 2. Promise.allSettled — estimator + impact-source paralelos
  const [estimatorResult, impactResult] = await Promise.allSettled([
    callEstimator(supabase, key, current_value, new_value),
    callImpactSource(supabase, source, current_value, new_value, days),
  ]);

  if (estimatorResult.status === 'rejected') {
    console.warn('[impact-preview] estimator_failed:', estimatorResult.reason);
  }
  if (impactResult.status === 'rejected') {
    console.warn('[impact-preview] impact_source_failed:', impactResult.reason);
  }

  const payload = composePayload({
    key,
    current_value,
    new_value,
    days,
    estimatorResult,
    impactResult,
    source,  // entry de IMPACT_SOURCES — fornece affected_label (pode ser undefined se key não está em IMPACT_SOURCES)
  });

  const headers = new Headers();
  if (skip_cache) {
    headers.set('cache-control', 'no-store');
  } else {
    headers.set('cache-control', 'private, max-age=60');
  }
  return NextResponse.json(payload, { headers });
}

async function callEstimator(supabase, key, currentValue, newValue) {
  // estimateImpact() é o entry point exportado por lib/admin/pipeline-impact-estimators.ts (cleanup v3.4 §3.16)
  // Assinatura real: (admin, key, oldVal, newVal) => Promise<ImpactEstimate>
  // Internamente dispatcha para `estimators` (lowercase, interno) por chave.
  // Note: 'days' NÃO é parâmetro do estimator — é parâmetro do endpoint (afeta apenas a janela de IMPACT_SOURCES).
  return await estimateImpact(supabase, key, currentValue, newValue);
}

async function callImpactSource(supabase, source, currentValue, newValue, days) {
  // source recebido do handler (hoist E-1). Defesas:
  if (!source || source.panel_path === '') return null;

  // Para histórico (janelas 7d/30d/90d), ler de dashboard_daily_summary cross-day.
  // Para sample do dia corrente, invocar RPC limiares_panel_N_snapshot('1 day').
  // Subset de painéis relevantes inferido do panel_path.

  // Detalhes da composição em lib/admin/pipeline-config-impact-aggregator.ts
  // (helper novo desta sprint)
  return await aggregateImpactSource(supabase, source, currentValue, newValue, days);
}

// composePayload e aggregateImpactSource definidos inline com contratos explícitos.
// Implementor não deve inventar contratos — usar as assinaturas abaixo.

/**
 * Compõe o payload final do endpoint impact-preview consolidando estimator + impact-source.
 * Preserva campos pré-existentes da cleanup v3.4 (§3.16) + adiciona campos novos desta sprint.
 *
 * Localização sugerida: lib/admin/pipeline-config-impact-aggregator.ts (helper novo).
 *
 * @param key - chave do pipeline_config sendo editada
 * @param current_value - valor atual
 * @param new_value - valor proposto
 * @param days - janela em dias (7/30/90)
 * @param estimatorResult - PromiseSettledResult de callEstimator()
 * @param impactResult - PromiseSettledResult de callImpactSource()
 * @returns objeto com campos pré-existentes (affected_count, projected_event_cost_usd, cost_window,
 *   cost_is_fallback, panels) + campos novos (window_days, source, sample_size, sample_status,
 *   current_impact, proposed_impact, histogram)
 */
function composePayload({
  key,
  current_value,
  new_value,
  days,
  estimatorResult,
  impactResult,
  source,
}: {
  key: string;
  current_value: string;
  new_value: string;
  days: 7 | 30 | 90;
  estimatorResult: PromiseSettledResult<ImpactEstimate | null>;
  impactResult: PromiseSettledResult<ImpactSourceResult | null>;
  source: ImpactSource | undefined;
}): ImpactPreviewPayload {
  const estimate = estimatorResult.status === 'fulfilled' ? estimatorResult.value : null;
  const impact = impactResult.status === 'fulfilled' ? impactResult.value : null;

  // F-2/F-3 fix: fallback 'unsupported' (alinhado ao SampleStatus de §4.1.5).
  // Quando impact retorna null (estimator-only key, falha de RPC, ou impact-source descartado),
  // emitir status que a UI sabe renderizar — não enum opaco fora do tipo.
  const sample_status: SampleStatus = impact?.sample_status
    ?? (source?.key_class === 'system_key' ? 'system_key'
      : source?.key_class === 'estimator_only' ? 'estimator_only'
      : 'unsupported');

  return {
    // Identidade da chave + valores avaliados
    key,
    current_value,
    new_value,
    window_days: days,

    // Origem da análise quantitativa
    // G-5 fix: source é enum literal, não path. Origem da análise quantitativa derivada de impact.source (sucesso) OU source.key_class (estimator_only/system_key) OU null (unsupported).
    source: impact?.source
      ?? (source?.key_class === 'estimator_only' ? 'estimator_only' as const
        : source?.key_class === 'system_key' ? 'system_key' as const
        : null),

    // Pré-existentes cleanup v3.4 (SEMPRE emitidos)
    affected_count: estimate?.affected_count ?? null,
    affected_label: source?.affected_label ?? null,
    projected_event_cost_usd: estimate?.projected_event_cost_usd ?? null,
    cost_window: estimate?.cost_window ?? null,
    cost_is_fallback: estimate?.cost_is_fallback ?? false,
    panels: source?.panels ?? [],  // J-2 fix v2.5: panels é metadado estático da chave em IMPACT_SOURCES (não output do estimator); ImpactEstimate real NÃO tem este campo

    // Novos desta sprint
    sample_size: impact?.sample_size ?? 0,
    sample_status,
    // F-14 fix: sample_threshold sempre emitido para sample_status que UI mostra tabela; null para system_key/unsupported
    sample_threshold: (sample_status === 'sufficient' || sample_status === 'insufficient' || sample_status === 'estimator_only')
      ? SAMPLE_THRESHOLD
      : null,
    current_impact: impact?.current_impact ?? null,
    proposed_impact: impact?.proposed_impact ?? null,
    // H-2 fix: histogram condicional a sample_status==='sufficient' (wire §4.1.6 documenta omit em outros estados).
    // Sem isso, histogram non-null aparece para insufficient/estimator_only/system_key e UI renderiza distribuição com amostra estatisticamente fraca.
    histogram: sample_status === 'sufficient' ? (impact?.histogram ?? null) : null,
    // H-4 fix: system_key_reason emitido apenas para sample_status==='system_key', derivado do entry IMPACT_SOURCES.
    // Type declara opcional (G-12) e §4.1.8 example emite — handshake fechado consumindo source.system_key_reason.
    ...(sample_status === 'system_key' && source?.system_key_reason ? { system_key_reason: source.system_key_reason } : {}),
  };
}

/**
 * Lê dashboard_daily_summary cross-day (janelas 7d/30d/90d) e invoca RPC
 * limiares_panel_N_snapshot('1 day') para sample do dia corrente.
 * Aplica derivação de impacto via deriveImpactFromDistribution() pré-existente.
 *
 * Localização sugerida: lib/admin/pipeline-config-impact-aggregator.ts.
 *
 * @param supabase - client Supabase autenticado
 * @param source - entry de IMPACT_SOURCES com panel_path
 * @param currentValue - valor atual
 * @param newValue - valor proposto
 * @param days - janela em dias
 * @returns { source, sample_size, sample_status, current_impact, proposed_impact, histogram }
 */
async function aggregateImpactSource(supabase, source, currentValue, newValue, days) {
  // 1. Histórico via dashboard_daily_summary cross-day (filtro por summary_date, não por interval SQL)
  const { data: historical, error: histError } = await supabase
    .from('dashboard_daily_summary')
    .select('summary_date, data')
    .gte('summary_date', new Date(Date.now() - days * 86400000).toISOString().slice(0, 10))
    .order('summary_date', { ascending: true });
  if (histError) {
    console.warn('[impact-preview] aggregator.historical_query_failed:', histError.message);
  }

  // 2. Dia corrente via RPC do panel relevante
  const panelN = parsePanelNFromPath(source.panel_path);
  if (panelN === null) {
    // panel_path mal-formado — sem dia corrente, mas não bloqueia histórico (defesa em profundidade).
    return null;
  }
  const { data: today, error: todayError } = await supabase.rpc(
    `limiares_panel_${panelN}_snapshot`,
    { p_interval: '1 day' },
  );
  if (todayError) {
    console.warn('[impact-preview] aggregator.today_rpc_failed:', todayError.message);
  }

  // 3. Combinar histórico + dia corrente em distribuição agregada e derivar impact
  const distribution = combineDistribution(historical, today, source);

  // J-19 fix v2.5: bucketEdges cobre faixa completa [0, 1] em 10 ranges uniformes ([0.0,0.1], [0.1,0.2], ..., [0.9,1.0]).
  // Producers SQL (§3.5.4/§3.5.7/§3.5.10) emitem bucket ∈ [1,10] via WIDTH_BUCKET(value, 0.0, 1.0, 10) + clamp.
  // Exemplo §4.1.5 mostra `range: [0.50, 0.55]` etc — esses são valores ILUSTRATIVOS de UI rendering com
  // sub-ranges visuais (zoom em subset > 0.5 onde tipicamente residem confidences reais); valor numérico
  // do bucket no payload vem dos edges canônicos abaixo. UI pode reformatar visualmente, mas o contrato é [0,1].
  const bucketEdges: Array<[number, number]> = Array.from({ length: 10 }, (_, i) => [i / 10, (i + 1) / 10]);

  // filter derivado de IMPACT_SOURCES (gte/lte) — controle direcional do impacto
  const filter = source.current_value_filter; // 'gte' | 'lte'

  // Coerção string→number (current_value e new_value chegam como string da request JSON)
  const currentNum = Number(currentValue);
  const newNum = Number(newValue);

  // E-5 fix: classifySample ativado — alinhamento de enum com wire/UI (5 estados: sufficient/insufficient/estimator_only/system_key/unsupported).
  // universeSize é o total amostrado em distribution.total; key_class vem de IMPACT_SOURCES.
  // G-15: UNIVERSE_SIZE_FALLBACK = distribution.total significa que o critério relativo (5% do universo)
  // vira sanity check tautológico quando itemCount > 0 (qualquer valor >= 5% de si mesmo). Aceitável pré-MVP
  // — sem ground truth de universo total fora da janela, usa o próprio total. Threshold absoluto (30) é o gate real.
  const UNIVERSE_SIZE_FALLBACK = distribution.total;
  const sample_status = classifySample(distribution.total, UNIVERSE_SIZE_FALLBACK, source.key_class);

  // E-4 fix: single chamada a deriveImpactFromDistribution + destructure {current, proposed}.
  // Helper §4.4 retorna {current: {items_total, items_passing, ...}, proposed: {items_total, ..., delta_passing, delta_rejecting}}.
  const impact = deriveImpactFromDistribution({
    buckets: distribution.histogram,
    bucketEdges,
    current_value: currentNum,
    new_value: newNum,
    filter,
  });

  return {
    // G-5 fix: source é enum literal (não panel_path string). panel_path interno para roteamento; consumer vê apenas o enum.
    source: 'dashboard_daily_summary' as const,
    sample_size: distribution.total,
    sample_status,
    current_impact: impact.current,    // shape flat: {items_total, items_passing, items_rejecting, pass_rate}
    proposed_impact: impact.proposed,  // shape flat: {items_total, items_passing, items_rejecting, pass_rate, delta_passing, delta_rejecting}
    // E-3 fix: wrappear histogram no shape do wire {buckets, bucket_count} (não array flat).
    // Transformação interna {bucket, cnt} (RPC format) → {range, count} (wire format §4.1.5).
    histogram: {
      buckets: distribution.histogram.map(({ bucket, cnt }) => ({
        range: bucketEdges[bucket - 1],
        count: cnt,
      })),
      bucket_count: 10,
    },
  };
}

/**
 * Extrai o número N do panel a partir de um panel_path como `data.limiares.panel_5.metric_x`.
 * Helper local de roteamento; não tem efeito colateral. Defesa-em-profundidade: callImpactSource
 * já filtra panel_path === '' antes de chamar, mas paths mal-formados (sem padrão panel_N)
 * retornam null em vez de throw.
 *
 * @param panelPath - caminho no formato `data.limiares.panel_N[.subkey]`
 * @returns inteiro N (1-10) ou null se path não bate
 */
function parsePanelNFromPath(panelPath: string): number | null {
  const match = panelPath.match(/panel_(\d+)/);
  if (!match) {
    console.warn('[impact-preview] parsePanelNFromPath: invalid panel_path:', panelPath);
    return null;
  }
  return parseInt(match[1], 10);
}

/**
 * Combina snapshots históricos (multi-dia, vindos de dashboard_daily_summary cross-day)
 * com snapshot single-day (vindo da RPC limiares_panel_N_snapshot('1 day')) em uma única
 * distribuição agregada usada pelo impact-preview.
 *
 * Localização sugerida: lib/admin/pipeline-config-impact-aggregator.ts.
 *
 * Contrato:
 * - **Histogramas diretos (buckets):** painéis 1, 2, 7, 8, 10 já emitem `[{bucket, cnt}]` — somar `cnt` por bucket cross-day + dia corrente; bucket é a chave de agrupamento.
 * - **Listas detalhadas com `similarity` ou `confidence`** (panel_3 merge_candidates `[{id, similarity, detected_at}]`, panel_5 pending stuck, panel_9 history): J-17 fix v2.5 — quando `source.histogram_path` aponta para uma lista (não para um histograma), `combineDistribution` aplica `WIDTH_BUCKET` em JS sobre o campo numérico (similarity para panel_3, confidence_at_promotion para panel_5, etc) para derivar o histograma de 10 buckets. Conversão semântica: cada item da lista vira 1 unidade de count no bucket correspondente.
 * - **Listas sem campo numérico** (panel_9 history): não derivar histograma; aggregate retorna `{total: list.length, histogram: []}` e o caller (composePayload) emite `sample_status: 'estimator_only'` para essas chaves.
 * - **Dedup cross-day:** para listas com `id`, deduplicar; em caso de duplicata cross-day vs single-day, escolher a entry com `detected_at`/`processed_at`/`changed_at` mais recente.
 * - **Métricas escalares** (counts, médias): usar a entry mais recente (single-day) em vez de somar — single-day é a fonte de verdade do estado atual.
 * - **Snapshots vazios:** historical=[] ou today=null → tratar como distribuição vazia (`total=0`, `histogram=[]`); NÃO lançar erro.
 *
 * @param historical - array de rows de dashboard_daily_summary cross-day, cada uma com `.data.limiares.panel_N` no JSONB
 * @param today - payload retornado pela RPC `limiares_panel_N_snapshot('1 day')`; pode ser null se RPC falhou
 * @param source - entry de IMPACT_SOURCES com `panel_path` e demais metadados
 * @returns { total: number, histogram: Array<{bucket: number, cnt: number}> } — distribuição agregada
 *   pronta para deriveImpactFromDistribution() pré-existente
 */
function combineDistribution(
  historical: Array<{ summary_date: string; data: unknown }>,
  today: unknown,
  source: ImpactSource,
): { total: number; histogram: Array<{ bucket: number; cnt: number }> } {
  // G-6 fix: placeholder seguro para TSC strict (function declarada com return value).
  // J-11 fix v2.5 — CRÍTICO de ordering: §4.1.10 chama combineDistribution + distribution.total/histogram
  // imediatamente após (linhas ~3192-3198). Deploy do endpoint em SUB-PR 4 + corpo do combineDistribution
  // em SUB-PR 8 DEVE ser MESMA janela de deploy — senão runtime do impact-preview explode na primeira call.
  // §8 (sequenciamento) crava SUB-PR 4 e SUB-PR 8 como deploy coordenado obrigatório.
  // Contrato acima é a fonte de verdade; implementor de SUB-PR 8 implementa seguindo-o literalmente.
  throw new Error('NOT_IMPLEMENTED — combineDistribution será implementado em SUB-PR 8 conforme contrato JSDoc acima; deploy coordenado com SUB-PR 4 (ver §8)');
}
```

**Critério de aceite (§7.2):** smoke admin via POST com `new_value=0.75` em `skill.hard_gate.min_confidence` retorna payload com `sample_size > 0`, `histogram.buckets.length = 10`, `current_impact + proposed_impact` válidos; admin não-admin recebe 401/403; chave inválida retorna 404.

### §4.2 — Refator `/api/admin/limiares/historical`

**SUB-PR:** 8.

**Caminho:** `app/api/admin/limiares/historical/route.ts`.

Endpoint pré-existente usa `pg.Pool` direto + Redis TTL 300s sobre query live em tabelas operacionais. Esta sprint refatora para consumir `dashboard_daily_summary` cross-day via Supabase client.

```typescript
// Pseudocódigo do refator

export async function GET(req: NextRequest) {
  const adminCheck = await requireAdmin(req);
  if (!adminCheck.ok) return adminCheck.response;
  const supabase = adminCheck.admin; // SERVICE_ROLE — bypass RLS de dashboard_daily_summary (sem policy permissiva; sessão retornaria 0 rows silenciosamente)

  const url = new URL(req.url);
  const range = url.searchParams.get('range') ?? '30d';  // '7d' ou '30d'
  if (range !== '7d' && range !== '30d') {
    return NextResponse.json({ error: 'invalid_range', accepted: ['7d', '30d'] }, { status: 400 });
  }
  const days = range === '7d' ? 7 : 30;

  const since = new Date(Date.now() - days * 24 * 60 * 60 * 1000).toISOString().slice(0, 10);

  // Lê snapshot diário dos 10 painéis do dashboard_daily_summary
  const { data: rows, error } = await supabase
    .from('dashboard_daily_summary')
    .select('summary_date, data')
    .gte('summary_date', since)
    .order('summary_date');

  if (error) {
    console.warn('[limiares/historical] dashboard_summary_query_failed:', error.message);
    return NextResponse.json({ error: 'query_failed' }, { status: 500 });
  }

  // Agrega cross-day os snapshots de cada painel para a janela
  // Cada row.data.limiares.panel_N é mesclado conforme semântica de cada painel
  // (histogramas somam, listas concatenam, etc.)
  const aggregated = aggregateLimiaresHistorical(rows ?? []);

  const headers = new Headers();
  headers.set('cache-control', 'private, max-age=60');
  return NextResponse.json(aggregated, { headers });
}
```

**Função `aggregateLimiaresHistorical`** em `lib/admin/limiares/aggregate-historical.ts` (arquivo novo):

```typescript
/**
 * Mescla snapshots cross-day de `dashboard_daily_summary.data.limiares.panel_N` em uma
 * estrutura única com 10 painéis prontos para consumo pela UI.
 *
 * Contrato:
 * - **Painéis com histograma (1, 2, 7, 8, 10):** somar `cnt` por `bucket` cross-day; bucket é a chave de agrupamento; ordenação preservada.
 * - **Painéis com lista (3, 5, 9):** concatenar listas cross-day + deduplicar por `id`; em caso de duplicata, escolher entry com timestamp (`detected_at`/`processed_at`/`changed_at`) mais recente.
 * - **Painéis 4, 6 (snapshot de estado):** usar entry da linha mais recente (maior `summary_date`) — estado atual de pending/retirement não muda historicamente, não soma cross-day.
 * - **Rows vazias** (`data.limiares` ausente ou painel inexistente no JSONB): tratar como `[]`/`null` sem lançar erro.
 *
 * @param rows - array de `{summary_date, data}` rows de dashboard_daily_summary cross-day
 * @returns objeto `{ panel_1, panel_2, ..., panel_10 }` com shape paritário ao do endpoint live
 */
// J-6 fix v2.5: type definido explicitamente — shape paritary ao RPCs limiares_panel_N_snapshot.
// Cada painel é JSONB unknown (shape específico por painel — ver §3.5.X); aggregator preserva a estrutura
// do RPC original sem reshape. Consumer (UI) já sabe ler cada panel_N pelo shape canônico.
// [VALIDAR EM E0d: confirmar que este nome não conflita com tipos pré-existentes do endpoint limiares/historical]
export type LimiaresHistoricalAggregate = {
  panel_1: unknown;
  panel_2: unknown;
  panel_3: unknown;
  panel_4: unknown;
  panel_5: unknown;
  panel_6: unknown;
  panel_7: unknown;
  panel_8: unknown;
  panel_9: unknown;
  panel_10: unknown;
};

export function aggregateLimiaresHistorical(rows: Array<{ summary_date: string; data: unknown }>): LimiaresHistoricalAggregate {
  // G-6 fix: placeholder seguro para TSC strict (function declarada com return value).
  // Contrato acima é a fonte de verdade; implementor de SUB-PR 8 implementa seguindo-o literalmente.
  throw new Error('NOT_IMPLEMENTED — aggregateLimiaresHistorical será implementado em SUB-PR 8 conforme contrato JSDoc acima');
}
```

**`pg.Pool` direto e Redis custom eliminados** deste endpoint.

**Critério de aceite (§7.2):** GET com `?range=30d` retorna estrutura com 10 painéis preenchidos; `cache-control` é `private, max-age=60`; smoke verifica ausência de import de `pg`/`redis` no arquivo.

### §4.3 — Refator `/api/admin/limiares/online`

**SUB-PR:** 9.

**Caminho:** `app/api/admin/limiares/online/route.ts`.

Endpoint pré-existente usa `pg.Pool` direto + Redis TTL 60s sobre query live com `INTERVAL '1 day'`. Esta sprint refatora para invocar 10 RPCs `limiares_panel_N_snapshot('1 day')` via Supabase client em paralelo.

```typescript
export async function GET(req: NextRequest) {
  const adminCheck = await requireAdmin(req);
  if (!adminCheck.ok) return adminCheck.response;
  const supabase = adminCheck.admin; // SERVICE_ROLE — bypass RLS de dashboard_daily_summary (sem policy permissiva; sessão retornaria 0 rows silenciosamente)

  const panelIds = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  const rpcResults = await Promise.allSettled(
    panelIds.map(n =>
      supabase.rpc(`limiares_panel_${n}_snapshot`, { p_interval: '1 day' })
    )
  );

  const panels: Record<string, unknown> = {};
  rpcResults.forEach((res, idx) => {
    const panelN = panelIds[idx];
    if (res.status === 'fulfilled' && !res.value.error) {
      panels[`panel_${panelN}`] = res.value.data;
    } else {
      const msg = res.status === 'rejected' ? String(res.reason) : res.value.error?.message;
      console.warn(`[limiares/online] panel_${panelN}_failed:`, msg);
      panels[`panel_${panelN}`] = { meta: { error: 'query_failed', message: msg }, series: [] };
    }
  });

  const headers = new Headers();
  headers.set('cache-control', 'private, max-age=60');
  return NextResponse.json(panels, { headers });
}
```

**`pg.Pool` direto e Redis custom eliminados** deste endpoint. Convergência arquitetônica completa: zero `pg.Pool` direto em endpoints admin, zero Redis custom em endpoints admin.

**Critério de aceite (§7.2):** GET retorna estrutura com 10 painéis; falha de RPC individual não derruba os 9 outros; ausência de import de `pg`/`redis`.

### §4.4 — Helper `deriveImpactFromDistribution`

**Caminho novo:** `lib/admin/limiares/derive-impact-from-distribution.ts`.

```typescript
export function deriveImpactFromDistribution(args: {
  buckets: Array<{ bucket: number; cnt: number }>;
  bucketEdges: Array<[number, number]>;  // 10 ranges [min, max]
  current_value: number;
  new_value: number;
  filter: 'gte' | 'lte';
}): {
  current: { items_total: number; items_passing: number; items_rejecting: number; pass_rate: number };
  proposed: { items_total: number; items_passing: number; items_rejecting: number; pass_rate: number; delta_passing: number; delta_rejecting: number };
} {
  const total = args.buckets.reduce((acc, b) => acc + b.cnt, 0);

  const sumPassing = (threshold: number) => {
    return args.buckets.reduce((acc, b) => {
      // Producers (§3.5.4/§3.5.7/§3.5.10) clamp bucket ∈ [1,10] via GREATEST(LEAST(WIDTH_BUCKET, 10), 1).
      // O fallback `?? [0, 1]` abaixo é defesa morta: nunca deve ser ativado em runtime; serve apenas
      // como guard de tipos para o caso (impossível por construção) de bucketEdges não cobrir o índice.
      const [bMin, bMax] = args.bucketEdges[b.bucket - 1] ?? [0, 1];
      const isPassing = args.filter === 'gte' ? bMin >= threshold : bMax <= threshold;
      return acc + (isPassing ? b.cnt : 0);
    }, 0);
  };

  const currentPassing = sumPassing(args.current_value);
  const proposedPassing = sumPassing(args.new_value);

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
      delta_rejecting: currentPassing - proposedPassing,  // simétrico negativo de delta_passing
    },
  };
}
```

**K-1 fix v2.6 — J-14 (cooldown branch) DROPPED:** versões anteriores (v2.5) afirmavam que o helper ramificaria por `source.key` para emitir `items_inside_cooldown` em chaves `*.cooldown_days`. Auditoria revelou que (a) helper signature não suportava `semantic_mode`, (b) aggregator não fazia threading de `source.key`, (c) panel_9/panel_3/panel_10 (rotas das chaves `*.cooldown_days`/`*.lookback_days`) não emitem campo numérico bucketable contra `current_value`. Decisão v2.6: dropar a feature inteira nesta sprint; reclassificar 8 entries (`*.lookback_days`, `*.merge_candidate.*_days`, `*.opus_review.cooldown_days`) como `estimator_only` em §4.1.4. Ver LK-PS-23 para frente futura.

---

## §5 UI

### §5.1 — Integração no modal de edição + extração de componentes (DV-4 opção a)

**Atende:** S-ORCH-9.
**SUB-PR:** 10.
**Pré-requisito:** cleanup v3.4 mergeada — modal de edição existe **inline em `app/admin/pipeline-config/page.tsx`** (não como componente separado). Função local `EditModal(...)` + função local `ImpactPreview(...)` ambas dentro do mesmo arquivo `page.tsx`. Cleanup v3.4 entregou o `ImpactPreview` inline consumindo `affected_count` + `projected_event_cost_usd` + `cost_window` + `cost_is_fallback` + `panels` do endpoint pré-existente.

**Caminhos:**
- `app/admin/pipeline-config/page.tsx` — page existente, contém `EditModal` + `ImpactPreview` inline
- `components/admin/pipeline-config/EditModal.tsx` — arquivo novo (extração)
- `components/admin/pipeline-config/ImpactPreview.tsx` — arquivo novo (extração + expansão)
- `components/admin/pipeline-config/ImpactTable.tsx` — sub-componente novo
- `components/admin/pipeline-config/ImpactHistogram.tsx` — sub-componente novo
- `components/admin/pipeline-config/types.ts` — types compartilhados

#### §5.1.0 — Pré-trabalho obrigatório: extração de componentes inline para arquivos próprios

Estado atual confirmado via grep no SUB-PR 1 (§6.1): tanto `EditModal` quanto `ImpactPreview` estão inline em `app/admin/pipeline-config/page.tsx`. Antes da integração do payload novo, **extrair ambos para arquivos próprios**:

1. Criar `components/admin/pipeline-config/EditModal.tsx` — recebe `configKey`, `currentValue`, `onSave`, `onCancel` como props
2. Criar `components/admin/pipeline-config/ImpactPreview.tsx` — recebe `configKey`, `currentValue`, `newValue` como props
3. Atualizar `app/admin/pipeline-config/page.tsx` para importar e usar os componentes extraídos
4. Migrar types compartilhados para `components/admin/pipeline-config/types.ts`

Custo estimado: ~1-2 horas. Benefícios:

- `page.tsx` que hoje contém tabela de chaves + botões + modal + ImpactPreview fica enxuto
- `ImpactPreview`, que nesta sprint vira componente complexo (histograma SVG, tabela com 3 estados, seletor de janela, render condicional por sample_status), fica em arquivo dedicado fácil de evoluir e testar
- Coerência arquitetônica com o resto do projeto (`components/admin/<feature>/` padrão)

**Notas operacionais:**

- Preservar fielmente o comportamento atual do `EditModal` (estados de loading, validação de input, criticality_level handling, confirmação textual "PUBLICAR" para criticidade high)
- Preservar fielmente o comportamento atual do `ImpactPreview` quanto a campos pré-existentes (`affected_count`, `projected_event_cost_usd`, `cost_window`, `cost_is_fallback`, `panels`) — apenas adicionar consumo dos campos novos do payload

#### §5.1.1 — Decisão arquitetural cravada — adição lateral, não substituição

O modal de edição mantém a estrutura cravada pela cleanup v3.4 — caixa qualitativa pré-existente preservada — e adiciona campos novos ao componente `ImpactPreview` que (após extração) consome o endpoint refatorado em §4.1. O layout passa a ter **dois blocos lado a lado** dentro da seção "Impacto estimado":

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ Impacto estimado                                                              │
│                                                                               │
│ ┌─────────────────────────────────┐  ┌─────────────────────────────────────┐ │
│ │ [BLOCO QUALITATIVO]              │  │ [BLOCO QUANTITATIVO — EXPANDIDO]    │ │
│ │ (pré-existente da v3.4 cleanup)  │  │ (ImpactPreview com payload novo)    │ │
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

- **Bloco qualitativo (esquerda):** preservado da cleanup v3.4. Inclui painéis afetados, observação esperada, horizonte de acompanhamento E linha "Custo Opus projetado (evento único)" quando o estimator pré-existente atender a chave. Para chaves cobertas só por `IMPACT_SOURCES` e sem estimator, a linha de custo é OMITIDA do bloco qualitativo (alinhado à cleanup v3.4 D-PS-67).
- **Bloco quantitativo (direita):** EXPANDIDO desta sprint. Tabela de contagem (current vs proposto + delta) + histograma + seletor de janela. Para chaves só com estimator (`auto_assign_family.*` role-only — `panel_path: ''` em `IMPACT_SOURCES`), bloco quantitativo exibe mensagem "Análise quantitativa não disponível para esta chave nesta versão — caixa qualitativa ao lado orienta o impacto." (status `'estimator_only'`).

**F-15 fix — caixa qualitativa também dentro do `ImpactPreview`:** após extração (§5.1.0), `ImpactPreview` engloba AMBOS os blocos (qualitativo à esquerda + quantitativo à direita) renderizados a partir do MESMO payload do endpoint. Isso evita fetch duplicado em `page.tsx` ou callback wireup entre componentes. Os campos pré-existentes da cleanup v3.4 (`affected_count`, `affected_label`, `projected_event_cost_usd`, `cost_window`, `cost_is_fallback`, `panels`) continuam emitidos pelo endpoint (§4.1.5) e consumidos no bloco qualitativo dentro de `ImpactPreview`. `page.tsx` simplesmente renderiza `<ImpactPreview configKey={...} currentValue={...} newValue={...} />` sem se preocupar com payload.

#### §5.1.2 — Componente `ImpactPreview.tsx` — expansão sobre o pré-existente da v3.4

Componente extraído de `page.tsx` da cleanup v3.4, ampliado para consumir também `current_impact` + `proposed_impact` + `histogram` + `sample_status` + janela [7d/30d/90d]:

```tsx
// components/admin/pipeline-config/ImpactPreview.tsx

import { useState, useEffect } from 'react';
import useSWR from 'swr';
import type { ImpactPreviewPayload } from '@/lib/admin/pipeline-config-impact-preview-types';

type Props = {
  configKey: string;
  currentValue: string;
  newValue: string;
};

const VALID_WINDOWS = [7, 30, 90] as const;

/**
 * Fetcher SWR para chamadas POST com payload JSON.
 * Contrato: recebe tuple [url, init], faz fetch e retorna body parseado.
 * Em caso de status >= 400, lança erro com `.status` para `error?.status` funcionar nos consumers.
 *
 * Localização sugerida: lib/swr/post-fetcher.ts (compartilhável entre componentes que usam POST + SWR).
 *
 * @param tuple - [url, requestInit]
 * @returns body parseado como JSON
 * @throws Error & { status: number } - quando response.ok === false
 */
// J-20 fix v2.5: fetcher generic — caller (useSWR<T>) determina o type via type parameter.
// Sem isso, return type `unknown` força cast em cada consumer; com generic, type-check do useSWR já cobre.
async function swrPOSTFetcher<T = unknown>([url, init]: [string, RequestInit]): Promise<T> {
  const res = await fetch(url, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...(init.headers ?? {}) },
  });
  if (!res.ok) {
    const error = new Error(`POST ${url} failed: ${res.status}`) as Error & { status: number };
    error.status = res.status;
    throw error;
  }
  return res.json() as Promise<T>;
}

export function ImpactPreview({ configKey, currentValue, newValue }: Props) {
  const [selectedDays, setSelectedDays] = useState<7 | 30 | 90>(30);

  // F-1 fix: debounce inline via useEffect+setTimeout (paridade com cleanup v3.4 page.tsx:176-201).
  // Hook useDebounce não existe no repositório; lib/hooks/ contém apenas useDomains.ts.
  // 500ms inline mantém UX cravada pela cleanup v3.4.
  const [debouncedNewValue, setDebouncedNewValue] = useState(newValue);
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedNewValue(newValue), 500);
    return () => clearTimeout(timer);
  }, [newValue]);

  // Endpoint POST pré-existente da cleanup v3.4, agora retorna payload expandido
  const { data, isLoading, error } = useSWR<ImpactPreviewPayload, Error & { status?: number }>(
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

  // F-16 fix: branches por status code (não só 400).
  if (error?.status === 400) {
    return <div className="text-sm text-red-600">Janela inválida.</div>;
  }
  if (error?.status === 401 || error?.status === 403) {
    return <div className="text-sm text-red-600">Sessão expirada ou sem permissão. Recarregue a página.</div>;
  }
  if (error?.status === 404) {
    return <div className="text-sm text-red-600">Chave não encontrada no pipeline_config.</div>;
  }
  if (error && (error.status === undefined || error.status >= 500)) {
    return <div className="text-sm text-red-600">Erro ao calcular impacto. Tente novamente em alguns segundos.</div>;
  }

  if (isLoading) {
    return <div className="text-sm">Calculando impacto...</div>;
  }

  if (!data) return null;

  // F-2/F-3 + Decisão 2b: render explícito por SampleStatus (5 estados — single source of truth de §4.1.5).

  // Caso 1: chave coberta apenas por estimator (auto_assign_family.* — role-only, panel_path: '' em IMPACT_SOURCES)
  if (data.sample_status === 'estimator_only') {
    return (
      <div className="text-sm text-muted">
        Análise quantitativa não disponível para esta chave nesta versão.
        Edição não é bloqueada — use a caixa qualitativa ao lado para orientação.
      </div>
    );
  }

  // Caso 2: chave de sistema (CURATE_PIPELINE_ENABLED, QUARANTINE_EXPIRY_DAYS)
  if (data.sample_status === 'system_key') {
    return (
      <div className="text-sm text-muted">
        {/* H-5 fix: usa system_key_reason emitido pelo composePayload (H-4) com fallback genérico */}
        {data.system_key_reason ?? 'Chave de sistema — não exposta na interface admin pública.'}
      </div>
    );
  }

  // Caso 3 (NOVO desta sprint — Decisão 2b): análise quantitativa indisponível nesta janela
  // (estimator falhou OU impact-source falhou OU ambas as fontes retornaram vazio).
  if (data.sample_status === 'unsupported') {
    return (
      <div className="text-sm text-muted">
        Análise quantitativa indisponível para esta janela. Experimente outra janela (7d/30d/90d)
        ou prossiga com julgamento qualitativo — a edição não está bloqueada.
        <div className="mt-2 flex gap-2 text-xs">
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
      </div>
    );
  }

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
      {/* G-7 fix: BLOCO QUALITATIVO (esquerda) — campos pré-existentes da cleanup v3.4 */}
      <div className="space-y-2 text-sm">
        {data.affected_label && data.affected_count !== null && (
          <div>
            <strong>{data.affected_count.toLocaleString('pt-BR')}</strong> {data.affected_label}
          </div>
        )}
        {data.projected_event_cost_usd !== null && (
          <div className="text-muted">
            Custo Opus projetado (evento único): <strong>US$ {data.projected_event_cost_usd.toFixed(2)}</strong>
            {data.cost_window && ` — ${data.cost_window}`}
            {data.cost_is_fallback && ' (estimativa fallback)'}
          </div>
        )}
        {data.panels.length > 0 && (
          <div className="text-xs text-muted">
            Painéis afetados:
            <ul className="list-disc list-inside">
              {data.panels.map(p => <li key={p}>{p}</li>)}
            </ul>
          </div>
        )}
      </div>

      {/* BLOCO QUANTITATIVO (direita) — campos novos desta sprint */}
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
            {/* G-2 fix: sample_threshold.relative_min_pct é decimal (0.05); UI renderiza como % */}
            {((data.sample_threshold?.relative_min_pct ?? 0.05) * 100).toFixed(0)}% do universo).
            Considere janela maior ou aguarde mais dados.
          </div>
        )}

        {/* Histograma — apenas quando amostra suficiente */}
        {data.sample_status === 'sufficient' && data.histogram && (
          <ImpactHistogram
            buckets={data.histogram.buckets}
            bucketCount={data.histogram.bucket_count}
            currentValue={Number(currentValue)}
            newValue={Number(debouncedNewValue)}
          />
        )}

        <div className="text-xs text-muted">
          Análise sobre janela de {data.window_days} dias. Fonte: {data.source ?? 'indisponível'}.
        </div>
      </div>
    </div>
  );
}
```

#### §5.1.3 — Caixa qualitativa pré-existente — linha de custo Opus

A caixa qualitativa preservada da cleanup v3.4 já renderiza linha de custo Opus quando o estimator pré-existente atende a chave editada. Esta sprint NÃO altera essa linha — terminologia `projected_event_cost_usd` + rótulo "Custo Opus projetado (evento único)" + nota de fallback quando `cost_is_fallback=true` + `cost_window` continuam exatamente como a cleanup v3.4 entregou.

Quando a chave editada está coberta apenas por `IMPACT_SOURCES` (sem estimator pré-existente), a caixa qualitativa **omite** a linha de custo (campos `affected_count` e `projected_event_cost_usd` retornam null no payload — ver §4.1.6). Bloco qualitativo continua mostrando painéis afetados + observação esperada + horizonte de acompanhamento.

#### §5.1.4 — Detalhes técnicos dos sub-componentes

- **`ImpactTable.tsx`**: render simples de current/proposed + % e delta absoluto. Adaptadores semânticos por classe de chave: gate → "Aprovados/Rejeitados"; promotion → "Promovidos/Não promovidos"; merge_candidate → "Pares dentro/fora do threshold"; retirement → "Aposentados/Ativos". K-1 fix v2.6: adapter "Dentro/Fora do cooldown" removido — entries `*.cooldown_days` reclassificadas como `estimator_only` nesta sprint (LK-PS-23 cobre reintrodução futura).
- **`ImpactHistogram.tsx`**: SVG inline manual com 10 buckets + 2 linhas verticais (atual e proposto) sobrepondo o histograma. Sem dependência de chart lib (`recharts`/`chart.js`) — escopo limitado não justifica peso extra. Tooltip nos buckets ao hover (ex: "0.70-0.75: 1.245 items, 92% aprovados"). Recebe prop `bucketCount` (sempre 10 nesta versão) e renderiza legenda explícita "Distribuição em {bucketCount} buckets" abaixo do SVG — uso ativo do campo do wire em vez de dead field. Entrega no MVP.
- **Debounce inline (500ms)**: paridade com a cleanup v3.4 que faz `useEffect+setTimeout(..., 500)` em `page.tsx:176-201`; mantido inline em vez de hook (decisão F-1 — não criar `lib/hooks/use-debounce.ts` para evitar deliverable extra).
- **Cancelamento de request inflight**: SWR usa cache key como mecanismo de deduplicação — quando `selectedDays` ou `debouncedNewValue` muda, a SWR cria nova entrada de cache e a anterior fica órfã (request anterior pode completar mas é descartada pela UI ao receber payload novo). **Decisão consciente vs `AbortController`**: a cleanup v3.4 usa `AbortController` em `app/admin/pipeline-config/page.tsx` (linhas ~186, ~193, ~200) com `fetch` direto — padrão diferente do SWR que esta sprint adota. SWR não expõe `AbortController` no contrato do fetcher (`(arg: SWRRequestArgs) => Promise<Data>`); cancelamento via SWR é feito por `mutate(key, undefined, { revalidate: false })` ou simplesmente trocando a key. Para o escopo desta sprint (modal admin com debounce 500ms + cache key SWR), o overhead de wrappear um `AbortController` no fetcher não se justifica — race condition residual é cosmética (UI renderiza o último payload recebido, indistinguível visualmente).
- **Cache via SWR**: `revalidateOnFocus: false` para evitar re-fetch ao voltar para a aba; sem `dedupingInterval` agressivo porque a chave SWR inclui `selectedDays`.
- **Botão de salvar do modal**: NÃO é bloqueado em nenhum cenário — admin tem autonomia para decidir mesmo com sinal incompleto. Confirmação textual "PUBLICAR" para criticidade high preservada da cleanup v3.4.

#### §5.1.5 — Comportamento de erro

- HTTP 400 (`invalid_window`): mensagem "Janela inválida" em vermelho.
- HTTP 404 (`key_not_found`): mensagem "Chave não encontrada — verifique o nome".
- HTTP 401/403: a UI não chega aqui (modal abre apenas para admin), mas se chegar, mensagem genérica "Sem permissão".
- Erro de rede / timeout: estado loading indeterminado; SWR retry default; após 3 tentativas, mensagem "Falha ao calcular impacto. Tente novamente.".

**Critério de aceite (§7.2):** smoke E2E em pré-prod abre modal de edição, digita `new_value`, vê histograma + tabela de impacto + linha de custo Opus na caixa qualitativa, alterna janela 7d/30d/90d com nova chamada disparada em cada troca, salva edição. `EditModal` e `ImpactPreview` em arquivos próprios sob `components/admin/pipeline-config/`. Smokes S7-S11 verdes.

### §5.2 — Refator UI dos painéis 2.1, 2.7, 2.8, 2.9 (Frente B parte UI) + 10 painéis Limiares

**Atende:** consumo dos campos novos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` + família `limiares.panel_1..panel_10` materializados em `dashboard_daily_summary.data` pela expansão de §3.6.
**SUB-PR:** 11 (sequencial após SUB-PRs 7, 9 e 10).
**Pré-requisito:** SUB-PR 7 mergeado e backfill rodado — todos os campos novos preenchidos em 26 dias rolling. SUB-PR 9 mergeado — endpoint Limiares online ativo via RPCs. SUB-PR 10 mergeado — impact-preview ativo como consumidor.

#### §5.2.1 — Decisão arquitetural cravada — adição lateral, não substituição

Os 3 painéis O7/O8/O9 hoje (`components/admin/dashboard/OperationalTab.tsx` ou similar — confirmar via E0a em §6.1) consomem exclusivamente `data.operational.o7`, `data.operational.o8`, `data.operational.o9` (todos role-only). Refator desta sprint **não toca** esses campos pré-existentes, apenas adiciona consumo dos campos novos paralelos:

| Painel | Campo pré-existente (mantido intacto) | Campo novo (consumido lateralmente) |
|---|---|---|
| 2.1 KPI "Habilidades Canônicas" | `live.operational.skillsActive` + delta `skills_active_added` | adicional `data.operational.items_processed` no tooltip |
| 2.7 O7 (drift) | `data.operational.o7.{canonicals_novos, vagas_curadas, drift_percent}` | `data.operational.o7_skill.{canonicals_novos, items_processed, drift_percent}` |
| 2.8 O8 (Caminho de resolução) | `data.operational.o8.{camada0, camada1, llm_direct, quarantined, total}` | `data.operational.o8_skill.{slug_match, alias_match, llm_new, race_recovered, gate_rejected, fallback_error, total}` |
| 2.9 O9 (Saúde) | `data.operational.o9.{quarantined, disobeyed, sem_canonico, total}` | `data.operational.o9_skill.{fallback_error, sem_canonical, total}` (status_summary calculado no render layer) |

Critério de design (cravado): cada painel ramifica em **dois blocos lado a lado ou empilhados** (decisão por painel, conforme espaço visual disponível em cada layout), com header indicando "Funções" e "Habilidades" no respectivo bloco. Sem toggle, sem opção de view — ambas as séries são sempre visíveis. Justificativa: a paridade role↔skill é exatamente o que admin precisa enxergar para validar a saúde simétrica do pipeline pós-MVP.

#### §5.2.2 — Painel 2.1 — KPI novo "Items processados pelo pipeline"

A linha 2.1 hoje tem 5 KPIs (Vagas curadas, Saldo Fantastic Jobs, Uploads, Créditos consumidos, Habilidades Canônicas).

Decisão: **adicionar tooltip ao KPI "Habilidades Canônicas" pré-existente** mostrando volume FORI + FOSI no período (não criar 6º card que quebraria o layout 5-colunas pré-existente). Header do KPI mantido como está. Tooltip aparece on-hover com texto:

```
Items processados pelo pipeline na janela
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

Cores: ambos os blocos usam mesma palette pré-existente (Verde < 10%, Âmbar 10-20%, Vermelho > 20%). Pode haver divergência entre os dois (role verde + skill âmbar é cenário possível e informativo).

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
- `gate_rejected` (laranja escuro) — NÃO exibido na barra principal (faz parte do "sem canônico", aparece no 2.9)
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

**Worst-of-N** define cor do badge skill (análogo ao worst-of-three pré-existente do role). **Cálculo do worst-of-N é no render layer**, NÃO no aggregator. Aggregator materializa counters puros (`fallback_error`, `sem_canonical`); UI aplica o algoritmo de traffic-light.

#### §5.2.6 — Refator UI dos 10 painéis Limiares

Painéis Limiares passam a consumir 2 fontes:

- **Janela histórica (7d/30d)** — `GET /api/admin/limiares/historical?range=<range>` retorna mesclado dos snapshots em `dashboard_daily_summary.data.limiares.panel_N` cross-day
- **Janela 24h (online)** — `GET /api/admin/limiares/online` invoca as 10 RPCs `limiares_panel_N_snapshot('1 day')`

Componentes UI dos 10 painéis mantêm contrato visual pré-existente. O fetcher dentro de cada componente é refatorado para consumir a estrutura nova do JSONB (paths `data.limiares.panel_N.<sub_chave>`).

Particularidades por painel:

- **Painel 1 (Hard Gate)** — exibe 3 séries paralelas (role events, skill events, skill FOSI) com tooltip explicando divergência quando houver
- **Painel 8 (creation_confidence)** — ramifica em modo "Por entidade" (skill/role — comportamento pré-existente) e modo novo "Por path" (slug_match / alias_match / llm_new / race_recovered) via toggle dentro do painel
- **Painel 3 (merge candidates)** — lista cross-day com dedup por `id`
- **Painel 5 (pending stuck >30d)** — snapshot mais recente da janela (não acumula histórico — estado é o que importa)
- **Painel 9 (audit trail pipeline_config_history)** — lista cronológica concatenada cross-day, ordenada por `changed_at` DESC (J-18 fix v2.5: paritary à RPC `limiares_panel_9_snapshot` em §3.5.11; tabela `pipeline_config_history` usa `changed_at` como anchor temporal, não `created_at`)

#### §5.2.7 — Comportamento histórico

Em janelas 7d/30d (`days=7` ou `days=30`), todos os painéis somam cross-day os campos respectivos do JSONB. Em janela 24h, lê o JSONB do dia atual via RPCs `limiares_panel_N_snapshot('1 day')`.

Para dias anteriores ao SUB-PR 4 (FOSI vazia), valores skill aparecem como 0 (não como erro). Tooltip nos containers skill em janelas que cobrem esses dias informa: "Habilidades só passaram a ser instrumentadas em <data SUB-PR 4 deploy>. Valores antes dessa data refletem zero por design, não problema operacional."

#### §5.2.8 — Layout responsivo

Em viewport < 768px (mobile/tablet), os 4 painéis O7/O8/O9 colapsam de grid 2-colunas para empilhamento vertical (role em cima, skill embaixo) preservando legibilidade. Componente atual da `OperationalTab` provavelmente já tem grid responsivo — adaptação trivial.

Painéis Limiares mantêm comportamento responsivo pré-existente; refator desta sprint é exclusivamente de fonte de dados, não de layout visual.

#### §5.2.9 — Resumo de mudanças por painel

| Painel | Esforço | Risco visual |
|---|---|---|
| 2.1 (tooltip) | ~30 min | Mínimo — adição de tooltip ao card existente |
| 2.7 (refator container) | ~2h | Baixo — replica estrutura existente em grid 2-col |
| 2.8 (segunda barra + legenda 5 estados) | ~3h | Médio — nova legenda; testar contraste e text-overflow |
| 2.9 (refator container) | ~1.5h | Baixo — replica estrutura existente em grid 2-col |
| 10 painéis Limiares (refator fetcher) | ~4h | Baixo — contrato visual preservado |

Esforço total: ~11h coding + ~3h testes visuais + ajustes = ~1.75 dia. Estimativa SUB-PR 11 = 2-2.5 dias com folga para revisão.

**Critério de aceite (§7.2):** painéis 2.1/2.7/2.8/2.9 + 10 painéis Limiares renderizam em pré-prod com dados sintéticos populados via aggregator + backfill; smoke verifica ausência de chamadas a endpoints removidos (`pg.Pool` + Redis); layout responsivo preservado em <768px.

---

## §6 Auditorias operacionais

### §6.1 — Auditoria de scripts ad-hoc + grep de call sites + verificação de helpers

**SUB-PR:** 1.

Pré-requisito da Frente A é evidence ground truth (E0) coletado em pre-checks paralelos. Antes do SUB-PR 1 começar, rodar os blocos abaixo e produzir relatório `AUDIT-orchestrator-symmetry-v2.8-<data>.md`:

#### E0a — Nome real da função de descoberta no codebase

```bash
grep -rn "discoverAndLinkSkills\|safeDiscoverAndLinkSkills" lib/pipeline/ --include="*.ts"
```

Esperado: `discoverAndLinkSkills` em `lib/pipeline/ingest-job-and-discover-skills.ts:48`, retorna `Promise<DiscoverResult>`; `safeDiscoverAndLinkSkills` em `lib/pipeline/persist-curation/skill-mapper.ts:37`, wrapper void. Caller em `lib/pipeline/persist-curation/persist-fn.ts:410` já em `await`.

#### E0b — Estado das tabelas centrais

```sql
SELECT to_regclass('public.function_orchestrator_items') AS legacy,
       to_regclass('public.function_orchestrator_role_items') AS post_rename,
       to_regclass('public.function_orchestrator_skill_items') AS new_skill,
       to_regclass('public.function_orchestrator_runs') AS runs;
```

Esperado conforme SUB-PR aplicado:
- Antes do SUB-PR 1: legacy=válido, post_rename=NULL, new_skill=NULL, runs=válido
- Pós SUB-PR 1: + new_skill=válido
- Pós SUB-PR 2: legacy=NULL, post_rename=válido, new_skill=válido

#### E0c — Shape de `function_orchestrator_runs`

```sql
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_schema='public' AND table_name='function_orchestrator_runs'
ORDER BY ordinal_position;
```

Confirmar 24 colunas atuais. Pós SUB-PR 1: 29 colunas (24 + 5 `skills_*`).

#### E0d — Callers de `discoverSkillsFromFunction` / `function_orchestrator_items` + audit de helpers TS

```bash
# 1. Call sites pré-rename
grep -rn "discoverSkillsFromFunction\|function_orchestrator_items" \
  lib app scripts tests --include="*.ts"

# 2. G-9 fix: audit de classifyError — confirmar se função existe em lib/pipeline/batch-processor/error-handlers.ts
grep -rn "export.*classifyError\|function classifyError" lib/pipeline/batch-processor/
# Resultado decidido na auditoria pré-SUB-PR 3: (a) se existe → reuse direto; (b) se não existe → criar em error-handlers.ts paritary a estrutura existente

# 3. G-4 fix: audit de DiscoverResult consumers pré-rename
grep -rn "DiscoverResult[^D]" lib/ app/
# Resultado: cada consumer atualizado para DiscoverResultDetailed.aggregate com campos renomeados
# (skills_discovered → extracted, skills_reused → reused, skills_pending_created → pending_created, skills_blocked → gate_rejected)
```

Identificar todos os call sites pré-rename + decidir reuse vs criação de `classifyError` + mapear consumers de `DiscoverResult`. Auditoria §6.2 cobre o rewrite mecânico.

#### E0e — Funções dependentes do nome antigo

```sql
SELECT proname, pg_get_functiondef(oid)
FROM pg_proc
WHERE pg_get_functiondef(oid) ILIKE '%function_orchestrator_items%'
  AND proname NOT IN ('reconcile_canonical_role', 'reconcile_canonical_skill');
```

Identificar todas as funções SQL que referenciam o nome antigo. Cobertura em §6.2.

#### E0f — Estado atual de `dashboard_daily_summary`

```sql
SELECT COUNT(*) AS day_count,
       MIN(summary_date) AS oldest,
       MAX(summary_date) AS newest,
       AVG(pg_column_size(data)) AS avg_jsonb_bytes
FROM dashboard_daily_summary;
```

Confirmar volume atual (~26 rows rolling) + tamanho médio do JSONB para dimensionar impacto da expansão (Frente B).

#### E0g — Validação de eventos `canonical_*_created`

```sql
SELECT event_name, COUNT(*) AS total
FROM events
WHERE event_name LIKE 'canonical_%_created'
GROUP BY event_name;
```

Confirma nome canônico `canonical_role_created` (cravado pela cleanup v3.4). `canonical_skill_created` é o nome esperado paritário a ser emitido onde skill canonical for criado (via trigger ou função).

#### E0h — Auditoria de cobertura paritária de `handleItemError` callers

Listar todos os call sites de `handleItemError` no pipeline de role:

```bash
grep -rn "handleItemError(" lib/pipeline/ --include="*.ts" | grep -v "^lib/pipeline/batch-processor/error-handlers.ts:"
```

Saída esperada: lista de arquivos:linhas com invocações de `handleItemError`. Cada call site precisa de invocação paritária de `handleSkillItemError` no pipeline de skill — instalar via SUB-PR 3 §3.2.

Documentar em `AUDIT-orchestrator-symmetry-v2.8-<data>.md` uma tabela com:
- Call site role (arquivo:linha)
- Equivalente skill esperado (arquivo:linha)
- Status: instalado / não-aplicável (justificar) / pendente

#### E0i — Auditoria de `event_names` ativos no DB para whitelist

**Ground truth:** DB tem **77 `event_names` distintos**.

Listar todos os `event_names` ativos no DB para construir whitelist exaustiva do `CHECK constraint` novo em §2.y:

```sql
SELECT DISTINCT event_name
FROM events
ORDER BY event_name;
```

Saída esperada: **77 valores únicos** (já inclui os 2 nomes legacy `canonical_promoted_dynamic` e `canonical_creation_blocked_low_confidence`, mantidos para rows `entity_type='unknown'`; e também `entity_type_backfilled` emitido pelo SUB-PR 0 §2.w antes da migration de §2.y rodar). Cravar lista COMPLETA literal na cláusula `event_name IN (...)` do `ALTER TABLE events ADD CONSTRAINT events_event_name_check` em §2.y, somando aos 77:
- Os **4 nomes paritários novos** desta sprint (`role_promoted_dynamic`, `skill_promoted_dynamic`, `role_creation_blocked_low_confidence`, `skill_creation_blocked_low_confidence`)
- Os **2 nomes de triggers** `canonical_role_created`, `canonical_skill_created` (cravados em §2.z)

Total: **77 + 4 + 2 = 83 valores** na whitelist. Legacy já contam nas 77; `entity_type_backfilled` também (emitido pelo SUB-PR 0 antes do snapshot da auditoria E0i, então aparece no SELECT DISTINCT).

**Entregável obrigatório (artefato anexo):** SUB-PR 1 produz arquivo `EVENT_NAMES-orchestrator-symmetry-v2.8.md` com:
1. Saída literal do `SELECT DISTINCT event_name FROM events ORDER BY event_name;` (77 nomes)
2. Os 6 nomes novos adicionados manualmente (4 paritários + 2 triggers)
3. Lista consolidada de 83 valores formatada como cláusula `IN (...)` pronta para colar em §2.y

Implementor da migration §2.y cola a lista literal do artefato antes de aplicar. Sem isso, CHECK violation ocorrerá em rows pré-existentes não capturadas.

**Validação adicional pré-migration:** `SELECT COUNT(*) FROM events WHERE event_name NOT IN (<whitelist completa de 83>)` deve retornar 0. Se retornar > 0, há nome ativo que não foi capturado na whitelist e CHECK violation ocorrerá na migration de §2.y. Re-rodar E0i e re-cravar artefato.

#### E0j — Auditoria de índices `fo_items_*` em `function_orchestrator_role_items`

Listar todos os índices reais da tabela para validar a lista de rename de §2.6a antes de aplicar:

```sql
SELECT indexname
FROM pg_indexes
WHERE schemaname='public'
  AND tablename='function_orchestrator_role_items'
ORDER BY indexname;
```

Ground truth: DB tem **6 índices** com prefixo `fo_items_*` (a tabela ainda se chama `function_orchestrator_items` antes do rename — adaptar `tablename` conforme estado pré ou pós SUB-PR 2):

1. `fo_items_canonical_proposed_idx`
2. `fo_items_canonical_role_id_idx`
3. `fo_items_job_posting_created_idx`
4. `fo_items_job_posting_id_idx`
5. `fo_items_run_id_idx`
6. `fo_items_status_idx`

Validação cruzada com §2.6a: a lista de `ALTER INDEX` deve conter exatamente esses 6 nomes. Se a saída de E0j diferir (índice novo criado em sprint paralela ou removido), atualizar §2.6a antes da migration. Nomes `fo_items_pipeline_stage_idx` e `fo_items_created_at_idx` não existem no DB; refutação documentada em §2.6a.

**Entrega:** `AUDIT-orchestrator-symmetry-v2.8-<data>.md` com saída de cada E0a-E0k + decisão de Go/No-Go para cada SUB-PR baseado em estado atual do banco.

#### E0k — Auditoria de CHECK constraints sobre `status` e `pipeline_stage` em FORI (paritary §0.2.12)

Validar valores literais das CHECK constraints pré-existentes antes de cravar qualquer `CREATE TYPE` em §2.x:

```sql
-- 1. Listar CHECK constraints relevantes na tabela FORI (pré-rename, ainda function_orchestrator_items)
SELECT conname, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'public.function_orchestrator_items'::regclass
  AND contype = 'c'
  AND (conname LIKE '%status%' OR conname LIKE '%stage%' OR conname LIKE '%pipeline%')
ORDER BY conname;
```

**Ground truth confirmado v2.7:**
- `function_orchestrator_items_status_check`: `CHECK ((status = ANY (ARRAY['success'::text, 'fallback'::text, 'low_quality'::text, 'pending_review'::text, 'failed'::text])))` — **5 valores**
- TS paritary: `lib/pipeline/types.ts:167-172` exports `ItemStatus = 'success' | 'fallback' | 'low_quality' | 'pending_review' | 'failed'`
- Spec §2.x linha 437-444: ENUM `role_item_status` cravado em v2.7 com OS MESMOS 5 valores (L-1 fix)

**Gate de bloqueio §2.x:** se saída de E0k diferir do esperado acima (CHECK adicionado/removido em sprint paralela), atualizar §2.x ANTES de criar o ENUM. M-2 fix v2.8: nomes de colunas em §1.2 D-PS (`curated`/`curated_fallback`/etc) JÁ CONFIRMADOS via psql como bare-named (sem prefixo `roles_`) — sem necessidade de revalidação nesta etapa.

#### §6.1.1 — Procedimento geral de auditoria

Além dos blocos E0a-E0k, rodar grep sistemático para mapear todos os pontos de toque desta sprint:

```bash
# 1. Verificação de callers da função de descoberta (pré-requisito SUB-PR 3)
grep -rn 'discoverAndLinkSkills\|safeDiscoverAndLinkSkills' lib/ app/ \
  --include='*.ts' > /tmp/audit-orchestrator.txt

# 2. Verificação de INSERTs em function_orchestrator_items (pré-rename SUB-PR 2)
grep -rn "function_orchestrator_items" lib/ app/ scripts/ tests/ \
  --include='*.ts' --include='*.sql' >> /tmp/audit-orchestrator.txt

# 3. Verificação de finalizers de run (pré-requisito SUB-PR 4)
grep -rn 'function_orchestrator_runs.*UPDATE\|finalizeFORun\|finalizeEmptyRun' lib/ app/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt

# 4. Verificação de helpers (classifyError, validateCanonicalConfidence)
grep -rn 'classifyError\|validateCanonicalConfidence' lib/ app/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt

# 5. Verificação de skills passadas/rejeitadas pelo gate
grep -rn 'skill.hard_gate' lib/ app/ --include='*.ts' >> /tmp/audit-orchestrator.txt

# 6. Verificação do skill-type-guard pré-existente
grep -rn 'skill-type-guard\|assertSkillType\|needsReview' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 7. Verificação dos estimators pré-existentes (cleanup v3.4 §3.16)
grep -rn 'pipeline-impact-estimators\|projected_event_cost_usd' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 8. Verificação do cache pré-existente do impact-preview (pré-requisito SUB-PR 10 — REMOÇÃO)
# Confirmar localização atual do cache in-process inline em route.ts para REMOÇÃO
grep -rn 'impactPreviewCache\|Map<string,.*CacheEntry>\|pipeline-impact-preview-cache' lib/ app/ --include='*.ts' \
  >> /tmp/audit-orchestrator.txt

# 9. Auditoria EditModal + ImpactPreview inline (pré-requisito SUB-PR 10)
# Confirmar que ambos estão INLINE em page.tsx (não como componentes separados)
grep -rn 'EditModal\|ImpactPreview' app/admin/pipeline-config/ components/admin/pipeline-config/ \
  --include='*.tsx' --include='*.ts' >> /tmp/audit-orchestrator.txt

# 10. Auditoria padrão Limiares pré-existente (pré-requisito SUB-PRs 8 e 9 — ELIMINAÇÃO)
# Confirmar localização de pg.Pool, redis-cache e queries pré-existentes para REMOÇÃO
grep -rn 'pg.Pool\|new Pool\|redis-cache\|INTERVAL_SQL\|limiares' lib/admin/limiares/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt

# 11. Auditoria aggregateDayData pré-existente (pré-requisito SUB-PR 7)
# Confirmar estrutura das famílias do JSONB
grep -rn 'aggregateDayData\|dashboard-day-aggregator\|dashboard_daily_summary' lib/admin/ app/api/ \
  --include='*.ts' >> /tmp/audit-orchestrator.txt
```

**Decisão por categoria (entrega em `AUDIT-orchestrator-symmetry-v2.8-<data>.md`):**

| Categoria | Decisão |
|---|---|
| CRONs em `app/api/cron/` que escrevem em FO_items | adicionar escrita simétrica em FO_skill_items se invocam função de descoberta; ignorar se apenas tocam FOR (que ganha colunas via §2.2) |
| Scripts em `scripts/` | se for backfill histórico, considerar script-pair; se for teste/diagnóstico, ajustar tabela nova |
| Migrations antigas (`supabase/migrations/*.sql`) | ignorar (fotografias históricas; não rerodam) |
| Testes em `tests/` | atualizar fixtures e expects para refletir nome novo da tabela e tabela nova `function_orchestrator_skill_items` |
| Path quarantined (`mapSkillsToCanonical`) | confirmar que NÃO é tocado nesta sprint — D-PS-83 |
| Helpers `classifyError`, `lookup_canonical_skill_by_normalized_alias`, `resolve_active_canonical_by_slug` | confirmar existência; se ausente, escalar para PO antes de SUB-PR 3 |
| Skill-type-guard | confirmar que `needsReview` está sendo propagado no tipo `RawSkill` consumido pela função de descoberta; se não, ampliar tipo no SUB-PR 3 |
| Estimators pré-existentes (cleanup v3.4 §3.16) | confirmar import path canônico de `pipeline-impact-estimators.ts` para reaproveitamento no endpoint refatorado (§4.1) |
| Cache in-process pré-existente do impact-preview | confirmar localização inline em `route.ts` para REMOÇÃO no SUB-PR 10 (substitui por `cache-control: max-age=60` HTTP) |
| EditModal + ImpactPreview inline | confirmar que ambos estão inline em `app/admin/pipeline-config/page.tsx`; extração para `components/admin/pipeline-config/EditModal.tsx` e `ImpactPreview.tsx` é pré-trabalho do SUB-PR 10 |
| Padrão Limiares (`pg.Pool` + Redis) | confirmar localização dos helpers compartilhados em `lib/admin/limiares/_shared/` (redis-cache, INTERVAL_SQL allowlist, pool config) para REMOÇÃO nos SUB-PRs 8 e 9 |
| `aggregateDayData` em `dashboard-day-aggregator/aggregator.ts` | confirmar estrutura das famílias pré-existentes no JSONB (`operational`, `ai`, `resources`, `communications`, `auth`, `errors`, `accumulators`); confirmar que `operational` é onde campos `o7`/`o8`/`o9` role pré-existentes residem — onde os novos `o7_skill`/`o8_skill`/`o9_skill`/`items_processed` devem ser inseridos por simetria (SUB-PR 7); confirmar que `limiares` é família NOVA a ser criada |

**Gate de bloqueio:** SUB-PR 3 só pode iniciar após E0a-E0d ser registrado (E0a confirma Cenário B vigente — ver §3.1.0). SUB-PR 7 só pode iniciar após E0f-E0g + grep 11 ser registrado, com mapa das famílias pré-existentes do `aggregateDayData`. SUB-PRs 8/9 só podem iniciar após grep 10 ser registrado, com localização dos helpers `pg.Pool`/Redis identificada para REMOÇÃO. SUB-PR 10 só pode iniciar após grep 8 + 9 serem registrados, com localização do cache in-process inline e dos componentes `EditModal`/`ImpactPreview` inline confirmadas.

### §6.2 — Auditoria de funções/triggers que referenciam `function_orchestrator_items` por NOME

**SUB-PR:** 1.

Auditoria sistemática complementar ao E0e, garantindo que o rename de §2.6 atualize TODOS os corpos de funções e triggers que referenciem o nome antigo.

#### Categorias a cobrir

```bash
# 1. Funções SQL no schema public + internal
grep -rn "function_orchestrator_items" supabase/migrations/ --include="*.sql"

# 2. Triggers nomeados (prefixo real é trg_foi_*)
psql -c "SELECT tgname FROM pg_trigger WHERE tgname LIKE 'trg_foi_%';"

# 3. Índices históricos com nomes pré-rename (real prefix fo_items_*)
psql -c "SELECT indexname FROM pg_indexes WHERE tablename='function_orchestrator_role_items' AND (indexname LIKE 'fo_items_%' OR indexname LIKE 'function_orchestrator_items_%');"

# 4. Scripts ad-hoc em scripts/
grep -rn "function_orchestrator_items" scripts/ --include="*.ts" --include="*.sql"

# 5. Testes em tests/
grep -rn "function_orchestrator_items" tests/ --include="*.ts"
```

**Ground truth pré-cravado via §6.1 E0e (4 funções confirmadas no DB):**

| # | Schema | Função | proconfig atual | Notas |
|---|---|---|---|---|
| 1 | `internal` | `reset_taxonomy_core` | (validar via §6.1) | **Namespace internal**, não public — alvo de rewrite |
| 2 | `public` | `cleanup_batch_items` | `search_path=public` (sem `pg_temp`) | **Preservar proconfig original exatamente** no rewrite |
| 3 | `public` | `fn_recompute_jcr_confidence_median` | `search_path=public, pg_temp` | Rewrite mecânico cravado em §2.6b |
| 4 | `public` | `merge_canonicals` | (validar via §6.1) | Alvo de rewrite |
| 5 | `public` | `release_quarantined_jobs_limited` | (validar via §6.1) | Alvo de rewrite |

**Funções alegadas em sessões anteriores e refutadas pelo ground truth (NÃO existem no DB):** `maintenance_phase_1`, `process_opus_create_new`. Removidas da lista de alvos.

**Decisão por categoria:**

| Categoria | Decisão |
|---|---|
| 5 funções SQL com nome antigo no body (4 + 1 namespace internal) | rewrite mecânico via `CREATE OR REPLACE` em §2.6b, preservando proconfig original específica de cada uma — diff mínimo |
| Migrations antigas (`supabase/migrations/*.sql`) | ignorar (fotografias históricas; não rerodam) |
| Scripts em `scripts/` ad-hoc | se for backfill histórico, atualizar para nome novo; se for teste/diagnóstico, idem |
| Testes em `tests/` | atualizar fixtures e expects para refletir nome novo |
| Triggers nomeados `trg_foi_*` | rename para `trg_fori_*` via `ALTER TRIGGER ... RENAME TO ...` (já cravado em §2.6a) |
| Path quarantined (`mapSkillsToCanonical`) | confirmar que NÃO é tocado — D-PS-83 |

**Entrega:** relatório consolidado dentro do `AUDIT-orchestrator-symmetry-v2.8-<data>.md` com cada hit + decisão + snapshot de `proconfig` de cada uma das 5 funções para garantir preservação no rewrite.

---

## §7 Evidence e smoke tests

### §7.1 — Blocos de evidence pré e pós-aplicação

#### E1 — Pré-aplicação de cada migration SQL

Antes de cada migration em §2, capturar snapshot:

- `pg_dump --schema-only` da tabela alvo
- `SELECT pg_get_functiondef(oid)` de funções alvo
- `SELECT * FROM pg_constraint WHERE conrelid = '<table>'::regclass`
- `SELECT * FROM pg_indexes WHERE tablename = '<table>'`

Arquivar como `evidence/before/<sub_pr>/`.

#### E2 — Pós-aplicação

Mesmas queries após `COMMIT`. Diff vs antes deve refletir exatamente as mudanças cravadas na migration (nada a mais, nada a menos).

#### E3 — Smoke de path completo Flow A admin

```sql
-- Pré-condição: 1 vaga sintética inserida em job_postings com curation_status='pending'
-- e dados mock que disparam o pipeline

-- Trigger curate-job-postings manualmente via API
-- POST /api/admin/curate-job-postings com job_ids=[<uuid>]

-- Pós-condição:
SELECT
  -- FORI ganhou item
  (SELECT COUNT(*) FROM function_orchestrator_role_items WHERE job_posting_id = '<uuid>') AS fori_items,
  -- FOSI ganhou items por skill
  (SELECT COUNT(*) FROM function_orchestrator_skill_items WHERE job_posting_id = '<uuid>') AS fosi_items,
  -- FOR ganhou contadores atualizados
  (SELECT row_to_json(r) FROM function_orchestrator_runs r WHERE r.id = '<run_id>') AS run_data;
```

Esperado: `fori_items >= 1` (1 role por vaga), `fosi_items >= 1` (1 ou mais skills por vaga), `run_data.skills_extracted = fosi_items`.

#### E4 — Smoke de idempotência FOSI + FORI

```sql
-- Inserir item duplicado deve falhar UNIQUE constraint sem corromper estado
BEGIN;
INSERT INTO function_orchestrator_skill_items
  (run_id, job_posting_id, raw_item_index, skill_raw_name, skill_raw_type, pipeline_stage, status)
VALUES
  ('<run_uuid>', '<job_uuid>', 0, 'python', 'hard', 'slug_match', 'success');
INSERT INTO function_orchestrator_skill_items
  (run_id, job_posting_id, raw_item_index, skill_raw_name, skill_raw_type, pipeline_stage, status)
VALUES
  ('<run_uuid>', '<job_uuid>', 0, 'python', 'hard', 'slug_match', 'success');
-- Segundo INSERT deve falhar com duplicate key
ROLLBACK;

-- Idem para FORI
BEGIN;
INSERT INTO function_orchestrator_role_items
  (run_id, job_posting_id, raw_item_index, title_original, pipeline_stage, status, processed_at, created_at)
VALUES
  ('<run_uuid>', '<job_uuid>', 0, 'Eng Software', 'llm_pure_layer_3', 'success', NOW(), NOW());
INSERT INTO function_orchestrator_role_items
  (run_id, job_posting_id, raw_item_index, title_original, pipeline_stage, status, processed_at, created_at)
VALUES
  ('<run_uuid>', '<job_uuid>', 0, 'Eng Software', 'llm_pure_layer_3', 'success', NOW(), NOW());
ROLLBACK;
```

#### E5 — Smoke CHECK 0..1

```sql
BEGIN;
-- Tentativa de inserir confidence fora da faixa deve falhar
INSERT INTO function_orchestrator_skill_items
  (run_id, job_posting_id, skill_raw_name, skill_raw_type, skill_confidence, pipeline_stage, status)
VALUES
  ('<run>', '<job>', 'x', 'hard', 1.5, 'slug_match', 'success');
-- Deve falhar com chk_foski_skill_confidence_range
ROLLBACK;

BEGIN;
INSERT INTO function_orchestrator_role_items
  (run_id, job_posting_id, title_original, confidence, pipeline_stage, status, processed_at, created_at)
VALUES
  ('<run>', '<job>', 'x', 1.5, 'llm_pure_layer_3', 'success', NOW(), NOW());
-- Deve falhar com chk_fori_confidence_range
ROLLBACK;
```

#### E6 — Smoke aggregator pós-deploy

```sql
-- Após SUB-PR 7 + backfill
SELECT
  summary_date,
  data->'operational'->'o7' AS o7_role_preexisting,        -- preservado
  data->'operational'->'o7_skill' AS o7_skill_new,         -- novo
  data->'operational'->'items_processed' AS items_total,   -- novo
  data->'limiares'->'panel_1' AS panel_1_snapshot,         -- novo (3 séries)
  data->'limiares'->'panel_8' AS panel_8_snapshot          -- novo (histograma + by_path)
FROM dashboard_daily_summary
ORDER BY summary_date DESC
LIMIT 3;
```

Esperado: campos `operational.o7` pré-existente preservado intacto; campos novos `o7_skill`, `items_processed`, `limiares.panel_*` preenchidos com valores válidos (não NULL) para dias com FOSI populada.

#### E7 — Smoke endpoint impact-preview

```bash
# Smoke POST como admin autenticado
curl -X POST 'https://<pre-prod>/api/admin/pipeline-config/skill.hard_gate.min_confidence/impact-preview' \
  -H 'Cookie: sb-access-token=<token>' \
  -H 'Content-Type: application/json' \
  -d '{"new_value": "0.75", "days": 30}'

# Esperado: HTTP 200 com payload completo:
# - Campos pré-existentes (cleanup v3.4): affected_count, affected_label, projected_event_cost_usd,
#   cost_window, cost_is_fallback, panels
# - Campos novos: window_days, source, sample_size, sample_status, current_impact, proposed_impact, histogram
# - Header cache-control: private, max-age=60
```

#### E8 — Smoke endpoint Limiares historical + online

```bash
# Historical 30d
curl 'https://<pre-prod>/api/admin/limiares/historical?range=30d' \
  -H 'Cookie: sb-access-token=<token>'
# Esperado: 10 painéis preenchidos lendo de dashboard_daily_summary; cache-control max-age=60

# Online (24h)
curl 'https://<pre-prod>/api/admin/limiares/online' \
  -H 'Cookie: sb-access-token=<token>'
# Esperado: 10 painéis via RPCs limiares_panel_N_snapshot; cache-control max-age=60
```

#### E9 — Smoke ausência de pg.Pool e Redis custom em endpoints admin

```bash
# Validação arquitetônica
grep -rn "new Pool\|require('pg')\|from 'pg'" app/api/admin/ --include="*.ts"
# Esperado: ZERO hits

grep -rn "@upstash/redis\|new Redis\|REDIS_URL" app/api/admin/ --include="*.ts"
# Esperado: ZERO hits em endpoints admin (Limiares/online, Limiares/historical, impact-preview)
# Hits em outros endpoints (não-admin) toleráveis se pré-existentes
```

#### E10 — Smoke orphan runs cleanup

```sql
-- Setup: criar run sintética em status='running' com started_at antigo
INSERT INTO function_orchestrator_runs (id, session_id, source, status, started_at, created_at)
VALUES ('<test_uuid>', '<session>', 'manual', 'running', NOW() - INTERVAL '25 hours', NOW() - INTERVAL '25 hours');

-- Trigger cleanup manualmente via CRON ou API
-- (chamar endpoint do cron analysis-cleanup ou novo orphan-runs-cleanup)

-- Validação pós-cleanup
SELECT status, error_message FROM function_orchestrator_runs WHERE id = '<test_uuid>';
-- Esperado: status='error', error_message='orphan_run_timeout'
```

### §7.2 — Smoke tests funcionais

Cada SUB-PR tem critério de aceite específico (referenciado nas §X.Y respectivas). Os 27 smokes abaixo cobrem ponta-a-ponta o comportamento esperado da sprint.

#### S1 — Fluxo A (admin manual JSON) escreve em `function_orchestrator_skill_items`

1. Subir JSON de teste via `/admin/ingestor` com 1 vaga contendo 3 skills variadas (uma com slug existente, uma com alias existente, uma genuinamente nova).
2. Disparar curate.
3. Verificar:

```sql
SELECT raw_item_index, skill_raw_name, status, pipeline_stage, canonical_status, needs_review
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id_de_teste>'
ORDER BY raw_item_index;
-- Esperado: 3 linhas, com stages 'slug_match', 'alias_match', 'llm_new' respectivamente.
-- raw_item_index: 0, 1, 2 (idempotência via UNIQUE).
-- canonical_status: 'active' para slug_match/alias_match, 'pending' para llm_new.
-- needs_review: false em casos comuns; true se algum alias de skill_type foi normalizado.
```

#### S2 — Fluxo B (LinkedIn busca automática) escreve em `function_orchestrator_skill_items`

Similar S1, disparado via execução do CRON `curate-job-postings` ou API automática. Validação idêntica.

#### S3 — Fluxo C (1:1 vaga colada) escreve em `function_orchestrator_skill_items`

Similar S1, via UI do usuário colando uma vaga no modal de análise 1:1. Validação idêntica.

#### S4 — Hard gate rejeita skills com confidence baixa e registra em FOSI

1. Definir `skill.hard_gate.min_confidence=0.80` em `pipeline_config`.
2. Subir vaga com 5 skills, sendo 2 com confidence < 0.80.
3. Verificar:

```sql
SELECT raw_item_index, skill_raw_name, status, pipeline_stage
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id>'
  AND status = 'gate_rejected'
ORDER BY raw_item_index;
-- Esperado: 2 linhas, ambas com pipeline_stage='gate_rejected' e canonical_skill_id=NULL.
-- raw_item_index único garantido pelo caller que monta os items rejeitados após os processados.
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
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status IN ('success', 'reused')) AS real_reused,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'created_pending') AS real_pending,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'gate_rejected') AS real_gate,
  (SELECT c FROM count_skill_items_by_status(r.id) WHERE status = 'failed') AS real_failed
FROM function_orchestrator_runs r
WHERE r.status = 'success'
  AND r.finished_at > NOW() - INTERVAL '24 hours';
-- Esperado: para cada linha, counters batem com agregação real via RPC.
-- skills_extracted = SUM de todas as colunas skill_*.
```

#### S6 — Validação de correctness sob volume >1000 (LK-CBO-31)

1. Inserir manualmente 1.500 rows em FOSI vinculadas a um único `run_id` de teste (com `raw_item_index` 0 a 1.499).
2. Rodar finalizer manualmente (chamar a função TS do finalizer apontando para o run de teste).
3. Verificar que `function_orchestrator_runs.skills_extracted = 1500` (não 1000).
4. Sem a RPC (via Supabase client `select count`), esperado: `skills_extracted ≤ 1000` (bug de truncamento do PostgREST). RPC SQL elimina o truncamento.

#### S7 — Modal de edição exibe impacto expandido (chave coberta por estimator + IMPACT_SOURCES)

1. Abrir `/admin/pipeline-config`.
2. Clicar em "Editar" em `skill.hard_gate.min_confidence`.
3. Verificar que a caixa qualitativa pré-existente (painéis afetados + linha "Custo Opus projetado (evento único)") está visível.
4. Digitar 0.75. Aguardar 500ms (debounce).
5. Verificar que **ao lado** da caixa qualitativa aparece o `ImpactPreview` com tabela + histograma + seletor 7d/30d/90d.
6. Verificar que linha "Custo Opus projetado" da caixa qualitativa atualiza com `projected_event_cost_usd` do payload.
7. Trocar para 7d. Verificar nova chamada disparada (POST `impact-preview` com `days: 7`); SWR muda a chave de cache.
8. Trocar para 90d. Verificar nova chamada.

#### S8 — Modal exibe impacto expandido para chave coberta APENAS por IMPACT_SOURCES (sem estimator)

**L-3 fix v2.7:** S8 e S9 reformulados — versões anteriores usavam `skill.opus_review.cooldown_days` que pós-K-1 (v2.6) foi reclassificada como `estimator_only`, fazendo `ImpactPreview` exibir placeholder em vez de tabela quantitativa. Adapter "Dentro/Fora do cooldown" removido em §5.1.4. Smokes agora testam chaves `covered` com adapter ativo.

1. Editar `skill.hard_gate.min_confidence` em ambiente com dados suficientes (≥30 items na janela).
2. Digitar 0.75. Aguardar.
3. `ImpactPreview` exibe tabela "Aprovados/Rejeitados" + histograma + seletor de janela (semântica de gate, paritary ao adapter cravado em §5.1.4 `ImpactTable.tsx`).
4. Caixa qualitativa pré-existente **NÃO** mostra linha de "Custo Opus projetado" (campo nulo no payload — `projected_event_cost_usd: null`).
5. Botão "Salvar" do modal continua habilitado.

#### S9 — Modal lida com amostra insuficiente

1. Editar `skill.merge_candidate.cosine_threshold` em ambiente SEM dados suficientes de candidatos (<30 pares no painel 3 na janela). Cenário típico: ambiente recém-criado/staging com poucos canonicals.
2. Digitar 0.85. Aguardar.
3. `ImpactPreview` exibe tabela com `current_impact` e `proposed_impact` (sempre exibida quando current_impact existe) + badge âmbar "Amostra insuficiente para distribuição confiável" mostrando `sample_size` e `sample_threshold`. Adapter aplicado: "Pares dentro/fora do threshold" (semântica de merge_candidate).
4. Histograma omitido.
5. Botão "Salvar" do modal continua habilitado.

#### S10 — Modal lida com chave de auto_assign_family (estimator_only — role-only)

1. Editar `role.auto_assign_family.min_similarity` (estimator pré-existente, registrada em `IMPACT_SOURCES` com `panel_path: ''` por D-PS-41 cleanup v3.4).
2. Digitar valor novo. Aguardar.
3. Caixa qualitativa pré-existente mostra "Custo Opus projetado (evento único)" com `projected_event_cost_usd`.
4. `ImpactPreview` (lado direito) exibe placeholder "Análise quantitativa não disponível para esta chave nesta versão" — payload tem `sample_status='estimator_only'`.
5. Botão "Salvar" continua habilitado.

#### S11 — Modal lida com chave de sistema (CURATE_PIPELINE_ENABLED)

1. Editar `CURATE_PIPELINE_ENABLED` via curl direto bypassando UI (`/admin/pipeline-config` filtra essas chaves):
   ```bash
   curl -X POST '/api/admin/pipeline-config/CURATE_PIPELINE_ENABLED/impact-preview' \
     -d '{"new_value": "false", "days": 30}'
   ```
2. Payload retorna HTTP 200 com `sample_status='system_key'` + `system_key_reason='Chave de sistema (feature flag de kill-switch global do pipeline). Não exposta na tela /admin/pipeline-config; edição apenas via API direta ou super-admin.'` (J-16 fix v2.5 — string literal paritary ao IMPACT_SOURCES entry de `CURATE_PIPELINE_ENABLED` §4.1.4).
3. UI normalmente não chega aqui; é caso defensivo do endpoint.

#### S12 — Pós-rename: trigger antigo (`trg_foi_jcr_confidence_*`) deixou de existir; novo (`trg_fori_*`) existe

```sql
SELECT tgname FROM pg_trigger
WHERE NOT tgisinternal
  AND tgname LIKE 'trg_fo%_jcr_confidence_%';
-- Esperado: 3 linhas, todas com prefixo 'trg_fori_'.
-- Esperado: zero linhas com prefixo 'trg_foi_' (legado).
```

#### S13 — Pós-rename: `fn_recompute_jcr_confidence_median` ainda funciona após rewrite mecânico

1. Inserir 5 rows em `function_orchestrator_role_items` (renomeado) para um canonical de teste, com `confidence` entre 0.7 e 0.95.
2. Verificar:

```sql
SELECT confidence_median FROM job_canonical_roles
WHERE id = '<canonical_id_de_teste>';
-- Esperado: valor entre 0.7 e 0.95 (mediana dos 5 inseridos).
-- Confirma que apenas o identificador da tabela mudou no corpo da função — lógica preservada.
-- GRANTs preservados via CREATE OR REPLACE (validar via \df+ no psql se necessário).
```

#### S14 — Path quarantined permanece invisível ao orchestrator (D-PS-83)

1. Forçar uma vaga a entrar em quarentena (mecanismo de quarantine pré-existente).
2. Confirmar que `mapSkillsToCanonical` foi chamada (não a função de descoberta orchestrada).
3. Confirmar que `function_orchestrator_skill_items` **NÃO** tem rows para essa vaga.
4. Confirmar que isso é comportamento esperado (D-PS-83). Skills extraídas em vagas quarantenadas permanecem invisíveis ao orchestrator.

#### S15 — Propagação de `needs_review` end-to-end (D-PS-65 + D-PS-69 cleanup v3.4)

1. Forçar LLM a emitir um alias PT-BR (`'tecnica'`) em uma skill via input controlado em ambiente de teste.
2. Verificar que `skill-type-guard.ts` normaliza para `'technical'` e marca `needsReview=true`.
3. Verificar que `RawSkill.needsReview` propaga para `SkillProcessingDetail.needs_review`.
4. Verificar que o item correspondente em `function_orchestrator_skill_items` tem `needs_review=true`:

```sql
SELECT raw_item_index, skill_raw_name, pipeline_stage, needs_review
FROM function_orchestrator_skill_items
WHERE run_id = '<run_id_de_teste>'
  AND needs_review = true;
-- Esperado: ≥1 linha, com raw_skill_name correspondente à skill que teve skill_type normalizado.
-- Propagação end-to-end: LLM → skill-type-guard → RawSkill → função de descoberta → SkillProcessingDetail → FOSI.
```

#### S16 — Smoke arquitetônico: ausência de `pg.Pool` e Redis custom em endpoints admin

```bash
# Validação arquitetônica: zero pg.Pool em endpoints admin
grep -rn "new Pool\|require('pg')\|from 'pg'" app/api/admin/ --include="*.ts"
# Esperado: ZERO hits

# Validação arquitetônica: zero Redis custom em endpoints admin
grep -rn "@upstash/redis\|new Redis\|REDIS_URL" app/api/admin/ --include="*.ts"
# Esperado: ZERO hits em app/api/admin/limiares/{online,historical} e app/api/admin/pipeline-config/*/impact-preview.
# Hits em outros endpoints (não-admin) toleráveis se pré-existentes (e.g., rate-limit de signup).
```

Validação adicional via HTTP: header `cache-control` paritário ao resto do dashboard global:

```bash
curl -I 'https://<pre-prod>/api/admin/limiares/online' -H 'Cookie: sb-access-token=<token>'
# Esperado: cache-control: private, max-age=60
```

#### S17 — Smoke endpoint Limiares historical lê de `dashboard_daily_summary`

1. Após backfill 26 dias (SUB-PR 7) e refator do endpoint (SUB-PR 8), bater GET com `?range=30d`.
2. Verificar via logs que a query foi `SELECT summary_date, data FROM dashboard_daily_summary WHERE summary_date >= ...` (Supabase client, não `pg.Pool`).
3. Payload retornado tem 10 painéis preenchidos via mesclagem cross-day dos snapshots em `data.limiares.panel_1..panel_10`.
4. Painéis com histograma (1, 2, 7, 8, 10) somam `cnt` por bucket/day; painéis com lista (3, 5, 9) concatenam + dedupam; painéis 4, 6 usam snapshot mais recente.

#### S18 — Smoke cleanup de orphan runs

```sql
-- Setup: criar run sintética em status='running' com started_at antigo
INSERT INTO function_orchestrator_runs (id, session_id, source, status, started_at, created_at)
VALUES (
  '<test_uuid>',
  '<session_uuid>',
  'manual',
  'running',
  NOW() - INTERVAL '25 hours',
  NOW() - INTERVAL '25 hours'
);

-- Disparar cleanup manualmente via CRON ou endpoint dedicado
-- (chamar handler do cron orphan-runs-cleanup ou função TS equivalente)

-- Validação pós-cleanup
SELECT status, error_message FROM function_orchestrator_runs WHERE id = '<test_uuid>';
-- Esperado: status='error', error_message='orphan_run_timeout'.

-- Validação adicional: runs criadas há mais de 90 dias são deletadas
SELECT COUNT(*) FROM function_orchestrator_runs WHERE created_at < NOW() - INTERVAL '90 days';
-- Esperado: 0 (todos os runs antigos deletados pelo cleanup).
```

#### S19 — Smoke evolução event_names role_*/skill_*

**S19a — Migração determinística pós-deploy:**

```sql
-- Pós §2.y migration
SELECT event_name, COUNT(*) AS cnt
FROM events
WHERE event_name IN (
  'canonical_promoted_dynamic',
  'canonical_creation_blocked_low_confidence',
  'role_promoted_dynamic',
  'skill_promoted_dynamic',
  'role_creation_blocked_low_confidence',
  'skill_creation_blocked_low_confidence'
)
GROUP BY event_name;
-- Esperado: 0 rows com os 2 nomes antigos; rows com os 4 nomes novos distribuídas
-- conforme entity_type pré-existente em metadata.
```

**S19b — Emissores TS atualizados:**

1. Disparar promotion sintética de canonical role: verificar evento emitido = `role_promoted_dynamic` (não `canonical_promoted_dynamic`).
2. Disparar promotion sintética de canonical skill: verificar evento emitido = `skill_promoted_dynamic` (paritário).
3. Disparar hard gate rejection em role: verificar evento = `role_creation_blocked_low_confidence`.
4. Disparar hard gate rejection em skill: verificar evento = `skill_creation_blocked_low_confidence` (paritário).

#### S20 — Smoke triggers canonical_role_created / canonical_skill_created

**S20a — INSERT em role emite evento:**

```sql
BEGIN;
-- E-13 fix: coluna created_via não existe em job_canonical_roles (D-PS-88) — removida do INSERT
INSERT INTO job_canonical_roles (label, status)
VALUES ('Smoke Test Role', 'pending');

-- Validação: evento foi emitido
SELECT event_name, metadata->>'canonical_id', metadata->>'label', metadata->>'entity_type'
FROM events
WHERE event_name = 'canonical_role_created'
  AND created_at >= NOW() - INTERVAL '1 minute';
-- Esperado: 1 row com label='Smoke Test Role' e entity_type='role'.
ROLLBACK;
```

**S20b — INSERT em skill emite evento paritário:**

```sql
BEGIN;
-- E-13 fix: coluna created_via não existe em job_canonical_skills (D-PS-88) — removida do INSERT
INSERT INTO job_canonical_skills (label, slug, status, kind)
VALUES ('Smoke Test Skill', 'smoke-test-skill', 'pending', 'hard');

SELECT event_name, metadata->>'canonical_id', metadata->>'label', metadata->>'entity_type'
FROM events
WHERE event_name = 'canonical_skill_created'
  AND created_at >= NOW() - INTERVAL '1 minute';
-- Esperado: 1 row com label='Smoke Test Skill' e entity_type='skill' (paritário).
ROLLBACK;
```

#### S21 — Smoke rewrite mecânico preserva proconfig

```sql
-- Pré-rewrite (capturado via §6.1):
-- internal.reset_taxonomy_core         → proconfig = '{search_path=public, pg_temp}' (ou similar)
-- public.cleanup_batch_items           → proconfig = '{search_path=public}' (sem pg_temp)
-- public.fn_recompute_jcr_*            → proconfig = '{search_path=public, pg_temp}'
-- public.merge_canonicals              → proconfig = (validar via §6.1)
-- public.release_quarantined_jobs_*    → proconfig = (validar via §6.1)

-- Pós-rewrite (§2.6b):
SELECT proname, pronamespace::regnamespace::text AS schema, proconfig
FROM pg_proc
WHERE proname IN (
  'reset_taxonomy_core',
  'cleanup_batch_items',
  'fn_recompute_jcr_confidence_median',
  'merge_canonicals',
  'release_quarantined_jobs_limited'
)
ORDER BY schema, proname;
-- Esperado: proconfig de cada função IDÊNTICO ao snapshot pré-rewrite.
-- ATENÇÃO: cleanup_batch_items deve continuar com search_path=public SEM pg_temp
-- (não acrescentar pg_temp por "padronização").
```

#### S22 — Smoke shape JSONB data.operational.* pós-backfill

```sql
-- Pós SUB-PR 7 (aggregator refator + backfill 26 dias)
-- Validação: TODOS os 26 dias têm shape novo (lowercase aninhado)
SELECT summary_date,
       data ? 'operational' AS has_operational_nested,
       data ? 'O7' AS has_O7_top_level,
       data->'operational' ? 'o7' AS has_o7_in_operational,
       data->'operational' ? 'o7_skill' AS has_o7_skill,
       data ? 'limiares' AS has_limiares_family
FROM dashboard_daily_summary
ORDER BY summary_date DESC
LIMIT 30;
-- Esperado: has_operational_nested=true em todos; has_O7_top_level=false em todos
-- (shape antigo eliminado pelo backfill); has_o7_in_operational=true; has_o7_skill=true;
-- has_limiares_family=true.

-- Validação cruzada: nenhum dia ainda tem o shape antigo top-level
SELECT COUNT(*) FROM dashboard_daily_summary WHERE data ? 'O7';
-- Esperado: 0. Se > 0, backfill não cobriu todos os dias — investigar.
```

#### S23 — Smoke ENUM cast FORI: validação pré e pós `ALTER COLUMN TYPE USING`

**Pré-cast (executado antes de §2.x):**

```sql
-- Validar que TODOS os valores existentes em pipeline_stage e status batem com os ENUMs
-- definidos em §2.x (role_pipeline_stage com 6 valores; role_item_status com 5 valores).
WITH role_pipeline_values AS (
  SELECT unnest(ARRAY['deterministic', 'cache_hit_layer_0', 'dict_match_layer_1',
                       'suggested_role_layer_2', 'llm_pure_layer_3', 'fallback_error']) AS v
),
role_status_values AS (
  -- L-1 fix v2.7: paritary ao DB CHECK real (5 valores)
  SELECT unnest(ARRAY['success', 'fallback',
                       'low_quality', 'pending_review', 'failed']) AS v
)
SELECT 'pipeline_stage' AS col, pipeline_stage AS value_in_db, COUNT(*) AS row_count
FROM function_orchestrator_role_items
WHERE pipeline_stage NOT IN (SELECT v FROM role_pipeline_values)
GROUP BY pipeline_stage
UNION ALL
SELECT 'status' AS col, status AS value_in_db, COUNT(*) AS row_count
FROM function_orchestrator_role_items
WHERE status NOT IN (SELECT v FROM role_status_values)
GROUP BY status;
-- Esperado: 0 rows. Se retornar qualquer row, valor órfão existe e cast falhará.
```

**Pós-cast (executado depois de §2.x):**

```sql
-- Validar tipos resultantes
SELECT column_name, data_type, udt_name
FROM information_schema.columns
WHERE table_name='function_orchestrator_role_items'
  AND column_name IN ('pipeline_stage', 'status');
-- Esperado: data_type='USER-DEFINED'; udt_name='role_pipeline_stage' e 'role_item_status'.

-- Validar enum values pós-criação
SELECT t.typname, array_agg(e.enumlabel ORDER BY e.enumsortorder) AS values
FROM pg_type t
JOIN pg_enum e ON e.enumtypid = t.oid
WHERE t.typname IN ('role_pipeline_stage', 'role_item_status', 'skill_pipeline_stage', 'skill_item_status')
GROUP BY t.typname;
-- Esperado: 4 rows (2 role + 2 skill paritários);
--   role_pipeline_stage com 6 valores; role_item_status com 5 valores;
--   skill_pipeline_stage com 6 valores; skill_item_status com 5 valores (paridade documental).
```

#### S24 — Smoke cobertura paritária `handleSkillItemError` vs `handleItemError`

Auditoria validando que cada call site de `handleItemError` no pipeline de role tem equivalente `handleSkillItemError` no pipeline de skill (ou justificativa documentada de não-aplicabilidade):

```bash
# Listar callers de cada um
grep -rn "handleItemError(" lib/pipeline/ --include="*.ts" \
  | grep -v "lib/pipeline/batch-processor/error-handlers.ts" \
  | sort > /tmp/role_callers.txt
grep -rn "handleSkillItemError(" lib/pipeline/ --include="*.ts" \
  | grep -v "lib/pipeline/batch-processor/error-handlers.ts" \
  | sort > /tmp/skill_callers.txt

# Diff esperado: para cada arquivo X em /tmp/role_callers.txt, arquivo análogo em skill_callers.txt
# (mesmo módulo, mesmo nível semântico do catch/path)
```

Validação manual cruzada com `AUDIT-orchestrator-symmetry-v2.8-<data>.md` (E0h): cada linha da tabela de mapeamento role↔skill marcada como `instalado` ou `não-aplicável (com justificativa)`. Linhas marcadas como `pendente` reprovam o smoke.

#### S25 — Smoke backfill `entity_type` (SUB-PR 0 §2.w)

```sql
-- Pós SUB-PR 0
SELECT
  metadata->>'entity_type' AS entity_type,
  COUNT(*) AS cnt
FROM events
WHERE event_name IN ('canonical_promoted_dynamic', 'canonical_creation_blocked_low_confidence')
GROUP BY metadata->>'entity_type'
ORDER BY metadata->>'entity_type';
-- Esperado: agregação por 'role', 'skill', 'unknown'. Soma = 36 (31 + 5).
-- Nenhuma row com entity_type IS NULL.

-- Event de auditoria emitido
SELECT metadata FROM events
WHERE event_name = 'entity_type_backfilled'
ORDER BY created_at DESC LIMIT 1;
-- Esperado: 1 row com metadata.role_count + skill_count + unknown_count somando 36.
```

#### S26 — Smoke `changed_by = SYSTEM_USER_ID` no panel_9 (rows de processo automático)

```sql
-- Setup: inserir 1 row sintética em pipeline_config_history via SYSTEM_USER_ID
-- (changed_by é NOT NULL por construção — ver migration 40; system user é o UUID conhecido)
INSERT INTO pipeline_config_history (key, previous_value, new_value, changed_by, reason, changed_at)
VALUES (
  'role.test.key', '0.5', '0.6',
  '00000000-0000-0000-0000-000000000000'::uuid,  -- SYSTEM_USER_ID via constants
  'smoke test system user',
  NOW() - INTERVAL '5 days'
);

-- Invocar RPC
SELECT limiares_panel_9_snapshot('30 days');
-- Esperado: payload retornado tem ao menos 1 entry em recent_changes com
-- changed_by='00000000-0000-0000-0000-000000000000'.

-- Cleanup
DELETE FROM pipeline_config_history WHERE key='role.test.key' AND reason='smoke test system user';
```

Validação adicional no admin UI: abrir aba Limiares → painel 9 → entry com `changed_by = SYSTEM_USER_ID` é renderizado como `'sistema'` (não como UUID literal nem como erro de UI).

#### S27 — Smoke cobertura `IMPACT_SOURCES.length === 28`

```typescript
// Validação de cardinalidade de IMPACT_SOURCES (cravado em §4.1.4)
// Executar antes do build de produção como gate.
import { IMPACT_SOURCES } from '@/lib/admin/pipeline-config-impact-sources';

describe('IMPACT_SOURCES cardinality gate', () => {
  it('expõe exatamente 28 chaves cobertas', () => {
    const keys = Object.keys(IMPACT_SOURCES);
    expect(keys.length).toBe(28);
    // Breakdown esperado (cravado em §4.1.4) — pós K-1/K-4/K-5 fix v2.6:
    //   12 covered (HARD_GATE 2 + PROMOTION 4 [auto_min_confidence+min_distinct_employers+min_vacancies role+skill]
    //   + MERGE_CANDIDATE 2 [cosine_threshold role+skill] + RETIREMENT 2)
    //   14 estimator_only (CONFIDENCE 4 + AUTO_ASSIGN_FAMILY 2 + 8 reclassificadas em v2.6:
    //     *.promotion.lookback_days x2 + *.merge_candidate.lookback_days x2 +
    //     *.merge_candidate.opus_review_cooldown_days x2 + *.opus_review.cooldown_days x2)
    //   2 system_key (CURATE_PIPELINE_ENABLED + QUARANTINE_EXPIRY_DAYS)
    //   Total: 12 + 14 + 2 = 28
  });

  it('cada entry covered tem current_value_filter', () => {
    for (const [key, entry] of Object.entries(IMPACT_SOURCES)) {
      if (entry.key_class === 'covered') {
        expect(entry.current_value_filter).toBeDefined();
      }
    }
  });

  it('cada entry system_key tem system_key_reason', () => {
    for (const [key, entry] of Object.entries(IMPACT_SOURCES)) {
      if (entry.key_class === 'system_key') {
        expect(entry.system_key_reason).toBeDefined();
        expect(entry.system_key_reason).not.toBe('');
      }
    }
  });
});
```

**E0 paralelo (auditoria operacional):** durante SUB-PR 1, rodar `node -e "console.log(Object.keys(require('@/lib/admin/pipeline-config-impact-sources').IMPACT_SOURCES).length)"` e documentar saída em `AUDIT-orchestrator-symmetry-v2.8-<data>.md`. Drift entre código e spec é detectado tanto em build (assertion) quanto em auditoria (E0 step), com duas camadas de defesa.

#### Sequência de validação por SUB-PR

Para cada SUB-PR:

1. Rodar evidence E1 (pré-aplicação) — §7.1
2. Aplicar migration/edit conforme §X.Y
3. Rodar evidence E2 (pós-aplicação) — §7.1
4. Rodar smokes E3-E10 da §7.1 e S1-S27 conforme aplicável
5. Rodar `test:sql` 5x consecutivos para detectar flakiness em transações concorrentes
6. TSC `--noEmit` limpo
7. Marcar item correspondente no TodoWrite somente após 5x test:sql verdes consecutivos + TSC limpo + smokes específicos OK

---

## §8 Sub-PRs breakdown

| SUB-PR | Frente | Conteúdo | Estimativa | Depende de |
|---|---|---|---|---|
| 0 | A | §2.w backfill `entity_type` em events legacy (FK lookup paritário) + auditoria de rows não-resolvidas | 0.5 dia | cleanup v3.4 mergeada |
| 1 | A | §2.1 FOSI (com canonical_status + 2 ENUMs) + §2.2 colunas skill em FOR + §2.3 validação + §6.1 E0a-E0k (incluindo E0h handleItemError callers + E0i event_names ativos) + §6.2 auditoria (5 funções reais) | 2 dias | cleanup v3.4 mergeada |
| 2 | A | §2.x corretivos retroativos FORI (CHECK + raw_item_index + ENUMs + CASCADE) + §2.4 RPC + §2.6 rename + §2.y event_names (CHECK constraint nova com whitelist E0i) + §2.z triggers (sem created_via + WHEN guard) + §3.4 call sites + atualização emissores TS | **4.5 dias** | SUB-PR 0 + SUB-PR 1 |
| 3 | A | §3.1 refator descoberta + §3.2 insertFOSkillItem (individual+bulk com canonical_status) + paridade bug fix FORI + handleSkillItemError com **cobertura paritária de callers (E0h)** | 1.75 dia | SUB-PR 2 |
| 4 | A | §3.3 finalizers via RPC compartilhada + UPDATE atômica | 0.75 dia | SUB-PR 3 |
| 5 | A | cron monthly-cleanup paritário FOSI + cleanup novo orphan runs (universal) | 0.5 dia | SUB-PR 1 (paralelo com 2-4) |
| 6 | B | 12 RPCs SQL (`count_*_in_window` + 10 `limiares_panel_N_snapshot` com `detected_at` + event_names role_*/skill_* + schema correto pipeline_config_history + fallback gap_days 365/365) | 2 dias | SUB-PR 4 + SUB-PR 2 |
| 7 | B | §3.6 aggregator expandido refator shape data.operational.* + leitura de `.error` paritária + backfill 26 dias com shape novo | 2.5 dias | SUB-PR 6 |
| 8 | B | refator `/api/admin/limiares/historical` para `dashboard_daily_summary` | 0.75 dia | SUB-PR 7 |
| 9 | B | refator `/api/admin/limiares/online` para Supabase client + RPCs | 0.5 dia | SUB-PR 6 |
| 10 | C | refator `impact-preview` (estimateImpact assinatura correta) + extração `EditModal`/`ImpactPreview` | 1.5 dia | SUB-PR 7 |
| 11 | B | refator UI painéis 2.1/2.7/2.8/2.9 (paths data.operational.*) + 10 painéis Limiares. Deploy coordenado com SUB-PR 7 | 2.5 dias | SUB-PR 7 + 9 + 10 |

**Janela total estimada:** 14-17 dias úteis. Buffer no SUB-PR 2 dado peso (5 migrations + auditoria E0h/E0i).

**Paralelismo possível:**
- SUB-PR 5 em paralelo com 2-4
- SUB-PRs 8 e 9 em paralelo entre si (após 6 e 7)
- SUB-PR 10 em paralelo com 8/9 (depende só de 7)

**J-11/J-12 fix v2.5 — Deploy coordenado obrigatório:**
- **SUB-PR 10** deploya o handler refatorado de `impact-preview` que chama `combineDistribution()` (§4.1.10). O corpo de `combineDistribution` está em `lib/admin/pipeline-config-impact-aggregator.ts` e é implementado em **SUB-PR 8**. Se SUB-PR 10 entrar em produção antes de SUB-PR 8 (ou em deploys separados), a primeira chamada de POST `/api/admin/pipeline-config/:key/impact-preview` explode com `NOT_IMPLEMENTED` em runtime. **Implementação:** SUB-PR 8 e SUB-PR 10 são mesma janela de deploy (mesmo merge train) OU SUB-PR 10 fica feature-flagged até SUB-PR 8 estar em produção.
- **SUB-PR 8** deploya `aggregateLimiaresHistorical()` (§4.2). Mesma classe de constraint: handler `/api/admin/limiares/historical` invoca a função; corpo precisa estar deployed antes do handler aceitar requests. SUB-PR 8 já contém ambos (handler + corpo), então não há split — mas o critério de aceite "estrutura com 10 painéis preenchidos" só passa após o corpo ser implementado, não com `throw NOT_IMPLEMENTED` stub.

**J-15 fix v2.5 — Ordering EVENT_NAMES whitelist:** o cálculo `77+4+2=83` em §6.1 (E0i) assume que o snapshot `SELECT DISTINCT event_name FROM events` é tirado **após** SUB-PR 0 (que emite `entity_type_backfilled`). Cravar ordering explícito em §8:

1. **SUB-PR 1** roda E0i pela primeira vez (snapshot pré-SUB-PR 0 — 76 event_names, sem `entity_type_backfilled`).
2. **SUB-PR 0** executa backfill + emite eventos `entity_type_backfilled` (passa a haver 77 event_names ativos).
3. **SUB-PR 1 re-roda E0i** (snapshot pós-SUB-PR 0 — 77 event_names ativos) e produz artefato `EVENT_NAMES-orchestrator-symmetry-v2.8.md` consolidado com 83 valores (77 + 4 paritários + 2 triggers).
4. **SUB-PR 2** aplica §2.y `ADD CONSTRAINT` com whitelist literal do artefato.

Sem esse ordering, SUB-PR 2 §2.y dispara CHECK violation: roda `events_event_name_check` enquanto whitelist do artefato ainda reflete snapshot pré-SUB-PR 0 (76 nomes; faltariam tanto `entity_type_backfilled` quanto rows novas emitidas).

**J-7 fix v2.5 — SUB-PR 5 soft dependency:** apesar de listado como paralelo com 2-4, o smoke de SUB-PR 5 (cleanup FOSI > 2 anos) só valida non-zero **após** SUB-PR 4 (FOSI populada com rows reais via pipeline ativo). Cravar como **soft dependency** em §0: SUB-PR 5 pode ser desenvolvido em paralelo, mas seu smoke é validado pós SUB-PR 4.

**J-10 fix v2.5 — count_skill_items_by_status soft ordering:** smoke desta RPC (§2.4) também depende de FOSI populada via SUB-PR 3 (insertFOSkillItem invocado). Documentar soft ordering na descrição do smoke.

---

## §9 D-PS, S-ORCH, LK-PS, decisões herdadas

### Decisões de produto (D-PS) desta sprint

**Nota sobre numeração:** existem gaps na numeração D-PS desta sprint (D-PS-80, D-PS-81, D-PS-82, D-PS-85). Esses números foram **reservados** durante o ciclo de revisão multi-AI para itens que acabaram sendo absorvidos por outras decisões ou retirados de escopo. Mantidos como reservados (não reutilizar).

- **D-PS-74 (fontes de `confidence_median` divergentes):** `confidence_median` em `job_canonical_skills` é populado por `fn_recompute_jcs_confidence_median` (renomeada de `fn_jps_recompute_jcs` pela cleanup v3.4 F21). Esta sprint apenas referencia o nome novo; não toca o circuito. `fn_recompute_jcr_confidence_median` (lado role) continua sendo fonte canônica do role, com corpo atualizado mecanicamente apenas para referenciar `function_orchestrator_role_items` pós-rename.
- **D-PS-75 (fontes assimétricas intencionais):** assimetria de fontes de `confidence_median` role vs skill é INTENCIONAL e fora do escopo. Lado role agrega FORI items; lado skill agrega job_posting_skills curated. Mudança de fonte exige avaliação de impacto em todo o circuito de promoção e fica como frente futura caso justificada.
- **D-PS-79 (enum stage-oriented em FOSI):** o enum `skill_pipeline_stage` reflete os paths reais do pipeline de skill (`slug_match`, `alias_match`, `llm_new`, `race_recovered`, `gate_rejected`, `fallback_error`). Não é mapeamento 1:1 dos paths role do FORI; reflete a estrutura semântica do pipeline de skill que difere por construção (cardinalidade alta, ausência de "outcome único", presença de `gate_rejected` como estado terminal explícito).
- **D-PS-83 (path quarantined fora):** skills extraídas em vagas quarentenadas (legacy `mapSkillsToCanonical`) permanecem invisíveis ao orchestrator. Frente futura caso volume operacional justifique.
- **D-PS-84 (contadores stage-oriented em FOR são assimétricos):** os 5 contadores `skills_*` novos em FOR são stage-oriented (refletem paths); os 8 contadores role pré-existentes são outcome-oriented. Justificativa: instrumentação retrospectiva sobre paths existentes em produção; mapear para 8 outcomes role exigiria reclassificação semântica que perderia fidelidade.
- **D-PS — `canonical_status` em FOSI paritário a FORI:** FOSI ganha coluna `canonical_status` com `CHECK IS NULL OR IN ('active', 'pending')` paritária à coluna pré-existente em FORI. `SkillProcessingDetail.canonical_status` (tipo TS) já existia; INSERT em §3.2 popula a coluna. Telemetria simétrica em ambos os lados; smoke S1 lê o campo normalmente.
- **D-PS — ENUMs paritários para `pipeline_stage` e `status` em FORI:** conversão retroativa de `text + CHECK` para `CREATE TYPE role_pipeline_stage AS ENUM` e `CREATE TYPE role_item_status AS ENUM` (§2.x). Paridade total de tipos com FOSI. Tradeoff conhecido: `ALTER TYPE ADD VALUE` necessário para adicionar valores futuros (ENUMs PG são imutáveis após criação); aceito dado o pipeline maduro com paths estáveis.
- **D-PS — `ON DELETE CASCADE` retroativo em FORI:** FKs `run_id` e `job_posting_id` em FORI ganham `ON DELETE CASCADE`; `canonical_role_id` ganha `ON DELETE SET NULL`. Paridade total com FOSI por construção. Garante deleção em cascata simétrica e coerência com retention.
- **D-PS — Migração de evolução de event_names role_*/skill_*:** `canonical_promoted_dynamic` e `canonical_creation_blocked_low_confidence` evoluem para nomes discriminados por entidade. Update determinístico via `metadata->>'entity_type'` (populado pelo SUB-PR 0 §2.w via FK lookup). Call sites TS atualizados em paralelo. Discriminação semântica forte; queries de aggregator simplificadas; convenção `<entity>_<verb>_<modifier>` paritária a `canonical_role_created`/`canonical_skill_created`.
- **D-PS-87 — Política de backfill `entity_type` em events legacy (SUB-PR 0 §2.w):** ground truth via §6.1 confirmou 36 rows legacy sem `entity_type` (31 `canonical_promoted_dynamic` + 5 `canonical_creation_blocked_low_confidence`). Backfill via FK lookup determinístico em `resource_id` cruzando `job_canonical_roles` e `job_canonical_skills` é a estratégia escolhida. Rows não-resolvidas (esperado 0 ou número pequeno) → `entity_type='unknown'` + log de auditoria + exclusão da rename de §2.y. Pré-step bloqueador de SUB-PR 2; sem isso, pre-check de §2.y trava na origem.
- **D-PS-88 — `created_via` removido do payload do trigger (opção c):** ground truth via §6.1 confirmou que a coluna `created_via` **não existe** em `job_canonical_roles` nem em `job_canonical_skills`. Cravar coluna nova como preparação para callers futuros geraria coluna perpetuamente vazia (nenhum caller existente popularia o campo) — overengineering sem ganho real dado que a base começa zerada no deploy. Payload do event fica `{canonical_id, label, entity_type}` paritário em ambas as funções trigger. Rastreabilidade de origem (LLM pipeline vs admin manual vs backfill) fica como dívida explícita LK-PS-NEW caso necessidade futura justifique adicionar a coluna em ambas as tabelas paritárias com callers populando-a.
- **D-PS — Triggers paritários `canonical_role_created`/`canonical_skill_created` com `WHEN (pg_trigger_depth() = 0)`:** cleanup v3.4 cravou os nomes via LK-PS-22 mas trigger nunca foi criado; ground truth via §6.1 confirma 0 rows desses nomes em events. Esta sprint cria 2 triggers paritários (`fn_emit_canonical_role_created` + `fn_emit_canonical_skill_created`) AFTER INSERT em ambas as tabelas, com guard `WHEN (pg_trigger_depth() = 0)` para prevenir loop caso algum trigger downstream insira de volta. Metadata `{canonical_id, label, entity_type}` sem `created_via` (ver D-PS-88). Sem trigger, drift O7 role+skill seria sempre 0. Paridade por construção.
- **D-PS — Refator do shape JSONB `dashboard_daily_summary.data`:** famílias top-level uppercase `data.O7/O8/O9` (role-only pré-existente) migram para família aninhada lowercase `data.operational.{o7, o8, o9}` paritária ao naming pattern das demais (`ai`/`resources`/`communications`/`auth`/`errors`/`accumulators`). Justificativa: enriquecimento de nomenclatura + coerência estrutural. Deploy coordenado obrigatório (aggregator + backfill + UI no mesmo SUB-PR 7 + SUB-PR 11) para evitar janela de inconsistência.
- **D-PS — bulk insertFOSkillItem operacional:** `insertFOSkillItem` tem duas modalidades: individual e bulk via `.insert(array)`. Bulk é o caminho preferencial para skill dada cardinalidade real (10-30 skills por job vs role 1-2). Assimetria operacional documentada e intencional: lado role mantém individual hoje; lado skill ganha bulk de partida. Lado role pode adotar bulk em sprint futura se métricas justificarem.
- **D-PS — paridade COM bug fix em `insertFOItem` retroativo:** `insertFOItem` lado role pré-existente engole silenciosamente `.error` do Supabase. Esta sprint corrige: `if (error) console.warn('[batch] insertFOItem failed:', error.message)` adicionado paritariamente em FOSI (novo) e FORI (retroativo).
- **D-PS — paridade COM bug fix em `aggregateDayData` `.error` reading:** `aggregateDayData` lado pré-existente nunca lê `.error` do Supabase (null data vira 0 via `?? 0` silenciosamente). Esta sprint corrige: cada fetcher do aggregator passa a ler `.error` e logar `[dashboard-aggregator] <fetcher>_query_failed` antes de fallback, paritariamente em ambos os lados.
- **D-PS — `handleSkillItemError` extraído em `error-handlers.ts` + cobertura paritária de callers (E0h):** paritário ao `handleItemError` pré-existente. Auditoria E0h lista todos os call sites de `handleItemError` no pipeline de role; SUB-PR 3 instala invocações paritárias de `handleSkillItemError` em call sites equivalentes do pipeline de skill (não apenas no catch upstream do `safeDiscoverAndLinkSkills`). NÃO substitui `insertEvent('persist_curation.skill_map_failed')` adicional — soma como dupla camada (event = auditoria global; FOSI = telemetria item-por-item).
- **D-PS — `raw_item_index` corretivo retroativo em FORI:** coluna nova + UNIQUE constraint paritários à criação em FOSI. Update obrigatório em `process-item.ts:199 + 206` para passar o índice explicitamente. Idempotência de retry garantida em ambos os lados.
- **D-PS — `CHECK (confidence 0..1)` corretivo retroativo em FORI:** CHECK constraint nova paritária à criação em FOSI. Defesa em profundidade contra parser bug ou LLM retornando valor fora da faixa. Pré-check obrigatório antes do `ALTER TABLE` para garantir 0 rows poluídos.
- **D-PS — rewrite mecânico de 5 funções dependentes (4 public + 1 internal):** lista cravada via ground truth: `internal.reset_taxonomy_core`, `public.cleanup_batch_items`, `public.fn_recompute_jcr_confidence_median`, `public.merge_canonicals`, `public.release_quarantined_jobs_limited`. Cada função preserva proconfig original específica — em particular `cleanup_batch_items` tem `search_path=public` (sem `pg_temp`) e DEVE ser preservada exatamente. `maintenance_phase_1` e `process_opus_create_new` alegadas em sessões anteriores NÃO existem no DB; removidas da lista.
- **D-PS — convenção `<entity>_canonical_<verb>` para eventos:** event_name canônico para criação de canonical é `canonical_role_created` e `canonical_skill_created` (ambos triggers cravados em §2.z). Aggregator filtra por esses nomes. Convenção paritária a `role_promoted_dynamic`/`skill_promoted_dynamic` e `role_creation_blocked_*`/`skill_creation_blocked_*` (§2.y migration).
- **D-PS — orphan runs cleanup é universal:** novo cleanup em CRON marca runs com `started_at < NOW() - 24h AND status='running'` como `status='error'`/`error_message='orphan_run_timeout'`. Deleta runs `created_at < 90d`. Universal (não Flow C-específico) — qualquer fluxo (A, B, C) que crie run e não a finalize por timeout/erro é coberto. Cleanup implementado como extensão do cron `monthly-cleanup` pré-existente (mesma janela operacional).
- **D-PS — worst-of-N traffic-light no render layer:** classificação visual (verde/âmbar/vermelho) dos painéis O9 e equivalente skill é computada na UI em `lib/admin/dashboard/o9-status-summary.ts` (arquivo novo desta sprint), NÃO no aggregator. Aggregator materializa counters puros (`fallback_error`, `sem_canonical`, etc); UI aplica o algoritmo.
- **D-PS — convergência arquitetônica completa:** 100% dos endpoints admin passam a usar Supabase client para acesso ao DB. Zero `pg.Pool` direto em endpoints admin. Zero Redis custom em endpoints admin. Queries complexas (WIDTH_BUCKET, CTE, UNION ALL, PERCENTILE_CONT) encapsuladas em RPCs SQL chamadas via Supabase client. Histórico via `dashboard_daily_summary` cross-day; dia corrente via RPCs com janela `'1 day'`. Cache via header HTTP `cache-control: max-age=60`.
- **D-PS — unificação Limiares no dashboard_daily_summary:** 10 painéis da aba Limiares passam a consumir o mesmo storage materializado que os ~50 painéis do dashboard global. Snapshot diário dos 10 painéis materializado pelo CRON `dashboard-consolidate` em `data.limiares.panel_1..panel_10`. Painéis 3/5/9 (listas detalhadas) inclusive — snapshots com listas serializadas em JSONB.
- **D-PS-86 (fallback `retirement.gap_days` 365/365 paritário):** `pipeline_config` tem `role.retirement.gap_days=365` e `skill.retirement.gap_days=365`. Fallback nas RPCs alinhado paritariamente em 365 para ambos lados refletindo o DB como fonte de verdade.
- **D-PS — schema correto em §3.5.5 (`detected_at` não `created_at`):** tabelas `canonical_role_merge_candidates` e `canonical_skill_merge_candidates` têm coluna `detected_at`. RPC `limiares_panel_3_snapshot` usa esse nome paritariamente em ambas queries (role + skill).
- **D-PS — CHECK constraint `events_event_name_check` criada do zero:** `events_event_name_check` não existe previamente no DB. Esta sprint cria a constraint via `ADD CONSTRAINT` único contendo whitelist exaustiva derivada de E0i (77 event_names ativos no DB + 4 paritários novos + 2 nomes de triggers = 83 valores totais; legacy `canonical_promoted_dynamic`/`canonical_creation_blocked_low_confidence` e auditoria `entity_type_backfilled` já contam nas 77 ativas). Lista literal entregue como artefato anexo `EVENT_NAMES-orchestrator-symmetry-v2.8.md` produzido pelo SUB-PR 1; implementor cola lista no §2.y antes da migration.

### Decisões herdadas da cleanup v3.4 que esta sprint utiliza, reafirma ou referencia

- **D-PS-41 cleanup v3.4 (assimetria de famílias):** `auto_assign_family.*` é role-only por construção. 2 estimators role-only preservados; nenhuma versão skill necessária.
- **D-PS-49 cleanup v3.4 (fontes assimétricas de confidence_median):** reafirmada — esta sprint não toca o circuito.
- **D-PS-50 cleanup v3.4 (canonical_created/canonical_promoted role-only):** reafirmada.
- **D-PS-51 cleanup v3.4 (fallback_ratio role-only):** reafirmada.
- **F11 cleanup v3.4 (cache key inclui updated_at):** referenciado; esta sprint substitui o cache in-process Map por leitura direta de `dashboard_daily_summary` + RPCs SQL.
- **F17 cleanup v3.4 (painéis Limiares paritários):** referenciado; esta sprint herda paridade pré-existente nos painéis 1/2/4/5/6/7 e mantém intacta nos painéis 3/8/10 que já tinham UNION ALL.
- **F21 cleanup v3.4 (rename `fn_jps_recompute_jcs` → `fn_recompute_jcs_confidence_median`):** referenciado; D-PS-74 desta sprint usa o nome novo canonicamente.
- **F22 cleanup v3.4 (Hard Gate skill ativo + 3 events `skill_creation_blocked_*`):** referenciado; RPC `limiares_panel_1_snapshot` consome esses events.
- **F23 cleanup v3.4 (cobertura 28/28 estimators):** referenciado; esta sprint usa os estimators como base do payload `affected_count`/`projected_event_cost_usd` no impact-preview, adicionando histograma WIDTH_BUCKET por cima para subset com `sample_size` suficiente.

### Simetrias planejadas (S-ORCH)

- **S-ORCH-1** — FORI ↔ FOSI tabelas paritárias com CHECKs + UNIQUE de idempotência
- **S-ORCH-2** — contadores `skills_*` em FOR paritários (stage-oriented) aos 8 role pré-existentes (outcome-oriented)
- **S-ORCH-3** — `insertFOItem` ↔ `insertFOSkillItem` (com bulk path adicional para skill por cardinalidade real)
- **S-ORCH-4** — `handleItemError` ↔ `handleSkillItemError`
- **S-ORCH-5** — 3 finalizers atualizam contadores role+skill em UPDATE atômica via RPC compartilhada
- **S-ORCH-6** — 12 RPCs SQL paritárias por painel + 2 RPCs de contagem por stage em janela
- **S-ORCH-7** — campos `data.operational.o7_skill`/`o8_skill`/`o9_skill`/`items_processed` paritários aos pré-existentes role-only
- **S-ORCH-8** — `data.limiares.panel_1..panel_10` materializado paritariamente para todos os 10 painéis
- **S-ORCH-9** — impact-preview consome storage unificado paritariamente para chaves role/skill

### Links (LK-PS)

- **LK-PS-07** — `bulk_run_id` para agregação multi-lote: dependência da tabela `bulk_curation_progress` do benchmark v14 (adiado). Não cravado nesta sprint.
- **LK-PS-19** — Flow C visibility model como sprint dedicada futura. Migration em `job_postings` com `visibility text NOT NULL DEFAULT 'public' CHECK (visibility IN ('public', 'private'))` + `owner_profile_id uuid REFERENCES profiles(id)`. Vagas privadas (Flow C) ficam `visibility='private'` com `owner_profile_id` do usuário. Atualização paritária de `fn_recompute_jcr_confidence_median`, `fn_recompute_jcs_confidence_median`, `fn_promote_role_on_threshold`, `fn_promote_skill_on_threshold` para filtrar `WHERE jp.visibility='public'`. Filtros em queries de análise CV pública. Análise condicional do dono para incluir vagas privadas próprias. UI do modal de vaga 1:1 colada. Estimativa: 3-4 dias dedicados.
- **LK-PS-20** — Rate-limit em endpoints admin como frente futura transversal. Avaliado com base em métricas reais pós-go-live com autorização expressa. Adicionar isoladamente em um endpoint quebra consistência arquitetônica dado que 0/~88 painéis admin têm rate-limit hoje.
- **LK-PS-23** — Análise quantitativa de chaves baseadas em dias (`*.lookback_days` + `*.cooldown_days`) exige campos numéricos bucketable nos RPCs `limiares_panel_3_snapshot`, `limiares_panel_9_snapshot` e `limiares_panel_10_snapshot` (ex: `days_since`, `days_until_expiry`). Esta sprint reclassificou as 8 entries afetadas como `estimator_only` (caixa qualitativa). Sprint futura cobrirá: (a) adicionar campos numéricos derivados aos payloads das RPCs panel_3/9/10; (b) threading de `source.key` no aggregator + `semantic_mode: 'gate' \| 'cooldown'` no helper §4.4 **(DROPPED em v2.6 — não existe atualmente no código; ver K-1 fix §4.4)**; (c) reintroduzir `items_inside_cooldown` no type `ImpactSeries` **(DROPPED em v2.6 — não existe atualmente no type; ver K-1 fix §4.1.5)**; (d) UI adapter "Dentro/Fora do cooldown" em `ImpactTable.tsx` **(DROPPED em v2.6 — não existe atualmente na UI; ver K-1 fix §5.1.4)**; (e) reclassificar as 8 entries de `estimator_only` para `covered`. Origem: K-1/K-3/K-4/K-5 fix v2.6. L-5 fix v2.7: notas DROPPED adicionadas para clarificar que esses 3 elementos NÃO existem no código atual desta sprint.

---
