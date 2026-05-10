# SPEC-sprint-paridade-skills

## Sumário executivo

Trazer a infraestrutura de competências canônicas para paridade arquitetural com a infraestrutura de funções canônicas. A sprint:

- Reescreve `canonical_skills` como `job_canonical_skills` (JCS) com paridade total a `job_canonical_roles` (JCR): estados canônicos, colunas estruturais, lógica de promoção por limiares, ressuscitação automática, merges, instrumentação de observabilidade.
- Instrumenta a sub-aba "Limiares" do dashboard admin com 10 painéis (incluindo distribuição de `confidence_median_at_promotion`).
- Unifica `canonical_role_cbo + canonical_role_cbo_links` em `canonical_cbo + canonical_cbo_links` com XOR `canonical_role_id`/`canonical_skill_id`.
- Cria triggers em runtime para `confidence_median` usando a fonte correta de cada lado: JCR via `function_orchestrator_items`, JCS via `job_posting_skills`. Filtros 120d + HAVING ≥ 5 (JCR) / ≥ 3 (JCS).
- Cria triggers de promoção `fn_promote_skill_on_threshold` e aposentadoria `fn_retire_skill_on_zero_vacancy` paritárias a JCR.
- Cria função `catchup_pending_skill_promotions` simétrica a `catchup_pending_promotions`. Expande esta última para incluir `status='retired'` na iteração, suportando ressuscitação JCR via catchup.
- Cria detector `detect_skill_merge_candidates` paritário a `detect_canonical_merge_candidates`.
- Adiciona triggers residuais de flagging `needs_opus_review` pós-promoção em JCR e JCS, substituindo o segundo bloco do CRON antigo.
- Evolui `pipeline_config` com `description`, `last_change_reason`, `last_changed_by_actor_id` e tabela paralela `pipeline_config_history` imutável com FORCE RLS.
- Migra 24 chaves hardcoded de promoção/Hard Gate/Opus review/merge/retirement/confidence runtime para `pipeline_config`.
- Implementa instrumentação de `layer` em `taxonomy_relations`, com XOR por `entity_type`.
- Aplica hardening `SET search_path` em funções de trigger.
- Refatora 3-query patterns em `compare.ts` e `SkillMap.tsx` para JOIN único.
- Corrige bug de UX no admin merge skills: endpoint `/api/admin/merge-skills` agora propaga flag de cross-type confirmation, frontend exibe modal de confirmação dupla quando skills de tipos diferentes são selecionadas.
- Executa cleanup imediato de schema de backup da própria sprint após validação SQL pós-aplicação, e drop direto de 9 tabelas de backup acumuladas de sprints anteriores que estão UNRESTRICTED em produção.

# Princípios arquiteturais

**Trigger autoritativo único por agregado, com fonte correta de cada lado.** Cada agregado em JCR e JCS é populado em runtime por trigger na sua tabela-fonte real:

| Agregado | JCR | JCS |
|---|---|---|
| `vacancy_count` | `job_postings` (ROW-level `fn_update_canonical_vacancy_count`) | `job_posting_skills` (ROW-level `fn_jps_recompute_jcs`) |
| `distinct_sources_count` | `job_postings` (ROW-level `recompute_distinct_sources_count`, filtro D11) | `job_posting_skills` (ROW-level com filtro de agências) |
| `confidence_median` | `function_orchestrator_items` (3 triggers ROW-level, filtros 120d + HAVING ≥ 5) | `job_posting_skills` (ROW-level, filtros 120d + HAVING ≥ 3) |
| `latest_posted_at` | (sem trigger nesta sprint) | `job_posting_skills` (ROW-level, recompute em DELETE/UPDATE) |

Atomicidade entre `vacancy_count` e `confidence_median` em JCR não é preservada (fontes diferentes), e nem é necessária — gates de promoção revalidam ambos via stored values.

**`pg_trigger_depth() > 1` é design intencional em triggers de promoção e aposentadoria.** Aparece em 3 triggers (`fn_promote_canonical_on_threshold`, `fn_promote_skill_on_threshold`, `fn_retire_skill_on_zero_vacancy`). O guard bloqueia cascade trigger→trigger evitando race com `auto_assign_family_to_canonical` durante curadoria em massa. **Não se aplica** aos triggers residuais de flagging (`fn_flag_needs_opus_review_jcr/jcs`) que são `BEFORE UPDATE OF confidence_median` e mutam `NEW.needs_opus_review` na mesma tupla — D-PS-56. Promoção legítima de canônicos pendentes que já atingiram threshold antes da sprint ou enquanto o trigger estava abafado é garantida pelos catchups (`catchup_pending_promotions()` para JCR, `catchup_pending_skill_promotions()` para JCS) que aplicam `UPDATE status='active', promoted_at=COALESCE(promoted_at, NOW()), ...` **diretamente** ao canônico — sem disparar trigger, sem efeitos colaterais MVCC, sem race entre `+1` e `-1` em `vacancy_count`. Eventos de promoção são inseridos explicitamente em `events` no corpo do catchup. **Assimetria entre roles e skills:** `fn_retire_canonical_on_zero_vacancy` (JCR em produção) **não tem** o guard `pg_trigger_depth() > 1`; `fn_retire_skill_on_zero_vacancy` (JCS, mig 25) tem. Manter como-está em JCR evita regressão; skill é mais defensivo por design da sprint.

**Ordem determinística de triggers BEFORE UPDATE.** Triggers em `job_canonical_skills` e `job_canonical_roles` que disparam antes de UPDATE de colunas agregadas executam em ordem alfabética por nome dentro do mesmo evento/timing (Postgres default). Para evitar dependência implícita quando um trigger BEFORE altera coluna que outro BEFORE observa, aplicar prefixo `z_` aos triggers de fim-de-cadeia (`z_trg_set_updated_at_jcr`, `z_trg_set_updated_at_jcs`) garantindo que rodem por último **entre os triggers BEFORE UPDATE da mesma tabela/evento**. O prefixo não ordena em relação a triggers AFTER UPDATE — esses sempre rodam após todos os BEFORE, independente do nome. Quando dependência for crítica, comentário inline explica a ordem esperada.

**Snapshot histórico vs estado operacional — distinção por coluna no contexto, não por tabela inteira.**

- Snapshot puro (não tocar em qualquer merge): `analysis_skill_matches`, `skill_enrichment_stats`, `curation_batch_metrics`, `resume_role_suggestions`, `rapidapi_usage_logs`.
- Híbrido por coluna: `resume_skill_enrichments` — `canonical_role_id` atualiza em `merge_canonicals`; `canonical_skill_id` atualiza em `merge_skills`.
- Operacional (atualiza em merge): `canonical_skills_summary`, `job_postings`, `job_canonical_role_sources`, `function_orchestrator_items`, `job_no_postings`, `submitted_job_skills`, `canonical_role_domain_links`, `canonical_cbo_links`, `taxonomy_relations`, `taxonomy_family_canonicals`.

**Soft-delete universal.** Canônicos role e skill nunca são hard-deletados. Operações usam `status='deprecated'` + `merged_into=winner_id`.

**Padrão de paridade evolutivo.** Replicar literalmente substituindo nomenclatura. Quando forma evoluída é XOR via `entity_type`, aplicar a forma evoluída. Assimetrias intencionais documentadas (skills sem `taxonomy_families`, sem `domain`, sem camada 2 — D-PS-03, D-PS-04).

**Calibração viva, sem hardcode.** Toda decisão numérica de pipeline lida de `pipeline_config` em tempo real via helper `getConfigValue`. Calibração via UPDATE no banco, sem deploy. Histórico via tabela `pipeline_config_history` imutável com FORCE RLS — UPDATE/DELETE bloqueados; INSERT permitido para `service_role` (na prática usado apenas pelo trigger `fn_pipeline_config_audit`); SELECT permissive.

**Zona Opus unificada com hard_gate e auto_min_confidence.** A faixa de confidence onde um canonical é flagado para review Opus tem como piso `hard_gate.min_confidence` (por construção: hard_gate filtra eventos individuais antes do agregado, logo a mediana nunca cai abaixo dele) e como teto `promotion.auto_min_confidence` (por convenção: zona Opus termina onde promoção automática começa). Não existem chaves separadas `opus_review.confidence_lower` e `opus_review.confidence_upper`: as duas funções consomem `hard_gate.min_confidence` e `promotion.auto_min_confidence` diretamente.

**Paridade de janelas role e skill.** `role.promotion.lookback_days = 60` e `skill.promotion.lookback_days = 60`. A simetria elimina assimetrias arquiteturais sem dados quantitativos que sustentassem janelas distintas, e é coerente com a constraint do fluxo B (cache de 100 vagas em janela 120d com filtro de relevância 15%, que captura naturalmente skills com cadência <= 8 dias — abaixo do limite do gate normal de 60d/5 vagas, cadência <= 12 dias).

**Reescrita explícita para mudanças semânticas.** Funções alteradas pela sprint que mudam apenas nome de tabela ou coluna usam `pg_get_functiondef + regexp_replace + EXECUTE` (mecânica). Funções que mudam lógica (guards adicionais, condições novas em WHERE, remoção de colunas dropadas) são reescritas com body completo na spec. A mistura de abordagens é deliberada: regex para mudanças triviais, reescrita para mudanças que poderiam ser corrompidas por substituição textual.

**Recompute em cadeia idempotente.** Triggers que cascateiam (job_postings → canonical, job_posting_skills → canonical_skill) usam padrão STATEMENT-level com tabela temp `ON COMMIT DROP` para deduplicar canonicais afetados, evitando complexidade O(N×M) em batch updates. Aplica-se a `fn_jps_recompute_jcs` (mig 27) e a `fn_update_canonical_vacancy_count` (mig nova paritária para JCR).

**Funções de trigger sem `SECURITY DEFINER`** (exceto `fn_pipeline_config_audit`, que precisa de owner privileges para INSERT em tabela com FORCE RLS). Triggers usam apenas `SET search_path TO 'public', 'pg_temp'`. RPCs (chamadas via `supabase.rpc()`) usam `SECURITY DEFINER + SET search_path`.

**INSERTs em `events` envoltos em `BEGIN/EXCEPTION`.** Padrão `WHEN foreign_key_violation/unique_violation/OTHERS THEN RAISE WARNING` — defesa em profundidade contra violações que abortariam transação inteira.

**Cleanup é imediato na janela ociosa.** Backup de sprint é transitório por design — quando aplicação completa e validação SQL pós-aplicação passa, schema de backup sai. Não há gate de espera. Aplica também ao cleanup de débito legado de sprints anteriores quando o ambiente está ocioso.

# Decisões cravadas

- **D-PS-01.** Sprint executada antes do benchmark E2E.
- **D-PS-02.** Reuso integral das tabelas de taxonomia com discriminador `entity_type IN ('role','skill')`.
- **D-PS-03.** Skills não entram em `taxonomy_families` nem em `taxonomy_family_canonicals`. Camada 2 não se aplica a skills.
- **D-PS-04.** Skills não ganham campo `domain`. Agrupamento via `skill_type`.
- **D-PS-05.** Apenas `canonical_skills` é renomeada para JCS.
- **D-PS-06.** Cache Redis segregado por `entity_type`. TTL 900s.
- **D-PS-07.** Limiares vivem em `pipeline_config` desde D0.
- **D-PS-08.** `is_emerging` removida.
- **D-PS-09.** `aliases TEXT[]` removida; sinônimos em `taxonomy_relations` com `entity_type='skill'`.
- **D-PS-10.** `canonical_skills_summary` mantida.
- **D-PS-11.** `creation_confidence NUMERIC NULL` em JCS e JCR.
- **D-PS-12.** Piloto de 50 ocupações CBO antes do batch full.
- **D-PS-13.** SYSTEM_PROMPT principal não é tocado.
- **D-PS-14.** Painéis Limiares via queries inline em `aggregateDayData()`.
- **D-PS-15.** Sub-abas "Cotidiano" e "Limiares" na aba Operacional.
- **D-PS-16.** Modos online (24h) e histórico (7d/30d) em endpoints distintos.
- **D-PS-17.** Sem deprecation com feature flag.
- **D-PS-18.** Trigger SQL é fonte autoritativa única em runtime para `vacancy_count`, `distinct_sources_count`, `confidence_median`, `latest_posted_at`.
- **D-PS-19.** Promoção pending→active via trigger consumindo limiares de `pipeline_config`. Para JCS o gate inclui `confidence_median >= <skill>.promotion.auto_min_confidence` (greenfield, sem NULL bypass). Para JCR o gate é exclusivamente `vacancy_count >= v_min_vacancies AND distinct_sources_count >= v_min_employers` — `confidence_median` em JCR existe apenas como snapshot pós-promoção (`confidence_median_at_promotion`) e como insumo do trigger residual de flagging. A description de `role.promotion.auto_min_confidence` em `pipeline_config` deixa isso explícito para o operador admin: a chave é seedada para uso em painéis e simulações, mas **não é consumida pelo trigger de promoção JCR**.
- **D-PS-20.** Aposentadoria em 3 caminhos: trigger, CRON, Opus. Todos chamam `retire_canonical(id, reason)` para JCR ou `retire_canonical(id, reason, 'skill')` para JCS via assinatura estendida com overloading.
- **D-PS-21.** Ressuscitação automática `retired→active` aplica a JCR e JCS via trigger de promoção (`OLD.status IN ('pending','retired')`). Força `needs_opus_review=TRUE` em ambos. `promoted_at = COALESCE(promoted_at, NOW())` preserva data original.
- **D-PS-22.** Detecção de `merge_candidate` para skills via `detect_skill_merge_candidates()`. Threshold por `pipeline_config` chave `skill.merge_candidate.cosine_threshold`.
- **D-PS-23.** `taxonomy_versions` permanece unificada.
- **D-PS-24.** Schema da junction CBO espelha o existente.
- **D-PS-25.** Prompt do batch CBO segue padrão de `lib/prompts/resume-analysis.ts`.
- **D-PS-26.** Sub-aba "Limiares" segue padrão dos drilldowns existentes.
- **D-PS-27.** `pipeline_calibration_metrics` ganha 5 `metric_name` para skills paritários aos existentes para roles e CBO.
- **D-PS-28.** Snapshots de promoção populados apenas em promoções pós-Sprint.
- **D-PS-29.** Unificação `canonical_role_cbo* → canonical_cbo*` com XOR `canonical_role_id`/`canonical_skill_id`.
- **D-PS-30.** `taxonomy_relations.layer IN (0,1,2,3)` + CHECK XOR (skill ≠ 2); backfill `layer=1` nos registros existentes; NOT NULL.
- **D-PS-31.** Princípio snapshot vs operacional aplica por coluna no contexto.
- **D-PS-32.** `resume_skill_enrichments` é caso híbrido por coluna: `canonical_role_id` atualiza em `merge_canonicals` (mig 36); `canonical_skill_id` atualiza em **ambas** `merge_skills` (pipeline, mig 35) e `merge_canonical_skills` (admin, mig 35) com NOT EXISTS guard + DELETE residual. Sem isso, callers TS de `app/api/optimization/validate/route.ts` e `app/api/optimization/resolve/route.ts` retornam label de loser deprecated em vez do winner pós-merge. Anexo B documenta o comportamento esperado.
- **D-PS-33.** Cache invalidado em todos os caminhos de `opus-arbitration.ts` após cada `processOpusXXX` bem-sucedido.
- **D-PS-34.** RPC `merge_canonical_skills` (plural, admin UI) preservada com assinatura existente. Endpoint admin TS aceita campo `cross_type_confirmed: boolean` e propaga para a RPC.
- **D-PS-35.** Triggers sem `SECURITY DEFINER` (exceto `fn_pipeline_config_audit`, que precisa de owner privileges para INSERT em tabela com FORCE RLS). RPCs com `SECURITY DEFINER + SET search_path`.
- **D-PS-36.** Hardening: 3 funções de vacancy_count recebem `SET search_path` se ainda não tiverem.
- **D-PS-37.** Padrão actor: `actor='system'`, `actor_id='00000000-0000-0000-0000-000000000001'::uuid`.
- **D-PS-38.** Refactor JOIN em `compare.ts` + `SkillMap.tsx`.
- **D-PS-39.** Branch arqueológico não existe nas funções de catchup. Análise matemática mostra redundância completa em JCR (gate normal 3/60d é mais permissivo que arqueológico 10/120d para qualquer cadência) e gap estreito em JCS sem caso de uso real (filtro de relevância 15% do fluxo B captura apenas skills com cadência <= 8d, dentro do gate normal). Catchups têm apenas branch normal: `vacancy_count >= min_vacancies AND distinct_employers >= min_distinct_employers AND latest_posted_at >= NOW() - INTERVAL lookback_days`. Casos edge de skills com volume histórico esparso requerem ação admin manual via Painel 5 (Pending stuck).
- **D-PS-40.** `is_active` em JCS preservada. INACTIVATE escreve `status='deprecated'`, trigger `fn_sync_jcs_is_active` sincroniza.
- **D-PS-41.** Todos os callers hardcoded migrados para leitura de config via `getConfigValue`.
- **D-PS-42.** Triggers JCR são heterogêneos por agregado: `vacancy_count` via ROW-level `fn_update_canonical_vacancy_count` em job_postings, `distinct_sources_count` via ROW-level `recompute_distinct_sources_count` em job_postings (filtro D11 — `is_recruitment_agency=FALSE`), `confidence_median` via 3 ROW-level em function_orchestrator_items (esta sprint). JCS usa ROW-level em job_posting_skills para todos os agregados via helper `fn_jps_recompute_jcs`.
- **D-PS-43.** `auto_assign_family_to_canonical(NEW.id, TRUE)` preservado em `fn_promote_canonical_on_threshold` em caso de promoção. Skills NÃO chamam função análoga (D-PS-03).
- **D-PS-44.** `confidence_median` JCR vem de `function_orchestrator_items.confidence`, não de `job_postings`. Triggers em FOI: AFTER INSERT/UPDATE/DELETE com filtro de janela `role.confidence.lookback_days` (120) + HAVING `>= role.confidence.min_count` (5) + CTE para preservar valor anterior se mínimo não atingido.
- **D-PS-45.** Triggers de runtime preservam filtros lidos de `pipeline_config`: `role.confidence.lookback_days/min_count` para JCR, `skill.confidence.lookback_days/min_count` para JCS. Janela diferente da janela de promoção (`*.promotion.lookback_days = 60d`) é intencional — promoção exige emergência recente, runtime preserva estabilidade média de longo prazo.
- **D-PS-46.** Trigger residual de flagging `needs_opus_review = TRUE` em canônicos `active` quando `confidence_median < <entity>.promotion.auto_min_confidence` (teto da zona Opus por convenção), com bloqueio de re-flag por `<entity>.opus_review.cooldown_days` (90 default) via `last_opus_review_at`. Aplica a JCR e JCS.
- **D-PS-47.** `last_opus_review_at TIMESTAMPTZ NULL` em JCS (paridade com JCR).
- **D-PS-48.** `last_opus_review_at` é setado via `opus-arbitration.ts` em `processOpusXXX` ao final de cada arbitragem bem-sucedida em todos os 7 verdicts: KEEP, REFINE, APPROVE, MERGE, INACTIVATE, CREATE_NEW, REJECT. Sem isso, o cooldown de `<entity>.opus_review.cooldown_days` do trigger residual nunca fecha. Helper `finalizeOpusArbitration` aplica em todos os success paths (D-PS-60).
- **D-PS-49.** Eventos canônicos role e skill registrados em `events` com `resource_type='job_canonical_role'` ou `'canonical_skill'`. `resource_id` UUID direto. Coluna `reason TEXT` para texto de auditoria primário; coluna `metadata JSONB` para payload estruturado com flags. Convenções coexistem por contexto.
- **D-PS-50.** Padrão `INSERT + ON CONFLICT DO NOTHING + SELECT` no batch CBO para preservar `skill_type` da primeira inserção em caso de race.
- **D-PS-51.** Endpoint `/api/admin/merge-skills` aceita campo opcional `cross_type_confirmed: boolean` no body. Quando RPC retorna `requires_cross_type_confirmation: true` (skills de tipos diferentes sem flag), endpoint responde 409 com payload estruturado para o frontend exibir confirmação dupla. O 409 carrega payload `{error, source_type, target_type, requires_cross_type_confirmation: true}` em vez de erro HTTP padrão para preservar callers existentes que esperam `data?.error` na response.
- **D-PS-52.** Zona Opus consome `hard_gate.min_confidence` (piso) e `promotion.auto_min_confidence` (teto) por construção, sem chaves intermediárias. Tanto o trigger residual de flagging (`fn_flag_needs_opus_review_*`) quanto Painel 2 (drilldown da zona) leem essas duas chaves diretamente.
- **D-PS-53.** `skill.promotion.lookback_days = 60` em paridade com `role.promotion.lookback_days`. Equiparação justificada por simetria arquitetural e pela constraint do fluxo B: skill que importa para usuário tem cadência <= 8 dias (15% de 100 vagas em 120d), abaixo do limite do gate normal (cadência <= 12 dias com janela 60d e gate 5).
- **D-PS-54.** Branch arqueológico não existe — eliminado por análise matemática + constraint do fluxo B (ver D-PS-39).
- **D-PS-55.** Triggers de recompute em cadeia externa usam padrão STATEMENT-level com tabela temp `ON COMMIT DROP`. Aplica a `fn_jps_recompute_jcs` (mig 27) e `fn_update_canonical_vacancy_count` (mig 27b paritária). Reduz O(N×M) para O(N) em batch updates de `job_postings` e `job_posting_skills`.
- **D-PS-56.** Triggers `fn_flag_needs_opus_review_jcr/jcs` são `BEFORE UPDATE OF confidence_median` com mutação em memória (`NEW.needs_opus_review := TRUE; RETURN NEW`). Elimina segunda gravação física da tupla (WAL duplo) e blinda contra recursão se trigger AFTER for adicionado no futuro.
- **D-PS-57.** `merge_canonicals` e `merge_skills` ganham guard de loser status: `IF v_loser.status NOT IN ('active','pending') THEN RAISE EXCEPTION` antes das fases de redirect. Previne sobrescrita de `merged_into` em cadeias de merge consecutivos.
- **D-PS-58.** `merge_skills` reescrita explicitamente para remover UPDATEs em `analysis_skill_matches.canonical_skill_id` e `.matched_via_similar_skill_id` — paridade com princípio "snapshot puro" do contexto. Análises antigas preservam o canonical originalmente matched; UI lateral lida com exibição de canonical deprecated.
- **D-PS-59.** Catchups (mig 31 e 32) retornam JSONB com `{promoted, archaeological, unchanged, resurrected}`. Campo `archaeological` permanece com valor zero por compatibilidade com caller `pipeline-maintenance/route.ts:105`. Campo `resurrected` é novo e capturável por painéis admin.
- **D-PS-60.** Funções com mudança apenas de identificador (renomeação de tabela/coluna) usam `pg_get_functiondef + regexp_replace + EXECUTE`. Funções com mudança lógica (guards adicionais, condições novas, remoção de colunas dropadas) são reescritas com body completo na spec. **Reescritas explícitas preservam a ordem de parâmetros da assinatura de produção** quando ela existe — convenção arquitetural existente (3+ funções, 5+ callers TS) é mais forte que estética. Especificamente: `merge_canonicals` e `merge_skills` mantêm `(p_loser_id, p_winner_id, p_actor, p_actor_id)` e `merge_canonical_skills` mantém `(p_loser_id, p_winner_id, p_decided_by_actor_id, p_reason, p_cross_type_confirmed)`. Preservar a ordem reduz superfície de mudança nos callers (apenas rename de parâmetros + ajuste de tipo do 4º arg de `TEXT 'system'` para `UUID actor_id`), eliminando classe de erro humano por reordenação. Lista das reescritas explícitas: `merge_canonicals` (mig 36), `merge_skills` (mig 35), `merge_canonical_skills` (mig 35), `o3_opus_canonical_label_disputes` (mig 12b — adiciona `AND tr.entity_type = 'role'`), `process_opus_disagree` (mig 12b — atualiza chamada interna de `merge_canonicals` para 4-arg com `p_actor_id`), `trg_jcr_set_updated_at` (mig 02b — sem `is_emerging` e sem `confidence_median` no watch list).
- **D-PS-61.** Mig 12 ordem definitiva: ADD COLUMNS (`target_role_id`, `target_skill_id`, `entity_type`) → backfill `target_role_id = target_canonical_id, entity_type='role'` → DROP INDEX `taxonomy_relations_source_term_key` (single-column) → ADD CONSTRAINT UNIQUE composto `(source_term, entity_type)` → CREATE INDEX nas duas colunas novas → mig 12b reescreve 3 funções enquanto `target_canonical_id` ainda existe → ALTER TABLE DROP COLUMN `target_canonical_id` CASCADE.

# Limitações conhecidas

- **LK-PS-01.** `creation_confidence` sem backfill histórico. Canônicos pré-sprint têm valor `NULL`. Não impacta gates.
- **LK-PS-02.** `vacancy_count_at_promotion`, `distinct_sources_count_at_promotion`, `confidence_median_at_promotion` sem backfill histórico para promoções pré-sprint. Painel 7 (distribuição) só tem dados pós-sprint.
- **LK-PS-03.** Triggers SQL não invalidam Redis nativamente. TTL 900s mitiga drift máximo de 15 minutos. Cache invalidate explícito em `opus-arbitration.ts`.
- **LK-PS-04.** Camada 2 (`taxonomy-cache findFamilyMatch`) não existe para skills. `tfc_target_xor_check` impede.
- **LK-PS-05.** Embedding de JCS gerado offline pelo batch CBO; runtime via CRON `auto_assign_family_to_canonical` não aplica a skills. Skills criadas via `llm_extractor` não têm embedding até batch posterior.
- **LK-PS-06.** 25 ocupações CBO sem `summary_description` ficam fora do batch.
- **LK-PS-07.** Coluna `aliases` em JCS dropada. Refator de `lookup_canonical_skill_by_normalized_alias` ocorre antes do drop para evitar janela órfã.
- **LK-PS-08.** Custo do batch CBO ±50% da estimativa (R$ 15-25 central).
- **LK-PS-09.** 10 painéis Limiares não cobrem todos os ângulos. Faltantes: distribuição de `latest_posted_at` por banda, custo Opus por canônico, taxa de merge auto-resolvido vs manual.
- **LK-PS-10.** Painéis Limiares modo histórico vazios nas primeiras 1-2 semanas pós-deploy. Esperado — `events` e `pipeline_calibration_metrics` precisam acumular dados.
- **LK-PS-11.** Snapshots imutáveis acumulam refs a `deprecated`/`merged_into`. UI fora do escopo — mostra label de loser com `[DEPRECATED]` em hover.
- **LK-PS-12.** Healing automático não implementado para corrupção de stored values em JCR ou JCS. Triggers são fonte autoritativa em runtime; corrupção exige SQL ad-hoc. RPCs `reconcile_canonical_role(id)` e `reconcile_canonical_skill(id)` são débito documentado.
- **LK-PS-13.** Atomicidade entre `vacancy_count` e `confidence_median` em JCR não é preservada (fontes diferentes). Não é necessária — gate de promoção em `fn_promote_canonical_on_threshold` revalida ambos via stored values.
- **LK-PS-14.** Triggers de runtime e gates de promoção usam janelas diferentes:
  - Runtime confidence (FOI para JCR, JPS para JCS): janela `<entity>.confidence.lookback_days` (120d) + HAVING `>= <entity>.confidence.min_count` (5 para role, 3 para skill).
  - Gate de promoção: janela `<entity>.promotion.lookback_days` (60d).

  Skills/roles com confidence baixa em janela longa mas alta em janela curta podem promover (intencional — emergência prevalece sobre estabilidade média).

- **LK-PS-15.** Cross-type confirmation no admin merge skills requer dois POSTs sequenciais quando admin escolhe skills de tipos diferentes:
  1. Primeiro POST com `cross_type_confirmed: false` (default) → backend retorna 409 `requires_cross_type_confirmation: true`.
  2. Frontend exibe modal de confirmação com `source_type` e `target_type` explícitos.
  3. Admin clica "Confirmar mesmo assim" → segundo POST com `cross_type_confirmed: true` → merge executa.

- **LK-PS-16.** `auto_assign_family_to_canonical(NEW.id, TRUE)` é chamada tanto pelo trigger `fn_promote_canonical_on_threshold` (em depth=1) quanto pelo catchup `catchup_pending_promotions_extend()` (em depth=0) durante ressuscitação. A função é idempotente por design: `INSERT INTO taxonomy_family_canonicals ... ON CONFLICT (family_id, canonical_role_id) DO NOTHING`. Dupla execução é inofensiva mas custa duas chamadas embedding+SELECT. Otimização (cache de família por canonical_id) é débito pós-MVP.

- **LK-PS-17.** Assimetria intencional entre `fn_retire_canonical_on_zero_vacancy` (produção, sem guard `pg_trigger_depth() > 1`) e `fn_retire_skill_on_zero_vacancy` (mig 25, com guard). A função JCR foi cravada antes da convenção e reescrevê-la implicaria revalidação completa de comportamento histórico — risco maior que o benefício de paridade cosmética. Skill nasce com guard. Padronização para JCR fica como débito documentado para sprint posterior.

- **LK-PS-18.** O pipeline de ingestão de roles existente em produção pode sofrer do mesmo bug de **slug collision pós-soft-delete** que esta sprint corrige no lado skill (RPC `resolve_active_canonical_by_slug`, §4.4). Quando uma role é deprecated via `merge_canonicals`, seu `slug` permanece em `job_canonical_roles`. Re-ingestão do mesmo label pode prender nova vaga em canônico morto. A RPC `resolve_active_canonical_by_slug` foi escrita com discriminador `entity_type` para servir aos dois lados; auditoria do pipeline de ingestão de roles para adoção é débito documentado para sprint posterior — fora do escopo desta sprint, que é paridade de skills.

- **LK-PS-19.** Trigger `trg_jcr_set_updated_at` foi recriado em mig 02b removendo simultaneamente `is_emerging` (dropada por mig 02) e `confidence_median` do watch list — paridade com `fn_jcs_set_updated_at` (mig 26) que já tinha esse padrão. A motivação para excluir `confidence_median`: mig 29 cria 3 triggers em `function_orchestrator_items` que escrevem em `jcr.confidence_median` em alta cardência; mantê-lo no watch list bumparia `updated_at` indiscriminadamente e corromperia o anti-starvation `ORDER BY jcr.updated_at ASC` em `detectStaleCanonicals`. A correção foi feita nesta sprint porque mig 02b já reescreve o trigger por outro motivo — fazer também o fix de `confidence_median` reduz superfície de débito.

- **LK-PS-20.** Trigger ROW-level `fn_promote_canonical_on_threshold` (JCR, em produção, não reescrito por esta sprint) ainda tem branch arqueológico vestigial — caminho que flagga `needs_opus_review = TRUE` quando thresholds atingidos sem postings recentes. Pelo mesmo argumento matemático que removeu o branch dos catchups (D-PS-39) e que dispensou o branch em `fn_promote_skill_on_threshold` (criada nesta sprint sem ele), o branch é dead code no caminho ROW-level também: trigger só dispara em UPDATE de agregados, e agregados só sobem por INSERT recente. Reescrita de `fn_promote_canonical_on_threshold` para remover o branch é débito documentado para sprint posterior. A mig 28 preserva validação que confirma o branch presente — reflete realidade de produção, não direção arquitetural.

  Por design — decisões de baixa reversibilidade requerem confirmação explícita.

# Convenções

Padrão de eventos para skills E roles (todos os INSERTs em `events`):

```
{
  event_name: <texto>,
  resource_type: 'canonical_skill' | 'job_canonical_role' | ...,
  resource_id: <UUID>,                            -- UUID direto, NUNCA ::TEXT
  actor: 'system',
  actor_id: '00000000-0000-0000-0000-000000000001'::uuid,
  previous_state: jsonb (opcional),
  new_state: jsonb (opcional),
  reason: <texto> (opcional, coluna explícita),    -- texto de auditoria primário
  metadata: jsonb (opcional)                       -- payload estruturado
}
```

`reason` (coluna TEXT) e `metadata` (coluna JSONB) coexistem em `events`. Convenção: `reason` para texto livre de auditoria primário (ex: `retire_canonical`, `merge_canonicals`); `metadata` para payload estruturado com flags (ex: `fn_promote_canonical_on_threshold` com `is_resurrection`, `has_recent_vacancy`, `thresholds`). Quando ambos fazem sentido, podem ser usados juntos.

Migrations idempotentes onde possível, aplicadas pelo Claude Code via conexão direta ao Supabase. FKs com `ON DELETE CASCADE`/`SET NULL`/`RESTRICT` explícito.

INSERTs em `events` envoltos em `BEGIN/EXCEPTION WHEN foreign_key_violation/unique_violation/OTHERS THEN RAISE WARNING`.

---

# Parte 2 — Schema SQL

56 migrations executadas em ordem. Numeração `paridade-skills/NN_descricao.sql` (com `02b`, `05b`, `12b`, `27b`, `42b` e `42c` quebrando a numeração estritamente sequencial para preservar ordem cronológica de aplicação). As migrations `42b` (RPC `resolve_active_canonical_by_slug`) e `42c` (RPC `seed_skill_batch_from_cbo`) são apresentadas em §4.4 e §4.5 — são RPCs novas associadas a arquivos TS novos, e a redação combina spec da função com spec do caller. Lista canônica completa em Anexo A.

## Bloco A — Backup

### 2.1 — `01_backup_pre_execution.sql`

```sql
CREATE SCHEMA IF NOT EXISTS backup_paridade_skills;

CREATE TABLE backup_paridade_skills.canonical_skills_pre AS SELECT * FROM canonical_skills;
CREATE TABLE backup_paridade_skills.taxonomy_relations_pre AS SELECT * FROM taxonomy_relations;
CREATE TABLE backup_paridade_skills.taxonomy_family_canonicals_pre AS SELECT * FROM taxonomy_family_canonicals;
CREATE TABLE backup_paridade_skills.canonical_role_cbo_pre AS SELECT * FROM canonical_role_cbo;
CREATE TABLE backup_paridade_skills.canonical_role_cbo_links_pre AS SELECT * FROM canonical_role_cbo_links;
CREATE TABLE backup_paridade_skills.pipeline_config_pre AS SELECT * FROM pipeline_config;

DO $$ DECLARE table_count INT; BEGIN
  SELECT COUNT(*) INTO table_count FROM information_schema.tables
  WHERE table_schema = 'backup_paridade_skills' AND table_name LIKE '%_pre';
  IF table_count < 6 THEN RAISE EXCEPTION 'Backup tabelas incompleto: %', table_count; END IF;
END $$;
```

## Bloco B — Drops housekeeping

### 2.2 — `02_drop_is_emerging.sql`

```sql
ALTER TABLE canonical_skills DROP COLUMN IF EXISTS is_emerging;
ALTER TABLE job_canonical_roles DROP COLUMN IF EXISTS is_emerging;
```

### 2.2b — `02b_recreate_jcr_set_updated_at_without_is_emerging.sql`

```sql
-- Recria trg_jcr_set_updated_at removendo simultaneamente:
--   1) is_emerging (dropada por mig 02)
--   2) confidence_median do watch list — paridade com fn_jcs_set_updated_at (mig 26)
-- A exclusão de confidence_median é deliberada: mig 29 cria 3 triggers em FOI que
-- escrevem em jcr.confidence_median em alta cardência. Mantê-lo no watch list bumparia
-- updated_at indiscriminadamente e corromperia o ORDER BY de detectStaleCanonicals.
-- DEVE rodar imediatamente após mig 02. Nenhum write em job_canonical_roles
-- pode ocorrer entre mig 02 e mig 02b — pause CRONs e ingestão.

CREATE OR REPLACE FUNCTION trg_jcr_set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF (
    NEW.label IS DISTINCT FROM OLD.label OR
    NEW.slug IS DISTINCT FROM OLD.slug OR
    NEW.status IS DISTINCT FROM OLD.status OR
    NEW.source IS DISTINCT FROM OLD.source OR
    NEW.merged_into IS DISTINCT FROM OLD.merged_into OR
    NEW.alias_of_id IS DISTINCT FROM OLD.alias_of_id OR
    NEW.promoted_at IS DISTINCT FROM OLD.promoted_at OR
    NEW.rejected_reason IS DISTINCT FROM OLD.rejected_reason OR
    NEW.blacklist_expiry_at IS DISTINCT FROM OLD.blacklist_expiry_at OR
    NEW.needs_opus_review IS DISTINCT FROM OLD.needs_opus_review OR
    NEW.human_validated_at IS DISTINCT FROM OLD.human_validated_at
  ) THEN
    NEW.updated_at = NOW();
  END IF;
  RETURN NEW;
END; $$;

DO $$ BEGIN
  IF (SELECT prosrc FROM pg_proc WHERE proname = 'trg_jcr_set_updated_at') ~ 'is_emerging' THEN
    RAISE EXCEPTION 'trg_jcr_set_updated_at ainda referencia is_emerging';
  END IF;
  IF (SELECT prosrc FROM pg_proc WHERE proname = 'trg_jcr_set_updated_at') ~ 'confidence_median' THEN
    RAISE EXCEPTION 'trg_jcr_set_updated_at ainda monitora confidence_median (LK-PS-19)';
  END IF;
END $$;
```

