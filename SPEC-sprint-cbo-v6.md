# SPEC-sprint-cbo-v6

**Versão:** 6.0
**Data:** 06/05/2026
**Autor:** Onsly (PO) e Claude (arquiteto)
**Implementação:** Antigravity (TS) e Claude Code (SQL)
**Origem:** consolidação da v5 + triagem da rodada de validação externa (8 revisores: DeepSeek, GenSpark, Gemini, ChatGPT, Grok, Manus, Claude Code da seção principal, Claude Code da seção do PO). 38 itens analisados: 25 acatados, 10 rejeitados com justificativa, 3 duplicatas. Achado mais crítico: §2.7 importou body da v16 spec text (que tinha bug semântico: NEW assignments em trigger AFTER) em vez do body da v16 deployada (que tem pattern UPDATE explícito + `pg_trigger_depth`). Corrigido em v6 §2.7 preservando v16 deployada exatamente, estendendo apenas enum para suportar `'retired'` na ressuscitação. Outros bloqueadores corrigidos: drop function sem cascade dos triggers reais em produção (V5-B2), regressão da v5.6 sem `DELETE FROM opus_arbitration_outcomes` no reset, validação de drift APÓS mutar estado em `process_opus_create_new`, `.maybeSingle()` em estrutura N:M, evento falso em `retire_canonical` para placeholder CBO, `INSERT` sem `ON CONFLICT (slug)` em `process_opus_create_new`, `is_primary` não normalizado em `replace_cbo_link`.

---

# Changelog v5 → v6

## 🚨 Bloqueadores corrigidos

| Item | Origem | Correção |
|---|---|---|
| V5-B1 / ChatGPT / Claude Code | §2.7 trigger AFTER modificando NEW silenciosamente | §2.7 reescrito sobre body real da v16 deployada (`schema.sql:3473-3567`): AFTER UPDATE + `pg_trigger_depth() > 1` + UPDATE explícito separado. Estendido apenas para `OLD.status IN ('pending', 'retired')` |
| V5-B2 / Claude Code | §2.9 DROP FUNCTION sem dropar triggers reais (`z_trg_cleanup_zumbi_after_*` em `job_postings`) | §2.9 ganha 3 DROP TRIGGER defensivos antes do DROP FUNCTION |
| V5-B3 / ChatGPT do PO | §2.10 perdeu `DELETE FROM opus_arbitration_outcomes` (regressão da v5.6 aprovada) | §2.10 ganha `DELETE FROM opus_arbitration_outcomes` antes do DELETE PCM |
| V5-B4 / ChatGPT | §2.11 `process_opus_create_new` valida drift APÓS mutar estado | §2.11 reordenado: `SELECT ... FOR UPDATE` + validação + early return em drift, depois mutações |
| V5-B5 / ChatGPT | §3.2 `.maybeSingle()` em estrutura N:M pode falhar com múltiplas linhas | §3.2 trocado por `.select()` lista + matriz de comportamento (zero/um/vários links) |
| V5-B6 / ChatGPT | §2.8 `retire_canonical` insere event mesmo se UPDATE afeta 0 linhas (placeholder CBO) | §2.8 ganha SELECT inicial com source + early return antes do INSERT do event |
| V5-B7 / DeepSeek | §2.11 `INSERT` em `job_canonical_roles` sem `ON CONFLICT (slug)` capota com Unique Violation | §2.11 ganha tratamento `ON CONFLICT (slug)`: ressuscita se `'retired'`, sufixa se `'rejected'` |
| V5-B8 / ChatGPT | §2.12 `replace_cbo_link` insere `is_primary=true` sem normalizar primary existente | §2.12 ganha `UPDATE ... SET is_primary=FALSE` antes do INSERT + `ON CONFLICT DO UPDATE` |

## 🟠 Alto impacto

| Item | Origem | Correção |
|---|---|---|
| V5-H1 / DeepSeek | §3.7 `detectStaleCanonicals` sem ORDER BY → starvation por CTID | §3.7 ganha `.order('updated_at', { ascending: true })` |
| V5-H2 / DeepSeek | §2.12 race condition via anti-pattern `SELECT EXISTS` + DELETE + INSERT | §2.12 trocado por `DELETE ... ; IF NOT FOUND THEN RAISE` |
| V5-H3 / DeepSeek | §2.11 RPCs pgvector com WHERE estrito antes do LIMIT podem cair em Sequential Scan | §2.11 reescrito com CTE: TOP-K via ORDER BY+LIMIT, depois filtro threshold |
| V5-H4 / Gemini | §2.10 `EXECUTE 'TRUNCATE role_merge_decisions'` em PL/pgSQL com SECURITY DEFINER | §2.10 trocado por `DELETE FROM role_merge_decisions` |
| V5-H5 / Gemini | `persistCalibrationMetric` referenciada mas não definida na spec | §3.13 nova: implementação dedicada do helper |

## 🟡 Médio impacto

| Item | Origem | Correção |
|---|---|---|
| V5-M1 / Gemini | §2.10 mistura `EXECUTE 'DELETE'` com DELETE direto (antipattern de leitura) | §2.10 normalizado: todos viram DELETE direto |
| V5-M2 / ChatGPT | `ADD CONSTRAINT` sem `DROP CONSTRAINT IF EXISTS` quebra idempotência | §2.4 e §2.14 ganham DROP CONSTRAINT IF EXISTS antes de ADD CONSTRAINT |
| V5-M3 / ChatGPT | §3.5 promete tiebreak alfabético mas código só ordena por vacancyCount | §3.5 ganha `.localeCompare(b)` no `.sort` |
| V5-M4 / ChatGPT | `detectStaleCanonicals` race entre MAX(posted_at) e retire_canonical | §2.8 ganha cláusula `NOT EXISTS` revalidando condição no UPDATE |
| V5-M5 / ChatGPT | `process_opus_create_new` não valida source do loser (pode depreciar placeholder CBO) | §2.11 ganha guard contra source='cbo_mte_2002_seed' |
| V5-M6 / Gemini | Sugestão `suggested_cbo` descartada silenciosamente quando candidatos vazios | §3.2 ganha registro em PCM via `opus_suggested_cbo_no_candidate` |
| V5-M7 / Claude Code | Trigger novo sem prefixo `z_` viola convenção alfabética com triggers de recompute | §2.9 renomeado para `z_trg_retire_canonical_on_zero_vacancy` |

## 🟢 Polishing

| Item | Origem | Correção |
|---|---|---|
| V5-P1 / DeepSeek/GenSpark/Grok | Drift entre constante 0.85 em RPC default e código TS | §3.3 ganha constante exportada `FAMILY_COSINE_THRESHOLD` passada explicitamente |
| V5-P2 / GenSpark | Sem validação pós-reset de embeddings em famílias legadas | Etapa CBO.4 ganha query verificando famílias active sem embedding |
| V5-P3 / Grok | Comentários reforçados em §3.4 nos filtros Camada 2 | §3.4 ganha comentário explicativo SQL vs in-memory |
| V5-P4 / ChatGPT | `bulk_curation_progress` citado em CBO.0a-bis sem migration explícita | Etapa CBO.0a-bis ganha verificação prévia da tabela com criação se ausente |
| V5-P5 / ChatGPT | Descartes em §4.4 (sinônimos colidentes) sem registro auditável | §4.4 ganha `persistCalibrationMetric` para `cbo_synonym_collision_dropped` |

## ❌ Rejeitados (com justificativa)

| Item | Origem | Justificativa |
|---|---|---|
| Manus: deadlock em soft delete via trigger AFTER em massa | Manus | RPC faz UPDATE em `WHERE id = p_canonical_id` (lock por linha, não por tabela) + INSERT em events append-only. Lotes de 10-100 não justificam mudança de design |
| Gemini: `pg_temp` desnecessário em search_path | Gemini | `pg_temp` no final é convenção defensiva contra ataques de schema search. Padrão recomendado |
| Gemini: renomear `replace_cbo_link` para `replace_canonical_cbo_link` | Gemini | RPCs criadas seguem padrão verbo + contexto curto (`process_opus_create_new`, `retire_canonical`). Renomear adiciona caracteres sem ganho |
| Gemini: dependência de `auto_assign_family_to_canonical(uuid, bool)` | Gemini | v16 §5.11 já tem assinatura com 2 parâmetros em produção. Pré-requisito atendido |
| GenSpark: rollback playbook explícito | GenSpark | Reset E2E já é o rollback. Sem benefício de duplicar |
| Grok: cache Redis para embeddings | Grok | HNSW com ~2.600 rows é instantâneo. Otimização prematura |
| Grok: particionamento futuro de PCM | Grok | Volume não justifica. Avaliar quando crescer |
| Manus: namespace de slugs CBO (`cbo_*`) | Manus | Pré-deduplicação por título + early return CBO já tratam o problema |
| Manus: incompatibilidade de dimensão de embeddings | Manus | v16 em produção usa 768d Gemini consistentemente |
| Manus: Camada 2 poluída por placeholders sem vagas | Manus | Placeholders sem vagas têm `vacancy_count=0` e Camada 2 ordena DESC. Ficam no fim |

## ⚠️ Duplicatas

- Grok: checksums SHA-256 dos CSVs já implementado em `scripts/seed-cbo-versions.ts` via `files_hash`
- Grok: monitoramento se cap atingido contemplado em LK-CBO-17 com critério >5%
- Grok: listar 12 colunas de updated_at trigger já listadas em §2.6

---

# Changelog v4 → v5 (preservado para histórico)

| Categoria | Item | Tipo |
|---|---|---|
| ALINHAMENTO V16 | §2.7 ALTER `fn_promote_canonical_on_threshold` reescrito sobre body real da v16 (AFTER, sem `pg_trigger_depth`, com critério `confidence_median >= 0.75`, com chamada a `auto_assign_family_to_canonical`) | Schema |
| ALINHAMENTO V16 | §2.9 reescrito sobre body real da v16: trigger é `trg_cleanup_zombie_canonical` em `job_canonical_roles`, AFTER UPDATE OF `vacancy_count`, com 14 FK guards. Soft delete remove FK guards (mortos em UPDATE), amplia filtro para `status IN ('active', 'pending', 'rejected')`, exclui `deprecated`/`alias_of`/`merge_candidate` | Schema |
| ALINHAMENTO V16 | §3.7 `detectStaleCanonicals` mantém `MAX_ALERTS_PER_EXECUTION = 10` herdado da v16, com instrumentação adicional em PCM | Código |
| ALINHAMENTO V16 | §3.11 sweep ajustado para nomes reais da v16 (`trg_cleanup_zombie_canonical`, `canonical_zombie_cleanup`) | Código |
| BLOCKER V4 | V4-B1 corrigido: pré-check de colisão de slug agora é script TypeScript dedicado lendo CSV real (Etapa CBO.0a) | Plano |
| ALTO V4 | V4-H1 corrigido: confidence em `canonical_role_cbo_links` para seeds CBO é NULL, alinhado ao comentário do schema | Código |
| ALTO V4 | V4-H2 corrigido: §3.9 e §3.10 declaram explicitamente que `callRoute` envia headers `X-Benchmark-Run-Id` e `X-Benchmark-Cache-Control` | Código |
| ALTO V4 | V4-H3 corrigido: §3.4 explicita que filtro a estender é o in-memory de `role.status` (linha ~112), não o `.eq('status', 'active')` SQL | Código |
| ALTO V4 | V4-H4 retirado: defesa `MAX_ALERTS_PER_EXECUTION = 10` já existe em v16, dispensa RPC nova `fetch_stale_canonical_candidates` | Código |
| MÉDIO V4 | V4-M1 corrigido: §3.4 ganha código exato do sort por `statusPriority` | Código |
| MÉDIO V4 | V4-M2 corrigido: D-CBO-25 explicitamente revisável via PCM, removendo contradição com §9.2 | Decisão |
| MÉDIO V4 | V4-M3 corrigido: `.like` trocado por `.eq` em `gatherStalenessTelemetry` | Código |
| NOVO | LK-CBO-17 documenta `MAX_ALERTS_PER_EXECUTION = 10` herdado da v16 sem justificativa empírica, com critério de revisão futura (>5% execuções com cap atingido) | Documentação |

---

# Estado-alvo

Sistema CalibraCV pós-sprint CBO, antes do benchmark E2E da v14, com:

- Catálogo CBO 2002 do MTE carregado como tabela de referência permanente, com embeddings Gemini 768d gerados antes do benchmark
- Cerca de 2.500 canônicos placeholder seedados em `job_canonical_roles` (deduplicados por título normalizado)
- Cerca de 600 famílias CBO populadas em `taxonomy_families`, todas com embedding 768d via display_name
- Cerca de 2.600 vínculos em `canonical_role_cbo_links` (1 placeholder pode ter N occupation_codes via N:M, com `confidence=NULL` em seeds)
- Cerca de 5.000 sinônimos oficiais MTE seedados em `taxonomy_relations` com `linguistic_category='synonym'`
- Coluna `cbo_codes TEXT[]` removida de `job_canonical_roles`
- Coluna `updated_at` adicionada a `job_canonical_roles` com trigger semântico (12 colunas monitoradas)
- Coluna `embedding VECTOR(768)` adicionada a `taxonomy_families`
- Status `'retired'` adicionado ao enum de `job_canonical_roles.status` (CHECK com 7 valores totais)
- Coluna `mode` em `benchmark_runs` ganha valor `'bulk'` no CHECK
- Função `fn_cleanup_zumbi_canonical` renomeada para `fn_retire_canonical_on_zero_vacancy`, sem FK guards (mortos em soft delete), com early return CBO + delegação à RPC `retire_canonical`
- Trigger `trg_cleanup_zombie_canonical` renomeado para `z_trg_retire_canonical_on_zero_vacancy` (em `job_canonical_roles`, alinhado à v16)
- Filtro Frente K ampliado para `status IN ('active', 'pending', 'rejected')`, excluindo `deprecated`, `alias_of`, `merge_candidate`
- RPC `retire_canonical(id, reason)` única consumida por 2 callers: trigger Frente K e CRON `detectStaleCanonicals`
- Trigger `fn_promote_canonical_on_threshold` da v16 ALTERADO para aceitar `OLD.status IN ('pending', 'retired')` na ressuscitação, preservando AFTER UPDATE, critério `confidence_median >= 0.75`, e chamada a `auto_assign_family_to_canonical`
- Arquivo `lib/pipeline/maintenance-zombie-detection.ts` renomeado para `lib/pipeline/maintenance-canonical-staleness.ts`
- Função `detectZombies` renomeada para `detectStaleCanonicals`, sem sugestão de merge, chamando `retire_canonical(id, 'no_recent_postings_365d')` quando gap >= 365d
- `MAX_ALERTS_PER_EXECUTION = 10` preservado da v16, com instrumentação em PCM
- `emitEventOnce` da v16 mantido para AMBER (dedup 30d); RED não usa `emitEventOnce` porque ação aposenta canônico que sai naturalmente do pool da próxima execução
- Constantes `RED_THRESHOLD_DAYS`/`AMBER_THRESHOLD_DAYS` renomeadas para `STALE_RED_THRESHOLD_DAYS`/`STALE_AMBER_THRESHOLD_DAYS`
- `event_name` `canonical_red_alert`/`canonical_amber_alert` renomeados para `canonical_stale_alert_red`/`canonical_stale_alert_amber`
- Novo `event_name` `canonical_retired` registrado pela RPC `retire_canonical` com contexto rico
- `internal.reset_taxonomy_core` v5.19-cbo-aware preserva rows com `cbo_version_id IS NOT NULL`; zera `pipeline_calibration_metrics` (sem backfill)
- Camadas 0, 1, 2 estendidas para incluir `'retired'` no filtro de canônicos visíveis
- Camada 2 com piso de 2 tokens + tie-break por `cbo_family_code IS NOT NULL` + tiebreak `ORDER BY label ASC`
- `findOrCreateFamily` refatorada para embedding cosine puro 0.85 (sem bonus aditivo) com réplica Opus em casos ambíguos; cada decisão registra row em PCM
- `processOpusTaxonomyRelationVerdict` e `processOpusOrphanCanonical` estendidos para `suggested_cbo` e verdict `CREATE_NEW`
- RPC `process_opus_create_new` com guards de range e atomicidade de vínculo CBO
- RPC `replace_cbo_link` para substituição atômica de vínculo CBO via Opus
- RPC `fetch_cbo_candidates` para top-K candidatos via cosine
- RPC `fetch_family_candidates` para top-K candidatos com cosine puro
- Tabela `pipeline_calibration_metrics` como infraestrutura permanente de calibragem, sem UNIQUE (append-only)
- Refactor de `scripts/benchmark-and-curate-bulk.ts` com 7 retries por size (3 cold + 4 hot, 1 warmup descartada), agregação estatística com média/DP/IC95, tiebreaker de IC95 sobreposto
- `callRoute` envia headers `X-Benchmark-Run-Id` e `X-Benchmark-Cache-Control` em todas as chamadas durante benchmark
- Endpoint `/api/admin/curate-job-postings` aceita header `X-Benchmark-Cache-Control: false`
- Página `/methodology` atualizada com seção "Cinco camadas que se auditam mutuamente" e referências CBO/ISCO

---

# Abreviações

| Sigla | Significado |
|---|---|
| `JCR` | `job_canonical_roles` |
| `JP` | `job_postings` |
| `TR` | `taxonomy_relations` |
| `TF` | `taxonomy_families` |
| `TFC` | `taxonomy_family_canonicals` |
| `CRD` | `canonical_role_domains` |
| `CRDL` | `canonical_role_domain_links` |
| `CRC` | `canonical_role_cbo`, catálogo de ocupações 6 dígitos |
| `CRCL` | `canonical_role_cbo_links`, junção N:M |
| `CV` | `cbo_versions` |
| `PCM` | `pipeline_calibration_metrics`, infraestrutura viva de calibragem |
| `MTE` | Ministério do Trabalho e Emprego |
| `CBO` | Classificação Brasileira de Ocupações 2002 |
| `ISCO` | International Standard Classification of Occupations da OIT |

---

# Convenções

- "ocupação CBO" é registro em `canonical_role_cbo` com `occupation_code` 6 dígitos
- "família CBO" é registro em `taxonomy_families` com `cbo_family_code` 4 dígitos
- "canônico placeholder" é registro em `JCR` criado pelo seed CBO com `source='cbo_mte_2002_seed'`
- "sinônimo MTE" é registro em `TR` com `cbo_version_id IS NOT NULL` e `linguistic_category='synonym'`
- "canônico aposentado" é registro em `JCR` com `status='retired'`, com contadores zerados, ainda visível para ressuscitação
- "calibragem viva" é registro em `pipeline_calibration_metrics`, podendo vir de benchmark, MVP ou produção
- "stale alert" é evento RED ou AMBER emitido por `detectStaleCanonicals` quando gap entre última vaga e momento atual ultrapassa o threshold
- "soft delete" é mudança de status para `'retired'` com zeragem de contadores (`vacancy_count=0`, `distinct_sources_count=0`); preserva o registro em `JCR` para ressuscitação futura
- "merge" é absorção de canônico A em canônico B via `merged_into`, deixando A com `status='deprecated'`. Operação distinta de soft delete: deprecated reflete histórico de fusão, retired reflete inatividade reversível

Padrão SQL:
- Migrations idempotentes via `CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`
- Aplicadas via Claude Code conectado ao Supabase
- FKs com `ON DELETE` explícito
- Toda row CBO carrega `cbo_version_id` para versionamento e preservação no reset
- Path padrão: `docs/migrations/sprint-cbo/`

---

# Decisões arquiteturais cravadas

| ID | Decisão |
|---|---|
| D-CBO-01 | CBO 2002 é fonte de seed e enriquecimento, não autoridade taxonômica do produto. Taxonomia viva permanece governada por Sonnet curador e Opus árbitro |
| D-CBO-02 | Tabela `canonical_role_cbo` permanente como catálogo MTE com embedding Gemini 768d. Tabelas intermediárias não são criadas. Dados vão direto às tabelas finais |
| D-CBO-03 | `canonical_role_cbo.occupation_code TEXT PRIMARY KEY` no formato `^\d{4}-\d{2}$`. Chave natural estável MTE preferida sobre UUID |
| D-CBO-04 | Família CBO 4 dígitos vive em `taxonomy_families` com nova coluna `cbo_family_code TEXT NULL`. Sem tabela paralela |
| D-CBO-05 | Canônico para Ocupação CBO modelado como N:M via `canonical_role_cbo_links`, não array `cbo_codes TEXT[]`. Coluna `cbo_codes` em `JCR` é dropada |
| D-CBO-06 | Tabela `cbo_versions` única para versionamento MTE. FK propagada para `taxonomy_families`, `taxonomy_relations`, `canonical_role_cbo` |
| D-CBO-07 | Reset E2E preserva rows com `cbo_version_id IS NOT NULL`. Sem coluna `seed_origin` separada |
| D-CBO-08 | Enum `linguistic_category` em `TR` ganha valor `'synonym'`. Aplicado fixo em todo seed CBO |
| D-CBO-09 | CHECK constraint em `TR`: `cbo_version_id IS NOT NULL` implica `linguistic_category='synonym'` |
| D-CBO-10 | Constraint `source` em `JCR` estendida para incluir `'cbo_mte_2002_seed'` |
| D-CBO-11 | Seed cria 1 canônico placeholder por **título CBO normalizado** (após dedup), não por occupation_code. Múltiplas ocupações com mesmo título compartilham 1 placeholder via N:M em `canonical_role_cbo_links`. Volume estimado: ~2.500 placeholders cobrindo ~2.600 ocupações |
| D-CBO-12 | Canônicos placeholder ficam sem `canonical_role_domain_links` no seed. Opus atribui domínio durante arbitragem normal pós-vagas |
| D-CBO-13 | Frente K renomeada e refatorada (D-CBO-30); placeholders CBO ganham early return via `source = 'cbo_mte_2002_seed'`; soft delete via `'retired'`. Filtro ampliado para `status IN ('active', 'pending', 'rejected')` em coerência com soft delete; exclui `deprecated` (já é histórico de merge), `alias_of` (parte de cadeia de redirect), `merge_candidate` (em fila Opus) |
| D-CBO-14 | Sonnet curador permanece transparente em relação à origem CBO. Zero mudança em `SYSTEM_PROMPT.ts` v2.7 |
| D-CBO-15 | Camada 1 e Camada 2 são modificadas para incluir `'retired'` no filtro |
| D-CBO-16 | Merge é não-destrutivo, preserva histórico em loser via `status='deprecated' + merged_into`. Cenários A, B1, B2, B3, D não exigem lógica nova de tratamento de links CBO. **Soft delete (`'retired'`) e merge (`'deprecated'`) são operações conceitualmente distintas: deprecated reflete absorção, retired reflete inatividade reversível** |
| D-CBO-17 | Cenário C, canônico próprio recebe vínculo CBO via Opus, exige novo campo `suggested_cbo` no schema do tool_use de `OPUS_PROMPTS.taxonomy_relation` e `orphan_canonical`. Política: 1 link CBO único por canônico via Opus, com flag `replace_existing` para substituição (D-CBO-31) |
| D-CBO-18 | Cenário E, Opus cria canônico genuinamente novo, exige verdict `CREATE_NEW`, RPC `process_opus_create_new` com validação de range e atomicidade |
| D-CBO-19 | Cosine via pgvector operador `<=>`, index HNSW `vector_cosine_ops` com `m=16, ef_construction=128`. Modelo único Gemini `gemini-embedding-001` 768d |
| D-CBO-20 | Cleanup de `generateE5Embedding`, alias morto, entra como sub-item da sprint |
| D-CBO-21 | Camada 0, precheck, segue SHA-256 exato. Estende filtro para incluir `'retired'` |
| D-CBO-22 | Página `/methodology` recebe atualização editorial dentro da sprint |
| D-CBO-23 | Durante benchmark, Sonnet e Opus rodam concorrentemente. Sonnet processa vagas continuamente; Opus CRON dispara a cada 15-20 minutos sobre relations envolvendo canônicos `active`. Em 3-4 horas de benchmark, Opus roda 10-15 vezes |
| D-CBO-24 | Soft delete via status `'retired'`. Aposentadoria zera `vacancy_count` e `distinct_sources_count` atomicamente. Ressuscitação por Sonnet ou Opus segue mesmas regras da Frente A: 3 vagas + 2 fontes distintas + 1 vaga ativa nos últimos 60 dias + `confidence_median >= 0.75`. Trigger `fn_promote_canonical_on_threshold` ALTERADO para aceitar `OLD.status IN ('pending', 'retired')` |
| D-CBO-25 | Camada 2 ganha 3 mitigações estruturais: piso de 2 tokens em `findFamilyMatch` (elimina overlap espúrio), tie-break por `cbo_family_code IS NOT NULL` (preferência institucional CBO em desempate), tiebreak `ORDER BY label ASC` (determinístico em vacancy_count empatado). O comportamento do tie-break CBO será observado via `pipeline_calibration_metrics` e revisitado pós-benchmark se evidência empírica indicar necessidade de bonus aditivo em vez de hard preference |
| D-CBO-26 | `findOrCreateFamily` refatorada para embedding cosine threshold 0.85 contra `taxonomy_families.embedding` (gerado via `display_name`). Cosine puro, sem bonus aditivo CBO. Quando candidato existente acima do threshold, réplica ao Opus decide reuso vs criação. Cada decisão registra row em PCM. Threshold 0.85 será revalidado em análise pós-benchmark |
| D-CBO-27 | Pré-deduplicação no seed por título normalizado. Se múltiplas ocupações CBO compartilham o mesmo `title`, criar 1 placeholder único e vincular múltiplos `occupation_code` via N:M. Slug do placeholder = slugify(title), sem sufixo de código. Quando título dedupado pertence a famílias CBO diferentes, vinculações M:N em `taxonomy_family_canonicals` para todas as famílias |
| D-CBO-28 | Benchmark roda 7 retries por batch_size: 3 SEM cache (baseline) + 4 COM cache (1 warmup descartada + 3 hot válidas). Decisão de batch_size vencedor via média estatística de `media_regime_hot_cache.custo_por_vaga`, com tiebreaker de IC95 sobreposto preferindo menor batch_size. Sleep de 6min entre fases SEM/COM cache. Lista coarse: `[10, 25, 35, 37, 39, 40, 50, 75, 100]` |
| D-CBO-29 | Tabela `pipeline_calibration_metrics` é infraestrutura **viva** de calibragem, não exclusiva do benchmark. Acomoda métricas durante benchmark, MVP e produção. Sem backfill de dados pré-refactor. Sem UNIQUE constraint (append-only) |
| D-CBO-30 | Aposentadoria de canônicos unifica via função única `retire_canonical(id, reason)` com 2 callers: trigger renomeado `fn_retire_canonical_on_zero_vacancy` (motivo `zero_vacancy_count`) e CRON `detectStaleCanonicals` (motivo `no_recent_postings_365d`). Renomeação total de zombie/zumbi em PT+EN no codebase, sem deixar resíduo. RED alert deixa de sugerir merge, vira ação automática de aposentadoria |
| D-CBO-31 | Substituição de vínculo CBO via Opus permitida via flag `replace_existing` no schema de `suggested_cbo`. Quando `replace_existing=true` e canônico já tem link, RPC nova `replace_cbo_link` faz remoção+inserção atômica com audit log. Quando `replace_existing=false` (ou ausente) e canônico já tem link, sugestão é descartada com warning |

---

# Investigação prévia (Claude Code)

Antes da redação desta v5, foram investigadas no codebase produtivo (consolidado de v3 + v4 + auditoria à v4 + diff v14→v16):

| Q | Achado | Impacto |
|---|---|---|
| Q1 | `merge_canonicals` não toca em `canonical_role_cbo_links`. Tabela ainda não existe | Trivial, tabela é criada nesta sprint |
| Q2 | Helpers `resolveCanonicalById` e `resolveCanonicalRedirect` existem em `lib/pipeline/upsert-canonical.ts:96-163`. `resolveCanonicalById` retorna `Promise<string>` | Criar `resolveCanonicalCbo()` paralelo |
| Q3 | `processOpusTaxonomyRelationVerdict` em `DISAGREE_MERGE` chama `process_opus_disagree` sem checar links CBO | Trivial agora. Após sprint, tratamento natural via cadeia merge |
| Q4 | `process_opus_disagree` com `p_existing_canonical_id=NULL` faz UPDATE de label/slug, NÃO cria canônico novo. `confidence_at_creation` e `updated_at` não existem em `JCR` | Cenário E exige RPC nova `process_opus_create_new`. Adicionar `updated_at` ao schema |
| Q5 | Tool_use Opus atual tem `suggested_family/domain/linguistic_category` mas SEM `suggested_cbo` | Adicionar `suggested_cbo` em `taxonomy_relation` e `orphan_canonical` |
| Q6 | System prompt do Opus em bloco com `cache_control: ephemeral`. Mudança invalida cache | Aceitável |
| Q7 | `JCR.source` aceita só `'seed'` ou `'llm_extractor'`. CHECK status em v16 aceita 6 valores: `pending`, `active`, `deprecated`, `alias_of`, `rejected`, `merge_candidate`. Frente K em v16 filtra apenas `status='pending'` | ALTER CHECK `source` para incluir `'cbo_mte_2002_seed'`. ALTER CHECK `status` adicionar `'retired'` PRESERVANDO os 6 originais (total 7). Frente K v5 amplia filtro para `('active', 'pending', 'rejected')` excluindo `deprecated`/`alias_of`/`merge_candidate` |
| Q8 | `canonical_role_cbo_links` ainda não existe. Padrão observado: `ON DELETE CASCADE` em tabelas filhas | Aplicar `ON DELETE CASCADE` em `canonical_role_id` |
| Q9 | `findOrCreateFamily` é idempotente, faz exact match → fuzzy 0.7 trigram → create. Threshold 0.7 é ad-hoc, sem calibragem documentada | Refatorar para embedding cosine 0.85 com réplica Opus, cosine puro |
| Q10 | RLS no projeto é tier-based. Tier 1 (pipeline interno) tem `ENABLE ROW LEVEL SECURITY` sem `CREATE POLICY` | Spec correto, sem `CREATE POLICY` necessário |
| Q11 | Camada 2 hoje filtra `('active', 'pending')`. Camada 0 idem. Camada 1 idem + gate `allowedForPreResolution` mais restrito. Em `taxonomy-cache.ts`, função `getRelations` tem filtro SQL (`taxonomy_relations.status='active'`) E filtro in-memory (`role.status IN ('active', 'pending')`). Estender APENAS o filtro in-memory; `taxonomy_relations.status` não tem `'retired'` no enum | Estender filtro in-memory para incluir `'retired'`. Camada 1 mantém gate (não ressuscita) |
| Q12 | Função real é `fn_cleanup_zumbi_canonical` em PT (na v16). Trigger é `trg_cleanup_zombie_canonical` em EN, em `job_canonical_roles` (não `job_postings`!), AFTER UPDATE OF `vacancy_count`. Filtro v16: `vacancy_count = 0 AND status = 'pending'`. 14 FK guards. 1 INSERT (`canonical_zombie_cleanup`) | Renomear todas referências PT+EN. Reescrever §2.9 sobre body real da v16. Ampliar filtro para `('active', 'pending', 'rejected')` |
| Q13 | Correlação batch ↔ vaga via `function_orchestrator_items.run_id` (UUID FK para `function_orchestrator_runs`). Cobertura 100% | Coletar métricas ANTES de qualquer reset |
| Q14 | Benchmark atual `benchmark-and-curate-bulk.ts` é single-shot por size. Sem cold/hot, sem warmup, sem agregação estatística. Pt. 14 cravou 7 retries (Leitura B: toggle cache_control via header) que nunca chegou ao código | Refactor obrigatório do script |
| Q15 | Endpoint `/api/admin/curate-job-postings` lê header `x-skip-opus-inline` mas NÃO controla `cache_control: ephemeral`. Necessário adicionar header `X-Benchmark-Cache-Control`. `callRoute` em `scripts/benchmark-and-curate-bulk.ts:180-185` hoje envia apenas `Content-Type` e `x-internal-key` | Modificar endpoint + chain TS (`llm-call.ts`, `batch-processor.ts`). Modificar `callRoute` para enviar headers `X-Benchmark-Run-Id` e `X-Benchmark-Cache-Control` |
| Q16 | `detectZombies` em `lib/pipeline/maintenance-zombie-detection.ts` apenas emite events `canonical_red_alert`/`canonical_amber_alert` sem alterar estado. Não há painel admin que exibe esses alertas. `MAX_ALERTS_PER_EXECUTION = 10` em produção desde v16, sem justificativa empírica documentada | Promover RED para ação automática (chamada a `retire_canonical`); manter event log estruturado para painel futuro. Manter cap=10. Adicionar instrumentação em PCM |
| Q17 | Trigger `fn_promote_canonical_on_threshold` em v16 é AFTER UPDATE OF (`vacancy_count`, `distinct_sources_count`), faz `NEW.status := 'active'` direto, sem `pg_trigger_depth`. Critério adicional: `confidence_median IS NULL OR confidence_median >= 0.75`. Janela 60 dias. Chama `auto_assign_family_to_canonical(NEW.id, TRUE)` no caminho de promoção | ALTER trigger preservando estrutura v16, apenas estendendo `OLD.status IN ('pending', 'retired')` para suportar ressuscitação |

---

# Parte 2 — Schema SQL

Path padrão: `docs/migrations/sprint-cbo/`. Migrations numeradas em ordem de execução.

## 2.1 — `01_cbo_versions.sql`

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS cbo_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  dataset_version TEXT UNIQUE NOT NULL,
  source_updated_at DATE NOT NULL,
  imported_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  imported_by TEXT NOT NULL,
  files_hash TEXT NOT NULL,
  is_current BOOLEAN NOT NULL DEFAULT FALSE,
  notes TEXT
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_cbo_versions_is_current
  ON cbo_versions(is_current) WHERE is_current = TRUE;

ALTER TABLE cbo_versions ENABLE ROW LEVEL SECURITY;

COMMIT;
```

## 2.2 — `02_taxonomy_families_alter.sql`

```sql
BEGIN;

ALTER TABLE taxonomy_families
  ADD COLUMN IF NOT EXISTS cbo_family_code TEXT NULL
    CHECK (cbo_family_code IS NULL OR cbo_family_code ~ '^\d{4}$'),
  ADD COLUMN IF NOT EXISTS cbo_version_id UUID NULL
    REFERENCES cbo_versions(id) ON DELETE RESTRICT,
  ADD COLUMN IF NOT EXISTS embedding VECTOR(768) NULL;

CREATE UNIQUE INDEX IF NOT EXISTS idx_tf_cbo_family_code_per_version
  ON taxonomy_families(cbo_family_code, cbo_version_id)
  WHERE cbo_family_code IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_tf_embedding_hnsw
  ON taxonomy_families
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 128)
  WHERE embedding IS NOT NULL;

COMMIT;
```

## 2.3 — `03_canonical_role_cbo.sql`

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS canonical_role_cbo (
  occupation_code TEXT PRIMARY KEY
    CHECK (occupation_code ~ '^\d{4}-\d{2}$'),
  title TEXT NOT NULL,
  family_id UUID NOT NULL REFERENCES taxonomy_families(id) ON DELETE RESTRICT,
  summary_description TEXT,
  embedding VECTOR(768),
  cbo_version_id UUID NOT NULL REFERENCES cbo_versions(id) ON DELETE RESTRICT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_crc_family
  ON canonical_role_cbo(family_id);

CREATE INDEX IF NOT EXISTS idx_crc_cbo_version
  ON canonical_role_cbo(cbo_version_id);

CREATE INDEX IF NOT EXISTS idx_crc_embedding_hnsw
  ON canonical_role_cbo
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 128);

ALTER TABLE canonical_role_cbo ENABLE ROW LEVEL SECURITY;

COMMIT;
```

## 2.4 — `04_taxonomy_relations_alter.sql`

```sql
BEGIN;

DO $$
DECLARE
  invalid_count INT;
BEGIN
  SELECT COUNT(*) INTO invalid_count
  FROM taxonomy_relations
  WHERE linguistic_category IS NOT NULL
    AND linguistic_category NOT IN (
      'acronym', 'translation', 'orthographic',
      'slang', 'specialty', 'domain_alias',
      'synonym', 'other'
    );
  IF invalid_count > 0 THEN
    RAISE EXCEPTION 'Migration abortada: % rows em taxonomy_relations com linguistic_category fora do novo enum', invalid_count;
  END IF;
END $$;

ALTER TABLE taxonomy_relations
  DROP CONSTRAINT IF EXISTS taxonomy_relations_linguistic_category_check;

ALTER TABLE taxonomy_relations
  ADD CONSTRAINT taxonomy_relations_linguistic_category_check
  CHECK (linguistic_category IS NULL OR linguistic_category IN (
    'acronym', 'translation', 'orthographic',
    'slang', 'specialty', 'domain_alias',
    'synonym',
    'other'
  ));

ALTER TABLE taxonomy_relations
  ADD COLUMN IF NOT EXISTS cbo_version_id UUID NULL
    REFERENCES cbo_versions(id) ON DELETE RESTRICT;

ALTER TABLE taxonomy_relations
  DROP CONSTRAINT IF EXISTS chk_tr_cbo_version_implies_synonym;

ALTER TABLE taxonomy_relations
  ADD CONSTRAINT chk_tr_cbo_version_implies_synonym
  CHECK (
    cbo_version_id IS NULL
    OR linguistic_category = 'synonym'
  );

COMMIT;
```

## 2.5 — `05_canonical_role_cbo_links.sql`

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS canonical_role_cbo_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  canonical_role_id UUID NOT NULL
    REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
  occupation_code TEXT NOT NULL
    REFERENCES canonical_role_cbo(occupation_code) ON DELETE RESTRICT,
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  confidence NUMERIC(4,3)
    CHECK (confidence IS NULL OR (confidence >= 0 AND confidence <= 1)),
  source TEXT NOT NULL
    CHECK (source IN ('cbo_mte_2002_seed', 'opus_arbitration', 'manual_admin')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE (canonical_role_id, occupation_code)
);

CREATE INDEX IF NOT EXISTS idx_crcl_canonical
  ON canonical_role_cbo_links(canonical_role_id);

CREATE INDEX IF NOT EXISTS idx_crcl_occupation
  ON canonical_role_cbo_links(occupation_code);

CREATE UNIQUE INDEX IF NOT EXISTS idx_crcl_one_primary_per_canonical
  ON canonical_role_cbo_links(canonical_role_id)
  WHERE is_primary = TRUE;

ALTER TABLE canonical_role_cbo_links ENABLE ROW LEVEL SECURITY;

COMMENT ON COLUMN canonical_role_cbo_links.confidence IS
  'NULL para seeds (cbo_mte_2002_seed) — dado institucional MTE não tem métrica de confiança aplicável. Preenchido (0 a 1) por opus_arbitration e manual_admin.';

COMMIT;
```

## 2.6 — `06_job_canonical_roles_alter.sql`

```sql
BEGIN;

ALTER TABLE job_canonical_roles
  DROP CONSTRAINT IF EXISTS job_canonical_roles_source_check;

ALTER TABLE job_canonical_roles
  ADD CONSTRAINT job_canonical_roles_source_check
  CHECK (source IN ('seed', 'llm_extractor', 'cbo_mte_2002_seed'));

ALTER TABLE job_canonical_roles
  DROP CONSTRAINT IF EXISTS job_canonical_roles_status_check;

ALTER TABLE job_canonical_roles
  ADD CONSTRAINT job_canonical_roles_status_check
  CHECK (status IN ('pending', 'active', 'deprecated', 'alias_of', 'rejected', 'merge_candidate', 'retired'));

ALTER TABLE job_canonical_roles DROP COLUMN IF EXISTS cbo_codes;

ALTER TABLE job_canonical_roles
  ADD COLUMN IF NOT EXISTS updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW();

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
    NEW.confidence_median IS DISTINCT FROM OLD.confidence_median OR
    NEW.human_validated IS DISTINCT FROM OLD.human_validated OR
    NEW.promoted_at IS DISTINCT FROM OLD.promoted_at OR
    NEW.rejected_reason IS DISTINCT FROM OLD.rejected_reason OR
    NEW.blacklist_expiry_at IS DISTINCT FROM OLD.blacklist_expiry_at OR
    NEW.is_emerging IS DISTINCT FROM OLD.is_emerging
  ) THEN
    NEW.updated_at = NOW();
  END IF;
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS jcr_set_updated_at ON job_canonical_roles;