### 2.3 — `03_lookup_skill_alias_stub.sql`

Stub no-op de validação. Função `lookup_canonical_skill_by_normalized_alias` em produção lê `aliases TEXT[]` em `canonical_skills`. Refator real ocorre em mig 12 após `taxonomy_relations.target_skill_id` existir. Esta migration apenas garante que a função existe antes de prosseguir.

```sql
DO $$ BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_proc WHERE proname = 'lookup_canonical_skill_by_normalized_alias'
  ) THEN
    RAISE EXCEPTION 'lookup_canonical_skill_by_normalized_alias ausente — sprint não pode prosseguir';
  END IF;
END $$;
```

### 2.4 — `04_drop_aliases_array_with_stub.sql`

```sql
-- Stub para lookup_canonical_skill_by_normalized_alias antes do drop de aliases.
-- A função real será reescrita em mig 12, após taxonomy_relations.target_skill_id existir.
-- Entre mig 04 e mig 12 (11 migrations), o stub retorna NULL com aviso, evitando
-- runtime error em callers que ainda invoquem a função.
CREATE OR REPLACE FUNCTION lookup_canonical_skill_by_normalized_alias(p_normalized_label TEXT)
RETURNS UUID LANGUAGE plpgsql AS $$
BEGIN
  RAISE NOTICE 'lookup_canonical_skill_by_normalized_alias em transição (mig 04 → 12). Refator real chega em mig 12.';
  RETURN NULL;
END; $$;

DROP INDEX IF EXISTS idx_canonical_skills_aliases;
ALTER TABLE canonical_skills DROP COLUMN IF EXISTS aliases;
```

## Bloco C — Rename + paridade JCS

### 2.5 — `05_rename_canonical_skills_to_jcs.sql`

```sql
ALTER TABLE canonical_skills RENAME TO job_canonical_skills;

ALTER TABLE job_canonical_skills RENAME CONSTRAINT canonical_skills_pkey TO job_canonical_skills_pkey;
ALTER TABLE job_canonical_skills RENAME CONSTRAINT canonical_skills_status_check TO job_canonical_skills_status_check;
ALTER TABLE job_canonical_skills RENAME CONSTRAINT canonical_skills_skill_type_check TO job_canonical_skills_skill_type_check;
ALTER TABLE job_canonical_skills RENAME CONSTRAINT canonical_skills_behavioral_group_check TO job_canonical_skills_behavioral_group_check;

ALTER INDEX IF EXISTS idx_canonical_skills_type RENAME TO idx_jcs_skill_type;
ALTER INDEX IF EXISTS idx_canonical_skills_behavioral_group RENAME TO idx_jcs_behavioral_group;
ALTER INDEX IF EXISTS idx_canonical_skills_needs_opus_review RENAME TO idx_jcs_needs_opus_review;
ALTER INDEX IF EXISTS uq_canonical_skills_label_normalized RENAME TO uq_jcs_label_normalized;
ALTER INDEX IF EXISTS idx_cs_merged_into RENAME TO idx_jcs_merged_into;
ALTER INDEX IF EXISTS idx_cs_status RENAME TO idx_jcs_status;

ALTER TRIGGER trg_sync_cs_is_active ON job_canonical_skills RENAME TO trg_sync_jcs_is_active;
ALTER FUNCTION fn_sync_canonical_skill_is_active() RENAME TO fn_sync_jcs_is_active;
```

### 2.5b — `05b_recreate_jcs_dependent_functions.sql`

**Crítico — deve rodar imediatamente após 2.5, antes de qualquer write em `job_posting_skills` ou `submitted_job_skills`** (`fn_redirect_deprecated_skill_junction` é trigger handler dessas tabelas e quebra com referência a `canonical_skills` inexistente).

15 funções referenciam textualmente `canonical_skills` no body. Após o rename da tabela, todas quebram em runtime. Esta migration aplica `pg_get_functiondef + replace + EXECUTE` em cada uma.

**Atenção crítica ao placeholder:** 9 das 15 funções referenciam **também** `canonical_skills_summary` (que **não é renomeada** — D-PS-10). Replace ingênuo `canonical_skills → job_canonical_skills` corrompe `canonical_skills_summary` em `job_canonical_skills_summary` que não existe. Padrão obrigatório:

```
v_body := replace(v_body, 'canonical_skills_summary', '##CSS##');
v_body := replace(v_body, 'canonical_skills', 'job_canonical_skills');
v_body := replace(v_body, '##CSS##', 'canonical_skills_summary');
```

```sql
-- Helper inline para não duplicar o padrão 15 vezes
DO $$
DECLARE
  fn_name TEXT;
  v_body TEXT;
  fn_list TEXT[] := ARRAY[
    -- Categoria simples (só canonical_skills, sem _summary):
    'fn_redirect_deprecated_skill_junction',
    'get_skill_merge_suggestions',
    'get_top_skills_for_canonical',
    'lookup_canonical_skill_by_normalized_label',
    'refresh_canonical_skills_confidence_median',
    'lookup_canonical_skill_by_normalized_alias',
    -- Categoria placeholder (canonical_skills + canonical_skills_summary):
    'merge_canonical_skill',
    'merge_canonical_skills',
    'merge_skills',
    'refresh_canonical_skills_summary',
    'refresh_canonical_skills_summary_single',
    'refresh_similarity_thresholds',
    'suggest_functions_by_skills'
  ];
BEGIN
  FOREACH fn_name IN ARRAY fn_list LOOP
    SELECT pg_get_functiondef(p.oid) INTO v_body
    FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
    WHERE n.nspname = 'public' AND p.proname = fn_name
    LIMIT 1;

    IF v_body IS NULL THEN
      RAISE EXCEPTION 'Função % não encontrada — backup pode estar incompleto', fn_name;
    END IF;

    -- Padrão placeholder defensivo (idempotente para funções sem _summary):
    v_body := replace(v_body, 'canonical_skills_summary', '##CSS##');
    v_body := replace(v_body, 'canonical_skills', 'job_canonical_skills');
    v_body := replace(v_body, '##CSS##', 'canonical_skills_summary');

    -- Garante CREATE OR REPLACE para idempotência:
    v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');

    EXECUTE v_body;
  END LOOP;
END $$;

-- Categoria 4 (internal schema, dependência dual canonical_skills + CBO):
DO $$
DECLARE v_body TEXT;
BEGIN
  -- internal.reset_taxonomy_core: refs a canonical_skills + canonical_skills_summary + canonical_role_cbo_links
  -- Esta migration trata APENAS canonical_skills*. CBO é tratado pela mig 22 (outro pass).
  SELECT pg_get_functiondef(p.oid) INTO v_body
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname = 'internal' AND p.proname = 'reset_taxonomy_core'
  LIMIT 1;

  IF v_body IS NOT NULL THEN
    v_body := replace(v_body, 'canonical_skills_summary', '##CSS##');
    v_body := replace(v_body, 'canonical_skills', 'job_canonical_skills');
    v_body := replace(v_body, '##CSS##', 'canonical_skills_summary');
    v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
    EXECUTE v_body;
  END IF;

  SELECT pg_get_functiondef(p.oid) INTO v_body
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname = 'internal' AND p.proname = 'reset_taxonomy_extended'
  LIMIT 1;

  IF v_body IS NOT NULL THEN
    v_body := replace(v_body, 'canonical_skills_summary', '##CSS##');
    v_body := replace(v_body, 'canonical_skills', 'job_canonical_skills');
    v_body := replace(v_body, '##CSS##', 'canonical_skills_summary');
    v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
    EXECUTE v_body;
  END IF;
END $$;

-- Validação pós-aplicação: zero ocorrências de 'canonical_skills' isolado nos bodies.
DO $$
DECLARE bad_count INT;
BEGIN
  SELECT COUNT(*) INTO bad_count
  FROM pg_proc p
  JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname IN ('public', 'internal')
    AND p.proname IN (
      'fn_redirect_deprecated_skill_junction','get_skill_merge_suggestions',
      'get_top_skills_for_canonical','lookup_canonical_skill_by_normalized_label',
      'refresh_canonical_skills_confidence_median','lookup_canonical_skill_by_normalized_alias',
      'merge_canonical_skill','merge_canonical_skills','merge_skills',
      'refresh_canonical_skills_summary','refresh_canonical_skills_summary_single',
      'refresh_similarity_thresholds','suggest_functions_by_skills',
      'reset_taxonomy_core','reset_taxonomy_extended'
    )
    AND p.prosrc ~ '\mcanonical_skills\M';
  -- word boundary \\m\\M já distingue 'canonical_skills' standalone de 'canonical_skills_summary'.
  -- A condição extra '!~ canonical_skills_summary' mascararia funções broken que tenham ambos os termos.

  IF bad_count > 0 THEN
    RAISE EXCEPTION 'Restam % funções com canonical_skills sem rename', bad_count;
  END IF;
END $$;
```

**Nota cirúrgica para `merge_canonical_skills`:** esta função tem **dois** problemas de runtime que esta sprint corrige em **dois passes separados**. Pass 1 é esta mig 2.5b (rename `canonical_skills → job_canonical_skills` no body). Pass 2 é a mig 2.17 (`17_smd_unique_expression.sql`) que recria `idx_smd_pair_unique` via expression `LEAST/GREATEST` e reescreve o `ON CONFLICT (source_id, target_id)` da função para casar com a nova UNIQUE. Os dois passes são idempotentes; ordem garantida pelo plano de execução.

**Observação sobre `lookup_canonical_skill_by_normalized_alias`:** é reescrita do zero em mig 2.13 (taxonomy_relations entity_type). Esta mig 2.5b apenas garante que a versão atual não quebre entre 2.5 e 2.13 — se 2.13 chega antes de qualquer caller, o pass desta mig é descartado pelo `CREATE OR REPLACE` da 2.13 (sem prejuízo, idempotente).


### 2.6 — `06_validate_fks_after_rename.sql`

Validação por listagem nominal — bloqueia regressão silenciosa caso ambiente de produção adicione/remova FK fora do escopo desta sprint.

```sql
DO $$
DECLARE
  expected_fks TEXT[] := ARRAY[
    'canonical_skills_summary_canonical_skill_id_fkey',
    'job_canonical_skills_alias_of_id_fkey',
    'job_canonical_skills_merged_into_fkey',
    'job_canonical_skills_taxonomy_content_version_id_fkey',
    'job_posting_skills_canonical_skill_id_fkey',
    'resume_skill_enrichments_canonical_skill_id_fkey',
    'skill_merge_decisions_source_id_fkey',
    'skill_merge_decisions_target_id_fkey',
    'submitted_job_skills_canonical_skill_id_fkey'
  ];
  fk TEXT;
  missing TEXT[] := ARRAY[]::TEXT[];
BEGIN
  FOREACH fk IN ARRAY expected_fks LOOP
    IF NOT EXISTS (
      SELECT 1 FROM information_schema.table_constraints
      WHERE constraint_type = 'FOREIGN KEY' AND constraint_name = fk
    ) THEN
      missing := missing || fk;
    END IF;
  END LOOP;

  IF array_length(missing, 1) > 0 THEN
    RAISE EXCEPTION 'FKs ausentes após rename JCS: %', missing;
  END IF;
END $$;
```

### 2.7 — `07_jcs_add_columns.sql`

```sql
ALTER TABLE job_canonical_skills
  ADD COLUMN IF NOT EXISTS slug TEXT NULL,
  ADD COLUMN IF NOT EXISTS embedding VECTOR(768) NULL,
  ADD COLUMN IF NOT EXISTS vacancy_count INT NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS distinct_sources_count INT NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS latest_posted_at TIMESTAMPTZ NULL,
  ADD COLUMN IF NOT EXISTS usage_count INT NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS source TEXT NOT NULL DEFAULT 'llm_extractor'
    CHECK (source IN ('llm_extractor','cbo_mte_2002_seed','manual_admin','opus_arbitration')),
  ADD COLUMN IF NOT EXISTS taxonomy_content_version_id INT NULL
    REFERENCES taxonomy_versions(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS promoted_at TIMESTAMPTZ NULL,
  ADD COLUMN IF NOT EXISTS creation_confidence NUMERIC NULL
    CHECK (creation_confidence IS NULL OR (creation_confidence BETWEEN 0 AND 1)),
  ADD COLUMN IF NOT EXISTS vacancy_count_at_promotion INT NULL,
  ADD COLUMN IF NOT EXISTS distinct_sources_count_at_promotion INT NULL,
  ADD COLUMN IF NOT EXISTS confidence_median_at_promotion NUMERIC NULL
    CHECK (confidence_median_at_promotion IS NULL OR (confidence_median_at_promotion BETWEEN 0 AND 1)),
  ADD COLUMN IF NOT EXISTS retired_at TIMESTAMPTZ NULL,
  ADD COLUMN IF NOT EXISTS retire_reason TEXT NULL
    CHECK (retire_reason IS NULL OR retire_reason IN
      ('zero_vacancy_count','no_recent_postings_365d','manual_admin','opus_inactivate')),
  ADD COLUMN IF NOT EXISTS human_validated_at TIMESTAMPTZ NULL,
  ADD COLUMN IF NOT EXISTS human_validated_by UUID NULL,
  ADD COLUMN IF NOT EXISTS min_similarity_threshold NUMERIC NULL
    CHECK (min_similarity_threshold IS NULL OR (min_similarity_threshold BETWEEN 0 AND 1)),
  ADD COLUMN IF NOT EXISTS alias_of_id UUID NULL
    REFERENCES job_canonical_skills(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS rejected_reason TEXT NULL;

ALTER TABLE job_canonical_skills DROP CONSTRAINT IF EXISTS job_canonical_skills_status_check;
ALTER TABLE job_canonical_skills ADD CONSTRAINT job_canonical_skills_status_check
  CHECK (status IN ('active','pending','deprecated','rejected','alias_of','merge_candidate','retired'));
```

### 2.8 — `08_jcs_indexes.sql`

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_jcs_slug ON job_canonical_skills(slug) WHERE slug IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_jcs_embedding_hnsw ON job_canonical_skills
  USING hnsw (embedding vector_cosine_ops) WHERE embedding IS NOT NULL AND status IN ('active','pending');
CREATE INDEX IF NOT EXISTS idx_jcs_status_pending ON job_canonical_skills(status) WHERE status IN ('pending','merge_candidate');
CREATE INDEX IF NOT EXISTS idx_jcs_vacancy_count ON job_canonical_skills(vacancy_count DESC) WHERE status='active';
CREATE INDEX IF NOT EXISTS idx_jcs_latest_posted_active ON job_canonical_skills(latest_posted_at) WHERE status='active';
CREATE INDEX IF NOT EXISTS idx_jcs_source ON job_canonical_skills(source) WHERE source != 'llm_extractor';
CREATE INDEX IF NOT EXISTS idx_jcs_alias_of_id ON job_canonical_skills(alias_of_id) WHERE alias_of_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_jcs_promoted_at ON job_canonical_skills(promoted_at) WHERE promoted_at IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_jcs_retired_at ON job_canonical_skills(retired_at) WHERE retired_at IS NOT NULL;
```

### 2.9 — `09_jcs_slug_backfill_not_null.sql`

Precondition `unaccent` defensiva (Supabase requer habilitação de extensões via Dashboard; `CREATE EXTENSION` via SQL pode falhar com permissão negada dependendo da config do projeto). Após backfill e `NOT NULL`, converte índice UNIQUE parcial em UNIQUE total — necessário para `ON CONFLICT (slug)` em TS inferir o índice corretamente (Postgres pode não inferir UNIQUE parcial mesmo quando `WHERE` cobre todas as linhas).

```sql
-- Precondition: extensão unaccent habilitada
DO $$ BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'unaccent') THEN
    BEGIN
      CREATE EXTENSION unaccent;
    EXCEPTION WHEN insufficient_privilege THEN
      RAISE EXCEPTION 'Extensão unaccent não habilitada. Habilite via Supabase Dashboard → Database → Extensions antes de aplicar esta migration.';
    END;
  END IF;
END $$;

UPDATE job_canonical_skills
SET slug = lower(regexp_replace(unaccent(label), '[^a-zA-Z0-9]+', '-', 'g'))
WHERE slug IS NULL;

DO $$ DECLARE c INT; BEGIN
  SELECT COUNT(*) INTO c FROM (
    SELECT slug FROM job_canonical_skills WHERE slug IS NOT NULL
    GROUP BY slug HAVING COUNT(*) > 1
  ) t;
  IF c > 0 THEN RAISE EXCEPTION '% colisões de slug', c; END IF;
END $$;

ALTER TABLE job_canonical_skills ALTER COLUMN slug SET NOT NULL;

-- Converter UNIQUE parcial criado em mig 08 para UNIQUE total agora que slug é NOT NULL.
-- Necessário para ON CONFLICT (slug) em TS inferir o índice corretamente.
DROP INDEX IF EXISTS idx_jcs_slug;
ALTER TABLE job_canonical_skills
  ADD CONSTRAINT job_canonical_skills_slug_key UNIQUE (slug);
```

### 2.10 — `10_jcr_paridade_columns.sql`

```sql
ALTER TABLE job_canonical_roles
  ADD COLUMN IF NOT EXISTS creation_confidence NUMERIC NULL
    CHECK (creation_confidence IS NULL OR (creation_confidence BETWEEN 0 AND 1)),
  ADD COLUMN IF NOT EXISTS vacancy_count_at_promotion INT NULL,
  ADD COLUMN IF NOT EXISTS distinct_sources_count_at_promotion INT NULL,
  ADD COLUMN IF NOT EXISTS retired_at TIMESTAMPTZ NULL,
  ADD COLUMN IF NOT EXISTS retire_reason TEXT NULL
    CHECK (retire_reason IS NULL OR retire_reason IN
      ('zero_vacancy_count','no_recent_postings_365d','manual_admin','opus_inactivate'));

CREATE INDEX IF NOT EXISTS idx_jcr_retired_at ON job_canonical_roles(retired_at) WHERE retired_at IS NOT NULL;
```

### 2.11 — `11_jcs_add_last_opus_review_at.sql`

```sql
ALTER TABLE job_canonical_skills
  ADD COLUMN IF NOT EXISTS last_opus_review_at TIMESTAMPTZ NULL;

CREATE INDEX IF NOT EXISTS idx_jcs_last_opus_review_at
  ON job_canonical_skills(last_opus_review_at) WHERE last_opus_review_at IS NOT NULL;
```

## Bloco D — `taxonomy_relations`

### 2.12 — `12_taxonomy_relations_entity_type.sql`

Inclui:
- Drop da UNIQUE legada `taxonomy_relations_source_term_key` (single column) que bloqueia mesmo `source_term` para `entity_type='skill'` quando o termo já existe como `entity_type='role'` (ex: "python", "sql", "aws").
- Criação de UNIQUE composto `(source_term, entity_type)` permitindo coexistência paritária.
- Refator de `lookup_canonical_skill_by_normalized_alias` mantém o nome do parâmetro `p_normalized` (3 callers TS já invocam por nome — `canonical-skills.ts:193`, `canonical-skills.ts:288`, `submitted-job-skills.ts:39`).
- DROP COLUMN `target_canonical_id` deslocado para mig 12b, após reescrita das funções que o referenciam — caso contrário, `pg_get_functiondef` retornaria body com coluna inválida e EXECUTE falharia.

```sql
BEGIN;

-- Pré-validação de integridade: aborta antes de qualquer ALTER se houver órfãos.
-- Garante que backfill de target_role_id não criará violações de FK posteriores.
DO $$ DECLARE orphan_count INT; BEGIN
  SELECT COUNT(*) INTO orphan_count
  FROM taxonomy_relations tr
  LEFT JOIN job_canonical_roles jcr ON jcr.id = tr.target_canonical_id
  WHERE tr.target_canonical_id IS NOT NULL AND jcr.id IS NULL;
  IF orphan_count > 0 THEN
    RAISE EXCEPTION 'Pré-validação falhou: % rows em taxonomy_relations apontam para job_canonical_roles inexistente. Corrija antes de aplicar.', orphan_count;
  END IF;
END $$;

ALTER TABLE taxonomy_relations
  ADD COLUMN IF NOT EXISTS entity_type TEXT NOT NULL DEFAULT 'role';
ALTER TABLE taxonomy_relations DROP CONSTRAINT IF EXISTS taxonomy_relations_entity_type_check;
ALTER TABLE taxonomy_relations ADD CONSTRAINT taxonomy_relations_entity_type_check
  CHECK (entity_type IN ('role','skill'));

ALTER TABLE taxonomy_relations
  ADD COLUMN IF NOT EXISTS target_role_id UUID NULL,
  ADD COLUMN IF NOT EXISTS target_skill_id UUID NULL;

UPDATE taxonomy_relations SET target_role_id = target_canonical_id
  WHERE target_canonical_id IS NOT NULL AND target_role_id IS NULL;

ALTER TABLE taxonomy_relations
  ADD CONSTRAINT taxonomy_relations_target_role_fk
    FOREIGN KEY (target_role_id) REFERENCES job_canonical_roles(id) ON DELETE CASCADE;
ALTER TABLE taxonomy_relations
  ADD CONSTRAINT taxonomy_relations_target_skill_fk
    FOREIGN KEY (target_skill_id) REFERENCES job_canonical_skills(id) ON DELETE CASCADE;
ALTER TABLE taxonomy_relations
  ADD CONSTRAINT taxonomy_relations_target_xor_check
  CHECK (
    (target_role_id IS NOT NULL AND target_skill_id IS NULL AND entity_type='role') OR
    (target_role_id IS NULL AND target_skill_id IS NOT NULL AND entity_type='skill')
  );

-- Drop UNIQUE legada single-column e recriar composta com entity_type.
-- Sem isso, source_term='python' já existente como entity_type='role' bloqueia INSERT entity_type='skill'.
ALTER TABLE taxonomy_relations DROP CONSTRAINT IF EXISTS taxonomy_relations_source_term_key;
DROP INDEX IF EXISTS taxonomy_relations_source_term_key;
CREATE UNIQUE INDEX uq_tr_source_term_entity_type
  ON taxonomy_relations(source_term, entity_type);

CREATE INDEX IF NOT EXISTS idx_tr_entity_type ON taxonomy_relations(entity_type);
CREATE INDEX IF NOT EXISTS idx_tr_target_role ON taxonomy_relations(target_role_id) WHERE target_role_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_tr_target_skill ON taxonomy_relations(target_skill_id) WHERE target_skill_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_tr_target_skill_active
  ON taxonomy_relations(target_skill_id, entity_type, status)
  WHERE entity_type='skill' AND status='active';

DROP INDEX IF EXISTS idx_taxonomy_relations_pending;
CREATE INDEX idx_tr_pending ON taxonomy_relations(source_term, entity_type) WHERE status='inactive';

DROP INDEX IF EXISTS idx_taxonomy_relations_source_term_active;
CREATE INDEX idx_tr_source_term_active ON taxonomy_relations(source_term, entity_type) WHERE status='active';

-- Parâmetro mantém nome `p_normalized` (não `p_normalized_alias`) para compatibilidade com 3 callers TS por nome.
CREATE OR REPLACE FUNCTION lookup_canonical_skill_by_normalized_alias(p_normalized TEXT)
RETURNS UUID
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public, pg_temp
AS $$
  SELECT jcs.id
  FROM taxonomy_relations tr
  JOIN job_canonical_skills jcs ON tr.target_skill_id = jcs.id
  WHERE tr.entity_type = 'skill'
    AND tr.source_term = p_normalized
    AND tr.status = 'active'
    AND jcs.status IN ('active','pending')
    AND jcs.merged_into IS NULL
  LIMIT 1;
$$;