CREATE TRIGGER jcr_set_updated_at
  BEFORE UPDATE ON job_canonical_roles
  FOR EACH ROW EXECUTE FUNCTION trg_jcr_set_updated_at();

COMMIT;
```

## 2.7 — `07_alter_promote_canonical_for_retired.sql`

ALTER do trigger Frente A baseado no **body real da v16 deployada** (`schema.sql:3473-3567`), não na v16 spec text. **Crítico:** v16 spec text e v16 deployada divergem nesse ponto. A spec text descreve `NEW.status := 'active'` em trigger AFTER, que é semanticamente vazio (atribuições em NEW só persistem em BEFORE). A v16 deployada usa pattern correto: `pg_trigger_depth() > 1` guard + UPDATE explícito separado. Esta versão preserva a deployada exatamente, estendendo apenas o enum para suportar ressuscitação (D-CBO-24).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION fn_promote_canonical_on_threshold()
RETURNS TRIGGER
LANGUAGE plpgsql
SET search_path = public, pg_temp
AS $function$
DECLARE
    v_has_recent_vacancy BOOLEAN := FALSE;
BEGIN
    -- Recursion guard: necessário porque AFTER trigger faz UPDATE na própria tabela.
    -- Sem este guard, qualquer promoção causaria loop infinito.
    IF pg_trigger_depth() > 1 THEN
        RETURN NULL;
    END IF;

    -- ÚNICA MUDANÇA SPRINT CBO vs v16 deployada:
    -- Estender enum para suportar ressuscitação retired→active (D-CBO-24).
    -- Resto do body preservado verbatim da v16 deployada (schema.sql:3473-3567).
    IF NEW.status IN ('pending', 'retired')
       AND NEW.vacancy_count >= 3
       AND NEW.distinct_sources_count >= 2
       AND OLD.status IN ('pending', 'retired')
    THEN
        SELECT EXISTS (
            SELECT 1 FROM job_postings
            WHERE canonical_role_id = NEW.id
              AND posted_at >= NOW() - INTERVAL '60 days'
              AND is_active = true
        ) INTO v_has_recent_vacancy;

        IF v_has_recent_vacancy THEN
            -- UPDATE explícito separado (pattern correto para AFTER trigger).
            -- COALESCE preserva valores originais em ressuscitação.
            UPDATE job_canonical_roles
            SET status = 'active',
                promoted_at = COALESCE(promoted_at, NOW()),
                confidence_median_at_promotion = COALESCE(
                    confidence_median_at_promotion, NEW.confidence_median
                )
            WHERE id = NEW.id
              AND status IN ('pending', 'retired');

            PERFORM auto_assign_family_to_canonical(NEW.id, TRUE);

            INSERT INTO events (
                event_name, resource_type, resource_id,
                actor, actor_id, previous_state, new_state, reason
            )
            VALUES (
                'canonical_promoted_on_threshold',
                'job_canonical_role',
                NEW.id,
                'system',
                '00000000-0000-0000-0000-000000000001',
                jsonb_build_object(
                    'previous_status', OLD.status,
                    'vacancy_count', OLD.vacancy_count,
                    'distinct_sources_count', OLD.distinct_sources_count
                ),
                jsonb_build_object(
                    'status', 'active',
                    'vacancy_count', NEW.vacancy_count,
                    'distinct_sources_count', NEW.distinct_sources_count,
                    'confidence_median', NEW.confidence_median,
                    'is_resurrection', (OLD.status = 'retired')
                ),
                CASE WHEN OLD.status = 'retired'
                     THEN 'Ressuscitação via gate cumulativo'
                     ELSE 'Promoção via gate cumulativo'
                END
            );
        END IF;
    END IF;

    -- AFTER trigger: retorno é ignorado pelo executor.
    RETURN NULL;
END;
$function$;

COMMIT;
```

Notas:

- Pattern AFTER + UPDATE explícito + `pg_trigger_depth()` é o que **realmente roda em produção** desde v16. Mudança mínima absoluta: extensão de enum no `IF NEW.status IN (...)` e no `IF OLD.status IN (...)`
- `COALESCE(promoted_at, NOW())` preserva timestamp original na ressuscitação. Importante para auditoria
- `COALESCE(confidence_median_at_promotion, NEW.confidence_median)` preserva snapshot original
- `is_resurrection` no metadata permite distinguir promoção inicial de ressuscitação
- O `WHERE ... AND status IN ('pending', 'retired')` no UPDATE é defesa anti-race: se outro processo já promoveu o canônico entre o trigger disparar e o UPDATE rodar, nada acontece

**Erro semântico identificado na v5 (V5-B1):** v5 §2.7 importou body da v16 spec text (com `NEW.status := 'active'` direto em AFTER trigger). Isso seria silenciosamente descartado em runtime. v6 corrige importando body real do `schema.sql` deployado.

## 2.8 — `08_retire_canonical_rpc.sql`

Função única consumida pelos 2 callers (D-CBO-30). Em v6, ganha 2 correções:

- **V5-B6:** SELECT inicial com `source` + early return antes de tudo se for placeholder CBO. Evita evento falso de aposentadoria
- **V5-M4:** UPDATE com cláusula `NOT EXISTS` revalidando ausência de vaga recente, fechando janela de race entre `MAX(posted_at)` e o UPDATE

```sql
BEGIN;

CREATE OR REPLACE FUNCTION retire_canonical(
    p_canonical_id UUID,
    p_reason TEXT
) RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
    v_previous_status TEXT;
    v_source TEXT;
    v_previous_vacancy_count INT;
    v_previous_distinct_sources INT;
    v_last_posted_at TIMESTAMPTZ;
    v_gap_days INT;
    v_rows_affected INT;
BEGIN
    IF p_reason NOT IN ('zero_vacancy_count', 'no_recent_postings_365d') THEN
        RAISE EXCEPTION 'p_reason inválido: %. Aceito: zero_vacancy_count | no_recent_postings_365d', p_reason;
    END IF;

    -- V5-B6: SELECT inicial com source + lock (FOR UPDATE) para evitar race
    SELECT status, source, vacancy_count, distinct_sources_count
      INTO v_previous_status, v_source, v_previous_vacancy_count, v_previous_distinct_sources
      FROM job_canonical_roles
     WHERE id = p_canonical_id
     FOR UPDATE;

    IF v_previous_status IS NULL THEN
        RAISE EXCEPTION 'Canônico % não encontrado', p_canonical_id;
    END IF;

    -- V5-B6: early return ANTES de qualquer escrita se for placeholder CBO
    IF v_source = 'cbo_mte_2002_seed' THEN
        RETURN;
    END IF;

    -- Idempotência: já aposentado, retorna sem efeito
    IF v_previous_status = 'retired' THEN
        RETURN;
    END IF;

    IF p_reason = 'no_recent_postings_365d' THEN
        SELECT MAX(posted_at) INTO v_last_posted_at
          FROM job_postings
         WHERE canonical_role_id = p_canonical_id
           AND curation_status = 'curated';

        IF v_last_posted_at IS NOT NULL THEN
            v_gap_days := EXTRACT(DAY FROM NOW() - v_last_posted_at);
        END IF;
    END IF;

    -- V5-M4: UPDATE com cláusula NOT EXISTS revalidando condição.
    -- Se vaga curada apareceu entre MAX(posted_at) acima e este UPDATE,
    -- aposentadoria é cancelada (UPDATE afeta 0 linhas e early return).
    UPDATE job_canonical_roles
       SET status = 'retired',
           vacancy_count = 0,
           distinct_sources_count = 0
     WHERE id = p_canonical_id
       AND source IS DISTINCT FROM 'cbo_mte_2002_seed'
       AND (
         p_reason = 'zero_vacancy_count'
         OR NOT EXISTS (
           SELECT 1 FROM job_postings
           WHERE canonical_role_id = p_canonical_id
             AND curation_status = 'curated'
             AND posted_at >= NOW() - INTERVAL '365 days'
         )
       );

    GET DIAGNOSTICS v_rows_affected = ROW_COUNT;

    -- V5-B6: só insere event se UPDATE realmente afetou row.
    IF v_rows_affected = 0 THEN
        RETURN;
    END IF;

    INSERT INTO events (
        event_name, resource_type, resource_id,
        actor, actor_id, previous_state, new_state, reason
    )
    VALUES (
        'canonical_retired',
        'job_canonical_role',
        p_canonical_id,
        'system',
        '00000000-0000-0000-0000-000000000001',
        jsonb_build_object(
            'previous_status', v_previous_status,
            'previous_vacancy_count', v_previous_vacancy_count,
            'previous_distinct_sources_count', v_previous_distinct_sources,
            'last_posted_at', v_last_posted_at,
            'gap_days', v_gap_days
        ),
        jsonb_build_object(
            'new_status', 'retired',
            'vacancy_count', 0,
            'distinct_sources_count', 0
        ),
        p_reason
    );
END;
$$;

REVOKE EXECUTE ON FUNCTION retire_canonical(UUID, TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION retire_canonical(UUID, TEXT) TO service_role;

COMMENT ON FUNCTION retire_canonical(UUID, TEXT) IS
  'Aposenta canônico via soft delete (status=retired) zerando contadores. Idempotente. Consumida por z_trg_retire_canonical_on_zero_vacancy (trigger Frente K) e detectStaleCanonicals (CRON). p_reason aceita: zero_vacancy_count | no_recent_postings_365d. Excluída para placeholders CBO via early return + UPDATE filter (defesa em profundidade). UPDATE inclui revalidação anti-race via NOT EXISTS quando reason=no_recent_postings_365d.';

COMMIT;
```

Notas:

- `FOR UPDATE` no SELECT inicial pega lock pessimista da row, evitando que dois callers simultâneos vejam o mesmo `v_previous_status='active'` e ambos tentem aposentar
- Early return em placeholder CBO acontece ANTES de qualquer escrita (UPDATE ou INSERT), garantindo que `events` não receba registro falso
- `v_rows_affected = 0` cobre 2 cenários: (1) placeholder CBO chegou aqui por engano (UPDATE filter), (2) vaga apareceu na janela de race (NOT EXISTS filter). Em ambos, sem event

## 2.9 — `09_rename_zumbi_to_retire.sql`

Renomeação total da Frente K e seus triggers (D-CBO-30). **Reescrito sobre o body real da v16** observado no `schema.sql` deployado em produção. **V5-B2:** v16 deployada mantém os triggers antigos `z_trg_cleanup_zumbi_after_*` em `job_postings` (a v16 spec text descrevia migração que nunca foi aplicada). DROP FUNCTION sem dropar triggers dependentes falha com erro `cannot drop function ... because other objects depend on it`. v6 adiciona DROP TRIGGER defensivo cobrindo os 3 nomes possíveis. **V5-M7:** novo trigger ganha prefixo `z_` por consistência com convenção de ordenação alfabética dos triggers de recompute. Soft delete remove FK guards (mortos em UPDATE). Filtro ampliado para 3 status, excluindo `deprecated`/`alias_of`/`merge_candidate` em coerência com soft delete.

```sql
BEGIN;

-- V5-B2: drops defensivos cobrindo todos nomes possíveis em ambientes diferentes.
-- v16 deployada usa z_trg_cleanup_zumbi_after_* em job_postings.
-- v16 spec text descreveu trg_cleanup_zombie_canonical em job_canonical_roles
-- (que pode estar ou não em algum ambiente parcialmente migrado).
DROP TRIGGER IF EXISTS trg_cleanup_zombie_canonical ON job_canonical_roles;
DROP TRIGGER IF EXISTS z_trg_cleanup_zumbi_after_update ON job_postings;
DROP TRIGGER IF EXISTS z_trg_cleanup_zumbi_after_delete ON job_postings;

DROP FUNCTION IF EXISTS fn_cleanup_zumbi_canonical();

CREATE OR REPLACE FUNCTION fn_retire_canonical_on_zero_vacancy()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.vacancy_count != 0
       OR NEW.status NOT IN ('active', 'pending', 'rejected') THEN
        RETURN NEW;
    END IF;

    IF NEW.source = 'cbo_mte_2002_seed' THEN
        RETURN NEW;
    END IF;

    PERFORM retire_canonical(NEW.id, 'zero_vacancy_count');

    RETURN NEW;
END;
$$;

-- V5-M7: prefixo z_ garante ordenação alfabética DEPOIS dos triggers de recompute.
-- Postgres executa triggers AFTER em ordem alfabética. Triggers de vacancy_count
-- (trg_vacancy_count_*) vêm antes alfabeticamente, então o cleanup só dispara
-- após vacancy_count atualizado. Convenção herdada da v16 deployada.
CREATE TRIGGER z_trg_retire_canonical_on_zero_vacancy
    AFTER UPDATE OF vacancy_count ON job_canonical_roles
    FOR EACH ROW
    WHEN (NEW.vacancy_count = 0)
    EXECUTE FUNCTION fn_retire_canonical_on_zero_vacancy();

COMMENT ON FUNCTION fn_retire_canonical_on_zero_vacancy() IS
  'Trigger function que detecta canônicos não-CBO com vacancy_count=0 (status active/pending/rejected) e delega aposentadoria à RPC retire_canonical. Renomeada de fn_cleanup_zumbi_canonical na sprint CBO v6. FK guards removidos pois soft delete (UPDATE) não viola FK. Filtro ampliado para 3 status em coerência com soft delete; exclui deprecated (já é histórico de merge), alias_of (parte de cadeia de redirect), merge_candidate (em fila Opus).';

COMMIT;
```

Notas:

- **Cobertura via cascade:** Frente K em v6 vive em `job_canonical_roles` (não em `job_postings`). Caminhos de DELETE/UPDATE em vagas não disparam mais este trigger diretamente, mas a cascade via `trg_vacancy_count_delete` e `trg_vacancy_count_update` (que decrementam `vacancy_count` em `job_canonical_roles`) garante cobertura: quando `vacancy_count` chega a 0, este novo trigger dispara
- `WHEN (NEW.vacancy_count = 0)`: atalho de performance, evita execução da função quando não há possibilidade de aposentar
- Filtro ampliado vs v16: v16 deployada filtrava apenas `status='pending'`. v6 amplia para `('active', 'pending', 'rejected')` em coerência com soft delete e princípio "1 registro por canônico ao longo do tempo"
- `source IS DISTINCT FROM` no UPDATE da `retire_canonical` é defesa em profundidade caso o trigger falhe ao detectar source

## 2.10 — `10_reset_taxonomy_core_alter.sql`

```sql
BEGIN;

CREATE OR REPLACE FUNCTION internal.reset_taxonomy_core(
  p_zombie_ids uuid[],
  p_deprecated_ids uuid[],
  p_affected_job_ids uuid[]
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $function$
DECLARE
    v_canonicals_deleted INT := 0;
    v_jobs_reset INT := 0;
    v_canonical_skills_deleted INT := 0;
    v_skill_merge_decisions_deleted INT := 0;
    v_job_posting_skills_deleted INT := 0;
    v_function_orchestrator_runs_deleted INT := 0;
    v_resumes_deleted INT := 0;
    v_taxonomy_versions_deleted INT := 0;
    v_canonical_merge_candidates_deleted INT := 0;
    v_description_hashes_zeroed INT := 0;
    v_taxonomy_content_version_zeroed INT := 0;
    v_canonicals_redirects_zeroed INT := 0;
    v_opus_arbitration_outcomes_deleted INT := 0;
BEGIN
    IF p_zombie_ids && p_deprecated_ids THEN
        RAISE EXCEPTION 'array_overlap entre p_zombie_ids e p_deprecated_ids';
    END IF;

    -- V5-M1: DELETE direto em vez de EXECUTE com string literal
    DELETE FROM function_orchestrator_items WHERE id IS NOT NULL;
    DELETE FROM job_canonical_role_sources WHERE canonical_role_id IS NOT NULL;

    -- V5-H4: DELETE em vez de TRUNCATE (proibido em SECURITY DEFINER + transação)
    DELETE FROM role_merge_decisions WHERE id IS NOT NULL;

    DELETE FROM taxonomy_family_canonicals
     WHERE canonical_role_id IN (
       SELECT id FROM job_canonical_roles
       WHERE source IS DISTINCT FROM 'cbo_mte_2002_seed'
     );

    DELETE FROM taxonomy_families WHERE cbo_family_code IS NULL;

    DELETE FROM taxonomy_relations WHERE cbo_version_id IS NULL;

    DELETE FROM canonical_role_domain_links WHERE id IS NOT NULL;
    DELETE FROM curation_batch_metrics WHERE id IS NOT NULL;
    DELETE FROM job_no_postings WHERE id IS NOT NULL;
    DELETE FROM skill_enrichment_stats WHERE id IS NOT NULL;
    DELETE FROM resume_skill_enrichments WHERE id IS NOT NULL;
    DELETE FROM resume_role_suggestions WHERE id IS NOT NULL;

    UPDATE job_canonical_roles
       SET merged_into=NULL, alias_of_id=NULL
     WHERE (merged_into IS NOT NULL OR alias_of_id IS NOT NULL)
       AND (
         source IS DISTINCT FROM 'cbo_mte_2002_seed'
         OR
         (source = 'cbo_mte_2002_seed' AND merged_into IN (
           SELECT id FROM job_canonical_roles
           WHERE source IS DISTINCT FROM 'cbo_mte_2002_seed'
         ))
         OR
         (source = 'cbo_mte_2002_seed' AND alias_of_id IN (
           SELECT id FROM job_canonical_roles
           WHERE source IS DISTINCT FROM 'cbo_mte_2002_seed'
         ))
       );
    GET DIAGNOSTICS v_canonicals_redirects_zeroed = ROW_COUNT;

    DELETE FROM job_posting_skills WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_job_posting_skills_deleted = ROW_COUNT;

    DELETE FROM skill_merge_decisions WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_skill_merge_decisions_deleted = ROW_COUNT;

    UPDATE canonical_skills SET merged_into=NULL WHERE merged_into IS NOT NULL;

    DELETE FROM canonical_skills WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_canonical_skills_deleted = ROW_COUNT;

    DELETE FROM canonical_skills_summary WHERE id IS NOT NULL;
    DELETE FROM canonical_seniority_distribution WHERE id IS NOT NULL;

    DELETE FROM function_orchestrator_runs WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_function_orchestrator_runs_deleted = ROW_COUNT;

    DELETE FROM resumes WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_resumes_deleted = ROW_COUNT;

    UPDATE job_postings SET description_hash=NULL WHERE description_hash IS NOT NULL;
    GET DIAGNOSTICS v_description_hashes_zeroed = ROW_COUNT;

    DELETE FROM canonical_merge_candidates WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_canonical_merge_candidates_deleted = ROW_COUNT;

    UPDATE job_postings SET taxonomy_content_version_id = NULL
     WHERE taxonomy_content_version_id IS NOT NULL;
    GET DIAGNOSTICS v_taxonomy_content_version_zeroed = ROW_COUNT;

    DELETE FROM taxonomy_versions WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_taxonomy_versions_deleted = ROW_COUNT;

    UPDATE job_postings
       SET canonical_role_id=NULL, curation_status='pending', curated_at=NULL
     WHERE id = ANY(p_affected_job_ids)
       AND (canonical_role_id IS NOT NULL OR curated_at IS NOT NULL OR curation_status <> 'pending');
    GET DIAGNOSTICS v_jobs_reset = ROW_COUNT;

    DELETE FROM job_canonical_roles WHERE source IS DISTINCT FROM 'cbo_mte_2002_seed';
    GET DIAGNOSTICS v_canonicals_deleted = ROW_COUNT;

    -- V5-B3: regressão da v5.6 corrigida em v6.
    -- Sem este DELETE, OAO acumula rows que contaminam v_opus_effectiveness pós-reset.
    DELETE FROM opus_arbitration_outcomes WHERE id IS NOT NULL;
    GET DIAGNOSTICS v_opus_arbitration_outcomes_deleted = ROW_COUNT;

    DELETE FROM pipeline_calibration_metrics WHERE id IS NOT NULL;

    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
        'sprint_total_reset_executed',
        'system',
        '00000000-0000-0000-0000-000000000001',
        'sprint',
        gen_random_uuid(),
        jsonb_build_object(
            'version', 'v6.20-cbo-aware',
            'canonicals_deleted', v_canonicals_deleted,
            'jobs_reset', v_jobs_reset,
            'canonical_skills_deleted', v_canonical_skills_deleted,
            'skill_merge_decisions_deleted', v_skill_merge_decisions_deleted,
            'job_posting_skills_deleted', v_job_posting_skills_deleted,
            'function_orchestrator_runs_deleted', v_function_orchestrator_runs_deleted,
            'resumes_deleted', v_resumes_deleted,
            'taxonomy_versions_deleted', v_taxonomy_versions_deleted,
            'canonical_merge_candidates_deleted', v_canonical_merge_candidates_deleted,
            'description_hashes_zeroed', v_description_hashes_zeroed,
            'taxonomy_content_version_zeroed', v_taxonomy_content_version_zeroed,
            'canonicals_redirects_zeroed', v_canonicals_redirects_zeroed,
            'opus_arbitration_outcomes_deleted', v_opus_arbitration_outcomes_deleted,
            'cbo_seeds_preserved', true,
            'replay_defense', true
        )
    );

    RETURN jsonb_build_object(
        'canonicals_deleted', v_canonicals_deleted,
        'jobs_reset', v_jobs_reset,
        'canonical_skills_deleted', v_canonical_skills_deleted,
        'skill_merge_decisions_deleted', v_skill_merge_decisions_deleted,
        'job_posting_skills_deleted', v_job_posting_skills_deleted,
        'function_orchestrator_runs_deleted', v_function_orchestrator_runs_deleted,
        'resumes_deleted', v_resumes_deleted,
        'taxonomy_versions_deleted', v_taxonomy_versions_deleted,
        'canonical_merge_candidates_deleted', v_canonical_merge_candidates_deleted,
        'description_hashes_zeroed', v_description_hashes_zeroed,
        'taxonomy_content_version_zeroed', v_taxonomy_content_version_zeroed,
        'canonicals_redirects_zeroed', v_canonicals_redirects_zeroed,
        'opus_arbitration_outcomes_deleted', v_opus_arbitration_outcomes_deleted,
        'cbo_seeds_preserved', true
    );
END;
$function$;

COMMIT;
```

## 2.11 — `11_process_opus_create_new.sql`

```sql
BEGIN;

CREATE OR REPLACE FUNCTION process_opus_create_new(
  p_relation_id UUID,
  p_loser_canonical_id UUID,
  p_new_label TEXT,
  p_new_slug TEXT,
  p_confidence NUMERIC,
  p_reason TEXT,
  p_suggested_cbo_code TEXT DEFAULT NULL,
  p_suggested_cbo_confidence NUMERIC DEFAULT NULL
)
RETURNS UUID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_new_id UUID;
  v_relation_target_was UUID;
  v_loser_source TEXT;
  v_loser_status TEXT;
  v_existing_slug_canonical_id UUID;
  v_existing_slug_status TEXT;
  v_final_slug TEXT;
  v_cbo_exists BOOLEAN;
BEGIN
  IF p_confidence < 0 OR p_confidence > 1 THEN
    RAISE EXCEPTION 'p_confidence fora do range [0, 1]: %', p_confidence;
  END IF;

  IF p_suggested_cbo_confidence IS NOT NULL AND
     (p_suggested_cbo_confidence < 0 OR p_suggested_cbo_confidence > 1) THEN
    RAISE EXCEPTION 'p_suggested_cbo_confidence fora do range [0, 1]: %', p_suggested_cbo_confidence;
  END IF;

  -- V5-B4: validar pré-condições ANTES de mutar estado.
  -- Lock pessimista na relation para detectar drift concorrente.
  SELECT target_canonical_id INTO v_relation_target_was
  FROM taxonomy_relations
  WHERE id = p_relation_id
  FOR UPDATE;

  IF v_relation_target_was IS DISTINCT FROM p_loser_canonical_id THEN
    -- Drift detectado: relation já não aponta para loser. Aborta sem mutar.
    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
      'opus_create_new_target_drift',
      'system',
      '00000000-0000-0000-0000-000000000001',
      'taxonomy_relation',
      p_relation_id,
      jsonb_build_object(
        'expected_target', p_loser_canonical_id,
        'actual_target', v_relation_target_was,
        'reason', 'Concurrent state change between Opus decision and RPC execution. No mutation performed.'
      )
    );
    RETURN NULL;
  END IF;

  -- V5-M5: guard contra depreciar placeholder CBO via CREATE_NEW.
  -- Decisão arquitetural: camada institucional CBO não pode ser enfraquecida por essa via.
  SELECT source, status INTO v_loser_source, v_loser_status
  FROM job_canonical_roles
  WHERE id = p_loser_canonical_id
  FOR UPDATE;

  IF v_loser_source = 'cbo_mte_2002_seed' THEN
    RAISE EXCEPTION 'Não é permitido depreciar placeholder CBO via CREATE_NEW. Loser: %', p_loser_canonical_id;
  END IF;

  IF v_loser_status IS NULL THEN
    RAISE EXCEPTION 'Loser canonical % não encontrado', p_loser_canonical_id;
  END IF;

  -- V5-B7: tratar conflito de slug.
  -- Se slug pertence a 'retired', ressuscita (alinhado com princípio "1 registro por canônico").
  -- Se slug pertence a 'rejected'/'deprecated'/'alias_of', sufixa com fragmento do UUID.
  -- Se slug livre, usa direto.
  SELECT id, status INTO v_existing_slug_canonical_id, v_existing_slug_status
  FROM job_canonical_roles
  WHERE slug = p_new_slug
  FOR UPDATE;

  IF v_existing_slug_canonical_id IS NULL THEN
    -- Slug livre: insere normalmente
    v_final_slug := p_new_slug;
    INSERT INTO job_canonical_roles (label, slug, status, source, confidence_median)
    VALUES (p_new_label, v_final_slug, 'active', 'llm_extractor', p_confidence)
    RETURNING id INTO v_new_id;

  ELSIF v_existing_slug_status = 'retired' THEN
    -- Slug ocupado por canônico aposentado: ressuscita em vez de criar novo.
    -- Coerente com D-CBO-24 e princípio "1 registro por canônico ao longo do tempo".
    UPDATE job_canonical_roles
    SET status = 'active',
        label = p_new_label,
        confidence_median = p_confidence,
        promoted_at = COALESCE(promoted_at, NOW())
    WHERE id = v_existing_slug_canonical_id;
    v_new_id := v_existing_slug_canonical_id;

    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
      'opus_resurrected_via_create_new',
      'system',
      '00000000-0000-0000-0000-000000000001',
      'job_canonical_role',
      v_new_id,
      jsonb_build_object(
        'slug_collision', p_new_slug,
        'previous_status', 'retired',
        'note', 'Opus propôs CREATE_NEW mas slug coincide com retired. Ressuscitado.'
      )
    );

  ELSE
    -- Slug ocupado por canônico ativo/rejeitado/etc: sufixa com fragmento UUID
    v_final_slug := p_new_slug || '-' || substr(gen_random_uuid()::TEXT, 1, 8);
    INSERT INTO job_canonical_roles (label, slug, status, source, confidence_median)
    VALUES (p_new_label, v_final_slug, 'active', 'llm_extractor', p_confidence)
    RETURNING id INTO v_new_id;

    INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
    VALUES (
      'opus_create_new_slug_suffixed',
      'system',
      '00000000-0000-0000-0000-000000000001',
      'job_canonical_role',
      v_new_id,
      jsonb_build_object(
        'original_slug', p_new_slug,
        'final_slug', v_final_slug,
        'colliding_canonical_id', v_existing_slug_canonical_id,
        'colliding_canonical_status', v_existing_slug_status
      )
    );
  END IF;

  -- Agora sim: muta loser e relations
  UPDATE job_canonical_roles
  SET status = 'deprecated', merged_into = v_new_id
  WHERE id = p_loser_canonical_id;

  UPDATE taxonomy_relations
  SET target_canonical_id = v_new_id
  WHERE target_canonical_id = p_loser_canonical_id;

  UPDATE taxonomy_relations
  SET status = 'active',
      target_canonical_id = v_new_id,
      validated_at = NOW(),
      validated_by = 'opus_4_7',
      opus_decision_reason = p_reason,
      opus_verdict = 'CREATE_NEW'
  WHERE id = p_relation_id;

  -- Vínculo CBO opcional
  IF p_suggested_cbo_code IS NOT NULL AND p_suggested_cbo_confidence >= 0.70 THEN
    SELECT EXISTS (
      SELECT 1 FROM canonical_role_cbo
      WHERE occupation_code = p_suggested_cbo_code
    ) INTO v_cbo_exists;

    IF v_cbo_exists THEN
      INSERT INTO canonical_role_cbo_links (
        canonical_role_id, occupation_code, is_primary, confidence, source
      ) VALUES (
        v_new_id, p_suggested_cbo_code, true, p_suggested_cbo_confidence, 'opus_arbitration'
      )
      ON CONFLICT (canonical_role_id, occupation_code) DO NOTHING;
    END IF;
  END IF;

  INSERT INTO events (
    event_name, resource_type, resource_id,
    actor, actor_id, previous_state, new_state, reason
  )
  VALUES (
    'opus_created_new_canonical',
    'job_canonical_role',
    v_new_id,
    'system',
    '00000000-0000-0000-0000-000000000001',
    jsonb_build_object('loser_canonical_id', p_loser_canonical_id),
    jsonb_build_object(
      'new_canonical_id', v_new_id,
      'label', p_new_label,
      'final_slug', v_final_slug,
      'confidence', p_confidence,
      'cbo_link_created', (p_suggested_cbo_code IS NOT NULL AND v_cbo_exists)
    ),
    p_reason
  );

  RETURN v_new_id;
END;
$$;

REVOKE EXECUTE ON FUNCTION process_opus_create_new(UUID, UUID, TEXT, TEXT, NUMERIC, TEXT, TEXT, NUMERIC) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION process_opus_create_new(UUID, UUID, TEXT, TEXT, NUMERIC, TEXT, TEXT, NUMERIC) FROM anon;
GRANT EXECUTE ON FUNCTION process_opus_create_new(UUID, UUID, TEXT, TEXT, NUMERIC, TEXT, TEXT, NUMERIC) TO service_role;

-- V5-H3: pgvector com CTE.
-- TOP-K via ORDER BY+LIMIT (índice HNSW), depois filtro de threshold.
-- Padrão recomendado para pgvector + filtro booleano de distância.
CREATE OR REPLACE FUNCTION fetch_cbo_candidates(
  p_embedding VECTOR(768),
  p_limit INT DEFAULT 5,
  p_min_similarity NUMERIC DEFAULT 0.70
)
RETURNS TABLE (
  occupation_code TEXT,
  title TEXT,
  cosine_similarity NUMERIC
) LANGUAGE plpgsql STABLE AS $$
BEGIN
  RETURN QUERY
  WITH top_k AS (
    SELECT
      crc.occupation_code,
      crc.title,
      (1 - (crc.embedding <=> p_embedding))::NUMERIC AS cosine_similarity
    FROM canonical_role_cbo crc
    WHERE crc.embedding IS NOT NULL
    ORDER BY crc.embedding <=> p_embedding
    LIMIT p_limit
  )
  SELECT t.occupation_code, t.title, t.cosine_similarity
  FROM top_k t
  WHERE t.cosine_similarity >= p_min_similarity;
END;
$$;

REVOKE EXECUTE ON FUNCTION fetch_cbo_candidates(VECTOR, INT, NUMERIC) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION fetch_cbo_candidates(VECTOR, INT, NUMERIC) TO service_role;

CREATE OR REPLACE FUNCTION fetch_family_candidates(
  p_embedding VECTOR(768),
  p_limit INT DEFAULT 5,
  p_min_similarity NUMERIC DEFAULT 0.85
)
RETURNS TABLE (
  family_id UUID,
  slug TEXT,
  display_name TEXT,
  cbo_family_code TEXT,
  cosine_similarity NUMERIC
) LANGUAGE plpgsql STABLE AS $$
BEGIN
  RETURN QUERY
  WITH top_k AS (
    SELECT
      tf.id AS family_id,
      tf.slug,
      tf.display_name,
      tf.cbo_family_code,
      (1 - (tf.embedding <=> p_embedding))::NUMERIC AS cosine_similarity
    FROM taxonomy_families tf
    WHERE tf.embedding IS NOT NULL
      AND tf.status = 'active'
    ORDER BY tf.embedding <=> p_embedding
    LIMIT p_limit
  )
  SELECT t.family_id, t.slug, t.display_name, t.cbo_family_code, t.cosine_similarity
  FROM top_k t
  WHERE t.cosine_similarity >= p_min_similarity;
END;
$$;

REVOKE EXECUTE ON FUNCTION fetch_family_candidates(VECTOR, INT, NUMERIC) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION fetch_family_candidates(VECTOR, INT, NUMERIC) TO service_role;

COMMIT;
```

Notas:

- **V5-B4:** `SELECT ... FOR UPDATE` na relation no início detecta drift antes de qualquer mutação. Se outro processo já mexeu, RPC retorna NULL sem efeito colateral. Padrão "validar primeiro, mutar depois"
- **V5-B7:** 3 caminhos para slug: livre (insere), retired (ressuscita), outro status (sufixa com 8 chars do UUID). Eventos distintos para auditoria
- **V5-H3:** CTE `top_k` força planner a usar índice HNSW para ORDER BY+LIMIT primeiro, depois aplica filtro de threshold no resultado. Evita Sequential Scan
- **V5-M5:** guard contra `source='cbo_mte_2002_seed'` no loser. Camada institucional CBO não pode ser depreciada por CREATE_NEW (preserva D-CBO-01)

## 2.12 — `12_replace_cbo_link_rpc.sql`

```sql
BEGIN;

CREATE OR REPLACE FUNCTION replace_cbo_link(
  p_canonical_role_id UUID,
  p_old_occupation_code TEXT,
  p_new_occupation_code TEXT,
  p_new_confidence NUMERIC,
  p_reason TEXT
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_new_cbo_exists BOOLEAN;
BEGIN
  IF p_new_confidence < 0 OR p_new_confidence > 1 THEN
    RAISE EXCEPTION 'p_new_confidence fora do range [0, 1]: %', p_new_confidence;
  END IF;

  -- V5-H2: validação por DELETE+FOUND em vez de anti-pattern SELECT EXISTS+DELETE.
  -- Postgres adquire lock no DELETE; instâncias concorrentes serializam naturalmente.
  DELETE FROM canonical_role_cbo_links
   WHERE canonical_role_id = p_canonical_role_id
     AND occupation_code = p_old_occupation_code;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Link antigo não existe: canonical=%, code=%',
                    p_canonical_role_id, p_old_occupation_code;
  END IF;

  SELECT EXISTS (
    SELECT 1 FROM canonical_role_cbo
    WHERE occupation_code = p_new_occupation_code
  ) INTO v_new_cbo_exists;

  IF NOT v_new_cbo_exists THEN
    RAISE EXCEPTION 'Ocupação CBO destino não existe: %', p_new_occupation_code;
  END IF;

  -- V5-B8: normalizar is_primary=FALSE em todos os links restantes do canônico
  -- ANTES de inserir o novo como primary. Evita violação do índice parcial
  -- idx_crcl_one_primary_per_canonical em cenários N:M.
  UPDATE canonical_role_cbo_links
  SET is_primary = FALSE
  WHERE canonical_role_id = p_canonical_role_id
    AND is_primary = TRUE;

  -- V5-B8: INSERT com ON CONFLICT DO UPDATE para idempotência.
  -- Se novo código já era link não-primary do canônico, vira primary.
  INSERT INTO canonical_role_cbo_links (
    canonical_role_id, occupation_code, is_primary, confidence, source
  ) VALUES (
    p_canonical_role_id, p_new_occupation_code, true, p_new_confidence, 'opus_arbitration'
  )
  ON CONFLICT (canonical_role_id, occupation_code)
  DO UPDATE SET
    is_primary = TRUE,
    confidence = EXCLUDED.confidence,
    source = EXCLUDED.source;

  INSERT INTO events (event_name, actor, actor_id, resource_type, resource_id, metadata)
  VALUES (
    'opus_replaced_cbo_link',
    'system',
    '00000000-0000-0000-0000-000000000001',
    'canonical_role_cbo_link',
    p_canonical_role_id,
    jsonb_build_object(
      'canonical_role_id', p_canonical_role_id,
      'old_occupation_code', p_old_occupation_code,
      'new_occupation_code', p_new_occupation_code,
      'new_confidence', p_new_confidence,
      'reason', p_reason
    )
  );
END;
$$;

REVOKE EXECUTE ON FUNCTION replace_cbo_link(UUID, TEXT, TEXT, NUMERIC, TEXT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION replace_cbo_link(UUID, TEXT, TEXT, NUMERIC, TEXT) TO service_role;

COMMIT;
```

Notas:

- **V5-H2:** `DELETE ... ; IF NOT FOUND THEN RAISE` é atomicamente seguro vs `SELECT EXISTS+DELETE`. PostgreSQL serializa via lock natural do DELETE. Duas instâncias concorrentes não conseguem ambas confirmar existência e ambas deletar
- **V5-B8:** UPDATE prévio garante zero links com `is_primary=TRUE` antes do INSERT. Necessário para honrar `idx_crcl_one_primary_per_canonical` (índice parcial que limita 1 primary por canônico)
- **V5-B8:** `ON CONFLICT DO UPDATE` torna a função idempotente: se Opus chamar 2x com mesmo `p_new_occupation_code`, segunda chamada apenas reafirma estado sem erro
- Event log final fica DEPOIS das mutações (não antes como na v5), garantindo que só auditamos operações que de fato aconteceram

## 2.13 — `13_pipeline_calibration_metrics.sql`

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS pipeline_calibration_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id UUID,
  batch_id UUID,
  batch_size INT,
  mode TEXT CHECK (mode IN ('coarse', 'fine', 'bulk', 'mvp', 'production')),
  metric_name TEXT NOT NULL,
  metric_value NUMERIC,
  metric_value_text TEXT,
  dimensions JSONB NOT NULL DEFAULT '{}',
  computed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_pcm_metric_name
  ON pipeline_calibration_metrics(metric_name, computed_at DESC);

CREATE INDEX IF NOT EXISTS idx_pcm_run
  ON pipeline_calibration_metrics(run_id, batch_size)
  WHERE run_id IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_pcm_mode_metric
  ON pipeline_calibration_metrics(mode, metric_name, computed_at DESC);

ALTER TABLE pipeline_calibration_metrics ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE pipeline_calibration_metrics IS
  'Infraestrutura viva de calibragem do pipeline. Acomoda métricas de benchmark, MVP e produção sem backfill de dados anteriores. Append-only sem UNIQUE constraint.';

COMMIT;
```

## 2.14 — `14_benchmark_runs_alter.sql`

```sql
BEGIN;

ALTER TABLE benchmark_runs
  DROP CONSTRAINT IF EXISTS benchmark_runs_mode_check;

ALTER TABLE benchmark_runs
  ADD CONSTRAINT benchmark_runs_mode_check
  CHECK (mode IN ('coarse', 'fine', 'bulk'));

ALTER TABLE benchmark_runs
  ADD COLUMN IF NOT EXISTS retry_index INT,
  ADD COLUMN IF NOT EXISTS retry_role TEXT,
  ADD COLUMN IF NOT EXISTS cache_control_enabled BOOLEAN,
  ADD COLUMN IF NOT EXISTS discard BOOLEAN DEFAULT FALSE,
  ADD COLUMN IF NOT EXISTS replicate_group_id UUID;

ALTER TABLE benchmark_runs
  DROP CONSTRAINT IF EXISTS chk_benchmark_runs_retry_role;

ALTER TABLE benchmark_runs
  ADD CONSTRAINT chk_benchmark_runs_retry_role
    CHECK (retry_role IS NULL OR retry_role IN ('baseline_no_cache', 'warmup', 'hot_cache_valid'));