REVOKE ALL ON FUNCTION lookup_canonical_skill_by_normalized_alias(TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION lookup_canonical_skill_by_normalized_alias(TEXT) TO service_role, authenticated;
COMMIT;
```

### 2.12b — `12b_recreate_target_canonical_id_dependents_and_drop.sql`

Reescreve as funções que ainda referenciam `taxonomy_relations.target_canonical_id` substituindo pela coluna correta de cada lado, e só então faz `DROP COLUMN target_canonical_id CASCADE`. Estratégia em 4 passes:

**Pass 1a — replace mecânico em `process_opus_create_new`** (role-only): texto puro, `target_canonical_id → target_role_id`.

**Pass 1b — reescrita explícita de `process_opus_disagree`**: regex sobre `PERFORM merge_canonicals(...)` é frágil porque assume nomes de variáveis específicos no body original. Em vez disso, body inteiro é substituído via `CREATE OR REPLACE FUNCTION` baseado na assinatura conhecida `(p_relation_id uuid, p_loser_canonical_id uuid, p_existing_canonical_id uuid, p_suggested_label text, p_suggested_slug text, p_reason text)` confirmada por ground truth. A chamada interna a `merge_canonicals` usa 4 args explícitos com named-params, passando `SYSTEM_USER_ID` como `p_actor_id`. Antigravity verifica diff contra produção antes da execução — se o body real diverge na lógica de negócio (campos específicos), ajusta preservando os 4-args nomeados na chamada `merge_canonicals`.

**Pass 2 — bridge textual por função** (mapeamento entidade → coluna correta): `merge_canonicals` recebe `target_canonical_id → target_role_id`; `merge_canonical_skills` e `merge_skills` recebem `target_canonical_id → target_skill_id`. Bridge errado (genérico `target_role_id` para todas) faria UPDATE em coluna inexistente nas funções de skill — janela de 15+ etapas até mig 35/36 reescreverem por completo. Mig 35/36 sobrescrevem com `CREATE OR REPLACE` idempotente, com bodies completos + AE-6/7/9.

**Pass 3 — reescrita explícita de `o3_opus_canonical_label_disputes`**: mudança lógica (filtro `entity_type = 'role'`).

```sql
-- Pass 1a: process_opus_create_new — replace mecânico
DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_body FROM pg_proc
    WHERE proname = 'process_opus_create_new' LIMIT 1;
  IF v_body IS NULL THEN RAISE EXCEPTION 'process_opus_create_new ausente'; END IF;
  v_body := replace(v_body, 'target_canonical_id', 'target_role_id');
  v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
  EXECUTE v_body;
END $$;

-- Pass 1b: process_opus_disagree — reescrita explícita do body completo.
-- Body completo substituído para garantir named parameters na chamada merge_canonicals
-- (coordenado com mig 36 que muda o tipo do 4º arg de TEXT para UUID actor_id).
-- Substitui regex frágil por CREATE OR REPLACE com 4-args nomeados.
--
-- ATENÇÃO ANTIGRAVITY: o body abaixo é a base sintática mínima para a transição.
-- Antes de aplicar, validar contra produção:
--   1. Rodar SELECT prosrc FROM pg_proc WHERE proname='process_opus_disagree' no Supabase.
--   2. Comparar com o body abaixo: identificar lógica auxiliar (eventos extras, returns,
--      atualizações de tabelas auxiliares) que esteja em produção mas não aqui.
--   3. Se houver divergência relevante, MERGE: preservar lógica de produção + aplicar
--      apenas as mudanças necessárias (target_canonical_id → target_role_id na coluna
--      do UPDATE, e a chamada merge_canonicals em 4-args nomeados).
--   4. NÃO aplicar este body cegamente se Pass 1a (validação prévia) falhar.

-- Validação prévia: confirmar estrutura esperada do body atual antes de reescrever.
-- Se o body em produção não casar com o esperado, abortar — Antigravity revisa manualmente.
DO $$ DECLARE v_current TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_current FROM pg_proc WHERE proname='process_opus_disagree';
  IF v_current IS NULL THEN
    RAISE EXCEPTION 'process_opus_disagree não existe em produção — investigar antes de prosseguir';
  END IF;
  IF v_current NOT LIKE '%PERFORM merge_canonicals%' THEN
    RAISE EXCEPTION 'process_opus_disagree em produção não tem chamada PERFORM merge_canonicals — body divergente do esperado, merge manual necessário';
  END IF;
  IF v_current NOT LIKE '%target_canonical_id%' THEN
    RAISE EXCEPTION 'process_opus_disagree em produção não referencia target_canonical_id — função já reescrita ou estado inesperado';
  END IF;
  -- Estrutura mínima esperada confirmada. Prossiga com a reescrita abaixo.
END $$;

CREATE OR REPLACE FUNCTION public.process_opus_disagree(
  p_relation_id uuid,
  p_loser_canonical_id uuid,
  p_existing_canonical_id uuid,
  p_suggested_label text,
  p_suggested_slug text,
  p_reason text
) RETURNS void
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_winner_id uuid;
BEGIN
  -- v_winner_id é o canônico existente (não o loser): Opus discordou da label proposta
  -- e arbitrou que o canonical existente (p_existing_canonical_id) é o vencedor correto.
  v_winner_id := p_existing_canonical_id;

  -- Atualiza taxonomy_relations: a relação que apontava para o loser passa a apontar
  -- para o winner (que é o existing). Coluna correta após mig 12 é target_role_id.
  UPDATE taxonomy_relations
  SET target_role_id = v_winner_id,
      validated_at = NOW(),
      canonical_label_disputed = TRUE,
      opus_suggested_label = p_suggested_label
  WHERE id = p_relation_id;

  -- Merge dos dois canonicals (loser foi proposto pelo Opus mas conflita com existing).
  -- Chamada usa 4-args nomeados conforme nova assinatura de mig 36.
  PERFORM merge_canonicals(
    p_loser_id  := p_loser_canonical_id,
    p_winner_id := v_winner_id,
    p_actor     := 'opus_disagree',
    p_actor_id  := '00000000-0000-0000-0000-000000000001'::uuid
  );

  -- Audit em events.
  BEGIN
    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, reason, metadata)
    VALUES (
      'process_opus_disagree_executed', 'system', '00000000-0000-0000-0000-000000000001'::uuid,
      'taxonomy_relation', p_relation_id, p_reason,
      jsonb_build_object(
        'loser_id', p_loser_canonical_id,
        'winner_id', v_winner_id,
        'suggested_label', p_suggested_label,
        'suggested_slug', p_suggested_slug
      )
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[process_opus_disagree] events INSERT failed: %', SQLERRM;
  END;
END;
$$;

REVOKE ALL ON FUNCTION process_opus_disagree(uuid,uuid,uuid,text,text,text) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION process_opus_disagree(uuid,uuid,uuid,text,text,text) TO service_role;

-- Validação semântica: ausência de chamada posicional antiga + presença de 4-args nomeados.
DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def FROM pg_proc WHERE proname='process_opus_disagree';
  IF v_def NOT LIKE '%p_actor_id%' THEN
    RAISE EXCEPTION 'process_opus_disagree não tem p_actor_id na chamada interna';
  END IF;
  IF v_def NOT LIKE '%p_actor%' THEN
    RAISE EXCEPTION 'process_opus_disagree não tem p_actor na chamada interna';
  END IF;
  IF v_def LIKE '%target_canonical_id%' THEN
    RAISE EXCEPTION 'process_opus_disagree ainda referencia target_canonical_id';
  END IF;
END $$;

-- Pass 2: bridge textual por função, mapeando entidade → coluna correta.
-- merge_canonicals (role-side): target_canonical_id → target_role_id.
-- merge_canonical_skills e merge_skills (skill-side): target_canonical_id → target_skill_id.
-- Mig 35/36 sobrescrevem com bodies completos via CREATE OR REPLACE (idempotente).
DO $$
DECLARE
  v_body TEXT;
  v_fn TEXT;
  v_new_col TEXT;
  fn_map RECORD;
BEGIN
  FOR fn_map IN
    SELECT * FROM (VALUES
      ('merge_canonicals',        'target_role_id'),
      ('merge_canonical_skills',  'target_skill_id'),
      ('merge_skills',            'target_skill_id')
    ) AS t(fname, new_col)
  LOOP
    SELECT pg_get_functiondef(p.oid) INTO v_body
    FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
    WHERE n.nspname = 'public' AND p.proname = fn_map.fname LIMIT 1;
    IF v_body IS NULL THEN
      RAISE NOTICE 'Função % ausente — bridge skip', fn_map.fname;
      CONTINUE;
    END IF;
    IF v_body NOT LIKE '%target_canonical_id%' THEN
      CONTINUE;  -- já reescrita; nada a fazer
    END IF;
    v_body := replace(v_body, 'target_canonical_id', fn_map.new_col);
    v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
    EXECUTE v_body;
  END LOOP;
END $$;

-- Pass 3: o3_opus_canonical_label_disputes — reescrita explícita
-- (a função é role-only por design; precisa de filtro entity_type='role' agora que
-- taxonomy_relations passa a ter linhas tanto de role quanto de skill).
CREATE OR REPLACE FUNCTION public.o3_opus_canonical_label_disputes(p_cutoff TIMESTAMPTZ)
RETURNS TABLE(canonical_id UUID, canonical_label TEXT, opus_suggested_label TEXT, count_disputes BIGINT)
LANGUAGE sql STABLE SECURITY DEFINER SET search_path TO 'public', 'pg_temp' AS $$
  SELECT
    tr.target_role_id,
    jcr.label,
    tr.opus_suggested_label,
    COUNT(*)
  FROM taxonomy_relations tr
  JOIN job_canonical_roles jcr ON jcr.id = tr.target_role_id
  WHERE tr.entity_type = 'role'
    AND tr.canonical_label_disputed = TRUE
    AND tr.validated_at >= p_cutoff
  GROUP BY tr.target_role_id, jcr.label, tr.opus_suggested_label
  ORDER BY 4 DESC
  LIMIT 20;
$$;

-- Validação final: nenhuma função pública/internal pode ainda referenciar target_canonical_id.
DO $$ DECLARE bad_count INT; BEGIN
  SELECT COUNT(*) INTO bad_count FROM pg_proc p
  JOIN pg_namespace n ON n.oid = p.pronamespace
  WHERE n.nspname IN ('public','internal') AND p.prosrc LIKE '%target_canonical_id%';
  IF bad_count > 0 THEN
    RAISE EXCEPTION '% funções ainda referenciam target_canonical_id', bad_count;
  END IF;
END $$;

-- Só agora é seguro dropar a coluna.
ALTER TABLE taxonomy_relations DROP COLUMN target_canonical_id CASCADE;
```

### 2.13 — `13_taxonomy_relations_layer_expand.sql`

```sql
BEGIN;
ALTER TABLE taxonomy_relations DROP CONSTRAINT IF EXISTS taxonomy_relations_layer_check;
ALTER TABLE taxonomy_relations ADD CONSTRAINT taxonomy_relations_layer_check
  CHECK (layer IS NULL OR layer IN (0, 1, 2, 3));

ALTER TABLE taxonomy_relations DROP CONSTRAINT IF EXISTS taxonomy_relations_layer_skill_xor;
ALTER TABLE taxonomy_relations ADD CONSTRAINT taxonomy_relations_layer_skill_xor
  CHECK (layer IS NULL OR (entity_type='skill' AND layer != 2) OR entity_type='role');
COMMIT;
```

### 2.14 — `14_taxonomy_relations_layer_backfill.sql`

```sql
UPDATE taxonomy_relations SET layer = 1 WHERE layer IS NULL;

DO $$ DECLARE c INT; BEGIN
  SELECT COUNT(*) INTO c FROM taxonomy_relations WHERE layer IS NULL;
  IF c > 0 THEN RAISE EXCEPTION 'Backfill incompleto: %', c; END IF;
END $$;
```

### 2.15 — `15_taxonomy_relations_layer_not_null.sql`

```sql
ALTER TABLE taxonomy_relations ALTER COLUMN layer SET NOT NULL;

ALTER TABLE taxonomy_relations DROP CONSTRAINT IF EXISTS taxonomy_relations_layer_skill_xor;
ALTER TABLE taxonomy_relations ADD CONSTRAINT taxonomy_relations_layer_skill_xor
  CHECK (
    (entity_type='skill' AND layer IN (0, 1, 3)) OR
    (entity_type='role' AND layer IN (0, 1, 2, 3))
  );
```

## Bloco E — TFC defensivo

### 2.16 — `16_tfc_entity_type_defensive.sql`

D-PS-03 crava que skills NÃO entram em `taxonomy_families`. A coluna `canonical_skill_id` é adicionada como **preparação estrutural sem callers** (caso decisão arquitetural mude). Para enforçar D-PS-03 no schema, a constraint XOR é cravada na forma restritiva: `canonical_skill_id IS NULL AND entity_type='role'` — nenhuma combinação satisfaz `entity_type='skill'` ou `canonical_skill_id IS NOT NULL`. INSERT de skill viola constraint, falha imediatamente.

```sql
BEGIN;
ALTER TABLE taxonomy_family_canonicals
  ADD COLUMN IF NOT EXISTS entity_type TEXT NOT NULL DEFAULT 'role'
    CHECK (entity_type IN ('role','skill'));
ALTER TABLE taxonomy_family_canonicals
  ADD COLUMN IF NOT EXISTS canonical_skill_id UUID NULL
    REFERENCES job_canonical_skills(id) ON DELETE CASCADE;

ALTER TABLE taxonomy_family_canonicals ALTER COLUMN canonical_role_id DROP NOT NULL;

-- Constraint cravada na forma restritiva: bloqueia skills até D-PS-03 ser revogada.
-- Quando/se um sprint futuro habilitar Camada 2 para skills, basta dropar e recriar com a forma evolutiva.
ALTER TABLE taxonomy_family_canonicals DROP CONSTRAINT IF EXISTS tfc_target_xor_check;
ALTER TABLE taxonomy_family_canonicals ADD CONSTRAINT tfc_target_xor_check
  CHECK (
    canonical_role_id IS NOT NULL AND
    canonical_skill_id IS NULL AND
    entity_type = 'role'
  );

ALTER TABLE taxonomy_family_canonicals
  DROP CONSTRAINT IF EXISTS taxonomy_family_canonicals_family_id_canonical_role_id_key;
ALTER TABLE taxonomy_family_canonicals ADD CONSTRAINT tfc_family_role_unique UNIQUE (family_id, canonical_role_id);
COMMIT;

-- Validação: zero skill rows existentes ou inseríveis.
DO $$ DECLARE c INT; BEGIN
  SELECT COUNT(*) INTO c FROM taxonomy_family_canonicals
  WHERE entity_type='skill' OR canonical_skill_id IS NOT NULL;
  IF c > 0 THEN RAISE EXCEPTION 'taxonomy_family_canonicals tem % rows skill — D-PS-03 violada', c; END IF;
END $$;
```

## Bloco F — UNIQUEs e FKs

### 2.17 — `17_smd_unique_expression.sql`

Dedup preserva a row com `decided_at` mais recente (preserva metadata `reason` e `similarity` mais novos). Após dedup, drop do índice UNIQUE single-direction e criação do UNIQUE expression `LEAST/GREATEST` que trata `(A,B)` e `(B,A)` como o mesmo par. Inclui **reescrita do `ON CONFLICT (source_id, target_id)` em `merge_canonical_skills`** — sem isso, RPC quebra inteira após drop do índice antigo (fica órfã, Postgres não infere o expression index automaticamente).

```sql
-- Dedup preserva metadata mais recente (decided_at DESC):
DELETE FROM skill_merge_decisions s1
WHERE EXISTS (
  SELECT 1 FROM skill_merge_decisions s2
  WHERE s2.source_id = s1.target_id
    AND s2.target_id = s1.source_id
    AND (
      s2.decided_at > s1.decided_at OR
      (s2.decided_at = s1.decided_at AND s2.id < s1.id)
    )
);

DROP INDEX IF EXISTS idx_smd_source_target;
CREATE UNIQUE INDEX idx_smd_pair_unique ON skill_merge_decisions (
  LEAST(source_id, target_id), GREATEST(source_id, target_id)
);

-- Reescrita do body de merge_canonical_skills para usar LEAST/GREATEST no ON CONFLICT.
-- Mig 5b já fez o rename canonical_skills → job_canonical_skills no body. Agora corrige ON CONFLICT.
DO $$
DECLARE v_body TEXT;
BEGIN
  SELECT pg_get_functiondef(p.oid) INTO v_body
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname = 'public' AND p.proname = 'merge_canonical_skills'
  LIMIT 1;

  IF v_body IS NULL THEN
    RAISE EXCEPTION 'merge_canonical_skills não encontrada — mig 2.5b deveria ter mantido a função';
  END IF;

  -- Substitui ON CONFLICT (source_id, target_id) pela forma com LEAST/GREATEST que
  -- casa com idx_smd_pair_unique recém-criado.
  v_body := replace(
    v_body,
    'ON CONFLICT (source_id, target_id)',
    'ON CONFLICT (LEAST(source_id, target_id), GREATEST(source_id, target_id))'
  );

  v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
  EXECUTE v_body;
END $$;

-- Validação: função tem o ON CONFLICT correto.
DO $$
DECLARE v_def TEXT;
BEGIN
  SELECT pg_get_functiondef(p.oid) INTO v_def
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname='public' AND p.proname='merge_canonical_skills' LIMIT 1;

  IF v_def NOT LIKE '%LEAST(source_id, target_id), GREATEST(source_id, target_id)%' THEN
    RAISE EXCEPTION 'merge_canonical_skills sem ON CONFLICT LEAST/GREATEST após reescrita';
  END IF;
END $$;
```

### 2.18 — `18_canonical_skills_summary_fks.sql`

```sql
-- Pré-validação: aborta se houver órfãos antes de criar FKs.
DO $$ DECLARE orphan_skills INT; orphan_roles INT; BEGIN
  SELECT COUNT(*) INTO orphan_skills FROM canonical_skills_summary css
  LEFT JOIN job_canonical_skills jcs ON jcs.id = css.canonical_skill_id
  WHERE css.canonical_skill_id IS NOT NULL AND jcs.id IS NULL;
  SELECT COUNT(*) INTO orphan_roles FROM canonical_skills_summary css
  LEFT JOIN job_canonical_roles jcr ON jcr.id = css.canonical_role_id
  WHERE css.canonical_role_id IS NOT NULL AND jcr.id IS NULL;
  IF orphan_skills > 0 OR orphan_roles > 0 THEN
    RAISE EXCEPTION 'Pré-validação falhou: % órfãos em canonical_skill_id, % em canonical_role_id. Limpe antes de criar FKs.',
      orphan_skills, orphan_roles;
  END IF;
END $$;

ALTER TABLE canonical_skills_summary
  ADD CONSTRAINT canonical_skills_summary_canonical_skill_id_fk
  FOREIGN KEY (canonical_skill_id) REFERENCES job_canonical_skills(id) ON DELETE CASCADE;

ALTER TABLE canonical_skills_summary
  ADD CONSTRAINT canonical_skills_summary_canonical_role_id_fk
  FOREIGN KEY (canonical_role_id) REFERENCES job_canonical_roles(id) ON DELETE CASCADE;
```

## Bloco G — Unificação CBO + rename

### 2.19 — `19_rename_canonical_role_cbo.sql`

```sql
ALTER TABLE canonical_role_cbo_links DROP CONSTRAINT IF EXISTS canonical_role_cbo_links_occupation_code_fkey;

ALTER TABLE canonical_role_cbo RENAME TO canonical_cbo;

ALTER INDEX IF EXISTS idx_crc_family RENAME TO idx_canonical_cbo_family;
ALTER INDEX IF EXISTS idx_crc_cbo_version RENAME TO idx_canonical_cbo_version;
ALTER INDEX IF EXISTS idx_crc_embedding_hnsw RENAME TO idx_canonical_cbo_embedding_hnsw;
```

### 2.20 — `20_rename_canonical_role_cbo_links.sql`

```sql
ALTER TABLE canonical_role_cbo_links RENAME TO canonical_cbo_links;

ALTER TABLE canonical_cbo_links
  ADD CONSTRAINT canonical_cbo_links_occupation_code_fkey
  FOREIGN KEY (occupation_code) REFERENCES canonical_cbo(occupation_code) ON DELETE RESTRICT;

ALTER INDEX IF EXISTS idx_crcl_canonical RENAME TO idx_canonical_cbo_links_canonical_role;
ALTER INDEX IF EXISTS idx_crcl_occupation RENAME TO idx_canonical_cbo_links_occupation;
ALTER INDEX IF EXISTS idx_crcl_one_primary_per_canonical RENAME TO idx_canonical_cbo_links_one_primary_role;
```

### 2.21 — `21_canonical_cbo_links_xor.sql`

```sql
BEGIN;
ALTER TABLE canonical_cbo_links ALTER COLUMN canonical_role_id DROP NOT NULL;

ALTER TABLE canonical_cbo_links
  ADD COLUMN canonical_skill_id UUID NULL
  REFERENCES job_canonical_skills(id) ON DELETE CASCADE;

ALTER TABLE canonical_cbo_links ADD CONSTRAINT canonical_cbo_links_xor_check
  CHECK ((canonical_role_id IS NOT NULL) != (canonical_skill_id IS NOT NULL));

ALTER TABLE canonical_cbo_links
  DROP CONSTRAINT IF EXISTS canonical_role_cbo_links_canonical_role_id_occupation_code_key;
CREATE UNIQUE INDEX uq_canonical_cbo_links_role_occupation
  ON canonical_cbo_links (canonical_role_id, occupation_code) WHERE canonical_role_id IS NOT NULL;
CREATE UNIQUE INDEX uq_canonical_cbo_links_skill_occupation
  ON canonical_cbo_links (canonical_skill_id, occupation_code) WHERE canonical_skill_id IS NOT NULL;

DROP INDEX IF EXISTS idx_canonical_cbo_links_one_primary_role;
CREATE UNIQUE INDEX idx_canonical_cbo_links_one_primary_role
  ON canonical_cbo_links (canonical_role_id) WHERE is_primary=true AND canonical_role_id IS NOT NULL;
CREATE UNIQUE INDEX idx_canonical_cbo_links_one_primary_skill
  ON canonical_cbo_links (canonical_skill_id) WHERE is_primary=true AND canonical_skill_id IS NOT NULL;

CREATE INDEX idx_canonical_cbo_links_skill_id
  ON canonical_cbo_links(canonical_skill_id) WHERE canonical_skill_id IS NOT NULL;
COMMIT;
```

### 2.22 — `22_recreate_cbo_pl_functions.sql`

Recria `process_opus_create_new`, `replace_cbo_link`, `upsert_primary_cbo_link` substituindo nomes de tabela renomeadas. Lê corpo via `pg_get_functiondef`, faz substituição textual, recria com `CREATE OR REPLACE`.

```sql
DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_body FROM pg_proc
  WHERE proname = 'process_opus_create_new' LIMIT 1;
  IF v_body IS NULL THEN RAISE EXCEPTION 'process_opus_create_new ausente'; END IF;
  v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
  v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
  EXECUTE v_body;
END $$;

DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_body FROM pg_proc
  WHERE proname = 'replace_cbo_link' LIMIT 1;
  IF v_body IS NULL THEN RAISE EXCEPTION 'replace_cbo_link ausente'; END IF;
  v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
  v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
  EXECUTE v_body;
END $$;

DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_body FROM pg_proc
  WHERE proname = 'upsert_primary_cbo_link' LIMIT 1;
  IF v_body IS NULL THEN RAISE EXCEPTION 'upsert_primary_cbo_link ausente'; END IF;
  v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
  v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
  EXECUTE v_body;
END $$;

DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(p.oid) INTO v_body
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE p.proname = 'reset_taxonomy_core' AND n.nspname = 'internal' LIMIT 1;
  IF v_body IS NOT NULL THEN
    v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
    v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
    EXECUTE v_body;
  END IF;
END $$;

DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(p.oid) INTO v_body
  FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE p.proname = 'reset_taxonomy_core_v2' AND n.nspname = 'internal' LIMIT 1;
  IF v_body IS NOT NULL THEN
    v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
    v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
    EXECUTE v_body;
  END IF;
END $$;

-- fetch_cbo_candidates: confirmada por V19 como dependente de canonical_role_cbo (FROM clause).
-- Não estava na lista original; adicionada após mapeamento ground truth.
DO $$ DECLARE v_body TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_body FROM pg_proc
  WHERE proname = 'fetch_cbo_candidates' LIMIT 1;
  IF v_body IS NULL THEN RAISE EXCEPTION 'fetch_cbo_candidates ausente'; END IF;
  v_body := replace(v_body, 'canonical_role_cbo_links', 'canonical_cbo_links');
  v_body := replace(v_body, 'canonical_role_cbo', 'canonical_cbo');
  v_body := regexp_replace(v_body, '^CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION');
  EXECUTE v_body;
END $$;

-- Validação final: nenhuma das 6 funções tem ref a canonical_role_cbo.
DO $$
DECLARE bad_count INT;
BEGIN
  SELECT COUNT(*) INTO bad_count FROM pg_proc p
  JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE p.proname IN (
      'process_opus_create_new','replace_cbo_link','upsert_primary_cbo_link',
      'reset_taxonomy_core','reset_taxonomy_core_v2','fetch_cbo_candidates'
    )
    AND (p.prosrc LIKE '%canonical_role_cbo%' OR p.prosrc LIKE '%target_canonical_id%');

  IF bad_count > 0 THEN
    RAISE EXCEPTION '% funções CBO ainda referenciam canonical_role_cbo* ou target_canonical_id após mig 22', bad_count;
  END IF;
END $$;
```

## Bloco H — Triggers e RPCs JCS

### 2.23 — `23_retire_canonical_extended.sql`

Adiciona overload de 3 args `retire_canonical(uuid, text, text)` para skills, preservando a assinatura existente de 2 args para roles via overloading. Ambos os callers JCR existentes (`fn_retire_canonical_on_zero_vacancy` e `detectStaleCanonicals`) continuam usando 2 args.

```sql
CREATE OR REPLACE FUNCTION retire_canonical(
  p_id UUID, p_reason TEXT, p_entity_type TEXT
) RETURNS VOID
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
    v_previous_status TEXT;
    v_source TEXT;
    v_previous_vacancy_count INT;
    v_previous_distinct_sources INT;
    v_rows_affected INT;
BEGIN
    IF p_entity_type NOT IN ('role','skill') THEN
        RAISE EXCEPTION 'p_entity_type inválido: %. Aceito: role | skill', p_entity_type;
    END IF;

    IF p_entity_type = 'role' THEN
        PERFORM retire_canonical(p_id, p_reason);
        RETURN;
    END IF;

    IF p_reason NOT IN ('zero_vacancy_count','no_recent_postings_365d','manual_admin','opus_inactivate') THEN
        RAISE EXCEPTION 'p_reason inválido para skill: %', p_reason;
    END IF;

    SELECT status, source, vacancy_count, distinct_sources_count
      INTO v_previous_status, v_source, v_previous_vacancy_count, v_previous_distinct_sources
      FROM job_canonical_skills
     WHERE id = p_id
     FOR UPDATE;

    IF v_previous_status IS NULL THEN
        RAISE EXCEPTION 'Skill % não encontrada', p_id;
    END IF;

    IF v_source = 'cbo_mte_2002_seed' AND p_reason = 'zero_vacancy_count' THEN
        RETURN;
    END IF;

    IF v_previous_status = 'retired' THEN
        RETURN;
    END IF;

    UPDATE job_canonical_skills
       SET status = 'retired',
           retired_at = NOW(),
           retire_reason = p_reason,
           vacancy_count = 0,
           distinct_sources_count = 0
     WHERE id = p_id
       AND (source IS DISTINCT FROM 'cbo_mte_2002_seed'
            OR p_reason IN ('no_recent_postings_365d', 'manual_admin', 'opus_inactivate'))
       AND status != 'retired';

    GET DIAGNOSTICS v_rows_affected = ROW_COUNT;

    IF v_rows_affected = 0 THEN
        RETURN;
    END IF;

    BEGIN
        INSERT INTO events (
            event_name, resource_type, resource_id,
            actor, actor_id, previous_state, new_state, reason
        )
        VALUES (
            'canonical_retired',
            'canonical_skill',
            p_id,
            'system',
            '00000000-0000-0000-0000-000000000001'::uuid,
            jsonb_build_object(
                'previous_status', v_previous_status,
                'previous_vacancy_count', v_previous_vacancy_count,
                'previous_distinct_sources_count', v_previous_distinct_sources
            ),
            jsonb_build_object(
                'new_status', 'retired',
                'vacancy_count', 0,
                'distinct_sources_count', 0
            ),
            p_reason
        );
    EXCEPTION
        WHEN foreign_key_violation THEN
            RAISE WARNING '[retire_canonical/skill] events FK violation: %', SQLERRM;
        WHEN OTHERS THEN
            RAISE WARNING '[retire_canonical/skill] events INSERT failed (SQLSTATE %): %', SQLSTATE, SQLERRM;
    END;
END;
$$;

REVOKE ALL ON FUNCTION retire_canonical(UUID, TEXT, TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION retire_canonical(UUID, TEXT, TEXT) TO service_role;
```

### 2.24 — `24_fn_promote_skill_on_threshold.sql`

```sql
CREATE OR REPLACE FUNCTION fn_promote_skill_on_threshold()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp
AS $$
DECLARE
  v_min_vacancies INT;
  v_min_employers INT;
  v_lookback_days INT;
  v_min_confidence NUMERIC;
  v_should_promote BOOLEAN;
  v_has_recent_postings BOOLEAN;
  v_is_resurrection BOOLEAN;
BEGIN
  IF pg_trigger_depth() > 1 THEN RETURN NULL; END IF;

  IF NEW.status NOT IN ('pending','retired') OR OLD.status NOT IN ('pending','retired') THEN
    RETURN NULL;
  END IF;

  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_vacancies'), 5) INTO v_min_vacancies;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_distinct_employers'), 2) INTO v_min_employers;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.lookback_days'), 30) INTO v_lookback_days;
  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85) INTO v_min_confidence;

  v_should_promote := (
    NEW.vacancy_count >= v_min_vacancies AND
    NEW.distinct_sources_count >= v_min_employers AND
    NEW.confidence_median IS NOT NULL AND
    NEW.confidence_median >= v_min_confidence
  );

  IF NOT v_should_promote THEN RETURN NULL; END IF;

  v_is_resurrection := (OLD.status = 'retired');

  SELECT EXISTS (
    SELECT 1
    FROM job_posting_skills jps
    JOIN job_postings jp ON jps.job_posting_id = jp.id
    WHERE jps.canonical_skill_id = NEW.id
      AND jp.posted_at >= NOW() - (v_lookback_days || ' days')::INTERVAL
      AND jp.curation_status = 'curated'
  ) INTO v_has_recent_postings;

  IF v_has_recent_postings THEN
    UPDATE job_canonical_skills
    SET status = 'active',
        promoted_at = COALESCE(promoted_at, NOW()),
        vacancy_count_at_promotion = NEW.vacancy_count,
        distinct_sources_count_at_promotion = NEW.distinct_sources_count,
        confidence_median_at_promotion = COALESCE(confidence_median_at_promotion, NEW.confidence_median),
        retired_at = NULL,
        retire_reason = NULL,
        needs_opus_review = CASE
          WHEN v_is_resurrection THEN TRUE
          ELSE needs_opus_review
        END
    WHERE id = NEW.id
      AND status IN ('pending', 'retired');

    BEGIN
      INSERT INTO events (
        event_name, actor, actor_id, resource_type, resource_id,
        previous_state, new_state, metadata
      ) VALUES (
        'skill_promoted_dynamic',
        'system',
        '00000000-0000-0000-0000-000000000001'::uuid,
        'canonical_skill',
        NEW.id,
        jsonb_build_object('status', OLD.status, 'vacancy_count', OLD.vacancy_count, 'distinct_sources_count', OLD.distinct_sources_count),
        jsonb_build_object('status', 'active', 'vacancy_count', NEW.vacancy_count, 'distinct_sources_count', NEW.distinct_sources_count),
        jsonb_build_object(
          'promotion_rule', 'cumulative_gate_with_recency',
          'thresholds', jsonb_build_object(
            'vacancy_count', v_min_vacancies,
            'distinct_sources_count', v_min_employers,
            'min_confidence', v_min_confidence,
            'recency_window_days', v_lookback_days
          ),
          'has_recent_postings', v_has_recent_postings,
          'is_resurrection', v_is_resurrection
        )
      );
    EXCEPTION
      WHEN foreign_key_violation THEN
        RAISE WARNING '[fn_promote_skill] events FK violation: %', SQLERRM;
      WHEN unique_violation THEN
        RAISE WARNING '[fn_promote_skill] events unique violation: %', SQLERRM;
      WHEN OTHERS THEN
        RAISE WARNING '[fn_promote_skill] events promoted_dynamic falhou (SQLSTATE %): %', SQLSTATE, SQLERRM;
    END;
  END IF;
  -- Sem branch arqueológico (D-PS-39, D-PS-54). Trigger ROW-level só dispara em UPDATE de
  -- vacancy_count/distinct_sources_count/confidence_median, e esses só sobem por INSERT
  -- em job_postings/job_posting_skills — sempre recente por definição. Caso edge raro de
  -- INSERT com posted_at antigo (migração de dados) cai em Painel 5 (Pending stuck) para
  -- ação admin manual.

  RETURN NULL;
END;
$$;

DROP TRIGGER IF EXISTS trg_promote_skill_on_threshold ON job_canonical_skills;
CREATE TRIGGER trg_promote_skill_on_threshold
  AFTER UPDATE OF vacancy_count, distinct_sources_count, confidence_median ON job_canonical_skills
  FOR EACH ROW EXECUTE FUNCTION fn_promote_skill_on_threshold();
```

### 2.25 — `25_fn_retire_skill_on_zero_vacancy.sql`

```sql
CREATE OR REPLACE FUNCTION fn_retire_skill_on_zero_vacancy()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp
AS $$
BEGIN
  IF pg_trigger_depth() > 1 THEN RETURN NEW; END IF;
  IF NEW.vacancy_count != 0 OR NEW.status NOT IN ('active', 'pending', 'rejected') THEN
    RETURN NEW;
  END IF;
  IF NEW.source = 'cbo_mte_2002_seed' THEN
    RETURN NEW;
  END IF;
  PERFORM retire_canonical(NEW.id, 'zero_vacancy_count', 'skill');
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS trg_retire_skill_on_zero_vacancy ON job_canonical_skills;
CREATE TRIGGER trg_retire_skill_on_zero_vacancy
  AFTER UPDATE OF vacancy_count ON job_canonical_skills
  FOR EACH ROW WHEN (NEW.vacancy_count = 0 AND OLD.vacancy_count > 0)
  EXECUTE FUNCTION fn_retire_skill_on_zero_vacancy();
```

### 2.26 — `26_fn_reset_skill_embedding_and_updated_at.sql`

```sql
CREATE OR REPLACE FUNCTION fn_reset_skill_embedding_on_semantic_change()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF NEW.label IS DISTINCT FROM OLD.label THEN
    NEW.embedding := NULL;
    NEW.updated_at := NOW();
  END IF;
  RETURN NEW;
END; $$;
DROP TRIGGER IF EXISTS trg_reset_skill_embedding ON job_canonical_skills;
CREATE TRIGGER trg_reset_skill_embedding BEFORE UPDATE OF label ON job_canonical_skills
  FOR EACH ROW EXECUTE FUNCTION fn_reset_skill_embedding_on_semantic_change();

CREATE OR REPLACE FUNCTION fn_jcs_set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  -- Monitora apenas colunas semânticas (paridade com JCR `trg_jcr_set_updated_at`).
  -- vacancy_count, distinct_sources_count, confidence_median, latest_posted_at NÃO contam — esses são
  -- atualizados em alta cardência por triggers cascade e bumpá-los corromperia o ORDER BY de
  -- detectStaleCanonicals (anti-starvation). Por isso o filtro IS DISTINCT FROM em colunas de domínio.
  IF (NEW.label IS DISTINCT FROM OLD.label
      OR NEW.slug IS DISTINCT FROM OLD.slug
      OR NEW.status IS DISTINCT FROM OLD.status
      OR NEW.skill_type IS DISTINCT FROM OLD.skill_type
      OR NEW.merged_into IS DISTINCT FROM OLD.merged_into
      OR NEW.needs_opus_review IS DISTINCT FROM OLD.needs_opus_review
      OR NEW.human_validated_at IS DISTINCT FROM OLD.human_validated_at) THEN
    NEW.updated_at := NOW();
  END IF;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_jcs_set_updated_at ON job_canonical_skills;
DROP TRIGGER IF EXISTS z_trg_jcs_set_updated_at ON job_canonical_skills;
-- Prefixo z_ garante execução por último entre triggers BEFORE UPDATE do mesmo evento/tabela.
-- Não ordena em relação a triggers AFTER UPDATE (esses sempre rodam depois de todos BEFORE).
-- Crítico para evitar race com cascade de outros triggers que alteram colunas observadas.
CREATE TRIGGER z_trg_jcs_set_updated_at BEFORE UPDATE ON job_canonical_skills
  FOR EACH ROW EXECUTE FUNCTION fn_jcs_set_updated_at();
```

### 2.27 — `27_jps_count_triggers.sql`

```sql
CREATE OR REPLACE FUNCTION fn_jps_recompute_jcs(p_canonical_skill_id UUID)
RETURNS VOID LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_new_median NUMERIC;
  v_new_median_set BOOLEAN := FALSE;
  v_lookback_days INT;
  v_min_count INT;
BEGIN
  IF p_canonical_skill_id IS NULL THEN RETURN; END IF;

  -- D-PS-45: parâmetros de runtime confidence lidos de pipeline_config (paridade com
  -- fn_recompute_jcr_confidence_median na mig 29). Hardcodes 120d / 3 violariam D-PS-45.
  v_lookback_days := COALESCE(
    (SELECT value::INT FROM pipeline_config WHERE key='skill.confidence.lookback_days'), 120);
  v_min_count := COALESCE(
    (SELECT value::INT FROM pipeline_config WHERE key='skill.confidence.min_count'), 3);

  -- Mediana sobre vagas curadas (paridade semântica com distinct_sources_count e latest_posted_at).
  -- Sem filtro curated, mediana seria influenciada por skills de vagas ainda em curadoria — vies
  -- temporal de cada batch entrando no agregado antes da validação.
  WITH median_calc AS (
    SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY jps.skill_confidence) AS m
    FROM job_posting_skills jps
    JOIN job_postings jp ON jps.job_posting_id = jp.id
    WHERE jps.canonical_skill_id = p_canonical_skill_id
      AND jps.skill_confidence IS NOT NULL
      AND jp.curation_status = 'curated'
      AND jp.posted_at >= NOW() - make_interval(days => v_lookback_days)
    HAVING COUNT(*) >= v_min_count
  )
  SELECT m, TRUE INTO v_new_median, v_new_median_set FROM median_calc;

  IF v_new_median_set THEN
    UPDATE job_canonical_skills jcs SET
      vacancy_count = (
        -- Vagas curadas (paridade semântica com demais agregados).
        SELECT COUNT(*)
        FROM job_posting_skills jps
        JOIN job_postings jp ON jps.job_posting_id = jp.id
        WHERE jps.canonical_skill_id = jcs.id
          AND jp.curation_status = 'curated'
      ),
      distinct_sources_count = (
        SELECT COUNT(DISTINCT jp.employer_id)
        FROM job_posting_skills jps
        JOIN job_postings jp ON jps.job_posting_id = jp.id
        LEFT JOIN employers e ON e.id = jp.employer_id
        WHERE jps.canonical_skill_id = jcs.id
          AND jp.curation_status = 'curated'
          AND COALESCE(e.is_recruitment_agency, FALSE) = FALSE
      ),
      latest_posted_at = (
        SELECT MAX(jp.posted_at)
        FROM job_posting_skills jps JOIN job_postings jp ON jps.job_posting_id = jp.id
        WHERE jps.canonical_skill_id = jcs.id AND jp.curation_status = 'curated'
      ),
      confidence_median = v_new_median
    WHERE jcs.id = p_canonical_skill_id;
  ELSE
    -- Sem dados suficientes para nova mediana: preserva valor anterior, atualiza apenas
    -- contadores/timestamps. Mediana legada continua válida até atingir min_count denovo.
    UPDATE job_canonical_skills jcs SET
      vacancy_count = (
        SELECT COUNT(*)
        FROM job_posting_skills jps
        JOIN job_postings jp ON jps.job_posting_id = jp.id
        WHERE jps.canonical_skill_id = jcs.id
          AND jp.curation_status = 'curated'
      ),
      distinct_sources_count = (
        SELECT COUNT(DISTINCT jp.employer_id)
        FROM job_posting_skills jps
        JOIN job_postings jp ON jps.job_posting_id = jp.id
        LEFT JOIN employers e ON e.id = jp.employer_id
        WHERE jps.canonical_skill_id = jcs.id
          AND jp.curation_status = 'curated'
          AND COALESCE(e.is_recruitment_agency, FALSE) = FALSE
      ),
      latest_posted_at = (
        SELECT MAX(jp.posted_at)
        FROM job_posting_skills jps JOIN job_postings jp ON jps.job_posting_id = jp.id
        WHERE jps.canonical_skill_id = jcs.id AND jp.curation_status = 'curated'
      )
    WHERE jcs.id = p_canonical_skill_id;
  END IF;
END; $$;

CREATE OR REPLACE FUNCTION fn_jps_insert_update_jcs_counts()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  PERFORM fn_jps_recompute_jcs(NEW.canonical_skill_id);
  RETURN NEW;
END; $$;
DROP TRIGGER IF EXISTS trg_jps_insert_jcs_counts ON job_posting_skills;
CREATE TRIGGER trg_jps_insert_jcs_counts AFTER INSERT ON job_posting_skills
  FOR EACH ROW EXECUTE FUNCTION fn_jps_insert_update_jcs_counts();

CREATE OR REPLACE FUNCTION fn_jps_update_jcs_counts()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF OLD.canonical_skill_id IS NOT DISTINCT FROM NEW.canonical_skill_id
     AND OLD.skill_confidence IS NOT DISTINCT FROM NEW.skill_confidence THEN
    RETURN NEW;
  END IF;
  IF OLD.canonical_skill_id IS DISTINCT FROM NEW.canonical_skill_id THEN
    PERFORM fn_jps_recompute_jcs(OLD.canonical_skill_id);
    PERFORM fn_jps_recompute_jcs(NEW.canonical_skill_id);
  ELSE
    PERFORM fn_jps_recompute_jcs(NEW.canonical_skill_id);
  END IF;
  RETURN NEW;
END; $$;
DROP TRIGGER IF EXISTS trg_jps_update_jcs_counts ON job_posting_skills;
CREATE TRIGGER trg_jps_update_jcs_counts
  AFTER UPDATE OF canonical_skill_id, skill_confidence ON job_posting_skills
  FOR EACH ROW EXECUTE FUNCTION fn_jps_update_jcs_counts();

CREATE OR REPLACE FUNCTION fn_jps_delete_jcs_counts()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  PERFORM fn_jps_recompute_jcs(OLD.canonical_skill_id);
  RETURN OLD;
END; $$;
DROP TRIGGER IF EXISTS trg_jps_delete_jcs_counts ON job_posting_skills;
CREATE TRIGGER trg_jps_delete_jcs_counts AFTER DELETE ON job_posting_skills
  FOR EACH ROW EXECUTE FUNCTION fn_jps_delete_jcs_counts();

-- ─────────────────────────────────────────────────────────────────
-- Triggers de recompute em dependências externas a job_posting_skills.
-- Sem isso, agregados de JCS ficam stale quando atributos da vaga mudam após o INSERT em JPS.
-- Casos cobertos:
--   1) job_postings.curation_status muda (vaga aprovada/rejeitada após ingestão)
--   2) job_postings.posted_at muda (correção de timestamp pelo admin)
--   3) job_postings.employer_id muda (admin reatribui vaga a empregador correto)
--   4) employers.is_recruitment_agency muda (CRON refresh_employer_recruitment_agency_flags)
--
-- Padrão: ROW-level acumula canonical_skill_ids em pg_temp + STATEMENT-level drena unique skills.
-- Sem esse padrão, batch UPDATE em 1000 vagas que compartilham skills comuns ('SQL', 'Python')
-- causaria recompute O(N×M) — uma chamada PERCENTILE_CONT por vaga × skill, derretendo CPU.
-- Com o padrão, recompute é O(N) — uma chamada por skill afetada distinta.
-- ─────────────────────────────────────────────────────────────────

CREATE OR REPLACE FUNCTION fn_jps_accumulate_on_job_postings_change()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  CREATE TEMP TABLE IF NOT EXISTS pending_jcs_recompute (skill_id UUID PRIMARY KEY) ON COMMIT DROP;
  INSERT INTO pending_jcs_recompute (skill_id)
  SELECT DISTINCT canonical_skill_id FROM job_posting_skills WHERE job_posting_id = NEW.id
  ON CONFLICT (skill_id) DO NOTHING;
  RETURN NEW;
END; $$;

CREATE OR REPLACE FUNCTION fn_jps_drain_pending_jcs_recompute()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE skill_id UUID;
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_class WHERE relname = 'pending_jcs_recompute' AND relpersistence = 't'
  ) THEN
    RETURN NULL;
  END IF;
  FOR skill_id IN SELECT pjr.skill_id FROM pending_jcs_recompute pjr LOOP
    PERFORM fn_jps_recompute_jcs(skill_id);
  END LOOP;
  TRUNCATE pending_jcs_recompute;
  RETURN NULL;
END; $$;

DROP TRIGGER IF EXISTS trg_jp_recompute_jcs ON job_postings;
DROP TRIGGER IF EXISTS trg_jp_accumulate_jcs ON job_postings;
DROP TRIGGER IF EXISTS trg_jp_drain_jcs ON job_postings;

CREATE TRIGGER trg_jp_accumulate_jcs
  AFTER UPDATE OF curation_status, posted_at, employer_id ON job_postings
  FOR EACH ROW
  WHEN (
    OLD.curation_status IS DISTINCT FROM NEW.curation_status OR
    OLD.posted_at IS DISTINCT FROM NEW.posted_at OR
    OLD.employer_id IS DISTINCT FROM NEW.employer_id
  )
  EXECUTE FUNCTION fn_jps_accumulate_on_job_postings_change();

CREATE TRIGGER trg_jp_drain_jcs
  AFTER UPDATE ON job_postings
  FOR EACH STATEMENT EXECUTE FUNCTION fn_jps_drain_pending_jcs_recompute();

CREATE OR REPLACE FUNCTION fn_jps_accumulate_on_employer_flag_change()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF OLD.is_recruitment_agency IS DISTINCT FROM NEW.is_recruitment_agency THEN
    CREATE TEMP TABLE IF NOT EXISTS pending_jcs_recompute (skill_id UUID PRIMARY KEY) ON COMMIT DROP;
    INSERT INTO pending_jcs_recompute (skill_id)
    SELECT DISTINCT jps.canonical_skill_id
    FROM job_posting_skills jps
    JOIN job_postings jp ON jps.job_posting_id = jp.id
    WHERE jp.employer_id = NEW.id
    ON CONFLICT (skill_id) DO NOTHING;
  END IF;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_employer_recruitment_flag_recompute_jcs ON employers;
DROP TRIGGER IF EXISTS trg_employer_accumulate_jcs ON employers;
DROP TRIGGER IF EXISTS trg_employer_drain_jcs ON employers;

CREATE TRIGGER trg_employer_accumulate_jcs
  AFTER UPDATE OF is_recruitment_agency ON employers
  FOR EACH ROW
  WHEN (OLD.is_recruitment_agency IS DISTINCT FROM NEW.is_recruitment_agency)
  EXECUTE FUNCTION fn_jps_accumulate_on_employer_flag_change();

CREATE TRIGGER trg_employer_drain_jcs
  AFTER UPDATE ON employers
  FOR EACH STATEMENT EXECUTE FUNCTION fn_jps_drain_pending_jcs_recompute();
```

### 2.27b — `27b_jp_count_triggers_paridade_jcr.sql`

Refactor paritário do recompute em cadeia para o lado JCR. A função `fn_update_canonical_vacancy_count` (existente em produção) é ROW-level com `COUNT(*)` por chamada — em batch UPDATE em massa (>500 vagas) sofre do mesmo padrão O(N×M). Substituir por padrão STATEMENT-level com `pending_jcr_recompute` simétrico.

```sql
CREATE OR REPLACE FUNCTION fn_jp_accumulate_jcr_recompute()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  CREATE TEMP TABLE IF NOT EXISTS pending_jcr_recompute (role_id UUID PRIMARY KEY) ON COMMIT DROP;
  IF TG_OP = 'INSERT' AND NEW.canonical_role_id IS NOT NULL THEN
    INSERT INTO pending_jcr_recompute VALUES (NEW.canonical_role_id) ON CONFLICT DO NOTHING;
  ELSIF TG_OP = 'UPDATE' THEN
    IF OLD.canonical_role_id IS NOT NULL AND OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id THEN
      INSERT INTO pending_jcr_recompute VALUES (OLD.canonical_role_id) ON CONFLICT DO NOTHING;
    END IF;
    IF NEW.canonical_role_id IS NOT NULL THEN
      INSERT INTO pending_jcr_recompute VALUES (NEW.canonical_role_id) ON CONFLICT DO NOTHING;
    END IF;
  ELSIF TG_OP = 'DELETE' AND OLD.canonical_role_id IS NOT NULL THEN
    INSERT INTO pending_jcr_recompute VALUES (OLD.canonical_role_id) ON CONFLICT DO NOTHING;
  END IF;
  RETURN COALESCE(NEW, OLD);
END; $$;

CREATE OR REPLACE FUNCTION fn_jp_drain_pending_jcr_recompute()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE role_id UUID;
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_class WHERE relname = 'pending_jcr_recompute' AND relpersistence = 't'
  ) THEN
    RETURN NULL;
  END IF;
  FOR role_id IN SELECT pjr.role_id FROM pending_jcr_recompute pjr LOOP
    UPDATE job_canonical_roles jcr SET vacancy_count = (
      SELECT COUNT(*) FROM job_postings WHERE canonical_role_id = jcr.id AND is_active = TRUE
    )
    WHERE jcr.id = role_id;
  END LOOP;
  TRUNCATE pending_jcr_recompute;
  RETURN NULL;
END; $$;