ALTER TABLE benchmark_runs
  DROP CONSTRAINT IF EXISTS chk_benchmark_runs_retry_index;

ALTER TABLE benchmark_runs
  ADD CONSTRAINT chk_benchmark_runs_retry_index
    CHECK (retry_index IS NULL OR (retry_index >= 1 AND retry_index <= 50));

CREATE INDEX IF NOT EXISTS idx_benchmark_runs_aggregation
  ON benchmark_runs(run_id, mode, batch_size, retry_role)
  WHERE retry_role = 'hot_cache_valid';

COMMENT ON COLUMN benchmark_runs.retry_index IS
  'Indice da repeticao dentro do mesmo (run_id, mode, batch_size). 1 a 7 no padrao atual.';
COMMENT ON COLUMN benchmark_runs.retry_role IS
  'Papel da medicao: baseline_no_cache, warmup, hot_cache_valid.';
COMMENT ON COLUMN benchmark_runs.cache_control_enabled IS
  'TRUE se a chamada incluiu cache_control: ephemeral.';
COMMENT ON COLUMN benchmark_runs.discard IS
  'TRUE se esta linha NAO deve ser incluida em medias estatisticas.';
COMMENT ON COLUMN benchmark_runs.replicate_group_id IS
  'UUID compartilhado pelas 7 rodadas do mesmo (run_id, mode, batch_size).';

COMMIT;
```

Sem backfill: dados anteriores ficam com `retry_index=NULL` e são excluídos das agregações pós-refactor.

---

# Parte 3 — Mudanças em arquivos existentes

## 3.1 — `lib/pipeline/opus-arbitration-prompts.ts`

Schema do `taxonomy_relation` ganha verdict `CREATE_NEW` e campo `suggested_cbo` com flag `replace_existing`:

```typescript
taxonomy_relation: {
  system: `Você é Opus, auditor de taxonomia do CalibraCV. Analise se a relação proposta entre o termo de origem e o canônico alvo é semanticamente correta.

Critérios:
- APPROVE: termo realmente é variante, sinônimo ou tradução do canônico alvo
- DISAGREE_MERGE: o canônico alvo deve ser mesclado com outro canônico mais apropriado
- DISAGREE_LABEL: o label do canônico alvo está incorreto. Sugerir novo label
- REJECT: a relação é falsa ou enganosa
- CREATE_NEW: o canônico alvo é semanticamente inadequado para a vaga, e nenhum outro canônico existente cobre a função real. Criar canônico novo com label apropriado. Use somente quando confidence maior ou igual a 0.85 e nem APPROVE, DISAGREE_MERGE, nem DISAGREE_LABEL servem.

Confidence: 0 a 1, sua certeza na decisão. Decisões destrutivas (DISAGREE_MERGE, CREATE_NEW) requerem confidence maior ou igual a 0.85.

Quando o canônico atual está vinculado a uma ocupação CBO 2002 do MTE, considere se faz sentido manter esse vínculo. Se a função real da vaga corresponde a uma ocupação CBO diferente, indique via campo suggested_cbo. Se o canônico já tem link CBO e você quer substituir por outro, marque suggested_cbo.replace_existing como true.`,
  tool: {
    name: 'audit_taxonomy_relation',
    input_schema: {
      type: 'object',
      properties: {
        verdict: {
          type: 'string',
          enum: ['APPROVE', 'DISAGREE_MERGE', 'DISAGREE_LABEL', 'REJECT', 'CREATE_NEW']
        },
        confidence: { type: 'number', minimum: 0, maximum: 1 },
        reasoning: { type: 'string' },
        suggested_relations: { /* mantido */ },
        suggested_family: { /* mantido */ },
        suggested_domain: { /* mantido */ },
        suggested_linguistic_category: {
          type: 'string',
          enum: ['acronym', 'translation', 'orthographic', 'slang', 'specialty',
                 'domain_alias', 'synonym', 'other']
        },
        suggested_cbo: {
          type: ['object', 'null'],
          description: 'Quando o canônico merece vínculo CBO próprio, indique aqui',
          properties: {
            occupation_code: { type: 'string', pattern: '^\\d{4}-\\d{2}$' },
            confidence: { type: 'number', minimum: 0, maximum: 1 },
            replace_existing: {
              type: 'boolean',
              description: 'true para substituir link CBO existente, false (ou ausente) para apenas adicionar quando não houver link'
            }
          }
        },
        suggested_label: { type: 'string' },
        suggested_merge_target_id: { type: 'string' }
      },
      required: ['verdict', 'confidence', 'reasoning']
    }
  }
}
```

Schema do `orphan_canonical` ganha `suggested_cbo` análogo (sem `replace_existing` porque canônico recém-promovido nunca tem link CBO prévio).

Schema novo `family_replica` para réplica Opus em `findOrCreateFamily`:

```typescript
family_replica: {
  system: `Você é Opus, decidindo se uma família semanticamente próxima já existe ou se uma nova deve ser criada.

Você sugeriu criar a família "{proposed_display_name}" (slug: {proposed_slug}).
Encontrei candidato existente: "{candidate_display_name}" (slug: {candidate_slug}, origem: {candidate_origin}, similaridade cosine: {cosine_similarity}).

A origem CBO indica família proveniente do Ministério do Trabalho e Emprego (MTE). Famílias CBO devem ser preferidas quando semanticamente próximas, por serem âncora institucional.

Decida: reusar a família candidata existente ou criar a nova proposta.`,
  tool: {
    name: 'decide_family_reuse',
    input_schema: {
      type: 'object',
      properties: {
        decision: {
          type: 'string',
          enum: ['REUSE_CANDIDATE', 'CREATE_NEW']
        },
        confidence: { type: 'number', minimum: 0, maximum: 1 },
        reasoning: { type: 'string' }
      },
      required: ['decision', 'confidence', 'reasoning']
    }
  }
}
```

## 3.2 — `lib/pipeline/opus-arbitration.ts`

`processOpusTaxonomyRelationVerdict` ganha tratamento de `CREATE_NEW`, `suggested_cbo` com flag `replace_existing`. Toda decisão registra row em PCM:

```typescript
case 'CREATE_NEW':
  if (opusResponse.suggested_label?.trim() && confidence >= 0.85) {
    const cboCandidates = await fetchCboCandidates(supabase, canonical.embedding);
    let suggestedCboCode: string | null = null;
    let suggestedCboConfidence: number | null = null;

    if (
      opusResponse.suggested_cbo &&
      opusResponse.suggested_cbo.confidence >= 0.70 &&
      cboCandidates.length > 0
    ) {
      const candidate = cboCandidates.find(
        c => c.occupation_code === opusResponse.suggested_cbo!.occupation_code
      );
      if (candidate) {
        suggestedCboCode = opusResponse.suggested_cbo.occupation_code;
        suggestedCboConfidence = opusResponse.suggested_cbo.confidence;
      }
    }

    await supabase.rpc('process_opus_create_new', {
      p_relation_id: relationId,
      p_loser_canonical_id: canonical.id,
      p_new_label: opusResponse.suggested_label.trim(),
      p_new_slug: toSlug(opusResponse.suggested_label.trim()),
      p_confidence: confidence,
      p_reason: reasoning,
      p_suggested_cbo_code: suggestedCboCode,
      p_suggested_cbo_confidence: suggestedCboConfidence,
    });

    await persistCalibrationMetric({
      mode: 'mvp',
      metric_name: 'opus_create_new_confidence',
      metric_value: confidence,
      dimensions: { canonical_id: canonical.id, has_cbo_link: !!suggestedCboCode }
    });
  } else {
    await supabase.from('taxonomy_relations').update({
      status: 'disagreed_pending_review',
      opus_verdict: 'CREATE_NEW',
      opus_decision_reason: reasoning + ' [confidence insuficiente ou label vazio]',
    }).eq('id', relationId);
  }
  break;

case 'APPROVE':
  if (opusResponse.suggested_cbo && opusResponse.suggested_cbo.confidence >= 0.70) {
    const cboCandidates = await fetchCboCandidates(supabase, canonical.embedding);

    // V5-M6: registrar quando candidatos vazios (mascara falha de embedding)
    if (cboCandidates.length === 0) {
      await persistCalibrationMetric({
        mode: 'mvp',
        metric_name: 'opus_suggested_cbo_no_candidate',
        dimensions: {
          canonical_id: canonical.id,
          suggested_code: opusResponse.suggested_cbo.occupation_code,
          reason: 'fetch_cbo_candidates returned empty; possibly missing embedding',
        },
      });
    }

    if (cboCandidates.length > 0) {
      const candidate = cboCandidates.find(
        c => c.occupation_code === opusResponse.suggested_cbo!.occupation_code
      );

      if (candidate) {
        // V5-B5: estrutura é N:M, então buscar lista (não .maybeSingle()).
        // Ordenação: is_primary=true primeiro, depois mais antigo.
        const { data: existingLinks } = await supabase
          .from('canonical_role_cbo_links')
          .select('id, occupation_code, source, is_primary')
          .eq('canonical_role_id', canonical.id)
          .order('is_primary', { ascending: false })
          .order('created_at', { ascending: true });

        const linkCount = existingLinks?.length ?? 0;
        const primaryLink = existingLinks?.find(l => l.is_primary) ?? null;

        // Matriz de comportamento N:M:
        // - 0 links: insere novo primary
        // - 1+ links E nenhum é o sugerido E replace_existing=true: substitui primary via replace_cbo_link
        // - 1+ links E o sugerido já existe: ignora (idempotente)
        // - 1+ links E nenhum é o sugerido E replace_existing=false: ignora com warning
        if (linkCount === 0) {
          await supabase.from('canonical_role_cbo_links').insert({
            canonical_role_id: canonical.id,
            occupation_code: opusResponse.suggested_cbo.occupation_code,
            is_primary: true,
            confidence: opusResponse.suggested_cbo.confidence,
            source: 'opus_arbitration',
          });

        } else {
          const suggestedAlreadyLinked = existingLinks!.some(
            l => l.occupation_code === opusResponse.suggested_cbo!.occupation_code
          );

          if (suggestedAlreadyLinked) {
            // Idempotente: nada a fazer
            await persistCalibrationMetric({
              mode: 'mvp',
              metric_name: 'opus_suggested_cbo_already_linked',
              dimensions: {
                canonical_id: canonical.id,
                suggested_code: opusResponse.suggested_cbo.occupation_code,
                existing_links_count: linkCount,
              },
            });

          } else if (opusResponse.suggested_cbo.replace_existing === true && primaryLink) {
            await supabase.rpc('replace_cbo_link', {
              p_canonical_role_id: canonical.id,
              p_old_occupation_code: primaryLink.occupation_code,
              p_new_occupation_code: opusResponse.suggested_cbo.occupation_code,
              p_new_confidence: opusResponse.suggested_cbo.confidence,
              p_reason: reasoning,
            });

          } else {
            console.warn(
              `Opus sugeriu CBO ${opusResponse.suggested_cbo.occupation_code} ` +
              `mas canônico ${canonical.id} já tem ${linkCount} link(s) ` +
              `(primary: ${primaryLink?.occupation_code ?? 'nenhum'}). ` +
              `Sem replace_existing=true, sugestão ignorada.`
            );
          }
        }

        await persistCalibrationMetric({
          mode: 'mvp',
          metric_name: 'opus_suggested_cbo_confidence',
          metric_value: opusResponse.suggested_cbo.confidence,
          dimensions: {
            canonical_id: canonical.id,
            existing_links_count: linkCount,
            replace_existing: opusResponse.suggested_cbo.replace_existing === true,
          },
        });
      }
    }
  }
  break;
```

Helpers `fetchCboCandidates`, `fetchFamilyCandidates`, `resolveCanonicalCbo`:

```typescript
async function fetchCboCandidates(
  supabase: SupabaseClient,
  embedding: number[] | null
): Promise<Array<{ occupation_code: string; title: string; cosine_similarity: number }>> {
  if (!embedding) return [];
  const { data } = await supabase.rpc('fetch_cbo_candidates', {
    p_embedding: embedding, p_limit: 5, p_min_similarity: 0.70,
  });
  return data ?? [];
}

async function fetchFamilyCandidates(
  supabase: SupabaseClient,
  embedding: number[] | null
): Promise<Array<{
  family_id: string; slug: string; display_name: string;
  cbo_family_code: string | null; cosine_similarity: number
}>> {
  if (!embedding) return [];
  const { data } = await supabase.rpc('fetch_family_candidates', {
    p_embedding: embedding, p_limit: 5, p_min_similarity: 0.85,
  });
  return data ?? [];
}

export async function resolveCanonicalCbo(
  supabase: SupabaseClient,
  canonicalId: string
): Promise<Array<{ occupation_code: string; is_primary: boolean }>> {
  const resolved = await resolveCanonicalById(supabase, canonicalId);
  if (!resolved) return [];

  const { data } = await supabase
    .from('canonical_role_cbo_links')
    .select('occupation_code, is_primary')
    .eq('canonical_role_id', resolved);

  return data ?? [];
}
```

## 3.3 — `lib/pipeline/find-or-create-family.ts`

Refactor completo para embedding cosine puro + réplica Opus + registro em PCM (D-CBO-26). **V5-P1:** threshold extraído para constante exportada, passada explicitamente para a RPC.

```typescript
import { generateEmbedding } from '@/lib/pipeline/embeddings';
import { callOpus } from '@/lib/pipeline/opus-call';
import { OPUS_PROMPTS } from '@/lib/pipeline/opus-arbitration-prompts';
import { persistCalibrationMetric } from '@/lib/pipeline/calibration-metrics';

// V5-P1: constante exportada compartilhada entre TS e RPC.
// Passada explicitamente para fetch_family_candidates evitando drift com default da RPC.
export const FAMILY_COSINE_THRESHOLD = 0.85;
export const FAMILY_CANDIDATE_LIMIT = 5;

export async function findOrCreateFamily(
  supabase: SupabaseClient,
  proposedSlug: string,
  proposedDisplayName: string,
  options: { runId?: string; batchSize?: number; mode?: string } = {}
): Promise<{ family_id: string; outcome: 'exact' | 'cosine_reuse' | 'opus_replica_reuse' | 'create_new' }> {

  const { data: exact } = await supabase
    .from('taxonomy_families')
    .select('id')
    .eq('slug', proposedSlug)
    .maybeSingle();

  if (exact) {
    await persistCalibrationMetric({
      run_id: options.runId, batch_size: options.batchSize, mode: options.mode ?? 'mvp',
      metric_name: 'find_or_create_family_outcome',
      metric_value_text: 'exact',
      dimensions: { proposed_slug: proposedSlug }
    });
    return { family_id: exact.id, outcome: 'exact' };
  }

  const embedding = await generateEmbedding(proposedDisplayName);

  // V5-P1: passa threshold explicitamente em vez de depender do default da RPC
  const candidates = await fetchFamilyCandidates(
    supabase,
    embedding,
    FAMILY_COSINE_THRESHOLD,
    FAMILY_CANDIDATE_LIMIT,
  );

  if (candidates.length === 0) {
    await persistCalibrationMetric({
      run_id: options.runId, batch_size: options.batchSize, mode: options.mode ?? 'mvp',
      metric_name: 'family_cosine_decision',
      metric_value: 0.0,
      metric_value_text: 'create_new_no_candidate',
      dimensions: { proposed_slug: proposedSlug, proposed_display_name: proposedDisplayName }
    });
    return await createFamily(supabase, proposedSlug, proposedDisplayName, embedding);
  }

  const bestCandidate = candidates[0];
  const bestCandidateIsCbo = bestCandidate.cbo_family_code !== null;

  const replicaResponse = await callOpus({
    system: OPUS_PROMPTS.family_replica.system
      .replace('{proposed_display_name}', proposedDisplayName)
      .replace('{proposed_slug}', proposedSlug)
      .replace('{candidate_display_name}', bestCandidate.display_name)
      .replace('{candidate_slug}', bestCandidate.slug)
      .replace('{candidate_origin}', bestCandidateIsCbo ? 'CBO MTE' : 'custom')
      .replace('{cosine_similarity}', bestCandidate.cosine_similarity.toFixed(3)),
    tools: [OPUS_PROMPTS.family_replica.tool],
  });

  await persistCalibrationMetric({
    run_id: options.runId, batch_size: options.batchSize, mode: options.mode ?? 'mvp',
    metric_name: 'family_cosine_decision',
    metric_value: bestCandidate.cosine_similarity,
    metric_value_text: replicaResponse.decision,
    dimensions: {
      proposed_slug: proposedSlug,
      candidate_slug: bestCandidate.slug,
      candidate_origin_cbo: bestCandidateIsCbo,
      opus_confidence: replicaResponse.confidence,
    }
  });

  if (replicaResponse.decision === 'REUSE_CANDIDATE') {
    return { family_id: bestCandidate.family_id, outcome: 'opus_replica_reuse' };
  }

  return await createFamily(supabase, proposedSlug, proposedDisplayName, embedding);
}

async function createFamily(
  supabase: SupabaseClient,
  slug: string,
  displayName: string,
  embedding: number[]
): Promise<{ family_id: string; outcome: 'create_new' }> {
  const { data } = await supabase.from('taxonomy_families').insert({
    slug,
    display_name: displayName,
    status: 'active',
    embedding,
  }).select('id').single();

  return { family_id: data!.id, outcome: 'create_new' };
}
```

## 3.4 — `lib/pipeline/taxonomy-cache.ts`

Estender filtros para incluir `'retired'` (D-CBO-15, D-CBO-24). **Atenção crítica a 2 filtros distintos** em `getRelations`. Confundir os dois quebra o sistema.

```typescript
// =============================================================================
// IMPORTANTE: getRelations() tem DOIS filtros de status com semânticas diferentes.
// =============================================================================
// Filtro 1 (linha ~99): .eq('status', 'active') em taxonomy_relations.status
//   - Coluna pertence à tabela TAXONOMY_RELATIONS
//   - Enum: ('active', 'pending_review', 'invalidated', 'archived', etc)
//   - NÃO TEM 'retired' (esse valor pertence a job_canonical_roles.status)
//   - NÃO ALTERAR este filtro. Manter .eq('status', 'active').
//
// Filtro 2 (linha ~112): role.status verificado em loop in-memory
//   - Acessa job_canonical_roles.status via join inline
//   - Enum: ('pending', 'active', 'deprecated', 'alias_of', 'rejected',
//            'merge_candidate', 'retired')
//   - ESTENDER este filtro para incluir 'retired'
// =============================================================================

// Função getRelations() (linhas ~99-114):

// MANTER inalterado:
const { data, error } = await supabase
  .from('taxonomy_relations')
  .select('source_term, target_canonical_id, job_canonical_roles!inner(label, status)')
  .eq('status', 'active');  // Filtro 1: taxonomy_relations.status (NÃO mexer)

// ALTERAR linha ~112 (filtro 2: in-memory de role.status):
// Antes:
//   if (role.status !== 'active' && role.status !== 'pending') continue;
// Depois:
if (role.status !== 'active' && role.status !== 'pending' && role.status !== 'retired') continue;

// =============================================================================
// Função getFullTaxonomyCache() (linhas ~388-394):
// =============================================================================
// Este filtro É em job_canonical_roles.status (que tem 'retired').
// ALTERAR de:
//   .in('status', ['active', 'pending'])
// para:
.in('status', ['active', 'pending', 'retired'])
.order('vacancy_count', { ascending: false })
.order('label', { ascending: true })
```

Sort de defesa anti-duplicata em `getFullTaxonomyCache` (linhas ~420-423): código exato:

```typescript
// statusPriority: maior valor = maior prioridade (active sobrescreve no Map)
const statusPriority: Record<string, number> = {
  retired: 0,   // menor prioridade
  pending: 1,
  active: 2,    // maior prioridade
};

canonicals.sort((a, b) =>
  (statusPriority[a.status] ?? 99) - (statusPriority[b.status] ?? 99)
);

// Map dedup por label: insertion order ascendente (retired primeiro, active último)
// active sobrescreve pending sobrescreve retired
const labelToCanonical = new Map<string, Canonical>();
for (const c of canonicals) {
  labelToCanonical.set(c.label, c);
}
```

## 3.5 — `lib/pipeline/suggested-roles-builder.ts`

Adicionar piso de 2 tokens em `findFamilyMatch` + tie-break CBO (D-CBO-25):

```typescript
function findFamilyMatch(
  normalizedTitle: string,
  families: Record<string, { isCbo: boolean; labels: string[] }>,
  validLabels: Set<string>,
  vacancyCount: Map<string, number>
): string[] | null {
  const uniqueTokens = new Set(normalizedTitle.split(/\s+/).filter(Boolean));

  let bestFamily: string | null = null;
  let bestScore = 0;
  let bestIsCbo = false;

  for (const [familyName, meta] of Object.entries(families)) {
    const familyTokens = getExpandedTokens(familyName);
    let overlap = 0;
    for (const t of uniqueTokens) {
      if (familyTokens.has(t)) overlap++;
    }

    if (overlap < 2) continue;

    const isBetter =
      overlap > bestScore ||
      (overlap === bestScore && meta.isCbo && !bestIsCbo);

    if (isBetter) {
      bestScore = overlap;
      bestFamily = familyName;
      bestIsCbo = meta.isCbo;
    }
  }

  if (!bestFamily) return null;

  return families[bestFamily].labels
    .filter(label => validLabels.has(label))
    .sort((a, b) => {
      // V5-M3: tiebreak alfabético quando vacancyCount empata.
      // Garante determinismo em volumes iguais.
      const diff = (vacancyCount.get(b) ?? 0) - (vacancyCount.get(a) ?? 0);
      if (diff !== 0) return diff;
      return a.localeCompare(b);
    });
}
```

Cache em `taxonomy-cache.ts:319-345` carrega flag `isCbo`:

```typescript
const familySynonyms: Record<string, { isCbo: boolean; labels: string[] }> = {};

for (const f of (families ?? []) as FamilyRow[]) {
    const labels: string[] = [];
    for (const link of f.taxonomy_family_canonicals ?? []) {
        const jr = link.job_canonical_roles;
        const role = Array.isArray(jr) ? jr[0] : jr;
        if (role?.label && ['active', 'pending', 'retired'].includes(role.status)) {
            labels.push(role.label);
        }
    }
    if (labels.length > 0) {
        familySynonyms[f.slug] = { isCbo: f.cbo_family_code !== null, labels };
    }
}
```

## 3.6 — `lib/pipeline/precheck-description-hash.ts`

Estender filtro `status IN ('active', 'pending')` para incluir `'retired'` (D-CBO-15):

```typescript
.in('status', ['active', 'pending', 'retired'])
```

## 3.7 — Renomeação `maintenance-zombie-detection.ts` → `maintenance-canonical-staleness.ts`

Arquivo renomeado. Conteúdo refatorado (D-CBO-30) preservando `MAX_ALERTS_PER_EXECUTION = 10` e helper `emitEventOnce` da v16:

```typescript
// lib/pipeline/maintenance-canonical-staleness.ts
import { SupabaseClient } from '@supabase/supabase-js';

export const STALE_AMBER_THRESHOLD_DAYS = 120;
export const STALE_RED_THRESHOLD_DAYS = 365;
const MAX_ALERTS_PER_EXECUTION = 10;
const SYSTEM_USER_ID = '00000000-0000-0000-0000-000000000001';

export async function detectStaleCanonicals(
  supabase: SupabaseClient
): Promise<{ amber_emitted: number; red_retired: number }> {
  const now = new Date();
  let amber = 0;
  let red = 0;

  // V5-H1: ORDER BY updated_at ASC garante varredura justa.
  // Sem ORDER BY, query retorna rows na ordem do disco (CTID), causando starvation:
  // primeiros 10 podem ser sempre os mesmos, canônicos no fim da tabela nunca seriam lidos.
  // Varrendo do mais antigo para o mais recente, criamos fila circular natural.
  const { data: canonicals } = await supabase
    .from('job_canonical_roles')
    .select('id, label, vacancy_count')
    .eq('status', 'active')
    .gt('vacancy_count', 0)
    .order('updated_at', { ascending: true });

  if (!canonicals) return { amber_emitted: 0, red_retired: 0 };

  for (const c of canonicals) {
    if (red >= MAX_ALERTS_PER_EXECUTION && amber >= MAX_ALERTS_PER_EXECUTION) break;

    const { data: lastVacancy } = await supabase
      .from('job_postings')
      .select('posted_at')
      .eq('canonical_role_id', c.id)
      .eq('curation_status', 'curated')
      .order('posted_at', { ascending: false })
      .limit(1)
      .maybeSingle();

    if (!lastVacancy?.posted_at) continue;

    const gapDays = Math.floor(
      (now.getTime() - new Date(lastVacancy.posted_at).getTime()) / (1000 * 60 * 60 * 24)
    );

    if (gapDays >= STALE_RED_THRESHOLD_DAYS && red < MAX_ALERTS_PER_EXECUTION) {
      // RED vira ação: aposenta via retire_canonical
      // Não usa emitEventOnce porque ação aposenta canônico, que sai naturalmente
      // do pool da próxima execução (filtro status='active' deixa de capturá-lo)
      await supabase.rpc('retire_canonical', {
        p_canonical_id: c.id,
        p_reason: 'no_recent_postings_365d',
      });
      red++;
    } else if (gapDays >= STALE_AMBER_THRESHOLD_DAYS && amber < MAX_ALERTS_PER_EXECUTION) {
      // AMBER continua emit, com dedup 30d via emitEventOnce
      await emitEventOnce(supabase, {
        event_name: 'canonical_stale_alert_amber',
        resource_type: 'job_canonical_role',
        resource_id: c.id,
        window_days: 30,
        payload: {
          label: c.label,
          gap_days: gapDays,
          last_vacancy_at: lastVacancy.posted_at,
        },
      });
      amber++;
    }
  }

  // Instrumentação em PCM (LK-CBO-17)
  await supabase.from('pipeline_calibration_metrics').insert({
    mode: 'production',
    metric_name: 'staleness_detection_outcome',
    metric_value: red,
    dimensions: {
      amber_emitted: amber,
      red_retired: red,
      cap_hit_red: red >= MAX_ALERTS_PER_EXECUTION,
      cap_hit_amber: amber >= MAX_ALERTS_PER_EXECUTION,
      total_active_canonicals_with_vacancies: canonicals.length,
    },
  });

  return { amber_emitted: amber, red_retired: red };
}

// Helper preservado da v16, sem mudanças funcionais
async function emitEventOnce(
  supabase: SupabaseClient,
  params: {
    event_name: string;
    resource_type: string;
    resource_id: string;
    window_days: number;
    payload: any;
  }
): Promise<void> {
  const cutoff = new Date(Date.now() - params.window_days * 24 * 60 * 60 * 1000).toISOString();

  const { data: existing } = await supabase
    .from('events')
    .select('id')
    .eq('event_name', params.event_name)
    .eq('resource_id', params.resource_id)
    .gte('created_at', cutoff)
    .limit(1)
    .maybeSingle();

  if (existing) return;

  await supabase.from('events').insert({
    event_name: params.event_name,
    resource_type: params.resource_type,
    resource_id: params.resource_id,
    actor: 'system',
    actor_id: SYSTEM_USER_ID,
    previous_state: {},
    new_state: params.payload,
  });
}

export async function gatherStalenessTelemetry(
  supabase: SupabaseClient
): Promise<{
  canonicals_active_total: number;
  canonicals_pending_total: number;
  canonicals_retired_total: number;
  amber_emitted_30d: number;
  red_retired_30d: number;
}> {
  const since = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();

  const [activeCount, pendingCount, retiredCount, amberCount, redRetiredCount] = await Promise.all([
    supabase.from('job_canonical_roles').select('*', { count: 'exact', head: true }).eq('status', 'active'),
    supabase.from('job_canonical_roles').select('*', { count: 'exact', head: true }).eq('status', 'pending'),
    supabase.from('job_canonical_roles').select('*', { count: 'exact', head: true }).eq('status', 'retired'),
    supabase.from('events').select('*', { count: 'exact', head: true })
      .eq('event_name', 'canonical_stale_alert_amber')
      .gte('created_at', since),
    supabase.from('events').select('*', { count: 'exact', head: true })
      .eq('event_name', 'canonical_retired')
      .eq('reason', 'no_recent_postings_365d')
      .gte('created_at', since),
  ]);

  return {
    canonicals_active_total: activeCount.count ?? 0,
    canonicals_pending_total: pendingCount.count ?? 0,
    canonicals_retired_total: retiredCount.count ?? 0,
    amber_emitted_30d: amberCount.count ?? 0,
    red_retired_30d: redRetiredCount.count ?? 0,
  };
}
```

Notas:

- `MAX_ALERTS_PER_EXECUTION = 10` preservado da v16 (LK-CBO-17 documenta valor herdado)
- `emitEventOnce` mantido com mesma assinatura da v16, sem mudanças funcionais
- RED dispara ação `retire_canonical`. AMBER continua como event com dedup 30d
- Métrica `staleness_detection_outcome` registrada em cada execução para análise de adequação do cap

## 3.8 — Atualização do CRON `pipeline-maintenance`

`app/api/cron/pipeline-maintenance/route.ts` atualizado:

```typescript
import {
  detectStaleCanonicals,
  gatherStalenessTelemetry,
} from '@/lib/pipeline/maintenance-canonical-staleness';

// Substituir chamada antiga (v16: detectZombies) pela nova
const [retryResult, stalenessResult, telemetry] = await Promise.all([
  retryFailedItems(supabase),
  detectStaleCanonicals(supabase),
  gatherStalenessTelemetry(supabase),
]);

await finalizeJobRun(supabase, cronRunId, finalStatus, {
  phase_errors: phaseErrors,
  retried_count: retryResult.retried,
  amber_emitted: stalenessResult.amber_emitted,
  red_retired: stalenessResult.red_retired,
  telemetry,
});
```

Mudanças vs v16:

- Import muda de `maintenance-zombie-detection` para `maintenance-canonical-staleness`
- `detectZombies` → `detectStaleCanonicals`
- `red_emitted` → `red_retired` (RED virou ação)
- `amber_emitted` mantido (AMBER continua emit)

## 3.9 — Refactor `scripts/benchmark-and-curate-bulk.ts` (D-CBO-28)

Detalhado na Etapa CBO.0a-bis da Parte 5. Resumo:

- `runBenchmarkBatch` ganha loop interno de 7 retries por size
- 3 SEM cache (`baseline_no_cache`) + 4 COM cache (1 `warmup` descartada + 3 `hot_cache_valid`)
- Sleep de 6min entre fases SEM/COM cache
- `pickFineWinner` usa média estatística sobre `retry_role='hot_cache_valid'`
- Nova `pickAbsoluteWinner` aplica tiebreaker de IC95 sobreposto
- `phaseF_Report` separa colunas baseline/hot/delta
- `DEFAULT_COARSE_SIZES = [10, 25, 35, 37, 39, 40, 50, 75, 100]`

`callRoute` modificado para enviar headers (D-CBO-28 + correção V4-H2):

```typescript
// scripts/benchmark-and-curate-bulk.ts:callRoute
async function callRoute(
  routeUrl: string,
  body: any,
  options: {
    runId: string;
    retryRole: 'baseline_no_cache' | 'warmup' | 'hot_cache_valid';
  }
): Promise<RouteResponse> {
  const cacheControlEnabled = options.retryRole !== 'baseline_no_cache';

  const res = await fetch(routeUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-internal-key': SERVICE_ROLE_KEY,
      'X-Benchmark-Run-Id': options.runId,
      'X-Benchmark-Cache-Control': cacheControlEnabled ? 'true' : 'false',
    },
    body: JSON.stringify(body),
  });

  if (!res.ok) {
    throw new Error(`callRoute failed: ${res.status} ${await res.text()}`);
  }

  return await res.json();
}
```

## 3.10 — `app/api/admin/curate-job-postings/route.ts`

Aceitar header `X-Benchmark-Cache-Control` para alternar cache do system prompt. Header `X-Benchmark-Run-Id` é gating: cache só é desativado em contexto de benchmark.

```typescript
export async function POST(req: Request) {
  // X-Benchmark-Run-Id é gating: cache só desativa em contexto de benchmark
  const isBenchmarkContext = req.headers.get('X-Benchmark-Run-Id') !== null;
  const cacheControlEnabled = isBenchmarkContext
    ? req.headers.get('X-Benchmark-Cache-Control') !== 'false'
    : true;

  await processChunk(vagas, { cacheControlEnabled, /* ... */ });
}
```

`lib/pipeline/llm-call.ts`:

```typescript
export async function callLLM(
  batch: Vaga[],
  options: { cacheControlEnabled?: boolean } = {}
) {
  const systemBlock = options.cacheControlEnabled !== false
    ? [{ type: 'text', text: SYSTEM_PROMPT, cache_control: { type: 'ephemeral' } }]
    : SYSTEM_PROMPT;

  return await anthropic.messages.create({
    model: LLM_MODEL,
    system: systemBlock,
    /* ... */
  });
}
```

## 3.11 — Sweep de renomeação zombie/zumbi

Antes do go-live, rodar:

```bash
grep -rni "zombie\|zumbi" --include="*.ts" --include="*.tsx" --include="*.sql" --include="*.md" .
```

Substituir matches conforme tabela (alinhada à v16):

| Padrão antigo (v16 real) | Substituição |
|---|---|
| `fn_cleanup_zumbi_canonical` | `fn_retire_canonical_on_zero_vacancy` |
| `trg_cleanup_zombie_canonical` (v16 spec text) | `z_trg_retire_canonical_on_zero_vacancy` |
| `z_trg_cleanup_zumbi_after_update` (v16 deployada) | (dropado em §2.9, sem substituto direto pois Frente K muda de tabela) |
| `z_trg_cleanup_zumbi_after_delete` (v16 deployada) | (dropado em §2.9, sem substituto direto pois Frente K muda de tabela) |
| `maintenance-zombie-detection` (filename + import paths) | `maintenance-canonical-staleness` |
| `detectZombies` (TS function) | `detectStaleCanonicals` |
| `gatherTelemetry` (no contexto de canonicals em maintenance) | `gatherStalenessTelemetry` |
| `RED_THRESHOLD_DAYS` | `STALE_RED_THRESHOLD_DAYS` |
| `AMBER_THRESHOLD_DAYS` | `STALE_AMBER_THRESHOLD_DAYS` |
| `canonical_red_alert` (event_name) | `canonical_stale_alert_red` |
| `canonical_amber_alert` (event_name) | `canonical_stale_alert_amber` |
| `canonical_zombie_cleanup` (event_name v16) | `canonical_retired` (com `reason='zero_vacancy_count'`) |
| `red_emitted` (campo telemetria) | `red_retired` |
| `findBestMergeCandidate` (helper antigo, removido com retire em vez de merge) | (descartado) |
| `zombie/zumbi` em comentários/docstrings PT/EN | reescrever sem o termo |

Critério: após sweep, `grep` retorna **zero matches**.

## 3.12 — `lib/pipeline/embeddings.ts`, cleanup

```bash
grep -rn "generateE5Embedding" lib/ app/ scripts/
```

Se zero callers além da definição, remover. Se houver, substituir por `generateEmbedding` direto antes de remover.

## 3.13 — `lib/pipeline/calibration-metrics.ts`

Helper dedicado para escrita em `pipeline_calibration_metrics`. **V5-H5:** implementação faltava na v5 mas era referenciada em §3.2, §3.3 e §3.7. v6 entrega.

```typescript
// lib/pipeline/calibration-metrics.ts
import { createClient } from '@supabase/supabase-js';

let serviceClient: ReturnType<typeof createClient> | null = null;

function getServiceClient() {
  if (!serviceClient) {
    serviceClient = createClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!
    );
  }
  return serviceClient;
}

export type CalibrationMode = 'coarse' | 'fine' | 'bulk' | 'mvp' | 'production';

export interface CalibrationMetricInput {
  run_id?: string;
  batch_id?: string;
  batch_size?: number;
  mode: CalibrationMode;
  metric_name: string;
  metric_value?: number;
  metric_value_text?: string;
  dimensions?: Record<string, any>;
}

/**
 * Registra métrica em pipeline_calibration_metrics.
 *
 * Append-only por design (D-CBO-29). Consumidores em §3.2 (opus arbitration),
 * §3.3 (findOrCreateFamily), §3.7 (detectStaleCanonicals), §4.4 (seed sinônimos).
 *
 * Falhas de escrita são logadas mas não interrompem o caller, porque PCM
 * é infraestrutura de observabilidade, não dado de negócio crítico.
 */
export async function persistCalibrationMetric(
  params: CalibrationMetricInput
): Promise<void> {
  try {
    const { error } = await getServiceClient()
      .from('pipeline_calibration_metrics')
      .insert({
        run_id: params.run_id ?? null,
        batch_id: params.batch_id ?? null,
        batch_size: params.batch_size ?? null,
        mode: params.mode,
        metric_name: params.metric_name,
        metric_value: params.metric_value ?? null,
        metric_value_text: params.metric_value_text ?? null,
        dimensions: params.dimensions ?? {},
      });

    if (error) {
      console.warn(
        `persistCalibrationMetric falhou para metric_name=${params.metric_name}:`,
        error.message
      );
    }
  } catch (err) {
    console.warn(
      `persistCalibrationMetric exceção para metric_name=${params.metric_name}:`,
      err
    );
  }
}
```

---

# Parte 4 — Scripts TS de seed

## 4.1 — `scripts/seed-cbo-versions.ts`

```typescript
import 'dotenv/config';
import { createClient } from '@supabase/supabase-js';
import { createHash } from 'node:crypto';
import { readFileSync } from 'node:fs';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const CBO_FILES = [
  'scripts/data/cbo/cbo2002-ocupacao.csv',
  'scripts/data/cbo/cbo2002-familia.csv',
  'scripts/data/cbo/cbo2002-sinonimo.csv',
  'scripts/data/cbo/cbo2002-perfilocupacional.csv',
];