-- Drop trigger ROW-level pré-existente (se houver) e cria par accumulate + drain.
-- A função fn_update_canonical_vacancy_count permanece em pg_proc por compatibilidade,
-- mas não é mais chamada por trigger ROW-level — pode ser dropada em sprint posterior.
DO $$ DECLARE trg_name TEXT; BEGIN
  FOR trg_name IN
    SELECT tgname FROM pg_trigger
    WHERE tgrelid = 'job_postings'::regclass
      AND tgfoid = 'fn_update_canonical_vacancy_count'::regproc
      AND NOT tgisinternal
  LOOP
    EXECUTE format('DROP TRIGGER %I ON job_postings', trg_name);
  END LOOP;
END $$;

DROP TRIGGER IF EXISTS trg_jp_accumulate_jcr ON job_postings;
DROP TRIGGER IF EXISTS trg_jp_drain_jcr ON job_postings;

-- Trigger ROW separado por TG_OP. WHEN refinado nos triggers UPDATE evita acumulação em
-- updates que tocam canonical_role_id ou is_active mas resultam em valores idênticos
-- (no-op semântico). Sem WHEN, qualquer UPDATE OF dispara mesmo quando OLD = NEW,
-- inflando pending_jcr_recompute desnecessariamente em batches grandes.
CREATE TRIGGER trg_jp_accumulate_jcr_insert
  AFTER INSERT ON job_postings
  FOR EACH ROW
  WHEN (NEW.canonical_role_id IS NOT NULL)
  EXECUTE FUNCTION fn_jp_accumulate_jcr_recompute();

CREATE TRIGGER trg_jp_accumulate_jcr_update
  AFTER UPDATE OF canonical_role_id, is_active ON job_postings
  FOR EACH ROW
  WHEN (
    OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id OR
    OLD.is_active IS DISTINCT FROM NEW.is_active
  )
  EXECUTE FUNCTION fn_jp_accumulate_jcr_recompute();

CREATE TRIGGER trg_jp_accumulate_jcr_delete
  AFTER DELETE ON job_postings
  FOR EACH ROW
  WHEN (OLD.canonical_role_id IS NOT NULL)
  EXECUTE FUNCTION fn_jp_accumulate_jcr_recompute();

CREATE TRIGGER trg_jp_drain_jcr
  AFTER INSERT OR UPDATE OR DELETE ON job_postings
  FOR EACH STATEMENT EXECUTE FUNCTION fn_jp_drain_pending_jcr_recompute();
```

### 2.28 — `28_validate_fn_promote_canonical_on_threshold.sql`

NO-OP de validação. Confirma que `fn_promote_canonical_on_threshold` em produção tem ressuscitação `OLD.status IN ('pending', 'retired')`, branch arqueológico, EXCEPTION blocks e `pg_trigger_depth` guard intencional. Falha duro se produção foi adulterada.

```sql
DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def
  FROM pg_proc WHERE proname='fn_promote_canonical_on_threshold' LIMIT 1;

  IF v_def IS NULL THEN
    RAISE EXCEPTION 'fn_promote_canonical_on_threshold ausente';
  END IF;

  IF v_def NOT LIKE E'%OLD.status IN (\'pending\', \'retired\')%' THEN
    RAISE EXCEPTION 'OLD.status IN (pending, retired) não encontrado — ressuscitação JCR ausente';
  END IF;
  IF v_def NOT LIKE E'%NEW.status IN (\'pending\', \'retired\')%' THEN
    RAISE EXCEPTION 'NEW.status IN (pending, retired) não encontrado';
  END IF;

  IF v_def NOT LIKE '%v_is_resurrection%' THEN
    RAISE EXCEPTION 'v_is_resurrection ausente';
  END IF;

  IF v_def NOT LIKE '%WHEN v_is_resurrection THEN TRUE%' THEN
    RAISE EXCEPTION 'needs_opus_review TRUE em ressuscitação ausente';
  END IF;

  IF v_def NOT LIKE '%COALESCE(promoted_at, NOW())%' THEN
    RAISE EXCEPTION 'COALESCE(promoted_at, NOW()) ausente';
  END IF;

  IF v_def NOT LIKE '%pg_trigger_depth() > 1%' THEN
    RAISE EXCEPTION 'pg_trigger_depth() > 1 guard ausente';
  END IF;

  IF v_def NOT LIKE '%INSERT INTO events%' THEN
    RAISE EXCEPTION 'INSERT INTO events ausente';
  END IF;

  IF v_def NOT LIKE '%auto_assign_family_to_canonical%' THEN
    RAISE EXCEPTION 'auto_assign_family_to_canonical ausente';
  END IF;

  IF v_def NOT LIKE '%canonical_promotion_deferred_archaeological%' THEN
    RAISE EXCEPTION 'branch arqueológico ausente';
  END IF;

  IF v_def NOT LIKE '%foreign_key_violation%' THEN
    RAISE EXCEPTION 'EXCEPTION handlers ausentes';
  END IF;

  RAISE NOTICE 'fn_promote_canonical_on_threshold validada';
END $$;
```

### 2.29 — `29_fn_update_jcr_confidence_median.sql`

```sql
CREATE OR REPLACE FUNCTION fn_recompute_jcr_confidence_median(p_canonical_role_id UUID)
RETURNS VOID LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_lookback_days INT;
  v_min_count INT;
BEGIN
  IF p_canonical_role_id IS NULL THEN RETURN; END IF;

  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.confidence.lookback_days'), 120)
    INTO v_lookback_days;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.confidence.min_count'), 5)
    INTO v_min_count;

  EXECUTE format($q$
    WITH new_median AS (
      SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY foi.confidence) AS m
      FROM function_orchestrator_items foi
      JOIN function_orchestrator_runs fo ON fo.id = foi.run_id
      WHERE foi.canonical_role_id = %L
        AND foi.confidence IS NOT NULL
        AND fo.started_at >= NOW() - INTERVAL '%s days'
      HAVING COUNT(*) >= %s
    )
    UPDATE job_canonical_roles jcr
    SET confidence_median = nm.m
    FROM new_median nm
    WHERE jcr.id = %L
      AND jcr.status IN ('active', 'pending')
  $q$, p_canonical_role_id, v_lookback_days, v_min_count, p_canonical_role_id);
END; $$;

CREATE OR REPLACE FUNCTION fn_jcr_confidence_median_insert()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF NEW.canonical_role_id IS NOT NULL AND NEW.confidence IS NOT NULL THEN
    PERFORM fn_recompute_jcr_confidence_median(NEW.canonical_role_id);
  END IF;
  RETURN NEW;
END; $$;

CREATE OR REPLACE FUNCTION fn_jcr_confidence_median_update()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF OLD.canonical_role_id IS NOT DISTINCT FROM NEW.canonical_role_id
     AND OLD.confidence IS NOT DISTINCT FROM NEW.confidence THEN
    RETURN NEW;
  END IF;
  IF OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id THEN
    PERFORM fn_recompute_jcr_confidence_median(OLD.canonical_role_id);
    PERFORM fn_recompute_jcr_confidence_median(NEW.canonical_role_id);
  ELSE
    PERFORM fn_recompute_jcr_confidence_median(NEW.canonical_role_id);
  END IF;
  RETURN NEW;
END; $$;

CREATE OR REPLACE FUNCTION fn_jcr_confidence_median_delete()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
BEGIN
  IF OLD.canonical_role_id IS NOT NULL THEN
    PERFORM fn_recompute_jcr_confidence_median(OLD.canonical_role_id);
  END IF;
  RETURN OLD;
END; $$;

DROP TRIGGER IF EXISTS trg_foi_jcr_confidence_insert ON function_orchestrator_items;
CREATE TRIGGER trg_foi_jcr_confidence_insert
  AFTER INSERT ON function_orchestrator_items
  FOR EACH ROW EXECUTE FUNCTION fn_jcr_confidence_median_insert();

DROP TRIGGER IF EXISTS trg_foi_jcr_confidence_update ON function_orchestrator_items;
CREATE TRIGGER trg_foi_jcr_confidence_update
  AFTER UPDATE OF canonical_role_id, confidence ON function_orchestrator_items
  FOR EACH ROW EXECUTE FUNCTION fn_jcr_confidence_median_update();

DROP TRIGGER IF EXISTS trg_foi_jcr_confidence_delete ON function_orchestrator_items;
CREATE TRIGGER trg_foi_jcr_confidence_delete
  AFTER DELETE ON function_orchestrator_items
  FOR EACH ROW EXECUTE FUNCTION fn_jcr_confidence_median_delete();
```

### 2.30 — `30_fn_flag_needs_opus_review_residual.sql`

```sql
-- BEFORE UPDATE com mutação em memória (NEW := value) — não AFTER + UPDATE same row.
-- AFTER + UPDATE forçaria gravação física duplicada (WAL duplo) e abriria risco de recursão
-- caso outro trigger AFTER UPDATE seja adicionado no futuro sem filtragem estrita.
-- Como o BEFORE roda dentro da mesma operação que está alterando confidence_median,
-- a mutação de needs_opus_review consolida na mesma tupla — uma única gravação.
-- INSERT em events permanece após o BEFORE: usa AFTER UPDATE separado guardado por
-- needs_opus_review = TRUE para evitar race com BEFORE.

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcr()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_upper_threshold NUMERIC;
  v_cooldown_days INT;
BEGIN
  IF NEW.status != 'active' THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NULL THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review = TRUE THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NOT DISTINCT FROM OLD.confidence_median THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='role.promotion.auto_min_confidence'), 0.85)
    INTO v_upper_threshold;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.opus_review.cooldown_days'), 90)
    INTO v_cooldown_days;

  IF NEW.confidence_median < v_upper_threshold
     AND (NEW.last_opus_review_at IS NULL OR NEW.last_opus_review_at < NOW() - (v_cooldown_days || ' days')::INTERVAL) THEN
    NEW.needs_opus_review := TRUE;
  END IF;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcr ON job_canonical_roles;
CREATE TRIGGER trg_flag_needs_opus_review_jcr
  BEFORE UPDATE OF confidence_median ON job_canonical_roles
  FOR EACH ROW EXECUTE FUNCTION fn_flag_needs_opus_review_jcr();

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcr_emit_event()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_upper_threshold NUMERIC;
BEGIN
  -- Emite evento somente quando a flag muda de FALSE/NULL para TRUE.
  -- Trigger separado (AFTER UPDATE) consome a flag já consolidada pelo BEFORE.
  IF OLD.needs_opus_review IS NOT DISTINCT FROM NEW.needs_opus_review THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review IS NOT TRUE THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='role.promotion.auto_min_confidence'), 0.85)
    INTO v_upper_threshold;

  BEGIN
    INSERT INTO events (
      event_name, actor, actor_id, resource_type, resource_id, metadata
    ) VALUES (
      'role_flagged_low_confidence_residual',
      'system',
      '00000000-0000-0000-0000-000000000001'::uuid,
      'job_canonical_role',
      NEW.id,
      jsonb_build_object(
        'confidence_median', NEW.confidence_median,
        'threshold', v_upper_threshold,
        'last_opus_review_at', NEW.last_opus_review_at,
        'reason', 'confidence_median caiu abaixo do upper threshold pós-promoção'
      )
    );
  EXCEPTION
    WHEN OTHERS THEN
      RAISE WARNING '[fn_flag_jcr_emit] events INSERT failed (SQLSTATE %): %', SQLSTATE, SQLERRM;
  END;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcr_emit ON job_canonical_roles;
CREATE TRIGGER trg_flag_needs_opus_review_jcr_emit
  AFTER UPDATE OF needs_opus_review ON job_canonical_roles
  FOR EACH ROW EXECUTE FUNCTION fn_flag_needs_opus_review_jcr_emit_event();

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcs()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_upper_threshold NUMERIC;
  v_cooldown_days INT;
BEGIN
  IF NEW.status != 'active' THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NULL THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review = TRUE THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NOT DISTINCT FROM OLD.confidence_median THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85)
    INTO v_upper_threshold;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.opus_review.cooldown_days'), 90)
    INTO v_cooldown_days;

  IF NEW.confidence_median < v_upper_threshold
     AND (NEW.last_opus_review_at IS NULL OR NEW.last_opus_review_at < NOW() - (v_cooldown_days || ' days')::INTERVAL) THEN
    NEW.needs_opus_review := TRUE;
  END IF;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcs ON job_canonical_skills;
CREATE TRIGGER trg_flag_needs_opus_review_jcs
  BEFORE UPDATE OF confidence_median ON job_canonical_skills
  FOR EACH ROW EXECUTE FUNCTION fn_flag_needs_opus_review_jcs();

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcs_emit_event()
RETURNS TRIGGER LANGUAGE plpgsql SET search_path = public, pg_temp AS $$
DECLARE
  v_upper_threshold NUMERIC;
BEGIN
  IF OLD.needs_opus_review IS NOT DISTINCT FROM NEW.needs_opus_review THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review IS NOT TRUE THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85)
    INTO v_upper_threshold;

  BEGIN
    INSERT INTO events (
      event_name, actor, actor_id, resource_type, resource_id, metadata
    ) VALUES (
      'skill_flagged_low_confidence_residual',
      'system',
      '00000000-0000-0000-0000-000000000001'::uuid,
      'canonical_skill',
      NEW.id,
      jsonb_build_object(
        'confidence_median', NEW.confidence_median,
        'threshold', v_upper_threshold,
        'last_opus_review_at', NEW.last_opus_review_at,
        'reason', 'confidence_median caiu abaixo do upper threshold pós-promoção'
      )
    );
  EXCEPTION
    WHEN OTHERS THEN
      RAISE WARNING '[fn_flag_jcs_emit] events INSERT failed (SQLSTATE %): %', SQLSTATE, SQLERRM;
  END;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcs_emit ON job_canonical_skills;
CREATE TRIGGER trg_flag_needs_opus_review_jcs_emit
  AFTER UPDATE OF needs_opus_review ON job_canonical_skills
  FOR EACH ROW EXECUTE FUNCTION fn_flag_needs_opus_review_jcs_emit_event();
```

### 2.31 — `31_catchup_pending_promotions.sql`

Catchup rodando em CRON pipeline-maintenance. Promove diretamente canônicos pendentes/retired que atingiram thresholds — **sem dummy +1/-1 em `vacancy_count`**. UPDATE direto de `status='active'` com WHERE defensivo (`AND status IN ('pending','retired')`) que aborta silenciosamente se outro processo (merge, admin) mudou o status entre o SELECT e o UPDATE. Lê `role.promotion.lookback_days` de `pipeline_config` (D-PS-41).

Por que UPDATE direto e não trigger:
- Eliminar anti-pattern MVCC (+1/-1 produz dois dead tuples por canônico verificado, força autovacuum)
- Eliminar race em READ COMMITTED entre `+1` e `-1` (insert real concorrente seria obliterado)
- Trigger `fn_promote_canonical_on_threshold` continua sendo o caminho único em ingestão real (depth=0); catchup é o caminho complementar para canônicos que atingiram o gate enquanto o trigger estava abafado.

Sem branch arqueológico (D-PS-39, D-PS-54). Apenas branch normal: `vacancy_count >= min_vacancies AND distinct_employers >= min_distinct_employers AND latest_posted_at >= NOW() - INTERVAL <lookback_days>`.

```sql
CREATE OR REPLACE FUNCTION catchup_pending_promotions()
RETURNS jsonb
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
    r RECORD;
    promoted_count INT := 0;
    archaeological_count INT := 0;  -- preservado por compatibilidade com caller TS; sempre zero
    resurrection_count INT := 0;
    unchanged_count INT := 0;
    v_min_vacancies INT;
    v_min_employers INT;
    v_lookback_days INT;
    v_has_recent_postings BOOLEAN;
    v_rows_affected INT;
BEGIN
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.promotion.min_vacancies'), 3)
      INTO v_min_vacancies;
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.promotion.min_distinct_employers'), 2)
      INTO v_min_employers;
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.promotion.lookback_days'), 60)
      INTO v_lookback_days;

    FOR r IN
        SELECT jcr.id, jcr.label, jcr.vacancy_count, jcr.distinct_sources_count,
               jcr.confidence_median, jcr.status AS status_before
        FROM job_canonical_roles jcr
        WHERE jcr.status IN ('pending', 'retired')
          AND jcr.vacancy_count >= v_min_vacancies
          AND jcr.distinct_sources_count >= v_min_employers
    LOOP
        EXECUTE format($q$
          SELECT EXISTS (
            SELECT 1 FROM job_postings jp
            WHERE jp.canonical_role_id = %L
              AND jp.posted_at >= NOW() - INTERVAL '%s days'
              AND jp.curation_status = 'curated'
          )
        $q$, r.id, v_lookback_days)
        INTO v_has_recent_postings;

        IF NOT v_has_recent_postings THEN
            unchanged_count := unchanged_count + 1;
            CONTINUE;
        END IF;

        -- WHERE defensivo: se merge/admin/Opus mudou status entre o SELECT e este UPDATE,
        -- ROW_COUNT = 0 e seguimos sem ressuscitar deprecated/merge_candidate/rejected.
        UPDATE job_canonical_roles SET
          status = 'active',
          promoted_at = COALESCE(promoted_at, NOW()),
          vacancy_count_at_promotion = r.vacancy_count,
          distinct_sources_count_at_promotion = r.distinct_sources_count,
          confidence_median_at_promotion = r.confidence_median,
          retired_at = NULL,
          retire_reason = NULL,
          needs_opus_review = (CASE WHEN r.status_before = 'retired' THEN TRUE ELSE needs_opus_review END)
        WHERE id = r.id
          AND status IN ('pending', 'retired');

        GET DIAGNOSTICS v_rows_affected = ROW_COUNT;
        IF v_rows_affected = 0 THEN
            unchanged_count := unchanged_count + 1;
            CONTINUE;
        END IF;

        promoted_count := promoted_count + 1;
        IF r.status_before = 'retired' THEN
            resurrection_count := resurrection_count + 1;
        END IF;

        PERFORM auto_assign_family_to_canonical(r.id, TRUE);

        BEGIN
          INSERT INTO events (
            event_name, actor, actor_id, resource_type, resource_id, metadata
          ) VALUES (
            'role_promoted_dynamic',
            'system',
            '00000000-0000-0000-0000-000000000001'::uuid,
            'job_canonical_role',
            r.id,
            jsonb_build_object(
              'is_resurrection', r.status_before = 'retired',
              'vacancy_count', r.vacancy_count,
              'distinct_sources_count', r.distinct_sources_count,
              'confidence_median', r.confidence_median,
              'origin', 'catchup_pending_promotions'
            )
          );
        EXCEPTION
          WHEN OTHERS THEN
            RAISE WARNING '[catchup_pending_promotions] events INSERT (promote) failed: %', SQLERRM;
        END;
    END LOOP;

    BEGIN
        INSERT INTO events (
            event_name, actor, actor_id, resource_type, resource_id, metadata
        ) VALUES (
            'catchup_pending_promotions_executed',
            'system',
            '00000000-0000-0000-0000-000000000001'::uuid,
            'cron_job',
            '00000000-0000-0000-0000-000000000002'::uuid,
            jsonb_build_object(
                'promoted', promoted_count,
                'resurrections', resurrection_count,
                'unchanged', unchanged_count,
                'archaeological', archaeological_count,
                'executed_at', NOW(),
                'scope', 'pending_and_retired',
                'thresholds', jsonb_build_object(
                  'min_vacancies', v_min_vacancies,
                  'min_employers', v_min_employers,
                  'lookback_days', v_lookback_days
                )
            )
        );
    EXCEPTION
        WHEN OTHERS THEN
            RAISE WARNING '[catchup_pending_promotions] events INSERT (summary) failed: %', SQLERRM;
    END;

    -- Retorno preserva 'promoted', 'archaeological', 'unchanged' (compat caller TS)
    -- + 'resurrected' aditivo (capturável por painéis).
    RETURN jsonb_build_object(
        'promoted', promoted_count,
        'archaeological', archaeological_count,
        'unchanged', unchanged_count,
        'resurrected', resurrection_count
    );
END;
$$;

REVOKE ALL ON FUNCTION catchup_pending_promotions() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION catchup_pending_promotions() TO service_role;
```

### 2.32 — `32_catchup_pending_skill_promotions.sql`

Catchup paritário a `catchup_pending_promotions` mas para skills. Mesma estrutura: UPDATE direto sem +1/-1, ler thresholds de `pipeline_config`, gate de confidence (skills aplicam, JCR não — D-PS-19), WHERE defensivo. Sem branch arqueológico (D-PS-39, D-PS-54). Skills NÃO chamam `auto_assign_family_to_canonical` (D-PS-03).

```sql
CREATE OR REPLACE FUNCTION catchup_pending_skill_promotions()
RETURNS jsonb
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
    r RECORD;
    promoted_count INT := 0;
    archaeological_count INT := 0;  -- preservado por compat caller TS; sempre zero
    resurrection_count INT := 0;
    unchanged_count INT := 0;
    v_min_vacancies INT;
    v_min_employers INT;
    v_min_confidence NUMERIC;
    v_lookback_days INT;
    v_has_recent_postings BOOLEAN;
    v_rows_affected INT;
BEGIN
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_vacancies'), 5)
      INTO v_min_vacancies;
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_distinct_employers'), 2)
      INTO v_min_employers;
    SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85)
      INTO v_min_confidence;
    SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.lookback_days'), 60)
      INTO v_lookback_days;

    FOR r IN
        SELECT jcs.id, jcs.label, jcs.vacancy_count, jcs.distinct_sources_count,
               jcs.confidence_median, jcs.status AS status_before
        FROM job_canonical_skills jcs
        WHERE jcs.status IN ('pending', 'retired')
          AND jcs.vacancy_count >= v_min_vacancies
          AND jcs.distinct_sources_count >= v_min_employers
          AND jcs.confidence_median IS NOT NULL
          AND jcs.confidence_median >= v_min_confidence
    LOOP
        SELECT EXISTS (
          SELECT 1 FROM job_posting_skills jps
          JOIN job_postings jp ON jps.job_posting_id = jp.id
          WHERE jps.canonical_skill_id = r.id
            AND jp.posted_at >= NOW() - (v_lookback_days || ' days')::INTERVAL
            AND jp.curation_status = 'curated'
        ) INTO v_has_recent_postings;

        IF NOT v_has_recent_postings THEN
            unchanged_count := unchanged_count + 1;
            CONTINUE;
        END IF;

        UPDATE job_canonical_skills SET
          status = 'active',
          promoted_at = COALESCE(promoted_at, NOW()),
          vacancy_count_at_promotion = r.vacancy_count,
          distinct_sources_count_at_promotion = r.distinct_sources_count,
          confidence_median_at_promotion = r.confidence_median,
          retired_at = NULL,
          retire_reason = NULL,
          needs_opus_review = (CASE WHEN r.status_before = 'retired' THEN TRUE ELSE needs_opus_review END)
        WHERE id = r.id
          AND status IN ('pending', 'retired');

        GET DIAGNOSTICS v_rows_affected = ROW_COUNT;
        IF v_rows_affected = 0 THEN
            unchanged_count := unchanged_count + 1;
            CONTINUE;
        END IF;

        promoted_count := promoted_count + 1;
        IF r.status_before = 'retired' THEN
            resurrection_count := resurrection_count + 1;
        END IF;

        BEGIN
          INSERT INTO events (
            event_name, actor, actor_id, resource_type, resource_id, metadata
          ) VALUES (
            'skill_promoted_dynamic',
            'system',
            '00000000-0000-0000-0000-000000000001'::uuid,
            'canonical_skill',
            r.id,
            jsonb_build_object(
              'is_resurrection', r.status_before = 'retired',
              'vacancy_count', r.vacancy_count,
              'distinct_sources_count', r.distinct_sources_count,
              'confidence_median', r.confidence_median,
              'origin', 'catchup_pending_skill_promotions'
            )
          );
        EXCEPTION
          WHEN OTHERS THEN
            RAISE WARNING '[catchup_pending_skill_promotions] events INSERT (promote) failed: %', SQLERRM;
        END;
    END LOOP;

    BEGIN
        INSERT INTO events (
            event_name, actor, actor_id, resource_type, resource_id, metadata
        ) VALUES (
            'catchup_pending_skill_promotions_executed',
            'system',
            '00000000-0000-0000-0000-000000000001'::uuid,
            'cron_job',
            '00000000-0000-0000-0000-000000000002'::uuid,
            jsonb_build_object(
                'promoted', promoted_count,
                'resurrections', resurrection_count,
                'unchanged', unchanged_count,
                'archaeological', archaeological_count,
                'executed_at', NOW(),
                'thresholds', jsonb_build_object(
                  'min_vacancies', v_min_vacancies,
                  'min_employers', v_min_employers,
                  'min_confidence', v_min_confidence,
                  'lookback_days', v_lookback_days
                )
            )
        );
    EXCEPTION
        WHEN OTHERS THEN
            RAISE WARNING '[catchup_pending_skill_promotions] events INSERT (summary) failed: %', SQLERRM;
    END;

    RETURN jsonb_build_object(
        'promoted', promoted_count,
        'archaeological', archaeological_count,
        'unchanged', unchanged_count,
        'resurrected', resurrection_count
    );
END;
$$;

REVOKE ALL ON FUNCTION catchup_pending_skill_promotions() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION catchup_pending_skill_promotions() TO service_role;
```

### 2.33 — `33_drop_merge_canonical_skill_singular.sql`

Função `merge_canonical_skill` (singular) faz hard-delete de `canonical_skills` no final, anti-pattern do soft-delete universal. `merge_canonical_skills` (plural, admin UI) e `merge_skills` (pipeline) cobrem todos os casos com soft-delete.

```sql
DROP FUNCTION IF EXISTS merge_canonical_skill(UUID, UUID, UUID);
```

### 2.34 — `34_canonical_skill_merge_candidates.sql`

```sql
CREATE TABLE IF NOT EXISTS canonical_skill_merge_candidates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  canonical_a_id UUID NOT NULL REFERENCES job_canonical_skills(id) ON DELETE CASCADE,
  canonical_b_id UUID NOT NULL REFERENCES job_canonical_skills(id) ON DELETE CASCADE,
  similarity NUMERIC NOT NULL,
  detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  resolved_at TIMESTAMPTZ NULL,
  opus_decision TEXT NULL CHECK (opus_decision IS NULL OR opus_decision IN ('MERGE','KEEP_BOTH','NEEDS_HUMAN')),
  opus_reasoning TEXT NULL,
  arbitration_attempts INT NOT NULL DEFAULT 0,
  last_arbitration_attempt_at TIMESTAMPTZ NULL,
  CONSTRAINT canonical_skill_merge_candidates_check CHECK (canonical_a_id < canonical_b_id),
  UNIQUE (canonical_a_id, canonical_b_id)
);

CREATE INDEX idx_csmc_canonical_a ON canonical_skill_merge_candidates(canonical_a_id);
CREATE INDEX idx_csmc_canonical_b ON canonical_skill_merge_candidates(canonical_b_id);
CREATE INDEX idx_csmc_unresolved ON canonical_skill_merge_candidates(detected_at) WHERE resolved_at IS NULL;
```

### 2.35 — `35_rewrite_merge_canonical_skills_and_merge_skills.sql`

Reescrita explícita das duas funções de merge no lado skill.

**Preservação de ordem dos parâmetros (D-PS-60).** Ambas mantêm a ordem `(p_loser_id, p_winner_id, ...)` da assinatura de produção, eliminando reorder dos callers. Mudanças efetivas: rename do 4º arg + ajuste de tipo de `TEXT 'system'` para `UUID actor_id`, e adição do guard de loser status.

`merge_canonical_skills` (5 args, admin UI) precisa migrar `taxonomy_relations.target_canonical_id → target_skill_id` e referenciar `job_canonical_skills` (renomeada por mig 05). `merge_skills` (4 args, pipeline) precisa de duas mudanças semânticas: AE-7 (guard de loser status antes de redirect) e AE-6 (remoção das linhas que UPDATEavam `analysis_skill_matches.canonical_skill_id` e `.matched_via_similar_skill_id` — paridade com princípio de snapshot puro).

**Sem mesclagem temporal.** Versão produção de `merge_canonicals` usa `first_seen_at`/`last_seen_at` em `job_canonical_role_sources` (tabela diferente, não em JCR). Tentativa de aplicar LEAST/GREATEST em colunas de `job_canonical_skills` ou `job_canonical_roles` falha em runtime — essas colunas não existem nas tabelas canônicas. Reescrita explícita mantém apenas `updated_at = NOW()` no winner, paritário com `merge_skills` (4 args, pipeline).

```sql
-- merge_canonical_skills (admin UI, 5 args) — preserva ordem (loser, winner)
CREATE OR REPLACE FUNCTION merge_canonical_skills(
  p_loser_id UUID,
  p_winner_id UUID,
  p_decided_by_actor_id UUID,
  p_reason TEXT,
  p_cross_type_confirmed BOOLEAN DEFAULT FALSE
) RETURNS jsonb
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_winner job_canonical_skills%ROWTYPE;
  v_loser job_canonical_skills%ROWTYPE;
BEGIN
  IF p_winner_id = p_loser_id THEN
    RAISE EXCEPTION 'winner_id e loser_id não podem ser iguais';
  END IF;

  -- Locks determinísticos em ordem de UUID para evitar deadlock cross-merge concorrente.
  IF p_winner_id < p_loser_id THEN
    SELECT * INTO v_winner FROM job_canonical_skills WHERE id = p_winner_id FOR UPDATE;
    SELECT * INTO v_loser  FROM job_canonical_skills WHERE id = p_loser_id  FOR UPDATE;
  ELSE
    SELECT * INTO v_loser  FROM job_canonical_skills WHERE id = p_loser_id  FOR UPDATE;
    SELECT * INTO v_winner FROM job_canonical_skills WHERE id = p_winner_id FOR UPDATE;
  END IF;

  IF v_winner.id IS NULL THEN RAISE EXCEPTION 'winner % não encontrado', p_winner_id; END IF;
  IF v_loser.id  IS NULL THEN RAISE EXCEPTION 'loser  % não encontrado', p_loser_id;  END IF;

  -- AE-7: guard de loser status. Previne sobrescrita de merged_into em cadeias.
  IF v_loser.status NOT IN ('active','pending') THEN
    RAISE EXCEPTION 'loser % está em status % (não active/pending) — merge bloqueado',
      v_loser.id, v_loser.status;
  END IF;
  IF v_winner.status NOT IN ('active','pending') THEN
    RAISE EXCEPTION 'winner % está em status % (não active/pending)',
      v_winner.id, v_winner.status;
  END IF;

  -- Cross-type confirmation: skills de tipos diferentes exigem flag explícita.
  IF v_winner.skill_type IS DISTINCT FROM v_loser.skill_type AND p_cross_type_confirmed = FALSE THEN
    RETURN jsonb_build_object(
      'requires_cross_type_confirmation', TRUE,
      'source_type', v_loser.skill_type,
      'target_type', v_winner.skill_type
    );
  END IF;

  -- Redirect operacional com NOT EXISTS guards (paridade com merge_skills pipeline).
  -- Sem isso, UPDATE direto pode disparar UNIQUE violation quando winner E loser têm
  -- linhas para o mesmo (job_posting_id, canonical_skill_id) ou tuplas equivalentes.
  UPDATE job_posting_skills SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM job_posting_skills jps2
        WHERE jps2.job_posting_id = job_posting_skills.job_posting_id
          AND jps2.canonical_skill_id = p_winner_id
      );
  DELETE FROM job_posting_skills WHERE canonical_skill_id = p_loser_id;

  UPDATE submitted_job_skills SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM submitted_job_skills sjs2
        WHERE sjs2.submitted_job_id = submitted_job_skills.submitted_job_id
          AND sjs2.canonical_skill_id = p_winner_id
      );
  DELETE FROM submitted_job_skills WHERE canonical_skill_id = p_loser_id;

  UPDATE canonical_skills_summary SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_skills_summary css2
        WHERE css2.canonical_role_id = canonical_skills_summary.canonical_role_id
          AND css2.canonical_skill_id = p_winner_id
      );
  DELETE FROM canonical_skills_summary WHERE canonical_skill_id = p_loser_id;

  -- D-PS-32: resume_skill_enrichments.canonical_skill_id é "Híbrido por coluna" — atualiza em
  -- merge_skills E em merge_canonical_skills. Sem isso, o canonical de loser fica em
  -- resume_skill_enrichments e callers TS de validate/route.ts e resolve/route.ts retornam
  -- label do loser (deprecated) em vez do winner. Anexo B documenta o comportamento esperado.
  UPDATE resume_skill_enrichments SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM resume_skill_enrichments rse2
        WHERE rse2.analysis_id = resume_skill_enrichments.analysis_id
          AND rse2.canonical_skill_id = p_winner_id
      );
  DELETE FROM resume_skill_enrichments WHERE canonical_skill_id = p_loser_id;

  -- Anexo B: canonical_cbo_links.canonical_skill_id é "Operacional" — atualiza em
  -- merge_skills E merge_canonical_skills com NOT EXISTS guard. Sem isso, links CBO
  -- ficam apontando para canonical deprecated pós-merge — paridade com canonical_role_id
  -- que mig 36 (merge_canonicals) trata em FASE 3.
  UPDATE canonical_cbo_links SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_cbo_links ccl2
        WHERE ccl2.occupation_code = canonical_cbo_links.occupation_code
          AND ccl2.canonical_skill_id = p_winner_id
      );
  DELETE FROM canonical_cbo_links WHERE canonical_skill_id = p_loser_id;

  -- taxonomy_relations: redirect target_skill_id (não mais target_canonical_id).
  UPDATE taxonomy_relations SET target_skill_id = p_winner_id
    WHERE target_skill_id = p_loser_id
      AND entity_type = 'skill'
      AND NOT EXISTS (
        SELECT 1 FROM taxonomy_relations tr2
        WHERE tr2.source_term = taxonomy_relations.source_term
          AND tr2.entity_type = taxonomy_relations.entity_type
          AND tr2.target_skill_id = p_winner_id
      );

  -- Soft-delete loser.
  UPDATE job_canonical_skills SET
    status = 'deprecated',
    merged_into = p_winner_id,
    updated_at = NOW()
  WHERE id = p_loser_id;

  -- Bump updated_at do winner. Sem mesclagem temporal (colunas first_seen_at/last_seen_at
  -- não existem em job_canonical_skills — mesclagem temporal de produção é em
  -- job_canonical_role_sources, não em JCS/JCR).
  UPDATE job_canonical_skills SET updated_at = NOW() WHERE id = p_winner_id;

  -- Audit em events.
  BEGIN
    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, reason, metadata)
    VALUES (
      'canonical_skill_merged_admin', 'admin', p_decided_by_actor_id, 'canonical_skill', p_loser_id,
      p_reason,
      jsonb_build_object(
        'winner_id', p_winner_id,
        'loser_id',  p_loser_id,
        'cross_type', v_winner.skill_type IS DISTINCT FROM v_loser.skill_type,
        'cross_type_confirmed', p_cross_type_confirmed
      )
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[merge_canonical_skills] events INSERT failed: %', SQLERRM;
  END;

  -- Decision log dedicado. Convenção source_id/target_id alinhada com mig 17 (UNIQUE expression
  -- LEAST/GREATEST sobre source_id, target_id). Schema real da tabela usa actor_id (não
  -- decided_by_actor_id) — confirmado por ground truth. O parâmetro da função permanece
  -- p_decided_by_actor_id por convenção semântica do contrato; apenas o nome da coluna no
  -- INSERT difere.
  BEGIN
    INSERT INTO skill_merge_decisions (source_id, target_id, actor_id, reason, decided_at)
    VALUES (p_loser_id, p_winner_id, p_decided_by_actor_id, p_reason, NOW());
  EXCEPTION WHEN unique_violation THEN
    NULL; -- idempotente via mig 17 LEAST/GREATEST
  WHEN OTHERS THEN
    RAISE WARNING '[merge_canonical_skills] skill_merge_decisions INSERT failed: %', SQLERRM;
  END;

  RETURN jsonb_build_object('merged', TRUE, 'winner_id', p_winner_id, 'loser_id', p_loser_id);
END; $$;

REVOKE ALL ON FUNCTION merge_canonical_skills(UUID,UUID,UUID,TEXT,BOOLEAN) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION merge_canonical_skills(UUID,UUID,UUID,TEXT,BOOLEAN) TO service_role;

-- merge_skills (pipeline, 4 args) — preserva ordem (loser, winner) + AE-6 + AE-7
CREATE OR REPLACE FUNCTION merge_skills(
  p_loser_id UUID,
  p_winner_id UUID,
  p_actor TEXT,
  p_actor_id UUID
) RETURNS VOID
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_winner_status TEXT;
  v_loser_status TEXT;
BEGIN
  IF p_winner_id = p_loser_id THEN RETURN; END IF;

  SELECT status INTO v_winner_status FROM job_canonical_skills WHERE id = p_winner_id FOR UPDATE;
  SELECT status INTO v_loser_status  FROM job_canonical_skills WHERE id = p_loser_id  FOR UPDATE;

  IF v_winner_status IS NULL OR v_loser_status IS NULL THEN
    RAISE WARNING '[merge_skills] winner ou loser ausente';
    RETURN;
  END IF;

  -- AE-7: guard de loser status. Previne sobrescrita em cadeias de merge.
  IF v_loser_status NOT IN ('active','pending') THEN
    RAISE WARNING '[merge_skills] loser % em status % — skip', p_loser_id, v_loser_status;
    RETURN;
  END IF;

  -- Redirect operacional com NOT EXISTS para evitar duplicate key.
  UPDATE job_posting_skills SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM job_posting_skills jps2
        WHERE jps2.job_posting_id = job_posting_skills.job_posting_id
          AND jps2.canonical_skill_id = p_winner_id
      );
  DELETE FROM job_posting_skills WHERE canonical_skill_id = p_loser_id;  -- duplicates

  UPDATE submitted_job_skills SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM submitted_job_skills sjs2
        WHERE sjs2.submitted_job_id = submitted_job_skills.submitted_job_id
          AND sjs2.canonical_skill_id = p_winner_id
      );
  DELETE FROM submitted_job_skills WHERE canonical_skill_id = p_loser_id;

  -- AE-6: NÃO atualizar analysis_skill_matches.canonical_skill_id nem matched_via_similar_skill_id.
  -- Tabela é snapshot histórico puro. UI lateral lida com label de canonical deprecated em hover.

  UPDATE canonical_skills_summary SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_skills_summary css2
        WHERE css2.canonical_role_id = canonical_skills_summary.canonical_role_id
          AND css2.canonical_skill_id = p_winner_id
      );
  DELETE FROM canonical_skills_summary WHERE canonical_skill_id = p_loser_id;

  -- D-PS-32: resume_skill_enrichments.canonical_skill_id é "Híbrido por coluna" — atualiza em
  -- merge_skills E em merge_canonical_skills (D-PS-32, Anexo B). Sem isso, callers TS de
  -- validate/route.ts e resolve/route.ts retornam label de loser deprecated em vez do winner.
  UPDATE resume_skill_enrichments SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM resume_skill_enrichments rse2
        WHERE rse2.analysis_id = resume_skill_enrichments.analysis_id
          AND rse2.canonical_skill_id = p_winner_id
      );
  DELETE FROM resume_skill_enrichments WHERE canonical_skill_id = p_loser_id;

  -- Anexo B: canonical_cbo_links.canonical_skill_id é "Operacional" — atualiza em
  -- merge_skills E merge_canonical_skills com NOT EXISTS guard. Sem isso, links CBO
  -- ficam apontando para canonical deprecated pós-merge.
  UPDATE canonical_cbo_links SET canonical_skill_id = p_winner_id
    WHERE canonical_skill_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_cbo_links ccl2
        WHERE ccl2.occupation_code = canonical_cbo_links.occupation_code
          AND ccl2.canonical_skill_id = p_winner_id
      );
  DELETE FROM canonical_cbo_links WHERE canonical_skill_id = p_loser_id;

  UPDATE taxonomy_relations SET target_skill_id = p_winner_id
    WHERE target_skill_id = p_loser_id
      AND entity_type = 'skill'
      AND NOT EXISTS (
        SELECT 1 FROM taxonomy_relations tr2
        WHERE tr2.source_term = taxonomy_relations.source_term
          AND tr2.entity_type = taxonomy_relations.entity_type
          AND tr2.target_skill_id = p_winner_id
      );

  -- Soft-delete loser + bump winner. Sem mesclagem temporal.
  UPDATE job_canonical_skills SET
    status = 'deprecated',
    merged_into = p_winner_id,
    updated_at = NOW()
  WHERE id = p_loser_id;

  UPDATE job_canonical_skills SET updated_at = NOW() WHERE id = p_winner_id;

  BEGIN
    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
      'canonical_skill_merged', p_actor, p_actor_id, 'canonical_skill', p_loser_id,
      jsonb_build_object('winner_id', p_winner_id, 'loser_id', p_loser_id)
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[merge_skills] events INSERT failed: %', SQLERRM;
  END;