async function main() {
  const concatenated = CBO_FILES.map(f => readFileSync(f)).reduce(
    (acc, buf) => Buffer.concat([acc, buf]),
    Buffer.alloc(0)
  );
  const filesHash = createHash('sha256').update(concatenated).digest('hex');

  await supabase.from('cbo_versions').update({ is_current: false }).eq('is_current', true);

  const { data: inserted } = await supabase.from('cbo_versions').insert({
    dataset_version: 'mte_2002_atualizado_2025-06-06',
    source_updated_at: '2025-06-06',
    imported_by: 'system_seed_cbo',
    files_hash: filesHash,
    is_current: true,
    notes: 'Carga inicial CBO 2002',
  }).select('id').single();

  console.log('cbo_versions populada:', inserted);
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.2 — `scripts/seed-cbo-occupations.ts`

```typescript
import 'dotenv/config';
import { createClient } from '@supabase/supabase-js';
import { readFileSync } from 'node:fs';
import { parse } from 'csv-parse/sync';
import iconv from 'iconv-lite';
import { generateEmbedding } from '@/lib/pipeline/embeddings';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

function readLatin1Csv(path: string): string {
  const buffer = readFileSync(path);
  return iconv.decode(buffer, 'latin1');
}

async function main() {
  const { data: currentVersion } = await supabase
    .from('cbo_versions').select('id').eq('is_current', true).single();
  if (!currentVersion) throw new Error('cbo_versions sem is_current=TRUE');

  const familiesCsv = readLatin1Csv('scripts/data/cbo/cbo2002-familia.csv');
  const families = parse(familiesCsv, { columns: true, delimiter: ';' });

  for (const family of families) {
    const slug = slugify(family.titulo);
    const familyEmbedding = await generateEmbedding(family.titulo);

    await supabase.from('taxonomy_families').upsert({
      slug,
      display_name: family.titulo,
      status: 'active',
      provisional: false,
      cbo_family_code: family.codigo,
      cbo_version_id: currentVersion.id,
      embedding: familyEmbedding,
    }, { onConflict: 'slug', ignoreDuplicates: true });
  }

  const occupationsCsv = readLatin1Csv('scripts/data/cbo/cbo2002-ocupacao.csv');
  const profilesCsv = readLatin1Csv('scripts/data/cbo/cbo2002-perfilocupacional.csv');

  const occupations = parse(occupationsCsv, { columns: true, delimiter: ';' });
  const profiles = parse(profilesCsv, { columns: true, delimiter: ';' });

  const profileByCode = new Map<string, string>();
  for (const p of profiles) {
    if (p.descricao_sumaria) profileByCode.set(p.codigo, p.descricao_sumaria);
  }

  for (const occ of occupations) {
    const familyCode = occ.codigo.substring(0, 4);
    const { data: family } = await supabase
      .from('taxonomy_families').select('id')
      .eq('cbo_family_code', familyCode).eq('cbo_version_id', currentVersion.id).single();

    if (!family) {
      console.warn(`Família ${familyCode} não encontrada para ocupação ${occ.codigo}`);
      continue;
    }

    const summaryDescription = profileByCode.get(occ.codigo) ?? null;
    const textForEmbedding = summaryDescription
      ? `${occ.titulo}. ${summaryDescription}` : occ.titulo;
    const embedding = await generateEmbedding(textForEmbedding);

    await supabase.from('canonical_role_cbo').upsert({
      occupation_code: occ.codigo,
      title: occ.titulo,
      family_id: family.id,
      summary_description: summaryDescription,
      embedding,
      cbo_version_id: currentVersion.id,
    }, { onConflict: 'occupation_code', ignoreDuplicates: true });
  }

  console.log('canonical_role_cbo populada com embeddings');
}

function slugify(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.3 — `scripts/seed-cbo-canonicals-and-links.ts`

Implementa pré-deduplicação por título (D-CBO-27) + M:N quando família diverge + confidence NULL nos seeds (V4-H1):

```typescript
import 'dotenv/config';
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

async function main() {
  const { count: nonSeedCount } = await supabase
    .from('job_canonical_roles')
    .select('*', { count: 'exact', head: true })
    .neq('source', 'cbo_mte_2002_seed');

  if ((nonSeedCount ?? 0) > 0) {
    console.error(
      'PRECONDIÇÃO VIOLADA: existem canônicos próprios no banco. ' +
      'Este script é destrutivo. Rode reset E2E antes ou aborte.'
    );
    process.exit(1);
  }

  const { data: currentVersion } = await supabase
    .from('cbo_versions').select('id').eq('is_current', true).single();

  const { data: occupations } = await supabase
    .from('canonical_role_cbo')
    .select('occupation_code, title, family_id')
    .eq('cbo_version_id', currentVersion!.id);

  const occupationsByTitle = new Map<string, typeof occupations>();
  for (const occ of occupations ?? []) {
    const normalizedTitle = normalizeTitle(occ.title);
    if (!occupationsByTitle.has(normalizedTitle)) {
      occupationsByTitle.set(normalizedTitle, []);
    }
    occupationsByTitle.get(normalizedTitle)!.push(occ);
  }

  let placeholdersCreated = 0;
  let linksCreated = 0;
  let familyLinksCreated = 0;

  for (const [normalizedTitle, group] of occupationsByTitle) {
    group.sort((a, b) => a.occupation_code.localeCompare(b.occupation_code));

    const representative = group[0];
    const slug = slugify(representative.title);

    const { data: canonical } = await supabase
      .from('job_canonical_roles')
      .upsert({
        label: representative.title,
        slug,
        status: 'pending',
        source: 'cbo_mte_2002_seed',
        confidence_median: null,
        vacancy_count: 0,
      }, { onConflict: 'slug', ignoreDuplicates: false })
      .select('id')
      .single();

    if (!canonical) continue;
    placeholdersCreated++;

    for (const [idx, occ] of group.entries()) {
      await supabase.from('canonical_role_cbo_links').upsert({
        canonical_role_id: canonical.id,
        occupation_code: occ.occupation_code,
        is_primary: (idx === 0),
        confidence: null,
        source: 'cbo_mte_2002_seed',
      }, { onConflict: 'canonical_role_id,occupation_code', ignoreDuplicates: true });
      linksCreated++;
    }

    const familyIds = new Set(group.map(o => o.family_id));

    for (const familyId of familyIds) {
      await supabase.from('taxonomy_family_canonicals').upsert({
        family_id: familyId,
        canonical_role_id: canonical.id,
      }, { onConflict: 'family_id,canonical_role_id', ignoreDuplicates: true });
      familyLinksCreated++;
    }
  }

  console.log(`Placeholders únicos criados: ${placeholdersCreated}`);
  console.log(`Links CBO criados: ${linksCreated}`);
  console.log(`Family links criados (M:N): ${familyLinksCreated}`);
  console.log(`Razão dedup: ${(linksCreated / placeholdersCreated).toFixed(2)} ocupações por placeholder`);
}

function normalizeTitle(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().trim().replace(/\s+/g, ' ');
}

function slugify(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.4 — `scripts/seed-cbo-synonyms.ts`

```typescript
import 'dotenv/config';
import { createClient } from '@supabase/supabase-js';
import { readFileSync } from 'node:fs';
import { parse } from 'csv-parse/sync';
import iconv from 'iconv-lite';
import { persistCalibrationMetric } from '@/lib/pipeline/calibration-metrics';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

function readLatin1Csv(path: string): string {
  const buffer = readFileSync(path);
  return iconv.decode(buffer, 'latin1');
}

async function main() {
  const { data: currentVersion } = await supabase
    .from('cbo_versions').select('id').eq('is_current', true).single();

  const synonymsCsv = readLatin1Csv('scripts/data/cbo/cbo2002-sinonimo.csv');
  const synonyms = parse(synonymsCsv, { columns: true, delimiter: ';' });

  const seenTerms = new Set<string>();
  let droppedDueToCollision = 0;
  const droppedRecords: Array<{ term: string; codigo: string; titulo: string }> = [];

  for (const syn of synonyms) {
    const sourceTerm = normalizeTerm(syn.titulo_sinonimo);
    if (!sourceTerm) continue;
    if (seenTerms.has(sourceTerm)) {
      droppedDueToCollision++;
      // V5-P5: registrar cada descarte para auditoria pós-seed
      droppedRecords.push({
        term: sourceTerm,
        codigo: syn.codigo,
        titulo: syn.titulo_sinonimo,
      });
      continue;
    }
    seenTerms.add(sourceTerm);

    const { data: link } = await supabase
      .from('canonical_role_cbo_links')
      .select('canonical_role_id')
      .eq('occupation_code', syn.codigo)
      .eq('is_primary', true)
      .maybeSingle();

    if (!link) continue;

    await supabase.from('taxonomy_relations').upsert({
      source_term: sourceTerm,
      target_canonical_id: link.canonical_role_id,
      status: 'active',
      layer: null,
      llm_proposed_label: null,
      linguistic_category: 'synonym',
      validated_at: new Date().toISOString(),
      validated_by: 'mte_cbo_2002_seed',
      cbo_version_id: currentVersion!.id,
    }, { onConflict: 'source_term', ignoreDuplicates: true });
  }

  // V5-P5: registrar descartes em PCM para auditoria
  if (droppedRecords.length > 0) {
    await persistCalibrationMetric({
      mode: 'mvp',
      metric_name: 'cbo_synonym_collision_dropped',
      metric_value: droppedRecords.length,
      dimensions: {
        sample: droppedRecords.slice(0, 50),
        total_dropped: droppedRecords.length,
        total_synonyms_in_csv: synonyms.length,
      },
    });
  }

  console.log(`taxonomy_relations populada com ${seenTerms.size} sinônimos MTE`);
  console.log(`Sinônimos descartados por colisão: ${droppedDueToCollision}`);
}

function normalizeTerm(text: string): string {
  return text?.normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().trim() ?? '';
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.5 — `scripts/precheck-slug-collision.ts` (V4-B1)

Pré-check de colisão de slug, executando antes da migration de schema.

```typescript
import 'dotenv/config';
import { createClient } from '@supabase/supabase-js';
import { readFileSync } from 'node:fs';
import { parse } from 'csv-parse/sync';
import iconv from 'iconv-lite';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

function slugify(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

async function main() {
  const csv = iconv.decode(
    readFileSync('scripts/data/cbo/cbo2002-ocupacao.csv'),
    'latin1'
  );
  const rows = parse(csv, { columns: true, delimiter: ';' });
  const titles = rows.map((r: any) => r.titulo);
  const cboPredictedSlugs = [...new Set(titles.map(slugify))];

  console.log(`Verificando ${cboPredictedSlugs.length} slugs CBO previstos contra canônicos próprios...`);

  const { data: collisions } = await supabase
    .from('job_canonical_roles')
    .select('slug, label, source')
    .in('slug', cboPredictedSlugs)
    .neq('source', 'cbo_mte_2002_seed');

  if (collisions && collisions.length > 0) {
    console.error('PRECONDIÇÃO VIOLADA: colisão de slug detectada');
    console.error(JSON.stringify(collisions, null, 2));
    process.exit(1);
  }

  console.log(`Pré-check OK. Zero colisões.`);
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.6 — `scripts/seed-cbo-master.ts`

```typescript
import 'dotenv/config';
import { execSync } from 'node:child_process';

async function main() {
  console.log('Seed CBO master, start');

  console.log('Etapa 1: cbo_versions');
  execSync('npx tsx --env-file=.env.local scripts/seed-cbo-versions.ts', { stdio: 'inherit' });

  console.log('Etapa 2: canonical_role_cbo, taxonomy_families CBO + embeddings');
  execSync('npx tsx --env-file=.env.local scripts/seed-cbo-occupations.ts', { stdio: 'inherit' });

  console.log('Etapa 3: canônicos placeholder (deduplicados), links CBO N:M, family_canonicals M:N');
  execSync('npx tsx --env-file=.env.local scripts/seed-cbo-canonicals-and-links.ts', { stdio: 'inherit' });

  console.log('Etapa 4: taxonomy_relations sinônimos MTE');
  execSync('npx tsx --env-file=.env.local scripts/seed-cbo-synonyms.ts', { stdio: 'inherit' });

  console.log('Seed CBO concluído');
}

main().catch(err => { console.error('FATAL:', err); process.exit(1); });
```

## 4.7 — Dependências adicionais

`package.json`:

```json
{
  "dependencies": {
    "csv-parse": "^5.5.6",
    "iconv-lite": "^0.6.3"
  }
}
```

---

# Parte 5 — Plano de execução

A sprint CBO se intercala com o plano da v14, com refactor do benchmark (Etapa CBO.0a-bis) ordenado para zero impacto nas demais atividades.

### Etapa 0, commit pré-sprint CBO

```bash
git status
git checkout -b sprint-cbo-v5
git add -A
git commit -m "chore: estado consolidado pré-sprint CBO v5"
git push origin sprint-cbo-v5
```

### Etapa CBO.0, anexar dependências e CSVs

1. Atualizar `package.json` com `csv-parse` e `iconv-lite`
2. `npm install`
3. Anexar 4 CSVs MTE em `scripts/data/cbo/`:
   - `cbo2002-ocupacao.csv`
   - `cbo2002-familia.csv`
   - `cbo2002-sinonimo.csv`
   - `cbo2002-perfilocupacional.csv`
4. Validar encoding latin-1 e separador `;`

### Etapa CBO.0a, pré-check colisão de slug

Script TypeScript dedicado (V4-B1):

```bash
npx tsx --env-file=.env.local scripts/precheck-slug-collision.ts
```

Critério: exit 0 (zero colisões). Se exit 1, abortar e investigar.

### Etapa CBO.0a-bis, refactor benchmark v14

Sub-PR 1: infraestrutura (~3h)
- Migration `14_benchmark_runs_alter.sql` adicionando colunas de retry semantics + extensão de CHECK mode para `'bulk'`
- Endpoint `/api/admin/curate-job-postings` aceita header `X-Benchmark-Cache-Control` com gating por `X-Benchmark-Run-Id`
- `lib/pipeline/llm-call.ts` e `lib/pipeline/batch-processor.ts` propagam flag
- `runBenchmarkBatch` ganha loop interno de N retries
- `callRoute` envia headers `X-Benchmark-Run-Id` e `X-Benchmark-Cache-Control` em todas as chamadas

Sub-PR 2: lógica de decisão (~3h)
- `pickFineWinner` refatorada para usar média estatística sobre `retry_role='hot_cache_valid'`
- Nova `pickAbsoluteWinner` com tiebreaker de IC95 sobreposto
- `phaseF_Report` separa colunas baseline/hot/delta

Sub-PR 3: guards e testes (~1h30)
- Cost guard >$5 com `CONFIRM_HIGH_COST=true` para bypass
- **V5-P4:** verificar existência da tabela `bulk_curation_progress` antes do checkpoint. Se ausente, criar via migration prévia:
  ```sql
  CREATE TABLE IF NOT EXISTS bulk_curation_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL,
    last_processed_batch_id UUID,
    completed_batches INT NOT NULL DEFAULT 0,
    total_batches INT NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  ALTER TABLE bulk_curation_progress ENABLE ROW LEVEL SECURITY;
  ```
- Checkpoint de retomada em `bulk_curation_progress`
- Testes de integração com `--dry-run --replicates=2 --sample-sizes=10,38,100`

`DEFAULT_COARSE_SIZES = [10, 25, 35, 37, 39, 40, 50, 75, 100]` (D-CBO-28).

### Etapa CBO.1, migrations de schema CBO

Executar via Claude Code, em ordem:

1. `01_cbo_versions.sql`
2. `02_taxonomy_families_alter.sql`
3. `03_canonical_role_cbo.sql`
4. `04_taxonomy_relations_alter.sql`
5. `05_canonical_role_cbo_links.sql`
6. `06_job_canonical_roles_alter.sql`
7. `07_alter_promote_canonical_for_retired.sql`
8. `08_retire_canonical_rpc.sql`
9. `09_rename_zumbi_to_retire.sql`
10. `10_reset_taxonomy_core_alter.sql`
11. `11_process_opus_create_new.sql`
12. `12_replace_cbo_link_rpc.sql`
13. `13_pipeline_calibration_metrics.sql`

### Etapa CBO.2, cleanup de dívida técnica + sweep de renomeação

```bash
grep -rn "generateE5Embedding" lib/ app/ scripts/
grep -rni "zombie\|zumbi" --include="*.ts" --include="*.tsx" --include="*.sql" --include="*.md" .
```

Remover `generateE5Embedding` se zero callers. Substituir todas referências zombie/zumbi conforme tabela §3.11. Critério: ambos os `grep` retornam zero matches.

### Etapa CBO.3, atualização de prompts e processamento Opus

1. Editar `lib/pipeline/opus-arbitration-prompts.ts` (§3.1)
2. Editar `lib/pipeline/opus-arbitration.ts` (§3.2)
3. Refatorar `lib/pipeline/find-or-create-family.ts` (§3.3)
4. Estender `lib/pipeline/taxonomy-cache.ts` (§3.4) — atenção ao filtro in-memory vs SQL
5. Estender `lib/pipeline/suggested-roles-builder.ts` (§3.5)
6. Estender `lib/pipeline/precheck-description-hash.ts` (§3.6)
7. Renomear `maintenance-zombie-detection.ts` → `maintenance-canonical-staleness.ts` (§3.7)
8. Atualizar CRON `pipeline-maintenance` (§3.8)
9. Rodar `npx tsc --noEmit`

### Etapa CBO.4, seed das tabelas CBO

```bash
npx tsx --env-file=.env.local scripts/seed-cbo-master.ts
```

Critério após execução:

```sql
SELECT
  (SELECT COUNT(*) FROM cbo_versions WHERE is_current=TRUE) AS versions_current,
  (SELECT COUNT(*) FROM canonical_role_cbo) AS occupations,
  (SELECT COUNT(*) FROM taxonomy_families WHERE cbo_family_code IS NOT NULL) AS cbo_families,
  (SELECT COUNT(*) FROM taxonomy_families WHERE cbo_family_code IS NOT NULL AND embedding IS NOT NULL) AS cbo_families_with_embedding,
  (SELECT COUNT(*) FROM job_canonical_roles WHERE source='cbo_mte_2002_seed') AS placeholders,
  (SELECT COUNT(*) FROM canonical_role_cbo_links) AS cbo_links,
  (SELECT COUNT(*) FROM canonical_role_cbo_links WHERE confidence IS NULL AND source='cbo_mte_2002_seed') AS cbo_links_seed_with_null_confidence,
  (SELECT COUNT(*) FROM taxonomy_family_canonicals tfc
    JOIN job_canonical_roles jcr ON jcr.id = tfc.canonical_role_id
    WHERE jcr.source='cbo_mte_2002_seed') AS family_links_seed,
  (SELECT COUNT(*) FROM taxonomy_relations
    WHERE cbo_version_id IS NOT NULL AND linguistic_category='synonym') AS mte_synonyms,
  (SELECT COUNT(*) FROM canonical_role_cbo WHERE embedding IS NULL) AS occupations_without_embedding;
```

**V5-P2:** validação adicional de invariante para `findOrCreateFamily`. Toda família ativa deve ter embedding (LK-CBO-13). Query deve retornar zero:

```sql
SELECT COUNT(*) AS active_families_without_embedding
FROM taxonomy_families
WHERE status = 'active'
  AND embedding IS NULL;
```

Se retornar > 0, abortar antes da Etapa 1 do plano v14. `findOrCreateFamily` falha em produção quando candidate sem embedding entra em fluxo de matching.

### Etapa CBO.5, atualização de `/methodology`

Aplicar 5 mudanças do Anexo A.

### Etapa 1 (v14), reset E2E modificado

```bash
npx tsx --env-file=.env.local scripts/reset-pipeline-e2e.ts
```

Critério: tabelas CBO preservadas; tabelas curadas zeradas; `pipeline_calibration_metrics` zerada.

### Etapas 2 a 3 (v14), validação de schema e env vars Vercel

Conforme plano original.

### Etapa 4 (v14), benchmark robusto

```bash
node scripts/benchmark-and-curate-bulk.ts \
  --replicates=7 \
  --cold-count=3 \
  --hot-count=4
```

Custo estimado: ~$120-370. Tempo: ~6-12h (incluindo sleeps de 6min entre fases).

### Etapas 5+ (v14), validações pós-benchmark

Conforme plano original + análise via `pipeline_calibration_metrics` (Parte 9).

---

# Parte 6 — Critérios de aceite

- `cbo_versions` populada com 1 row `is_current=TRUE`
- `canonical_role_cbo` com ~2.600 rows, todas com `embedding IS NOT NULL`
- `taxonomy_families` com ~600 rows com `cbo_family_code IS NOT NULL` e `embedding IS NOT NULL`
- `job_canonical_roles` com ~2.500 placeholders (deduplicados por título)
- `canonical_role_cbo_links` com ~2.600 rows, todas com `confidence IS NULL` e `source='cbo_mte_2002_seed'`
- `taxonomy_family_canonicals` com ~2.500 rows (M:N quando família diverge)
- `taxonomy_relations` com ~5.000 rows com `cbo_version_id IS NOT NULL` e `linguistic_category='synonym'`
- Coluna `cbo_codes` removida de `job_canonical_roles`
- Coluna `updated_at` adicionada com trigger semântico (12 colunas monitoradas)
- Status `'retired'` aceito no CHECK (7 valores totais, preservando `alias_of` e `merge_candidate`)
- Constraint `JCR.source` aceita `'cbo_mte_2002_seed'`
- Constraint `TR.linguistic_category` aceita `'synonym'`
- CHECK `cbo_version_id IS NOT NULL` implica `linguistic_category='synonym'` ativo
- `fn_retire_canonical_on_zero_vacancy` (renomeada) implementada sem FK guards, com early return CBO + delegação a `retire_canonical`. Filtro ampliado: `status IN ('active', 'pending', 'rejected')`, excluindo `deprecated`/`alias_of`/`merge_candidate`
- Trigger `z_trg_retire_canonical_on_zero_vacancy` ativo em `job_canonical_roles` (AFTER UPDATE OF `vacancy_count`)
- Função `retire_canonical(id, reason)` única consumida por 2 callers
- Reset E2E preserva rows CBO; zera `pipeline_calibration_metrics`
- Tool_use Opus aceita `suggested_cbo` com flag `replace_existing` e verdict `CREATE_NEW`
- RPCs `process_opus_create_new`, `replace_cbo_link`, `fetch_cbo_candidates`, `fetch_family_candidates` em produção
- `findOrCreateFamily` refatorada para cosine puro 0.85 + réplica Opus, registrando decisões em PCM
- `detectStaleCanonicals` chama `retire_canonical` quando gap >= 365d, sem sugerir merge. `MAX_ALERTS_PER_EXECUTION = 10` preservado da v16. Métrica `staleness_detection_outcome` registrada em PCM por execução
- `emitEventOnce` da v16 preservado para AMBER (dedup 30d)
- Camadas 0, 1, 2 incluem `'retired'` no filtro
- Camada 2 com piso de 2 tokens + tie-break CBO + tiebreak alfabético
- Trigger `fn_promote_canonical_on_threshold` aceita `('pending', 'retired')` para ressuscitação, preservando AFTER UPDATE da v16, critério `confidence_median >= 0.75`, e chamada a `auto_assign_family_to_canonical`
- `benchmark_runs` ganha colunas `retry_index`, `retry_role`, `cache_control_enabled`, `discard`, `replicate_group_id`
- CHECK `mode` em `benchmark_runs` aceita `'bulk'`
- Endpoint `/api/admin/curate-job-postings` respeita `X-Benchmark-Cache-Control` com gating por `X-Benchmark-Run-Id`
- `callRoute` envia headers `X-Benchmark-Run-Id` e `X-Benchmark-Cache-Control` em todas as chamadas durante benchmark; valor reflete `retryRole`
- `pipeline_calibration_metrics` populada durante benchmark com 10+ tipos de métricas
- `generateE5Embedding` removida
- Sweep zombie/zumbi: `grep -rni "zombie\|zumbi"` retorna zero matches em todo codebase
- Página `/methodology` atualizada com 5 mudanças
- Sonnet `SYSTEM_PROMPT.ts` v2.7 sem alterações

---

# Parte 7 — Limitações conhecidas

| LK | Descrição |
|---|---|
| LK-CBO-01 | CBO 2002 não cobre profissões digitais modernas |
| LK-CBO-02 | Atualização MTE futura exige nova execução do seed com bump em `cbo_versions` |
| LK-CBO-03 | Cosine similarity com 768d Gemini é o único método de matching CBO |
| LK-CBO-04 | Canônicos placeholder com `vacancy_count=0` ficam no banco indefinidamente |
| LK-CBO-05 | Cenário `CREATE_NEW` exige confidence Opus >= 0.85; abaixo cai em `disagreed_pending_review` |
| LK-CBO-06 | Famílias CBO no slug podem colidir com famílias futuras criadas pelo Opus |
| LK-CBO-07 | Sinônimos MTE colidentes entre múltiplas ocupações são deduplicados via "primeiro vencedor" |
| LK-CBO-08 | Concorrência Sonnet/Opus durante benchmark tratada via guard de pré-condição |
| LK-CBO-09 | Sonnet ressuscita `'retired'` sem arbitragem extra (drift semântico aceito) |
| LK-CBO-10 | Threshold cosine 0.85 (família) e 0.70 (CBO) inicialmente empíricos; revalidados pós-benchmark via `pipeline_calibration_metrics` |
| LK-CBO-11 | Réplica Opus em `findOrCreateFamily` adiciona ~30-50min de latência ao tempo total de arbitragem (CRON, sem impacto operacional) |
| LK-CBO-12 | Pré-deduplicação no seed por título normalizado pode falhar para variações ortográficas (acentos, plurais). Estimativa: <1% dos ~2.600 títulos |
| LK-CBO-13 | `findOrCreateFamily` exige `embedding IS NOT NULL` em todas as famílias ativas. Estado pré-benchmark garantido pelo reset E2E que deleta famílias custom (`cbo_family_code IS NULL`). Eventual re-seed sem reset prévio exige migration de backfill |
| LK-CBO-14 | Embedding em ressuscitação preserva valor original. Não há refresh proativo; coerente com fato de embedding ser função apenas do label, e label não muda durante aposentadoria |
| LK-CBO-15 | Canônicos aposentados via `detectStaleCanonicals` (gap >= 365d) são detectados apenas quando `vacancy_count > 0`. Canônicos com vacancy_count já zerado pelo trigger Frente K não disparam o caminho de gap_days. Os dois caminhos são complementares, não sobrepostos |
| LK-CBO-16 | Painel admin para canônicos aposentados não construído nesta sprint. Eventos `canonical_retired` carregam contexto rico em `metadata` para painel futuro |
| LK-CBO-17 | `MAX_ALERTS_PER_EXECUTION = 10` herdado da v16 sem justificativa empírica documentada. Métrica `staleness_detection_outcome` em `pipeline_calibration_metrics` permite análise pós-produção. Critério de revisão: se `dimensions.cap_hit_red = true` em >5% das execuções após 30-60 dias de operação, considerar elevar o cap |
| LK-CBO-18 | Frente K em v5 amplia filtro vs v16: aposenta canônicos `('active', 'pending', 'rejected')` em vez de apenas `'pending'`. Comportamento de v16 mais restritivo era inconsistente com princípio de soft delete; v5 corrige. Canônicos `deprecated`/`alias_of`/`merge_candidate` continuam fora do filtro porque cada um reflete operação distinta (merge, redirect, fila Opus) |
| LK-CBO-19 | §2.7 preserva pattern AFTER UPDATE + `pg_trigger_depth()` + UPDATE explícito da v16 deployada (`schema.sql:3473-3567`). Decisão arquitetural opção (a) priorizando estabilidade vs opção (b) que reverteria para BEFORE+NEW assignments. Justificativa: preservar pattern em produção minimiza risco de regressão, evita janela onde Frente A (BEFORE promote) e Frente K (AFTER retire) interagem em batches simultâneos sem guard adicional. Lição metodológica: ground truth para body de trigger é `schema.sql` gerado de live Supabase, não spec text histórica. v5 importou body bugado da spec text (`NEW.status := 'active'` em AFTER trigger, silenciosamente descartado pelo executor); v6 corrige importando body real do schema deployado |

---

# Parte 8 — Anexo A: textos da página `/methodology`

### Mudança 1, nova seção após "O princípio"

A estrutura por trás da pontuação

## Cinco camadas que se auditam mutuamente

A análise que você recebe não vem de uma única fonte. Cinco camadas independentes alimentam a metodologia. Cada uma tem base teórica ou empírica reconhecida, e cada uma serve de contraponto às demais. Esse desenho garante que o resultado final reflita o mercado real, sem viés único.

01

### Camada empírica, vagas reais do mercado

Vagas publicadas por empresas brasileiras nos últimos 120 dias. É a fonte primária da análise. Captura o mercado como ele se comporta hoje, com toda a sua heterogeneidade e ritmo de mudança.

02

### Camada institucional, Classificação Brasileira de Ocupações

A CBO 2002 do Ministério do Trabalho e Emprego, derivada do padrão internacional ISCO-88 da Organização Internacional do Trabalho com correlações para ISCO-08, fornece a estrutura de referência para reconhecer ocupações tradicionais, suas famílias profissionais e suas variações terminológicas oficiais. Nem toda função moderna existe na CBO, e nesses casos a camada empírica prevalece. Mas onde a CBO cobre, ela serve de âncora institucional para a nossa taxonomia, conectando o mercado real à terminologia regulamentada e validada por comitês setoriais com participação tripartite de governo, empregadores e trabalhadores.[^cbo][^isco]

03

### Camada estatística, pós-estratificação por nível de experiência

Quando a base de vagas tem distribuição desigual entre níveis de senioridade, aplicamos pós-estratificação para que sua análise reflita melhor o seu nível profissional. É a mesma técnica usada por IBGE e Pew Research em pesquisas populacionais.

04

### Camada de matching, análise semântica

Para reconhecer equivalências entre o que está no seu currículo e o que as vagas pedem, usamos modelos de linguagem que entendem significado contextual, não apenas palavras-chave. "Coordenação de equipes distribuídas" é reconhecido como evidência de "liderança de times remotos".

05

### Camada de governança, curadoria automatizada com auditoria independente

Toda função identificada no mercado passa por dois algoritmos com responsabilidades distintas. Um curador, que extrai estrutura e propõe classificação a partir das descrições de vaga. Um árbitro, que audita as decisões do curador, valida famílias e domínios, identifica duplicidades e refina a taxonomia ao longo do tempo. Essa separação evita que decisões automatizadas se acumulem sem revisão. O resultado é um catálogo vivo, que evolui com o mercado mas mantém coerência interna auditável.

Cada camada tem limites próprios e nenhuma sozinha define o resultado. Quando o mercado muda mais rápido que a regulação, a camada empírica prevalece. Quando uma função tem volume insuficiente para análise estatística, a camada institucional informa. Quando uma vaga usa nomenclatura ambígua, a camada de governança decide. **A combinação é o que torna a análise consistente.**

### Mudança 2, parágrafo em "Como o catálogo de funções se mantém vivo"

A combinação das três fontes não é suficiente sozinha. Precisa de mecanismos de monitoramento que detectem quando uma função perde relevância no mercado, quando duas funções convergem semanticamente, ou quando uma nova nomenclatura emerge com volume consistente. Esses mecanismos operam continuamente:

### Mudança 3, item adicional em "Limitações conhecidas"

#### Funções emergentes sem código oficial brasileiro

Funções que surgiram no mercado digital nos últimos anos, como Tech Lead, Product Owner, SRE, Customer Success Manager, Data Engineer e DevOps, ainda não possuem código próprio na Classificação Brasileira de Ocupações. Isso não afeta a qualidade da análise. A camada empírica de vagas reais do mercado prevalece nesses casos, e a pontuação de aderência continua tecnicamente válida. A ausência de código institucional apenas indica que a regulação trabalhista brasileira ainda não formalizou essas ocupações.

### Mudança 4, reescrita do item de viés de agências

Se mais de 20% das vagas analisadas vêm de agências de recrutamento, sinalizamos "alta presença de intermediários". Agências e RPOs, modelo conhecido como Recruitment Process Outsourcing, atuam concentrando volume de múltiplos clientes. Quando essa concentração ultrapassa um quinto das vagas analisadas, há indício de que a descrição das funções pode refletir mais o padrão de redação dos intermediários do que o padrão de empregadores diretos. Estudos internacionais reportam que agências representam tipicamente cerca de 2% do volume natural de vagas em mercados profissionalizados.[^asa] O limite inicial de 20% adotado pelo CalibraCV é confortavelmente acima desse padrão, e será calibrado conforme acumulemos dados reais de uso no mercado brasileiro.

### Mudança 5, reescrita da frase em "Como o catálogo de funções se mantém vivo"

A janela de 120 dias é a mesma utilizada em toda a metodologia, conforme detalhado na seção sobre janela temporal. Os 365 dias correspondem a quatro trimestres de ciclo orçamentário completo.

### Footnotes

```
[^cbo]: Ministério do Trabalho e Emprego (2002). Classificação Brasileira de Ocupações. https://www.gov.br/trabalho-e-emprego/pt-br/assuntos/cbo

[^isco]: International Labour Organization (2008). International Standard Classification of Occupations (ISCO-08). https://www.ilo.org/public/english/bureau/stat/isco/isco08/

[^asa]: American Staffing Association. Staffing Industry Statistics. https://americanstaffing.net/research/fact-sheets-analysis-staffing-industry-trends/staffing-industry-statistics/
```

---

# Parte 9 — Metodologia de calibragem pós-benchmark

## 9.1 — Princípio

Todos os thresholds, limites e parâmetros calibráveis introduzidos ou tocados nesta sprint têm valores iniciais empíricos. O benchmark de ~10.150 vagas é o evento de calibragem total: único momento controlado com volume e diversidade suficientes para refinar valores antes do MVP.

`pipeline_calibration_metrics` é infraestrutura **viva**: nasce no benchmark, segue ativa no MVP e produção, acomodando métricas operacionais que só fazem sentido medir com usuários reais.

## 9.2 — Variáveis calibradas pelo benchmark

| metric_name | Valor inicial | Como medir | Decisão |
|---|---|---|---|
| Cosine threshold em `findOrCreateFamily` | 0.85 | Distribuição de cosine quando réplica Opus foi feita | Bucket onde reuse/total = ~0.5 |
| Cosine threshold em `fetchCboCandidates` | 0.70 | Distribuição de cosine entre canônico e top-K candidatos | Top-K retorna ruído ou descarta válidos |
| Tie-break CBO em ranking de famílias | Hard preference | Distribuição de matches CBO vs custom em PCM, com flag `candidate_origin_cbo` | Se CBO sempre ganha desnecessariamente, considerar bonus aditivo (revisar pós-benchmark com dado real) |
| Confidence Opus para CREATE_NEW | 0.85 | Canônicos `CREATE_NEW` que sobreviveram vs viraram deprecated | Bucket onde still_active >> later_merged |
| Confidence Opus para `suggested_cbo` | 0.70 | Distribuição de confidence em `suggested_cbo` | Filtrar abaixo de threshold ótimo |
| Piso tokens em `findFamilyMatch` | 2 | Análise de matches por overlap | 1, 2, 3 mudam qualidade? |
| Hard Gate Frente A | 3 vagas + 2 fontes distintas + `confidence_median >= 0.75` + 60d recência | Distribuição de promoções e canônicos que viraram deprecated | Par (vagas, fontes) onde proven_redundant/canonicals < 5% |
| MAX_SUGGESTED_ROLES | 12 | Distribuição de qual posição no shortlist o Sonnet escolheu | Se quase sempre é top-3, 12 é exagero |
| batch_size benchmark | descoberto via Fases A+C | media_regime_hot_cache.custo_por_vaga | Menor com IC95 não-sobreposto a vizinhos |
| `MAX_ALERTS_PER_EXECUTION` em `detectStaleCanonicals` | 10 (herdado da v16) | Métrica `staleness_detection_outcome` em PCM com flag `cap_hit_red` | Elevar se cap atingido em >5% das execuções após 30-60 dias |

## 9.3 — Variáveis para futuro (registradas pós-MVP)

| metric_name | Valor atual | Quando medir |
|---|---|---|
| Cooldown 72h | 72h | Após usuários reais fazerem upload |
| Limite 20% RPO | 20% | Após volume diverso de análises |

## 9.4 — Pontos de coleta durante benchmark

### Categoria 1: inline em `runBenchmarkBatch`

Após `persistBenchmarkRun`, função `persistCalibrationMetrics(row, resp)` registra `vagas_curated_pct`, `vagas_low_quality_pct`, `canonical_created_per_vaga`, `latency_route_seconds`.

### Categoria 2: pós-batch via query agregada

Função `collectQualityMetricsForSession(sessionId, runId, batchSize, mode)` correlaciona via `function_orchestrator_items.run_id` e registra distribuição de `pipeline_stage`, `confidence_buckets`, contagem de verdicts Opus.

### Categoria 3: nas decisões da sprint CBO

Distribuídas pelo código:
- `family_cosine_decision` em `findOrCreateFamily`
- `cbo_candidate_cosine` em `fetchCboCandidates`
- `opus_create_new_confidence` em RPC
- `opus_suggested_cbo_confidence` em arbitragem
- `family_match_token_overlap` em `findFamilyMatch`
- `opus_replica_decision` em `findOrCreateFamily`
- `find_or_create_family_outcome` em wrapper
- `hard_gate_promotion` em Frente A
- `shortlist_position_chosen` após Sonnet escolher
- `staleness_detection_outcome` em `detectStaleCanonicals` por execução do CRON

## 9.5 — Análise pós-benchmark

Queries SQL de análise para cada métrica, executadas após benchmark concluir:

**Threshold cosine de família:**

```sql
SELECT
  WIDTH_BUCKET(metric_value, 0.5, 1.0, 20) AS bucket,
  COUNT(*) AS occurrences,
  COUNT(*) FILTER (WHERE metric_value_text = 'REUSE_CANDIDATE') AS reused,
  COUNT(*) FILTER (WHERE metric_value_text = 'CREATE_NEW') AS created
FROM pipeline_calibration_metrics
WHERE metric_name = 'family_cosine_decision'
GROUP BY bucket
ORDER BY bucket;
```

**Confidence Opus para CREATE_NEW:**

```sql
SELECT
  WIDTH_BUCKET(jcr.confidence_median, 0.5, 1.0, 10) AS bucket,
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE jcr.status = 'deprecated') AS later_merged,
  COUNT(*) FILTER (WHERE jcr.status = 'active') AS still_active
FROM job_canonical_roles jcr
WHERE jcr.source = 'llm_extractor'
  AND EXISTS (
    SELECT 1 FROM events
    WHERE event_name = 'opus_created_new_canonical'
      AND resource_id = jcr.id
  )
GROUP BY bucket
ORDER BY bucket;
```

**Hard Gate:**

```sql
SELECT
  vacancy_count AS vagas_at_promotion,
  distinct_sources_count AS sources_at_promotion,
  COUNT(*) AS canonicals,
  COUNT(*) FILTER (WHERE status = 'deprecated') AS proven_redundant
FROM job_canonical_roles
WHERE promoted_at IS NOT NULL
GROUP BY vacancy_count, distinct_sources_count
ORDER BY canonicals DESC;
```

**Custo Sonnet vs Opus inline (separação via `ai_usage_logs.call_type`):**

```sql
SELECT
  br.batch_size,
  br.retry_role,
  SUM(aiu.cost_usd) FILTER (WHERE aiu.call_type = 'job_curation_manual') AS sonnet_cost,
  SUM(aiu.cost_usd) FILTER (WHERE aiu.call_type LIKE '%opus%') AS opus_cost
FROM benchmark_runs br
JOIN function_orchestrator_runs forun ON forun.session_id = (br.raw_metadata->>'session_id')
JOIN ai_usage_logs aiu ON aiu.run_id = forun.id
WHERE br.run_id = $benchmark_run_id
  AND br.retry_role = 'hot_cache_valid'
GROUP BY br.batch_size, br.retry_role
ORDER BY br.batch_size;
```

**batch_size vencedor com IC95:**

```sql
WITH metrics AS (
  SELECT
    batch_size,
    AVG(custo_por_vaga) AS media,
    STDDEV(custo_por_vaga) AS dp,
    COUNT(*) AS n
  FROM benchmark_runs
  WHERE run_id = $benchmark_run_id
    AND mode = 'fine'
    AND retry_role = 'hot_cache_valid'
  GROUP BY batch_size
)
SELECT
  batch_size,
  media,
  dp,
  n,
  media - 1.96 * dp / SQRT(n) AS ic95_low,
  media + 1.96 * dp / SQRT(n) AS ic95_high
FROM metrics
ORDER BY media ASC;
```

**Aposentadorias por motivo (D-CBO-30):**

```sql
SELECT
  reason,
  metadata->>'previous_status' AS previous_status,
  COUNT(*) AS quantidade
FROM events
WHERE event_name = 'canonical_retired'
  AND created_at >= $benchmark_start
GROUP BY reason, metadata->>'previous_status'
ORDER BY quantidade DESC;
```

**Análise de cap em `detectStaleCanonicals` (LK-CBO-17):**

```sql
SELECT
  COUNT(*) AS total_executions,
  COUNT(*) FILTER (WHERE (dimensions->>'cap_hit_red')::boolean = true) AS executions_cap_hit_red,
  COUNT(*) FILTER (WHERE (dimensions->>'cap_hit_amber')::boolean = true) AS executions_cap_hit_amber,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE (dimensions->>'cap_hit_red')::boolean = true) / NULLIF(COUNT(*), 0),
    2
  ) AS pct_cap_hit_red
FROM pipeline_calibration_metrics
WHERE metric_name = 'staleness_detection_outcome'
  AND mode = 'production'
  AND computed_at >= NOW() - INTERVAL '30 days';
```

## 9.6 — Protocolo de decisão de ajuste

Para cada threshold:

1. Reportar valor atual, valor sugerido pela análise, e justificativa empírica
2. Se diferença > 10% E dataset > 50 amostras: **ajustar antes do go-live MVP**
3. Se diferença < 10% OU dataset < 50: **manter valor inicial, marcar para reanálise pós-MVP**
4. Documentar decisão em commit antes do go-live

Saída esperada: tabela markdown comparativa, commitada em `docs/calibration/post-benchmark-2026-XX.md`.

---

**Fim da SPEC-sprint-cbo-v5.**