END; $$;

REVOKE ALL ON FUNCTION merge_skills(UUID,UUID,TEXT,UUID) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION merge_skills(UUID,UUID,TEXT,UUID) TO service_role;
```

### 2.36 — `36_rewrite_merge_canonicals.sql`

Reescrita explícita de `merge_canonicals` (4 args, lado role).

**Preservação de ordem produção** `(p_loser_id, p_winner_id, p_actor, p_actor_id)` — D-PS-60. A ordem matches a assinatura legada `(p_loser_id, p_winner_id, p_reason, p_decided_by DEFAULT 'system')`. Mudanças em relação à versão de produção:
- 4º arg muda de `TEXT 'system'` para `UUID actor_id` (sem DEFAULT) — alinha com D-PS-37 (separação `actor TEXT` + `actor_id UUID`).
- 3º arg renomeia `p_reason` → `p_actor` para refletir D-PS-37.
- AE-7: guard de loser status antes das fases de redirect.
- AE-9: NOT EXISTS guards de `taxonomy_relations` ganham `AND tr2.entity_type = tr1.entity_type` para não cross-contaminar role↔skill em runs concorrentes.
- `target_canonical_id` substituído por `target_role_id` (paridade com mig 12 e mig 12b).
- **FASE 2 reescrita em 3 etapas** (UNIQUE-safe): `job_canonical_role_sources` tem constraint `uq_jcrs_canonical_employer (canonical_role_id, employer_id)`. UPDATE direto SET `canonical_role_id = p_winner_id` viola constraint quando winner E loser têm row para o mesmo employer. Padrão correto: (2a) temporal merge nas winner rows fazendo JOIN com loser pelo employer_id (preserva LEAST/GREATEST sem criar duplicatas); (2b) redirect das loser rows que não têm conflito (NOT EXISTS guard); (2c) DELETE residual das loser rows duplicadas.
- **FASE 3 corrigida**: NOT EXISTS guard em `canonical_role_domain_links` usa coluna `domain_id` (correta pela UNIQUE constraint `(canonical_role_id, domain_id)`), não `canonical_role_domain_id` (coluna inexistente).
- **Sem mesclagem temporal em JCR.** Versão de produção usa `first_seen_at`/`last_seen_at` em `job_canonical_role_sources` (FASE 2). Aqui o UPDATE com LEAST/GREATEST em colunas de JCR falharia em runtime — colunas não existem na tabela canônica.

```sql
CREATE OR REPLACE FUNCTION merge_canonicals(
  p_loser_id UUID,
  p_winner_id UUID,
  p_actor TEXT,
  p_actor_id UUID
) RETURNS VOID
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_winner job_canonical_roles%ROWTYPE;
  v_loser  job_canonical_roles%ROWTYPE;
  v_similarity NUMERIC;
BEGIN
  IF p_winner_id = p_loser_id THEN RETURN; END IF;

  IF p_winner_id < p_loser_id THEN
    SELECT * INTO v_winner FROM job_canonical_roles WHERE id = p_winner_id FOR UPDATE;
    SELECT * INTO v_loser  FROM job_canonical_roles WHERE id = p_loser_id  FOR UPDATE;
  ELSE
    SELECT * INTO v_loser  FROM job_canonical_roles WHERE id = p_loser_id  FOR UPDATE;
    SELECT * INTO v_winner FROM job_canonical_roles WHERE id = p_winner_id FOR UPDATE;
  END IF;

  IF v_winner.id IS NULL OR v_loser.id IS NULL THEN
    RAISE WARNING '[merge_canonicals] winner ou loser ausente';
    RETURN;
  END IF;

  -- AE-7: guard de loser status.
  IF v_loser.status NOT IN ('active','pending') THEN
    RAISE WARNING '[merge_canonicals] loser % em status % — skip', p_loser_id, v_loser.status;
    RETURN;
  END IF;

  -- Similaridade via embedding cosine (só para audit metadata).
  IF v_winner.embedding IS NOT NULL AND v_loser.embedding IS NOT NULL THEN
    v_similarity := 1 - (v_winner.embedding <=> v_loser.embedding);
  ELSE
    v_similarity := NULL;
  END IF;

  -- FASE 1: redirect operacional simples.
  UPDATE job_postings SET canonical_role_id = p_winner_id WHERE canonical_role_id = p_loser_id;
  UPDATE function_orchestrator_items SET canonical_role_id = p_winner_id WHERE canonical_role_id = p_loser_id;
  UPDATE job_no_postings SET canonical_role_id = p_winner_id WHERE canonical_role_id = p_loser_id;

  -- FASE 2: job_canonical_role_sources tem UNIQUE (canonical_role_id, employer_id) — uq_jcrs_canonical_employer.
  -- UPDATE direto SET canonical_role_id = p_winner_id viola constraint quando winner E loser têm
  -- linha para o mesmo employer. Padrão de 3 etapas (alinhado à versão de produção):
  --   2a) temporal merge nas winner rows fazendo JOIN com loser pelo employer_id (preserva LEAST/GREATEST sem criar duplicatas)
  --   2b) redirect das loser rows que NÃO têm conflito com o winner (NOT EXISTS guard)
  --   2c) DELETE residual das loser rows duplicadas (winner já tem row para mesmo employer)

  -- 2a — temporal merge nas winner rows (employer-matched).
  UPDATE job_canonical_role_sources winner SET
    first_seen_at = LEAST(winner.first_seen_at, loser.first_seen_at),
    last_seen_at  = GREATEST(winner.last_seen_at, loser.last_seen_at)
  FROM job_canonical_role_sources loser
  WHERE winner.canonical_role_id = p_winner_id
    AND loser.canonical_role_id  = p_loser_id
    AND winner.employer_id = loser.employer_id;

  -- 2b — redirect loser rows sem conflito.
  UPDATE job_canonical_role_sources SET canonical_role_id = p_winner_id
  WHERE canonical_role_id = p_loser_id
    AND NOT EXISTS (
      SELECT 1 FROM job_canonical_role_sources w
      WHERE w.canonical_role_id = p_winner_id
        AND w.employer_id = job_canonical_role_sources.employer_id
    );

  -- 2c — delete residuais (loser rows com employer já presente no winner).
  DELETE FROM job_canonical_role_sources WHERE canonical_role_id = p_loser_id;

  -- FASE 3: redirect com NOT EXISTS guard.
  UPDATE canonical_cbo_links SET canonical_role_id = p_winner_id
    WHERE canonical_role_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_cbo_links ccl2
        WHERE ccl2.occupation_code = canonical_cbo_links.occupation_code
          AND ccl2.canonical_role_id = p_winner_id
      );
  DELETE FROM canonical_cbo_links WHERE canonical_role_id = p_loser_id;

  UPDATE taxonomy_family_canonicals SET canonical_role_id = p_winner_id
    WHERE canonical_role_id = p_loser_id
      AND entity_type = 'role'
      AND NOT EXISTS (
        SELECT 1 FROM taxonomy_family_canonicals tfc2
        WHERE tfc2.family_id = taxonomy_family_canonicals.family_id
          AND tfc2.canonical_role_id = p_winner_id
      );
  DELETE FROM taxonomy_family_canonicals WHERE canonical_role_id = p_loser_id;

  -- canonical_role_domain_links: UNIQUE constraint é (canonical_role_id, domain_id).
  -- Coluna correta para guard NOT EXISTS é domain_id, não canonical_role_domain_id.
  UPDATE canonical_role_domain_links SET canonical_role_id = p_winner_id
    WHERE canonical_role_id = p_loser_id
      AND NOT EXISTS (
        SELECT 1 FROM canonical_role_domain_links crdl2
        WHERE crdl2.domain_id = canonical_role_domain_links.domain_id
          AND crdl2.canonical_role_id = p_winner_id
      );
  DELETE FROM canonical_role_domain_links WHERE canonical_role_id = p_loser_id;

  -- AE-9: NOT EXISTS com filtro de entity_type para isolar role↔skill.
  UPDATE taxonomy_relations tr1 SET target_role_id = p_winner_id
    WHERE tr1.target_role_id = p_loser_id
      AND tr1.entity_type = 'role'
      AND NOT EXISTS (
        SELECT 1 FROM taxonomy_relations tr2
        WHERE tr2.source_term = tr1.source_term
          AND tr2.entity_type = tr1.entity_type
          AND tr2.target_role_id = p_winner_id
      );
  DELETE FROM taxonomy_relations
    WHERE target_role_id = p_loser_id AND entity_type = 'role';

  UPDATE resume_skill_enrichments SET canonical_role_id = p_winner_id
    WHERE canonical_role_id = p_loser_id;

  -- Bump updated_at do winner (sem mesclagem temporal em JCR — colunas não existem).
  UPDATE job_canonical_roles SET updated_at = NOW() WHERE id = p_winner_id;

  -- Soft-delete loser.
  UPDATE job_canonical_roles SET
    status = 'deprecated',
    merged_into = p_winner_id,
    updated_at = NOW()
  WHERE id = p_loser_id;

  -- Notificar usuários se label mudar.
  -- Assinatura de produção: mark_users_for_label_change_notification(p_canonical_id uuid, p_cutoff_iso text).
  -- O 2º arg é timestamp em formato ISO 8601 — body interno faz p_cutoff_iso::TIMESTAMPTZ.
  -- Padrão da produção: cutoff = NOW() - 24 hours (notifica usuários ativos nas últimas 24h).
  PERFORM mark_users_for_label_change_notification(
    p_loser_id,
    (NOW() - INTERVAL '24 hours')::text
  );

  BEGIN
    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
      'canonical_role_merged', p_actor, p_actor_id, 'job_canonical_role', p_loser_id,
      jsonb_build_object(
        'winner_id', p_winner_id,
        'loser_id',  p_loser_id,
        'similarity', v_similarity
      )
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[merge_canonicals] events INSERT failed: %', SQLERRM;
  END;
END; $$;

REVOKE ALL ON FUNCTION merge_canonicals(UUID,UUID,TEXT,UUID) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION merge_canonicals(UUID,UUID,TEXT,UUID) TO service_role;
```

## Bloco I — Hardening + calibração viva

### 2.37 — `37_hardening_search_path.sql`

```sql
DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def FROM pg_proc
  WHERE proname = 'fn_update_canonical_vacancy_count' LIMIT 1;
  IF v_def NOT LIKE '%search_path%' THEN
    ALTER FUNCTION fn_update_canonical_vacancy_count() SET search_path = public, pg_temp;
  END IF;
END $$;

DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def FROM pg_proc
  WHERE proname = 'fn_update_employer_vacancy_count' LIMIT 1;
  IF v_def NOT LIKE '%search_path%' THEN
    ALTER FUNCTION fn_update_employer_vacancy_count() SET search_path = public, pg_temp;
  END IF;
END $$;

DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def FROM pg_proc
  WHERE proname = 'fn_retire_canonical_on_zero_vacancy' LIMIT 1;
  IF v_def NOT LIKE '%search_path%' THEN
    ALTER FUNCTION fn_retire_canonical_on_zero_vacancy() SET search_path = public, pg_temp;
  END IF;
END $$;

DO $$ DECLARE bad INT; BEGIN
  SELECT COUNT(*) INTO bad FROM pg_proc
  WHERE pronamespace = 'public'::regnamespace
    AND proname IN ('fn_update_canonical_vacancy_count','fn_update_employer_vacancy_count','fn_retire_canonical_on_zero_vacancy')
    AND NOT EXISTS (
      SELECT 1 FROM unnest(COALESCE(proconfig, ARRAY[]::TEXT[])) AS c
      WHERE c LIKE 'search_path=%'
    );
  IF bad > 0 THEN RAISE EXCEPTION '% funções sem search_path', bad; END IF;
END $$;
```

### 2.38 — `38_pipeline_config_seed.sql`

```sql
INSERT INTO pipeline_config (key, value, updated_by, updated_at) VALUES
  -- hard_gate (2)
  ('skill.hard_gate.min_confidence','0.70','sprint_paridade_skills', NOW()),
  ('role.hard_gate.min_confidence','0.70','sprint_paridade_skills', NOW()),
  -- promotion (8)
  ('skill.promotion.auto_min_confidence','0.85','sprint_paridade_skills', NOW()),
  ('skill.promotion.min_vacancies','5','sprint_paridade_skills', NOW()),
  ('skill.promotion.min_distinct_employers','2','sprint_paridade_skills', NOW()),
  ('skill.promotion.lookback_days','60','sprint_paridade_skills', NOW()),
  ('role.promotion.auto_min_confidence','0.85','sprint_paridade_skills', NOW()),
  ('role.promotion.min_vacancies','3','sprint_paridade_skills', NOW()),
  ('role.promotion.min_distinct_employers','2','sprint_paridade_skills', NOW()),
  ('role.promotion.lookback_days','60','sprint_paridade_skills', NOW()),
  -- merge_candidate (6)
  ('skill.merge_candidate.cosine_threshold','0.85','sprint_paridade_skills', NOW()),
  ('skill.merge_candidate.lookback_days','7','sprint_paridade_skills', NOW()),
  ('skill.merge_candidate.opus_review_cooldown_days','90','sprint_paridade_skills', NOW()),
  ('role.merge_candidate.cosine_threshold','0.92','sprint_paridade_skills', NOW()),
  ('role.merge_candidate.lookback_days','7','sprint_paridade_skills', NOW()),
  ('role.merge_candidate.opus_review_cooldown_days','90','sprint_paridade_skills', NOW()),
  -- retirement (2)
  ('skill.retirement.gap_days','365','sprint_paridade_skills', NOW()),
  ('role.retirement.gap_days','365','sprint_paridade_skills', NOW()),
  -- opus_review (2 — apenas cooldown; piso e teto via hard_gate e auto_min_confidence)
  ('skill.opus_review.cooldown_days','90','sprint_paridade_skills', NOW()),
  ('role.opus_review.cooldown_days','90','sprint_paridade_skills', NOW()),
  -- confidence runtime (4)
  ('skill.confidence.lookback_days','120','sprint_paridade_skills', NOW()),
  ('skill.confidence.min_count','3','sprint_paridade_skills', NOW()),
  ('role.confidence.lookback_days','120','sprint_paridade_skills', NOW()),
  ('role.confidence.min_count','5','sprint_paridade_skills', NOW())
ON CONFLICT (key) DO NOTHING;

DO $$ DECLARE c INT; BEGIN
  SELECT COUNT(*) INTO c FROM pipeline_config WHERE key LIKE 'skill.%' OR key LIKE 'role.%';
  IF c < 24 THEN RAISE EXCEPTION 'Esperadas 24 chaves, encontradas %', c; END IF;
END $$;
```

### 2.39 — `39_pipeline_config_evolve.sql`

```sql
ALTER TABLE pipeline_config
  ADD COLUMN IF NOT EXISTS description TEXT NOT NULL DEFAULT '',
  ADD COLUMN IF NOT EXISTS last_change_reason TEXT NULL,
  ADD COLUMN IF NOT EXISTS last_changed_by_actor_id UUID NULL;
```

### 2.40 — `40_pipeline_config_history.sql`

```sql
CREATE TABLE IF NOT EXISTS pipeline_config_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key TEXT NOT NULL,
  previous_value TEXT NULL,
  new_value TEXT NOT NULL,
  changed_by UUID NOT NULL,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reason TEXT NULL
);

CREATE INDEX idx_pch_key_changed_at ON pipeline_config_history(key, changed_at DESC);

ALTER TABLE pipeline_config_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE pipeline_config_history FORCE ROW LEVEL SECURITY;

CREATE POLICY pch_select_all ON pipeline_config_history FOR SELECT TO authenticated, service_role USING (TRUE);
CREATE POLICY pch_insert_service ON pipeline_config_history FOR INSERT TO service_role WITH CHECK (TRUE);

CREATE OR REPLACE FUNCTION fn_pipeline_config_audit()
RETURNS TRIGGER LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp AS $$
BEGIN
  IF OLD.value IS DISTINCT FROM NEW.value THEN
    INSERT INTO pipeline_config_history (key, previous_value, new_value, changed_by, changed_at, reason)
    VALUES (NEW.key, OLD.value, NEW.value,
            COALESCE(NEW.last_changed_by_actor_id, '00000000-0000-0000-0000-000000000001'::uuid),
            NOW(), NEW.last_change_reason);
  END IF;
  RETURN NEW;
END; $$;

DROP TRIGGER IF EXISTS trg_pipeline_config_audit ON pipeline_config;
CREATE TRIGGER trg_pipeline_config_audit
  AFTER UPDATE ON pipeline_config
  FOR EACH ROW EXECUTE FUNCTION fn_pipeline_config_audit();
```

### 2.41 — `41_pipeline_config_descriptions_seed.sql`

```sql
UPDATE pipeline_config SET description = CASE key
  WHEN 'skill.hard_gate.min_confidence' THEN 'Confidence mínima para skill ser aceita pelo Hard Gate em runtime (filtro de entrada antes do INSERT em job_posting_skills). Também serve como piso da zona Opus por construção: filtra eventos individuais antes do agregado, logo a mediana nunca cai abaixo dele.'
  WHEN 'skill.promotion.auto_min_confidence' THEN 'Mediana mínima de skill_confidence para auto-promoção de skill pending para active. Também é o teto da zona Opus por convenção: o trigger residual fn_flag_needs_opus_review_jcs flagga canonicals com confidence_median abaixo desta linha.'
  WHEN 'skill.promotion.min_vacancies' THEN 'Quantidade mínima de vagas distintas com a skill para auto-promoção.'
  WHEN 'skill.promotion.min_distinct_employers' THEN 'Quantidade mínima de empregadores distintos para auto-promoção.'
  WHEN 'skill.promotion.lookback_days' THEN 'Janela em dias para verificar existência qualitativa de postings recentes na promoção. Diferente da janela de skill.confidence.lookback_days usada para confidence_median runtime.'
  WHEN 'skill.merge_candidate.cosine_threshold' THEN 'Similaridade cosine mínima para par de skills ser detectado como candidato a merge.'
  WHEN 'skill.merge_candidate.lookback_days' THEN 'Janela em dias para detect_skill_merge_candidates considerar pares como candidatos novos.'
  WHEN 'skill.merge_candidate.opus_review_cooldown_days' THEN 'Cooldown em dias entre arbitragens Opus de um mesmo par de skill candidatas a merge.'
  WHEN 'skill.retirement.gap_days' THEN 'Dias sem novas vagas para skill ativa ser detectada como stale e aposentada.'
  WHEN 'skill.opus_review.cooldown_days' THEN 'Cooldown em dias entre flags needs_opus_review consecutivas para a mesma skill via trigger residual fn_flag_needs_opus_review_jcs.'
  WHEN 'skill.confidence.lookback_days' THEN 'Janela em dias para fn_jps_recompute_jcs computar confidence_median runtime.'
  WHEN 'skill.confidence.min_count' THEN 'HAVING COUNT mínimo para fn_jps_recompute_jcs publicar nova mediana; abaixo disso a mediana anterior é preservada.'
  WHEN 'role.hard_gate.min_confidence' THEN 'Confidence mínima para role ser aceita pelo Hard Gate em runtime. Também serve como piso da zona Opus por construção.'
  WHEN 'role.promotion.auto_min_confidence' THEN 'Snapshot de confidence registrado em confidence_median_at_promotion no momento da promoção. NÃO é gate do trigger fn_promote_canonical_on_threshold — JCR promove apenas por vacancy_count e distinct_sources_count (D-PS-19). Chave existe para auditoria, painéis, simulações de calibração e para servir como teto da zona Opus do trigger residual fn_flag_needs_opus_review_jcr.'
  WHEN 'role.promotion.min_vacancies' THEN 'Quantidade mínima de vagas distintas com a role para auto-promoção.'
  WHEN 'role.promotion.min_distinct_employers' THEN 'Quantidade mínima de empregadores distintos.'
  WHEN 'role.promotion.lookback_days' THEN 'Janela em dias para verificar postings recentes na promoção.'
  WHEN 'role.merge_candidate.cosine_threshold' THEN 'Similaridade cosine mínima para par de roles ser detectado como candidato a merge.'
  WHEN 'role.merge_candidate.lookback_days' THEN 'Janela em dias para detect_canonical_merge_candidates considerar pares como candidatos novos.'
  WHEN 'role.merge_candidate.opus_review_cooldown_days' THEN 'Cooldown em dias entre arbitragens Opus de um mesmo par de roles candidatas a merge.'
  WHEN 'role.retirement.gap_days' THEN 'Dias sem novas vagas para role ativa ser detectada como stale.'
  WHEN 'role.opus_review.cooldown_days' THEN 'Cooldown em dias entre flags needs_opus_review consecutivas para a mesma role via trigger residual fn_flag_needs_opus_review_jcr.'
  WHEN 'role.confidence.lookback_days' THEN 'Janela em dias para fn_recompute_jcr_confidence_median computar mediana de confidence (function_orchestrator_items).'
  WHEN 'role.confidence.min_count' THEN 'HAVING COUNT mínimo para fn_recompute_jcr_confidence_median publicar nova mediana.'
END
WHERE key LIKE 'skill.%' OR key LIKE 'role.%';

DO $$ DECLARE c INT; BEGIN
  SELECT COUNT(*) INTO c FROM pipeline_config
  WHERE (key LIKE 'skill.%' OR key LIKE 'role.%') AND (description IS NULL OR description = '');
  IF c > 0 THEN RAISE EXCEPTION '% chaves sem description', c; END IF;
END $$;
```

### 2.42 — `42_rpc_set_pipeline_config_value.sql`

```sql
CREATE OR REPLACE FUNCTION set_pipeline_config_value(
  p_key TEXT, p_value TEXT, p_actor_id UUID, p_reason TEXT DEFAULT NULL
) RETURNS VOID
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pipeline_config WHERE key = p_key) THEN
    RAISE EXCEPTION 'chave % não existe em pipeline_config', p_key;
  END IF;

  UPDATE pipeline_config
  SET value = p_value,
      last_changed_by_actor_id = p_actor_id,
      updated_at = NOW(),
      last_change_reason = p_reason
  WHERE key = p_key;
END; $$;

REVOKE ALL ON FUNCTION set_pipeline_config_value(TEXT, TEXT, UUID, TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION set_pipeline_config_value(TEXT, TEXT, UUID, TEXT) TO service_role;
```

### 2.43 — `43_helper_get_pipeline_config_value.sql`

```sql
CREATE OR REPLACE FUNCTION get_pipeline_config_value(p_key TEXT)
RETURNS TEXT LANGUAGE sql STABLE SET search_path = public, pg_temp
AS $$
  SELECT value FROM pipeline_config WHERE key = p_key LIMIT 1;
$$;

REVOKE ALL ON FUNCTION get_pipeline_config_value(TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION get_pipeline_config_value(TEXT) TO service_role, authenticated;
```

## Bloco J — Detectores

### 2.44 — `44_detect_canonical_merge_candidates_rewrite.sql`

Reescrita de `detect_canonical_merge_candidates` para consumir 3 chaves de `pipeline_config`: `role.merge_candidate.cosine_threshold`, `role.merge_candidate.lookback_days`, `role.merge_candidate.opus_review_cooldown_days`. Hardcodes (`0.92`, `7 days`, `90 days`) eliminados.

```sql
CREATE OR REPLACE FUNCTION detect_canonical_merge_candidates()
RETURNS jsonb
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_threshold NUMERIC;
  v_lookback_days INT;
  v_cooldown_days INT;
  v_inserted_count INT := 0;
BEGIN
  SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='role.merge_candidate.cosine_threshold'), 0.92)
    INTO v_threshold;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.merge_candidate.lookback_days'), 7)
    INTO v_lookback_days;
  SELECT COALESCE((SELECT value::INT FROM pipeline_config WHERE key='role.merge_candidate.opus_review_cooldown_days'), 90)
    INTO v_cooldown_days;

  EXECUTE format($q$
    WITH active_roles AS (
      SELECT id, embedding, last_opus_review_at FROM job_canonical_roles
      WHERE status = 'active' AND embedding IS NOT NULL
        AND (last_opus_review_at IS NULL OR last_opus_review_at < NOW() - INTERVAL '%s days')
    ),
    pairs AS (
      SELECT a.id AS canonical_a_id, b.id AS canonical_b_id,
             1 - (a.embedding <=> b.embedding) AS similarity
      FROM active_roles a
      CROSS JOIN LATERAL (
        SELECT id, embedding FROM active_roles ar2
        WHERE ar2.id > a.id
        ORDER BY a.embedding <=> ar2.embedding
        LIMIT 5
      ) b
      WHERE 1 - (a.embedding <=> b.embedding) >= %L::NUMERIC
    ),
    inserted AS (
      INSERT INTO canonical_merge_candidates (canonical_a_id, canonical_b_id, similarity, detected_at)
      SELECT canonical_a_id, canonical_b_id, similarity, NOW() FROM pairs
      WHERE NOT EXISTS (
        SELECT 1 FROM canonical_merge_candidates cmc
        WHERE cmc.canonical_a_id = pairs.canonical_a_id
          AND cmc.canonical_b_id = pairs.canonical_b_id
          AND cmc.detected_at >= NOW() - INTERVAL '%s days'
      )
      RETURNING 1
    )
    SELECT COUNT(*) FROM inserted
  $q$, v_cooldown_days, v_threshold, v_lookback_days)
  INTO v_inserted_count;

  RETURN jsonb_build_object('inserted', v_inserted_count, 'threshold', v_threshold);
END; $$;

REVOKE ALL ON FUNCTION detect_canonical_merge_candidates() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION detect_canonical_merge_candidates() TO service_role;
```

### 2.45 — `45_detect_skill_merge_candidates.sql`

```sql
CREATE OR REPLACE FUNCTION detect_skill_merge_candidates()
RETURNS jsonb
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
    pairs_detected INT := 0;
    canonicals_scanned INT := 0;
    last_run_at TIMESTAMPTZ;
    rec RECORD;
    partner RECORD;
    v_threshold NUMERIC;
    v_lookback_days INT;
    v_cooldown_days INT;
BEGIN
    v_threshold := COALESCE(
      (SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.merge_candidate.cosine_threshold'),
      0.85
    );
    v_lookback_days := COALESCE(
      (SELECT value::INT FROM pipeline_config WHERE key='skill.merge_candidate.lookback_days'),
      7
    );
    v_cooldown_days := COALESCE(
      (SELECT value::INT FROM pipeline_config WHERE key='skill.merge_candidate.opus_review_cooldown_days'),
      90
    );

    SELECT MAX(finished_at) INTO last_run_at
    FROM job_runs
    WHERE job_name = 'run_daily_pipeline_maintenance'
      AND status IN ('success', 'partial');

    IF last_run_at IS NULL THEN
        last_run_at := NOW() - (v_lookback_days || ' days')::INTERVAL;
    END IF;

    FOR rec IN
        EXECUTE format($q$
          SELECT id, embedding
          FROM job_canonical_skills
          WHERE status = 'active'
            AND embedding IS NOT NULL
            AND promoted_at IS NOT NULL
            AND promoted_at >= %L
            AND (last_opus_review_at IS NULL OR last_opus_review_at < NOW() - INTERVAL '%s days')
          LIMIT 1000
        $q$, last_run_at, v_cooldown_days)
    LOOP
        canonicals_scanned := canonicals_scanned + 1;

        FOR partner IN
            EXECUTE format($q$
              SELECT
                  jcs2.id AS partner_id,
                  1 - (jcs2.embedding <=> %L::vector) AS similarity
              FROM job_canonical_skills jcs2
              WHERE jcs2.id != %L
                AND jcs2.status = 'active'
                AND jcs2.embedding IS NOT NULL
                AND (jcs2.last_opus_review_at IS NULL OR jcs2.last_opus_review_at < NOW() - INTERVAL '%s days')
              ORDER BY jcs2.embedding <=> %L::vector
              LIMIT 5
            $q$, rec.embedding, rec.id, v_cooldown_days, rec.embedding)
        LOOP
            IF partner.similarity >= v_threshold THEN
                IF rec.id < partner.partner_id THEN
                    INSERT INTO canonical_skill_merge_candidates (canonical_a_id, canonical_b_id, similarity)
                    VALUES (rec.id, partner.partner_id, partner.similarity)
                    ON CONFLICT (canonical_a_id, canonical_b_id) DO NOTHING;
                ELSE
                    INSERT INTO canonical_skill_merge_candidates (canonical_a_id, canonical_b_id, similarity)
                    VALUES (partner.partner_id, rec.id, partner.similarity)
                    ON CONFLICT (canonical_a_id, canonical_b_id) DO NOTHING;
                END IF;

                IF FOUND THEN
                    pairs_detected := pairs_detected + 1;
                END IF;
            END IF;
        END LOOP;
    END LOOP;

    UPDATE job_canonical_skills
    SET status = 'merge_candidate'
    WHERE id IN (
        SELECT DISTINCT canonical_a_id FROM canonical_skill_merge_candidates
            WHERE resolved_at IS NULL AND opus_decision IS NULL
        UNION
        SELECT DISTINCT canonical_b_id FROM canonical_skill_merge_candidates
            WHERE resolved_at IS NULL AND opus_decision IS NULL
    )
    AND status = 'active';

    BEGIN
      INSERT INTO pipeline_calibration_metrics (
        metric_name, metric_value, mode, dimensions, computed_at
      ) VALUES (
        'skill_merge_candidate_detected',
        pairs_detected,
        'production',
        jsonb_build_object(
          'canonicals_scanned', canonicals_scanned,
          'threshold', v_threshold,
          'lookback_days', v_lookback_days,
          'cooldown_days', v_cooldown_days,
          'last_run_at', last_run_at
        ),
        NOW()
      );
    EXCEPTION WHEN OTHERS THEN
      RAISE WARNING '[detect_skill_merge_candidates] PCM INSERT failed: %', SQLERRM;
    END;

    RETURN jsonb_build_object(
        'pairs_detected', pairs_detected,
        'canonicals_scanned', canonicals_scanned,
        'last_run_at', last_run_at,
        'threshold', v_threshold,
        'strategy', 'delta_based_knn_top5_skill'
    );
END;
$$;

REVOKE ALL ON FUNCTION detect_skill_merge_candidates() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION detect_skill_merge_candidates() TO service_role;
```

## Bloco K — Drops finais

### 2.46 — `46_drop_refresh_canonical_confidence_median.sql`

```sql
DO $$ BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_trigger WHERE tgname = 'trg_foi_jcr_confidence_insert'
  ) THEN
    RAISE EXCEPTION 'trg_foi_jcr_confidence_insert ausente; não é seguro dropar refresh';
  END IF;
END $$;

DROP FUNCTION IF EXISTS refresh_canonical_confidence_median();

DO $$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_proc WHERE proname = 'refresh_canonical_confidence_median') THEN
    RAISE EXCEPTION 'refresh_canonical_confidence_median ainda existe';
  END IF;
END $$;
```

### 2.47 — `47_drop_refresh_canonical_skills_confidence_median.sql`

```sql
DO $$ BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_trigger WHERE tgname = 'trg_jps_insert_jcs_counts'
  ) THEN
    RAISE EXCEPTION 'trg_jps_insert_jcs_counts ausente; não é seguro dropar refresh';
  END IF;
END $$;

DROP FUNCTION IF EXISTS refresh_canonical_skills_confidence_median();

DO $$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_proc WHERE proname = 'refresh_canonical_skills_confidence_median') THEN
    RAISE EXCEPTION 'refresh_canonical_skills_confidence_median ainda existe';
  END IF;
END $$;
```

### 2.48 — `48_validate_no_legacy_callers.sql`

```sql
DO $$
DECLARE has_old_funcs BOOLEAN; BEGIN
  SELECT EXISTS (
    SELECT 1 FROM pg_proc
    WHERE proname IN ('refresh_canonical_confidence_median', 'refresh_canonical_skills_confidence_median',
                      'merge_canonical_skill')
  ) INTO has_old_funcs;

  IF has_old_funcs THEN
    RAISE EXCEPTION 'Funções legadas ainda existem';
  END IF;
END $$;

DO $$
DECLARE missing TEXT[]; BEGIN
  missing := ARRAY[]::TEXT[];

  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_promote_on_threshold')
    THEN missing := missing || 'trg_promote_on_threshold'; END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_foi_jcr_confidence_insert')
    THEN missing := missing || 'trg_foi_jcr_confidence_insert'; END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_flag_needs_opus_review_jcr')
    THEN missing := missing || 'trg_flag_needs_opus_review_jcr'; END IF;

  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_promote_skill_on_threshold')
    THEN missing := missing || 'trg_promote_skill_on_threshold'; END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_jps_insert_jcs_counts')
    THEN missing := missing || 'trg_jps_insert_jcs_counts'; END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname='trg_flag_needs_opus_review_jcs')
    THEN missing := missing || 'trg_flag_needs_opus_review_jcs'; END IF;

  IF NOT EXISTS (SELECT 1 FROM pg_proc WHERE proname='catchup_pending_skill_promotions')
    THEN missing := missing || 'catchup_pending_skill_promotions'; END IF;

  IF array_length(missing, 1) > 0 THEN
    RAISE EXCEPTION 'Triggers/funções críticos ausentes: %', missing;
  END IF;
END $$;

DO $$ DECLARE v_def TEXT; BEGIN
  SELECT pg_get_functiondef(oid) INTO v_def
  FROM pg_proc WHERE proname='catchup_pending_promotions' LIMIT 1;
  IF v_def NOT LIKE E'%status IN (\'pending\', \'retired\')%' THEN
    RAISE EXCEPTION 'catchup_pending_promotions sem suporte a retired';
  END IF;
END $$;
```

## Bloco L — Cleanup

### 2.49 — `49_drop_backup_schema.sql`

```sql
DROP SCHEMA IF EXISTS backup_paridade_skills CASCADE;

DO $$ BEGIN
  IF EXISTS (
    SELECT 1 FROM information_schema.schemata WHERE schema_name = 'backup_paridade_skills'
  ) THEN
    RAISE EXCEPTION 'backup_paridade_skills ainda existe';
  END IF;
END $$;
```

### 2.50 — `50_cleanup_legacy_backups.sql`

```sql
DROP TABLE IF EXISTS public.analyses_backup_pre_pr1;
DROP TABLE IF EXISTS public.job_canonical_role_sources_backup_2026_05_04;
DROP TABLE IF EXISTS public.job_canonical_role_sources_backup_pre_pr1;
DROP TABLE IF EXISTS public.job_canonical_roles_backup_2026_05_04;
DROP TABLE IF EXISTS public.job_canonical_roles_backup_pre_pr1;
DROP TABLE IF EXISTS public.job_postings_backup_2026_05_04;
DROP TABLE IF EXISTS public.job_postings_backup_pre_pr1;
DROP TABLE IF EXISTS public.taxonomy_family_canonicals_backup_2026_05_04;
DROP TABLE IF EXISTS public.taxonomy_relations_backup_2026_05_04;

DO $$ DECLARE remaining INT; BEGIN
  SELECT COUNT(*) INTO remaining
  FROM information_schema.tables
  WHERE table_schema = 'public'
    AND (table_name LIKE '%_backup_%' OR table_name LIKE '%_backup_pre_%');
  IF remaining > 0 THEN
    RAISE WARNING '% tabelas de backup ainda existem em public — verificar manualmente', remaining;
  END IF;
END $$;
```

---

# Parte 3 — Mudanças TypeScript

33 mudanças. §3.1 a §3.17 cobrem o pipeline operacional principal (Hard Gate, resolvers, cache, opus arbitration, dashboard, refactors JOIN). §3.18 e §3.19 corrigem bug de UX cross-type confirmation no admin merge skills + tratam 422 com similarity/threshold. §3.20 a §3.26 cobrem o pipeline de ingestão e enriquecimento de skills (gravação direta em JCS via embedding com guard de slug colidido pós-soft-delete, batch via RPC `seed_skill_batch_from_cbo`, deprecação do evaluator legado, wiring resume → curation). §3.27 e §3.28 enumeram exaustivamente os 12 callsites hardcoded migrados para `pipeline_config` (com instrução explícita de `git grep` prévio). §3.29 fornece helpers de cache padronizados. §3.30 renomeia o CRON `update-canonical-stats` para refletir escopo reduzido após migração de `confidence_median` para triggers em runtime. §3.31 audita e renomeia 22 callsites externos confirmados (com 13+ adicionais identificados via git grep do Claude Code, totalizando ao menos 22 arquivos) a opus-arbitration que usam string literal `canonical_skills` em `.from()/.table()` para `job_canonical_skills`. §3.32 migra hardcode de similaridade no endpoint admin merge-skills para `pipeline_config`. §3.33 atualiza wrapper programático `lib/taxonomy/merge-canonicals.ts` para a nova assinatura da RPC (4º arg muda de TEXT para UUID).

## 3.1 — `lib/pipeline/persist-curation.ts`

Hard Gate de skill lê `skill.hard_gate.min_confidence` de `pipeline_config` via `getConfigValue`. Boundary `< upper` no gate de promoção.

```ts
import { getConfigValue } from '@/lib/pipeline/pipeline-config';

const skillHardGateMin = parseFloat(await getConfigValue('skill.hard_gate.min_confidence') ?? '0.70');
if (skill.confidence < skillHardGateMin) {
  // skill descartada antes do INSERT em job_posting_skills
  continue;
}
```

## 3.2 — `lib/pipeline/skill-resolver.ts`

`needsOpusReviewSkill(confidence)` lê limiares de `pipeline_config` para boundary `[lower, upper)`. **D-PS-52**: zona Opus consome `hard_gate.min_confidence` (piso) e `promotion.auto_min_confidence` (teto) por construção, sem chaves intermediárias `confidence_lower/upper` (que mig 38 não semeia).

```ts
const lower = parseFloat(await getConfigValue('skill.hard_gate.min_confidence') ?? '0.70');
const upper = parseFloat(await getConfigValue('skill.promotion.auto_min_confidence') ?? '0.85');
return confidence >= lower && confidence < upper;
```

**Simetria**: `lib/pipeline/role-resolver.ts` (§3.8) aplica o equivalente com chaves `role.hard_gate.min_confidence` e `role.promotion.auto_min_confidence`.

## 3.3 — `lib/pipeline/taxonomy-cache.ts`

Cache segregado por `entity_type`. `getRelations(sourceTerm, entityType)` aceita parâmetro novo. Invalidate granular: `invalidateRelations(sourceTerm, entityType)`. Bucket Redis `tax:rel:role:<term>` e `tax:rel:skill:<term>` distintos.

## 3.4 — `lib/pipeline/curate-job-postings.ts`

Sonnet curador continua emitindo skills com `skill_confidence`. Filtro Hard Gate em `persist-curation.ts`.

## 3.5 — `lib/pipeline/upsert-canonical.ts`

`upsertCanonicalRole` e `upsertCanonicalSkill` setam `creation_confidence` no INSERT. Reaproveitamento de canonical existente preserva valor anterior.

## 3.6 — `lib/pipeline/opus-arbitration.ts`

Quatro blocos de mudanças (em ordem de aplicação). Total: **11 substituições**.

**Bloco 1 — 5 substituições de `canonical_skills` → `job_canonical_skills`** (após mig 05 rename). Linhas no arquivo `lib/pipeline/opus-arbitration.ts`:

| Linha | Tipo | Substituição |
|---|---|---|
| 157 | Constante string em mapa de queues | `quality_review_skill: 'canonical_skills'` → `quality_review_skill: 'job_canonical_skills'` |
| 511 | `.from()` | leitura da queue de quality_review_skill |
| 1201 | `.from()` | UPDATE de label |
| 1209 | `.from()` | UPDATE de needs_opus_review |
| 1226 | `.from()` | UPDATE de last_opus_review_at |

**Bloco 2 — 4 substituições de tabelas CBO renomeadas** (após migs 19-20). Linhas no arquivo `lib/pipeline/opus-arbitration.ts`:

| Linha | Tipo | Substituição |
|---|---|---|
| 656 | `.from('canonical_role_cbo')` | → `.from('canonical_cbo')` |
| 663 | `.from('canonical_role_cbo_links')` | → `.from('canonical_cbo_links')` |
| 831 | `.from('canonical_role_cbo')` | → `.from('canonical_cbo')` |
| 1118 | `.from('canonical_role_cbo')` | → `.from('canonical_cbo')` |

**Bloco 3 — 2 substituições em chamadas RPC `merge_canonicals`** (coordenado com mig 36 que muda 4º arg de `TEXT 'system'` para `UUID actor_id`). Linhas no arquivo `lib/pipeline/opus-arbitration.ts`:

| Linha | Antes | Depois |
|---|---|---|
| 937 | `rpc('merge_canonicals', { p_loser_id, p_winner_id, p_reason: 'opus_arbitration', p_decided_by: 'opus_4_7' })` | `rpc('merge_canonicals', { p_loser_id, p_winner_id, p_actor: 'opus_arbitration', p_actor_id: SYSTEM_USER_ID })` |
| 1020 | `rpc('merge_canonicals', { p_loser_id, p_winner_id, p_reason: 'opus_quality_review_merge', p_decided_by: 'opus_4_7' })` | `rpc('merge_canonicals', { p_loser_id, p_winner_id, p_actor: 'opus_quality_review_merge', p_actor_id: SYSTEM_USER_ID })` |

`SYSTEM_USER_ID` é a constante `'00000000-0000-0000-0000-000000000001'` (D-PS-37). Sem essa atualização, ambas as chamadas falham após mig 36 com "function does not exist" (rename de parâmetros) e/ou "invalid input syntax for type uuid: 'opus_4_7'" (mudança de tipo do 4º arg).

**Bloco 4 — Após `processOpusXXX` bem-sucedido** (KEEP, REFINE, APPROVE, MERGE, INACTIVATE, CREATE_NEW, REJECT — todos os 7 verdicts): invalidar cache E setar `last_opus_review_at = NOW()`. Aplicar em **todos** os caminhos de saída bem-sucedida — sem isso, cooldown de `<entity>.opus_review.cooldown_days` do trigger residual de flagging (mig 30) nunca fecha (D-PS-48).

```ts
// Helper centralizado — aplicar em todos os 7 success paths de processOpusXXX:
async function finalizeOpusArbitration(
  entityType: 'role' | 'skill',
  canonicalId: string,
  sourceTerm: string,
) {
  await supabase
    .from(entityType === 'role' ? 'job_canonical_roles' : 'job_canonical_skills')
    .update({ last_opus_review_at: new Date().toISOString() })
    .eq('id', canonicalId);

  await invalidateCanonicalCache(entityType, canonicalId);
  await invalidateRelations(sourceTerm, entityType);
}
```

Validação pós-deploy: `git grep -nE "canonical_role_cbo|'canonical_skills'|p_decided_by" lib/pipeline/opus-arbitration.ts` deve retornar zero matches.

## 3.7 — `lib/pipeline/skill-upsert.ts`

Passa `entity_type='skill'` em todas chamadas a `taxonomy-cache`. Lookup via `lookup_canonical_skill_by_normalized_alias`.

## 3.8 — `lib/pipeline/role-resolver.ts`

`needsOpusReviewRole(confidence)` lê limiares de `pipeline_config` para boundary `[lower, upper)`. **D-PS-52**: zona Opus consome `role.hard_gate.min_confidence` (piso) e `role.promotion.auto_min_confidence` (teto) por construção. Mesmo padrão de §3.2 (skill).

```ts
const lower = parseFloat(await getConfigValue('role.hard_gate.min_confidence') ?? '0.70');
const upper = parseFloat(await getConfigValue('role.promotion.auto_min_confidence') ?? '0.85');
return confidence >= lower && confidence < upper;
```

## 3.9 — `lib/api/admin/canonical-stats/update-canonical-stats.ts`

Steps 1+4 preservados. Steps 2 (`canonical_role_confidence_median`) e 3 (`canonical_skills_confidence_median`) removidos — agora computados em runtime via triggers.

## 3.10 — `lib/api/admin/dashboard/aggregateDayData.ts`

10 painéis Limiares via queries inline. Modos online (24h) e histórico (7d/30d) com endpoints distintos. Detalhe das queries em §5.

## 3.11 — `lib/admin/components/AdminDashboard.tsx`

Sub-aba "Limiares" na aba Operacional, abaixo da sub-aba "Cotidiano" existente. 10 painéis dispostos em grid 2 colunas.

## 3.12 — `app/api/cron/pipeline-maintenance/route.ts`

Adiciona invocação de `catchup_pending_skill_promotions` paritária a `catchup_pending_promotions`. Adiciona invocação de `detect_skill_merge_candidates`. Cadência atual via `vercel.json`: `*/15 * * * *`.

```ts
// Fase 5.x: catchups
await supabase.rpc('catchup_pending_promotions');
await supabase.rpc('catchup_pending_skill_promotions');

// Fase 5.y: detectores de merge
await supabase.rpc('detect_canonical_merge_candidates');
await supabase.rpc('detect_skill_merge_candidates');
```

## 3.13 — `lib/analysis/compare.ts`

**Path correto** (não `lib/api/compare/compare.ts`, que não existe). Arquivo está em `lib/analysis/`. Confirmado por ground truth.

**Refactor 3-query → 1-query**: JOIN em vez de N+1. Mantém schema de retorno.

**Mudanças coordenadas com mig 04 (drop `aliases` array) e mig 12 (taxonomy_relations)**:

- Linha 173 atual usa `.overlaps('aliases', normalizedCvSkills)` na tabela `canonical_skills`. Coluna `aliases` é dropada por mig 04. Sem este fix o lookup quebra em runtime após Etapa 3.
- Substituir lookup por aliases por chamada à RPC `lookup_canonical_skill_by_normalized_alias` (criada em mig 04 como stub e reescrita em mig 12 lendo de `taxonomy_relations` com `entity_type='skill'`).
- Refactor JOIN também elimina queries adicionais por linha — versão final faz uma única consulta combinando lookup direto (por slug/label) + lookup por alias (via RPC).

```ts
// ANTES (3 queries por skill, com lookup em coluna aliases dropada):
const direct = await supabase.from('canonical_skills')
  .select('id, label, skill_type')
  .eq('slug', slugify(cvSkill))
  .single();
if (!direct.data) {
  const byAlias = await supabase.from('canonical_skills')
    .select('id, label, skill_type')
    .overlaps('aliases', [normalize(cvSkill)])  // ← coluna dropada por mig 04
    .single();
  // ...
}

// DEPOIS (1 query principal + 1 RPC para alias fallback):
const direct = await supabase.from('job_canonical_skills')
  .select('id, label, skill_type')
  .eq('slug', slugify(cvSkill))
  .single();
if (!direct.data) {
  const { data: aliasId } = await supabase.rpc('lookup_canonical_skill_by_normalized_alias', {
    p_normalized: normalize(cvSkill),
  });
  // RPC retorna UUID do canonical ativo (ou NULL se não houver match em taxonomy_relations).
  // NULL é caso legítimo: skill nova no CV, ainda não taxonomizada — caller cria novo canonical
  // via fluxo de §3.20 (resolve_active_canonical_by_slug → INSERT em job_canonical_skills).
  if (aliasId === null) {
    // skill nova: criar novo canonical pendente (ver §3.20).
    return await createPendingCanonicalSkill(cvSkill);
  }
  // alias resolvido: buscar canonical ativo.
  const { data: byAlias } = await supabase.from('job_canonical_skills')
    .select('id, label, skill_type')
    .eq('id', aliasId)
    .single();
  // ...
}
```

Linhas 172, 173 e 216 também precisam atualizar `'canonical_skills'` → `'job_canonical_skills'` (parte do escopo §3.31).

## 3.14 — `app/api/admin/limiares/online/route.ts` + `app/api/admin/limiares/historical/route.ts`

Endpoints distintos para painéis Limiares modo online (24h, sliding) e histórico (7d/30d, agregado por dia).

## 3.15 — `lib/admin/components/SkillMap.tsx`

Refactor JOIN. Mantém schema de retorno.

## 3.16 — `scripts/seed-skills-from-cbo-batch.ts`

Batch de extração de skills a partir das ocupações CBO. Padrão `INSERT + ON CONFLICT DO NOTHING + SELECT`. Detalhes em §4.3.

## 3.17 — `scripts/pilot-skills-from-cbo.ts`

Piloto estratificado de 50 ocupações antes do batch full. Detalhes em §4.2.

## 3.18 — `app/api/admin/merge-skills/route.ts`

**Não é rewrite.** É um incremento que tem 4 mudanças coordenadas com a mig 35: adicionar passagem do campo `cross_type_confirmed`, **atualizar nomes de parâmetros da RPC para a nova assinatura** (`p_loser_id`, `p_winner_id`, `p_decided_by_actor_id`, `p_reason`, `p_cross_type_confirmed`), tratamento da resposta 409 quando a RPC retorna `requires_cross_type_confirmation`, e tratamento da resposta 422 quando similarity está abaixo do threshold de pipeline_config (§3.32). Toda a funcionalidade existente é preservada — o endpoint atual em produção já implementa cálculo de similarity, governance event, persistência em `skill_merge_decisions`, refresh de summary e validação de admin.

A mudança é cirúrgica:

```ts
// (1) Aceitar campo opcional cross_type_confirmed no body:
const {
  source_id, target_id, actor_id: _actorId,
  reason, similarity: _similarity,
  session_id: sessionId, profile_id: profileId,
  cross_type_confirmed = false,  // NOVO
} = await req.json();

// (2) Passar para a RPC com NOMES NOVOS (mig 35 renomeou parâmetros e mudou ordem para
//     a convenção de produção de merge_canonicals: loser primeiro, winner depois):
const { data, error } = await supabase.rpc('merge_canonical_skills', {
  p_loser_id: source_id,                    // antes: p_source_id
  p_winner_id: target_id,                   // antes: p_target_id
  p_decided_by_actor_id: finalActorId,      // antes: p_actor_id
  p_reason: reason ?? 'Merge manual via Admin',
  p_cross_type_confirmed: cross_type_confirmed,
});

// (3) Tratar resposta cross-type (RPC retorna jsonb com flag, não erro Postgres):
if (data?.requires_cross_type_confirmation) {
  return NextResponse.json({
    error: 'Cross-type merge requer confirmação explícita',
    requires_cross_type_confirmation: true,
    source_type: data.source_type,
    target_type: data.target_type,
  }, { status: 409 });
}

// (4) Resto do fluxo permanece idêntico — similarity, governance event,
//     skill_merge_decisions upsert, refresh_canonical_skills_summary_single.
//     A validação de threshold de similarity vive em §3.32 (pipeline_config), que retorna 422
//     com payload {error, similarity, threshold} antes mesmo de chegar à RPC.
```

Validação pós-deploy: chamadas com `p_source_id`/`p_target_id`/`p_actor_id` (nomes antigos) retornariam erro PostgREST "function not found in schema cache" — o rename é coordenado com mig 35 na Etapa 24, deploy TS na Etapa 28.

## 3.19 — `lib/admin/components/MergeSkillsModal.tsx`

Detecta resposta 409 com `requires_cross_type_confirmation` (cross-type) e resposta 422 com `similarity` + `threshold` (similaridade abaixo do limiar configurado em pipeline_config). Exibe modais distintos:

- **409 cross-type**: confirmação explícita com `source_type` e `target_type`. Botão "Confirmar mesmo assim" retentar POST com `cross_type_confirmed: true`.
- **422 similarity-low**: feedback informativo com `similarity` calculada e `threshold` atual. Não permite retry — admin precisa escolher canonicals mais próximos ou ajustar `skill.merge_candidate.cosine_threshold` em pipeline_config.

**Parent component:** importado e renderizado em `app/admin/merge-skills/page.tsx` (que já existe como UI inline para listar candidatos a merge). O modal substitui o caminho de POST direto que faz hoje pelo fluxo de duplo POST com confirmação. Adicionar import e estado de modal nesse componente:

```tsx
// app/admin/merge-skills/page.tsx (mudança incremental)
import { MergeSkillsModal } from '@/lib/admin/components/MergeSkillsModal';

// ... dentro do componente:
const [pendingMerge, setPendingMerge] = useState<{ source: Skill; target: Skill } | null>(null);

// Substituir o handler atual de "Mesclar" por:
function onClickMerge(source: Skill, target: Skill) {
  setPendingMerge({ source, target });
}

// Renderizar modal quando pendingMerge não for null:
{pendingMerge && (
  <MergeSkillsModal
    sourceSkill={pendingMerge.source}
    targetSkill={pendingMerge.target}
    onClose={() => setPendingMerge(null)}
    onSuccess={() => { setPendingMerge(null); refetchCandidates(); }}
  />
)}
```

```tsx
import { useState } from 'react';

export function MergeSkillsModal({ sourceSkill, targetSkill, onClose, onSuccess }: Props) {
  const [reason, setReason] = useState('Merge manual via Admin');
  const [submitting, setSubmitting] = useState(false);
  const [crossTypeConfirmation, setCrossTypeConfirmation] = useState<{
    sourceType: string;
    targetType: string;
  } | null>(null);
  const [similarityFeedback, setSimilarityFeedback] = useState<{
    similarity: number;
    threshold: number;
  } | null>(null);

  async function handleMerge(crossTypeConfirmed = false) {
    setSubmitting(true);
    try {
      const resp = await fetch('/api/admin/merge-skills', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          source_id: sourceSkill.id,
          target_id: targetSkill.id,
          reason,
          cross_type_confirmed: crossTypeConfirmed,
        }),
      });

      if (resp.status === 409) {
        const data = await resp.json();
        if (data.requires_cross_type_confirmation) {
          setCrossTypeConfirmation({
            sourceType: data.source_type,
            targetType: data.target_type,
          });
          return;
        }
      }

      if (resp.status === 422) {
        const data = await resp.json();
        if (typeof data.similarity === 'number' && typeof data.threshold === 'number') {
          setSimilarityFeedback({
            similarity: data.similarity,
            threshold: data.threshold,
          });
          return;
        }
      }

      if (!resp.ok) {
        const data = await resp.json();
        throw new Error(data.error ?? 'Erro ao fazer merge');
      }

      const result = await resp.json();
      onSuccess(result);
      onClose();
    } catch (err) {
      alert(`Falhou: ${(err as Error).message}`);
    } finally {
      setSubmitting(false);
    }
  }

  if (similarityFeedback) {
    return (
      <div className="modal modal-similarity-low">
        <h3>Similaridade abaixo do limiar configurado</h3>
        <p>
          Similaridade calculada: <strong>{similarityFeedback.similarity.toFixed(4)}</strong>
        </p>
        <p>
          Limiar mínimo (`skill.merge_candidate.cosine_threshold`):{' '}
          <strong>{similarityFeedback.threshold.toFixed(4)}</strong>
        </p>
        <p>
          O merge não pode prosseguir. Opções: escolher canonicals mais próximos,
          ou ajustar o limiar em pipeline_config (apenas para admins com permissão).
        </p>
        <button onClick={() => { setSimilarityFeedback(null); onClose(); }}>
          Entendi
        </button>
      </div>
    );
  }

  if (crossTypeConfirmation) {
    return (
      <div className="modal modal-cross-type-warning">
        <h3>Confirmação obrigatória — merge entre tipos diferentes</h3>
        <p>
          A skill de origem é do tipo <strong>{crossTypeConfirmation.sourceType}</strong>{' '}
          e a de destino é do tipo <strong>{crossTypeConfirmation.targetType}</strong>.
        </p>
        <p>
          Mesclar skills de tipos diferentes é decisão de baixa reversibilidade.
          Confirme apenas se a operação foi avaliada manualmente como correta.
        </p>
        <div className="actions">
          <button onClick={() => setCrossTypeConfirmation(null)} disabled={submitting}>
            Cancelar
          </button>
          <button
            onClick={() => {
              setCrossTypeConfirmation(null);
              handleMerge(true);
            }}
            disabled={submitting}
            className="btn-confirm-cross-type"
          >
            Confirmar mesmo assim
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="modal">
      <h3>Mesclar skills</h3>
      <textarea value={reason} onChange={e => setReason(e.target.value)} />
      <button onClick={() => handleMerge(false)} disabled={submitting}>
        {submitting ? 'Mesclando…' : 'Mesclar'}
      </button>
      <button onClick={onClose}>Cancelar</button>
    </div>
  );
}
```

## 3.20 — `lib/pipeline/ingest-job-and-discover-skills.ts` (arquivo novo)

Pipeline de ingestão de vaga: ao detectar skill nova via embedding (não há canonical existente acima do limiar de similaridade), escrever direto em `job_canonical_skills` com `source='llm_extractor'`, `status='pending'`, `creation_confidence=<conf>`. Trigger `fn_promote_skill_on_threshold` faz o resto via gate cumulativo conforme volume vai entrando em `job_posting_skills`.

**Diretório:** `lib/pipeline/`. Convivem aqui `curate-job-postings.ts`, `skill-resolver.ts`, `opus-arbitration.ts` e demais módulos do pipeline de ingestão.

**Resolução de slug colidido pós-soft-delete:** usar a RPC `resolve_active_canonical_by_slug` (§4.4) em vez de `INSERT ... ON CONFLICT(slug) DO NOTHING`. Quando uma skill foi previamente deprecated via merge, seu slug permanece em JCS (soft-delete universal); um INSERT do mesmo label colidiria silenciosamente, e a RPC retorna o canônico ativo (ou cria um novo após verificar que nenhum existe). Aplicação simétrica para roles e skills.

```ts
const hardGateMin = parseFloat(await getConfigValue('skill.hard_gate.min_confidence') ?? '0.70');

if (skill.confidence < hardGateMin) continue;

const { data: existingActive } = await supabase.rpc('resolve_active_canonical_by_slug', {
  p_entity_type: 'skill',
  p_slug: slugify(skill.label),
});

let canonicalId = existingActive;

if (!canonicalId) {
  const { data: newSkill } = await supabase
    .from('job_canonical_skills')
    .insert({
      label: skill.label,
      slug: slugify(skill.label),
      skill_type: skill.skill_type,
      status: 'pending',
      source: 'llm_extractor',
      creation_confidence: skill.confidence,
      taxonomy_content_version_id: currentVersionId,
    })
    .select('id')
    .single();
  canonicalId = newSkill.id;
}

await supabase.from('job_posting_skills').insert({
  job_posting_id: postingId,
  canonical_skill_id: canonicalId,
  skill_confidence: skill.confidence,
});
```

## 3.21 — `lib/pipeline/bulk-skill-enrichment.ts` (arquivo novo)

Funcionamento equivalente a §3.20 mas em batch. **Não usa `supabase.rpc('begin_transaction')`** — esse RPC não existe e o cliente Supabase JS não expõe controle transacional manual. A atomicidade do lote é garantida por uma RPC `seed_skill_batch_from_cbo(p_rows jsonb)` (§4.5) que faz INSERT + SELECT atomicamente dentro de uma transação plpgsql única, retornando o mapping `slug → id` para o caller fazer os inserts em `job_posting_skills`.

**Diretório:** `lib/pipeline/`.

```ts
const BATCH_SIZE = 100;

for (let i = 0; i < skills.length; i += BATCH_SIZE) {
  const batch = skills.slice(i, i + BATCH_SIZE);

  const skillRows = batch.map(s => ({
    label: s.label,
    slug: slugify(s.label),
    skill_type: s.skill_type,
    creation_confidence: s.confidence,
  }));

  // RPC plpgsql única: INSERT + ON CONFLICT DO NOTHING + SELECT atomicamente.
  // Retorna jsonb { mapping: { slug: id, ... }, inserted_count, reused_count }.
  const { data: result, error } = await supabase.rpc('seed_skill_batch_from_cbo', {
    p_rows: skillRows,
    p_taxonomy_content_version_id: currentVersionId,
  });

  if (error) throw new Error(`seed_skill_batch_from_cbo falhou: ${error.message}`);

  const slugToId = result.mapping as Record<string, string>;

  const linkRows = batch.map(s => ({
    job_posting_id: s.posting_id,
    canonical_skill_id: slugToId[slugify(s.label)],
    skill_confidence: s.confidence,
  }));

  const { error: linkError } = await supabase.from('job_posting_skills').upsert(linkRows, {
    onConflict: 'job_posting_id,canonical_skill_id',
    ignoreDuplicates: true,
  });

  if (linkError) throw new Error(`upsert job_posting_skills falhou: ${linkError.message}`);
}
```

## 3.22 — `lib/pipeline/skill-promotion-evaluator.ts` — DEPRECAR

A lógica de avaliação de promoção migra para o trigger SQL `fn_promote_skill_on_threshold` (D-PS-19). Caso o arquivo exista como módulo separado, é apagado; caso esteja inline em outro arquivo (`skill-upsert.ts` ou similar), o trecho é removido. Antigravity executa `git grep skill-promotion-evaluator` antes de qualquer remoção para localizar a forma real em código.

Validação pós-deploy: `git grep skill-promotion-evaluator` deve retornar zero ocorrências.

## 3.23 — `lib/pipeline/extract-skills-from-resume.ts` (arquivo novo)

Hard Gate por chave de config. Skills abaixo do threshold não entram em `resume_skill_enrichments`.

```ts
const hardGateMin = parseFloat(await getConfigValue('skill.hard_gate.min_confidence') ?? '0.70');

const filteredSkills = extracted.filter(s => s.confidence >= hardGateMin);

await supabase.from('resume_skill_enrichments').insert(
  filteredSkills.map(s => ({
    resume_id: resumeId,
    canonical_skill_id: s.canonical_skill_id,
    skill_confidence: s.confidence,
    canonical_role_id: roleContext?.id,
  }))
);
```

## 3.24 — `lib/jobs/process-resume.ts`

Wiring do extractor para curation: confidence individual é passada adiante para `persist-curation.ts` (§3.1) que aplica o gate de skill. Sem mudanças em schema; apenas atualizar tipos para incluir `skill_confidence: number` no payload.

## 3.25 — `lib/jobs/linkedin-fetch.ts`

Sem mudanças funcionais — apenas atualizar tipos consumidos para incluir as novas colunas de JCS quando ler skills associadas a vagas (`creation_confidence`, `vacancy_count_at_promotion`, `confidence_median_at_promotion`, `last_opus_review_at`).

## 3.26 — `lib/curation/data-curation.ts`

Wrapper consolidando §3.1 + §3.20 + §3.24 em fluxo único de curação. Confidence flui de extractor → wrapper → persist-curation → JCS+JPS. Garante que o mesmo valor de `skill_confidence` é usado tanto no Hard Gate quanto na agregação posterior em `confidence_median`.

## 3.27 — Callers hardcoded de `hard_gate.min_confidence` (5 arquivos)

**Nota de localização — aplicável a §3.27 e §3.28:** os módulos `lib/jobs/` e `lib/skills/` referenciados abaixo são rótulos lógicos. O critério estável é a chave de configuração consumida, não o nome do arquivo. Antigravity executa `git grep -lnE "0\.70|HARD_GATE|hard_gate\.min_confidence"` antes de cada edição para localizar o arquivo real (provavelmente em `lib/pipeline/`, `lib/extraction/`, `lib/curation/` ou inline em `curate-job-postings.ts`/`skill-resolver.ts`/`role-resolver.ts`). Se um arquivo listado por nome não for encontrado, a chave correspondente está em outro lugar — seguir pela chave.

Substituir constantes hardcoded por leitura de `pipeline_config` via helper `getConfigValue`. Lista por chave consumida (rótulo de arquivo é orientativo):

1. `lib/skills/skill-extraction.ts` — chave `skill.hard_gate.min_confidence`
2. `lib/jobs/ingest-job-and-discover-skills.ts` — chave `skill.hard_gate.min_confidence` (ver §3.20)
3. `lib/skills/bulk-skill-enrichment.ts` — chave `skill.hard_gate.min_confidence`
4. `lib/jobs/extract-roles.ts` — chave `role.hard_gate.min_confidence`
5. `lib/jobs/role-canonicalizer.ts` — chave `role.hard_gate.min_confidence`

Padrão de substituição:

```ts
// ANTES:
const HARD_GATE_MIN_CONFIDENCE = 0.70;

// DEPOIS:
const hardGateMinConfidence = parseFloat(
  await getConfigValue('skill.hard_gate.min_confidence') ?? '0.70'
);
```

Validação pós-deploy: `git grep -nE "(HARD_GATE|hard_gate.+0\.[0-9]+|0\.70.*hard)"` deve retornar zero ocorrências em código de produção (apenas em comentários de migração ou testes de regressão).

## 3.28 — Callers hardcoded de `promotion.*` (7 arquivos)

Substituir constantes de promoção (`lookback_days`, `min_vacancies`, `min_distinct_employers`, `auto_min_confidence`) por leitura de `pipeline_config`. Lista enumerada:

1. `lib/admin/canonical-promotion-checker.ts` — todas as 4 chaves de cada lado (role + skill = 8 leituras)
2. `lib/cron/promote-stale-pending.ts` — `*.promotion.lookback_days`, `*.promotion.min_vacancies`
3. `lib/admin/skills-dashboard-aggregator.ts` — `skill.promotion.auto_min_confidence` (para painel "Threshold Active")
4. `lib/admin/roles-dashboard-aggregator.ts` — `role.promotion.auto_min_confidence`
5. `lib/skills/manual-promote.ts` — todas as 4 chaves de skill
6. `lib/jobs/role-canonicalizer.ts` — `role.promotion.lookback_days` (segundo uso após §3.27)
7. `lib/admin/limites-snapshot.ts` — todas as 24 chaves (página snapshot da sub-aba Limiares)

Padrão idêntico ao §3.27. Ordem de prioridade na migração: arquivos #1, #5 e #7 são gates críticos; #2-4 e #6 são informativos.

## 3.29 — Helpers de cache em `lib/opus/opus-arbitration.ts`

3 funções helper para padronizar acesso a Redis com chaves segregadas por `entity_type` (D-PS-06):

```ts
export async function getCanonicalFromCache(
  entityType: 'role' | 'skill',
  id: string
): Promise<CanonicalRow | null> {
  const key = `canonical:${entityType}:${id}`;
  const cached = await redis.get(key);
  return cached ? JSON.parse(cached) : null;
}

export async function setCanonicalToCache(
  entityType: 'role' | 'skill',
  id: string,
  data: CanonicalRow
): Promise<void> {
  const key = `canonical:${entityType}:${id}`;
  await redis.set(key, JSON.stringify(data), { ex: 900 }); // TTL 900s
}

export async function invalidateCanonicalCache(
  entityType: 'role' | 'skill',
  id: string
): Promise<void> {
  const keys = [
    `canonical:${entityType}:${id}`,
    `canonical:${entityType}:list:active`,
    `canonical:${entityType}:embeddings`,
  ];
  await redis.del(...keys);
}
```

Chaves segregadas: `canonical:role:${id}`, `canonical:skill:${id}`. Helper `invalidateCanonicalCache` é chamado em todos os success paths de `processOpusXXX` (§3.6 D-PS-33).

## 3.30 — Rename CRON `update-canonical-stats` → `refresh-flags-and-retire-stale`

A CRON `update-canonical-stats` realizava 4 steps. Após esta sprint, Steps 2 e 3 (refresh de `confidence_median` para JCR e JCS) migram para triggers em runtime. Steps 1 e 4 permanecem ativos sem substituto trigger:

- **Step 1 preservado:** `refresh_employer_recruitment_agency_flags()` — atualiza flag `is_recruitment_agency` em `employers` baseado em heurística de proporção de postings + signals.
- **Step 2 removido:** migrou para 3 triggers em `function_orchestrator_items` (mig 29).
- **Step 3 removido:** migrou para 3 triggers em `job_posting_skills` (mig 27).
- **Step 4 preservado:** detectores de stale + `retire_canonical(id, reason, entity_type)` para JCR e JCS.

CRON renomeado para `refresh-flags-and-retire-stale` em `vercel.json` para refletir escopo reduzido. Schedule preservado (`0 3 * * *` daily). Renomear arquivo via Git para preservar histórico:

```bash
git mv app/api/cron/update-canonical-stats/route.ts \
       app/api/cron/refresh-flags-and-retire-stale/route.ts
git mv app/api/cron/update-canonical-stats \
       app/api/cron/refresh-flags-and-retire-stale  # rename do diretório se necessário
```

Atualizar também referências em logs e dashboards (Sentry tags, Vercel CRON name).

```json
// vercel.json (trecho)
{
  "crons": [
    { "path": "/api/cron/refresh-flags-and-retire-stale", "schedule": "0 3 * * *" }
  ]
}
```

A função `retire_canonical` é invocada com discriminador explícito para skills:

```ts
// Para roles (assinatura legada de 2 args, default 'role'):
await supabase.rpc('retire_canonical', { p_id: id, p_reason: reason });

// Para skills (assinatura estendida de 3 args, mig 23):
await supabase.rpc('retire_canonical', { p_id: id, p_reason: reason, p_entity_type: 'skill' });
```

## 3.31 — Rename global de callsites de `canonical_skills` (fora de opus-arbitration)

Após mig 05 renomear `canonical_skills → job_canonical_skills`, callsites TS que usam string literal `'canonical_skills'` em `.from()` quebram em runtime (PostgREST retorna 404). Auditoria por `git grep` real do Claude Code identificou 22 arquivos fora de `lib/opus/opus-arbitration.ts` (que tem tratamento próprio em §3.6):

```
# Pipeline core
lib/pipeline/upsert-canonical.ts
lib/pipeline/skill-resolver.ts
lib/pipeline/skill-upsert.ts
lib/pipeline/skill-promotion-evaluator.ts
lib/pipeline/canonical-skills.ts

# Análise (CV ↔ vagas)
lib/analysis/compare.ts                                  # linhas 172, 173, 216 — coordenado com §3.13
lib/analysis/skill-map.ts

# Admin endpoints
app/api/admin/merge-skills/route.ts                      # coordenado com §3.18
app/api/admin/merge-skills/search/route.ts
app/api/admin/merge-skills/preview/route.ts
app/api/admin/dashboard-state/route.ts
app/api/admin/dashboard-events/route.ts

# Admin libs
lib/api/admin/canonical-stats/update-canonical-stats.ts
lib/api/admin/dashboard/aggregateDayData.ts
lib/admin/components/SkillMap.tsx
lib/admin/components/MergeSkillsModal.tsx

# Optimization endpoints
app/api/optimization/validate/route.ts                   # linhas 118, 140
app/api/optimization/skill-labels/route.ts

# Analysis endpoints
app/api/analysis/suggest-functions/route.ts
app/api/analysis/calculate-score/route.ts

# Scripts
scripts/seed-skills-from-cbo-batch.ts
scripts/pilot-skills-from-cbo.ts
```

A lista é a baseline confirmada por ground truth no momento da redação. Antigravity ainda precisa rodar o `git grep` no commit pré-mig 05 para confirmar e capturar drift. Se grep retornar arquivos adicionais fora dessa lista, registrar em PR e atualizar a lista — **não é safe** confiar exclusivamente nesta lista sem validação mecânica.

**Procedimento de auditoria mecânica:**

```bash
# Antes da mig 05: snapshot baseline
git grep -nE "from\(['\"]canonical_skills['\"]\)|table\(['\"]canonical_skills['\"]\)" \
  -- 'lib/**/*.ts' 'app/**/*.ts' 'scripts/**/*.ts' \
  > /tmp/canonical_skills_callsites_before.txt

# Conferir contagem mínima:
wc -l /tmp/canonical_skills_callsites_before.txt
# Esperado: ≥22 arquivos

# Após substituições: deve retornar zero
git grep -nE "from\(['\"]canonical_skills['\"]\)|table\(['\"]canonical_skills['\"]\)" \
  -- 'lib/**/*.ts' 'app/**/*.ts' 'scripts/**/*.ts'
```

**Padrão de substituição:**

```ts
// ANTES:
await supabase.from('canonical_skills').select(...)

// DEPOIS:
await supabase.from('job_canonical_skills').select(...)
```

Aplicar em todos os arquivos retornados por `git grep`. Antigravity usa o mesmo padrão sed/replace no batch:

```bash
git ls-files '*.ts' | xargs sed -i \
  "s/\.from('canonical_skills')/\.from('job_canonical_skills')/g; \
   s/\.table('canonical_skills')/\.table('job_canonical_skills')/g"
```

**Não substituir** dentro de:
- Comentários explicativos (`// canonical_skills foi renomeada para...`)
- Descrições de `pipeline_config_history` (auditoria histórica)
- Strings em arquivos de migração SQL (corpo legítimo dessas migrations)
- `canonical_skills_summary` — esta tabela **não** é renomeada (D-PS-10)

**Validação pós-deploy:**

```bash
git grep -nE "['\"]canonical_skills['\"]" -- 'lib/**/*.ts' 'app/**/*.ts' 'scripts/**/*.ts' \
  | grep -v "canonical_skills_summary" \
  | grep -v "^[^:]*:[0-9]*:[[:space:]]*//"
# Esperado: zero linhas
```

Tipos TypeScript gerados por `supabase gen types` precisam ser regenerados para refletir o nome novo. Antigravity executa `npx supabase gen types typescript --project-id <id> > database.types.ts` após mig 05 e antes do build.

## 3.32 — `app/api/admin/merge-skills/route.ts` consome `pipeline_config`

O endpoint admin `merge-skills` calcula `similarity` via cosine entre embeddings antes de chamar `merge_canonical_skills`. Atualmente usa hardcode `>= 0.85` para aceitar o par; valor migra para `pipeline_config` chave `skill.merge_candidate.cosine_threshold` (paritário com detect_skill_merge_candidates).

```ts
// ANTES (hardcode):
if (similarity < 0.85) {
  return NextResponse.json({ error: 'Similaridade abaixo do limiar' }, { status: 422 });
}

// DEPOIS (via pipeline_config):
const { data: cfg } = await supabase
  .from('pipeline_config')
  .select('value')
  .eq('key', 'skill.merge_candidate.cosine_threshold')
  .single();
const minSimilarity = parseFloat(cfg?.value ?? '0.85');
if (similarity < minSimilarity) {
  return NextResponse.json({
    error: 'Similaridade abaixo do limiar',
    similarity,
    threshold: minSimilarity,
  }, { status: 422 });
}
```

Calibração admin via UPDATE em `pipeline_config` propaga sem deploy.

## 3.33 — `lib/taxonomy/merge-canonicals.ts`

Wrapper programático que invoca a RPC `merge_canonicals` a partir de código TS fora do pipeline Opus (admin scripts, testes E2E, ferramentas de migração). Caller único confirmado por ground truth: linha 26 do arquivo.

**Mudanças coordenadas com mig 36:**

| Aspecto | Antes | Depois |
|---|---|---|
| Nomes dos parâmetros | `p_loser_id, p_winner_id, p_reason` | `p_loser_id, p_winner_id, p_actor, p_actor_id` |
| Tipo do 4º arg | (não passado, default `'system'` TEXT) | `p_actor_id UUID` (sem default em mig 36) |
| Assinatura da função `mergeCanonicals()` exportada | `(loserId, winnerId, reason)` | `(loserId, winnerId, reason, actorId?: string)` |

```ts
// ANTES (linha 26):
await supabase.rpc('merge_canonicals', {
  p_loser_id: loserId,
  p_winner_id: winnerId,
  p_reason: reason,
});

// DEPOIS:
const SYSTEM_USER_ID = '00000000-0000-0000-0000-000000000001';

export async function mergeCanonicals(
  loserId: string,
  winnerId: string,
  reason: string,
  actorId: string = SYSTEM_USER_ID,  // default mantém compat com callers existentes
) {
  const { data, error } = await supabase.rpc('merge_canonicals', {
    p_loser_id: loserId,
    p_winner_id: winnerId,
    p_actor: reason,        // o reason vira o actor TEXT (contexto de origem)
    p_actor_id: actorId,    // UUID auditável
  });
  if (error) throw new Error(`merge_canonicals failed: ${error.message}`);
  await invalidateAllRelations();  // paridade com produção (linha 38 do wrapper atual)
  return data;
}
```

**Mapeamento semântico**: o campo `reason` legado (TEXT livre passado pelo caller, ex: `'admin_merge_via_script'`) vira o `p_actor` TEXT. O `p_actor_id` é UUID auditável — default `SYSTEM_USER_ID` para chamadas automáticas, ou UUID real do admin se invocado via ferramenta interativa.

Validação pós-deploy: `git grep -nE "rpc\(['\"]merge_canonicals['\"]" -- 'lib/**' 'app/**' 'scripts/**'` retorna 3 callers (linhas 26 deste arquivo + 937 e 1020 de `lib/pipeline/opus-arbitration.ts`), todos com a nova assinatura `p_actor`/`p_actor_id`. Zero matches de `p_decided_by` ou `p_reason` como nome de parâmetro RPC.

---

# Parte 4 — Arquivos novos

## 4.1 — `lib/pipeline/cbo-skill-extraction-prompt.ts`

```ts
export const CBO_SKILL_EXTRACTION_SYSTEM = `
Você é um extrator de skills profissionais a partir de descrições oficiais do MTE Brasil (CBO 2002).

Para cada ocupação, identifique skills concretas e operacionais que aparecem ou são fortemente implicadas pela descrição. Skills devem ser:
- Específicas (não genéricas tipo "trabalho em equipe")
- Operacionais (verificáveis em uma vaga real)
- Reutilizáveis (aparecerão em mais de uma ocupação)

Categorize em 'hard' (técnica/tangível) ou 'soft' (comportamental/interpessoal).

Output JSON apenas, sem prose:
{
  "skills": [
    { "label": "<nome canônico>", "skill_type": "hard"|"soft", "confidence": <0..1> }
  ]
}
`.trim();

export function buildCboUserPrompt(occupation: { code: string; title: string; summary: string }): string {
  return `Ocupação CBO: ${occupation.code} — ${occupation.title}\n\nDescrição oficial:\n${occupation.summary}\n\nExtraia skills.`;
}
```

## 4.2 — `scripts/pilot-skills-from-cbo.ts`

Piloto estratificado de 50 ocupações em 5 famílias diversas (10 cada). Persiste resultados em `pilots/cbo-skills-pilot-<timestamp>.json` para revisão humana antes do batch full. Não escreve em `job_canonical_skills`.

## 4.3 — `scripts/seed-skills-from-cbo-batch.ts`

Batch das ocupações CBO. Padrão `INSERT + ON CONFLICT DO NOTHING + SELECT` para preservar `skill_type` da primeira inserção em caso de race.

```ts
// Para cada ocupação:
//   skills = await callLLMSkillExtraction(occupation, CBO_SKILL_EXTRACTION_SYSTEM)
//   for (const s of skills) {
//     INSERT INTO job_canonical_skills (label, slug, status, source, skill_type, creation_confidence)
//     VALUES (s.label, slugify(s.label), 'pending', 'cbo_mte_2002_seed', s.skill_type, s.confidence)
//     ON CONFLICT (slug) DO NOTHING
//
//     SELECT id FROM job_canonical_skills WHERE slug = slugify(s.label)
//
//     INSERT INTO canonical_cbo_links (canonical_skill_id, occupation_code, is_primary, source)
//     VALUES (skill_id, occupation.code, FALSE, 'cbo_mte_2002_seed')
//     ON CONFLICT DO NOTHING
//   }
```

LK-PS-06: 25 ocupações CBO sem `summary_description` ficam fora.

## 4.4 — RPC `resolve_active_canonical_by_slug` (mig 2.42b — slug collision pós-soft-delete)

RPC com discriminador `entity_type` para servir aos dois lados (skill imediato em §3.20; role como débito documentado em LK-PS-18). Resolve o problema do slug colidido após soft-delete: quando uma skill ou role é deprecated via `merge_canonicals`/`merge_canonical_skills`, seu `slug` permanece em JCS/JCR com `status='deprecated'` e `merged_into = winner_id`. Re-ingestão do mesmo label sem essa RPC prenderia novas vagas em canônico morto, ou pior — caller tentaria INSERT com slug UNIQUE e bateria violação.

A função **segue a cadeia `merged_into` recursivamente** até encontrar o winner ativo final. Funciona também se o winner foi mergeado novamente em sequência (cadeia de merges):

```sql
CREATE OR REPLACE FUNCTION resolve_active_canonical_by_slug(
  p_entity_type TEXT,
  p_slug TEXT
) RETURNS UUID
LANGUAGE plpgsql STABLE SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_id UUID;
BEGIN
  IF p_entity_type NOT IN ('role','skill') THEN
    RAISE EXCEPTION 'entity_type inválido: %', p_entity_type;
  END IF;

  IF p_slug IS NULL OR length(trim(p_slug)) = 0 THEN
    RETURN NULL;
  END IF;

  IF p_entity_type = 'skill' THEN
    -- Caso 1: slug aponta direto para canonical ativo.
    SELECT id INTO v_id FROM job_canonical_skills
    WHERE slug = p_slug
      AND status IN ('active','pending')
      AND merged_into IS NULL
    LIMIT 1;

    IF v_id IS NOT NULL THEN RETURN v_id; END IF;

    -- Caso 2: slug aponta para canonical deprecated com merged_into.
    -- Segue a cadeia até encontrar winner ativo (proteção contra ciclo via depth limit).
    WITH RECURSIVE chain AS (
      SELECT id, merged_into, status, 0 AS depth
      FROM job_canonical_skills
      WHERE slug = p_slug AND status = 'deprecated' AND merged_into IS NOT NULL
      UNION ALL
      SELECT jcs.id, jcs.merged_into, jcs.status, c.depth + 1
      FROM job_canonical_skills jcs
      JOIN chain c ON jcs.id = c.merged_into
      WHERE c.depth < 10  -- proteção contra ciclo (não deve ocorrer em produção, defesa em profundidade)
    )
    SELECT c.id INTO v_id FROM chain c
    JOIN job_canonical_skills jcs ON jcs.id = c.id
    WHERE jcs.status IN ('active','pending') AND jcs.merged_into IS NULL
    ORDER BY c.depth DESC LIMIT 1;
  ELSE
    -- Lado role: estrutura paritária via discriminador.
    SELECT id INTO v_id FROM job_canonical_roles
    WHERE slug = p_slug
      AND status IN ('active','pending')
      AND merged_into IS NULL
    LIMIT 1;

    IF v_id IS NOT NULL THEN RETURN v_id; END IF;

    WITH RECURSIVE chain AS (
      SELECT id, merged_into, status, 0 AS depth
      FROM job_canonical_roles
      WHERE slug = p_slug AND status = 'deprecated' AND merged_into IS NOT NULL
      UNION ALL
      SELECT jcr.id, jcr.merged_into, jcr.status, c.depth + 1
      FROM job_canonical_roles jcr
      JOIN chain c ON jcr.id = c.merged_into
      WHERE c.depth < 10
    )
    SELECT c.id INTO v_id FROM chain c
    JOIN job_canonical_roles jcr ON jcr.id = c.id
    WHERE jcr.status IN ('active','pending') AND jcr.merged_into IS NULL
    ORDER BY c.depth DESC LIMIT 1;
  END IF;

  RETURN v_id;  -- NULL se não há canônico ativo na cadeia (caso legítimo: slug genuinamente novo).
END;
$$;

REVOKE ALL ON FUNCTION resolve_active_canonical_by_slug(TEXT, TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION resolve_active_canonical_by_slug(TEXT, TEXT) TO service_role, authenticated;
```

**Comportamento esperado:**

| Estado dos canonicals com slug `p_slug` | Retorno |
|---|---|
| Existe ativo/pending sem merged_into | UUID do ativo |
| Existe deprecated com merged_into → winner ativo | UUID do winner ativo |
| Existe cadeia deprecated → deprecated → winner ativo | UUID do winner ativo final |
| Existe deprecated com cadeia que termina em outro deprecated sem merged_into (orfão) | NULL — caller deve criar novo canonical |
| Slug genuinamente novo (zero matches) | NULL — caller cria novo canonical |
| Cadeia com ciclo (proteção depth=10) | NULL — caso patológico, log via `RAISE NOTICE` recomendado em produção |

Migration carregada via Anexo A com nome `42b_resolve_active_canonical_by_slug.sql`. Caller skill em §3.20; caller role é débito documentado (LK-PS-18).

## 4.5 — RPC `seed_skill_batch_from_cbo` (mig 2.42c — atomicidade do batch)

Substitui o pattern inválido de `supabase.rpc('begin_transaction')` em §3.21. RPC plpgsql única que faz INSERT em `job_canonical_skills` + INSERT em `canonical_cbo_links` + retorna mapping `slug → id` atomicamente dentro de uma transação implícita do servidor.

Pontos críticos:
- `source = 'cbo_mte_2002_seed'` (constraint check em mig 07).
- `occupation_code` por linha amarra cada skill à ocupação CBO de origem via `canonical_cbo_links`.
- `creation_confidence` é validada `BETWEEN 0 AND 1` antes do INSERT.
- `skill_type` é validada contra `('hard','soft','tool','language','certification')` antes do INSERT.
- Mapping retorna union de inseridas + reusadas via JOIN com `merged_into` resolvido (`COALESCE(merged_into, id)`) — se uma skill foi mergada entre runs do batch, mapping aponta para o winner ativo, não para o canonical deprecated.

```sql
CREATE OR REPLACE FUNCTION seed_skill_batch_from_cbo(
  p_rows JSONB,                          -- array de { label, slug, skill_type, creation_confidence, occupation_code }
  p_taxonomy_content_version_id INT
) RETURNS JSONB
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp
AS $$
DECLARE
  v_inserted INT := 0;
  v_reused INT := 0;
  v_link_inserted INT := 0;
  v_mapping JSONB := '{}'::JSONB;
  v_invalid_count INT;
BEGIN
  IF jsonb_typeof(p_rows) <> 'array' THEN
    RAISE EXCEPTION 'p_rows deve ser jsonb array';
  END IF;

  -- Validação up-front de skill_type e creation_confidence (rejeita batch inteiro se houver inválido).
  SELECT COUNT(*) INTO v_invalid_count FROM jsonb_array_elements(p_rows) AS elem
  WHERE (elem->>'skill_type') NOT IN ('hard','soft','tool','language','certification')
     OR (elem->>'creation_confidence')::NUMERIC NOT BETWEEN 0 AND 1
     OR (elem->>'occupation_code') IS NULL;
  IF v_invalid_count > 0 THEN
    RAISE EXCEPTION '% rows com skill_type, creation_confidence ou occupation_code inválido', v_invalid_count;
  END IF;

  -- INSERT em massa em job_canonical_skills, ON CONFLICT preserva primeira inserção (D-PS-50).
  WITH input AS (
    SELECT
      (elem->>'label') AS label,
      (elem->>'slug') AS slug,
      (elem->>'skill_type') AS skill_type,
      (elem->>'creation_confidence')::NUMERIC AS creation_confidence,
      (elem->>'occupation_code') AS occupation_code
    FROM jsonb_array_elements(p_rows) AS elem
  ),
  inserted AS (
    INSERT INTO job_canonical_skills (
      label, slug, skill_type, status, source,
      creation_confidence, taxonomy_content_version_id
    )
    SELECT label, slug, skill_type, 'pending', 'cbo_mte_2002_seed',
           creation_confidence, p_taxonomy_content_version_id
    FROM input
    ON CONFLICT (slug) DO NOTHING
    RETURNING id, slug
  )
  SELECT COUNT(*) INTO v_inserted FROM inserted;

  -- Mapping completo (inseridas + reusadas) com COALESCE(merged_into, id) para apontar para winner ativo.
  WITH input_slugs AS (
    SELECT (elem->>'slug') AS slug FROM jsonb_array_elements(p_rows) AS elem
  )
  SELECT jsonb_object_agg(jcs.slug, COALESCE(jcs.merged_into, jcs.id))
  INTO v_mapping
  FROM job_canonical_skills jcs
  JOIN input_slugs i ON i.slug = jcs.slug;

  -- canonical_cbo_links: liga cada skill à ocupação CBO de origem. ON CONFLICT idempotente.
  WITH link_input AS (
    SELECT
      (elem->>'occupation_code') AS occupation_code,
      (v_mapping ->> (elem->>'slug'))::UUID AS canonical_skill_id
    FROM jsonb_array_elements(p_rows) AS elem
    WHERE v_mapping ? (elem->>'slug')
  ),
  link_inserted AS (
    -- INSERT defensivo: campos source e is_primary explícitos para o caso da tabela de produção
    -- exigir esses campos (NOT NULL sem default). Antigravity valida o schema real antes de aplicar:
    -- se source/is_primary não existirem, REMOVER do INSERT (falha cedo em staging, não silent runtime).
    INSERT INTO canonical_cbo_links
      (occupation_code, canonical_skill_id, canonical_role_id, source, is_primary)
    SELECT
      occupation_code,
      canonical_skill_id,
      NULL,                       -- skill-side; canonical_role_id NULL (XOR)
      'cbo_mte_2002_seed',        -- alinhado ao source da skill (mig 38 D-PS-50)
      FALSE                       -- defensivo; CBO link de skill nunca é primary (primary é role-only)
    FROM link_input
    WHERE canonical_skill_id IS NOT NULL
    -- ON CONFLICT por nome de constraint quando índice é parcial. UNIQUE em produção é
    -- (canonical_skill_id, occupation_code) WHERE canonical_skill_id IS NOT NULL.
    -- Postgres pode falhar inferência por colunas com índice parcial — usar predicado explícito:
    ON CONFLICT (occupation_code, canonical_skill_id) WHERE canonical_skill_id IS NOT NULL DO NOTHING
    RETURNING 1
  )
  SELECT COUNT(*) INTO v_link_inserted FROM link_inserted;

  v_reused := jsonb_array_length(p_rows) - v_inserted;

  RETURN jsonb_build_object(
    'mapping', COALESCE(v_mapping, '{}'::JSONB),
    'inserted_count', v_inserted,
    'reused_count', v_reused,
    'cbo_links_inserted', v_link_inserted
  );
END;
$$;

REVOKE ALL ON FUNCTION seed_skill_batch_from_cbo(JSONB, INT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION seed_skill_batch_from_cbo(JSONB, INT) TO service_role;
```

Migration carregada via Anexo A com nome `42c_seed_skill_batch_from_cbo.sql`. Caller único em §3.21.

---

# Parte 5 — Painéis Limiares

10 painéis. Modos online (24h) e histórico (7d/30d). Endpoints `/api/admin/limiares/online` e `/api/admin/limiares/historical`. Layout grid 2 colunas em `AdminDashboard.tsx`.

## Painel 1 — Hard Gate

Skills filtradas por confidence < limiar de Hard Gate por dia.

```sql
SELECT date_trunc('day', created_at) AS day,
       COUNT(*) AS skills_filtered_below_hard_gate
FROM events
WHERE event_name = 'skill_filtered_hard_gate'
  AND created_at >= NOW() - INTERVAL '7 days'
GROUP BY 1 ORDER BY 1;
```

## Painel 2 — Promoção vs Zona Opus

JCS por banda de confidence (verde ≥ 0.85, amarelo 0.70-0.84, vermelho < 0.70).

```sql
SELECT
  CASE
    WHEN confidence_median >= 0.85 THEN 'green'
    WHEN confidence_median >= 0.70 THEN 'amber'
    ELSE 'red'
  END AS band,
  COUNT(*) AS skills
FROM job_canonical_skills
WHERE status = 'active' AND confidence_median IS NOT NULL
GROUP BY 1;
```

## Painel 3 — Merge Candidate

Pares detectados nos últimos 7d.

```sql
SELECT csmc.canonical_a_id, csmc.canonical_b_id,
       a.label AS skill_a, b.label AS skill_b, csmc.similarity, csmc.detected_at
FROM canonical_skill_merge_candidates csmc
JOIN job_canonical_skills a ON a.id = csmc.canonical_a_id
JOIN job_canonical_skills b ON b.id = csmc.canonical_b_id
WHERE csmc.resolved_at IS NULL
ORDER BY csmc.detected_at DESC LIMIT 10;
```

## Painel 4 — Gate cumulativo

Skills com `vacancy_count >= min_vacancies` E `distinct_sources_count >= min_employers` ainda em `pending`.

```sql
WITH config AS (
  SELECT (SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_vacancies') AS v,
         (SELECT value::INT FROM pipeline_config WHERE key='skill.promotion.min_distinct_employers') AS e
)
SELECT jcs.id, jcs.label, jcs.vacancy_count, jcs.distinct_sources_count, jcs.confidence_median
FROM job_canonical_skills jcs, config c
WHERE jcs.status = 'pending'
  AND jcs.vacancy_count >= c.v
  AND jcs.distinct_sources_count >= c.e
ORDER BY jcs.vacancy_count DESC LIMIT 50;
```

## Painel 5 — Pending stuck

Skills `pending` há mais de 30 dias. Sem branch arqueológico (D-PS-39, D-PS-54), itens deste painel exigem **ação admin manual**: revisar caso a caso e decidir entre (a) promoção forçada via UPDATE direto se houver justificativa, (b) marcação como `merge_candidate` para arbitragem Opus, ou (c) `retire_canonical(id, 'manual_admin', 'skill')` se obsoleta. Casos típicos que aparecem aqui: skills com volume histórico esparso fora da janela de 60d, skills nichadas pré-sprint, ou variações de label que não foram detectadas como merge_candidate.

```sql
SELECT id, label, created_at, vacancy_count, distinct_sources_count, confidence_median, latest_posted_at
FROM job_canonical_skills
WHERE status = 'pending'
  AND created_at < NOW() - INTERVAL '30 days'
ORDER BY created_at ASC LIMIT 50;
```

## Painel 6 — Aposentadoria por gap

Skills `active` cujas `latest_posted_at` é anterior ao gap configurado.

```sql
WITH config AS (
  SELECT (SELECT value::INT FROM pipeline_config WHERE key='skill.retirement.gap_days') AS gap_days
)
SELECT id, label, latest_posted_at,
       EXTRACT(DAY FROM NOW() - latest_posted_at)::INT AS gap_days
FROM job_canonical_skills, config c
WHERE status = 'active'
  AND latest_posted_at < NOW() - (c.gap_days || ' days')::INTERVAL
ORDER BY latest_posted_at ASC LIMIT 50;
```

## Painel 7 — Distribuição `confidence_median_at_promotion`

Histograma para calibrar `auto_min_confidence` baseado em comportamento real.

```sql
SELECT
  WIDTH_BUCKET(confidence_median_at_promotion, 0.0, 1.0, 20) AS bucket,
  COUNT(*) AS promotions,
  MIN(confidence_median_at_promotion) AS bucket_min,
  MAX(confidence_median_at_promotion) AS bucket_max
FROM job_canonical_skills
WHERE confidence_median_at_promotion IS NOT NULL
GROUP BY 1 ORDER BY 1;
```

## Painel 8 — Distribuição `creation_confidence`

Histograma de `creation_confidence` em JCS e JCR (filtros por status: ativo, pending, retired). Barras coloridas por banda — vermelho `< 0.70`, amarelo `[0.70, 0.85)`, verde `>= 0.85`. Boundary intencional: confidence exatamente `0.85` cai na banda verde, paritário com gate de promoção que usa `>= v_min_confidence`.

```sql
SELECT
  'skill' AS entity_type,
  status,
  WIDTH_BUCKET(creation_confidence, 0.0, 1.0, 20) AS bucket,
  CASE
    WHEN creation_confidence >= 0.85 THEN 'green'
    WHEN creation_confidence >= 0.70 THEN 'amber'
    ELSE 'red'
  END AS band,
  COUNT(*) AS canonicals
FROM job_canonical_skills
WHERE creation_confidence IS NOT NULL
GROUP BY 2, 3, 4

UNION ALL

SELECT
  'role' AS entity_type,
  status,
  WIDTH_BUCKET(creation_confidence, 0.0, 1.0, 20) AS bucket,
  CASE
    WHEN creation_confidence >= 0.85 THEN 'green'
    WHEN creation_confidence >= 0.70 THEN 'amber'
    ELSE 'red'
  END AS band,
  COUNT(*) AS canonicals
FROM job_canonical_roles
WHERE creation_confidence IS NOT NULL
GROUP BY 2, 3, 4
ORDER BY 1, 3;
```

LK-PS-01 documenta que canônicos pré-sprint têm `creation_confidence = NULL` e ficam fora deste painel.

## Painel 9 — Histórico de `pipeline_config`

Timeline lendo de `pipeline_config_history` (mig 40). Cada linha mostra `key`, `previous_value → new_value`, `changed_by`, `changed_at`, `reason`. Filtro por chave e janela de tempo. Paginação 50/página por padrão.

```sql
SELECT
  pch.key,
  pch.previous_value,
  pch.new_value,
  pch.changed_by,
  pch.changed_at,
  pch.reason,
  pc.description
FROM pipeline_config_history pch
LEFT JOIN pipeline_config pc ON pc.key = pch.key
WHERE pch.changed_at >= NOW() - $1::INTERVAL  -- janela ajustável
  AND ($2::TEXT IS NULL OR pch.key = $2)      -- filtro por chave
ORDER BY pch.changed_at DESC
LIMIT 50 OFFSET $3;
```

Painel inclui ação rápida "Reverter para `previous_value`" que abre modal de confirmação e chama RPC `set_pipeline_config_value` com payload de rollback.

## Painel 10 — Promoções vs Rejeições por dia

Stacked bar chart agregando transições de status em janela ajustável (24h, 7d, 30d). Stacks: `pending → active` (verde), `pending → rejected` (vermelho), `active → retired` (cinza), `retired → active` ressuscitação (azul, série separada).

```sql
SELECT
  date_trunc('day', created_at) AS day,
  CASE
    WHEN event_name IN ('role_promoted_dynamic','skill_promoted_dynamic')
         AND COALESCE(metadata->>'is_resurrection','false') = 'true' THEN 'resurrected'
    WHEN event_name IN ('role_promoted_dynamic','skill_promoted_dynamic') THEN 'promoted'
    WHEN event_name = 'canonical_retired' THEN 'retired'
    WHEN event_name IN ('role_rejected','skill_rejected') THEN 'rejected'
  END AS transition,
  resource_type,
  COUNT(*) AS count
FROM events
WHERE event_name IN (
    'role_promoted_dynamic', 'skill_promoted_dynamic',
    'canonical_retired', 'role_rejected', 'skill_rejected'
  )
  AND created_at >= NOW() - $1::INTERVAL
GROUP BY 1, 2, 3
ORDER BY 1;
```

A série `resurrected` é separada da `promoted` no CASE acima — sem isso, ressuscitações ficariam mascaradas dentro do verde de promoção normal e o operador admin perderia visibilidade dos retornos automáticos via catchup.

<!-- MARKER_PARTE_6 -->

---

# Parte 6 — Plano de execução

## 6.1 — Pré-requisitos

- Branch `sprint-paridade-skills` criada a partir de `main`.
- Backup full do banco via Supabase dashboard antes da etapa 1.
- Rota `/admin/dashboard` testada localmente.
- `pipeline_calibration_metrics` existente em produção.

## 6.2 — Sequência de aplicação

**Pré-requisito operacional.** Antes de aplicar mig 09, habilitar a extensão `unaccent` via Supabase Dashboard → Database → Extensions. A mig 09 contém `CREATE EXTENSION unaccent` defensivo, mas projetos sem permissão de DDL para extensões precisam de habilitação manual prévia, ou a mig aborta com mensagem explícita.

**Etapa 1.** Aplicar `01_backup_pre_execution.sql`. Validar 6 tabelas backup criadas.

**Etapa 2.** Aplicar `02_drop_is_emerging.sql` e imediatamente `02b_recreate_jcr_set_updated_at_without_is_emerging.sql`. Pause CRONs e ingestão entre 02 e 02b — qualquer write em `job_canonical_roles` no intervalo dispararia o trigger ainda referenciando `is_emerging` dropada. Validar coluna ausente + função recriada sem referência a `is_emerging`.

**Etapa 3.** Aplicar `03_lookup_skill_alias_stub.sql` (NO-OP de validação) e `04_drop_aliases_array_with_stub.sql`. A mig 04 cria stub de `lookup_canonical_skill_by_normalized_alias` retornando NULL com RAISE NOTICE; cobre janela mig 04 → mig 12 (11 migrations) onde a função real ainda não existe. Validar `aliases` ausente em JCS.

**Etapa 4.** Aplicar `05_rename_canonical_skills_to_jcs.sql`. Validar tabela renomeada e PK + 4 CHECKs renomeadas. **Crítico:** nenhum write em `job_posting_skills` ou `submitted_job_skills` pode ocorrer entre esta etapa e a 4b — pause CRONs e desabilite ingestão.

**Etapa 4b.** Aplicar `05b_recreate_jcs_dependent_functions.sql` imediatamente após 4. Reescreve 15 funções dependentes textualmente de `canonical_skills` (incluindo `fn_redirect_deprecated_skill_junction` que é trigger handler de JPS e SJS). Padrão placeholder `##CSS##` evita corromper `canonical_skills_summary` (D-PS-10). Validar zero ocorrências de `canonical_skills` isolado em bodies das 15 funções listadas.

**Etapa 5.** Aplicar `06_validate_fks_after_rename.sql`. Validar 9 FKs nominais apontam para JCS. **Ingestão e CRONs continuam pausados** — `lookup_canonical_skill_by_normalized_alias` ainda está em modo stub (retorna NULL com RAISE NOTICE) entre mig 04 e mig 12. Reativar agora causaria callers de alias lookup a receber NULL silenciosamente, criando duplicatas em vez de associar a skill existente. Ingestão só pode reativar após a Etapa 9 (mig 12 que reescreve a função real lendo `taxonomy_relations`).

**Etapa 6.** Aplicar `07_jcs_add_columns.sql` (+ 20 colunas) e `08_jcs_indexes.sql` (+ 9 índices).

**Etapa 7.** Aplicar `09_jcs_slug_backfill_not_null.sql`. Validar zero colisões + `NOT NULL` ativo.

**Etapa 8.** Aplicar `10_jcr_paridade_columns.sql` e `11_jcs_add_last_opus_review_at.sql`.

**Etapa 9.** Aplicar `12_taxonomy_relations_entity_type.sql` (pré-validação de órfãos + ADD COLUMNS + backfill + DROP UNIQUE single + ADD UNIQUE composto + CREATE INDEXES) e em seguida `12b_recreate_target_canonical_id_dependents_and_drop.sql` (3 passes: replace mecânico de `process_opus_create_new` + `process_opus_disagree` com reescrita da chamada interna a `merge_canonicals` para 4-arg + bridge textual de `merge_canonicals`/`merge_canonical_skills`/`merge_skills` + reescrita explícita de `o3_opus_canonical_label_disputes`, depois DROP COLUMN `target_canonical_id` CASCADE). Validar XOR + função `lookup_canonical_skill_by_normalized_alias` lendo `taxonomy_relations` + ausência de `target_canonical_id` em pg_proc bodies. **Ingestão e CRONs podem reativar a partir desta etapa** — função real de alias lookup está disponível.

**Etapa 10.** Aplicar `13`, `14`, `15` (layer expand + backfill + NOT NULL).

**Etapa 11.** Aplicar `16_tfc_entity_type_defensive.sql`. Validar XOR.

**Etapa 12.** Aplicar `17_smd_unique_expression.sql` (pré-clean dups + UNIQUE expression).

**Etapa 13.** Aplicar `18_canonical_skills_summary_fks.sql` (com pré-validação de órfãos).

**Etapa 14.** Aplicar `19`, `20`, `21` (rename CBO + XOR). Validar 2 UNIQUE indexes (role/skill).

**Etapa 15.** Aplicar `22_recreate_cbo_pl_functions.sql`. Validar 6 funções CBO recriadas (`process_opus_create_new`, `replace_cbo_link`, `upsert_primary_cbo_link`, `internal.reset_taxonomy_core`, `internal.reset_taxonomy_core_v2`, `fetch_cbo_candidates`). Validação interna rejeita bodies que ainda contenham `canonical_role_cbo` OU `target_canonical_id`.

**Etapa 16.** Aplicar `23_retire_canonical_extended.sql`. Validar overload de 3 args criado.

**Etapa 17.** Aplicar `24_fn_promote_skill_on_threshold.sql` e `25_fn_retire_skill_on_zero_vacancy.sql`. Smoke S1.

**Etapa 18.** Aplicar `26_fn_reset_skill_embedding_and_updated_at.sql`.

**Etapa 19.** Aplicar `27_jps_count_triggers.sql` (3 triggers ROW-level em job_posting_skills + helper `fn_jps_recompute_jcs` + par accumulate/drain STATEMENT-level em job_postings e employers) e em seguida `27b_jp_count_triggers_paridade_jcr.sql` (par accumulate/drain STATEMENT-level paritário em job_postings para JCR). Smoke S8 e S9 (performance).

**Etapa 20.** Aplicar `28_validate_fn_promote_canonical_on_threshold.sql`. NO-OP. Se validação falhar, sprint para.

**Etapa 21.** Aplicar `29_fn_update_jcr_confidence_median.sql` (3 triggers em FOI consumindo `role.confidence.lookback_days/min_count`). Smoke: INSERT manual em FOI → verificar UPDATE em `jcr.confidence_median`.

**Etapa 22.** Aplicar `30_fn_flag_needs_opus_review_residual.sql` (BEFORE UPDATE com `NEW := value` + AFTER UPDATE separado para emit_event, lendo `<entity>.opus_review.cooldown_days` e `<entity>.promotion.auto_min_confidence`). Smoke: UPDATE manual em `jcr.confidence_median` com valor < 0.85 → verificar `needs_opus_review = TRUE` na mesma tupla, sem segunda gravação.

**Etapa 23.** Aplicar `31_catchup_pending_promotions.sql` e `32_catchup_pending_skill_promotions.sql` (sem branch arqueológico, com WHERE defensivo, lendo `<entity>.promotion.lookback_days` de `pipeline_config`). Smoke S2 e S3.

**Etapa 24.** Aplicar `33_drop_merge_canonical_skill_singular.sql`, `34_canonical_skill_merge_candidates.sql`, `35_rewrite_merge_canonical_skills_and_merge_skills.sql` (reescrita explícita com AE-6 e AE-7), `36_rewrite_merge_canonicals.sql` (reescrita explícita com AE-7 e AE-9).

**Etapa 25.** Aplicar `37_hardening_search_path.sql`.

**Etapa 26.** Aplicar `38_pipeline_config_seed.sql` (24 chaves) → `39_pipeline_config_evolve.sql` → `40_pipeline_config_history.sql` → `41_pipeline_config_descriptions_seed.sql` (24 descriptions) → `42_rpc_set_pipeline_config_value.sql` → `42b_resolve_active_canonical_by_slug.sql` → `42c_seed_skill_batch_from_cbo.sql` → `43_helper_get_pipeline_config_value.sql`. As migrations `42b` e `42c` são RPCs consumidas pelo código TS da Etapa 28 — devem estar no banco antes do deploy da branch.

**Etapa 27.** Aplicar `44_detect_canonical_merge_candidates_rewrite.sql` (consume 3 chaves de pipeline_config) e `45_detect_skill_merge_candidates.sql` (paritário, mesmas 3 chaves).

**Etapa 28.** Deploy da branch TS. 33 mudanças de §3 ativas (§3.1 a §3.33). Smoke tests:
- Hard Gate filtrando skills com confidence < 0.70 (§3.1).
- 11 substituições enumeradas em opus-arbitration (§3.6) — 5 rename `canonical_skills` + 4 rename CBO + 2 chamadas `merge_canonicals` (linhas 937 e 1020) com nova assinatura 4-arg (`p_actor TEXT`, `p_actor_id UUID`).
- `last_opus_review_at` setado após Opus arbitration em todos os 7 verdicts (§3.6).
- 22 callsites externos renomeados (§3.31) — `git grep canonical_skills` em código TS retorna zero (S11).
- CRON pipeline-maintenance executando `catchup_pending_skill_promotions` sem +1/-1 (§3.12, mig 32).
- RPC `resolve_active_canonical_by_slug('skill', slug)` invocada em §3.20 (S10) — agora segue cadeia `merged_into` recursiva.
- RPC `seed_skill_batch_from_cbo(rows, version_id)` invocada em §3.21 com `occupation_code` por linha.
- Endpoint `/api/admin/merge-skills` com nomes novos de parâmetros RPC (§3.18: `p_loser_id`, `p_winner_id`, `p_decided_by_actor_id`) e consumindo `skill.merge_candidate.cosine_threshold` de pipeline_config (§3.32). Modal admin (§3.19) trata 409 (cross-type) e 422 (similarity below threshold).
- Wrapper programático `lib/taxonomy/merge-canonicals.ts` (§3.33) atualizado para nova assinatura RPC (4º arg `p_actor_id UUID` em vez de `p_decided_by TEXT`).

**Etapa 29.** Aplicar `46_drop_refresh_canonical_confidence_median.sql` e `47_drop_refresh_canonical_skills_confidence_median.sql`.

**Etapa 30.** Aplicar `48_validate_no_legacy_callers.sql`.

**Etapa 31.** Executar piloto CBO (§4.2): 50 ocupações estratificadas. Aprovação humana.

**Etapa 32.** Executar batch CBO completo (§4.3): 2694 ocupações. Tempo estimado 8-12h.

**Etapa 33.** Aplicar `49_drop_backup_schema.sql`. Sem espera.

**Etapa 34.** Aplicar `50_cleanup_legacy_backups.sql`. Drop direto das 9 tabelas legadas.

**Etapa 35.** Validação SQL pós-aplicação completa via Anexo C. Aprovação final do PO.

## 6.3 — Smoke tests críticos

**S1.** INSERT em `job_posting_skills` referenciando JCS com `vacancy_count >= 5`, `distinct_sources_count >= 2` e `confidence_median >= 0.85` → status muda de `pending` para `active`.

**S2.** UPDATE manual `job_canonical_skills SET status='retired'` em skill com volume → `catchup_pending_skill_promotions` ressuscita para `active` na próxima execução do CRON (15 min).

**S3.** UPDATE manual em `job_canonical_roles SET status='retired'` em role com volume → `catchup_pending_promotions` promove para `active`.

**S4.** Admin tenta merge cross-type sem `cross_type_confirmed` → endpoint retorna 409 com `requires_cross_type_confirmation`. Admin aprova → segundo POST com `cross_type_confirmed: true` → merge executa.

**S5.** Opus arbitration roda em canonical com `confidence_median = 0.78` → após Opus completar, `last_opus_review_at = NOW()` em jcr/jcs. Cooldown 90d efetivo no trigger residual.

**S6.** UPDATE em pipeline_config (via RPC `set_pipeline_config_value`) → row criada em `pipeline_config_history` com `previous_value`, `new_value`, `changed_by`, `reason`.

**S7.** `detect_skill_merge_candidates` executado → emite métrica em `pipeline_calibration_metrics` com `metric_name = 'skill_merge_candidate_detected'`.

**S8.** UPDATE em `job_postings.curation_status` (de `'pending'` para `'curated'`) numa vaga com 5 skills associadas → triggers STATEMENT-level (`trg_jp_accumulate_jcs` + `trg_jp_drain_jcs`) recomputam as 5 JCS afetadas em uma passagem, não em 5 chamadas redundantes. Análogo: UPDATE em `employers.is_recruitment_agency` propaga recompute via `trg_employer_accumulate_jcs` + `trg_employer_drain_jcs`. Para JCR: UPDATE em `job_postings.canonical_role_id` aciona `trg_jp_accumulate_jcr` + `trg_jp_drain_jcr` (mig 27b) — vacancy_count atualiza em uma única passagem por role distinta afetada.

**S9.** Performance pós-mig 27 e 27b: UPDATE em batch de 1000 vagas (`UPDATE job_postings SET curation_status='curated' WHERE id IN (...)`) que compartilham 50 skills comuns deve completar em < 5s. Verificação: `EXPLAIN ANALYZE` mostra trigger STATEMENT-level executando `fn_jps_drain_pending_jcs_recompute` uma única vez ao final, não 1000× por linha.

**S10.** RPC `resolve_active_canonical_by_slug('skill', 'python')` retorna ID do canonical ativo após mig 42 deployada. Sem ela, callers TS de §3.20 falhariam em runtime.

**S11.** Pós-deploy TS, `git grep -nE "from\(['\\"]canonical_skills['\\"]\)" -- 'lib/**/*.ts' 'app/**/*.ts' 'scripts/**/*.ts'` retorna zero linhas (excluindo `canonical_skills_summary` e comentários). O glob `lib/**/*.ts` cobre recursivamente todos os subdiretórios incluindo `lib/analysis/` (que tem `compare.ts` e `skill-map.ts` listados em §3.31), `lib/admin/`, `lib/api/`, `lib/pipeline/`, `lib/taxonomy/` e demais.

## 6.4 — Estratégia de recuperação

- Etapa 1 (backup completo): dump full pré-aplicação. Recovery via PITR Supabase ou restore do dump.
- Etapas 1-22 (DDL aditivo + triggers novos): drops manuais possíveis. Schema `backup_paridade_skills` ainda existe.
- Etapa 23 (drop `merge_canonical_skill` singular): restaurável via `pg_get_functiondef` capturado.
- Etapa 14 (rename CBO): rollback exige rename inverso + recompose constraint. Recomendar PITR.
- Etapa 33 (drop schema backup): ponto de não-retorno. Recovery exige PITR.
- Etapa 34 (drop backups legados): ponto de não-retorno.

---

# Parte 7 — Critérios de aceite

## Schema
- `job_canonical_skills` existe com 31+ colunas incluindo `slug NOT NULL`, `embedding VECTOR(768)`, `vacancy_count`, `distinct_sources_count`, `latest_posted_at`, `confidence_median`, `creation_confidence`, `vacancy_count_at_promotion`, `distinct_sources_count_at_promotion`, `confidence_median_at_promotion`, `retired_at`, `retire_reason`, `last_opus_review_at`, `needs_opus_review`.
- `aliases TEXT[]` ausente em JCS.
- `is_emerging` ausente em JCS e JCR.
- `taxonomy_relations` tem `entity_type` (CHECK role/skill), `target_role_id` FK CASCADE, `target_skill_id` FK CASCADE, XOR check, `layer NOT NULL` (CHECK 0/1/2/3 com XOR skill≠2). UNIQUE composto `(source_term, entity_type)` substituindo a UNIQUE legada single-column.
- `taxonomy_family_canonicals` tem `entity_type` (default `'role'`), `canonical_skill_id` FK preparatório, e CHECK restritiva que **bloqueia inserção de skills** (D-PS-03 enforcement).
- `canonical_cbo` (renomeada) e `canonical_cbo_links` (renomeada) com `canonical_skill_id` opcional + XOR + 2 UNIQUE indexes.
- `idx_jcs_slug` parcial substituído por `job_canonical_skills_slug_key` UNIQUE total após NOT NULL (mig 09).
- `skill_merge_decisions` tem `idx_smd_pair_unique` via expression `LEAST/GREATEST` (mig 17).
- 24 chaves em `pipeline_config` com `description` populada — `role.promotion.auto_min_confidence` com description honesta esclarecendo que **não é gate de promoção JCR** mas serve como teto da zona Opus por convenção (mig 41).
- `pipeline_config_history` com FORCE RLS + trigger AFTER UPDATE.
- `canonical_skill_merge_candidates` criada.
- Schema `backup_paridade_skills` ausente.
- 9 tabelas legadas ausentes em `public`.

## Funções e triggers
- `fn_promote_canonical_on_threshold` em produção tem ressuscitação `OLD.status IN ('pending','retired')`, EXCEPTION blocks, `pg_trigger_depth` guard (validado por mig 28). Sem branch arqueológico (D-PS-39, D-PS-54).
- `fn_promote_skill_on_threshold` criada com paridade total a JCR.
- `fn_retire_skill_on_zero_vacancy` criada com filtro `status IN ('active','pending','rejected')`, early return CBO, delegação a `retire_canonical(id, reason, 'skill')`.
- Assimetria entre roles e skills no guard de retire: `fn_retire_canonical_on_zero_vacancy` em produção mantém-se sem guard `pg_trigger_depth() > 1`; `fn_retire_skill_on_zero_vacancy` nasce com guard.
- `trg_jcr_set_updated_at` recriada sem referência a `is_emerging` (mig 02b).
- `fn_jps_recompute_jcs` + 3 triggers ROW-level em `job_posting_skills` + par STATEMENT-level (`trg_jp_accumulate_jcs`/`trg_jp_drain_jcs`) em `job_postings` + par STATEMENT-level (`trg_employer_accumulate_jcs`/`trg_employer_drain_jcs`) em `employers` — recompute em cadeia idempotente, O(N) por skill distinta afetada (mig 27).
- Par STATEMENT-level `trg_jp_accumulate_jcr`/`trg_jp_drain_jcr` em `job_postings` para JCR — paridade ao padrão JCS, lê role-affected de DELETE/UPDATE/INSERT (mig 27b).
- `fn_recompute_jcr_confidence_median` consome `role.confidence.lookback_days` e `role.confidence.min_count` de `pipeline_config` + 3 triggers em `function_orchestrator_items` (mig 29).
- `fn_flag_needs_opus_review_jcr` e `fn_flag_needs_opus_review_jcs` são `BEFORE UPDATE OF confidence_median` com `NEW := value` (mig 30). Triggers `_emit_event` separados em `AFTER UPDATE OF needs_opus_review` para INSERT em `events` sem race com BEFORE. Lê `<entity>.opus_review.cooldown_days` e `<entity>.promotion.auto_min_confidence` de `pipeline_config`.
- `fn_jcs_set_updated_at` monitora apenas colunas semânticas (label, slug, status, skill_type, merged_into, needs_opus_review, human_validated_at). Trigger renomeado para `z_trg_jcs_set_updated_at` para garantir execução por último entre triggers BEFORE UPDATE do mesmo evento/tabela.
- `catchup_pending_promotions` e `catchup_pending_skill_promotions` reescritos sem padrão `+1/-1`, com WHERE defensivo `AND status IN ('pending','retired')`, GET DIAGNOSTICS para skip silencioso, leitura de `<entity>.promotion.lookback_days` de `pipeline_config`. Retorno preserva `archaeological=0` por compat caller TS, com novo campo `resurrected` (mig 31, mig 32).
- `merge_canonicals` reescrita explicitamente com AE-7 (loser status guard) + AE-9 (NOT EXISTS com `entity_type` filter em `taxonomy_relations`) + `target_role_id` direto (mig 36).
- `merge_canonical_skills` (admin UI, 5 args) reescrita explicitamente com AE-7 (loser status guard), `target_skill_id` em `taxonomy_relations` (mig 35).
- `merge_skills` (pipeline, 4 args) reescrita explicitamente com AE-7 + AE-6 (sem UPDATEs em `analysis_skill_matches.canonical_skill_id` e `.matched_via_similar_skill_id` — snapshot puro) (mig 35).
- `process_opus_create_new`, `process_opus_disagree`, `o3_opus_canonical_label_disputes` reescritas para consumir `taxonomy_relations.target_role_id` (mig 12b). `o3_opus_canonical_label_disputes` ganhou `AND tr.entity_type = 'role'` no WHERE.
- `merge_canonical_skill` (singular, hard-delete) ausente.
- `retire_canonical(uuid, text, text)` overloading criado preservando assinatura legada.
- `lookup_canonical_skill_by_normalized_alias` lê `taxonomy_relations` com `entity_type='skill'` e parâmetro `p_normalized` (preservado para compatibilidade com 3 callers TS). Stub temporário em mig 04 cobre janela mig 04 → mig 12.
- `set_pipeline_config_value` e `get_pipeline_config_value` criadas.
- `resolve_active_canonical_by_slug(p_entity_type, p_slug)` criada com discriminador para servir aos dois lados.
- `seed_skill_batch_from_cbo(p_rows, p_taxonomy_content_version_id)` criada para atomicidade do batch CBO com `source='cbo_mte_2002_seed'`, validação de `skill_type` e `creation_confidence`, INSERT em `canonical_cbo_links` por linha, mapping resolvendo `COALESCE(merged_into, id)` (Ge-Cri-1 captura).
- `fetch_cbo_candidates`, `internal.reset_taxonomy_core` e `internal.reset_taxonomy_core_v2` recriadas via `pg_get_functiondef + replace` em mig 22.
- 15 funções dependentes de `canonical_skills` recriadas em mig 05b com placeholder `##CSS##` para preservar referências a `canonical_skills_summary` (D-PS-10).
- `detect_canonical_merge_candidates` reescrita consumindo `role.merge_candidate.cosine_threshold/lookback_days/opus_review_cooldown_days` de `pipeline_config` (mig 44).
- `detect_skill_merge_candidates` criada paritária consumindo as 3 chaves equivalentes de skill (mig 45).
- `refresh_canonical_confidence_median` e `refresh_canonical_skills_confidence_median` ausentes.
- `SET search_path` presente em `fn_update_canonical_vacancy_count`, `fn_update_employer_vacancy_count`, `fn_retire_canonical_on_zero_vacancy`.

## TypeScript
- 33 mudanças aplicadas (§3.1 a §3.33).
- `lib/pipeline/persist-curation.ts` lê `skill.hard_gate.min_confidence` via `getConfigValue`.
- `lib/pipeline/skill-resolver.ts` e `lib/pipeline/role-resolver.ts` leem chaves de `pipeline_config` em vez de hardcodes.
- `lib/pipeline/taxonomy-cache.ts` segregado por `entity_type` com TTL 900s.
- `lib/pipeline/opus-arbitration.ts` com 11 substituições enumeradas (5 `canonical_skills` + 4 CBO + 2 chamadas `merge_canonicals` com nova assinatura 4-arg) e helper `finalizeOpusArbitration` aplicado em todos os 7 success paths (KEEP, REFINE, APPROVE, MERGE, INACTIVATE, REJECT, CREATE_NEW) — invalidate cache + set `last_opus_review_at` (D-PS-48).
- `lib/pipeline/ingest-job-and-discover-skills.ts` (novo) usa RPC `resolve_active_canonical_by_slug` antes de INSERT em JCS (slug collision pós-soft-delete).
- `lib/pipeline/bulk-skill-enrichment.ts` (novo) usa RPC `seed_skill_batch_from_cbo` em vez de `begin_transaction`.
- `app/api/cron/pipeline-maintenance/route.ts` invoca `catchup_pending_skill_promotions` e `detect_skill_merge_candidates`.
- `app/api/admin/merge-skills/route.ts` aceita `cross_type_confirmed: boolean` E lê `skill.merge_candidate.cosine_threshold` de `pipeline_config` (§3.32). Chamada RPC usa nomes novos (`p_loser_id`, `p_winner_id`, `p_decided_by_actor_id`) coordenados com mig 35 (§3.18).
- `lib/admin/components/MergeSkillsModal.tsx` renderizado em `app/admin/merge-skills/page.tsx` — exibe modal de confirmação cross-type quando 409 retornado e modal informativo quando 422 (similarity abaixo do threshold) retornado (§3.19).
- `lib/taxonomy/merge-canonicals.ts` (wrapper programático) atualizado para nova assinatura RPC: 4º arg muda de `p_reason TEXT` para `p_actor TEXT + p_actor_id UUID` (§3.33). Função wrapper exporta novo parâmetro opcional `actorId` com default `SYSTEM_USER_ID`.
- 22 callsites externos de `'canonical_skills'` em `.from()/.table()` renomeados para `'job_canonical_skills'` (§3.31). `git grep` retorna zero matches em código de produção (S11).
- CRON `update-canonical-stats` renomeado via `git mv` para `refresh-flags-and-retire-stale` preservando histórico Git (§3.30).
- Sub-aba "Limiares" no Admin Dashboard com 10 painéis funcionais.
- Endpoints `/api/admin/limiares/online` e `/api/admin/limiares/historical` distintos.
- Refactor JOIN em `compare.ts` e `SkillMap.tsx`.

## Comportamento runtime
- Promoção JCS automática quando thresholds atingidos via `catchup_pending_skill_promotions` no CRON 15min — sem `+1/-1` em vacancy_count.
- Promoção JCR ressuscitando roles `retired` via catchup — sem `+1/-1`, com auto_assign_family idempotente. Sem branch arqueológico (D-PS-39, D-PS-54).
- Aposentadoria JCS automática quando `vacancy_count` zera (não-CBO).
- Trigger residual flagging consolida `needs_opus_review = TRUE` na mesma tupla via BEFORE UPDATE com `NEW := value` quando `confidence_median < auto_min_confidence` e cooldown cumprido — uma única gravação física (Ge-Cri-3).
- `last_opus_review_at` atualizado pelo TS após cada Opus arbitration via helper `finalizeOpusArbitration`.
- `pipeline_config_history` recebe rows automáticos via trigger.
- `pipeline_calibration_metrics` recebe rows com `metric_name = 'skill_merge_candidate_detected'` quando `detect_skill_merge_candidates` executa.
- Admin merge cross-type retorna 409 + payload estruturado, frontend exibe modal de confirmação.
- Triggers STATEMENT-level (`trg_jp_drain_jcs`, `trg_jp_drain_jcr`, `trg_employer_drain_jcs`) recomputam agregados em uma única passagem por skill/role distinta afetada — performance O(N) verificada via S9.

## Validação SQL pós-aplicação
- Mig 05b validation passa sem exceções (15 funções sem `canonical_skills` isolado, sem condição que mascararia bodies com `canonical_skills_summary`).
- Mig 22 validation passa (6 funções CBO sem `canonical_role_cbo` E sem `target_canonical_id`).
- Mig 17 validation passa (`merge_canonical_skills` com `LEAST/GREATEST` no ON CONFLICT).
- Mig 16 validation passa (zero rows skill em `taxonomy_family_canonicals`).
- Mig 48 (validate_no_legacy_callers) passa sem exceções.
- Mig 49 (drop schema backup) confirma schema ausente.
- Mig 50 (cleanup legados) confirma 9 tabelas ausentes.
- Anexo D check 16 confirma `target_canonical_id` ausente em colunas e em pg_proc bodies.
- Anexo D check 17 confirma `catchup_*` sem branch arqueológico.

## Operacional
- Piloto CBO de 50 ocupações revisado e aprovado.
- Batch CBO completo executado.
- Embeddings JCS gerados via Gemini 768d para skills com vacancy_count > 0.
- Smoke tests S1-S11 passam.

---

# Parte 8 — Anexos

## Anexo A — Lista canônica das 56 migrations

| # | Arquivo |
|---|---|
| 01 | `01_backup_pre_execution.sql` |
| 02 | `02_drop_is_emerging.sql` |
| 02b | `02b_recreate_jcr_set_updated_at_without_is_emerging.sql` |
| 03 | `03_lookup_skill_alias_stub.sql` |
| 04 | `04_drop_aliases_array_with_stub.sql` |
| 05 | `05_rename_canonical_skills_to_jcs.sql` |
| 05b | `05b_recreate_jcs_dependent_functions.sql` |
| 06 | `06_validate_fks_after_rename.sql` |
| 07 | `07_jcs_add_columns.sql` |
| 08 | `08_jcs_indexes.sql` |
| 09 | `09_jcs_slug_backfill_not_null.sql` |
| 10 | `10_jcr_paridade_columns.sql` |
| 11 | `11_jcs_add_last_opus_review_at.sql` |
| 12 | `12_taxonomy_relations_entity_type.sql` |
| 12b | `12b_recreate_target_canonical_id_dependents_and_drop.sql` |
| 13 | `13_taxonomy_relations_layer_expand.sql` |
| 14 | `14_taxonomy_relations_layer_backfill.sql` |
| 15 | `15_taxonomy_relations_layer_not_null.sql` |
| 16 | `16_tfc_entity_type_defensive.sql` |
| 17 | `17_smd_unique_expression.sql` |
| 18 | `18_canonical_skills_summary_fks.sql` |
| 19 | `19_rename_canonical_role_cbo.sql` |
| 20 | `20_rename_canonical_role_cbo_links.sql` |
| 21 | `21_canonical_cbo_links_xor.sql` |
| 22 | `22_recreate_cbo_pl_functions.sql` |
| 23 | `23_retire_canonical_extended.sql` |
| 24 | `24_fn_promote_skill_on_threshold.sql` |
| 25 | `25_fn_retire_skill_on_zero_vacancy.sql` |
| 26 | `26_fn_reset_skill_embedding_and_updated_at.sql` |
| 27 | `27_jps_count_triggers.sql` |
| 27b | `27b_jp_count_triggers_paridade_jcr.sql` |
| 28 | `28_validate_fn_promote_canonical_on_threshold.sql` |
| 29 | `29_fn_update_jcr_confidence_median.sql` |
| 30 | `30_fn_flag_needs_opus_review_residual.sql` |
| 31 | `31_catchup_pending_promotions.sql` |
| 32 | `32_catchup_pending_skill_promotions.sql` |
| 33 | `33_drop_merge_canonical_skill_singular.sql` |
| 34 | `34_canonical_skill_merge_candidates.sql` |
| 35 | `35_rewrite_merge_canonical_skills_and_merge_skills.sql` |
| 36 | `36_rewrite_merge_canonicals.sql` |
| 37 | `37_hardening_search_path.sql` |
| 38 | `38_pipeline_config_seed.sql` |
| 39 | `39_pipeline_config_evolve.sql` |
| 40 | `40_pipeline_config_history.sql` |
| 41 | `41_pipeline_config_descriptions_seed.sql` |
| 42 | `42_rpc_set_pipeline_config_value.sql` |
| 42b | `42b_resolve_active_canonical_by_slug.sql` |
| 42c | `42c_seed_skill_batch_from_cbo.sql` |
| 43 | `43_helper_get_pipeline_config_value.sql` |
| 44 | `44_detect_canonical_merge_candidates_rewrite.sql` |
| 45 | `45_detect_skill_merge_candidates.sql` |
| 46 | `46_drop_refresh_canonical_confidence_median.sql` |
| 47 | `47_drop_refresh_canonical_skills_confidence_median.sql` |
| 48 | `48_validate_no_legacy_callers.sql` |
| 49 | `49_drop_backup_schema.sql` |
| 50 | `50_cleanup_legacy_backups.sql` |

**Notas operacionais sobre a ordem:**

- Total: 56 migrations (50 core + 6 com sufixo: `02b`, `05b`, `12b`, `27b`, `42b`, `42c`).
- `02b` recria `trg_jcr_set_updated_at` sem `is_emerging` — deve rodar imediatamente após `02`, sem writes em JCR no intervalo.
- `05b` cravado imediatamente após `05` (rename) e antes de qualquer write em `job_posting_skills` ou `submitted_job_skills` — sem isso `fn_redirect_deprecated_skill_junction` (trigger handler) quebra com referência a tabela inexistente.
- `12b` reescreve `process_opus_create_new`, `process_opus_disagree`, `o3_opus_canonical_label_disputes` antes do DROP COLUMN `target_canonical_id`. Sem essa ordem, `pg_get_functiondef` em mig 22 retornaria body com coluna inválida.
- `27b` é a contraparte JCR do refactor STATEMENT-level introduzido em `27`.
- `35` e `36` são reescritas explícitas (não validações NO-OP) — `merge_canonical_skills`/`merge_skills` em mig 35 (com AE-6 e AE-7), `merge_canonicals` em mig 36 (com AE-7 e AE-9).
- `42b` e `42c` são RPCs novas referenciadas por código TS da sprint (§3.20 e §3.21).
- `44` é reescrita (consume 3 chaves de `pipeline_config`) e não validação NO-OP.

## Anexo B — Snapshot vs operacional (mapa por tabela e coluna)

| Tabela | Coluna | Categoria | Comportamento em merge |
|---|---|---|---|
| `analysis_skill_matches` | `canonical_skill_id` | Snapshot puro | Não tocar (D-PS-58, AE-6 — `merge_skills` não atualiza esta coluna) |
| `analysis_skill_matches` | `matched_via_similar_skill_id` | Snapshot puro | Não tocar (D-PS-58, AE-6) |
| `skill_enrichment_stats` | `canonical_skill_id` | Snapshot puro | Não tocar — análise histórica de runs preserva canonical original do momento da execução |
| `skill_enrichment_stats` | `canonical_role_id` | Snapshot puro | Não tocar |
| `curation_batch_metrics` | `canonical_role_id` | Snapshot puro | Não tocar — métricas históricas de batch preservam canonical original |
| `resume_role_suggestions` | `canonical_role_id` | Snapshot puro | Não tocar — UI lateral lida com label de canonical deprecated em hover/badge |
| `resume_skill_enrichments` | `canonical_role_id` | Híbrido por coluna | Atualiza em `merge_canonicals` |
| `resume_skill_enrichments` | `canonical_skill_id` | Híbrido por coluna | Atualiza em `merge_skills` (pipeline) E `merge_canonical_skills` (admin) com NOT EXISTS guard + DELETE residual — D-PS-32 |
| `rapidapi_usage_logs` | `canonical_role_id` | Snapshot puro | Não tocar — log de chamadas externas é registro imutável; eventual SET NULL apenas se constraint exigir |
| `canonical_skills_summary` | `canonical_skill_id` | Operacional | Atualiza em merges + DELETE residual |
| `canonical_skills_summary` | `canonical_role_id` | Operacional | Atualiza em merges + DELETE residual |
| `job_postings` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 1 |
| `job_canonical_role_sources` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 2 com mesclagem temporal LEAST/GREATEST de `first_seen_at`/`last_seen_at` (colunas existem nesta tabela) |
| `function_orchestrator_items` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 1 |
| `job_no_postings` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 1 |
| `job_posting_skills` | `canonical_skill_id` | Operacional | Atualiza em `merge_skills` FASE 2 com NOT EXISTS guard |
| `submitted_job_skills` | `canonical_skill_id` | Operacional | Atualiza em `merge_skills` FASE 2 com NOT EXISTS guard |
| `canonical_role_domain_links` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 3 com NOT EXISTS + is_primary handling |
| `canonical_cbo_links` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 3 com NOT EXISTS |
| `canonical_cbo_links` | `canonical_skill_id` | Operacional | Atualiza em `merge_skills` FASE 2 com NOT EXISTS |
| `taxonomy_relations` | `target_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 3 com NOT EXISTS + DELETE residual + filtro `entity_type='role'` (AE-9) |
| `taxonomy_relations` | `target_skill_id` | Operacional | Atualiza em `merge_skills` FASE 2 com NOT EXISTS + DELETE residual + filtro `entity_type='skill'` |
| `taxonomy_family_canonicals` | `canonical_role_id` | Operacional | Atualiza em `merge_canonicals` FASE 3 com NOT EXISTS + DELETE residual |
| `taxonomy_family_canonicals` | `canonical_skill_id` | Operacional | D-PS-03 impede em prática; defensivo se reverter |

## Anexo C — Estratificação CBO para piloto

Piloto de 50 ocupações (§4.2). Distribuição estratificada por código CBO de 2 dígitos para garantir cobertura representativa antes do batch full:

| Faixa CBO | Descrição | Ocupações no piloto |
|---|---|---|
| 01-04 | Forças Armadas, Policiais e Bombeiros Militares | 2 |
| 11-14 | Membros do Poder Público, Dirigentes e Gerentes | 5 |
| 20-25 | Profissionais das Ciências e Artes | 12 |
| 30-35 | Técnicos de Nível Médio | 10 |
| 40-42 | Trabalhadores de Serviços Administrativos | 8 |
| 51-54 | Trabalhadores dos Serviços, Vendedores | 7 |
| 71-78 | Trabalhadores da Produção de Bens e Serviços Industriais | 4 |
| 9X | Trabalhadores em Serviços de Reparação e Manutenção | 2 |
| **Total** | | **50** |

Critério de seleção dentro de cada faixa: ocupações com `summary_description` populada e tamanho ≥ 100 caracteres, priorizando variedade de famílias dentro da faixa.

Output do piloto em `pilots/cbo-skills-pilot-<timestamp>.json` para revisão humana antes da aprovação do batch full (§4.3).

## Anexo D — Checklist de validação SQL pós-aplicação

Roda após mig 50. Output esperado: zero exceções.

```sql
-- 1. Schema JCS completo
SELECT COUNT(*) FROM information_schema.columns WHERE table_name='job_canonical_skills'; -- esperado: ≥31

-- 2. Aliases ausente
SELECT 1 FROM information_schema.columns
WHERE table_name='job_canonical_skills' AND column_name='aliases'; -- esperado: 0 rows

-- 3. is_emerging ausente em ambas
SELECT table_name FROM information_schema.columns
WHERE column_name='is_emerging'
  AND table_name IN ('job_canonical_skills','job_canonical_roles'); -- esperado: 0 rows

-- 4. Triggers críticos JCS presentes
SELECT tgname FROM pg_trigger WHERE tgname IN (
  'trg_promote_skill_on_threshold','trg_retire_skill_on_zero_vacancy',
  'trg_jps_insert_jcs_counts','trg_jps_update_jcs_counts','trg_jps_delete_jcs_counts',
  'trg_flag_needs_opus_review_jcs','trg_flag_needs_opus_review_jcs_emit',
  'z_trg_jcs_set_updated_at','trg_reset_skill_embedding',
  'trg_jp_accumulate_jcs','trg_jp_drain_jcs',
  'trg_employer_accumulate_jcs','trg_employer_drain_jcs'
); -- esperado: 13 rows

-- 5. Triggers críticos JCR
SELECT tgname FROM pg_trigger WHERE tgname IN (
  'trg_promote_on_threshold','z_trg_retire_canonical_on_zero_vacancy',
  'trg_foi_jcr_confidence_insert','trg_foi_jcr_confidence_update','trg_foi_jcr_confidence_delete',
  'trg_flag_needs_opus_review_jcr','trg_flag_needs_opus_review_jcr_emit',
  'trg_jp_accumulate_jcr','trg_jp_drain_jcr'
); -- esperado: 9 rows

-- 6. Funções legadas removidas
SELECT proname FROM pg_proc WHERE proname IN (
  'refresh_canonical_confidence_median','refresh_canonical_skills_confidence_median',
  'merge_canonical_skill'
); -- esperado: 0 rows

-- 7. catchup_pending_promotions itera retired e WHERE defensivo
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='catchup_pending_promotions'
  AND pg_get_functiondef(oid) LIKE E'%status IN (\'pending\', \'retired\')%'
  AND pg_get_functiondef(oid) LIKE '%GET DIAGNOSTICS%'; -- esperado: 1 row

-- 8. catchup_pending_skill_promotions criada com retorno preservando archaeological=0
SELECT 1 FROM pg_proc WHERE proname='catchup_pending_skill_promotions'
  AND pg_get_functiondef(oid) LIKE '%''archaeological'', archaeological_count%'; -- esperado: 1 row

-- 9. retire_canonical overloading (2 args + 3 args)
SELECT pronargs FROM pg_proc WHERE proname='retire_canonical' ORDER BY pronargs;
-- esperado: 2 rows com pronargs=2 e pronargs=3

-- 10. 24 chaves config seedadas com description
SELECT COUNT(*) FROM pipeline_config WHERE (key LIKE 'skill.%' OR key LIKE 'role.%')
  AND description IS NOT NULL AND description != ''; -- esperado: 24

-- 10b. Chaves antigas opus_review.confidence_lower/upper ausentes
SELECT COUNT(*) FROM pipeline_config
WHERE key IN (
  'skill.opus_review.confidence_lower','skill.opus_review.confidence_upper',
  'role.opus_review.confidence_lower','role.opus_review.confidence_upper'
); -- esperado: 0

-- 11. pipeline_config_history com FORCE RLS
SELECT relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname='pipeline_config_history';
-- esperado: relrowsecurity=true, relforcerowsecurity=true

-- 12. Schema backup paridade-skills ausente
SELECT 1 FROM information_schema.schemata WHERE schema_name='backup_paridade_skills';
-- esperado: 0 rows

-- 13. 9 tabelas backup legadas ausentes
SELECT table_name FROM information_schema.tables
WHERE table_schema='public' AND (
  table_name LIKE '%_backup_%' OR table_name LIKE '%_backup_pre_%'
); -- esperado: 0 rows

-- 14. fn_promote_canonical_on_threshold tem ressuscitação
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='fn_promote_canonical_on_threshold'
  AND pg_get_functiondef(oid) LIKE E'%OLD.status IN (\'pending\', \'retired\')%';
-- esperado: 1 row

-- 15. last_opus_review_at sendo populado pelo TS (smoke após Opus arbitration)
SELECT last_opus_review_at FROM job_canonical_roles
WHERE last_opus_review_at IS NOT NULL ORDER BY last_opus_review_at DESC LIMIT 5;
-- esperado: > 0 rows pós-deploy

-- 16. taxonomy_relations sem coluna target_canonical_id e nenhuma função referenciando
SELECT 1 FROM information_schema.columns
WHERE table_name='taxonomy_relations' AND column_name='target_canonical_id';
-- esperado: 0 rows

SELECT proname FROM pg_proc p JOIN pg_namespace n ON n.oid=p.pronamespace
WHERE n.nspname IN ('public','internal') AND p.prosrc LIKE '%target_canonical_id%';
-- esperado: 0 rows

-- 17. Triggers de promoção JCR e JCS sem branch arqueológico
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='catchup_pending_promotions';
-- esperado: body sem 'role_promotion_deferred_archaeological' e sem v_min_vacancies_archaeological

SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='catchup_pending_skill_promotions';
-- esperado: body sem 'skill_promotion_deferred_archaeological' e sem v_min_vacancies_archaeological

-- 18. trg_jcr_set_updated_at não monitora confidence_median (LK-PS-19 resolvido em mig 02b)
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='trg_jcr_set_updated_at';
-- esperado: body sem 'NEW.confidence_median IS DISTINCT FROM OLD.confidence_median'

-- 19. merge_skills e merge_canonical_skills atualizam resume_skill_enrichments (D-PS-32)
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_skills'
  AND pg_get_functiondef(oid) LIKE '%UPDATE resume_skill_enrichments%';
-- esperado: 1 row

SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonical_skills'
  AND pg_get_functiondef(oid) LIKE '%UPDATE resume_skill_enrichments%';
-- esperado: 1 row

-- 20. merge_canonical_skills usa NOT EXISTS guards em job_posting_skills, submitted_job_skills,
-- canonical_skills_summary (paridade com merge_skills)
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonical_skills'
  AND pg_get_functiondef(oid) LIKE '%NOT EXISTS%';
-- esperado: 1 row

-- 21. skill_merge_decisions INSERT usa convenção source_id/target_id (alinhado a mig 17)
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonical_skills'
  AND pg_get_functiondef(oid) LIKE '%skill_merge_decisions (source_id, target_id%';
-- esperado: 1 row

-- 22. merge_canonicals FASE 2 usa padrão 3 etapas (UPDATE temporal merge JOIN +
-- UPDATE NOT EXISTS + DELETE residual) — UNIQUE-safe em job_canonical_role_sources
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonicals'
  AND pg_get_functiondef(oid) LIKE '%winner.canonical_role_id = p_winner_id%loser.canonical_role_id%loser.employer_id%';
-- esperado: 1 row (confirma temporal merge JOIN da etapa 2a)

-- 23. merge_canonicals NOT EXISTS guard de canonical_role_domain_links usa coluna correta domain_id
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonicals'
  AND pg_get_functiondef(oid) NOT LIKE '%canonical_role_domain_id%';
-- esperado: 1 row (ausência da coluna inexistente)

-- 24. merge_canonicals chama mark_users_for_label_change_notification com cutoff text válido
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonicals'
  AND pg_get_functiondef(oid) LIKE '%mark_users_for_label_change_notification%(NOW() - INTERVAL%';
-- esperado: 1 row (BLK-V9-A resolvido — não passa UUID como cutoff)

-- 25. merge_canonical_skills INSERT em skill_merge_decisions usa coluna correta actor_id
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonical_skills'
  AND pg_get_functiondef(oid) LIKE '%skill_merge_decisions (source_id, target_id, actor_id%';
-- esperado: 1 row (MED-V9-A resolvido — não usa decided_by_actor_id inexistente)

-- 26. fn_jps_recompute_jcs lê pipeline_config (D-PS-45) — sem hardcodes 120/3
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='fn_jps_recompute_jcs'
  AND pg_get_functiondef(oid) LIKE '%skill.confidence.lookback_days%'
  AND pg_get_functiondef(oid) LIKE '%skill.confidence.min_count%';
-- esperado: 1 row

-- 27. fn_jps_recompute_jcs filtra curated em vacancy_count e confidence_median (paridade semântica)
-- vacancy_count agora exige curation_status='curated' nos JOIN/WHERE
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='fn_jps_recompute_jcs'
  AND pg_get_functiondef(oid) NOT LIKE '%SELECT COUNT(*) FROM job_posting_skills WHERE canonical_skill_id = jcs.id%';
-- esperado: 1 row (a query antiga sem JOIN para job_postings + filtro curated foi removida)

-- 28. merge_skills e merge_canonical_skills atualizam canonical_cbo_links (paridade Anexo B)
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_skills'
  AND pg_get_functiondef(oid) LIKE '%UPDATE canonical_cbo_links SET canonical_skill_id%';
-- esperado: 1 row

SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='merge_canonical_skills'
  AND pg_get_functiondef(oid) LIKE '%UPDATE canonical_cbo_links SET canonical_skill_id%';
-- esperado: 1 row
```

---

**Fim da especificação.**

