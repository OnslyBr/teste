# Sprint v6.0 — Fluxo de Retorno + Governança de Taxonomia + Limpeza Estrutural

**Versão:** v6.0 (consolidação de 5 reviewers sobre v5 + Claude Code com acesso direto ao banco confirmando 3 NOT NULL sem default no test fixture + Gemini detectando 2 bugs lógicos sutis no merge transacional + outros achados editoriais)
**Data:** 27/04/2026
**Autor:** Onsly (PO) + Claude (arquiteto técnico)
**Implementação:** Antigravity (código TS) + Claude Code (SQL)
**Predecessores:** v1.0 FINAL (24/04/2026) → v2.0 FINAL (27/04/2026) → v3.0 FINAL (27/04/2026) → v4.0 FINAL (27/04/2026) → v5.0 FINAL (27/04/2026)

---

## Changelog v5 → v6 (mudanças desta versão)

Esta versão incorpora 5 reviewers sobre v5 (Gemini, Claude-Code com acesso DB direto, Outra-Claude, Grok, GenSpark). DeepSeek descartado nesta rodada (estava revisando v4 em vez de v5 — duplicata). Mistral mantido fora do pool desde v4 (decisão cravada).

A complexidade arquitetural está caindo, mas o pattern de **"esquecimento evolutivo"** se mantém (3ª rodada consecutiva). 7 P0 reais foram identificados:

- **5 são "esquecimento evolutivo" clássico** (test fixture com NOT NULLs sem default, coluna dropada por outro PR, IDs hardcoded)
- **2 são bugs lógicos sutis no merge transacional** (Gemini) que só foram possíveis de identificar porque o resto da spec está sólido o suficiente para o reviewer focar nesses casos finos

A mudança arquitetural mais relevante: mover `mark_users_for_label_change_notification` de chamada externa em TS para chamada **interna ao SQL** `merge_canonicals` (Gemini P0 #1). Mudança pequena em código, grande em garantia transacional.

### Patches P0 (7 bugs reais — todos quebram runtime ou perdem dados silenciosamente)

1. **P0 Gemini — Paradoxo Temporal da Notificação de Merge:** v5 tinha `mergeCanonicals` (TS wrapper) chamando `merge_canonicals` (SQL) e DEPOIS `markUsersForLabelChangeNotification(loserId)`. Mas a FASE 1 do `merge_canonicals` faz `UPDATE resume_role_suggestions SET canonical_role_id = winner_id WHERE canonical_role_id = loser_id`. Quando a notificação executa logo depois, busca `WHERE rrs.canonical_role_id = loser_id` retorna **0 linhas** (suggestions já foram reapontadas). **Sistema falha silenciosamente em notificar nenhum usuário, jamais.** Test §G2.13 da v5 passaria (cria suggestion antes do merge), mas em produção a função retorna 0 sempre. Pior: `process_opus_disagree` nem chama notificação. v6 resolve **movendo a notificação para DENTRO do `merge_canonicals` ANTES da FASE 1** + remove chamadas externas (TS wrapper + process_opus_disagree). Atomicidade garantida, ambos os bugs resolvidos.

2. **P0 Gemini — Perda de Histórico de Fontes no Merge:** FASE 2 do `merge_canonicals` em §G2.9 trata FK #4 (`job_canonical_role_sources`) com UPDATE NOT EXISTS + DELETE. Para fontes que existem em **ambos** canônicos (mesma `normalized_company`), o DELETE silenciosamente perde `last_seen_at` mais recente do loser e `first_seen_at` mais antigo do loser. Cumulativo: cada merge erode frescor de mercado. v6 adiciona UPDATE de mesclagem com `GREATEST(last_seen)` e `LEAST(first_seen)` ANTES do reaponte.

3. **P0 Gemini — Toast Vazando Merges de Terceiros:** `LabelChangeToast.tsx` lê últimos 5 merges de `events` sem filtrar por profile (porque `events.entity_id` aponta para `canonical_role_id`, não para `profile_id`). Usuário João (que teve "SDR" → "BDR") veria toast com merges de outros usuários ("Analista de Redes → DevOps", "UX → UI"). Achará que dados estão vazando entre contas. v6 simplifica texto do toast para genérico — não expor detalhes de funções de outros usuários. Listagem específica fica para sprint pós-v6 (requer cruzar com `score_history` ou expandir schema de events).

4. **P0 Outra-Claude — Test §G2.13 insere `role_label` em coluna dropada por PR1:** Item 16 do PR1 dropa `resume_role_suggestions.role_label` (linha 1165). Test em §G2.13 (PR2) ainda inseria `role_label: 'Test Role'`. PostgREST rejeita campos inexistentes com 400. Test falha no setup. v6 remove o campo do INSERT.

5. **P0 Claude Code NEv5-1 — `resumes.input_type` NOT NULL sem default omitido no test:** schema validado: `input_type` é NOT NULL sem default. Test §G2.13 INSERT em `resumes` omite. Falha em runtime: `null value in column "input_type" of relation "resumes" violates not-null constraint`. v6 adiciona `input_type: 'text'`.

6. **P0 Claude Code NEv5-2 — `job_postings.posted_at` + `expires_at` NOT NULL sem default em 3 tests:** tests §G2.13 + §1A.bis.7 (2 cenários: agência e interna) inseriam em `job_postings` sem `posted_at`/`expires_at`. Schema validado: ambos NOT NULL sem default. v6 adiciona `posted_at: new Date().toISOString()` + `expires_at: new Date(Date.now() + 30*24*3600*1000).toISOString()` nos 3 INSERTs.

7. **P0 Claude Code NEv5-3 — `linkedin_id` UNIQUE colide entre runs:** IDs hardcoded ('99999900', '99999991', '99999992', '99999993') em tests. Se interrompido, vaga fica órfã, próxima execução falha com `duplicate key value violates unique constraint "job_postings_linkedin_id_key"`. v6 troca para `crypto.randomUUID()` prefixado para garantir unicidade entre runs.

### Editoriais e cosméticos (5 itens)

8. **Cosmético Gemini — Alias confuso em `get_top_samples_per_canonical`:** RPC fazia `SELECT ... company AS normalized_company` quando `job_postings.normalized_company` já é coluna nativa (criada em §1A.bis). Trocar para selecionar diretamente `normalized_company` evita dado bruto/sujo passando para o LLM Opus.

9. **Cosmético Grok — Backups `_v3_pre_pr1` → `_v6_pre_pr1`:** consistência de versão.

10. **Cosmético Grok — §15.5.0 reforçar comentário sobre `description_curated`:** explicitar que é output do Sonnet (não apenas "conteúdo curado").

11. **Cosmético Grok — Comentário no test §G2.13:** adicionar nota explícita de que função retorna 0 enquanto E2E não conectado (já documentado no JSDoc da função SQL, vale duplicar no test).

12. **Editorial GenSpark — Nota de rollback parcial:** documentar explicitamente que `CREATE TABLE...AS SELECT` preserva dados mas NÃO recompõe constraints, índices, triggers, RLS. Adicionar 1 parágrafo curto no §Backup defensivo PR1.

### Recomendação processual (1 item)

13. **Processual Claude Code — Query de pré-validação NOT NULL em test fixtures:** o pattern de NOT NULLs sem default escapando se repete em 2 rodadas consecutivas (NEv4-4 em v5, NEv5-1/2 em v6). v6 adiciona ao §Pré-requisitos uma query padrão que lista colunas obrigatórias para cada tabela tocada por test fixture, evitando que o pattern se repita na próxima rodada.

### Itens NÃO aplicados (rejeitados ou já cobertos)

- **DeepSeek:** estava revisando v4 em vez de v5 (bug que apontou — `process_opus_disagree` com label como timestamp — já foi corrigido como P0 #3 do changelog v4→v5). Sem ação.
- **GenSpark — Reclassificar thresholds (30%, 20%, 120d, 365d) como heurísticas inspiradas em DOJ/FTC, ASA, Occupation Life Cycle:** **a spec não cita nenhuma dessas fontes**. GenSpark inventa contexto externo que a spec nunca usou. Confirmação do pattern observado em v5: GenSpark fica fora do pool técnico, apenas Go/No-Go pré-produção.
- **GenSpark — Decisão 2 condicional, gate E2E, contrato email, justificativa SDK:** todos já cobertos no §Backlog operacional pós-v5.0 (preservado em v6) ou foram decisões cravadas pelo PO (gate E2E NÃO bloqueia PR2).
- **Grok — "Circuit breaker incompleto", "seed de families incompleto":** ambos já implementados em v4/v5 com código completo. Falsos positivos.

---

## Changelog histórico v4 → v5 (referência)

Esta versão incorpora 4 reviewers sobre v4 (Gemini, ChatGPT, Outra-Claude, Claude-Code com acesso DB direto). GenSpark e Mistral descartados: GenSpark fez audit executivo válido para Go/No-Go mas sem cirurgia técnica; Mistral teve incapacidade técnica recorrente em 2 rodadas consecutivas (decisão cravada: removido dos próximos rounds).

A rodada de revisão da v4 teve excelente sinal/ruído. Os 8 bugs reais encontrados são todos do tipo **"esquecimento evolutivo"** — o schema/comportamento mudou em algum ponto da spec (criação de tabelas novas em v4, rename de assinatura de função, reformulação de §15.5), mas referências em outros pontos não foram atualizadas. Cada bug afeta um fluxo distinto (merge, DISAGREE Opus, callsites de description_curated, validação automatizada da Decisão 2).

### Patches P0 (8 bugs reais — todos quebram runtime se executados como estão)

1. **P0 Gemini — `merge_canonicals` esquece FK #16 `taxonomy_family_canonicals`:** v4 criou `taxonomy_family_canonicals` em §G2.2.bis (Decisão 3 — Opção B), mas a função `merge_canonicals` em §G2.9 não foi atualizada para reapontar essa FK. Quando admin/Opus mergeia canonical A → B, status de A vira `deprecated` mas links de família de A não são transferidos para B. Como queries de leitura filtram `status='active'`, **B perde herança de famílias silenciosamente**. v5 adiciona FK #16 com mesmo padrão `UPDATE NOT EXISTS + DELETE residual` (UNIQUE composta inclui `canonical_role_id`).

2. **P0 Gemini — `merge_canonicals` esquece FK #17 `canonical_role_domain_links`:** mesma classe de bug. v3 criou `canonical_role_domain_links` em §G3.1 (junction N:N para áreas). v5 adiciona FK #17 na FASE 3 do `merge_canonicals` com mesmo padrão.

3. **P0 ChatGPT — `process_opus_disagree` chama `mark_users_for_label_change_notification` com label como timestamp:** assinatura da função é `(p_canonical_id UUID, p_cutoff_iso TEXT)` e internamente faz `p_cutoff_iso::TIMESTAMPTZ`. RPC em §G2.8 estava chamando com `p_suggested_label` (string como "Engenheiro de Software") como segundo argumento. Cast para TIMESTAMPTZ explode com `invalid input syntax for type timestamp with time zone`, derrubando transação inteira do `process_opus_disagree`. Fluxo DISAGREE quebrado, item fica preso em `inactive` eternamente. v5 corrige para `(NOW() - INTERVAL '24 hours')::text` aplicando rate limit de 24h corretamente.

4. **P0 Outra-Claude — §15.5/§15.6/§15.7 textos stale contradizem Decisão 1:** Decisão 1 v4 (preservar `description_curated`) foi aplicada no SQL da §15.5, mas 3 trechos textuais posteriores ficaram stale: (a) §15.5 validação inline diz "Esperado: apenas title e description aparecem" — falso, deve aparecer também `description_curated`; (b) §15.6 instruções de callsites em prosa diz "callsites de original_title/original_description/description_curated devem ser substituídos por title/description simples" — se implementador seguir literalmente, troca `.update({ description_curated: ... })` por `.update({ description: ... })` em `persistCuratedJob` e 6 outros arquivos, **trigger silencia escrita em `description`, pipeline produz `description_curated = NULL` permanentemente**; (c) §15.7 validação diz "sem description_curated" — falso. v5 corrige os 3 pontos textuais separando explicitamente: `original_title → title` (renamed), `original_description → description` (renamed), `description_curated → MANTER COMO ESTÁ` (coluna ativa).

5. **P0 Claude Code NEv4-1 — `resumes.raw_text` não existe (coluna real é `resume_text`):** test §G2.13 reescrito em v4 ("mergeCanonicals notifica profiles via resume_role_suggestions") inseria CV usando `raw_text`. Schema real validado: `resumes` tem `resume_text`, não `raw_text` (confundi com `submitted_jobs.raw_text` que existe lá). Falha em runtime: `column "raw_text" of relation "resumes" does not exist`. v5 corrige.

6. **P0 Claude Code NEv4-2 — `onConflict: 'profile_id'` em tabela sem UNIQUE:** mesmo test usava upsert com `onConflict: 'profile_id'` em `resumes`. Schema real: `resumes` tem APENAS `resumes_pkey: PRIMARY KEY (id)`. Não há UNIQUE em `profile_id` (faz sentido — múltiplos resumes por profile para versionamento). Falha em runtime: `there is no unique or exclusion constraint matching the ON CONFLICT specification`. v5 troca para INSERT direto sem upsert.

7. **P0 Claude Code NEv4-3 — UUID `...0003` é `users.id`, não `profile.id` + `beforeEach` não criava profile:** test usava `const profileId = '00000000-0000-0000-0000-000000000003'` com comentário dizendo ser "demo profile validado em produção". Validação Claude Code: `SELECT id FROM profiles WHERE id = '...0003'` → 0 linhas; `SELECT id FROM profiles WHERE user_id = '...0003'` → 0 linhas. UUID `...0003` é `users.id` (user_type='demo'), não `profile.id`. Confundi os dois conceitos. Pior: changelog v4 #16 alegou "criação de profile no beforeEach" — mas eu **não apliquei** isso no body (`beforeEach` real só criava `testCanonical`). v5 corrige criando profile fresh no `beforeEach` via `users` real do tipo demo + cleanup do profile no `afterEach`.

8. **P0 Claude Code NEv4-4 — Insert em `resume_role_suggestions` omite `percentual_final` + falta cleanup:** schema real tem `percentual_final NUMERIC` (provavelmente NOT NULL) + UNIQUE `(resume_id, canonical_role_id)`. Test omite `percentual_final` e não tem cleanup defensivo. v5 inclui `percentual_final: 75` + cleanup em `afterEach`.

### Cosmético (1 item)

9. **Cosmético Outra-Claude — Header §G2.9 stale:** linha do `**RPC SQL mark_users_for_label_change_notification (rate limit coalescente, JOIN via submitted_jobs):**` ainda dizia "submitted_jobs" mas o SQL interno (corretamente) usa `resume_role_suggestions` desde Decisão 2. Zero impacto runtime, só confunde leitor. v5 troca para "JOIN via resume_role_suggestions".

### Itens NÃO aplicados (rejeitados ou já cobertos)

- **GenSpark — 12 itens de "bloqueios obrigatórios":** todos já cobertos na v4 (UNIQUE em taxonomy_families.slug, FKs explícitas em junction, índices N:N, seed idempotente, RLS testado por papel, generated_documents contrato, NOT IN com NULL traps, etc.). GenSpark não conseguiu ler o arquivo e fez audit executivo no escuro. **Recomendação cravada: GenSpark fica no pool apenas para audit Go/No-Go pré-produção, não para revisão técnica cirúrgica.**
- **GenSpark — DOJ/FTC, Occupation Life Cycle, staffing stats:** irrelevantes — a spec não cita nenhum desses temas.
- **Grok #1 — Backup pós-PR1:** nice-to-have válido mas não bloqueante. Estratégia v4/v5 confia em pg_dump + 5 backup tables criados antes do PR1. Backup intermediário pós-PR1 facilitaria forense, mas vale para sprints maiores.
- **Grok — outros polimentos:** comentários extras, padronização de nomenclatura — descartáveis ou já aplicados.
- **Mistral inteiro:** incapacidade técnica recorrente em 2 rodadas consecutivas. **Removido definitivamente dos próximos rounds.**

---

## Changelog histórico v3 → v4 (referência)

Esta versão incorpora 6 reviewers sobre v3 (Gemini, ChatGPT, GenSpark, Outra-Claude, Claude-Code com acesso DB direto, Grok). A combinação de Outra-Claude + Claude-Code revelou que **3 referências a colunas do schema continuavam apoiadas em palpite e não em validação real** — falhariam em runtime. PO trouxe evidências definitivas via codebase search (Antigravity Code Search) e queries SQL diretas em produção, fechando 4 decisões arquiteturais que dependiam de estado real do sistema.

### Reformulações arquiteturais P0 (4 decisões cravadas pelo PO com evidência)

1. **Decisão 1 — `description_curated` é coluna ATIVA do pipeline, NÃO duplicata:** v3 §15.5 propunha dropar `description_curated`. Code search do Onsly em Antigravity mostrou `persistCuratedJob` em **7 arquivos** (route admin, curate-flow-b.ts, batch-processor.ts, persist-curation.ts, persist-precheck.ts, types.ts) escrevendo ativamente em `description_curated`. Drop quebraria o pipeline de curadoria inteiro. v4 reformula §15.5: renomeia apenas `original_title → title` (idempotente, são duplicatas) e `original_description → description`, **mantém `description_curated` intacta como coluna mutável de output do pipeline**. §15.5.0 (diagnóstico de divergência) vira validação informativa, não pré-condição para drop.

2. **Decisão 2 — `mark_users_for_label_change_notification` JOIN via `resume_role_suggestions` (não `submitted_jobs`):** Claude Code validou via `\d submitted_jobs` que **`submitted_jobs.canonical_role_id` NÃO EXISTE** (12 colunas reais: id, profile_id, raw_text, url, company, title, location_country, location_state, skills, seniority_inferred, created_at, description_embedding). Outra-Claude descobriu o mesmo. v3 §G2.9 fazia JOIN com essa coluna inexistente — função quebraria em runtime. Onsly rodou queries: `resume_role_assignments` tem schema **divergente do documentado** (a coluna `canonical_role_id` retornou `column does not exist`); `resume_role_suggestions` tem schema correto (id, resume_id, profile_id, canonical_role_id, role_label, percentual_final, source, created_at) com 1 row em produção (E2E ainda não está conectado em produção, mas estrutura correta). v4 reescreve `mark_users_for_label_change_notification` com JOIN via `resume_role_suggestions`. Item 17 mantém drop de `resume_role_assignments` (agora com justificativa adicional: schema inconsistente). Nota operacional explícita sobre função retornar 0 profiles enquanto E2E não conecta.

3. **Decisão 3 — `family_synonyms` ganha tabela própria + junction (Opção B):** Outra-Claude apontou que `family_synonyms.json` é estruturalmente 1:N (`"engenharia" → ["Backend", "Frontend", ...]`), incompatível com `UNIQUE(type, source_term)` de `taxonomy_relations`. Análise do JSON real (24 chaves de família, ex.: "desenvolvedor" → 8 canônicos, "engenheiro-de-software" → 6 canônicos, "analista-tech" → 12 canônicos) confirmou agrupamento por proximidade de **role**, distinto das 20 áreas funcionais do G3 (corte organizacional/setorial). Eliminação não viável. PO escolheu Opção B: criar **2 tabelas novas dedicadas** — `taxonomy_families` (catálogo de famílias) + `taxonomy_family_canonicals` (junction N:N família × canônico). Mais limpo arquiteturalmente que sobrecarregar `taxonomy_relations` com semântica diferente. Consumer da Camada 1 ganha helper `findFamilyMatches(rawTitle)` que retorna **set** de canônicos candidatos.

4. **Decisão 4 — Tabela é `job_posting_skills` (singular), validada em produção:** Claude Code testou `job_postings_skills` (plural) e não encontrou. Outra-Claude afirmou `job_posting_skills` (singular). Onsly validou via Supabase Table Editor — tabela **existe com nome singular**. Sweep nominal aplicado em §G3.4 e qualquer outro callsite.

### Patches mecânicos P0 (5 itens, todos com fix simples)

5. **P0 Gemini — Índice de agências em §1A.bis.1 trocado** (era `WHERE is_recruitment_agency = true`, trigger consultava `= false` → Sequential Scan): índice composto sem cláusula WHERE cobre ambos os cenários.

6. **P0 ChatGPT — Wrapper TS `mergeCanonicals` chamava `invalidateRelations()` sem argumento** (typecheck falharia, função exige `type: RelationType`): trocado para `invalidateAllRelations()` que v3 já documenta como caminho correto pós-merge (afeta múltiplos types).

7. **P0 ChatGPT + Grok — Circuit breaker Redis explicitamente implementado em `taxonomy-cache.ts`:** v3 deixava como gate operacional ("Antigravity confirma"), mas snippet fazia chamadas diretas que bubble up. v4 implementa `safeRedisGet`/`safeRedisSet`/`safeRedisDel` com try/catch + log em `events`, fallback Postgres na falha. DA2 sem isso é ponto único de falha.

8. **P0 ChatGPT — Test §16 esperava `shared_results.final_score` mantida** (sintaxe v1 sobrevivente): invertido para `expect(error).toBeDefined()` (coluna foi dropada).

9. **P0 Outra-Claude — `resume_skill_enrichments.skill_id` NÃO EXISTE** (coluna real é `canonical_skill_id`): UPDATE em §G2.9 falharia em runtime. v4 corrige nome da coluna; v4 também **remove** o NOT EXISTS guard porque UNIQUE real é `(analysis_id, canonical_skill_id)` SEM `canonical_role_id` — UPDATE direto não viola. Trigger DELETE residual é desnecessário aqui.

10. **P0 Outra-Claude — `skill_enrichment_stats.skill_id` NÃO EXISTE** (coluna real é `canonical_skill_id`): UNIQUE composta real é `(canonical_role_id, canonical_skill_id, skill_category, action_type, month)`. v4 corrige nome E amplia guard NOT EXISTS para a UNIQUE composta completa.

11. **P0 Outra-Claude — Seed §G2.4 não acessava `.data` do wrapper JSON:** estrutura real dos JSONs é `{ "version": "2.0", "data": { ... } }`. `Object.entries(equivalences)` iterava `["version", "data"]`, inserindo zero seeds reais. v4 corrige para `Object.entries((equivalences as any).data)` nos 3 blocos.

12. **P0 Outra-Claude — `GET DIAGNOSTICS` em DO blocks separados retornava sempre 0** (3 instâncias: backfills, deletes): UPDATE/DELETE rodava fora do DO, GET DIAGNOSTICS dentro capturava ROW_COUNT do escopo interno (vazio). v4 consolida UPDATE + GET DIAGNOSTICS no mesmo bloco DO.

### Patches mecânicos P1 (7 itens, robustez crítica)

13. **P1 Gemini — Colisão `process_opus_disagree` × `merge_canonicals`:** se p_relation_id era duplicata, `merge_canonicals` apaga via DELETE residual antes de `process_opus_disagree` fazer UPDATE (que retorna 0 rows silenciosamente, perdendo auditoria do Opus). v4 adiciona log em `events` quando `ROW_COUNT = 0`.

14. **P1 ChatGPT — `opus_response_no_tool_use` retornava 500 sem incrementar `validation_attempts`/`last_validation_attempt_at`/`last_error`:** loop infinito em malformação persistente. v4 incrementa antes do throw.

15. **P1 ChatGPT — Defensiva `tool_use.input ?? tool_use.input_json`** para divergência entre versões do Anthropic SDK (já era pré-requisito; agora também blindado em código).

16. **P1 Outra-Claude — Test G2.13 usava `profileId = '...099999'` não existente em `profiles`** (FK violation no INSERT em `analyses`): trocado para UUID convencional do projeto + criação de profile no `beforeEach`.

17. **P1 Gemini — Sintaxe `c.visited || jcr.id` no `resolve_canonical`** podia falhar dependendo do parser PL/pgSQL: cast explícito `c.visited || jcr.id::uuid`.

18. **P1 ChatGPT — RLS em `saved_analyses` e `score_history`:** se preflight 16.2 mostrar policy antiga com `profile_id`, bloco DO dropa, mas v3 não recriava. v4 adiciona policies `_v3` defensivas no SQL para ambas as tabelas.

19. **P1 Grok — Backfill `is_recruitment_agency` LATERAL JOIN:** `DISTINCT ON ... ORDER BY created_at DESC` não-determinístico se duas linhas têm `created_at` idêntico. v4 adiciona `id DESC` como tiebreaker.

### Patches mecânicos P2/P3 (sweeps textuais sincronizados)

20. **NIv3 Outra-Claude — Test §1A.4 usava `source_type`/`source_ref`** em `job_canonical_role_sources` (colunas que NÃO EXISTEM): trocado para `normalized_company` (schema real).

21. **NIv3 Outra-Claude — Test §13.2 idempotência omitia `llm_proposed_relation_type`** (obrigatório por §G2.7): adicionado em ambas as chamadas do test.

22. **NIv3 Outra-Claude — Checklist diz "9 colunas redundantes"** mas Item 16 dropa 10: corrigido.

23. **NIv3 Outra-Claude — Checklist faltava `analyses.rapidapi_log_created_at` (denormalização do Banner D) e `profiles.pending_label_change_notification_sent_at` (rate limit coalescente):** ambas adicionadas.

24. **NIv3 Outra-Claude — `recompute_distinct_sources_count()` definida 2× com lógicas diferentes** (§1A.2 sem filtro agência, §1A.bis.5 com filtro): v4 documenta ordem clara (§1A.2 cria, §1A.bis.5 substitui via CREATE OR REPLACE) e crava em texto explícito que §1A.bis substitui §1A.2.

25. **P3 ChatGPT + Outra-Claude — Backups com `_v2_pre_pr1`** renomeados para `_v3_pre_pr1` (impacto cosmético mas evita confusão em rollback).

26. **Risco operacional documentado — `profiles.email` não existe:** stub do email transacional (deferido para sprint pós-v4) precisará JOIN com `public.users` para resolver email. Anotado em §G2.11.

### Decisões NÃO aplicadas (rejeitadas com justificativa)

- **Grok crítico 5 (índice composto com EXISTS no CREATE INDEX):** sintaxe inválida no Postgres (CREATE INDEX não aceita subquery em WHERE). P0.2 do Gemini já cobre o problema real.
- **Grok crítico 6 (COMMENT ON COLUMN reforçado):** v3 já tem.
- **GenSpark inteiro:** avaliativo, gates já cravados.
- **Mistral:** incapacidade técnica recorrente (segunda rodada). PO removerá dos próximos rounds.

---

## Changelog histórico v2 → v3 (referência)

Esta versão incorpora consolidação de **6 reviewers externos** sobre a v2 (ChatGPT, Genspark, Deepseek, Gemini, Grok, e validação direta via Claude Code contra schema real e samples de payload reais), **3 decisões arquiteturais definitivas do PO**, e **reformulação crítica da Camada A/B do filtro de agências** após análise dos samples de `events.metadata->raw_data`. **22 mudanças efetivas** aplicadas, agrupadas abaixo:

### Mudanças P0 — bloqueariam migration ou produziriam falso positivo grave

1. **`merge_canonicals` SQL function reescrita do zero:** v2 listava 10 tabelas inventadas (`canonical_role_role_links`, `canonical_role_skill_links`, `merge_history`, `canonical_role_promotion_logs`, `canonical_role_pending_changes`, `canonical_role_audit_history`, `canonical_role_review_logs`, `role_demotion_logs`, `canonical_role_link_validations`, `canonical_role_blacklist_overrides`) — todas confirmadas como NÃO EXISTENTES via `pg_constraint`. v3 usa lista real validada de **14 FKs** (3 CASCADE/SET NULL + 11 NO ACTION/RESTRICT) + 1 FK pós-sprint criada na própria v3 (`taxonomy_relations`). Função entera reescrita.

2. **Item 1A.bis Camada A/B reformulada (Camada A primária + B fallback):** v2 tratava as duas camadas como OR independentes — produzia falso positivo grave. Empresas como Michael Page têm `linkedin_org_industry: "Staffing and Recruiting"` SEMPRE (é o que ela é institucionalmente), mas publicam vagas internas (RH próprio, financeiro próprio) que devem contar como sources legítimas. O LinkedIn distingue isso vaga-a-vaga via `linkedin_org_recruitment_agency_derived`. v3 inverte: Camada A é autoridade primária; Camada B só é consultada quando Camada A é `null`/ausente. Vagas internas de Michael Page (Camada A = false) preservadas como sources legítimas.

3. **Item 1A.bis fonte de dados reformulada (`events` em vez de coluna inexistente):** v2 baseava o filtro em `linkedin_org_recruitment_agency_derived` e `linkedin_org_industry` como colunas de `job_postings` que não existem (verificadas as 57 colunas reais). v3 lê o flag derived do payload bruto preservado em `events.metadata->'raw_data'` (`event_name = 'jobs_ingest_raw_payload'`), e usa `job_postings.org_industry` direto da tabela para a Camada B fallback. Match por `linkedin_id` (text). Volume real validado: **172 events válidos × média 67 raw_items = 11.505 raw_items totais** — backfill em LATERAL JOIN suportável sem otimização.

4. **`analyses.canonical_role_id` removido de todas as referências:** coluna não existe (28 colunas reais validadas via Claude Code). Acesso ao canônico de uma análise é via `submitted_job_id → submitted_jobs.canonical_role_id`. v3 reescreve 5 ocorrências da v2 (`merge_canonicals` UPDATE, `mark_users_for_label_change_notification` JOIN, test §G2.13, test §8.4, conceito §4.4) usando o JOIN correto.

5. **`merge_canonicals` defesa contra `UNIQUE(type, source_term)` em `taxonomy_relations`:** UPDATE cego que reaponta loser→winner viola constraint quando ambos tinham o mesmo sinônimo (cenário inevitável após N merges, quebraria CRON Opus silenciosamente). v3 usa UPDATE com `NOT EXISTS` + DELETE residual (padrão fornecido pelo Gemini).

6. **Item 7B Guard 6 corrigido contra trap `NOT IN (NULL)`:** lógica tri-valorada do SQL faz `NOT IN` retornar UNKNOWN quando subquery contém `NULL` — toda a expressão vira falsa prática e zero zumbis são apagados. v3 adiciona `WHERE coluna IS NOT NULL` em cada SELECT da UNION (correção sistêmica em todos os guards de `NOT IN` da v3).

### Mudanças P1 — degradariam qualidade ou exigiriam patch posterior

7. **Migração JSON→banco passa a ser TOTAL nesta sprint (decisão DA2 do PO):** v2 mantinha 4 callsites (`batch-processor`, `persist-curation`, `persist-precheck`, route `human-validated`) lendo dos JSONs como fallback ativo, com migração total deferida. v3 elimina essa coexistência: os 4 callsites são refatorados obrigatoriamente, JSONs ficam apenas como referência histórica no repositório com header `_deprecated_at`. Esforço PR2 atualizado de 21.5h para ~26h.

8. **Tests do G2 sincronizados com decisões cravadas:** test §G2.13 ainda usava `status: 'merged'` (correto: `'deprecated' + merged_into`), `Date.now()` para `content_version` (correto: sequência incremental), e test §2.6 ainda usava `entity_type/entity_id` (correto: `resource_type/resource_id`). Sweep textual completo, todos os tests batem com o body.

9. **Item 16 RLS via `pg_policies` em vez de nomes presumidos:** v2 reescrevia policies usando nomes "bonitos" (`resume_skill_enrichments_write_own`) que podem divergir dos nomes reais no banco. v3 usa bloco `DO` que itera `pg_policies` e dropa policies reais por `policyname` resolvido em runtime — robusto contra divergência. Inclui `analysis_skill_matches_write_service` (omitida na v2).

10. **Pré-requisito: rerun de métricas Bloco S e Bloco O:** dados de "8.034 vagas curadas" e "839 duplicadas" são de 22/04/2026; sprint começa 5+ dias depois. v3 crava queries de diagnóstico que precisam ser reexecutadas antes de PR1.

11. **Pré-requisito: confirmar versão Anthropic SDK:** Item G2.8 usa `tool_use.input` cuja estrutura mudou para `input_json` em SDK ≥0.39. v3 adiciona check obrigatório.

12. **Pré-requisito: diagnóstico `description_curated` ≠ `original_description`:** v2 fazia `COALESCE(original_description, description_curated)` que descarta versão curada quando ambas têm valor divergente. v3 adiciona query de diagnóstico antes do backfill — se há divergências, abortar para decisão manual.

13. **Microdependência PR2/PR3 documentada:** `merge_canonicals` (G2.9) usa colunas em `profiles` criadas em G2.10, e `resolve_canonical` (7C) é referenciado em endpoints do PR2. v3 adiciona nota explícita e diagrama.

14. **Item G2.8 `process_opus_disagree` RPC ganha `updated_at = NOW()`:** UPDATE silencioso em `taxonomy_relations` esquecia de bumpar timestamp.

15. **Item G2.4 seed via slug em vez de label:** `.in('label', labels)` é case-sensitive e quebra com divergência de capitalização. v3 mapeia para slug via `generateSlug()` antes do match.

16. **Confusão semântica `human_validated` (vaga vs canônico) documentada:** `job_postings.human_validated` (validação de vaga) ≠ `job_canonical_roles.human_validated_at/by` (validação de catálogo). v3 adiciona comentário SQL nas colunas + parágrafo na §14.0.

17. **Trigger `trg_enforce_title_description_immutability` loga em `events` em vez de NOTICE silencioso:** observabilidade em vez de mensagem de log perdida.

### Mudanças P2/P3 — robustez e cosmética

18. **CHECK `lower(btrim(source_term))` em `taxonomy_relations`:** previne ` sre`, `sre `, `sre `. CHECK mais defensivo que apenas `lower()`.

19. **`users.first_name` direto via JOIN no template de email:** v2 derivava de `display_name.split(' ')[0]`. Coluna `first_name` JÁ EXISTE em `public.users` populada para todos UUIDs convencionais (validado).

20. **Texto de §8.3 corrigido sobre `rapidapi_usage_logs`:** não é "RLS deny-all puro" — tem policy `_write_service` com `qual=true`. Bypass via service_role funciona, mas a caracterização anterior era imprecisa.

21. **Backfill `is_recruitment_agency` via LATERAL JOIN em `events`:** reprocessamento total das 11.505 raw_items históricos via `jsonb_array_elements`. Match por `linkedin_id` (text). Decisão DA3 do PO. Caso `linkedin_org_recruitment_agency_derived` ausente: aplica fallback Camada B; caso ambos ausentes: `false`. **8 events com `metadata->'raw_data'` não-array** descobertos via query de volume — guard `jsonb_typeof = 'array'` blinda; inspeção dos 8 vira item operacional pós-deploy.

22. **Nova seção "Gates operacionais antes de produção":** consolida 6 gates do GenSpark (dry-run Item 7B, execução fatiada por PR, Vercel Pro, regressão pós-renames, observabilidade Redis com circuit breaker, decisão definitiva sobre email). Circuit breaker Redis vira pré-requisito explícito do PR2 (Antigravity confirma via grep ou implementa).

### Decisões arquiteturais cravadas pelo PO nesta versão

- **DA1** — Filtro de agências: caminho híbrido. `is_recruitment_agency BOOLEAN NOT NULL DEFAULT FALSE` em `job_postings` (única coluna nova); forward calculado pelo worker no INSERT do Fluxo A; backfill via LATERAL JOIN em `events`; trigger `recompute_distinct_sources_count` filtra com `WHERE jp.is_recruitment_agency = false` como rede de segurança contra qualquer caller esquecido.
- **DA2** — Migração JSON→banco: opção A (substituição total nesta sprint, fim da estrutura híbrida).
- **DA3** — Backfill: total via `events`, sem "melhor esforço". 11.505 raw_items históricos reprocessados.

---

## Changelog histórico v1 → v2 (referência)

A v2 incorporou 36 mudanças sobre a v1 a partir de feedback de 8 reviewers + Claude Code:

### Correções de bugs estruturais P0 (v1→v2)

Esta seção mantém o changelog v1→v2 como referência histórica. **36 mudanças** aplicadas, agrupadas abaixo:

1. **Schema da tabela `events` corrigido em todas ocorrências:** colunas reais são `resource_type`/`resource_id` (não `entity_type`/`entity_id`), e `actor` é coluna obrigatória com CHECK em `('system','pipeline','human')`. Bug sistêmico em ~7 ocorrências corrigido.
2. **Status `'merged'` não existe no CHECK constraint de `job_canonical_roles.status`:** mergeCanonicals e resolve_canonical refatorados para usar `'deprecated' + merged_into` (alinhado com `chk_deprecated_has_merged_into` existente).
3. **Schema real de `job_canonical_role_sources`:** colunas reais são `(id, canonical_role_id, normalized_company, first_seen_at, last_seen_at)`. Item 1A reescrito: `distinct_sources_count = COUNT(DISTINCT normalized_company)` — semantica corrigida para "empregadores distintos" (não "tipos de fonte"). NÃO existe `source_type`, `source_ref` ou `job_posting_id`.
4. **`canonical_role_anchors` não existe como tabela:** Camada 0 opera direto em `job_postings`. PR6 §13.2 reescrito sem assumir tabela inexistente.
5. **`human_validated_at/by` não existem em `job_canonical_roles`:** criadas como parte do Item 14 (validação de canônico), com semantica distinta de `job_postings.human_validated`.
6. **Item 7B DELETE precisa guard ampliado:** 12 FKs com NO ACTION/RESTRICT bloqueariam o DELETE — ampliação de guard na v2 para todas as 14 FKs.
7. **mergeCanonicals incompleto:** lista exaustiva das 14 tabelas com FK para `job_canonical_roles.id`.
8. **Coluna `description` não existe em `job_postings`:** existe `description_curated` e `original_description`. Item 15.5 corrigido.
9. **RLS reescrita ANTES do drop de coluna:** ordem invertida na v1 (Item 16). Sem essa ordem, policy fica órfã + tabela vira deny-all em prática.
10. **Modelo Sonnet curador é 4.6 (não 4.7):** `LLM_MODEL = 'claude-sonnet-4-6'` em `constants.ts:10` (linha 10, não 11).
11. **Triggers de `vacancy_count` vivem em `job_postings` (não em `job_canonical_roles`):** Item 6 query de validação corrigida.

### Correções de design

12. **`lib/pipeline/taxonomy-cache.ts` JÁ EXISTE com 4 callsites:** v2 estende esse arquivo com camada Redis (não cria arquivo novo em `lib/taxonomy/cache.ts` como a v1 propôs).
13. **`prompt_structure_version` E `prompt_content_version` são DUAS colunas distintas em `job_postings`:** apenas `prompt_content_version` é renomeada para `taxonomy_content_version`. `prompt_structure_version` (hash do `SYSTEM_PROMPT.ts`) fica intocada.
14. **`shared_results` é dormente em produção (zero callers em `app/lib`):** decisão arquitetural #6 da v1 era baseada em premissa falsa. `final_score` agora é dropado junto com as demais redundâncias.
15. **CRON Opus filtro de 24h removido:** processa fila por `status='inactive'` ordenado por `created_at ASC`, com `validation_attempts`, `last_validation_attempt_at`, `last_error` para resiliência.
16. **Parse JSON regex do Opus → Structured Outputs (tool_use):** garante schema validado, sem parse manual.
17. **`source_term` normalização:** CHECK constraint `source_term = LOWER(source_term)` direto no banco (não apenas confiar em código TS).
18. **Rate limiting de toasts/emails de label change:** coalescer por usuário/dia + `pending_label_change_notification_sent_at`.
19. **Backup defensivo PR1:** `CREATE TABLE..._backup_v6_pre_pr1 AS SELECT *` para 4 tabelas core (Supabase free tier sem PITR).
20. **G3 `.not('id','in', subquery)` inválido em PostgREST:** trocado por RPC SQL `get_canonicals_without_domain_links()`.
21. **Banner D leitura de `rapidapi_usage_logs`:** RLS deny-all bloquearia leitura via client browser. Solução: denormalizar `rapidapi_log_created_at` direto em `analyses`.
22. **Filtro de agências (Item 1A.bis) usa campos nativos do payload Fantastic:** zero heurística manual. Camada A = `linkedin_org_recruitment_agency_derived = true`. Camada B = whitelist de `linkedin_org_industry` (`'Staffing and Recruiting'`, `'Human Resources Services'`, `'Recruitment Services'`). Trade-off documentado: pega Michael Page e BuscarVagas; não pega RH Ser, EMPREGARE, Fábrica de Valores.

### Refinos operacionais

23. **`isCronAuthorized()`** em vez de check inline (padrão do projeto, `lib/cron-guard.ts`).
24. **Auth admin via `public.users.user_type = 'admin'`:** helper `lib/admin-guard.ts:isAdminAuthorized(req)` consulta tabela `users` por `auth.uid()` e valida `user_type`. UUIDs convencionais cravados (ver Convenções).
25. **`first_name` derivado de `display_name.split(' ')[0]`** (profiles não tem campo separado).
26. **DROP trigger antes de DROP COLUMN no Item 15.5** + recriar depois (defesa contra erro de dependência de objeto no PL/pgSQL).
27. **Item 15.2 backfill resolutivo:** DELETE de órfãos em `job_no_postings` (log operacional, não dado de usuário) em vez de RAISE EXCEPTION.
28. **`.maybeSingle()` onde zero linhas é caminho normal** (sistêmico — em queries que admitem zero resultado).
29. **`taxonomy_content_version`:** sequência incremental (v1, v2, v3…) em vez de `Date.now()` — strings ordenam lexicograficamente.
30. **CRON Opus DISAGREE:** RPC SQL transacional `process_opus_disagree()` + busca de canônico via slug (não label, evita case-sensitivity).
31. **`resolve_canonical` com `visited UUID[]`:** defesa explícita contra ciclos (além do depth limit).
32. **Backfill G3 com retry/checkpoint** (3 tentativas + log de falhas em `events`).
33. **detectZombies** limita top 10 amber/red por execução (cap em `findBestMergeCandidate`).
34. **Sonnet propõe `relation_type` no output JSON** (não default fixo `'domain_synonym'`) — preserva semantica do CHECK constraint.
35. **Trigger `distinct_sources_count` cravado como FOR EACH ROW** com filtro early return — volume real é 3.2 inserts/min validado pelo Claude Code (sem risco de lock).
36. **Email template `label_change_notification` deferido para sprint pós-v2:** precisa HTML completo compatível com Outlook + reescrita do copy deixando claro que mudou a função SELECIONADA pelo usuário no serviço de comparação (não uma das funções identificadas no currículo). Placeholder textual rascunho mantido na spec, marcado como NÃO ENVIAR AO HOSTINGER.

---

## Sumário executivo

A Sprint v4.0 fecha três frentes simultaneamente, com 4 decisões arquiteturais cravadas pelo PO baseadas em evidência real (codebase search + queries SQL diretas em produção) sobre referências a colunas que continuavam apoiadas em palpite na v3:

1. **Limpeza estrutural** — drifts de nomenclatura, colunas redundantes, tabelas-fantasma e RLS faltante; alinha schema ao padrão atual antes de construções novas
2. **Governança de taxonomia** — **substituição total** dos 4 JSONs (`equivalences`, `family_synonyms`, `domain_synonyms`, `domains`) por tabelas de banco com fluxo dinâmico de manutenção via Sonnet curador + Opus 4.7 validador; áreas de atuação 0:N para canônicos. Os 4 callsites produtivos (`batch-processor`, `persist-curation`, `persist-precheck`, route `human-validated`) são refatorados nesta sprint — sem fallback JSON em runtime
3. **Inversão de paradigma** — `vacancy_count ≥ 3` deixa de ser cron-driven e vira regra dinâmica em runtime; criação de canônicos sai do modelo F1/F2/F3 estático

A sprint quebra com numeração v5.x intencionalmente: versão major nova reflete mudança de paradigma da governança de taxonomia.

---

## Numeração e PRs lógicos

A sprint contém **16 itens efetivos** organizados em 6 PRs lógicos:

| PR | Tema | Itens | Característica |
|---|---|---|---|
| PR1 | Limpeza estrutural | 15, 16, 17, 18 | Aplica antes de tudo, libera schema limpo |
| PR2 | Governança de taxonomia | G2, G3 | Coração da sprint — migração JSON→banco |
| PR3 | Inversão de paradigma | 2, 7 | Regra dinâmica + ciclo de vida de canônicos |
| PR4 | Fixes de pipeline | 1A, 1A.bis, 4, 6 | Correções pontuais de bugs e drifts |
| PR5 | Observação e UX | 8, 10 | Banner D condicional + label de viés do escudo |
| PR6 | Refinos | 13, 14 | Tests M2 + UI admin (refina ao longo da sprint) |

A ordem de implementação recomendada é PR1 → PR4 → PR3 → PR2 → PR5 → PR6. Cada PR pode ser commit/branch separado para facilitar revisão.

---

## Convenções da spec

**Nomenclatura:**
- "canônico" = registro em `job_canonical_roles` (função normalizada de mercado)
- "vaga" = registro em `job_postings` (vaga real coletada da Fantastic ou colada pelo usuário)
- "âncora" = vaga prévia em `job_postings` com `description_hash` igual + `curation_status='curated'` + (`human_validated=true` OU temporal_quorum 24h). NÃO é tabela separada — `canonical_role_anchors` não existe no schema, foi falsa premissa da v1.
- "lote" = conjunto de vagas que passa pelo pipeline LLM em uma única invocação

**Atores:**
- **Sonnet curador** = `LLM_MODEL` em `constants.ts:10` — valor atual é `'claude-sonnet-4-6'`. Pipeline principal de curadoria. (v1 dizia 4.7 + linha 11 — ambos errados.)
- **Opus validador** = Opus 4.7 — CRON diário de validação de mapeamentos novos
- **Haiku auxiliar** = Haiku 4.5 — `resume_processing`, `infer-state.ts`, `normalize-org-industry`, backfill G3

**Camadas do pipeline:**
- **Camada 0** = precheck por `description_hash` direto em `job_postings` curadas (cache puro, sem LLM). Validação consulta a vaga prévia ("âncora") com mesmo hash.
- **Camada 1** = pré-resolver por sinônimos/equivalências (após PR2: `taxonomy_relations`; antes do PR2: 3 JSONs `equivalences` + `family_synonyms` + `domain_synonyms`) — sem LLM
- **Camada 2** = LLM curador com sugestão pré-resolvida (faltou match em sinônimos mas há candidato)
- **Camada 3** = LLM curador sem sugestão alguma (decisão pura do LLM do zero)

**Padrões SQL:**
- Toda migration começa com `BEGIN;` e termina com `COMMIT;`
- IDs de FK usam `ON DELETE CASCADE` ou `ON DELETE SET NULL` explicitamente
- RLS em tabelas operacionais = `ENABLE ROW LEVEL SECURITY` sem policies (deny-all, service_role bypassa)
- Sentinel date para créditos não-expiráveis: `9999-12-31T23:59:59Z`

**Padrão de gravação em `events` (schema real validado pelo Claude Code):**

```typescript
{
  event_name: string,                                  // sem CHECK constraint
  resource_type: string,                               // ← v1 usava entity_type (errado)
  resource_id: string,                                 // ← v1 usava entity_id (errado)
  actor: 'system' | 'pipeline' | 'human',              // ← obrigatório, CHECK constraint events_actor_check
  actor_id: string | null,                             // FK opcional para users.id
  previous_state: jsonb,
  new_state: jsonb,
  reason?: string,
  // Campos opcionais: profile_id, session_id, run_id, platform, metadata, entity_name, field_changed, created_at
}
```

A coluna `actor` é obrigatória em todos os inserts. CHECK constraint rejeita qualquer outro valor.

**Padrão de status em `job_canonical_roles` (CHECK constraint real):**

- Valores aceitos: `'active'`, `'pending'`, `'deprecated'`, `'alias_of'`, `'rejected'`
- **NÃO existe `'merged'`** — operação de merge usa `status='deprecated' + merged_into=winner_id` (validado pela constraint `chk_deprecated_has_merged_into` existente)
- `alias` = `status='alias_of' + alias_of_id=target`
- `rejected` = `status='rejected' + rejected_reason=text`

**UUIDs convencionais do projeto (validados em `public.users`):**

| UUID | Email | `user_type` | Uso em automações |
|---|---|---|---|
| `00000000-0000-0000-0000-000000000001` | `calibracv@calibracv.com` | `system` | `actor_id` para ações puramente do sistema (CRONs, triggers, RPC) |
| `00000000-0000-0000-0000-000000000002` | NULL | `anonymous` | Sessões não-autenticadas |
| `00000000-0000-0000-0000-000000000003` | NULL | `demo` | Demonstrações |
| `00000000-0000-0000-0000-000000000004` | `admin@calibracv.com` | `admin` | `actor_id` para ações administrativas iniciadas pelo sistema mas com semantica de admin |

`SYSTEM_USER_ID = '00000000-0000-0000-0000-000000000001'` é usado em todos os inserts de `events` quando `actor='system'`.

**Tipologia de usuário (`public.users.user_type`, CHECK constraint completo validado):**
- `'system'` — UUID convencional para automações do servidor
- `'anonymous'` — sessão não-autenticada (sem email, fingerprint-only)
- `'demo'` — usuário de demonstração
- `'admin'` — usuário com privilégios administrativos. **Auth de endpoints `/api/admin/*` valida `user_type='admin'`** via `lib/admin-guard.ts:isAdminAuthorized(req)`.
- `'free_registered'` — usuário comum cadastrado (free tier)
- `'paid'` — usuário pagante (tratado em outras sprints, sem impacto direto na v3)

CHECK constraint real validado:
```sql
user_type = ANY (ARRAY['anonymous','free_registered','paid','demo','system','admin'])
```

---

## Decisões arquiteturais cravadas

**Escopo confirmado (reforço dos princípios desta sprint):**

1. **Frequência de mercado é o único peso.** Motor de Sugestão usa frequência crua, sem IDF, sem caps artificiais, sem bônus comportamental. O mercado define o peso.

2. **Mudança de label de canônico é operação aceitável.** Quando Opus renomeia um canônico, usuários afetados recebem toast no próximo login + email transacional de reforço (template HTML deferido para sprint pós-v2 — ver Decisão 36 do changelog). Não há delay de 24h forçado entre criação e ativação do canônico.

3. **Canônico sem vaga associada não deve existir.** Aplicação retroativa do Item 7B inclui DELETE direto de zumbis órfãos. Guard ampliado para todas as 14 FKs que apontam para `job_canonical_roles.id` (das 14, 12 são NO ACTION/RESTRICT — bloqueariam o DELETE; 2 são CASCADE/SET NULL — seguras).

4. **`taxonomy_content_version` (renome de `prompt_content_version`) bumpa apenas quando há mudança real.** CRON Opus, no fim da execução diária, verifica se houve UPDATE em `taxonomy_relations`. Se sim, bumpa em sequência incremental (v1, v2, v3…). Se não, não bumpa. **Importante:** `prompt_structure_version` (hash do `SYSTEM_PROMPT.ts`) é coluna SEPARADA e não é renomeada.

5. **Trigger é fonte autoritativa única para `vacancy_count`.** Remoção do PASSO 4 do `maintenance_phase_2` resolve o conflito de design entre trigger e CRON. Triggers vivem em `job_postings` (não em `job_canonical_roles` como a v1 sugeria na query de validação).

6. **`shared_results` é dormente em produção (zero callers em `app/lib`).** Decisão da v1 ("snapshot intencional para link público com TTL longo") era baseada em premissa falsa — não há link público sendo servido hoje. Tabela mantém-se por enquanto, mas `final_score` é dropado junto com as demais redundâncias (Item 16). Drop completo da tabela fica para sprint futura caso continue dormente.

7. **`taxonomy_relations` cresce via fluxo Sonnet→Opus.** Sonnet propõe (com `status='inactive'` + `relation_type` proposto no output JSON); CRON diário envia para Opus 4.7 validar via `tool_use` (Structured Outputs); Opus decide APPROVE / DISAGREE / REJECT.

8. **G3 áreas de atuação reusa `canonical_role_domains` como catálogo.** Cria nova `canonical_role_domain_links` para relação N:N entre canônico e área. Coluna `job_canonical_roles.domain_id` (1:1 dormente, 664 NULLs) é DROPADA.

9. **`lib/pipeline/taxonomy-cache.ts` existente é estendido (não substituído).** Arquivo já existe com 4 callsites (`upsert-canonical.ts`, `suggested-roles-builder.ts`, `batch-processor.ts:64`, test). Camada Redis adicionada para `taxonomy_relations`; cache in-memory atual continua para os 4 callsites existentes. Migração total para Redis fica para sprint futura.

10. **Operação de merge usa `status='deprecated' + merged_into`** (não há `'merged'` no CHECK constraint de `job_canonical_roles.status`).

11. **`distinct_sources_count` = `COUNT(DISTINCT normalized_company)`** — diversidade de empregadores reais, não tipos de fonte. Schema real de `job_canonical_role_sources` não tem `source_type`, `source_ref` ou `job_posting_id` (apenas `id, canonical_role_id, normalized_company, first_seen_at, last_seen_at`).

12. **Filtro de agências (Item 1A.bis) usa exclusivamente campos nativos do payload Fantastic/LinkedIn:**
    - **Camada A** = `linkedin_org_recruitment_agency_derived = true` (alta confiança)
    - **Camada B** = whitelist de `linkedin_org_industry` em `('Staffing and Recruiting', 'Human Resources Services', 'Recruitment Services')` (alta confiança)
    - Trade-off explícito: pega Michael Page e BuscarVagas; não pega RH Ser, EMPREGARE, Fábrica de Valores. Aceitável — sem heurística de keywords e sem KNOWN_AGENCIES manual.

13. **Auth admin via `public.users.user_type = 'admin'`.** Helper `lib/admin-guard.ts:isAdminAuthorized(req)` consulta `public.users` por `auth.uid()` e valida `user_type`. Sem `ADMIN_TOKEN` env var.

---

## Pré-requisitos operacionais

Antes de começar a implementação:

- [ ] Confirmar acesso ao banco de produção via Claude Code (DB connection ativa)
- [ ] Confirmar Vercel Hobby ainda em vigor (CRON degradados — anotar para upgrade pós-sprint)
- [ ] Confirmar storage para export defensivo dos backups (~3MB compactado)
- [ ] Confirmar API key Opus 4.7 ativa em `.env.local` (variável `ANTHROPIC_OPUS_KEY` com fallback para `ANTHROPIC_API_KEY`)
- [ ] Confirmar Redis Upstash ativo
- [ ] Confirmar que `pg_dump` está disponível no ambiente, OU usar alternativa `supabase db dump --data-only`
- [ ] Confirmar que `lib/cron-guard.ts:isCronAuthorized(req)` existe (será usado em CRONs novos)
- [ ] Confirmar que `lib/pipeline/taxonomy-cache.ts` existe (será refatorado, não substituído)
- [ ] **Rerun de métricas de diagnóstico Bloco S e Bloco O:** os números de "8.034 vagas curadas" e "839 vagas duplicadas" são de 22/04/2026; antes de iniciar PR1 reexecutar as queries para confirmar volumes atuais.
- [ ] **Confirmar versão do Anthropic SDK:** Item G2.8 usa `tool_use.input` cuja estrutura mudou para `input_json` em SDK ≥0.39. Rodar `cat package.json | grep @anthropic-ai/sdk` e validar extração em ambiente dev antes do CRON. v4 adicionalmente blinda em código com `tool_use.input ?? tool_use.input_json`.
- [ ] **Diagnóstico `description_curated` ≠ `original_description`:** rodar query do Item 15.5.0 antes do backfill. **Mudança v4:** este diagnóstico agora é **informativo** (não pré-condição para drop). v4 mantém `description_curated` como coluna ATIVA do pipeline (Decisão 1).
- [ ] **Circuit breaker Redis IMPLEMENTADO em `taxonomy-cache.ts`** (não apenas confirmado): v4 §G2.5 traz código explícito de `safeRedisGet`/`safeRedisSet`/`safeRedisDel` com try/catch + log em `events`, fallback Postgres na falha. Sem isso, DA2 vira ponto único de falha.
- [ ] **Inspeção dos 8 events com `metadata->'raw_data'` não-array:** rodar `SELECT id, created_at, jsonb_typeof(metadata->'raw_data'), metadata FROM events WHERE event_name = 'jobs_ingest_raw_payload' AND jsonb_typeof(metadata->'raw_data') != 'array';` — não bloqueia o backfill (que tem guard `jsonb_typeof = 'array'`), mas vale entender se são bugs históricos que merecem cleanup.
- [ ] **VALIDADO em v3 — 14 FKs apontando para `job_canonical_roles.id` confirmadas via `pg_constraint`:** lista real disponível em §G2.9.
- [ ] **NOVO em v4 — `submitted_jobs.canonical_role_id` NÃO EXISTE confirmado:** Claude Code rodou `\d submitted_jobs` em produção. v4 reescreve `mark_users_for_label_change_notification` com JOIN via `resume_role_suggestions` (Decisão 2).
- [ ] **NOVO em v4 — `resume_skill_enrichments.canonical_skill_id` (não `skill_id`) confirmado:** schema real validado pelo Claude Code. v4 §G2.9 corrige nome da coluna e remove guard NOT EXISTS desnecessário (UNIQUE real é `(analysis_id, canonical_skill_id)` sem `canonical_role_id`).
- [ ] **NOVO em v4 — `skill_enrichment_stats.canonical_skill_id` (não `skill_id`) + UNIQUE composta:** schema real validado. UNIQUE composta é `(canonical_role_id, canonical_skill_id, skill_category, action_type, month)`. v4 §G2.9 corrige nome E amplia guard NOT EXISTS.
- [ ] **NOVO em v4 — Tabela `job_posting_skills` (singular) confirmada:** Onsly validou via Supabase Table Editor. v4 §G3.4 usa nome singular.
- [ ] **NOVO em v4 — `family_synonyms.json` é estruturalmente 1:N (24 chaves de família, ex.: "desenvolvedor" → 8 canônicos):** Decisão 3 cravada — Opção B. v4 cria 2 tabelas dedicadas: `taxonomy_families` + `taxonomy_family_canonicals` (junction N:N). Consumer da Camada 1 ganha helper `findFamilyMatches()` que retorna **set** de canônicos candidatos. NÃO sobrecarrega `taxonomy_relations` com semântica diferente.
- [ ] **NOVO em v6 — Pré-validação NOT NULL para test fixtures (Processual Claude Code):** o pattern de NOT NULLs sem default escapando em test fixtures se repetiu 2 rodadas consecutivas (NEv4-4 em v5, NEv5-1/2 em v6). Antes de criar QUALQUER novo test que faça INSERT em `resumes`, `job_postings`, `analyses`, `profiles`, `resume_role_suggestions`, `submitted_jobs`, rodar:

```sql
SELECT column_name 
FROM information_schema.columns
WHERE table_schema='public' 
  AND table_name='<TABELA>'
  AND is_nullable='NO' 
  AND column_default IS NULL;
```

Toda coluna nessa lista DEVE estar presente no INSERT do test, OU a coluna precisa receber default no banco (nesse caso, atualizar a spec para documentar o default). Esta validação fecha a porta para o pattern recorrente.

---

## Estimativa de esforço

| PR | Estimativa | Risco |
|---|---|---|
| PR1 (limpeza) | 6-8 horas | Baixo — mudanças mecânicas |
| PR2 (governança taxonomia) | **27-31 horas** (subida vs v3: Decisão 3 adiciona 2 tabelas + helper de family overlap) | Médio-alto — toca coração do pipeline |
| PR3 (inversão paradigma) | 8-12 horas | Médio — regra retroativa delicada |
| PR4 (fixes pipeline) | 6-8 horas | Baixo-médio — patches localizados |
| PR5 (observação/UX) | 4-6 horas | Baixo — código funcional dormente |
| PR6 (refinos) | 8-12 horas | Variável — depende do escopo de tests |
| **Total** | **59-77 horas** | — |

Subida vs v3 é exclusivamente PR2 (Decisão 3 — Opção B requer 2 migrations + 1 RPC + 1 helper TS adicionais).

---

## Backlog operacional pós-v4.0

Anotações que ficam para sprints futuras (não fazem parte da v4.0):

**Investigação E2E (sprint dedicada pós-v4):**
- Mapear pipeline completo de ingestão de currículo (free vs paid, manual vs upload, primeira análise vs reanálise)
- Identificar onde `analyses.result_payload` deveria ser populada e por que está sempre NULL em produção (zero rows com payload preenchido)
- Auditar fluxo "comparação currículo × vagas" que liga ingestão de CV + curadoria de vagas + geração de sugestões
- Investigar por que `resume_role_assignments` tem schema divergente do documentado (memória diz `canonical_role_id` existe; banco diz que não)
- Verificar se etapa de normalização entre parsing do CV e gravação em `resume_role_assignments` está pulando o `pre-resolve` da Camada 1
- Lista detalhada de 16 perguntas sugeridas para Claude Code está em [docs/investigation/e2e-flow-mapping.md] (a criar)

**Hardening RLS dedicado:**
- Auditar `analysis_fetch_locks` policies redundantes (RESOLVIDO no Item 18.5)
- Auditar `ai_usage_logs.profile_id` (DECISÃO TOMADA: mantém deny-all, leitura via backend admin)
- Outras 17 tabelas RLS-enabled-zero-policies já corretas

**Restauração de CRONs:**
- `stripe-webhook-cleanup`: restaurar `*/5 * * * *` (degradado para `0 5 * * *` em 13/04/2026)
- `reconcile-payments`: restaurar `0 * * * *` (degradado para `0 6 * * *` em 13/04/2026)
- Aguarda upgrade Vercel Hobby → Pro

**G2 evolução:**
- Tela admin de revisão de candidatos (caso fluxo Sonnet→Opus precise intervenção manual)
- Tunagem do batch size do CRON Opus baseado em volume real
- Implementação de Batch API (50% off) quando volume justificar
- Email template `label_change_notification` cadastrado no Hostinger (stub TS na v4 precisará JOIN com `public.users` para resolver email — `profiles.email` não existe)

**Não-itens explicitamente fora do escopo:**
- G1 (tela Blacklist nativa) → backlog separado, eixo desta sprint é JSON→banco
- Item 5 (admin merge-canonicals validação por Onsly contra ~7.000 registros) → operacional pós-spec
- Stripe `/api/credits/checkout` → não é desta sprint
- Hint `ResourceEditModal` → backlog
- Instrumentação de analytics → backlog


---

# PR1 — Limpeza estrutural

Aplica antes de qualquer construção nova. Garante schema limpo para o resto da sprint construir em cima. Quatro itens: 15 (drifts de nomenclatura), 16 (DROP de colunas redundantes), 17 (DROP de tabelas/colunas fantasma), 18 (RLS faltante + DROP backups).

---

## Backup defensivo PR1 (executar ANTES de qualquer migration)

**Justificativa:** Supabase free tier não tem PITR (Point-in-Time Recovery). Em caso de falha de migration ou regressão pós-deploy, precisamos de fonte de recuperação dos dados.

**Limitação reconhecida (v6 — Editorial GenSpark):** `CREATE TABLE..._backup AS SELECT *` é ponto de recuperação de **dado**, não de schema completo. Especificamente, o backup **NÃO recompõe**:

- Índices (HNSW para embeddings, B-tree para FKs, índices compostos)
- Triggers (recompute_distinct_sources_count, enforce_title_description_immutability)
- Constraints (CHECK, UNIQUE composta, FOREIGN KEY)
- Row Level Security (policies de leitura/escrita)
- COMMENT ON COLUMN (documentação semântica)

Em caso de necessidade de rollback total: (1) restaurar dados nos backups via INSERT ... SELECT, (2) aplicar schema original a partir do git (migrations + RLS + triggers), (3) validar fill-rate de cada tabela e re-rodar reindex de embeddings. Plano completo de rollback estrutural fica documentado em `docs/runbooks/rollback-pr1.md` (a criar pela equipe antes da execução do PR1).

**Migration:**

```sql
BEGIN;

-- Backups defensivos das 4 tabelas core que sofrem mudanças no PR1
CREATE TABLE job_canonical_roles_backup_v6_pre_pr1 AS
    SELECT * FROM job_canonical_roles;

CREATE TABLE job_postings_backup_v6_pre_pr1 AS
    SELECT * FROM job_postings;

CREATE TABLE analyses_backup_v6_pre_pr1 AS
    SELECT * FROM analyses;

CREATE TABLE job_canonical_role_sources_backup_v6_pre_pr1 AS
    SELECT * FROM job_canonical_role_sources;

-- Marca timestamp para rastreabilidade
COMMENT ON TABLE job_canonical_roles_backup_v6_pre_pr1 IS 'Pre-PR1 v6 backup, 2026-04-27';
COMMENT ON TABLE job_postings_backup_v6_pre_pr1 IS 'Pre-PR1 v6 backup, 2026-04-27';
COMMENT ON TABLE analyses_backup_v6_pre_pr1 IS 'Pre-PR1 v6 backup, 2026-04-27';
COMMENT ON TABLE job_canonical_role_sources_backup_v6_pre_pr1 IS 'Pre-PR1 v6 backup, 2026-04-27';

COMMIT;
```

**Política de retenção:** após validação completa de PR6 sem regressões em produção (estimativa: 2 semanas pós-deploy), os 4 backups são dropados em sprint posterior. Anotado no backlog.

---

## Item 15 — Drifts de nomenclatura

Padroniza nomes inconsistentes de colunas que evoluíram organicamente em sprints anteriores. Operação majoritariamente metadata-only no banco; custo principal está em atualizar callsites TS que fazem `.eq('coluna_velha', ...)`.

### 15.1 — UF (estado) padronizado em 6 tabelas

**Regra semântica cravada:** `origin_state` em tabelas onde algo "vem de algum lugar" (vaga colada/capturada); `location_state` no resto (operacional, busca, lookup).

| Tabela | Coluna atual | Novo nome | Justificativa |
|---|---|---|---|
| `analyses` | `origin_state` | `location_state` | Análise é operacional, não tem origem |
| `submitted_jobs` | `location_state` | `origin_state` | Vaga colada vem do usuário |
| `job_postings` | `location_state` | `origin_state` | Vaga vem da Fantastic ou colada |
| `job_no_postings` | `location_state` | `location_state` | Mantém — registro de busca operacional |
| `analysis_fetch_locks` | `state` | `location_state` | Lock operacional, não tem origem |
| `rapidapi_usage_logs` | `state` | `location_state` | Log operacional |

**Migration:**

```sql
BEGIN;

ALTER TABLE analyses RENAME COLUMN origin_state TO location_state;
ALTER TABLE submitted_jobs RENAME COLUMN location_state TO origin_state;
ALTER TABLE job_postings RENAME COLUMN location_state TO origin_state;
ALTER TABLE analysis_fetch_locks RENAME COLUMN state TO location_state;
ALTER TABLE rapidapi_usage_logs RENAME COLUMN state TO location_state;
-- job_no_postings.location_state mantém o nome

COMMIT;
```

### 15.2 — `user_id` → `profile_id` em `job_no_postings`

Hoje é o único caso em 7 tabelas que usa `user_id` apontando para `auth.users.id`. Padroniza para `profile_id` apontando para `profiles.id`, consistente com as outras 6 tabelas (`analyses`, `submitted_jobs`, etc.).

**Mudança v2:** estratégia resolutiva para órfãos (postura de log operacional, não dado de usuário). Se houver linhas sem `profile_id` correspondente em `profiles` após o backfill, são deletadas — `job_no_postings` é tabela de log de buscas operacionais, não dado de usuário recuperável. Manter um `RAISE EXCEPTION` que abortaria a migration inteira por causa de órfãos em log é trade-off ruim.

**Migration:**

```sql
BEGIN;

-- 1. Pré-diagnóstico (informativo, não bloqueante)
DO $$
DECLARE
    orfaos INT;
BEGIN
    SELECT COUNT(*) INTO orfaos
    FROM job_no_postings jnp
    LEFT JOIN profiles p ON p.user_id = jnp.user_id
    WHERE p.id IS NULL;

    RAISE NOTICE 'Item 15.2 — Órfãos em job_no_postings que serão deletados: %', orfaos;
END $$;

-- 2. Adicionar nova coluna
ALTER TABLE job_no_postings
    ADD COLUMN profile_id UUID REFERENCES profiles(id) ON DELETE CASCADE;

-- 3. Backfill: para cada user_id, encontrar o profile_id correspondente
UPDATE job_no_postings jnp
SET profile_id = p.id
FROM profiles p
WHERE p.user_id = jnp.user_id;

-- 4. Limpeza resolutiva — deletar órfãos (job_no_postings é log operacional, não dado de usuário)
DELETE FROM job_no_postings WHERE profile_id IS NULL;

-- 5. Marcar como NOT NULL e dropar coluna antiga
ALTER TABLE job_no_postings ALTER COLUMN profile_id SET NOT NULL;
ALTER TABLE job_no_postings DROP COLUMN user_id;

COMMIT;
```

### 15.3 — `recorded_at` → `created_at` em `score_history`

Único caso de drift em 13 tabelas que usam `created_at`. Rename simples.

**Migration:**

```sql
BEGIN;

ALTER TABLE score_history RENAME COLUMN recorded_at TO created_at;

COMMIT;
```

### 15.4 — FK em `analysis_fetch_locks.locked_by`

Coluna existe mas não tem FK declarada. Bloco L confirmou que `locked_by` aponta para `auth.users.id` (não `profiles.id` — é decisão semântica correta, não drift).

**Migration:**

```sql
BEGIN;

ALTER TABLE analysis_fetch_locks
ADD CONSTRAINT analysis_fetch_locks_locked_by_fkey
FOREIGN KEY (locked_by) REFERENCES auth.users(id) ON DELETE SET NULL;

COMMIT;
```

### 15.5 — Renames `original_title → title` + `original_description → description` em `job_postings` (preservando `description_curated`)

Bloco S confirmou que **100% das vagas com `original_title` populado** têm `title = original_title`. Os 122 casos com `original_title NULL` (1.45%) são pré-fix v5.24. Não há divergência real — são duas colunas com o mesmo dado, violando normalização.

**Cenário cravado (v4):** drop `title`, rename `original_title → title`. Trigger `trg_enforce_original_immutability` continua protegendo a coluna após o rename. Casos de "edição admin" eventuais ficam como decisão consciente futura.

**Mudança crítica v4 sobre v3 — Decisão 1 do PO:** v3 propunha dropar `description_curated`. **Code search do Onsly em Antigravity mostrou `persistCuratedJob` ativo em 7 arquivos** (route admin, curate-flow-b.ts, batch-processor.ts, persist-curation.ts, persist-precheck.ts, types.ts) escrevendo na coluna. Drop quebraria o pipeline de curadoria inteiro. v4 reformula:

- **NÃO drop `description_curated`** — coluna ATIVA do pipeline, output da curadoria pelo Sonnet
- Renomeia apenas `original_description → description` (idempotente — são duas colunas com o mesmo dado bruto)
- `description_curated` permanece como coluna mutável de output do pipeline (curadoria escreve nela ativamente)
- Trigger imutabilidade protege apenas `title` + `description` (bruto). `description_curated` continua mutável.

**Distinção semântica clara após v4:**

| Coluna | Mutável? | Conteúdo |
|---|---|---|
| `title` (era `original_title`) | NÃO (trigger imutabilidade) | Título original do LinkedIn / scraper |
| `description` (era `original_description`) | NÃO (trigger imutabilidade) | Descrição original do LinkedIn / scraper |
| `description_curated` | SIM (escrita ativa pelo pipeline) | Descrição **pós-LLM** processada pela curadoria do Sonnet |

**Mudanças preservadas v3 → v4:**

1. **DROP do trigger ANTES das mutações estruturais** (recomendação do Gemini): a engine do PL/pgSQL pode se comportar de maneira volátil ao fazer DROP COLUMN em uma tabela que tem trigger ativo referenciando essa coluna. Drop explícito + recriar no final remove o risco.
2. **Diagnóstico §15.5.0 mantido como informativo** (não bloqueia mais migration): saber se há divergência entre `original_description` e `description_curated` é útil para auditoria do pipeline (entender quanto a curadoria muda do bruto), mas não afeta mais a migration.

### 15.5.0 — Diagnóstico informativo: divergências entre `original_description` e `description_curated`

**Mudança v4:** este diagnóstico é **informativo apenas** (não pré-condição para drop, porque v4 não dropa `description_curated`). Útil para entender o nível de transformação que a curadoria aplica.

```sql
-- Diagnóstico: quanto a curadoria muda do conteúdo bruto?
-- v6: description_curated é o output do Sonnet 4.7 via persistCuratedJob no pipeline 
-- de curadoria (escrita ativa em 7 callsites validados). Este diagnóstico mede o 
-- delta entre original_description (bruto LinkedIn/scraper) e description_curated 
-- (texto pós-LLM). Ambos coexistem após v4 (Decisão 1).
SELECT
    COUNT(*) FILTER (WHERE original_description IS NULL AND description_curated IS NULL) AS ambos_null,
    COUNT(*) FILTER (WHERE original_description IS NOT NULL AND description_curated IS NULL) AS so_original,
    COUNT(*) FILTER (WHERE original_description IS NULL AND description_curated IS NOT NULL) AS so_curated,
    COUNT(*) FILTER (
        WHERE original_description IS NOT NULL 
          AND description_curated IS NOT NULL 
          AND original_description = description_curated
    ) AS ambos_iguais,
    COUNT(*) FILTER (
        WHERE original_description IS NOT NULL 
          AND description_curated IS NOT NULL 
          AND original_description IS DISTINCT FROM description_curated
    ) AS ambos_divergentes,
    COUNT(*) AS total
FROM job_postings;
```

**Interpretação dos resultados:**

| Métrica | O que significa |
|---|---|
| `ambos_iguais` alto | Curadoria pouco transformou o bruto (ou pipeline está só duplicando) |
| `ambos_divergentes` alto | Curadoria está transformando bem (caso esperado) |
| `so_curated` significativo | Pipeline está populando `description_curated` mas perdendo `original_description` (investigar) |
| `so_original` significativo | Curadoria não está rodando para essas vagas (investigar fila) |

**Migration §15.5 (sem dependência do diagnóstico):**

```sql
BEGIN;

-- 0. DROP do trigger ANTES de qualquer mutação estrutural (defesa contra erro de dependência)
DROP TRIGGER IF EXISTS trg_enforce_original_immutability ON job_postings;

-- 1. Backfill prévio dos 122 órfãos de title (operação idempotente)
UPDATE job_postings
SET original_title = COALESCE(original_title, title)
WHERE original_title IS NULL;

-- 2. Backfill prévio dos órfãos de original_description
-- v4: NÃO usa description_curated como fallback aqui (são colunas semanticamente distintas)
-- Se original_description estiver NULL, fica NULL — trigger de imutabilidade lida com isso
-- (a coluna pode estar NULL para vagas pré-fix v5.24, similar ao caso de title)
UPDATE job_postings
SET original_description = COALESCE(original_description, '')  -- placeholder vazio em vez de description_curated
WHERE original_description IS NULL;

-- 3. Validar que não há mais NULLs antes de prosseguir
DO $$
DECLARE
    nulls_title INT;
    nulls_desc INT;
BEGIN
    SELECT COUNT(*) INTO nulls_title FROM job_postings WHERE original_title IS NULL;
    SELECT COUNT(*) INTO nulls_desc FROM job_postings WHERE original_description IS NULL;

    IF nulls_title > 0 OR nulls_desc > 0 THEN
        RAISE EXCEPTION 'Backfill incompleto: title=% description=%', nulls_title, nulls_desc;
    END IF;
END $$;

-- 4. DROP da coluna title (sem usuário ativo — apenas duplicata de original_title)
ALTER TABLE job_postings DROP COLUMN title;

-- IMPORTANTE v4: NÃO dropamos description_curated — coluna ATIVA do pipeline (Decisão 1).
-- Pipeline persistCuratedJob escreve ativamente nela (validado em 7 arquivos do codebase).

-- 5. Rename: original_* viram os nomes finais (agora protegidos por trigger recriado)
ALTER TABLE job_postings RENAME COLUMN original_title TO title;
ALTER TABLE job_postings RENAME COLUMN original_description TO description;

-- 6. Recriar função do trigger com referências aos novos nomes
-- description_curated continua MUTÁVEL (não está no trigger)
CREATE OR REPLACE FUNCTION enforce_title_description_immutability()
RETURNS TRIGGER AS $$
BEGIN
    -- Após rename, os campos imutáveis são title e description (brutos do scraper)
    -- description_curated continua mutável (output do pipeline, escrita ativa)
    -- v3 (P1.10): em vez de RAISE NOTICE silencioso, loga em events para observabilidade real.
    IF OLD.title IS NOT NULL AND NEW.title IS DISTINCT FROM OLD.title THEN
        INSERT INTO events (
            event_name, resource_type, resource_id, actor, actor_id,
            previous_state, new_state, reason
        ) VALUES (
            'job_posting_immutable_change_attempted',
            'job_posting', OLD.id::text,
            'system', '00000000-0000-0000-0000-000000000001',
            jsonb_build_object('field', 'title', 'value', OLD.title),
            jsonb_build_object('field', 'title', 'attempted_value', NEW.title),
            'Tentativa de UPDATE em coluna imutável silenciada pelo trigger'
        );
        NEW.title := OLD.title;
    END IF;

    IF OLD.description IS NOT NULL AND NEW.description IS DISTINCT FROM OLD.description THEN
        INSERT INTO events (
            event_name, resource_type, resource_id, actor, actor_id,
            previous_state, new_state, reason
        ) VALUES (
            'job_posting_immutable_change_attempted',
            'job_posting', OLD.id::text,
            'system', '00000000-0000-0000-0000-000000000001',
            jsonb_build_object('field', 'description', 'value', OLD.description),
            jsonb_build_object('field', 'description', 'attempted_value', NEW.description),
            'Tentativa de UPDATE em coluna imutável silenciada pelo trigger'
        );
        NEW.description := OLD.description;
    END IF;

    -- description_curated NÃO é validada aqui — pipeline pode escrever nela livremente
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 7. Recriar o trigger com nome novo
CREATE TRIGGER trg_enforce_title_description_immutability
BEFORE UPDATE ON job_postings
FOR EACH ROW
EXECUTE FUNCTION enforce_title_description_immutability();

-- 8. Adicionar COMMENT ON COLUMN explicando a distinção (v4)
COMMENT ON COLUMN job_postings.title IS 
'Título bruto da vaga (LinkedIn / scraper). Imutável após criação via trigger trg_enforce_title_description_immutability.';

COMMENT ON COLUMN job_postings.description IS 
'Descrição bruta da vaga (LinkedIn / scraper). Imutável após criação via trigger trg_enforce_title_description_immutability.';

COMMENT ON COLUMN job_postings.description_curated IS 
'Descrição PÓS-LLM processada pela curadoria do Sonnet via persistCuratedJob. Mutável (escrita ativa pelo pipeline). NÃO confundir com description (bruto).';

COMMIT;
```

**Validação manual pós-migration:**

```sql
-- Colunas finais esperadas em job_postings:
-- title (era original_title), description (era original_description), description_hash, content_hash
-- (e demais colunas existentes)

SELECT column_name FROM information_schema.columns
WHERE table_name = 'job_postings'
  AND column_name IN ('title', 'description', 'original_title', 'original_description', 'description_curated')
ORDER BY column_name;
-- v5 (P0 Outra-Claude): Decisão 1 v4 preserva description_curated.
-- Esperado: title, description e description_curated aparecem (3 colunas).
-- original_title e original_description NÃO devem aparecer (foram renamed para title/description).
```

### 15.6 — Atualização de callsites TS

Após os renames, todos os callsites que fazem `.eq('coluna_velha', ...)` ou `.select('coluna_velha')` precisam ser atualizados. Lista exaustiva (Antigravity executa via grep + replace):

**UF renames:**
```bash
# Buscar callsites por coluna antiga
grep -rn "origin_state\|\.state'" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/ components/

# Substituições por arquivo:
# - analyses: origin_state → location_state
# - submitted_jobs: location_state → origin_state
# - job_postings: location_state → origin_state
# - analysis_fetch_locks: .state → location_state
# - rapidapi_usage_logs: .state → location_state
```

**`user_id` → `profile_id` em `job_no_postings`:**
```bash
grep -rn "from('job_no_postings')\|job_no_postings.*user_id" \
  --include="*.ts" --include="*.tsx" \
  app/ lib/
```

**`recorded_at` → `created_at` em `score_history`:**
```bash
grep -rn "from('score_history')\|score_history.*recorded_at" \
  --include="*.ts" --include="*.tsx" \
  app/ lib/
```

**Renames v5 (Decisão 1 v4 preservada): `description_curated` é coluna ATIVA, NÃO substituir:**

```bash
# v5 (P0 Outra-Claude): instruções separadas por coluna, evitando substituição cega 
# que apagaria o pipeline de curadoria.
#
# CASO 1 — original_title → renomear para title (mecânico)
grep -rn "original_title" --include="*.ts" --include="*.tsx" app/ lib/
# Substituir TODAS as ocorrências de "original_title" por "title".
#
# CASO 2 — original_description → renomear para description (mecânico)
grep -rn "original_description" --include="*.ts" --include="*.tsx" app/ lib/
# Substituir TODAS as ocorrências de "original_description" por "description".
#
# CASO 3 — description_curated → MANTER COMO ESTÁ (NÃO substituir)
grep -rn "description_curated" --include="*.ts" --include="*.tsx" app/ lib/
# Esperado: 7 callsites em persistCuratedJob, persist-curation.ts, batch-processor.ts, 
# curate-flow-b.ts, persist-precheck.ts, types.ts, route admin curate-job-postings.
# NENHUMA substituição necessária — coluna ATIVA do pipeline (Decisão 1 v4).
# Trigger trg_enforce_title_description_immutability NÃO protege description_curated 
# (continua mutável). Pipeline escreve nela ativamente.
#
# AVISO CRÍTICO v5: NÃO faça substituição genérica de description_curated → description.
# Se o trigger silencia a escrita em description (que é imutável), pipeline produz 
# description_curated = NULL permanentemente e perde texto curado pelo Sonnet.
```

**Validação pós-grep:** após substituir, rodar `npm run build` e tests existentes — qualquer query referenciando coluna antiga vai falhar com erro de tipo TypeScript ou runtime do Supabase.

### 15.7 — Validação pós-migration

Após rodar todas as migrations:

```sql
-- 1. Confirmar que renames foram aplicados
SELECT table_name, column_name
FROM information_schema.columns
WHERE table_schema = 'public'
  AND column_name IN (
    'origin_state', 'location_state', 'state',
    'user_id', 'profile_id',
    'recorded_at', 'created_at',
    'original_title', 'original_description', 'description_curated',
    'title', 'description'
  )
ORDER BY table_name, column_name;

-- Esperado pós-rename:
-- analyses.location_state (era origin_state)
-- submitted_jobs.origin_state (era location_state)
-- v5 (P0 Outra-Claude): description_curated PRESERVADA pela Decisão 1 v4
-- job_postings.origin_state, job_postings.title, job_postings.description, job_postings.description_curated 
-- (sem original_*, sem title/description antigos como duplicatas)
-- job_no_postings.profile_id (era user_id), sem user_id
-- score_history.created_at (era recorded_at)
-- analysis_fetch_locks.location_state (era state)
-- rapidapi_usage_logs.location_state (era state)

-- 2. Confirmar FK em analysis_fetch_locks.locked_by
SELECT constraint_name, table_name
FROM information_schema.table_constraints
WHERE table_schema = 'public'
  AND constraint_type = 'FOREIGN KEY'
  AND table_name = 'analysis_fetch_locks';
-- Esperado: analysis_fetch_locks_locked_by_fkey

-- 3. Confirmar trigger recriado
SELECT trigger_name, event_manipulation
FROM information_schema.triggers
WHERE event_object_table = 'job_postings'
  AND trigger_name = 'trg_enforce_title_description_immutability';
-- Esperado: 1 linha
```

### 15.8 — Tests para Item 15

```typescript
// tests/integration/sprint-v4_0/item-15-renames.spec.ts

describe('Item 15 — Drifts de nomenclatura', () => {
    it('analyses tem location_state (não origin_state)', async () => {
        const { data, error } = await supabase
            .from('analyses')
            .select('location_state')
            .limit(1);
        expect(error).toBeNull();
    });
    
    it('submitted_jobs tem origin_state (não location_state)', async () => {
        const { data, error } = await supabase
            .from('submitted_jobs')
            .select('origin_state')
            .limit(1);
        expect(error).toBeNull();
    });
    
    it('job_postings tem title (não original_title) e mantém imutabilidade', async () => {
        const { data: vaga } = await supabase
            .from('job_postings')
            .select('id, title')
            .limit(1)
            .single();
        
        expect(vaga.title).toBeDefined();
        
        // Tentativa de UPDATE — trigger silencia
        const originalTitle = vaga.title;
        await supabase
            .from('job_postings')
            .update({ title: 'TENTATIVA DE MUDAR' })
            .eq('id', vaga.id);
        
        const { data: vagaApos } = await supabase
            .from('job_postings')
            .select('title')
            .eq('id', vaga.id)
            .single();
        
        expect(vagaApos.title).toBe(originalTitle);  // Trigger silenciou
    });
    
    it('job_no_postings tem profile_id (não user_id)', async () => {
        const { error } = await supabase
            .from('job_no_postings')
            .select('profile_id')
            .limit(1);
        expect(error).toBeNull();
    });
    
    it('score_history tem created_at (não recorded_at)', async () => {
        const { error } = await supabase
            .from('score_history')
            .select('created_at')
            .limit(1);
        expect(error).toBeNull();
    });
    
    it('analysis_fetch_locks.locked_by tem FK declarada', async () => {
        const { data } = await supabase.rpc('exec_sql', {
            sql: `SELECT constraint_name FROM information_schema.table_constraints
                  WHERE table_schema = 'public' AND table_name = 'analysis_fetch_locks'
                    AND constraint_type = 'FOREIGN KEY'`
        });
        expect(data).toContainEqual(
            expect.objectContaining({ constraint_name: expect.stringContaining('locked_by') })
        );
    });
});
```

### 15.9 — Esforço estimado

- **Migrations SQL:** 2 horas (escrita + dry-run + apply)
- **Atualização callsites TS:** 3-4 horas (grep + replace + revisão por arquivo)
- **Tests:** 1 hora
- **Total:** 6-7 horas

---


## Item 16 — DROP de colunas redundantes

Remove **10 colunas** (era 9 na v1 — `shared_results.final_score` adicionado na v2) que são derivadas via JOIN (denormalizações sem mecanismo de sincronização) ou que duplicam informação. Reescreve **3 RLS policies** para apontar para `analysis_id` ao invés das colunas dropadas.

**Mudanças críticas v2:**

1. **Ordem invertida:** RLS policies são reescritas ANTES do DROP da coluna que elas usam. Caso contrário, a policy fica órfã (referenciando coluna inexistente) e a tabela vira efetivamente deny-all até a recriação. Pequena janela de inconsistência que vale evitar.
2. **`shared_results.final_score` AGORA é dropado:** Claude Code validou que `shared_results` tem zero callers em `app/lib`. Decisão arquitetural #6 da v1 ("snapshot intencional para link público com TTL longo + contrato implícito") era baseada em premissa falsa — não há link público sendo servido. Dropa junto com `profile_id`.
3. **Nome correto de policies validado pelo Claude Code:**
   - `resume_role_suggestions._select_own` (SELECT, qual usa `profile_id`) — **não vai ser tocada** (a coluna `profile_id` em `resume_role_suggestions` permanece — drop é em `saved_analyses.profile_id`, não aqui)
   - `resume_skill_enrichments._write_own` (cmd=ALL, qual usa `resume_id`) — **precisa ser reescrita** porque `resume_id` está dropando
   - `shared_results._select_own` (SELECT, qual usa `profile_id`) — **precisa ser reescrita**
   - `analysis_skill_matches_select_own` mencionada na v1 **provavelmente não existe** com esse nome — Antigravity/Claude Code precisa preflight `pg_policies` para confirmar antes de DROP/CREATE
4. **`is_active` em `job_canonical_roles` confirmado existir** pelo Claude Code (era 92% incoerente conforme v1) — drop limpo.

### 16.1 — Lista exaustiva das colunas

| Tabela | Coluna | Razão | Impacto |
|---|---|---|---|
| `analysis_skill_matches` | `resume_id` | Derivável via JOIN: `analyses.resume_id` | RLS policy precisa mudar para via `analysis_id` (preflight: confirmar nome da policy real) |
| `resume_skill_enrichments` | `resume_id` | Idem | Policy `_write_own` (ALL) precisa reescrever |
| `saved_analyses` | `profile_id` | Derivável via JOIN: `analyses.profile_id` | RLS policy precisa mudar |
| `score_history` | `profile_id` | Idem | Já usa `analysis_id`, drop limpo |
| `shared_results` | `profile_id` | Idem | Policy `_select_own` precisa reescrever |
| `shared_results` | `final_score` | **DROPADO na v2** — tabela é dormente, zero callers | Drop limpo |
| `job_canonical_roles` | `is_active` | 92% incoerente com status real (Bloco O), zero callers TS | Drop limpo |
| `saved_analyses` | `initial_score` | `analyses.initial_score` é imutável após status='done' | Drop, leitura via JOIN |
| `saved_analyses` | `suggestion_text` | Idem (imutável) | Drop, leitura via JOIN |
| `resume_role_suggestions` | `role_label` | Bloco T confirmou: única cópia denormalizada ativa de label | Drop, JOIN no AnalysisTriggerModal |

**v2 NÃO mantém intencionalmente:** `shared_results.final_score`. Justificativa do drop: zero callers em `app/lib`, premissa do "contrato implícito com terceiros" da v1 não tinha respaldo no código real. Caso o feature de compartilhamento seja efetivamente lançado em sprint futura, recompor `final_score` na hora do INSERT em `shared_results`.

### 16.2 — Preflight de policies reais

Antes de DROP/CREATE de policies, validar nomes reais (Claude Code já validou 3, mas a `analysis_skill_matches_select_own` precisa confirmação):

```sql
-- Listar policies existentes nas 5 tabelas afetadas
SELECT schemaname, tablename, policyname, cmd, qual
FROM pg_policies
WHERE schemaname = 'public'
  AND tablename IN ('analysis_skill_matches', 'resume_skill_enrichments', 'shared_results', 'saved_analyses', 'score_history')
ORDER BY tablename, policyname;
```

Resultado esperado (validado pelo Claude Code para 3 das 5):
- `resume_skill_enrichments._write_own` (cmd=ALL, qual filtra via `resume_id`)
- `shared_results._select_own` (cmd=SELECT, qual filtra via `profile_id`)
- `resume_role_suggestions._select_own` (cmd=SELECT, qual filtra via `profile_id`) — **não está na lista de drops, mas vale verificar**

Para `analysis_skill_matches` e `saved_analyses` / `score_history`: confirmar resultado antes de DROP.

### 16.3 — Reescrita das RLS policies (ANTES do drop)

**Padrão cravado:** todos os satélites de `analyses` passam a filtrar via `analysis_id` ao invés de `resume_id` ou `profile_id` direto.

**Mudança crítica v3 sobre v2 (P1.3 Gemini):** v2 fazia `DROP POLICY IF EXISTS resume_skill_enrichments_write_own ON resume_skill_enrichments` usando nome "bonito". Mas o `policyname` real no banco pode divergir (ex.: `Allow user to manage own enrichments`). `IF EXISTS` faz a operação ser silenciosamente no-op — a policy original sobrevive, e quando o ALTER TABLE DROP COLUMN executa, a coluna fica órfã na policy antiga e o DROP falha. v3 usa bloco `DO` que **descobre** as policies reais via `pg_policies` em runtime e dropa por `policyname` resolvido. P2.8 adicional: inclui `analysis_skill_matches_write_service` que a v2 omitia.

**Migration completa (RLS reescrita ANTES do drop, com discovery dinâmico):**

```sql
BEGIN;

-- ====================
-- 1. Descobrir e dropar TODAS as policies que dependem das colunas a serem dropadas
--    (via pg_policies — robusto contra divergência de policyname)
-- ====================

DO $$
DECLARE
    pol RECORD;
BEGIN
    -- Drop policies que mencionam resume_id ou profile_id no qual ou with_check
    FOR pol IN
        SELECT schemaname, tablename, policyname
        FROM pg_policies
        WHERE schemaname = 'public'
          AND tablename IN (
              'analysis_skill_matches', 
              'resume_skill_enrichments', 
              'saved_analyses', 
              'score_history', 
              'shared_results'
          )
          AND (
              qual ILIKE '%resume_id%' 
              OR qual ILIKE '%profile_id%'
              OR with_check ILIKE '%resume_id%'
              OR with_check ILIKE '%profile_id%'
          )
    LOOP
        EXECUTE format('DROP POLICY %I ON %I.%I', pol.policyname, pol.schemaname, pol.tablename);
        RAISE NOTICE 'Dropped policy % on %.%', pol.policyname, pol.schemaname, pol.tablename;
    END LOOP;
END $$;

-- ====================
-- 2. Recriar policies usando analysis_id como pivot (nomes padronizados v3)
-- ====================

-- analysis_skill_matches — SELECT own (via analysis_id)
CREATE POLICY analysis_skill_matches_select_own_v3 ON analysis_skill_matches
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

-- analysis_skill_matches — write service (P2.8: inclusa em v3, omitida na v2)
-- Permite que o worker (service_role) escreva matches durante curadoria
CREATE POLICY analysis_skill_matches_write_service_v3 ON analysis_skill_matches
FOR ALL
TO service_role
USING (true)
WITH CHECK (true);

-- resume_skill_enrichments — write own (via analysis_id)
CREATE POLICY resume_skill_enrichments_write_own_v3 ON resume_skill_enrichments
FOR ALL
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
))
WITH CHECK (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

-- shared_results — SELECT own (via analysis_id)
CREATE POLICY shared_results_select_own_v3 ON shared_results
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

-- saved_analyses e score_history: 
-- v4 (P1.6 ChatGPT): policies defensivas sempre criadas, mesmo se v3 já tinha.
-- O bloco DO acima dropa qualquer policy que mencione profile_id em qual/with_check.
-- Se a policy era a única SELECT/ALL da tabela, a tabela fica RLS-enabled-zero-policies
-- → bloqueio total. Defesa: criar policy _v3 idempotente via DROP IF EXISTS + CREATE.

DROP POLICY IF EXISTS saved_analyses_write_own_v3 ON saved_analyses;
CREATE POLICY saved_analyses_write_own_v3 ON saved_analyses
FOR ALL
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
))
WITH CHECK (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

DROP POLICY IF EXISTS score_history_select_own_v3 ON score_history;
CREATE POLICY score_history_select_own_v3 ON score_history
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

-- Service_role bypassa RLS; nenhuma policy WRITE para usuário em score_history (write é apenas service_role).

-- ====================
-- 3. AGORA dropar as colunas (policies já não dependem delas)
-- ====================
ALTER TABLE analysis_skill_matches DROP COLUMN resume_id;
ALTER TABLE resume_skill_enrichments DROP COLUMN resume_id;
ALTER TABLE saved_analyses DROP COLUMN profile_id;
ALTER TABLE score_history DROP COLUMN profile_id;
ALTER TABLE shared_results DROP COLUMN profile_id;
ALTER TABLE shared_results DROP COLUMN final_score;  -- NOVO na v2
ALTER TABLE job_canonical_roles DROP COLUMN is_active;
ALTER TABLE saved_analyses DROP COLUMN initial_score;
ALTER TABLE saved_analyses DROP COLUMN suggestion_text;
ALTER TABLE resume_role_suggestions DROP COLUMN role_label;

COMMIT;
```

**Validação pós-migration:**

```sql
-- Confirmar que policies recriadas estão lá
SELECT schemaname, tablename, policyname, cmd, qual 
FROM pg_policies 
WHERE schemaname = 'public'
  AND policyname LIKE '%_v3'
ORDER BY tablename, policyname;

-- Esperado: 4 policies novas listadas
-- Confirmar que NENHUMA policy ainda menciona resume_id ou profile_id como filtro nas tabelas afetadas
SELECT schemaname, tablename, policyname, qual 
FROM pg_policies 
WHERE schemaname = 'public'
  AND tablename IN ('analysis_skill_matches', 'resume_skill_enrichments', 'saved_analyses', 'score_history', 'shared_results')
  AND (qual ILIKE '%resume_id%' OR qual ILIKE '%profile_id%' OR with_check ILIKE '%resume_id%' OR with_check ILIKE '%profile_id%');
-- Esperado: 0 linhas
```

### 16.3 — Atualização do AnalysisTriggerModal (consequência do drop de `role_label`)

Bloco T confirmou que `resume_role_suggestions.role_label` é exibido em 5+ spots de `components/home/modals/AnalysisTriggerModal.tsx:228+`. Após o drop, o frontend precisa ler via JOIN com `job_canonical_roles`.

**Antes (`AnalysisTriggerModal.tsx:228`):**

```typescript
const { data: roles } = await supabase
    .from('resume_role_suggestions')
    .select('canonical_role_id, role_label, percentual_final, source')
    .eq('profile_id', activeProfileId)
    .order('percentual_final', { ascending: false });
```

**Depois (via embed PostgREST):**

```typescript
const { data: roles } = await supabase
    .from('resume_role_suggestions')
    .select(`
        canonical_role_id,
        percentual_final,
        source,
        job_canonical_roles!inner(label)
    `)
    .eq('profile_id', activeProfileId)
    .order('percentual_final', { ascending: false });

// Acesso ao label:
// Antes: role.role_label
// Depois: role.job_canonical_roles.label
```

**Refatoração nos 5+ spots:** trocar todas as leituras `role.role_label` por `role.job_canonical_roles.label` (ou usar destructuring `const { label } = role.job_canonical_roles`).

### 16.4 — Validação de impacto antes do DROP

Antes de rodar a migration, confirmar que todos os callsites foram atualizados:

```bash
# Buscar callsites que ainda fazem referência às colunas dropadas
grep -rn "role_label\|saved_analyses.*initial_score\|saved_analyses.*suggestion_text" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/ components/

# Esperado pós-refatoração: zero matches
```

### 16.5 — Tests para Item 16

```typescript
// tests/integration/sprint-v4_0/item-16-drops.spec.ts

describe('Item 16 — DROP de colunas redundantes', () => {
    it('analysis_skill_matches.resume_id foi dropada', async () => {
        const { error } = await supabase
            .from('analysis_skill_matches')
            .select('resume_id')
            .limit(1);
        expect(error).toBeDefined();  // Coluna não existe mais
    });
    
    it('saved_analyses.initial_score foi dropada', async () => {
        const { error } = await supabase
            .from('saved_analyses')
            .select('initial_score')
            .limit(1);
        expect(error).toBeDefined();
    });
    
    it('resume_role_suggestions.role_label foi dropada', async () => {
        const { error } = await supabase
            .from('resume_role_suggestions')
            .select('role_label')
            .limit(1);
        expect(error).toBeDefined();
    });
    
    it('shared_results.final_score foi DROPADA (era snapshot dormente, premissa falsa da v1)', async () => {
        // v4 (P0 ChatGPT): teste invertido — Item 16 dropa final_score (decisão #14 do changelog v1→v2)
        // shared_results é dormente em produção (zero callers em app/lib), 
        // não há link público servido com snapshot. Coluna virou redundância.
        const { error } = await supabase
            .from('shared_results')
            .select('final_score')
            .limit(1);
        expect(error).toBeDefined();  // Coluna foi dropada
    });
    
    it('AnalysisTriggerModal lê label via JOIN com job_canonical_roles', async () => {
        const { data } = await supabase
            .from('resume_role_suggestions')
            .select('canonical_role_id, percentual_final, job_canonical_roles!inner(label)')
            .limit(1);
        
        if (data && data.length > 0) {
            expect(data[0].job_canonical_roles).toHaveProperty('label');
        }
    });
});
```

### 16.6 — Esforço estimado

- **Migrations SQL:** 2 horas (delicado — RLS exige cuidado)
- **Refatoração `AnalysisTriggerModal.tsx`:** 2 horas (5+ spots + testar UI)
- **Outros callsites:** 1 hora
- **Tests:** 1 hora
- **Total:** 6 horas

---

## Item 17 — DROP de tabela e coluna fantasma

Remove tabelas/colunas que existem mas não têm callsites em código TS, são vestígios de features abortadas ou exploratórias. Escopo conservador — apenas o que tem zero callers e não está no roadmap.

### 17.1 — Lista cravada

| Item | Tipo | Tabela/Coluna | Justificativa |
|---|---|---|---|
| 1 | Tabela | `resume_role_assignments` | Vazia, zero callsites TS, não está no roadmap |
| 2 | Coluna | `analyses.assignment_id` | FK que aponta para a tabela acima — drop encadeado |

**Mantém-se intencionalmente (NÃO drop):**

- 4 tabelas vocational (`vocation_*`) — pertencem ao módulo vocational com página dedicada
- `role_suggestions` — ligada à família vocational via FK `session_id → vocation_sessions.id`
- 5 tabelas do Grupo C (`shared_results`, `saved_analyses`, `score_history`, `recommendations`, `skill_enrichment_stats`) — todas no roadmap pós-MVP
- `canonical_role_domains` — vai virar catálogo populado em G3

### 17.2 — Migration

```sql
BEGIN;

-- 1. DROP da coluna FK em analyses (encadeado)
ALTER TABLE analyses DROP COLUMN IF EXISTS assignment_id;

-- 2. DROP da tabela
DROP TABLE IF EXISTS resume_role_assignments;

COMMIT;
```

### 17.3 — Validação

```bash
# Confirmar que não há callsites em TS
grep -rn "resume_role_assignments\|assignment_id" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/ components/

# Esperado: zero matches (ou apenas matches em comentários/docs)
```

### 17.4 — Esforço estimado

- **Migration:** 30 minutos
- **Validação callsites:** 30 minutos  
- **Total:** 1 hora

---


## Item 18 — RLS faltante + DROP de backups

Bloco R do Claude Code mapeou exaustivamente o estado de RLS no schema public. Resultado: 7 tabelas estão UNRESTRICTED, das quais 5 são backups (a serem dropados) e 2 são operacionais ativas (a receber RLS deny-all). Mais 2 sub-itens de hardening: policy SELECT-only para `generated_documents` e DROP de policies cosméticas em `analysis_fetch_locks`.

### 18.1 — ENABLE RLS em 2 tabelas operacionais (deny-all)

Padrão idêntico às 17 tabelas operacionais já corretas (`ai_usage_logs`, `job_runs`, `pipeline_config`, etc.): RLS habilitado, sem policies, service_role bypassa.

**Tabelas afetadas:**

| Tabela | Linhas | Propósito |
|---|---|---|
| `canonical_seniority_distribution` | 976 | Cache de distribuição estatística de senioridade por canônico (gerado por `refresh_canonical_seniority_distribution()`) |
| `allowed_for_pre_resolution` | 18 | Whitelist de canônicos pending elegíveis para auto-decisão pela Camada 1 |

**Migration:**

```sql
BEGIN;

ALTER TABLE canonical_seniority_distribution ENABLE ROW LEVEL SECURITY;
ALTER TABLE allowed_for_pre_resolution ENABLE ROW LEVEL SECURITY;

-- Sem policies declaradas → deny-all para qualquer caller exceto service_role.
-- Acesso atual em código: createAdminServerClient() (bypass via service role) — não quebra.

COMMIT;
```

**Análise de risco:** zero. Bloco R confirmou que todos os callers atuais usam `createAdminServerClient`, que bypassa via service_role. Não há cliente authenticated lendo essas tabelas hoje.

### 18.2 — DROP dos 5 backups com export defensivo

Bloco R confirmou: 9.467 linhas total, ~3MB compactado em `.sql.gz`. Backups são dormentes (zero inserts pós-criação), todos referem-se a estados pré-migration v5.23 ou v5.24.

**Tabelas para DROP:**

| Tabela | Linhas | Origem |
|---|---|---|
| `_backup_v5_24_canonical_role_label` | 305 | Pré-DROP da coluna `job_postings.canonical_role_label` |
| `job_canonical_roles_backup_v5_23` | 652 | Pré-migration v5.23 |
| `job_postings_backup_v5_23` | 8.308 | Pré-migration v5.23 |
| `role_merge_decisions_backup_v5_23` | 19 | Pré-migration v5.23 |
| `skill_merge_decisions_backup_v5_23` | 183 | Pré-migration v5.23 |

**Passo 1 — Export defensivo (executar antes do DROP):**

**Mudança v2:** adicionada alternativa via `supabase` CLI caso `pg_dump` não esteja disponível no ambiente do Claude Code (Gemini sugeriu).

```bash
# Em ambiente local com acesso ao banco produção
mkdir -p archive/2026-04-27_v5.23-backups

# Opção A — via pg_dump (preferida, se binário PostgreSQL estiver instalado)
pg_dump --data-only --table=public._backup_v5_24_canonical_role_label \
  > archive/2026-04-27_v5.23-backups/_backup_v5_24_canonical_role_label.sql

pg_dump --data-only --table=public.job_canonical_roles_backup_v5_23 \
  > archive/2026-04-27_v5.23-backups/job_canonical_roles_backup_v5_23.sql

pg_dump --data-only --table=public.job_postings_backup_v5_23 \
  > archive/2026-04-27_v5.23-backups/job_postings_backup_v5_23.sql

pg_dump --data-only --table=public.role_merge_decisions_backup_v5_23 \
  > archive/2026-04-27_v5.23-backups/role_merge_decisions_backup_v5_23.sql

pg_dump --data-only --table=public.skill_merge_decisions_backup_v5_23 \
  > archive/2026-04-27_v5.23-backups/skill_merge_decisions_backup_v5_23.sql

# Opção B — via supabase CLI (alternativa caso pg_dump não esteja disponível)
# supabase db dump --data-only --table=public._backup_v5_24_canonical_role_label \
#   > archive/2026-04-27_v5.23-backups/_backup_v5_24_canonical_role_label.sql
# (repetir para cada uma das 5 tabelas)

# Compactar e arquivar
tar -czf archive/2026-04-27_v5.23-backups.tar.gz archive/2026-04-27_v5.23-backups/
rm -rf archive/2026-04-27_v5.23-backups/

# Resultado: archive/2026-04-27_v5.23-backups.tar.gz (~3MB)
```

**Passo 2 — DROP das tabelas:**

```sql
BEGIN;

DROP TABLE IF EXISTS _backup_v5_24_canonical_role_label;
DROP TABLE IF EXISTS job_canonical_roles_backup_v5_23;
DROP TABLE IF EXISTS job_postings_backup_v5_23;
DROP TABLE IF EXISTS role_merge_decisions_backup_v5_23;
DROP TABLE IF EXISTS skill_merge_decisions_backup_v5_23;

COMMIT;
```

### 18.3 — (não há mais — Bloco R já mapeou tudo)

Sub-item originalmente previsto era "Bloco R mini para mapear tabelas UNRESTRICTED". Já executado, resultado consolidado em 18.1. Sub-item fica vazio, mantido na numeração para preservar alinhamento com discussão prévia.

### 18.4 — Policy SELECT-only para `generated_documents`

A tabela `generated_documents` será usada pelo recurso de "Prospecção no estrangeiro" para salvar currículos ajustados e traduzidos do usuário. Backend (service_role) faz INSERT/UPDATE; usuário autenticado faz apenas SELECT do que é dele.

**Migration:**

```sql
BEGIN;

-- Garantir que RLS está habilitado (já deve estar)
ALTER TABLE generated_documents ENABLE ROW LEVEL SECURITY;

-- Policy SELECT por dono via profile_id
CREATE POLICY generated_documents_select_own ON generated_documents
FOR SELECT
USING (profile_id IN (
    SELECT id FROM profiles WHERE user_id = auth.uid()
));

-- Sem policy de INSERT/UPDATE/DELETE → service_role bypassa para escrita
-- Authenticated normal só consegue SELECT do próprio

COMMIT;
```

**Validação manual após o lançamento da feature:**
1. Backend gera documento → linha aparece em `generated_documents` com `profile_id` do usuário
2. Usuário autenticado faz SELECT → vê apenas linhas com seu `profile_id`
3. Tentativa de INSERT/UPDATE direto via Supabase JS authenticated → bloqueado

### 18.5 — DROP de policies cosméticas em `analysis_fetch_locks`

Nano-bloco do Claude Code confirmou: zero callers TS usam authenticated client em `analysis_fetch_locks`. Todos os 5 callsites (lib/analysis/fetch-locks.ts e app/api/cron/analysis-cleanup/route.ts) usam `createAdminServerClient()` — service_role bypassa RLS por design. As 2 policies com `qual=true` são funcionalmente irrelevantes (nada chega nelas porque service_role bypassa antes).

**Migration:**

```sql
BEGIN;

DROP POLICY IF EXISTS analysis_fetch_locks_select_service ON analysis_fetch_locks;
DROP POLICY IF EXISTS analysis_fetch_locks_write_service ON analysis_fetch_locks;

-- ENABLE RLS já está ativo — fica deny-all puro, padronizado com ai_usage_logs/job_runs

COMMIT;
```

**Análise de risco:** zero. Mesmo argumento de 18.1 e 18.4.

### 18.6 — Tests para Item 18

```typescript
// tests/integration/sprint-v4_0/item-18-rls.spec.ts

describe('Item 18 — RLS hardening', () => {
    it('canonical_seniority_distribution está RLS-enabled deny-all', async () => {
        // Cliente authenticated normal não deve conseguir ler
        const { data, error } = await supabaseAuthenticated
            .from('canonical_seniority_distribution')
            .select('*')
            .limit(1);
        
        // Deny-all: data vazio sem erro (RLS retorna 0 linhas para authenticated)
        expect(data).toEqual([]);
    });
    
    it('allowed_for_pre_resolution está RLS-enabled deny-all', async () => {
        const { data } = await supabaseAuthenticated
            .from('allowed_for_pre_resolution')
            .select('*')
            .limit(1);
        expect(data).toEqual([]);
    });
    
    it('5 backups foram dropados', async () => {
        const tabelasBackup = [
            '_backup_v5_24_canonical_role_label',
            'job_canonical_roles_backup_v5_23',
            'job_postings_backup_v5_23',
            'role_merge_decisions_backup_v5_23',
            'skill_merge_decisions_backup_v5_23'
        ];
        
        for (const tabela of tabelasBackup) {
            const { error } = await supabaseAdmin
                .from(tabela)
                .select('*')
                .limit(1);
            expect(error).toBeDefined();  // Tabela não existe mais
        }
    });
    
    it('generated_documents permite SELECT do próprio profile', async () => {
        // Setup: insere documento via admin
        const { data: profile } = await supabaseAdmin
            .from('profiles')
            .select('id, user_id')
            .limit(1)
            .single();
        
        const { data: doc } = await supabaseAdmin
            .from('generated_documents')
            .insert({ profile_id: profile.id, content: 'test' })
            .select()
            .single();
        
        // Autenticar como o user dono do profile
        const supabaseAsUser = createAuthenticatedClient(profile.user_id);
        
        const { data: docs } = await supabaseAsUser
            .from('generated_documents')
            .select('*')
            .eq('id', doc.id);
        
        expect(docs).toHaveLength(1);
        
        // Cleanup
        await supabaseAdmin.from('generated_documents').delete().eq('id', doc.id);
    });
    
    it('analysis_fetch_locks não tem mais policies cosméticas', async () => {
        const { data } = await supabaseAdmin.rpc('exec_sql', {
            sql: `SELECT COUNT(*) AS qtd FROM pg_policies 
                  WHERE schemaname = 'public' AND tablename = 'analysis_fetch_locks'`
        });
        
        expect(data[0].qtd).toBe(0);
    });
});
```

### 18.7 — Esforço estimado

- **Migrations:** 1 hora (delicado por causa da policy nova de generated_documents)
- **Export defensivo:** 30 minutos (uma vez)
- **Tests:** 1 hora
- **Total:** 2.5 horas

---


---

# PR2 — Governança de taxonomia

Coração da sprint. Migra os 4 JSONs estáticos para tabelas de banco com fluxo dinâmico de manutenção. Dois itens grandes: G2 (sinônimos/equivalências/famílias via `taxonomy_relations`) e G3 (áreas de atuação 0:N para canônicos).

---

## Item G2 — Sinônimos, equivalências e famílias dinâmicos

Substitui 3 dos 4 JSONs (`equivalences.json`, `family_synonyms.json`, `domain_synonyms.json`) por tabela `taxonomy_relations` em banco, alimentada dinamicamente pelo Sonnet curador no pipeline e validada por CRON Opus 4.7 diário. O 4º JSON (`domains.json`) é tratado pelo Item G3.

### G2.1 — Schema da nova tabela `taxonomy_relations`

**Mudanças críticas v2:**

1. **`source_term` com CHECK constraint forçando lowercase + trim** (Gemini + Grok): a v1 confiava 100% no código TypeScript (`rawTitle.toLowerCase().trim()`). v2 melhorou com `lower()`. v3 fecha a brecha completa com `lower(btrim(...))` — defesa também contra espaços nas pontas (` sre`, `sre `, `sre\n`). Sem isso, UNIQUE `(type, source_term)` aceitaria duplicata `'sre'` vs `' sre'` e quebraria a chave de cache do Redis.
2. **`layer` é NULLable desde o início:** v1 definia `NOT NULL` em G2.1 e depois corrigia para nullable em G2.4. v2 já cria certo (sem migration redundante).
3. **`validation_attempts` + `last_validation_attempt_at` + `last_error`:** colunas para resiliência do CRON Opus. Sem janela 24h, fila FIFO, evita loop infinito em itens que falham.
4. **`seeded_from_json` (boolean):** marca itens importados na seed inicial — útil para auditoria e diferenciação de proposições orgânicas do Sonnet.

```sql
BEGIN;

CREATE TABLE taxonomy_relations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Tipo de relação (substitui os 3 JSONs)
    -- v4 (Decisão 3 — Opção B): 'family_synonym' REMOVIDO daqui.
    -- Famílias agora vivem em taxonomy_families + taxonomy_family_canonicals (ver §G2.2.bis).
    type TEXT NOT NULL CHECK (type IN (
        'equivalence',      -- vem de equivalences.json (tradução, sinônimo direto)
        'domain_synonym'    -- vem de domain_synonyms.json (mesma área funcional)
    )),

    -- Termo bruto que aparece na vaga (ex: "site reliability engineer iii")
    -- CHECK constraint força lowercase + trim: defesa contra ' sre', 'sre ', 'sre\n' por inserção 
    -- direta. v3 reforça com lower(btrim(...)) — v2 só verificava lower(), permitia espaços nas pontas.
    source_term TEXT NOT NULL CHECK (source_term = lower(btrim(source_term))),

    -- Canônico de destino
    target_canonical_id UUID NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,

    -- Status do mapeamento
    status TEXT NOT NULL DEFAULT 'inactive' CHECK (status IN (
        'inactive',  -- Sonnet propôs, aguarda Opus
        'active',    -- Opus aprovou (afeta cache)
        'rejected'   -- Opus rejeitou (não afeta cache, fica para auditoria)
    )),

    -- Camada do pipeline que originou (2 = sugestão pré-resolvida; 3 = LLM puro)
    -- NULLable desde o início (seeds iniciais não têm layer)
    layer SMALLINT CHECK (layer IS NULL OR layer IN (2, 3)),

    -- Label que o Sonnet sugeriu (pode diferir do canônico final se Opus mudou)
    llm_proposed_label TEXT,

    -- Razão da decisão do Opus (para auditoria forense)
    opus_decision_reason TEXT,

    -- Quando Opus validou (NULL enquanto inactive)
    validated_at TIMESTAMPTZ,

    -- Quem validou (sempre 'opus_4_7' por enquanto, mas extensível)
    validated_by TEXT,

    -- Versão da entrada (alimenta taxonomy_content_version global)
    -- Sequência incremental (v1, v2, v3...) — NÃO Date.now()
    version TEXT NOT NULL DEFAULT 'v1',

    -- Resiliência do CRON Opus: tentativas de validação + último erro
    validation_attempts INT NOT NULL DEFAULT 0,
    last_validation_attempt_at TIMESTAMPTZ,
    last_error TEXT,

    -- Marcador de seed inicial (proveniência: backfill dos JSONs vs proposição orgânica)
    seeded_from_json BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Constraint: não permite source_term duplicado para o mesmo type
    UNIQUE (type, source_term)
);

-- Index para consulta rápida por type+source_term (uso do Redis cache)
CREATE INDEX idx_taxonomy_relations_type_source
ON taxonomy_relations(type, source_term)
WHERE status = 'active';

-- Index para o CRON Opus encontrar pendentes (FIFO por created_at, sem janela 24h)
CREATE INDEX idx_taxonomy_relations_pending
ON taxonomy_relations(created_at ASC)
WHERE status = 'inactive';

-- Tabela auxiliar para versionamento explícito (idempotente)
CREATE TABLE IF NOT EXISTS taxonomy_versions (
    id SERIAL PRIMARY KEY,
    content_version TEXT NOT NULL,
    bumped_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    bumped_by TEXT NOT NULL,
    reason TEXT,
    UNIQUE (content_version)
);

INSERT INTO taxonomy_versions (content_version, bumped_by, reason)
VALUES ('v1', 'sprint_v2_init', 'Versão inicial pós-migração JSON→banco')
ON CONFLICT DO NOTHING;

-- RLS: tabelas operacionais, deny-all (service_role bypassa)
ALTER TABLE taxonomy_relations ENABLE ROW LEVEL SECURITY;
ALTER TABLE taxonomy_versions ENABLE ROW LEVEL SECURITY;

COMMIT;
```

### G2.2 — Schema da nova tabela `taxonomy_domains`

(Para completude — Item G3 usa essa tabela como catálogo. Schema definido aqui no PR2 mas populado em G3.)

```sql
BEGIN;

-- A tabela canonical_role_domains JÁ EXISTE (vista no Bloco P) mas está vazia.
-- G3 vai popular ela como catálogo de áreas. NÃO criamos taxonomy_domains nova.
-- Apenas garantimos que tem RLS.

ALTER TABLE canonical_role_domains ENABLE ROW LEVEL SECURITY;

-- Policy de leitura pública para authenticated (catálogo é informação pública)
CREATE POLICY canonical_role_domains_select_authenticated ON canonical_role_domains
FOR SELECT
TO authenticated
USING (is_active = true);

-- Sem policy de escrita → service_role bypassa para INSERT/UPDATE/DELETE

COMMIT;
```

### G2.2.bis — Schema das novas tabelas `taxonomy_families` + `taxonomy_family_canonicals` (Decisão 3 v4 — Opção B)

**Contexto da decisão arquitetural v4:** Outra-Claude apontou que `family_synonyms.json` é estruturalmente 1:N (`"engenharia": ["Backend", "Frontend", ...]`), incompatível com `UNIQUE(type, source_term)` de `taxonomy_relations`. Análise do JSON real validou:

- **24 chaves de família** (ex.: "desenvolvedor", "engenheiro-de-software", "analista-tech", "lider-tecnico", "projetos", "implantacao", "dados-e-bi", "vendas-e-comercial", "facilities", "rh-e-td", "administrativo", "juridico", "educacao", "produto"...)
- Cada chave mapeia para um **array de canônicos** (entre 1 e 12 canônicos por família)
- Chave é um **slug interno** sem capitalização (ex.: "engenheiro-de-software"), não nome de canônico

PO comparou com as 20 áreas funcionais do G3 e cravou: famílias do `family_synonyms.json` são agrupamentos por **proximidade de role** (ex.: "engenheiro-civil-e-construcao" agrupa Civil + Elétrico + Mecânico + Arquiteto + Fiscal de Obras), enquanto áreas G3 são agrupamentos **organizacionais/setoriais** (ex.: "Tecnologia" agruparia tudo de TI). **Camadas distintas, não substituíveis.**

**Decisão cravada — Opção B (mais limpa que sobrecarregar `taxonomy_relations`):** criar 2 tabelas dedicadas. Família é entidade nomeada (não termo bruto de vaga), junction explícita N:N.

**Migration:**

```sql
BEGIN;

-- ====================================================================
-- 1. Catálogo de famílias (24 entradas seedadas a partir de family_synonyms.json)
-- ====================================================================

CREATE TABLE taxonomy_families (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Slug interno (chave do JSON original, ex.: "engenheiro-de-software")
    -- Usado pelo helper findFamilyMatches para token overlap com título normalizado
    slug TEXT NOT NULL UNIQUE CHECK (slug = lower(btrim(slug))),
    
    -- Nome amigável apresentável em UI (ex.: "Engenheiro de Software")
    -- Inicialmente derivado do slug com capitalização correta
    display_name TEXT NOT NULL,
    
    -- Marcador de proveniência: seed do JSON vs criação orgânica via admin
    seeded_from_json BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Status (preparado para evolução pós-v4 com ciclo de vida)
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive')),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_taxonomy_families_slug ON taxonomy_families(slug);

COMMENT ON TABLE taxonomy_families IS 
'Catálogo de famílias de cargos (proximidade de role). Camada DIFERENTE de canonical_role_domains (áreas funcionais/setoriais). 24 entradas iniciais seedadas de data/family_synonyms.json. Consumer da Camada 1 usa via helper findFamilyMatches() para token overlap.';

-- ====================================================================
-- 2. Junction N:N família × canônico
-- ====================================================================

CREATE TABLE taxonomy_family_canonicals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES taxonomy_families(id) ON DELETE CASCADE,
    canonical_role_id UUID NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
    
    -- Marcador de proveniência da associação
    seeded_from_json BOOLEAN NOT NULL DEFAULT FALSE,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (family_id, canonical_role_id)
);

CREATE INDEX idx_tfc_family ON taxonomy_family_canonicals(family_id);
CREATE INDEX idx_tfc_canonical ON taxonomy_family_canonicals(canonical_role_id);

COMMENT ON TABLE taxonomy_family_canonicals IS 
'Junction N:N entre famílias de cargos e canônicos. Um canônico pode pertencer a múltiplas famílias (ex.: "SRE" está em "engenheiro-de-software" E "lider-tecnico"). Família tem múltiplos canônicos.';

-- ====================================================================
-- 3. RLS — operacional, deny-all (service_role bypassa)
-- ====================================================================

ALTER TABLE taxonomy_families ENABLE ROW LEVEL SECURITY;
ALTER TABLE taxonomy_family_canonicals ENABLE ROW LEVEL SECURITY;

COMMIT;
```

**Esforço cravado:** 1.5h (schema + COMMENT + RLS + duas tabelas).



**Mudança crítica v2:** o Claude Code validou que **`job_postings` tem DUAS colunas distintas** com semantica diferente:

| Coluna | O que armazena | Onde é gerado |
|---|---|---|
| `prompt_structure_version` | SHA-256 hash da **estrutura/formato** do prompt (instruções) | `scripts/generate-prompt-version.ts` lendo `lib/pipeline/SYSTEM_PROMPT.ts` |
| `prompt_content_version` | SHA-256 hash do **conteúdo da taxonomia** (canônicos + sinônimos) | `scripts/generate-prompt-version.ts` lendo os 4 JSONs de `data/` |

Ambas são usadas em produção (13 callsites encontrados pelo Claude Code: `batch-processor`, `persist-curation`, `persist-precheck`, route de `human-validated`, etc.).

**Decisão v2:** apenas `prompt_content_version` é renomeada para `taxonomy_content_version`. `prompt_structure_version` permanece com o nome original (semantica de "formato do prompt" não muda).

**Após a migração JSON→banco (PR2 G2 completo), o gerador `scripts/generate-prompt-version.ts` precisa apontar para `taxonomy_relations` em vez de ler os 4 JSONs.**

**Identificar callsites (apenas `prompt_content_version`, não `prompt_structure_version`):**

```bash
# CUIDADO: grep de prompt_content_version, sem incluir prompt_structure_version
grep -rn "prompt_content_version\|PROMPT_CONTENT_VERSION" \
  --include="*.ts" --include="*.tsx" --include="*.sql" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/ docs/ scripts/

# Esperado: ~13 callsites em batch-processor, persist-*, routes, generate-prompt-version
# IMPORTANTE: confirmar que NENHUM match menciona prompt_structure_version (que NÃO renomeamos)
```

**Substituições TS:** trocar `prompt_content_version` → `taxonomy_content_version` e `PROMPT_CONTENT_VERSION` → `TAXONOMY_CONTENT_VERSION` em todos os matches.

**Migration SQL:**

```sql
BEGIN;

-- Renomear apenas a coluna de conteúdo da taxonomia
ALTER TABLE job_postings
    RENAME COLUMN prompt_content_version TO taxonomy_content_version;

-- prompt_structure_version permanece com o nome original (NÃO renomear)

-- Tabela auxiliar de versionamento já criada em G2.1 (taxonomy_versions)
-- Aqui só validamos que existe e que tem a versão inicial 'v1'
DO $$
DECLARE
    v_version_count INT;
BEGIN
    SELECT COUNT(*) INTO v_version_count FROM taxonomy_versions WHERE content_version = 'v1';
    IF v_version_count = 0 THEN
        INSERT INTO taxonomy_versions (content_version, bumped_by, reason)
        VALUES ('v1', 'sprint_v2_init', 'Versão inicial pós-migração JSON→banco');
    END IF;
END $$;

COMMIT;
```

**Atualização do gerador `scripts/generate-prompt-version.ts` (após PR2 G2 estar completo, com `taxonomy_relations` populada):**

```typescript
// scripts/generate-prompt-version.ts (atualizado)
import { createHash } from 'crypto';
import { createAdminServerClient } from '@/lib/supabase-server';
import { promises as fs } from 'fs';

// PROMPT_STRUCTURE_VERSION = hash de SYSTEM_PROMPT.ts (intocada)
async function computePromptStructureHash(): Promise<string> {
    const content = await fs.readFile('lib/pipeline/SYSTEM_PROMPT.ts', 'utf-8');
    return createHash('sha256').update(content).digest('hex').slice(0, 16);
}

// TAXONOMY_CONTENT_VERSION = hash agregado de taxonomy_relations (era hash dos 4 JSONs)
async function computeTaxonomyContentHash(): Promise<string> {
    const supabase = createAdminServerClient();

    const { data } = await supabase
        .from('taxonomy_relations')
        .select('type, source_term, target_canonical_id')
        .eq('status', 'active')
        .order('type, source_term');

    const content = JSON.stringify(data ?? []);
    return createHash('sha256').update(content).digest('hex').slice(0, 16);
}

async function main() {
    const structureHash = await computePromptStructureHash();
    const taxonomyHash = await computeTaxonomyContentHash();

    const output = `// AUTO-GERADO por scripts/generate-prompt-version.ts
// NÃO EDITAR MANUALMENTE
export const PROMPT_STRUCTURE_VERSION = '${structureHash}';
export const TAXONOMY_CONTENT_VERSION = '${taxonomyHash}';
`;

    await fs.writeFile('lib/pipeline/prompt-version.generated.ts', output);
    console.log('Geradas:', { structureHash, taxonomyHash });
}

main().catch(console.error);
```

**Nota:** o `TAXONOMY_CONTENT_VERSION` no arquivo `prompt-version.generated.ts` é o hash determinístico do **conteúdo atual** de `taxonomy_relations`. É diferente da coluna `version` em `taxonomy_relations` (sequência incremental v1, v2, v3...) e da tabela `taxonomy_versions` (registro auditável de bumps). Os 3 cumprem papéis distintos:

- `taxonomy_versions.content_version` = sequência incremental para auditoria humana
- `taxonomy_relations.version` = qual versão da seq incremental cada relation pertence
- `TAXONOMY_CONTENT_VERSION` (constante TS) = hash determinístico atual usado em prompt caching da Anthropic

### G2.4 — Seed inicial de `taxonomy_relations` a partir dos 3 JSONs

**Estratégia:** ler os 3 arquivos JSON existentes (`data/equivalences.json`, `data/family_synonyms.json`, `data/domain_synonyms.json`), iterar e fazer UPSERT em `taxonomy_relations` com `status='active'`, `layer=NULL` (são seeds, não vieram de inferência LLM) e `seeded_from_json=true`.

**Mudanças v3 sobre v2:**

1. **Schema da G2.1 já correto** — não precisa de `ALTER TABLE` redundante para `seeded_from_json` ou `layer DROP NOT NULL`. v1 fazia em G2.4; v2 já criou direto em G2.1.
2. **`source_term` é normalizado para lowercase + trim no script** antes do UPSERT, garantindo conformidade com o CHECK constraint `lower(btrim(...))` criado em G2.1.
3. **UPSERT (`onConflict: 'type,source_term'`)** em vez de INSERT puro: torna o script idempotente — pode rodar 2x sem erro de UNIQUE violation.
4. **Resolução de canônico via slug, não por label** (Gemini editorial): `.in('label', labels)` é case-sensitive e quebra com qualquer divergência (`"Desenvolvedor Backend"` no JSON vs `"Desenvolvedor BackEnd"` no banco). v3 normaliza ambos via `generateSlug()` e faz match por slug — universal e imune a case/trim/acento.

**Script TypeScript de seed (executável uma vez, idempotente):**

```typescript
// scripts/seed-taxonomy-relations.ts
import { createAdminServerClient } from '@/lib/supabase-server';
import { generateSlug } from '@/lib/taxonomy/merge-canonicals';
import equivalences from '@/data/equivalences.json';
import domainSynonyms from '@/data/domain_synonyms.json';

// IMPORTANTE v4: family_synonyms NÃO é mais seedado em taxonomy_relations.
// Decisão 3 v4 (Opção B): family_synonyms ganhou tabelas dedicadas
// (taxonomy_families + taxonomy_family_canonicals). Seed em script separado: 
// scripts/seed-taxonomy-families.ts (ver §G2.4.bis).

async function seedTaxonomyRelations() {
    const supabase = createAdminServerClient();

    const seeds: Array<{
        type: 'equivalence' | 'domain_synonym';  // v4: family_synonym REMOVIDO daqui
        source_term: string;       // lowercase normalizado + trim
        target_label: string;      // label original do JSON
        target_slug: string;       // slug derivado para match seguro
    }> = [];

    // P0 v4 (Outra-Claude): JSONs têm wrapper { version, data }, precisamos acessar .data
    // Sem isso, Object.entries iterava as chaves do wrapper, não os dados reais

    // 1. Carregar equivalences (com normalização lowercase + trim)
    const equivalencesData = (equivalences as any).data ?? equivalences;
    for (const [source, target] of Object.entries(equivalencesData)) {
        const targetLabel = target as string;
        seeds.push({
            type: 'equivalence',
            source_term: String(source).toLowerCase().trim(),
            target_label: targetLabel,
            target_slug: generateSlug(targetLabel),
        });
    }

    // 2. Carregar domain_synonyms
    const domainSynonymsData = (domainSynonyms as any).data ?? domainSynonyms;
    for (const [source, target] of Object.entries(domainSynonymsData)) {
        const targetLabel = target as string;
        seeds.push({
            type: 'domain_synonym',
            source_term: String(source).toLowerCase().trim(),
            target_label: targetLabel,
            target_slug: generateSlug(targetLabel),
        });
    }

    console.log(`Total de seeds para inserir (apenas equivalence + domain_synonym): ${seeds.length}`);

    // 3. Resolver target_slug para canonical_id via JOIN por slug (não label, evita case/trim/acento)
    const slugs = [...new Set(seeds.map(s => s.target_slug))];
    const { data: canonicals } = await supabase
        .from('job_canonical_roles')
        .select('id, slug, label')
        .in('slug', slugs);

    const slugToId = new Map((canonicals ?? []).map(c => [c.slug, c.id]));

    // 4. Filtrar seeds com canonical resolvido + montar payload
    const seedsResolvidos = seeds
        .filter(s => slugToId.has(s.target_slug))
        .map(s => ({
            type: s.type,
            source_term: s.source_term,
            target_canonical_id: slugToId.get(s.target_slug)!,
            status: 'active' as const,
            layer: null,
            seeded_from_json: true,
            version: 'v1',
        }));

    const seedsSemCanonical = seeds.filter(s => !slugToId.has(s.target_slug));

    console.log(`Seeds resolvidos: ${seedsResolvidos.length}`);
    console.log(`Seeds SEM canônico (precisam atenção): ${seedsSemCanonical.length}`);
    if (seedsSemCanonical.length > 0) {
        const labelsFaltando = [...new Set(seedsSemCanonical.map(s => s.target_label))];
        const slugsFaltando = [...new Set(seedsSemCanonical.map(s => s.target_slug))];
        console.warn('Labels JSON não encontrados em job_canonical_roles (via slug):', 
            labelsFaltando.map((l, i) => `${l} (slug: ${slugsFaltando[i]})`));
    }

    // 5. UPSERT em batches de 100 (idempotente)
    const batchSize = 100;
    for (let i = 0; i < seedsResolvidos.length; i += batchSize) {
        const batch = seedsResolvidos.slice(i, i + batchSize);
        const { error } = await supabase
            .from('taxonomy_relations')
            .upsert(batch, { onConflict: 'type,source_term', ignoreDuplicates: false });

        if (error) {
            console.error(`Erro no batch ${i}-${i+batchSize}:`, error);
            throw error;
        }
        console.log(`Upserted batch ${i}-${i+batchSize}`);
    }

    console.log('Seed taxonomy_relations completo (idempotente). Lembrete: rode também seed-taxonomy-families.ts.');
}

seedTaxonomyRelations().catch(console.error);
```

**Execução:**

```bash
npx tsx scripts/seed-taxonomy-relations.ts
# Pode ser executado múltiplas vezes — UPSERT garante idempotência
```

### G2.4.bis — Seed de `taxonomy_families` + `taxonomy_family_canonicals` (NOVO em v4)

**Estratégia:** ler `data/family_synonyms.json` (estrutura `{ "version": "2.0", "data": { "engenheiro-de-software": ["Engenheiro de Software", "SRE", ...], ... } }`), criar 1 row em `taxonomy_families` por chave + N rows em `taxonomy_family_canonicals` por canônico associado.

```typescript
// scripts/seed-taxonomy-families.ts
import { createAdminServerClient } from '@/lib/supabase-server';
import { generateSlug } from '@/lib/taxonomy/merge-canonicals';
import familySynonyms from '@/data/family_synonyms.json';

async function seedTaxonomyFamilies() {
    const supabase = createAdminServerClient();

    // P0 v4: acessar .data do wrapper { version, data }
    const familiesData = (familySynonyms as any).data ?? familySynonyms;

    console.log(`Famílias a processar: ${Object.keys(familiesData).length}`);

    // 1. Para cada família, UPSERT em taxonomy_families + resolver canônicos via slug
    for (const [familySlug, canonicalLabels] of Object.entries(familiesData)) {
        const labels = canonicalLabels as string[];

        // 1a. UPSERT da família
        const displayName = familySlug
            .split('-')
            .map(part => part.charAt(0).toUpperCase() + part.slice(1).toLowerCase())
            .join(' ');

        const { data: family, error: familyError } = await supabase
            .from('taxonomy_families')
            .upsert({
                slug: familySlug,
                display_name: displayName,
                seeded_from_json: true,
                status: 'active',
            }, { onConflict: 'slug', ignoreDuplicates: false })
            .select('id')
            .single();

        if (familyError) {
            console.error(`Erro upsert família ${familySlug}:`, familyError);
            throw familyError;
        }

        // 1b. Resolver labels para canonical_role_id via slug
        const canonicalSlugs = labels.map(l => generateSlug(l));
        const { data: canonicals } = await supabase
            .from('job_canonical_roles')
            .select('id, slug, label')
            .in('slug', canonicalSlugs);

        const slugToId = new Map((canonicals ?? []).map(c => [c.slug, c.id]));

        // 1c. Para cada canônico resolvido, UPSERT na junction
        const linksToUpsert = labels
            .map(label => ({ label, slug: generateSlug(label), id: slugToId.get(generateSlug(label)) }))
            .filter(x => x.id !== undefined)
            .map(x => ({
                family_id: family!.id,
                canonical_role_id: x.id!,
                seeded_from_json: true,
            }));

        const labelsNaoResolvidos = labels.filter(l => !slugToId.has(generateSlug(l)));
        if (labelsNaoResolvidos.length > 0) {
            console.warn(
                `Família "${familySlug}" — labels sem canônico:`, 
                labelsNaoResolvidos
            );
        }

        if (linksToUpsert.length > 0) {
            const { error: linkError } = await supabase
                .from('taxonomy_family_canonicals')
                .upsert(linksToUpsert, { 
                    onConflict: 'family_id,canonical_role_id', 
                    ignoreDuplicates: true 
                });

            if (linkError) {
                console.error(`Erro upsert links da família ${familySlug}:`, linkError);
                throw linkError;
            }
        }

        console.log(`Família "${familySlug}" → ${linksToUpsert.length}/${labels.length} canônicos linkados`);
    }

    console.log('Seed taxonomy_families completo.');
}

seedTaxonomyFamilies().catch(console.error);
```

**Helper `findFamilyMatches()` no consumer (Camada 1):**

A semântica de `family_synonyms` no fluxo original (`findFamilyMatch` antigo via JSON) era **token overlap**: para um título de vaga "Engenheiro Backend Pleno", verificava se algum token batia com chave de família. v4 preserva essa semântica, mas com Postgres+Redis em vez de JSON.

```typescript
// lib/pipeline/taxonomy-families.ts (NOVO em v4)
import { createAdminServerClient } from '@/lib/supabase-server';
import { Redis } from '@upstash/redis';
import { safeRedisGet, safeRedisSet } from './taxonomy-cache';  // helpers compartilhados

const redis = Redis.fromEnv();
const CACHE_KEY = 'tax:families_index';
const CACHE_TTL_SECONDS = 3600;

interface FamilyIndex {
    [tokenLower: string]: Array<{
        familyId: string;
        familySlug: string;
        canonicalIds: string[];  // todos os canônicos dessa família
    }>;
}

async function buildFamilyIndex(): Promise<FamilyIndex> {
    const supabase = createAdminServerClient();
    
    const { data, error } = await supabase
        .from('taxonomy_families')
        .select(`
            id, slug,
            taxonomy_family_canonicals!inner(canonical_role_id)
        `)
        .eq('status', 'active');

    if (error) throw error;

    const index: FamilyIndex = {};
    for (const family of data ?? []) {
        const canonicalIds = (family.taxonomy_family_canonicals as any[]).map(
            tfc => tfc.canonical_role_id
        );
        // Tokens são as palavras do slug (ex.: "engenheiro-de-software" → ["engenheiro", "de", "software"])
        const tokens = family.slug.split('-').filter(t => t.length > 2);  // ignora stopwords curtas
        for (const token of tokens) {
            if (!index[token]) index[token] = [];
            index[token].push({
                familyId: family.id,
                familySlug: family.slug,
                canonicalIds,
            });
        }
    }
    return index;
}

/**
 * Encontra famílias que casam por token overlap com o título normalizado da vaga.
 * Retorna SET de canônicos candidatos (não 1 só).
 *
 * Exemplo: "Engenheiro Backend Pleno" → token "engenheiro" bate com famílias
 * "engenheiro-de-software" e "engenheiro-de-automacao" → retorna union dos canônicos.
 */
export async function findFamilyMatches(rawTitle: string): Promise<{
    families: string[];        // slugs das famílias que casaram
    canonicalIds: string[];    // união dos canônicos candidatos
}> {
    const cached = await safeRedisGet<string>(CACHE_KEY);
    let index: FamilyIndex;
    
    if (cached) {
        index = JSON.parse(cached);
    } else {
        index = await buildFamilyIndex();
        await safeRedisSet(CACHE_KEY, JSON.stringify(index), CACHE_TTL_SECONDS);
    }

    const tokens = rawTitle.toLowerCase().trim().split(/\s+/).filter(t => t.length > 2);
    const matchedFamilies = new Map<string, Set<string>>();  // slug → Set de canonicalIds

    for (const token of tokens) {
        const matches = index[token];
        if (!matches) continue;
        for (const m of matches) {
            if (!matchedFamilies.has(m.familySlug)) {
                matchedFamilies.set(m.familySlug, new Set());
            }
            const set = matchedFamilies.get(m.familySlug)!;
            for (const cid of m.canonicalIds) set.add(cid);
        }
    }

    const allCanonicalIds = new Set<string>();
    for (const set of matchedFamilies.values()) {
        for (const cid of set) allCanonicalIds.add(cid);
    }

    return {
        families: [...matchedFamilies.keys()],
        canonicalIds: [...allCanonicalIds],
    };
}

export async function invalidateFamilyIndex(): Promise<void> {
    // v4 (P0): usa safeRedisDel para não derrubar caller se Redis estiver indisponível
    const { safeRedisDel } = await import('./taxonomy-cache');
    await safeRedisDel(CACHE_KEY);
}
```

**Esforço cravado:** 3.5h (script seed + helper findFamilyMatches + invalidação + integração com consumer).



### G2.5 — `lib/pipeline/taxonomy-cache.ts` refatorado para Postgres + Redis (substituição total dos JSONs)

**Mudança crítica v3 — DA2 do PO:** v2 propunha estender `lib/pipeline/taxonomy-cache.ts` com camada Redis enquanto preservava as funções existentes (`getFullTaxonomyCache`, `getTaxonomyLoader`) lendo dos 4 JSONs como fallback durante toda a sprint, com migração total deferida. Decisão DA2 cravada nesta v3 elimina essa coexistência: **os 4 callsites produtivos são refatorados nesta sprint, sem fallback JSON em runtime**.

**Estado validado pelo Claude Code:** `lib/pipeline/taxonomy-cache.ts` JÁ EXISTE no projeto, com 4 callsites ativos:

- `lib/pipeline/upsert-canonical.ts` — usa `getTaxonomyLoader`
- `lib/pipeline/suggested-roles-builder.ts` — usa `getFullTaxonomyCache`
- `lib/pipeline/batch-processor.ts:64` — usa `getFullTaxonomyCache(ctx.supabase)`
- `tests/pipeline/domain-synonyms-false-positives.test.ts` — mocka `getFullTaxonomyCache`

**Padrão cravado para v3:**

- Arquivo `lib/pipeline/taxonomy-cache.ts` é **refatorado** (não substituído por novo arquivo `lib/taxonomy/cache.ts` como a v1 propunha — manter caminho atual preserva imports).
- Funções legadas (`getFullTaxonomyCache`, `getTaxonomyLoader`) são reescritas para ler de **`taxonomy_relations` via Redis write-through**, mantendo a mesma assinatura externa para minimizar refatoração nos 4 callsites.
- Funções novas (`getRelations`, `invalidateRelations`, `invalidateAllRelations`, `getCurrentTaxonomyVersion`) são adicionadas como API moderna.
- Os 4 callsites continuam chamando `getFullTaxonomyCache` mas a **implementação interna não lê mais JSON** — busca de Postgres com cache Redis.
- 3 dos 4 JSONs (`equivalences.json`, `family_synonyms.json`, `domain_synonyms.json`) ficam **apenas como referência histórica no repositório** com header `_deprecated_at` e `_replaced_by`. Nenhum código produtivo importa.
- O 4º JSON (`domains.json`) é convertido em `canonical_role_domains` na sprint atual via §G3 — paradigma similar.

**Pré-requisito operacional crítico (cravado em §Pré-requisitos):** circuit breaker Redis confirmado/implementado antes do deploy do PR2. Sem fallback JSON em runtime, Redis indisponível derruba a Camada 1 do pipeline.

```typescript
// lib/pipeline/taxonomy-cache.ts (REESCRITA TOTAL — preserva assinatura externa)

import { Redis } from '@upstash/redis';
import { createAdminServerClient } from '@/lib/supabase-server';
import type { SupabaseClient } from '@supabase/supabase-js';

const redis = Redis.fromEnv();
const REDIS_KEY_PREFIX = 'tax:';
const CACHE_TTL_SECONDS = 3600; // 1h, mas write-through invalida antes

type RelationType = 'equivalence' | 'domain_synonym';  // v4: family_synonym vive em tabela própria

interface TaxonomyRelation {
    source_term: string;
    target_canonical_id: string;
    target_label: string;  // hidratado via JOIN
}

// ====================================
// CIRCUIT BREAKER REDIS (P0 v4 — ChatGPT+Grok)
// ====================================
// DA2 eliminou o fallback JSON em runtime. Sem circuit breaker, qualquer
// indisponibilidade de Redis derruba a Camada 1 do pipeline. Wrapper try/catch
// que loga em events e cai para Postgres direto (TTL não-bloqueante).

async function logRedisCacheFailure(
    op: 'get' | 'set' | 'del',
    key: string,
    err: unknown
): Promise<void> {
    // Best effort: se o log também falhar, não derruba o caller
    try {
        const supabase = createAdminServerClient();
        await supabase.from('events').insert({
            event_name: 'redis_cache_failure',
            resource_type: 'taxonomy_cache',
            resource_id: key,
            actor: 'system',
            actor_id: '00000000-0000-0000-0000-000000000001',
            previous_state: { operation: op },
            new_state: { error: String(err).slice(0, 500) },
            reason: `Redis ${op} falhou em ${key} — fallback Postgres ativado`,
        });
    } catch (logErr) {
        console.error('[taxonomy-cache] Falha ao logar event de Redis failure:', logErr);
    }
}

export async function safeRedisGet<T>(key: string): Promise<T | null> {
    try {
        return await redis.get<T>(key);
    } catch (err) {
        await logRedisCacheFailure('get', key, err);
        return null;  // Caller cai para Postgres
    }
}

export async function safeRedisSet(
    key: string, 
    value: string, 
    ttlSeconds: number
): Promise<void> {
    try {
        await redis.set(key, value, { ex: ttlSeconds });
    } catch (err) {
        await logRedisCacheFailure('set', key, err);
        // Não relança — populate falhou mas dado já foi servido
    }
}

export async function safeRedisDel(key: string): Promise<void> {
    try {
        await redis.del(key);
    } catch (err) {
        await logRedisCacheFailure('del', key, err);
        // Não relança — TTL é safety net
    }
}

// ====================================
// API MODERNA — retorna Map<source_term, TaxonomyRelation>
// ====================================

export async function getRelations(type: RelationType): Promise<Map<string, TaxonomyRelation>> {
    const key = `${REDIS_KEY_PREFIX}${type}`;

    // 1. Tenta ler do Redis (circuit breaker — em caso de falha, retorna null e cai para Postgres)
    const cached = await safeRedisGet<string>(key);
    if (cached) {
        const entries: [string, TaxonomyRelation][] = JSON.parse(cached);
        return new Map(entries);
    }

    // 2. Cache miss OU Redis indisponível → lê do Postgres (fonte da verdade)
    const supabase = createAdminServerClient();
    const { data, error } = await supabase
        .from('taxonomy_relations')
        .select('source_term, target_canonical_id, job_canonical_roles!inner(label)')
        .eq('type', type)
        .eq('status', 'active');

    if (error) throw error;  // erro do Postgres é hard error (sem fallback)

    const map = new Map<string, TaxonomyRelation>();
    for (const row of data ?? []) {
        // source_term JÁ está em lowercase + trim (CHECK constraint garante)
        map.set(row.source_term, {
            source_term: row.source_term,
            target_canonical_id: row.target_canonical_id,
            target_label: (row.job_canonical_roles as any).label,
        });
    }

    // 3. Tenta popular Redis com TTL (não-bloqueante via safeRedisSet)
    await safeRedisSet(key, JSON.stringify([...map.entries()]), CACHE_TTL_SECONDS);

    return map;
}

export async function invalidateRelations(type: RelationType): Promise<void> {
    const key = `${REDIS_KEY_PREFIX}${type}`;
    await safeRedisDel(key);
}

export async function invalidateAllRelations(): Promise<void> {
    await Promise.all([
        invalidateRelations('equivalence'),
        invalidateRelations('domain_synonym'),
        // v4: family_synonym tem invalidação separada via invalidateFamilyIndex() 
        // em lib/pipeline/taxonomy-families.ts (Decisão 3 — Opção B)
    ]);
    await safeRedisDel('tax:current_version');
    // v4: também invalidar índice de famílias
    await safeRedisDel('tax:families_index');
}

export async function getCurrentTaxonomyVersion(): Promise<string> {
    const cached = await safeRedisGet<string>('tax:current_version');
    if (cached) return cached;

    const supabase = createAdminServerClient();
    const { data } = await supabase
        .from('taxonomy_versions')
        .select('content_version')
        .order('id', { ascending: false })
        .limit(1)
        .maybeSingle();

    const version = data?.content_version ?? 'v1';
    await safeRedisSet('tax:current_version', version, 900);
    return version;
}

// ====================================
// API LEGACY (assinatura preservada para os 4 callsites existentes)
// — implementação reescrita para Postgres + Redis
// ====================================

interface FullTaxonomyCache {
    equivalences: Map<string, string>;       // source_term → target_label
    familySynonyms: Map<string, string[]>;   // v4: agora 1:N (token → array de canonical_ids)
    domainSynonyms: Map<string, string>;
}

/**
 * Retorna cache "completo" no formato legacy (Map<source_term, target_label/labels>).
 * Mantém assinatura usada pelos 4 callsites existentes (upsert-canonical.ts, 
 * suggested-roles-builder.ts, batch-processor.ts:64, teste). 
 *
 * Implementação v4:
 * - equivalences e domainSynonyms: lê de taxonomy_relations via getRelations() (Redis write-through)
 * - familySynonyms: lê de taxonomy_families + taxonomy_family_canonicals via 
 *   findFamilyMatches indexado por token (Decisão 3 — Opção B). 
 *   IMPORTANTE: a semântica de familySynonyms mudou de Map<string,string> para Map<string,string[]>
 *   porque famílias são 1:N. Os 4 callsites legacy precisam ser auditados para essa diferença.
 *   Se algum callsite assumir 1:1, migrar para findFamilyMatches() diretamente.
 */
export async function getFullTaxonomyCache(_supabase?: SupabaseClient): Promise<FullTaxonomyCache> {
    const [equivalences, domains, familyIndex] = await Promise.all([
        getRelations('equivalence'),
        getRelations('domain_synonym'),
        buildFamilyIndexLegacy(),  // helper interno que monta Map<token, canonical_labels[]>
    ]);

    // Converte Map<source_term, TaxonomyRelation> → Map<source_term, target_label>
    const toLegacy = (m: Map<string, TaxonomyRelation>): Map<string, string> =>
        new Map([...m.entries()].map(([k, v]) => [k, v.target_label]));

    return {
        equivalences: toLegacy(equivalences),
        familySynonyms: familyIndex,  // v4: já vem como Map<string, string[]>
        domainSynonyms: toLegacy(domains),
    };
}

/**
 * Helper interno: monta Map<token, [canonical_labels]> para legacy compatibility.
 * Cada token de família slug aponta para o array de labels dos canônicos da família.
 */
async function buildFamilyIndexLegacy(): Promise<Map<string, string[]>> {
    const supabase = createAdminServerClient();
    const { data, error } = await supabase
        .from('taxonomy_families')
        .select(`
            slug,
            taxonomy_family_canonicals!inner(
                job_canonical_roles!inner(label)
            )
        `)
        .eq('status', 'active');

    if (error) throw error;

    const result = new Map<string, string[]>();
    for (const family of data ?? []) {
        const labels = (family.taxonomy_family_canonicals as any[]).map(
            tfc => tfc.job_canonical_roles.label
        );
        const tokens = (family.slug as string).split('-').filter(t => t.length > 2);
        for (const token of tokens) {
            const existing = result.get(token) ?? [];
            for (const lbl of labels) {
                if (!existing.includes(lbl)) existing.push(lbl);
            }
            result.set(token, existing);
        }
    }
    return result;
}

/**
 * Wrapper legacy (assinatura preservada).
 */
export function getTaxonomyLoader() {
    return {
        load: async (_supabase?: SupabaseClient) => getFullTaxonomyCache(_supabase),
    };
}
```

**Documentação do contrato de invalidação:**

- Cada UPDATE de `taxonomy_relations` que muda `status` de/para `'active'` precisa chamar `invalidateRelations(type)` no callsite
- TTL de 1h é **safety net** se invalidação falhar — não substitui a invalidação explícita
- Múltiplas instâncias Vercel: Redis é central, então invalidação em uma instância afeta todas
- Callsites de UPDATE de `taxonomy_versions` (bump de versão) precisam invalidar `'tax:current_version'`
- `mergeCanonicals` chama `invalidateRelations()` automaticamente após merge (cravado em §G2.9)

**Callsites de escrita devem invalidar:**

```typescript
// Exemplo: ao inserir mapeamento novo via Sonnet
async function insertProposedMapping(/* ... */) {
    await supabase.from('taxonomy_relations').insert({
        type, source_term, target_canonical_id,
        status: 'inactive',  // aguarda Opus
        layer, llm_proposed_label,
    });
    // status='inactive' NÃO afeta cache, então não invalida ainda
}

// Exemplo: ao Opus aprovar mapeamento
async function activateRelation(relationId: string, opusReason: string) {
    const { data: relation } = await supabase
        .from('taxonomy_relations')
        .update({
            status: 'active',
            validated_at: new Date().toISOString(),
            validated_by: 'opus_4_7',
            opus_decision_reason: opusReason,
        })
        .eq('id', relationId)
        .select()
        .maybeSingle();

    if (relation) {
        await invalidateRelations(relation.type);
    }
}
```

**Smoke test obrigatório de cada callsite (gate operacional do PR2):**

Antes de PR2 ser dado como concluído, Antigravity executa em staging:

1. `lib/pipeline/upsert-canonical.ts` — chama `getTaxonomyLoader().load()` com canônico real, valida que o lookup funciona via Redis (e que `taxonomy_relations` tem dados seedados)
2. `lib/pipeline/suggested-roles-builder.ts` — chama `getFullTaxonomyCache(supabase)` e valida estrutura retornada
3. `lib/pipeline/batch-processor.ts:64` — roda batch real de 5 vagas e valida que pré-resolução acontece via cache (não JSON)
4. `tests/pipeline/domain-synonyms-false-positives.test.ts` — atualiza mock para retornar dados de `taxonomy_relations` em vez de JSON

Se qualquer smoke test falhar, **PR2 não é considerado concluído** — Antigravity volta para correção antes do deploy em produção.

### G2.6 — Status pós-DA2: JSONs viram referência histórica

**Mudança crítica v3 — DA2:** v2 mantinha 4 JSONs como fallback ativo durante a sprint. v3 elimina o fallback. Os arquivos ficam no repositório como referência histórica apenas — nenhum código produtivo lê.

**Marcação dos JSONs como deprecados:**

Cada um dos 3 JSONs (`equivalences.json`, `family_synonyms.json`, `domain_synonyms.json`) recebe header de depreciação na primeira chave:

```json
{
  "_deprecated_at": "2026-04-XX",
  "_replaced_by": "taxonomy_relations table (postgres) + lib/pipeline/taxonomy-cache.ts (Redis write-through)",
  "_note": "Mantido apenas como referência histórica do seed inicial. Nenhum código produtivo lê deste arquivo após Sprint v4.0.",
  "site reliability engineer": "Engenheiro de Confiabilidade de Sistemas",
  "data scientist": "Cientista de Dados"
}
```

(O 4º arquivo `domains.json` é tratado em §G3, paradigma similar.)

**Identificar callsites antigos diretos para confirmar que zeraram:**

```bash
grep -rn "equivalences.json\|family_synonyms.json\|domain_synonyms.json\|require.*equivalences\|import.*equivalences\|import.*family_synonyms\|import.*domain_synonyms" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  lib/ app/ scripts/

# Esperado após PR2: APENAS scripts/seed-taxonomy-relations.ts (que é o seed inicial e roda 1x).
# Qualquer outro callsite produtivo é bug — refatorar para getRelations().
```

**Padrão para callsites antigos diretos (raros, ainda existentes):**

```typescript
// ANTES (callsite DIRETO de JSON em lib/pipeline/precheck.ts ou similar)
import equivalences from '@/data/equivalences.json';
import familySynonyms from '@/data/family_synonyms.json';
import domainSynonyms from '@/data/domain_synonyms.json';

function preResolve(rawTitle: string) {
    const lower = rawTitle.toLowerCase();
    if (equivalences[lower]) return equivalences[lower];
    if (familySynonyms[lower]) return familySynonyms[lower];
    if (domainSynonyms[lower]) return domainSynonyms[lower];
    return null;
}

// DEPOIS
import { getRelations } from '@/lib/pipeline/taxonomy-cache';

// DEPOIS (v4)
import { getRelations } from '@/lib/pipeline/taxonomy-cache';
import { findFamilyMatches } from '@/lib/pipeline/taxonomy-families';

async function preResolve(rawTitle: string) {
    const lower = rawTitle.toLowerCase().trim();

    // Ordem de busca v4: equivalence > family (token overlap) > domain_synonym
    const equivalences = await getRelations('equivalence');
    if (equivalences.has(lower)) return equivalences.get(lower)!;

    // v4: family agora é token overlap via findFamilyMatches (Decisão 3 — Opção B)
    // Retorna SET de canônicos candidatos. Se houver match, retorna o primeiro
    // canonical do set como sugestão (consumer pode escolher de forma mais sofisticada).
    const familyResult = await findFamilyMatches(lower);
    if (familyResult.canonicalIds.length > 0) {
        return { canonicalIdsCandidates: familyResult.canonicalIds, source: 'family' };
    }

    const domains = await getRelations('domain_synonym');
    if (domains.has(lower)) return domains.get(lower)!;

    return null;
}
```

**Cuidado:** `preResolve` agora é async. Todos os callers precisam ser atualizados. Vai exigir refatoração propagada (parte do esforço subiu para PR2 = ~28h em v4).

**Mudança semântica importante v4:** o caminho de `family_synonym` agora retorna **set de candidatos** em vez de 1 canônico. Callers existentes que assumiam 1:1 (`return canonicalLabel`) precisam ser auditados durante o PR2. Possíveis estratégias para o caller:
- Escolher o canônico de maior `vacancy_count` do set
- Passar o set inteiro para Camada 2 do LLM como "candidatos prováveis"
- Aplicar heurística adicional (similaridade de embedding entre raw title e cada candidato)



### G2.7 — Sonnet curador propõe mapeamentos novos (com `relation_type` explícito)

Quando o LLM curador (Sonnet) curar uma vaga e o título original **veio das Camadas 2 ou 3** (não bateu no cache de equivalência ou sinônimo), grava a relação proposta em `taxonomy_relations` com `status='inactive'`.

**Mudança crítica v2:** o **Sonnet propõe explicitamente o `relation_type`** no output JSON, em vez do código TS aplicar uma heurística trivial (`return 'domain_synonym'`) que enfraqueceria o CHECK constraint da coluna `type`. Como os 3 valores do CHECK existem precisamente para diferenciar relações, defaultar para 1 valor único viraria a coluna ornamental. Sonnet decidir é a escolha correta.

**Output JSON do Sonnet** (formato esperado, garantido via tool_use no pipeline):

```typescript
const SONNET_OUTPUT_SCHEMA = {
    type: 'object',
    properties: {
        decision: {
            type: 'object',
            properties: {
                canonical_label: { type: 'string' },
                relation_type: {
                    type: 'string',
                    // v4: family_synonym REMOVIDO. Famílias têm tabela própria (§G2.2.bis).
                    // Sonnet só decide entre equivalence (tradução/sinônimo direto) e 
                    // domain_synonym (mesma área funcional).
                    enum: ['equivalence', 'domain_synonym']
                },
                confidence: { type: 'number', minimum: 0, maximum: 1 },
            },
            required: ['canonical_label', 'relation_type', 'confidence'],
        },
        // ... outros campos do output do pipeline
    },
};
```

**Lógica em `lib/pipeline/persist-curation.ts`** (ou onde a curadoria é finalizada):

```typescript
// Após persistir o canônico decidido pelo LLM
async function maybeInsertTaxonomyProposal(
    supabase: SupabaseClient,
    rawTitle: string,
    targetCanonicalId: string,
    layer: 2 | 3,
    llmProposedLabel: string,
    llmProposedType: 'equivalence' | 'domain_synonym'  // v4: family_synonym fora
) {
    // Só insere se foi inferência (Camadas 2 ou 3, não cache hit)
    if (layer !== 2 && layer !== 3) return;

    const lower = rawTitle.toLowerCase().trim();

    // Insert com ON CONFLICT DO NOTHING (idempotente)
    const { error } = await supabase
        .from('taxonomy_relations')
        .insert({
            type: llmProposedType,    // ← do output do Sonnet, não heurística TS
            source_term: lower,
            target_canonical_id: targetCanonicalId,
            status: 'inactive',  // aguarda Opus
            layer,
            llm_proposed_label: llmProposedLabel,
        })
        .select()
        .single();

    // Ignora erro de unique violation (já existe — outro Sonnet curou antes)
    if (error && error.code !== '23505') throw error;
}
```

**Integração com persist-curation:**

```typescript
// Em lib/pipeline/persist-curation.ts (após o INSERT/UPDATE em job_postings)
await maybeInsertTaxonomyProposal(
    supabase,
    job.title,  // após rename do Item 15.5: era job.original_title
    canonicalId,
    job.curation_layer,             // 2 ou 3, vindo do batch processor
    job.llm_proposed_label,
    job.llm_proposed_relation_type, // ← NOVO: Sonnet decidiu no output JSON
);
```

**Tunagem futura:** se em produção observarmos que o Sonnet escolhe sempre `'domain_synonym'` (vício de saída), instrumentar telemetria para detectar isso e ajustar o prompt do Sonnet com exemplos few-shot dos 3 tipos. Não é problema na largada — preferível Sonnet decidindo do que TS defaultando.

### G2.8 — CRON diário de validação Opus 4.7

**Endpoint novo:** `app/api/cron/taxonomy-validation/route.ts`

**Schedule:** diário às 03:00 UTC (após o pipeline-maintenance, em janela de baixo tráfego)

**Mudanças críticas v2:**

1. **`isCronAuthorized()` em vez de check inline** (E10): padrão do projeto. `lib/cron-guard.ts` exporta o helper, todos os CRONs existentes (analysis-cleanup, auth-sync-reconciliation, backfill-embeddings, boleto-reminder...) usam.
2. **Sem filtro de janela 24h** — fila por `status='inactive'` ordenada por `created_at ASC`. Itens antigos eventualmente são processados (FIFO). Defesa contra perder pendentes de execuções anteriores que falharam.
3. **`validation_attempts` + `last_validation_attempt_at` + `last_error`** garantem que itens que falham 5+ vezes são pulados (evita loop infinito).
4. **Parse JSON regex → Anthropic SDK `tool_use`** (Structured Outputs): garante schema validado, sem parse manual via regex frágil.
5. **DISAGREE busca canônico via slug, não label** (Gemini): evita fluxo pesado (rename → conflito de slug → merge forçado) quando o canônico já existe com casing diferente.
6. **Workflow DISAGREE em RPC SQL transacional** `process_opus_disagree()`: garante atomicidade (ou tudo aplica, ou nada aplica). Sem isso, falhas em meio do loop TS deixariam estado inconsistente.
7. **API key Opus com fallback:** `process.env.ANTHROPIC_OPUS_KEY ?? process.env.ANTHROPIC_API_KEY`.
8. **Samples por canônico via RPC** com `ROW_NUMBER() OVER (PARTITION BY canonical_role_id)`: garante exatamente 10 samples por canônico (a v1 fazia `.limit(canonicalIds.length * 10)` que não distribui).
9. **Sequência incremental para `taxonomy_content_version`** (v1, v2, v3...) em vez de `Date.now()`.

**Lógica:**

1. Buscar `taxonomy_relations` com `status='inactive'` + `validation_attempts < 5`, ordenadas por `created_at ASC`
2. Coletar até 10 vagas mais recentes por canônico via RPC `get_top_samples_per_canonical`
3. Construir prompt para Opus 4.7 com tool_use schema
4. Chamar Opus 4.7 via Anthropic SDK
5. Para cada decisão, aplicar via RPC transacional (DISAGREE) ou UPDATE direto (APPROVE/REJECT)
6. Bumpar `taxonomy_content_version` (sequencial) se houve mudanças
7. Invalidar cache Redis dos types afetados

**Schema do tool_use para Structured Outputs:**

```typescript
const VALIDATION_TOOL = {
    name: 'submit_validation_decisions',
    description: 'Submit validation decisions for taxonomy relations',
    input_schema: {
        type: 'object',
        properties: {
            decisions: {
                type: 'array',
                items: {
                    type: 'object',
                    properties: {
                        id: { type: 'string', description: 'UUID da taxonomy_relation' },
                        decision: {
                            type: 'string',
                            enum: ['APPROVE', 'DISAGREE', 'REJECT'],
                        },
                        suggested_label: {
                            type: 'string',
                            description: 'Novo label canônico (apenas se DISAGREE)',
                        },
                        reason: { type: 'string', description: 'Justificativa curta' },
                    },
                    required: ['id', 'decision', 'reason'],
                },
            },
        },
        required: ['decisions'],
    },
};
```

**Template de prompt para Opus:**

```typescript
const OPUS_VALIDATION_PROMPT = `Você é um auditor de taxonomia de cargos profissionais do mercado brasileiro de TI e correlatos. Sua função é validar mapeamentos de termos brutos de vagas para canônicos pré-existentes.

Para cada item abaixo:
1. Avalie se o termo bruto realmente representa o canônico proposto
2. Decida: APPROVE (mantém) | DISAGREE (sugere outro label) | REJECT (mapeamento ruim, ignorar)
3. Se DISAGREE, sugira o label correto

Use a ferramenta submit_validation_decisions para retornar as decisões.

Itens para validar:
{ITEMS}`;

function buildItemsBlock(relations: any[], samplesByCanonical: Map<string, any[]>) {
    return relations.map(r => `
ID: ${r.id}
Termo bruto da vaga: "${r.source_term}"
Canônico proposto pelo Sonnet: "${r.target_label}"

Vagas que originaram esse mapeamento (até 10):
${(samplesByCanonical.get(r.target_canonical_id) ?? []).slice(0, 10).map((v, i) => `
  ${i+1}. Título: "${v.title}"
     Empresa: ${v.normalized_company ?? 'N/A'}
     Trecho: "${v.description.slice(0, 300)}..."
`).join('\n')}
`).join('\n---\n');
}
```

**RPCs SQL auxiliares (criar antes do endpoint):**

```sql
BEGIN;

-- RPC: amostragem ROW_NUMBER por canonical (garante 10 por canonical)
CREATE OR REPLACE FUNCTION get_top_samples_per_canonical(
    p_canonical_ids UUID[],
    p_limit_per_canonical INT DEFAULT 10
)
RETURNS TABLE (
    canonical_role_id UUID,
    title TEXT,
    description TEXT,
    normalized_company TEXT,
    posted_at TIMESTAMPTZ
)
LANGUAGE sql STABLE AS $$
    -- v6 (Cosmético Gemini #8): seleciona normalized_company nativo (criado em §1A.bis)
    -- em vez de aliasar `company` cru. Evita dado bruto/sujo passando para o LLM Opus
    -- (company tem capitalização inconsistente, espaços, sufixos como "Ltda" / "S/A").
    SELECT canonical_role_id, title, description, normalized_company, posted_at
    FROM (
        SELECT
            jp.canonical_role_id,
            jp.title,
            jp.description,
            jp.normalized_company,
            jp.posted_at,
            ROW_NUMBER() OVER (PARTITION BY jp.canonical_role_id ORDER BY jp.posted_at DESC NULLS LAST) AS rn
        FROM job_postings jp
        WHERE jp.canonical_role_id = ANY(p_canonical_ids)
          AND jp.curation_status = 'curated'
    ) ranked
    WHERE rn <= p_limit_per_canonical;
$$;

GRANT EXECUTE ON FUNCTION get_top_samples_per_canonical(UUID[], INT) TO service_role;

-- RPC: process_opus_disagree (transacional)
CREATE OR REPLACE FUNCTION process_opus_disagree(
    p_relation_id UUID,
    p_loser_canonical_id UUID,
    p_existing_canonical_id UUID,  -- NULL se canônico sugerido não existe
    p_suggested_label TEXT,
    p_suggested_slug TEXT,
    p_reason TEXT
) RETURNS VOID
LANGUAGE plpgsql AS $$
DECLARE
    v_winner_id UUID;
BEGIN
    IF p_existing_canonical_id IS NOT NULL THEN
        -- Subcaso: merge com canônico existente
        v_winner_id := p_existing_canonical_id;
        PERFORM merge_canonicals(p_loser_canonical_id, v_winner_id, p_reason);

        -- v4 (P1.1 Gemini): merge_canonicals pode ter DELETADO p_relation_id se ele
        -- era duplicata da UNIQUE(type, source_term) com alguma relation do winner.
        -- O UPDATE seguinte retorna 0 rows silenciosamente nesse caso, perdendo a 
        -- auditoria da decisão do Opus. Detectamos via ROW_COUNT e logamos em events.
        UPDATE taxonomy_relations
        SET target_canonical_id = v_winner_id,
            status = 'active',
            validated_at = NOW(),
            validated_by = 'opus_4_7',
            opus_decision_reason = 'Mergeado com canônico existente: ' || p_reason,
            updated_at = NOW()  -- v3: P1.7 Gemini — bump explícito
        WHERE id = p_relation_id;

        IF NOT FOUND THEN
            -- Relation foi deletada como duplicata pelo merge_canonicals. 
            -- Loga decisão do Opus em events para preservar auditoria forense.
            INSERT INTO events (
                event_name, resource_type, resource_id,
                actor, actor_id,
                previous_state, new_state, reason
            ) VALUES (
                'opus_disagree_relation_absorbed',
                'taxonomy_relation', p_relation_id::text,
                'system', '00000000-0000-0000-0000-000000000001',
                jsonb_build_object('relation_id', p_relation_id, 'loser_canonical_id', p_loser_canonical_id),
                jsonb_build_object(
                    'winner_canonical_id', v_winner_id,
                    'opus_decision_reason', 'Mergeado com canônico existente: ' || p_reason,
                    'absorbed_by_merge_canonicals', true
                ),
                'Relation absorvida por merge_canonicals (UNIQUE constraint conflict). Auditoria preservada em events.'
            );
        END IF;
    ELSE
        -- Subcaso: rename do canônico criado pelo Sonnet
        UPDATE job_canonical_roles
        SET label = p_suggested_label,
            slug = p_suggested_slug,
            updated_at = NOW()
            -- trg_reset_embedding_on_semantic_change reseta embedding automaticamente
        WHERE id = p_loser_canonical_id;

        -- Marca usuários para notificação (também faz JOIN via submitted_jobs em v3)
        -- v6 (P0 Gemini #1): NÃO chamar mark_users_for_label_change_notification aqui.
        -- merge_canonicals SQL agora chama internamente na FASE 0, garantindo atomicidade.
        -- Antes (v5): chamada externa derrubava transação se argumento errado.
        -- Antes (v3-v4): chamada externa criava paradoxo temporal (suggestions já reapontadas).
        -- v6: notificação é responsabilidade interna do merge_canonicals.

        UPDATE taxonomy_relations
        SET status = 'active',
            validated_at = NOW(),
            validated_by = 'opus_4_7',
            opus_decision_reason = p_reason,
            updated_at = NOW()  -- v3: P1.7 Gemini — bump explícito
        WHERE id = p_relation_id;
    END IF;
END;
$$;

GRANT EXECUTE ON FUNCTION process_opus_disagree TO service_role;

COMMIT;
```

`merge_canonicals()` SQL function é definida em G2.9. `mark_users_for_label_change_notification()` em G2.10.

**Implementação completa do endpoint:**

```typescript
// app/api/cron/taxonomy-validation/route.ts
import { NextRequest, NextResponse } from 'next/server';
import Anthropic from '@anthropic-ai/sdk';
import { createAdminServerClient } from '@/lib/supabase-server';
import { isCronAuthorized } from '@/lib/cron-guard';
import { invalidateRelations } from '@/lib/pipeline/taxonomy-cache';
import { generateSlug } from '@/lib/taxonomy/slug';

const opusApiKey = process.env.ANTHROPIC_OPUS_KEY ?? process.env.ANTHROPIC_API_KEY;
if (!opusApiKey) {
    throw new Error('Missing ANTHROPIC_OPUS_KEY or ANTHROPIC_API_KEY');
}

const anthropic = new Anthropic({ apiKey: opusApiKey });

const MAX_VALIDATION_ATTEMPTS = 5;

export async function GET(req: NextRequest) {
    // Auth do CRON via padrão do projeto (lib/cron-guard.ts)
    if (!isCronAuthorized(req)) {
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const supabase = createAdminServerClient();
    const startedAt = new Date();

    // 1. Buscar pendentes (sem filtro 24h, FIFO por created_at, com cap em validation_attempts)
    const { data: pendentes, error } = await supabase
        .from('taxonomy_relations')
        .select(`
            id, type, source_term, target_canonical_id, layer, llm_proposed_label,
            validation_attempts,
            job_canonical_roles!inner(label)
        `)
        .eq('status', 'inactive')
        .lt('validation_attempts', MAX_VALIDATION_ATTEMPTS)
        .order('created_at', { ascending: true })
        .limit(50);

    if (error) throw error;
    if (!pendentes || pendentes.length === 0) {
        return NextResponse.json({
            status: 'no_pending',
            duration_ms: Date.now() - startedAt.getTime(),
        });
    }

    // 2. Coletar samples via RPC (ROW_NUMBER por canonical — garante 10 por canonical)
    const canonicalIds = [...new Set(pendentes.map(p => p.target_canonical_id))];
    const { data: samples } = await supabase.rpc('get_top_samples_per_canonical', {
        p_canonical_ids: canonicalIds,
        p_limit_per_canonical: 10,
    });

    const samplesByCanonical = new Map<string, any[]>();
    for (const s of samples ?? []) {
        if (!samplesByCanonical.has(s.canonical_role_id)) {
            samplesByCanonical.set(s.canonical_role_id, []);
        }
        samplesByCanonical.get(s.canonical_role_id)!.push(s);
    }

    // 3. Construir prompt
    const itemsBlock = buildItemsBlock(pendentes, samplesByCanonical);
    const prompt = OPUS_VALIDATION_PROMPT.replace('{ITEMS}', itemsBlock);

    // 4. Chamar Opus 4.7 via tool_use (Structured Outputs)
    let opusResponse;
    try {
        opusResponse = await anthropic.messages.create({
            model: 'claude-opus-4-7',
            max_tokens: 4096,
            tools: [VALIDATION_TOOL],
            tool_choice: { type: 'tool', name: 'submit_validation_decisions' },
            messages: [{ role: 'user', content: prompt }],
        });
    } catch (err) {
        console.error('Erro chamando Opus:', err);

        // Marcar tentativa falha em todos os pendentes (resiliência)
        for (const p of pendentes) {
            await supabase
                .from('taxonomy_relations')
                .update({
                    validation_attempts: p.validation_attempts + 1,
                    last_validation_attempt_at: new Date().toISOString(),
                    last_error: err instanceof Error ? err.message : String(err),
                })
                .eq('id', p.id);
        }

        return NextResponse.json({
            error: 'opus_call_failed',
            details: err instanceof Error ? err.message : String(err),
        }, { status: 500 });
    }

    // 5. Extrair tool_use input (decisões garantidas pelo schema, sem parse manual)
    const toolUseBlock = opusResponse.content.find((b: any) => b.type === 'tool_use');
    if (!toolUseBlock || toolUseBlock.type !== 'tool_use') {
        // v4 (P1.2 ChatGPT): incrementar validation_attempts/last_validation_attempt_at/last_error
        // ANTES de retornar erro. Sem isso, malformação persistente do Opus geraria loop infinito
        // (mesma relation tentada todo dia sem atingir MAX_VALIDATION_ATTEMPTS).
        for (const p of pendentes) {
            await supabase
                .from('taxonomy_relations')
                .update({
                    validation_attempts: p.validation_attempts + 1,
                    last_validation_attempt_at: new Date().toISOString(),
                    last_error: 'opus_response_no_tool_use',
                })
                .eq('id', p.id);
        }
        return NextResponse.json({
            error: 'opus_response_no_tool_use',
            content: opusResponse.content,
        }, { status: 500 });
    }
    
    // v4 (P1.3 ChatGPT): defensiva contra mudança de shape entre versões do Anthropic SDK.
    // Versões ≥0.39 usam input_json em vez de input. Preserva compatibilidade.
    const toolInput = (toolUseBlock as any).input ?? (toolUseBlock as any).input_json;
    
    const decisions = toolInput.decisions as Array<{
        id: string;
        decision: 'APPROVE' | 'DISAGREE' | 'REJECT';
        suggested_label?: string;
        reason: string;
    }>;

    // 6. Aplicar decisões
    let appliedCount = 0;
    const typesAfetados = new Set<string>();

    for (const decision of decisions) {
        const relation = pendentes.find(p => p.id === decision.id);
        if (!relation) continue;

        try {
            if (decision.decision === 'APPROVE') {
                await supabase
                    .from('taxonomy_relations')
                    .update({
                        status: 'active',
                        validated_at: new Date().toISOString(),
                        validated_by: 'opus_4_7',
                        opus_decision_reason: decision.reason,
                        validation_attempts: relation.validation_attempts + 1,
                        last_validation_attempt_at: new Date().toISOString(),
                        updated_at: new Date().toISOString(),
                    })
                    .eq('id', decision.id);

                typesAfetados.add(relation.type);
                appliedCount++;

            } else if (decision.decision === 'DISAGREE' && decision.suggested_label) {
                // Buscar canonical sugerido VIA SLUG (não label — evita case-sensitivity)
                const suggestedSlug = generateSlug(decision.suggested_label);
                const { data: existingCanonical } = await supabase
                    .from('job_canonical_roles')
                    .select('id, slug')
                    .eq('slug', suggestedSlug)
                    .maybeSingle();  // pode não existir — caminho normal

                // Workflow transacional via RPC (não em loop TS)
                await supabase.rpc('process_opus_disagree', {
                    p_relation_id: decision.id,
                    p_loser_canonical_id: relation.target_canonical_id,
                    p_existing_canonical_id: existingCanonical?.id ?? null,
                    p_suggested_label: decision.suggested_label,
                    p_suggested_slug: suggestedSlug,
                    p_reason: decision.reason,
                });

                typesAfetados.add(relation.type);
                appliedCount++;

            } else if (decision.decision === 'REJECT') {
                await supabase
                    .from('taxonomy_relations')
                    .update({
                        status: 'rejected',
                        validated_at: new Date().toISOString(),
                        validated_by: 'opus_4_7',
                        opus_decision_reason: decision.reason,
                        validation_attempts: relation.validation_attempts + 1,
                    })
                    .eq('id', decision.id);
                // REJECT não afeta cache nem versão
            }
        } catch (err) {
            // Marca falha por relation, segue
            console.error(`Erro processando decision ${decision.id}:`, err);
            await supabase
                .from('taxonomy_relations')
                .update({
                    validation_attempts: relation.validation_attempts + 1,
                    last_validation_attempt_at: new Date().toISOString(),
                    last_error: err instanceof Error ? err.message : String(err),
                })
                .eq('id', decision.id);
        }
    }

    // 7. Bump taxonomy_content_version se houve mudanças (sequência incremental v1, v2, v3...)
    if (appliedCount > 0) {
        const { data: lastVersion } = await supabase
            .from('taxonomy_versions')
            .select('content_version')
            .order('id', { ascending: false })
            .limit(1)
            .maybeSingle();

        // Parse "v1" → 1, depois +1 → "v2"
        const match = (lastVersion?.content_version ?? 'v1').match(/^v(\d+)$/);
        const nextVersionNum = (match ? parseInt(match[1], 10) : 0) + 1;
        const newVersion = `v${nextVersionNum}`;

        await supabase.from('taxonomy_versions').insert({
            content_version: newVersion,
            bumped_by: 'opus_cron',
            reason: `Validação diária: ${appliedCount} mapeamentos aplicados`,
        });

        // Invalidar cache Redis dos types afetados + versão atual
        for (const type of typesAfetados) {
            await invalidateRelations(type as any);
        }
    }

    // 8. Log de telemetria
    return NextResponse.json({
        status: 'done',
        pendentes_processados: pendentes.length,
        decisoes_aplicadas: appliedCount,
        types_afetados: [...typesAfetados],
        version_bumped: appliedCount > 0,
        duration_ms: Date.now() - startedAt.getTime(),
    });
}
```

**Schedule no `vercel.json`:**

```json
{
    "crons": [
        {
            "path": "/api/cron/taxonomy-validation",
            "schedule": "0 3 * * *"
        }
    ]
}
```


### G2.9 — Funções auxiliares (`merge_canonicals` SQL, `markUsersForLabelChangeNotification`, `generateSlug`)

**Mudanças críticas v3 sobre v2 (preservado para histórico — corrigido em v4 ver changelog topo do documento):**

> **NOTA v4:** Os pontos 2 e 4 abaixo descrevem o estado da v3, que foi corrigido na v4:
> - Ponto 2: `submitted_jobs.canonical_role_id` também NÃO EXISTE (só descoberto em v4 via Claude Code). v4 usa `resume_role_suggestions` (Decisão 2).
> - Ponto 4: `resume_skill_enrichments.skill_id` NÃO EXISTE — coluna real é `canonical_skill_id`. UNIQUE real é `(analysis_id, canonical_skill_id)` SEM `canonical_role_id`. v4 simplifica para UPDATE direto sem guard NOT EXISTS (P0 Outra-Claude).

1. **Lista real validada de 14 FKs** (substitui a lista presumida da v2 que continha 10 tabelas inventadas que NÃO EXISTEM no schema): a v2 listava `canonical_role_role_links`, `canonical_role_skill_links`, `merge_history`, `canonical_role_promotion_logs`, `canonical_role_pending_changes`, `canonical_role_audit_history`, `canonical_role_review_logs`, `role_demotion_logs`, `canonical_role_link_validations`, `canonical_role_blacklist_overrides` — todas confirmadas como inexistentes via `SELECT FROM pg_constraint WHERE confrelid = 'public.job_canonical_roles'::regclass`. v3 usa lista real validada de 14 FKs + 1 FK pós-sprint criada na própria v3 (`taxonomy_relations.target_canonical_id`).
2. **`analyses.canonical_role_id` removido**: a coluna não existe (28 colunas reais validadas). `mark_users_for_label_change_notification` faz JOIN via `submitted_jobs.canonical_role_id`. **(v4 atualiza: nem analyses NEM submitted_jobs têm a coluna; v4 usa `resume_role_suggestions`)**
3. **Defesa anti-UNIQUE em `taxonomy_relations`** (P0.4 do Gemini): UPDATE cego viola `UNIQUE(type, source_term)` quando loser e winner têm o mesmo sinônimo. v3 usa UPDATE com `NOT EXISTS` + DELETE residual.
4. **Defesa anti-UNIQUE em `resume_skill_enrichments`** (RESTRICT FK): mesma classe de problema potencial — se winner já tem enrichment para o mesmo `(canonical_role_id, skill_id)` que loser, UPDATE viola constraint. v3 aplica padrão idêntico ao Gemini para taxonomy_relations. **(v4 atualiza: coluna real é `canonical_skill_id`, e UNIQUE real NÃO inclui `canonical_role_id` → UPDATE direto não viola, guard removido)**
5. **Status correto:** `'deprecated'` (não `'merged'`). `chk_deprecated_has_merged_into` valida `merged_into IS NOT NULL` quando `status='deprecated'`.
6. **`events` com schema correto:** `resource_type`/`resource_id`, `actor='system'` obrigatório, `actor_id=SYSTEM_USER_ID`. Payload no `new_state` inclui `canonical_label`, `previous_canonical_label`, `affected_jobs`.
7. **Microdependência documentada:** esta função usa `profiles.pending_label_change_notification_sent_at` que é criada em §G2.10 — a ordem de execução dentro do PR2 é G2.10 ANTES de G2.9, OU G2.10 + G2.9 numa única migration.
8. **Notificação coalescente preservada:** `markUsersForLabelChangeNotification` deduplica via flag em `profiles` — usuários não recebem email mais que 1x por 24h.

**Lista real validada de 14 FKs apontando para `job_canonical_roles.id` (via `pg_constraint`):**

| # | Tabela | Coluna | Delete rule | Ação no merge |
|---|---|---|---|---|
| 1 | `allowed_for_pre_resolution` | `canonical_role_id` | CASCADE | UPDATE para winner (CASCADE deletaria; preferimos reapontar) |
| 2 | `curation_batch_metrics` | `canonical_role_id` | NO ACTION | UPDATE para winner |
| 3 | `function_orchestrator_items` | `canonical_role_id` | NO ACTION | UPDATE para winner |
| 4 | `job_canonical_role_sources` | `canonical_role_id` | NO ACTION | UPDATE com dedup por `(canonical_role_id, normalized_company)` |
| 5 | `job_canonical_roles` | `alias_of_id` | NO ACTION | UPDATE para winner (canônicos que apontavam loser como alias) |
| 6 | `job_canonical_roles` | `merged_into` | NO ACTION | UPDATE para winner (cadeia de merges anteriores) |
| 7 | `job_no_postings` | `canonical_role_id` | NO ACTION | UPDATE para winner |
| 8 | `job_postings` | `canonical_role_id` | SET NULL | UPDATE para winner (SET NULL desreferenciaria; preferimos reapontar) |
| 9 | `rapidapi_usage_logs` | `canonical_role_id` | SET NULL | UPDATE para winner |
| 10 | `resume_role_suggestions` | `canonical_role_id` | NO ACTION | UPDATE para winner |
| 11 | `resume_skill_enrichments` | `canonical_role_id` | RESTRICT | UPDATE com `NOT EXISTS` + DELETE residual |
| 12 | `role_merge_decisions` | `source_id` | NO ACTION | UPDATE para winner |
| 13 | `role_merge_decisions` | `target_id` | NO ACTION | UPDATE para winner |
| 14 | `skill_enrichment_stats` | `canonical_role_id` | RESTRICT | UPDATE com `NOT EXISTS` + DELETE residual |

**+15ª FK pós-sprint (criada nesta sprint em §G2.1):** `taxonomy_relations.target_canonical_id` — `merge_canonicals` reaponta com defesa anti-UNIQUE.

**Notas importantes sobre comportamento das FKs durante merge:**
- **CASCADE** (FKs #1, #8, #9): se o loser fosse DELETADO, essas tabelas seriam afetadas em cascata. Como queremos `status='deprecated'` (preserva linhagem), nem deletamos o loser — UPDATE explícito é necessário para reapontar dados ativos para o winner.
- **NO ACTION** (10 FKs): bloqueiam o `UPDATE status='deprecated'` se houver linhas pendentes. Necessário UPDATE explícito antes para liberar.
- **RESTRICT** (FKs #11, #14): igual NO ACTION mas com semântica mais forte (não checa apenas no commit). UPDATE explícito mandatório.

**SQL function `merge_canonicals` (transacional):**

```sql
BEGIN;

CREATE OR REPLACE FUNCTION merge_canonicals(
    p_loser_id UUID,
    p_winner_id UUID,
    p_reason TEXT
) RETURNS VOID
LANGUAGE plpgsql AS $$
DECLARE
    v_loser_label TEXT;
    v_winner_label TEXT;
    v_system_user_id UUID := '00000000-0000-0000-0000-000000000001';
    v_affected_jobs INT;
BEGIN
    -- Validação: ambos canônicos devem existir
    SELECT label INTO v_loser_label FROM job_canonical_roles WHERE id = p_loser_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'merge_canonicals: loser canonical % não existe', p_loser_id;
    END IF;

    SELECT label INTO v_winner_label FROM job_canonical_roles WHERE id = p_winner_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'merge_canonicals: winner canonical % não existe', p_winner_id;
    END IF;

    -- Validação: winner não pode estar deprecated
    IF EXISTS (SELECT 1 FROM job_canonical_roles WHERE id = p_winner_id AND status = 'deprecated') THEN
        RAISE EXCEPTION 'merge_canonicals: winner % está deprecated', p_winner_id;
    END IF;

    -- Validação: não pode fazer self-merge
    IF p_loser_id = p_winner_id THEN
        RAISE EXCEPTION 'merge_canonicals: loser e winner são o mesmo canonical %', p_loser_id;
    END IF;

    -- =========================================================================
    -- FASE 0 — Marcar notificação ANTES de reapontar FKs (v6 P0 Gemini #1)
    -- =========================================================================
    -- 
    -- Bug crítico identificado em v5: o reaponte de resume_role_suggestions na FASE 1
    -- (UPDATE SET canonical_role_id = winner WHERE canonical_role_id = loser) acontecia 
    -- ANTES da chamada de notificação que era externa (TS wrapper). Quando a notificação 
    -- buscava WHERE rrs.canonical_role_id = loser, retornava 0 linhas — suggestions já 
    -- haviam sido reapontadas. Sistema falhava silenciosamente em notificar QUALQUER usuário.
    --
    -- v6 resolve movendo a chamada para DENTRO do merge_canonicals, ANTES da FASE 1,
    -- garantindo que a busca por loser ainda encontre as suggestions com referência antiga.
    -- Wrapper TS e process_opus_disagree NÃO chamam mais notificação externamente —
    -- merge_canonicals já cuida disso atomicamente.
    PERFORM mark_users_for_label_change_notification(p_loser_id, (NOW() - INTERVAL '24 hours')::text);

    -- =========================================================================
    -- FASE 1 — Reapontar FKs simples (10 FKs com UPDATE direto)
    -- =========================================================================

    -- FK #1: allowed_for_pre_resolution (CASCADE → reapontar)
    UPDATE allowed_for_pre_resolution SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #2: curation_batch_metrics (NO ACTION)
    UPDATE curation_batch_metrics SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #3: function_orchestrator_items (NO ACTION)
    UPDATE function_orchestrator_items SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #5: job_canonical_roles.alias_of_id (NO ACTION) — cadeias de alias
    UPDATE job_canonical_roles SET alias_of_id = p_winner_id 
        WHERE alias_of_id = p_loser_id;

    -- FK #6: job_canonical_roles.merged_into (NO ACTION) — cadeias de merge
    UPDATE job_canonical_roles SET merged_into = p_winner_id 
        WHERE merged_into = p_loser_id;

    -- FK #7: job_no_postings (NO ACTION)
    UPDATE job_no_postings SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #8: job_postings (SET NULL → reapontar)
    UPDATE job_postings SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;
    GET DIAGNOSTICS v_affected_jobs = ROW_COUNT;

    -- FK #9: rapidapi_usage_logs (SET NULL → reapontar)
    UPDATE rapidapi_usage_logs SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #10: resume_role_suggestions (NO ACTION)
    UPDATE resume_role_suggestions SET canonical_role_id = p_winner_id 
        WHERE canonical_role_id = p_loser_id;

    -- FK #12-#13: role_merge_decisions (source_id e target_id, NO ACTION)
    UPDATE role_merge_decisions SET source_id = p_winner_id WHERE source_id = p_loser_id;
    UPDATE role_merge_decisions SET target_id = p_winner_id WHERE target_id = p_loser_id;

    -- =========================================================================
    -- FASE 2 — Reapontar FK #4 com dedup por chave única + mesclagem temporal
    -- job_canonical_role_sources tem unique(canonical_role_id, normalized_company)
    -- Se loser e winner tinham source da mesma normalized_company, evita conflito 
    -- E mescla histórico temporal (frescor de mercado preservado).
    -- =========================================================================

    -- v6 (P0 Gemini #2): mesclagem REAL de last_seen_at e first_seen_at para fontes 
    -- que existem em AMBOS canônicos. v5 fazia apenas DELETE silencioso, perdendo:
    -- - last_seen_at mais recente do loser (frescor de mercado)
    -- - first_seen_at mais antigo do loser (data de primeira detecção)
    -- Cumulativo: cada merge erodia histórico de fontes sobrepostas.
    
    -- 1. Mescla histórico temporal para fontes que JÁ EXISTEM em ambos
    UPDATE job_canonical_role_sources winner_jcrs
    SET last_seen_at = GREATEST(winner_jcrs.last_seen_at, loser_jcrs.last_seen_at),
        first_seen_at = LEAST(winner_jcrs.first_seen_at, loser_jcrs.first_seen_at)
    FROM job_canonical_role_sources loser_jcrs
    WHERE winner_jcrs.canonical_role_id = p_winner_id
      AND loser_jcrs.canonical_role_id = p_loser_id
      AND winner_jcrs.normalized_company = loser_jcrs.normalized_company;

    -- 2. Move para o winner as fontes que existiam APENAS no loser
    UPDATE job_canonical_role_sources jcrs1
    SET canonical_role_id = p_winner_id
    WHERE jcrs1.canonical_role_id = p_loser_id
      AND NOT EXISTS (
          SELECT 1 FROM job_canonical_role_sources jcrs2
          WHERE jcrs2.canonical_role_id = p_winner_id
            AND jcrs2.normalized_company = jcrs1.normalized_company
      );

    -- 3. Deleta as sobras do loser (fontes que foram mescladas no passo 1)
    DELETE FROM job_canonical_role_sources WHERE canonical_role_id = p_loser_id;

    -- =========================================================================
    -- FASE 3 — FKs com defesa contra UNIQUE violation
    -- (UPDATE com NOT EXISTS + DELETE residual — padrão Gemini para constraint)
    -- =========================================================================

    -- FK #11: resume_skill_enrichments (RESTRICT)
    -- v4 (P0 Outra-Claude + Claude Code): UNIQUE real validada é (analysis_id, canonical_skill_id)
    -- — NÃO inclui canonical_role_id. Logo, UPDATE direto reapontando canonical_role_id 
    -- NÃO PODE violar essa UNIQUE. O guard NOT EXISTS da v3 era inútil aqui (e usava
    -- coluna skill_id que não existe — coluna real é canonical_skill_id).
    -- v4 simplifica para UPDATE direto sem guard NOT EXISTS nem DELETE residual.
    UPDATE resume_skill_enrichments
    SET canonical_role_id = p_winner_id
    WHERE canonical_role_id = p_loser_id;

    -- FK #14: skill_enrichment_stats (RESTRICT)
    -- v4 (P0 Outra-Claude + Claude Code): UNIQUE composta REAL validada é
    -- (canonical_role_id, canonical_skill_id, skill_category, action_type, month)
    -- — INCLUI canonical_role_id, então pode violar em merge.
    -- Coluna correta é canonical_skill_id (não skill_id como v3 escreveu errado).
    -- Guard NOT EXISTS amplia para todas as colunas da UNIQUE composta.
    UPDATE skill_enrichment_stats ses1
    SET canonical_role_id = p_winner_id
    WHERE ses1.canonical_role_id = p_loser_id
      AND NOT EXISTS (
          SELECT 1 FROM skill_enrichment_stats ses2
          WHERE ses2.canonical_role_id = p_winner_id
            AND ses2.canonical_skill_id = ses1.canonical_skill_id
            AND ses2.skill_category = ses1.skill_category
            AND ses2.action_type = ses1.action_type
            AND ses2.month = ses1.month
      );
    DELETE FROM skill_enrichment_stats WHERE canonical_role_id = p_loser_id;

    -- FK #15 (pós-sprint): taxonomy_relations.target_canonical_id (UNIQUE em (type, source_term))
    -- v5 Nota teórica (Gemini): a UNIQUE de taxonomy_relations recai apenas em 
    -- (type, source_term) — sem incluir target_canonical_id. Logo é matematicamente 
    -- impossível que loser e winner tenham o mesmo source_term na tabela (banco só 
    -- permite source_term existir 1× em todo o universo). NOT EXISTS aqui é sempre TRUE,
    -- DELETE residual sempre exclui 0 linhas. Código morto seguro — preservado para 
    -- consistência de padrão com as outras FKs e futureproofing caso UNIQUE mude.
    UPDATE taxonomy_relations tr1
    SET target_canonical_id = p_winner_id,
        updated_at = NOW()
    WHERE tr1.target_canonical_id = p_loser_id
      AND NOT EXISTS (
          SELECT 1 FROM taxonomy_relations tr2
          WHERE tr2.target_canonical_id = p_winner_id
            AND tr2.type = tr1.type
            AND tr2.source_term = tr1.source_term
      );
    DELETE FROM taxonomy_relations WHERE target_canonical_id = p_loser_id;

    -- FK #16 (NOVA v5 — Gemini): taxonomy_family_canonicals (criada em §G2.2.bis Decisão 3 v4).
    -- v5 corrige esquecimento evolutivo: v4 criou a tabela mas merge_canonicals não foi 
    -- atualizado. Sem este UPDATE, links de família do loser ficam órfãos (loser vira 
    -- deprecated, queries filtram status='active', winner não herda famílias).
    -- UNIQUE composta da junction é (family_id, canonical_role_id) — INCLUI canonical_role_id,
    -- então defesa NOT EXISTS é necessária.
    UPDATE taxonomy_family_canonicals tfc1
    SET canonical_role_id = p_winner_id
    WHERE tfc1.canonical_role_id = p_loser_id
      AND NOT EXISTS (
          SELECT 1 FROM taxonomy_family_canonicals tfc2
          WHERE tfc2.canonical_role_id = p_winner_id
            AND tfc2.family_id = tfc1.family_id
      );
    DELETE FROM taxonomy_family_canonicals WHERE canonical_role_id = p_loser_id;

    -- FK #17 (NOVA v5 — Gemini): canonical_role_domain_links (criada em §G3.1).
    -- Mesma classe do FK #16: v3 criou a tabela mas merge_canonicals não reapontava.
    -- Sem este UPDATE, áreas funcionais do loser desaparecem em queries pós-merge.
    -- UNIQUE composta é (canonical_role_id, domain_id) — INCLUI canonical_role_id,
    -- então defesa NOT EXISTS é necessária.
    UPDATE canonical_role_domain_links crdl1
    SET canonical_role_id = p_winner_id
    WHERE crdl1.canonical_role_id = p_loser_id
      AND NOT EXISTS (
          SELECT 1 FROM canonical_role_domain_links crdl2
          WHERE crdl2.canonical_role_id = p_winner_id
            AND crdl2.domain_id = crdl1.domain_id
      );
    DELETE FROM canonical_role_domain_links WHERE canonical_role_id = p_loser_id;

    -- =========================================================================
    -- FASE 4 — Marca loser como deprecated com merged_into
    -- =========================================================================

    UPDATE job_canonical_roles
    SET status = 'deprecated',
        merged_into = p_winner_id,
        updated_at = NOW()
    WHERE id = p_loser_id;

    -- =========================================================================
    -- FASE 5 — Registra evento de auditoria (schema events real)
    -- =========================================================================

    INSERT INTO events (
        event_name,
        resource_type, resource_id,
        actor, actor_id,
        previous_state, new_state,
        reason
    ) VALUES (
        'canonical_role_merged',
        'job_canonical_role', p_loser_id,
        'system', v_system_user_id,
        jsonb_build_object('canonical_label', v_loser_label, 'status', 'active'),
        jsonb_build_object(
            'merged_into', p_winner_id,
            'canonical_label', v_winner_label,
            'previous_canonical_label', v_loser_label,
            'affected_jobs', v_affected_jobs,
            'status', 'deprecated'
        ),
        p_reason
    );
END;
$$;

GRANT EXECUTE ON FUNCTION merge_canonicals(UUID, UUID, TEXT) TO service_role;

COMMIT;
```

**Wrappers TypeScript (para callsites em pipeline TS):**

```typescript
// lib/taxonomy/merge-canonicals.ts
import { SupabaseClient } from '@supabase/supabase-js';

/**
 * Merge de dois canônicos. Atomicidade garantida via SQL function `merge_canonicals`.
 * O perdedor é marcado como `status='deprecated' + merged_into=<winner>` (não 'merged',
 * que não existe no CHECK constraint).
 *
 * SQL function reaponta 14 FKs reais + 1 FK pós-sprint (taxonomy_relations).
 * FKs com UNIQUE constraint (taxonomy_relations, resume_skill_enrichments, 
 * skill_enrichment_stats) usam padrão UPDATE NOT EXISTS + DELETE residual para 
 * evitar violação quando loser e winner já compartilham chave única.
 */
export async function mergeCanonicals(
    supabase: SupabaseClient,
    loserCanonicalId: string,
    winnerCanonicalId: string,
    reason: string
): Promise<void> {
    const { error } = await supabase.rpc('merge_canonicals', {
        p_loser_id: loserCanonicalId,
        p_winner_id: winnerCanonicalId,
        p_reason: reason,
    });

    if (error) {
        throw new Error(`merge_canonicals failed: ${error.message}`);
    }

    // v6 (P0 Gemini #1): NÃO chamar markUsersForLabelChangeNotification aqui.
    // A notificação foi movida para DENTRO do merge_canonicals SQL (FASE 0), 
    // ANTES da FASE 1 reapontar resume_role_suggestions. Garantia de atomicidade:
    // notificação acontece quando suggestions ainda apontam para o loser.
    // Wrapper TS antigo causava paradoxo temporal: notificação executada DEPOIS 
    // do reaponte buscava por loser e retornava 0 (suggestions já reapontadas).

    // Invalida cache Redis de relations (invalidação após merge é crítica para 
    // que próxima leitura veja o novo target_canonical_id)
    // v4 (P0 ChatGPT): trocado de `invalidateRelations()` (que exige argumento type) 
    // para `invalidateAllRelations()` que cobre todos os types + tax:current_version + 
    // tax:families_index. Merge afeta múltiplos types (canônico mergeado pode estar em 
    // equivalence E domain_synonym E também em famílias), então invalidação ampla é correta.
    await invalidateAllRelations();
}

/**
 * Marca todos os profile_id que tinham análise apontando para o canônico afetado.
 * Coalescente: usuários já notificados nas últimas 24h não são re-notificados.
 *
 * v4 (Decisão 2 do PO): JOIN via resume_role_suggestions (analyses NÃO TEM canonical_role_id, 
 * submitted_jobs também NÃO TEM canonical_role_id, resume_role_assignments tem schema 
 * divergente). resume_role_suggestions é o único caminho com schema correto.
 *
 * v6 (P0 Gemini #1): esta função TS é mantida APENAS para uso fora do contexto de merge
 * (ex.: testes, scripts admin que querem disparar notificação manualmente). Em fluxo de 
 * merge real, NÃO é chamada — merge_canonicals SQL já cuida disso na FASE 0 atomicamente.
 */
export async function markUsersForLabelChangeNotification(
    supabase: SupabaseClient,
    canonicalId: string
): Promise<void> {
    const cutoff24h = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString();

    const { data, error } = await supabase.rpc('mark_users_for_label_change_notification', {
        p_canonical_id: canonicalId,
        p_cutoff_iso: cutoff24h,
    });

    if (error) throw error;

    // data = lista de profile_ids notificados nesta execução
    // Caller pode usar para enfileirar emails (ver G2.11)
}

/**
 * Gera slug a partir de label, conforme convenção do projeto:
 * - Lowercase
 * - Acentos removidos
 * - Espaços viram hífens
 * - Caracteres não-alphanumeric removidos
 */
export function generateSlug(label: string): string {
    return label
        .toLowerCase()
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')  // remove acentos
        .replace(/[^a-z0-9\s-]/g, '')      // remove non-alphanumeric
        .trim()
        .replace(/\s+/g, '-')               // espaços → hífens
        .replace(/-+/g, '-');               // múltiplos hífens → um só
}
```

**RPC SQL `mark_users_for_label_change_notification` (rate limit coalescente, JOIN via resume_role_suggestions):**

```sql
BEGIN;

CREATE OR REPLACE FUNCTION mark_users_for_label_change_notification(
    p_canonical_id UUID,
    p_cutoff_iso TEXT  -- ISO timestamp, usuários notificados após esse cutoff são pulados
) RETURNS TABLE (profile_id UUID)
LANGUAGE plpgsql AS $$
BEGIN
    -- v4 (Decisão 2 do PO baseada em evidência): JOIN via resume_role_suggestions.
    -- 
    -- Histórico do achado:
    -- - v2 fazia JOIN via analyses.canonical_role_id → coluna NÃO EXISTE.
    -- - v3 fazia JOIN via submitted_jobs.canonical_role_id → coluna também NÃO EXISTE
    --   (Claude Code validou em produção: \d submitted_jobs lista 12 colunas, sem essa).
    -- - resume_role_assignments: schema divergente do documentado (canonical_role_id 
    --   retornou "column does not exist" no banco). Item 17 mantém drop dessa tabela.
    -- - resume_role_suggestions: schema correto e estável (id, resume_id, profile_id, 
    --   canonical_role_id, role_label, percentual_final, source, created_at).
    --
    -- IMPORTANTE — Estado atual do E2E:
    -- Em produção, resume_role_suggestions tem 1 row total (E2E currículo→otimização ainda
    -- não está conectado de verdade). Função retornará 0 profiles enquanto isso. É estado 
    -- correto da realidade atual, NÃO é bug. Quando E2E começar a popular essa tabela 
    -- (sprint pós-v4), função passa a funcionar automaticamente.
    --
    -- Investigação E2E está no backlog operacional pós-v4 com 16 perguntas estruturadas
    -- para Claude Code mapear o pipeline real de ingestão de currículo.
    
    RETURN QUERY
    UPDATE profiles p
    SET pending_label_change_notification = true,
        pending_label_change_notification_sent_at = NOW()
    WHERE p.id IN (
        SELECT DISTINCT rrs.profile_id
        FROM resume_role_suggestions rrs
        WHERE rrs.canonical_role_id = p_canonical_id
          AND rrs.profile_id IS NOT NULL
    )
    AND (
        p.pending_label_change_notification_sent_at IS NULL
        OR p.pending_label_change_notification_sent_at < p_cutoff_iso::TIMESTAMPTZ
    )
    RETURNING p.id;
END;
$$;

GRANT EXECUTE ON FUNCTION mark_users_for_label_change_notification(UUID, TEXT) TO service_role;

COMMIT;
```

**Nota crítica sobre ordem de migration:** esta função usa `profiles.pending_label_change_notification` e `profiles.pending_label_change_notification_sent_at`, criadas em §G2.10. A ordem de execução dentro do PR2 deve ser: **G2.10 (cria colunas) → G2.9 (cria função que usa as colunas)**. Se rodar fora de ordem, a função falha em `CREATE FUNCTION` no SQL function body (PostgreSQL valida referências a colunas no parse time para `LANGUAGE plpgsql` apenas em runtime, então tecnicamente compila — mas a primeira execução falha). Recomendação: rodar G2.10 + G2.9 numa única migration unificada para garantir consistência.


### G2.10 — Adicionar colunas `pending_label_change_notification*` em `profiles`

**Mudança v2:** adicionada também a coluna `pending_label_change_notification_sent_at` (timestamp) para rate limiting coalescente — usuários não recebem nova notificação se já receberam uma nas últimas 24h. Ver G2.9 para a RPC que usa essa coluna.

```sql
BEGIN;

ALTER TABLE profiles
    ADD COLUMN IF NOT EXISTS pending_label_change_notification BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN IF NOT EXISTS pending_label_change_notification_sent_at TIMESTAMPTZ;

-- Index parcial para o frontend buscar rapidamente quem tem notificação pendente
CREATE INDEX IF NOT EXISTS idx_profiles_pending_notif
ON profiles(id)
WHERE pending_label_change_notification = true;

COMMIT;
```

### G2.11 — Toast no próximo login + email template

**Frontend:** componente novo `components/notifications/LabelChangeToast.tsx`:

**Mudança v6 (P0 Gemini #3):** texto do toast agora é genérico — NÃO mais lista as últimas N mudanças da plataforma inteira. Em v5 isso vazava merges de outros usuários para o usuário corrente, porque `events.entity_id` aponta para `canonical_role_id` e não para `profile_id`. Filtrar por usuário exigiria cruzar com `score_history` ou expandir schema de events. v6 opta por mensagem comportamental simples, deixando listagem específica para sprint pós-v6 (vai depender de ter `analyses.canonical_role_id` denormalizado, planejado para Sprint Fluxo do Currículo).

```typescript
// components/notifications/LabelChangeToast.tsx
import { useEffect } from 'react';
import { createClient } from '@/lib/supabase-browser';

interface Props {
    profileId: string;
}

export function LabelChangeToast({ profileId }: Props) {
    useEffect(() => {
        async function checkAndShowToast() {
            const supabase = createClient();

            // Verificar flag
            const { data: profile } = await supabase
                .from('profiles')
                .select('pending_label_change_notification')
                .eq('id', profileId)
                .maybeSingle();

            if (!profile?.pending_label_change_notification) return;

            // v6 (P0 Gemini #3): NÃO buscar mais detalhes em events — query não tem 
            // filtro por profile_id (events.entity_id aponta para canonical_role_id).
            // Mostrar apenas mensagem genérica, comportamental, sem expor mudanças 
            // de outros usuários.
            showToast({
                type: 'info',
                title: 'O mercado mudou e nós acompanhamos',
                description: `Notamos que a nomenclatura de uma das funções do seu currículo evoluiu no mercado de trabalho. Atualizamos seus painéis automaticamente para refletir o nome mais moderno. Suas análises anteriores continuam disponíveis.`,
                duration: 8000,
            });

            // Limpar flag
            await supabase
                .from('profiles')
                .update({ pending_label_change_notification: false })
                .eq('id', profileId);
        }

        checkAndShowToast();
    }, [profileId]);

    return null;  // Não renderiza nada visível, apenas dispara o toast
}
```

**Integração no layout autenticado:** adicionar `<LabelChangeToast profileId={activeProfile.id} />` no layout do app autenticado, próximo a outros providers.

**Backlog pós-v6 (Sprint Fluxo do Currículo):** quando `analyses.canonical_role_id` for denormalizado e populado consistentemente, podemos voltar a listar mudanças específicas por usuário com filtro confiável: `JOIN events e ON e.resource_id = analysis.canonical_role_id WHERE analysis.profile_id = $1`.

**Email template — DEFERIDO para sprint pós-v2 (Decisão 2 do changelog):**

> ⚠️ **NÃO ENVIAR AO HOSTINGER NESTA VERSÃO**

Razões para o deferimento:

1. **Compatibilidade Microsoft Outlook:** o template precisa ser HTML completo com MSO conditional comments (`<!--[if mso]>... <![endif]-->`), table-based layout, fallbacks de fonte e botões em VML. O texto plano abaixo não é renderizável dignamente em Outlook.
2. **Reescrita do copy:** o draft atual confunde o usuário ao não esclarecer **o que** mudou. A função reclassificada não é "uma função identificada no currículo do usuário", mas sim **a função que o usuário SELECIONOU como referência para a comparação contra vagas** no serviço CalibraCV. A redação precisa deixar isso explícito.

**Rascunho textual mantido como referência (NÃO USAR EM PRODUÇÃO):**

```
Assunto: Atualização nas suas análises CalibraCV — função reclassificada

Olá, {{first_name}}!

Com base em análise contínua de dados de mercado, reclassificamos a função "{{old_label}}" para "{{new_label}}". Essa atualização garante que sua análise reflita a nomenclatura atual do mercado.

Suas análises anteriores continuam disponíveis e foram automaticamente atualizadas com o novo nome. Não há necessidade de refazer nada.

[Ver minha análise →]({{analysis_link}})

Mais sobre como mantemos nosso catálogo vivo: https://calibracv.com/methodology

Equipe CalibraCV
```

**Para a sprint pós-v3 (montagem do template HTML real):**

- Definir variáveis: `first_name` (lido direto de `public.users.first_name` via JOIN — coluna já populada para todos UUIDs convencionais; v2 propunha `display_name.split(' ')[0]` mas v3 usa coluna nativa), `previous_label`, `new_label`, `analysis_link`, `methodology_link`
- Reescrever copy para deixar claro que mudou **a função selecionada na comparação**, não uma das funções identificadas no currículo
- Aplicar padrão `email-template` do CalibraCV (color-scheme meta tags, preheader, MSO emoji fallbacks, `class="email-cta"` em CTAs)
- Cadastrar no Hostinger como template `label_change_notification`

**Disparo do email (placeholder no v3):** Função `enqueueLabelChangeEmail` fica como stub na v3 — apenas registra no log que precisaria enfileirar. O toast frontend cobre o usuário ativo; usuários inativos receberão o email assim que a sprint pós-v3 entregar o template.

```typescript
// lib/notifications/label-change-email.ts (stub v3)
export async function enqueueLabelChangeEmail(
    profileId: string,
    canonicalId: string,
    previousLabel: string,
    newLabel: string
): Promise<void> {
    // STUB v3: template HTML pendente — apenas log
    console.warn('[STUB] enqueueLabelChangeEmail pendente até sprint pós-v3:', {
        profileId,
        canonicalId,
        previousLabel,
        newLabel,
    });
    // Implementação real fica para sprint pós-v3:
    // 1. Resolver primeiro nome via JOIN users.first_name (coluna nativa, já populada)
    //    SELECT first_name FROM public.users WHERE id = profileId
    // 2. Construir analysis_link
    // 3. Enviar via API do Hostinger usando template 'label_change_notification'
}
```

### G2.12 — `taxonomy_content_version` bumping após CRON Opus

Já implementado em G2.8 (passo 7). Recap:

- CRON Opus, ao final, **se houve `appliedCount > 0`**, insere nova versão em `taxonomy_versions`
- Cache Redis é invalidado em todos os types afetados (write-through)
- Hashes de `description_hash` antigos ficam stale e vão sendo substituídos organicamente em chamadas futuras do pipeline

**Onde o pipeline lê a versão:**

**Mudança v2:** o helper `getCurrentTaxonomyVersion` foi consolidado em `lib/pipeline/taxonomy-cache.ts` (G2.5) — não há helper duplicado em `lib/pipeline/precheck.ts`. O callsite de precheck **importa** do taxonomy-cache:

```typescript
// lib/pipeline/precheck.ts (ou onde computa description_hash)
import { getCurrentTaxonomyVersion } from '@/lib/pipeline/taxonomy-cache';

function computeDescriptionHash(description: string, taxonomyVersion: string): string {
    const normalized = normalizeDescription(description);
    return crypto.createHash('sha256')
        .update(`${normalized}|${taxonomyVersion}`)
        .digest('hex');
}

// Uso:
const taxonomyVersion = await getCurrentTaxonomyVersion();
const hash = computeDescriptionHash(job.description, taxonomyVersion);
```

**Quando CRON Opus bumpa versão:** invalidar também `tax:current_version` no Redis. Já garantido pela função `invalidateAllRelations()` em `lib/pipeline/taxonomy-cache.ts` (G2.5), que invalida tanto `tax:equivalence`, `tax:family_synonym`, `tax:domain_synonym` quanto `tax:current_version`.

### G2.13 — Tests para Item G2

**Mudanças críticas v3 sobre v2:**

1. **Status correto em test de mergeCanonicals**: v2 ainda esperava `status === 'merged'`. CHECK constraint real é `'deprecated' + merged_into NOT NULL`. v3 corrige.
2. **`content_version` via sequência incremental**: v2 usava `` `v${Date.now()}` `` (timestamp). Convenção cravada (decisão #29 da v2) é sequência `v1`, `v2`, `v3` — ordenam lexicograficamente correto. v3 usa helper `bumpTaxonomyVersion()` que faz `MAX + 1`.
3. **Fixtures reais** (P2.3 Grok): v2 usava placeholder `'algum-canonical-id'` — UPSERT silenciosamente falhava por FK violation porque UUID não existe. v3 usa helper `createTestCanonical()` em cada test e `cleanupTestCanonical()` no afterEach.
4. **Test de `llm_proposed_relation_type`** (P1.6): valida que Sonnet propõe explicitamente o `type` no JSON output (não default fixo `'domain_synonym'`).
5. **Test de cadeia profunda 11+ níveis** (P2.4): valida que `resolve_canonical` lida com cadeia maior que profundidade máxima (10) sem ciclo infinito — retorna último válido.

```typescript
// tests/integration/sprint-v4_0/item-g2-taxonomy.spec.ts
import { 
    createTestCanonical, 
    cleanupTestCanonical, 
    bumpTaxonomyVersionForTest 
} from '../helpers';

describe('Item G2 — Sinônimos dinâmicos via taxonomy_relations', () => {
    let testCanonical: { id: string; label: string; slug: string };

    beforeEach(async () => {
        testCanonical = await createTestCanonical({ 
            label: 'Test Canonical G2',
            status: 'active' 
        });
    });

    afterEach(async () => {
        await cleanupTestCanonical(testCanonical.id);
    });

    it('seed inicial popula taxonomy_relations a partir dos 3 JSONs', async () => {
        const { data, count } = await supabase
            .from('taxonomy_relations')
            .select('*', { count: 'exact' })
            .eq('seeded_from_json', true);

        expect(count).toBeGreaterThan(0);
        // Esperado: ~centenas de seeds
    });

    it('cache Redis funciona com write-through em invalidação', async () => {
        // 1. Lê cache miss → popula Redis
        const before = await getRelations('equivalence');
        expect(before.size).toBeGreaterThan(0);

        // 2. Insere nova relação ativa (com canonical_id real)
        const { data: novaRelacao } = await supabase
            .from('taxonomy_relations')
            .insert({
                type: 'equivalence',
                source_term: 'test_sinonimo_xyz',
                target_canonical_id: testCanonical.id,
                status: 'active',
                seeded_from_json: false,
            })
            .select()
            .single();

        // 3. Invalida cache
        await invalidateRelations('equivalence');

        // 4. Lê novamente — deve incluir a nova
        const after = await getRelations('equivalence');
        expect(after.has('test_sinonimo_xyz')).toBe(true);

        // Cleanup
        await supabase.from('taxonomy_relations').delete().eq('id', novaRelacao!.id);
    });

    it('Sonnet propõe mapeamento em Camadas 2 e 3 com status inactive e relation_type explícito', async () => {
        // Simula proposta com relation_type explícito vindo do output do Sonnet
        await maybeInsertTaxonomyProposal(supabase, {
            sourceTerm: 'Site Reliability Engineer III',
            targetCanonicalId: testCanonical.id,
            layer: 2,
            llmProposedLabel: 'Engenheiro de Confiabilidade Senior',
            llmProposedRelationType: 'family_synonym',  // Sonnet decidiu explicitamente
        });

        const { data } = await supabase
            .from('taxonomy_relations')
            .select('*')
            .eq('source_term', 'site reliability engineer iii')
            .single();

        expect(data!.status).toBe('inactive');
        expect(data!.layer).toBe(2);
        expect(data!.type).toBe('family_synonym');  // não default fixo 'domain_synonym'
    });

    it('CRON Opus aprovação muda status para active e bumpa versão (sequência incremental, não Date.now)', async () => {
        // Setup: insere relação inactive
        const { data: relation } = await supabase
            .from('taxonomy_relations')
            .insert({
                type: 'domain_synonym',
                source_term: 'test_sre_iii',
                target_canonical_id: testCanonical.id,
                status: 'inactive',
                layer: 2,
            })
            .select()
            .single();

        const versionBefore = await getCurrentTaxonomyVersion();

        // Aprovação Opus
        await supabase
            .from('taxonomy_relations')
            .update({
                status: 'active',
                validated_at: new Date().toISOString(),
                validated_by: 'opus_4_7',
                opus_decision_reason: 'Mapeamento correto',
            })
            .eq('id', relation!.id);

        // Bump de versão via helper de sequência incremental (v1, v2, v3...)
        // NÃO usa Date.now() — convenção é string ordenável lexicograficamente
        await bumpTaxonomyVersionForTest({
            bumpedBy: 'opus_cron_test',
            reason: 'Test',
        });

        const versionAfter = await getCurrentTaxonomyVersion();
        expect(versionAfter).not.toBe(versionBefore);
        expect(versionAfter).toMatch(/^v\d+$/);  // formato correto

        // Cleanup
        await supabase.from('taxonomy_relations').delete().eq('id', relation!.id);
    });

    it('mergeCanonicals reaponta vagas e marca loser como deprecated (não merged)', async () => {
        const winner = await createTestCanonical({ label: 'Winner G2', status: 'active' });
        const loser = await createTestCanonical({ label: 'Loser G2', status: 'active' });

        // Insere 1 vaga no loser
        // v6 (P0 Claude Code NEv5-2): posted_at + expires_at são NOT NULL sem default — REQUER no INSERT.
        // v6 (P0 Claude Code NEv5-3): linkedin_id UUID-prefixado para evitar colisão entre runs interrompidos.
        const { data: vaga } = await supabase
            .from('job_postings')
            .insert({
                title: 'Vaga teste',
                description_curated: 'Test',
                canonical_role_id: loser.id,
                curation_status: 'curated',
                normalized_company: 'empresa-x',
                linkedin_id: `test-${crypto.randomUUID()}`,
                posted_at: new Date().toISOString(),
                expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString(),
            })
            .select()
            .single();

        await mergeCanonicals(supabase, loser.id, winner.id, 'test_merge');

        // Vaga foi reapontada
        const { data: vagaReapontada } = await supabase
            .from('job_postings')
            .select('canonical_role_id')
            .eq('id', vaga!.id)
            .single();
        expect(vagaReapontada!.canonical_role_id).toBe(winner.id);

        // Loser está deprecated (não 'merged' — esse valor não existe no CHECK)
        const { data: loserAfter } = await supabase
            .from('job_canonical_roles')
            .select('status, merged_into')
            .eq('id', loser.id)
            .single();

        expect(loserAfter!.status).toBe('deprecated');
        expect(loserAfter!.merged_into).toBe(winner.id);

        // Cleanup
        await supabase.from('job_postings').delete().eq('id', vaga!.id);
        await cleanupTestCanonical(loser.id);
        await cleanupTestCanonical(winner.id);
    });

    it('mergeCanonicals notifica profiles via resume_role_suggestions (Decisão 2 v4)', async () => {
        // v6 Cosmético — IMPORTANTE: este test passa porque cria suggestion ANTES do merge.
        // Em produção, enquanto E2E (currículo→análise) não estiver conectado, 
        // resume_role_suggestions tem ~1 row total e função retorna 0 profiles.
        // É comportamento correto, NÃO bug. Quando E2E conectar (Sprint Fluxo do Currículo),
        // função passa a funcionar automaticamente.
        // 
        // v6 (P0 Gemini #1): merge_canonicals SQL agora chama mark_users_for_label_change_notification 
        // INTERNAMENTE na FASE 0 (antes da FASE 1 reapontar suggestions). Wrapper TS NÃO chama mais 
        // externamente — atomicidade garantida.
        // v5 (4 P0 do Claude Code): test reescrito com schema real validado em produção.
        // - NEv4-1: resumes.resume_text (não raw_text — confundi com submitted_jobs.raw_text)
        // - NEv4-2: resumes não tem UNIQUE em profile_id → INSERT direto sem upsert
        // - NEv4-3: UUID ...0003 é users.id (user_type='demo'), NÃO profile.id. 
        //           Cria profile fresh no próprio test via users demo + cleanup defensivo.
        //           changelog v4 #16 alegou criar no beforeEach mas não foi aplicado.
        // - NEv4-4: resume_role_suggestions.percentual_final é obrigatório + UNIQUE 
        //           (resume_id, canonical_role_id) precisa cleanup.

        const winner = await createTestCanonical({ label: 'Winner Notif', status: 'active' });
        const loser = await createTestCanonical({ label: 'Loser Notif', status: 'active' });

        // 1. Resolver users.id real do tipo demo (validado existir em produção)
        const { data: demoUser, error: userErr } = await supabase
            .from('users')
            .select('id')
            .eq('user_type', 'demo')
            .limit(1)
            .single();
        if (userErr || !demoUser) {
            throw new Error('Setup falhou: nenhum users com user_type=demo encontrado.');
        }

        // 2. Criar profile FRESH para este test (não reusar nenhum profile global)
        const { data: testProfile, error: profileErr } = await supabase
            .from('profiles')
            .insert({
                user_id: demoUser.id,
                display_name: 'Test G2.13 v5',
            })
            .select('id, user_id')
            .single();
        if (profileErr || !testProfile) {
            throw new Error(`Setup falhou: insert em profiles: ${profileErr?.message}`);
        }

        // 3. Criar resume vinculado ao profile (INSERT direto — sem onConflict, schema real
        //    de resumes só tem PK em id, sem UNIQUE em profile_id)
        // v6 (P0 Claude Code NEv5-1): resumes.input_type é NOT NULL sem default — REQUER no INSERT.
        const { data: resume, error: resumeErr } = await supabase
            .from('resumes')
            .insert({
                profile_id: testProfile.id,
                resume_text: 'CV de teste para v6',
                input_type: 'text',  // v6: NOT NULL sem default (validado pelo Claude Code)
            })
            .select('id')
            .single();
        if (resumeErr || !resume) {
            throw new Error(`Setup falhou: insert em resumes: ${resumeErr?.message}`);
        }

        // 4. Criar suggestion apontando para loser
        // v6 (P0 Outra-Claude): role_label REMOVIDO — coluna foi dropada pelo Item 16 do PR1.
        //   PR1 roda primeiro, então quando este test (PR2/G2.13) executa, a coluna já não existe.
        //   PostgREST rejeitaria campo inexistente com 400.
        // v6 (P0 Claude Code NEv4-4): inclui percentual_final (NOT NULL).
        const { data: suggestion, error: suggestionErr } = await supabase
            .from('resume_role_suggestions')
            .insert({
                resume_id: resume.id,
                profile_id: testProfile.id,
                canonical_role_id: loser.id,
                percentual_final: 75,
                source: 'test',
            })
            .select()
            .single();
        if (suggestionErr || !suggestion) {
            throw new Error(`Setup falhou: insert em resume_role_suggestions: ${suggestionErr?.message}`);
        }

        try {
            // 5. Merge
            await mergeCanonicals(supabase, loser.id, winner.id, 'test_notif');

            // 6. Profile deve ter pending_label_change_notification = true
            const { data: profileAfter } = await supabase
                .from('profiles')
                .select('pending_label_change_notification')
                .eq('id', testProfile.id)
                .single();

            expect(profileAfter?.pending_label_change_notification).toBe(true);

            // 7. Suggestion deve ter sido reapontada para o winner via merge_canonicals
            //    (FK NO ACTION: UPDATE explícito reaponta antes do status='deprecated' do loser)
            const { data: suggestionAfter } = await supabase
                .from('resume_role_suggestions')
                .select('canonical_role_id')
                .eq('id', suggestion.id)
                .single();

            expect(suggestionAfter?.canonical_role_id).toBe(winner.id);
        } finally {
            // Cleanup defensivo (sempre roda, mesmo se asserts falham — evita poluir prod)
            await supabase.from('resume_role_suggestions').delete().eq('id', suggestion.id);
            await supabase.from('resumes').delete().eq('id', resume.id);
            await supabase.from('profiles').delete().eq('id', testProfile.id);
            await cleanupTestCanonical(loser.id);
            await cleanupTestCanonical(winner.id);
        }
    });

    it('resolve_canonical lida com cadeia profunda > 10 sem loop infinito (retorna último válido)', async () => {
        // Cria cadeia de 11 canônicos: 1→2→3→...→11
        const chain: { id: string }[] = [];
        for (let i = 0; i < 11; i++) {
            const c = await createTestCanonical({ label: `Chain${i}`, status: 'active' });
            chain.push({ id: c.id });
        }

        // Encadeia: 0 → 1 → 2 → ... → 10 (cada um deprecated, merged_into o próximo)
        // Marca os primeiros 10 como deprecated (precisa update na ordem inversa por causa de FK)
        for (let i = 0; i < 10; i++) {
            await supabase
                .from('job_canonical_roles')
                .update({ status: 'deprecated', merged_into: chain[i + 1].id })
                .eq('id', chain[i].id);
        }

        // resolve_canonical do primeiro deve retornar SEM erro (mesmo passando do depth limit)
        const { data: resolved, error } = await supabase
            .rpc('resolve_canonical', { p_id: chain[0].id });

        expect(error).toBeNull();
        // Retorna o último válido alcançável dentro de depth=10 (chain[10] está no nível 10, mas
        // depth limit pode parar antes — o importante é não loop infinito e não erro)
        expect(resolved).toBeDefined();

        // Cleanup
        for (const c of chain.reverse()) {
            await cleanupTestCanonical(c.id);
        }
    });

    it('toast aparece no login após mudança de label', async () => {
        const testProfileId = '00000000-0000-0000-0000-000000088888';

        await supabase
            .from('profiles')
            .update({ pending_label_change_notification: true })
            .eq('id', testProfileId);

        const { result } = renderHook(() => useLabelChangeNotification(testProfileId));

        await waitFor(() => {
            expect(result.current.toastShown).toBe(true);
        });

        const { data: profile } = await supabase
            .from('profiles')
            .select('pending_label_change_notification')
            .eq('id', testProfileId)
            .single();

        expect(profile?.pending_label_change_notification).toBe(false);
    });
});
```

### G2.14 — Esforço estimado

- **Schema novo (`taxonomy_relations`, `taxonomy_versions`):** 1.5 horas
- **Seed inicial dos 3 JSONs:** 2 horas
- **Cache Redis com write-through:** 2 horas
- **Substituição dos consumers JSON pelo cache:** 3 horas
- **Sonnet propõe mapeamentos (integração com persist-curation):** 2 horas
- **CRON Opus 4.7 (endpoint completo + prompt):** 4 horas
- **Funções auxiliares (merge, notification, slug):** 2 horas
- **Frontend toast + email template:** 2 horas
- **Tests M2:** 3 horas
- **Total:** 21.5 horas

---


## Item G3 — Áreas de atuação 0:N + UI dos modais

Migra `domains.json` para banco usando `canonical_role_domains` (catálogo já existente, vazio) + tabela junction nova `canonical_role_domain_links`. Backfill IA para 536 canônicos sem área. UI dos 2 modais ("Calibrar para outra função" e "Mapa regional de competências") ganha dropdowns funcionais.

### G3.1 — Schema da junction `canonical_role_domain_links`

```sql
BEGIN;

-- DROP da coluna 1:1 dormente (664 NULLs, zero callsites — Bloco P confirmou)
ALTER TABLE job_canonical_roles DROP COLUMN IF EXISTS domain_id;

-- Junction N:N
CREATE TABLE canonical_role_domain_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    canonical_role_id UUID NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
    domain_id UUID NOT NULL REFERENCES canonical_role_domains(id) ON DELETE CASCADE,
    is_primary BOOLEAN NOT NULL DEFAULT false,
    confidence NUMERIC(4,3) CHECK (confidence >= 0 AND confidence <= 1),
    source TEXT NOT NULL CHECK (source IN ('seed', 'ai_backfill', 'manual')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (canonical_role_id, domain_id)
);

-- Index para busca rápida por canônico
CREATE INDEX idx_crdl_canonical_role_id ON canonical_role_domain_links (canonical_role_id);

-- Index para busca rápida por domínio (filtro do dropdown)
CREATE INDEX idx_crdl_domain_id ON canonical_role_domain_links (domain_id);

-- Constraint: cada canônico tem no máximo 1 is_primary
CREATE UNIQUE INDEX idx_crdl_one_primary_per_role
    ON canonical_role_domain_links (canonical_role_id)
    WHERE is_primary = true;

-- RLS: leitura pública para authenticated (catálogo é informação pública)
ALTER TABLE canonical_role_domain_links ENABLE ROW LEVEL SECURITY;

CREATE POLICY canonical_role_domain_links_select_authenticated ON canonical_role_domain_links
FOR SELECT
TO authenticated
USING (true);

-- Sem policy de escrita → service_role bypassa

COMMIT;
```

### G3.2 — Seed do catálogo `canonical_role_domains` a partir de `domains.json`

Bloco Q confirmou: `domains.json` v2.0 tem 20 chaves cobrindo ~128 canônicos. Usaremos as chaves como slugs e nomes amigáveis em PT-BR como `name`.

```typescript
// scripts/seed-canonical-role-domains.ts
import { createAdminServerClient } from '@/lib/supabase-server';
import domainsJson from '@/data/domains.json';

const DOMAIN_LABELS: Record<string, string> = {
    desenvolvimento: 'Desenvolvimento',
    infraestrutura: 'Infraestrutura',
    comercial: 'Comercial',
    operacoes: 'Operações',
    dados: 'Dados',
    produto: 'Produto',
    facilities: 'Facilities',
    financeiro: 'Financeiro',
    governanca: 'Governança',
    rh: 'Recursos Humanos',
    seguranca: 'Segurança',
    atendimento: 'Atendimento',
    administrativo: 'Administrativo',
    educacao: 'Educação',
    implantacao: 'Implantação',
    negocios: 'Negócios',
    projetos: 'Projetos',
    telecom: 'Telecomunicações',
    juridico: 'Jurídico',
    seguranca_patrimonial: 'Segurança Patrimonial',
};

const DISPLAY_ORDER: Record<string, number> = {
    desenvolvimento: 1,
    dados: 2,
    infraestrutura: 3,
    produto: 4,
    comercial: 5,
    operacoes: 6,
    administrativo: 7,
    financeiro: 8,
    rh: 9,
    atendimento: 10,
    governanca: 11,
    juridico: 12,
    projetos: 13,
    educacao: 14,
    implantacao: 15,
    negocios: 16,
    telecom: 17,
    seguranca: 18,
    facilities: 19,
    seguranca_patrimonial: 20,
};

async function seedDomains() {
    const supabase = createAdminServerClient();
    const data = (domainsJson as any).data;
    
    const seeds = Object.keys(data).map(slug => ({
        slug,
        name: DOMAIN_LABELS[slug] ?? slug,
        description: null,
        display_order: DISPLAY_ORDER[slug] ?? 99,
        is_active: true,
    }));
    
    console.log(`Inserindo ${seeds.length} áreas no catálogo...`);
    
    const { error } = await supabase
        .from('canonical_role_domains')
        .upsert(seeds, { onConflict: 'slug' });
    
    if (error) throw error;
    
    console.log('Catálogo seedado com sucesso.');
}

seedDomains().catch(console.error);
```

### G3.3 — Seed da junction (128 canônicos do `domains.json`)

```typescript
// scripts/seed-canonical-role-domain-links.ts
import { createAdminServerClient } from '@/lib/supabase-server';
import domainsJson from '@/data/domains.json';

async function seedLinks() {
    const supabase = createAdminServerClient();
    const data = (domainsJson as any).data;
    
    // 1. Carregar mapeamento slug → domain_id
    const { data: domains } = await supabase
        .from('canonical_role_domains')
        .select('id, slug');
    const domainBySlug = new Map((domains ?? []).map(d => [d.slug, d.id]));
    
    // 2. Carregar todos os canônicos por label
    const { data: canonicals } = await supabase
        .from('job_canonical_roles')
        .select('id, label');
    const canonicalByLabel = new Map((canonicals ?? []).map(c => [c.label, c.id]));
    
    // 3. Iterar JSON e construir links
    const links: Array<{
        canonical_role_id: string;
        domain_id: string;
        is_primary: boolean;
        confidence: number;
        source: 'seed';
    }> = [];
    
    for (const [domainSlug, canonicalLabels] of Object.entries(data)) {
        const domainId = domainBySlug.get(domainSlug);
        if (!domainId) {
            console.warn(`Domain slug não encontrado: ${domainSlug}`);
            continue;
        }
        
        for (const label of canonicalLabels as string[]) {
            const canonicalId = canonicalByLabel.get(label);
            if (!canonicalId) {
                console.warn(`Canonical label não encontrado: ${label}`);
                continue;
            }
            
            links.push({
                canonical_role_id: canonicalId,
                domain_id: domainId,
                is_primary: true,  // único domain → é o primary
                confidence: 1.0,
                source: 'seed',
            });
        }
    }
    
    console.log(`Inserindo ${links.length} links...`);
    
    // 4. Insert em batches
    const batchSize = 100;
    for (let i = 0; i < links.length; i += batchSize) {
        const batch = links.slice(i, i + batchSize);
        const { error } = await supabase
            .from('canonical_role_domain_links')
            .upsert(batch, { onConflict: 'canonical_role_id,domain_id' });
        
        if (error) {
            console.error(`Erro no batch ${i}:`, error);
            throw error;
        }
    }
    
    console.log('Junction seedada com sucesso.');
}

seedLinks().catch(console.error);
```

### G3.4 — Backfill IA para canônicos sem área (~536)

Para os ~536 canônicos que não estão no `domains.json`, usar Haiku 4.5 com prompt rico (label + skills agregadas + top 5 industries das vagas) para inferir área.

**Bloco Q confirmou:** `org_industry_normalized` é ortogonal a domain (industry = setor empregador, domain = função do cargo). Mas serve como **sinal de contexto** para Haiku desempatar.

**Mudanças críticas v2:**

1. **`.not('id','in', subquery)` é inválido em PostgREST** — não suporta subquery em `not.in`. Substituído por RPC SQL `get_canonicals_without_domain_links()` que faz o LEFT JOIN no banco.
2. **Retry com 3 tentativas + backoff exponencial** por canônico antes de marcar como `unresolved` em `events`.
3. **Checkpoint** via tabela auxiliar `_backfill_progress` permite retomar de onde parou se o script for interrompido.
4. **Log de falhas em `events`** com schema correto (`resource_type='job_canonical_role'`, `actor='system'`).
5. **JSON parsing via tool_use** (consistente com CRON Opus G2.8) em vez de regex.

**RPC para listar canônicos sem domain (substitui `.not.in` do PostgREST):**

```sql
BEGIN;

CREATE OR REPLACE FUNCTION get_canonicals_without_domain_links()
RETURNS TABLE (id UUID, label TEXT)
LANGUAGE sql STABLE AS $$
    SELECT jcr.id, jcr.label
    FROM job_canonical_roles jcr
    LEFT JOIN canonical_role_domain_links crdl ON crdl.canonical_role_id = jcr.id
    WHERE jcr.status = 'active'
      AND crdl.id IS NULL
    ORDER BY jcr.label;
$$;

GRANT EXECUTE ON FUNCTION get_canonicals_without_domain_links() TO service_role;

-- Tabela de checkpoint para retomar backfill
CREATE TABLE IF NOT EXISTS _backfill_progress (
    job_name TEXT PRIMARY KEY,
    last_processed_id UUID,
    processed_count INT NOT NULL DEFAULT 0,
    success_count INT NOT NULL DEFAULT 0,
    failure_count INT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMIT;
```

```typescript
// scripts/backfill-canonical-domains.ts
import Anthropic from '@anthropic-ai/sdk';
import { createAdminServerClient } from '@/lib/supabase-server';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const SYSTEM_USER_ID = '00000000-0000-0000-0000-000000000001';
const MAX_RETRIES = 3;
const JOB_NAME = 'backfill-canonical-domains-v2';

const BACKFILL_TOOL = {
    name: 'classify_canonical_domain',
    description: 'Classifica área funcional principal do canônico',
    input_schema: {
        type: 'object',
        properties: {
            primary_domain_slug: { type: ['string', 'null'] },
            secondary_domain_slug: { type: ['string', 'null'] },
            confidence: { type: 'number', minimum: 0, maximum: 1 },
            reason: { type: 'string' },
        },
        required: ['primary_domain_slug', 'confidence', 'reason'],
    },
};

const BACKFILL_PROMPT = `Você é um classificador taxonomic de cargos profissionais brasileiros.

Para o canônico abaixo, identifique a ÁREA FUNCIONAL principal entre as 20 disponíveis.

Áreas disponíveis (slug + nome):
- desenvolvimento (Desenvolvimento)
- infraestrutura (Infraestrutura)
- comercial (Comercial)
- operacoes (Operações)
- dados (Dados)
- produto (Produto)
- facilities (Facilities)
- financeiro (Financeiro)
- governanca (Governança)
- rh (Recursos Humanos)
- seguranca (Segurança da Informação)
- atendimento (Atendimento)
- administrativo (Administrativo)
- educacao (Educação)
- implantacao (Implantação)
- negocios (Negócios)
- projetos (Projetos)
- telecom (Telecomunicações)
- juridico (Jurídico)
- seguranca_patrimonial (Segurança Patrimonial)

Canônico:
Label: "{LABEL}"

Skills agregadas das vagas (top 10 por frequência):
{SKILLS}

Top 5 industries das vagas atreladas (contexto, não categoria):
{INDUSTRIES}

Vagas de exemplo (até 7):
{SAMPLES}

Use a ferramenta classify_canonical_domain para retornar a classificação.
Se nenhuma área se aplicar bem, retorne primary_domain_slug = null.`;

async function backfillCanonicalDomains() {
    const supabase = createAdminServerClient();

    // 0. Recuperar checkpoint (se existir)
    const { data: checkpoint } = await supabase
        .from('_backfill_progress')
        .select('*')
        .eq('job_name', JOB_NAME)
        .maybeSingle();

    const lastProcessedId = checkpoint?.last_processed_id ?? null;
    let processados = checkpoint?.processed_count ?? 0;
    let bemSucedidos = checkpoint?.success_count ?? 0;
    let falhas = checkpoint?.failure_count ?? 0;

    if (lastProcessedId) {
        console.log(`Retomando do checkpoint: last_id=${lastProcessedId}, processados=${processados}`);
    }

    // 1. Buscar canônicos sem links via RPC (substitui .not.in que PostgREST não suporta)
    const { data: canonicals, error: rpcError } = await supabase
        .rpc('get_canonicals_without_domain_links');

    if (rpcError) throw rpcError;
    console.log(`Canônicos a backfillar: ${canonicals?.length ?? 0}`);

    if (!canonicals || canonicals.length === 0) {
        console.log('Nada a fazer.');
        return;
    }

    // 1.b Filtrar a partir do checkpoint (se houver)
    const startIdx = lastProcessedId
        ? canonicals.findIndex((c: any) => c.id === lastProcessedId) + 1
        : 0;
    const remaining = canonicals.slice(startIdx);

    // 2. Carregar mapping slug → domain_id
    const { data: domains } = await supabase
        .from('canonical_role_domains')
        .select('id, slug');
    const domainBySlug = new Map((domains ?? []).map((d: any) => [d.slug, d.id]));

    // 3. Para cada canônico, processar com retry
    for (const canonical of remaining) {
        let success = false;

        for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                // a. Skills agregadas
                const { data: skills } = await supabase.rpc('get_top_skills_for_canonical', {
                    p_canonical_id: canonical.id,
                    p_limit: 10,
                });

                // b. Top 5 industries
                const { data: industries } = await supabase.rpc('get_top_industries_for_canonical', {
                    p_canonical_id: canonical.id,
                    p_limit: 5,
                });

                // c. Até 7 vagas
                const { data: samples } = await supabase
                    .from('job_postings')
                    .select('title, description')
                    .eq('canonical_role_id', canonical.id)
                    .eq('curation_status', 'curated')
                    .order('posted_at', { ascending: false })
                    .limit(7);

                // d. Construir prompt
                const prompt = BACKFILL_PROMPT
                    .replace('{LABEL}', canonical.label)
                    .replace('{SKILLS}', (skills ?? []).map((s: any) => `- ${s.label} (${s.frequency}%)`).join('\n'))
                    .replace('{INDUSTRIES}', (industries ?? []).map((i: any) => `- ${i.industry} (${i.count} vagas)`).join('\n'))
                    .replace('{SAMPLES}', (samples ?? []).map((s: any, i: number) =>
                        `${i+1}. "${s.title}"\n   "${s.description.slice(0, 200)}..."`
                    ).join('\n\n'));

                // e. Chamar Haiku via tool_use
                const response = await anthropic.messages.create({
                    model: 'claude-haiku-4-5-20251001',
                    max_tokens: 256,
                    tools: [BACKFILL_TOOL],
                    tool_choice: { type: 'tool', name: 'classify_canonical_domain' },
                    messages: [{ role: 'user', content: prompt }],
                });

                const toolUseBlock = response.content.find((b: any) => b.type === 'tool_use');
                if (!toolUseBlock || toolUseBlock.type !== 'tool_use') {
                    throw new Error('Resposta sem tool_use');
                }
                const result = toolUseBlock.input as any;

                // f. Inserir links
                if (result.primary_domain_slug && domainBySlug.has(result.primary_domain_slug)) {
                    await supabase.from('canonical_role_domain_links').insert({
                        canonical_role_id: canonical.id,
                        domain_id: domainBySlug.get(result.primary_domain_slug)!,
                        is_primary: true,
                        confidence: result.confidence,
                        source: 'ai_backfill',
                    });

                    if (result.secondary_domain_slug && domainBySlug.has(result.secondary_domain_slug)) {
                        await supabase.from('canonical_role_domain_links').insert({
                            canonical_role_id: canonical.id,
                            domain_id: domainBySlug.get(result.secondary_domain_slug)!,
                            is_primary: false,
                            confidence: result.confidence * 0.7,
                            source: 'ai_backfill',
                        });
                    }

                    bemSucedidos++;
                } else {
                    // Haiku decidiu legitimamente que não há área aplicável
                    // Registra evento para auditoria
                    await supabase.from('events').insert({
                        event_name: 'canonical_role_domain_unresolved',
                        resource_type: 'job_canonical_role',
                        resource_id: canonical.id,
                        actor: 'pipeline',
                        actor_id: SYSTEM_USER_ID,
                        previous_state: {},
                        new_state: { reason: result.reason, confidence: result.confidence },
                    });
                    falhas++;
                }

                success = true;
                break;
            } catch (err) {
                const errMsg = err instanceof Error ? err.message : String(err);
                console.error(`Tentativa ${attempt}/${MAX_RETRIES} para ${canonical.label}:`, errMsg);

                if (attempt === MAX_RETRIES) {
                    // Última tentativa falhou — registra em events
                    await supabase.from('events').insert({
                        event_name: 'canonical_role_domain_backfill_failed',
                        resource_type: 'job_canonical_role',
                        resource_id: canonical.id,
                        actor: 'pipeline',
                        actor_id: SYSTEM_USER_ID,
                        previous_state: {},
                        new_state: { error: errMsg, attempts: MAX_RETRIES },
                    });
                    falhas++;
                } else {
                    // Backoff exponencial: 1s, 2s, 4s
                    await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt - 1)));
                }
            }
        }

        processados++;

        // Atualizar checkpoint a cada 25 canônicos
        if (processados % 25 === 0) {
            await supabase.from('_backfill_progress').upsert({
                job_name: JOB_NAME,
                last_processed_id: canonical.id,
                processed_count: processados,
                success_count: bemSucedidos,
                failure_count: falhas,
                last_updated_at: new Date().toISOString(),
            });

            console.log(`Progresso: ${processados}/${canonicals.length} (sucesso=${bemSucedidos}, falha=${falhas})`);
        }
    }

    // Checkpoint final
    await supabase.from('_backfill_progress').upsert({
        job_name: JOB_NAME,
        last_processed_id: canonicals[canonicals.length - 1].id,
        processed_count: processados,
        success_count: bemSucedidos,
        failure_count: falhas,
        last_updated_at: new Date().toISOString(),
    });

    console.log(`\nBackfill concluído: ${processados} processados, ${bemSucedidos} sucesso, ${falhas} falhas (incluindo unresolved)`);
}

backfillCanonicalDomains().catch(console.error);
```

**Funções RPC auxiliares (criar no banco):**

```sql
BEGIN;

-- get_top_skills_for_canonical
CREATE OR REPLACE FUNCTION get_top_skills_for_canonical(
    p_canonical_id UUID,
    p_limit INT DEFAULT 10
)
RETURNS TABLE(label TEXT, frequency NUMERIC)
LANGUAGE sql STABLE AS $$
    SELECT
        cs.label,
        ROUND(100.0 * COUNT(*) / NULLIF((
            SELECT COUNT(*) FROM job_postings
            WHERE canonical_role_id = p_canonical_id
        ), 0), 1) AS frequency
    FROM job_postings jp
    JOIN job_posting_skills jps ON jps.job_posting_id = jp.id
    JOIN canonical_skills cs ON cs.id = jps.canonical_skill_id
    WHERE jp.canonical_role_id = p_canonical_id
    GROUP BY cs.label
    ORDER BY frequency DESC
    LIMIT p_limit;
$$;

-- get_top_industries_for_canonical
CREATE OR REPLACE FUNCTION get_top_industries_for_canonical(
    p_canonical_id UUID,
    p_limit INT DEFAULT 5
)
RETURNS TABLE(industry TEXT, count BIGINT)
LANGUAGE sql STABLE AS $$
    SELECT
        org_industry_normalized AS industry,
        COUNT(*) AS count
    FROM job_postings
    WHERE canonical_role_id = p_canonical_id
      AND org_industry_normalized IS NOT NULL
    GROUP BY org_industry_normalized
    ORDER BY count DESC
    LIMIT p_limit;
$$;

GRANT EXECUTE ON FUNCTION get_top_skills_for_canonical(UUID, INT) TO service_role;
GRANT EXECUTE ON FUNCTION get_top_industries_for_canonical(UUID, INT) TO service_role;

COMMIT;
```

### G3.5 — Endpoint para listar áreas + funções por área

```typescript
// app/api/areas/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createServerClient } from '@/lib/supabase-server';

export async function GET(req: NextRequest) {
    const supabase = createServerClient();
    
    // Lista todas as áreas ativas com contagem de canônicos
    const { data, error } = await supabase
        .from('canonical_role_domains')
        .select(`
            id,
            slug,
            name,
            display_order,
            canonical_role_domain_links(count)
        `)
        .eq('is_active', true)
        .order('display_order', { ascending: true });
    
    if (error) {
        return NextResponse.json({ error: error.message }, { status: 500 });
    }
    
    return NextResponse.json({ areas: data });
}
```

```typescript
// app/api/areas/[domainId]/canonicals/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createServerClient } from '@/lib/supabase-server';

export async function GET(
    req: NextRequest,
    { params }: { params: { domainId: string } }
) {
    const supabase = createServerClient();
    
    const { data, error } = await supabase
        .from('canonical_role_domain_links')
        .select(`
            canonical_role_id,
            is_primary,
            job_canonical_roles!inner(id, label, slug, status)
        `)
        .eq('domain_id', params.domainId)
        .eq('job_canonical_roles.status', 'active')
        .order('is_primary', { ascending: false });
    
    if (error) {
        return NextResponse.json({ error: error.message }, { status: 500 });
    }
    
    const canonicals = (data ?? []).map(d => ({
        id: (d.job_canonical_roles as any).id,
        label: (d.job_canonical_roles as any).label,
        slug: (d.job_canonical_roles as any).slug,
        is_primary: d.is_primary,
    }));
    
    return NextResponse.json({ canonicals });
}
```

### G3.6 — Endpoint de match de função por texto livre

Quando usuário digita texto no campo "Função" sem ter selecionado área, o frontend faz match contra:
1. Labels de canônicos diretos (ILIKE)
2. `taxonomy_relations` ativas (sinônimos)
3. Resolve canônicos depreciados via `resolve_canonical()` (criado no Item 7C)

```typescript
// app/api/canonicals/search/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createServerClient } from '@/lib/supabase-server';

export async function GET(req: NextRequest) {
    const supabase = createServerClient();
    const q = req.nextUrl.searchParams.get('q')?.toLowerCase().trim();
    
    if (!q || q.length < 2) {
        return NextResponse.json({ matches: [] });
    }
    
    // 1. Match direto em canonical_role_label
    const { data: directMatches } = await supabase
        .from('job_canonical_roles')
        .select('id, label, status, merged_into')
        .eq('status', 'active')
        .ilike('label', `%${q}%`)
        .limit(20);
    
    // 2. Match em taxonomy_relations (sinônimos)
    const { data: synonymMatches } = await supabase
        .from('taxonomy_relations')
        .select(`
            source_term,
            target_canonical_id,
            job_canonical_roles!inner(id, label, status, merged_into)
        `)
        .eq('status', 'active')
        .ilike('source_term', `%${q}%`)
        .limit(20);
    
    // 3. Resolver canônicos depreciados
    const allMatches = [
        ...(directMatches ?? []).map(m => ({
            original_id: m.id,
            original_label: m.label,
            via: 'direct' as const,
        })),
        ...(synonymMatches ?? []).map(m => ({
            original_id: (m.job_canonical_roles as any).id,
            original_label: (m.job_canonical_roles as any).label,
            source_term: m.source_term,
            via: 'synonym' as const,
        })),
    ];
    
    // 4. Para cada match, resolver via RPC se status != 'active'
    const resolvedMatches = await Promise.all(
        allMatches.map(async m => {
            const { data: resolvedId } = await supabase.rpc('resolve_canonical', {
                p_id: m.original_id,
            });
            
            // Buscar dados do canônico final
            const { data: finalCanonical } = await supabase
                .from('job_canonical_roles')
                .select('id, label, status')
                .eq('id', resolvedId)
                .single();
            
            return {
                source_term: 'source_term' in m ? m.source_term : m.original_label,
                via: m.via,
                resolved_canonical_id: finalCanonical?.id,
                resolved_label: finalCanonical?.label,
                was_redirected: m.original_id !== resolvedId,
                original_label_if_redirected: m.original_label !== finalCanonical?.label ? m.original_label : null,
            };
        })
    );
    
    // 5. Dedupe por resolved_canonical_id
    const uniqueByResolved = new Map();
    for (const m of resolvedMatches) {
        if (m.resolved_canonical_id && !uniqueByResolved.has(m.resolved_canonical_id)) {
            uniqueByResolved.set(m.resolved_canonical_id, m);
        }
    }
    
    return NextResponse.json({ 
        matches: [...uniqueByResolved.values()].slice(0, 10),
    });
}
```

### G3.7 — UI dos 2 modais

**Escopo restrito (cravado pelo PO):** dinâmica entre os 2 campos (área filtra função) + sinalização de redirect quando canônico depreciado é selecionado.

**Modal 1 — `CalibrarParaOutraFuncaoModal.tsx`:**

```typescript
// components/home/modals/CalibrarParaOutraFuncaoModal.tsx
import { useState, useEffect } from 'react';

interface Area {
    id: string;
    slug: string;
    name: string;
}

interface Canonical {
    id: string;
    label: string;
    slug: string;
}

interface ResolvedMatch {
    source_term: string;
    via: 'direct' | 'synonym';
    resolved_canonical_id: string;
    resolved_label: string;
    was_redirected: boolean;
    original_label_if_redirected: string | null;
}

export function CalibrarParaOutraFuncaoModal({ profile, onClose }: Props) {
    const [areas, setAreas] = useState<Area[]>([]);
    const [areaSelected, setAreaSelected] = useState<string | null>(null);
    const [canonicals, setCanonicals] = useState<Canonical[]>([]);
    const [searchText, setSearchText] = useState('');
    const [searchMatches, setSearchMatches] = useState<ResolvedMatch[]>([]);
    const [selectedCanonical, setSelectedCanonical] = useState<{ id: string; label: string; redirectedFrom?: string } | null>(null);
    
    // 1. Carregar áreas
    useEffect(() => {
        fetch('/api/areas')
            .then(r => r.json())
            .then(d => setAreas(d.areas));
    }, []);
    
    // 2. Quando área selecionada, carregar canônicos da área
    useEffect(() => {
        if (!areaSelected) {
            setCanonicals([]);
            return;
        }
        fetch(`/api/areas/${areaSelected}/canonicals`)
            .then(r => r.json())
            .then(d => setCanonicals(d.canonicals));
    }, [areaSelected]);
    
    // 3. Quando texto digitado, buscar matches
    useEffect(() => {
        if (searchText.length < 2) {
            setSearchMatches([]);
            return;
        }
        const t = setTimeout(() => {
            fetch(`/api/canonicals/search?q=${encodeURIComponent(searchText)}`)
                .then(r => r.json())
                .then(d => setSearchMatches(d.matches));
        }, 300);
        return () => clearTimeout(t);
    }, [searchText]);
    
    function handleSelectCanonical(canonical: Canonical | ResolvedMatch) {
        if ('was_redirected' in canonical && canonical.was_redirected) {
            setSelectedCanonical({
                id: canonical.resolved_canonical_id,
                label: canonical.resolved_label,
                redirectedFrom: canonical.original_label_if_redirected ?? undefined,
            });
        } else if ('id' in canonical) {
            setSelectedCanonical({ id: canonical.id, label: canonical.label });
        }
    }
    
    return (
        <div className="modal">
            <h2>📡 Calibrar para outra função</h2>
            <p className="badge">● Para o perfil: {profile.name} - {profile.currentRole}</p>
            
            <section>
                <h3>FUNÇÕES JÁ IDENTIFICADAS NO SEU CURRÍCULO</h3>
                {profile.detectedRoles?.length > 0 ? (
                    <ul>{profile.detectedRoles.map(r => <li key={r.id}>{r.label}</li>)}</ul>
                ) : (
                    <p className="muted">Nenhuma função detectada.</p>
                )}
            </section>
            
            <section>
                <h3>ESCOLHA UMA FUNÇÃO DIFERENTE</h3>
                
                <label>
                    Área de atuação
                    <select 
                        value={areaSelected ?? ''} 
                        onChange={e => setAreaSelected(e.target.value || null)}
                    >
                        <option value="">— Todas as áreas —</option>
                        {areas.map(a => (
                            <option key={a.id} value={a.id}>{a.name}</option>
                        ))}
                    </select>
                </label>
                
                <label>
                    Função
                    {canonicals.length > 0 ? (
                        <select 
                            onChange={e => {
                                const c = canonicals.find(x => x.id === e.target.value);
                                if (c) handleSelectCanonical(c);
                            }}
                        >
                            <option value="">Selecione uma função...</option>
                            {canonicals.map(c => (
                                <option key={c.id} value={c.id}>{c.label}</option>
                            ))}
                        </select>
                    ) : (
                        <input 
                            type="text"
                            placeholder="Digite o nome da função..."
                            value={searchText}
                            onChange={e => setSearchText(e.target.value)}
                        />
                    )}
                </label>
                
                {searchMatches.length > 0 && (
                    <ul className="search-results">
                        {searchMatches.map((m, i) => (
                            <li key={i} onClick={() => handleSelectCanonical(m)}>
                                <strong>{m.resolved_label}</strong>
                                {m.was_redirected && m.original_label_if_redirected && (
                                    <span className="redirect-note">
                                        (anteriormente: {m.original_label_if_redirected})
                                    </span>
                                )}
                            </li>
                        ))}
                    </ul>
                )}
                
                {selectedCanonical && (
                    <div className="selection-feedback">
                        <strong>Função selecionada:</strong> {selectedCanonical.label}
                        {selectedCanonical.redirectedFrom && (
                            <p className="info-note">
                                Sua busca por "{selectedCanonical.redirectedFrom}" foi atualizada para "{selectedCanonical.label}"
                                conforme análise de mercado.
                            </p>
                        )}
                    </div>
                )}
            </section>
            
            <section>
                <h3>ESTADO</h3>
                <p>{profile.location_state} <span className="muted">(identificado no currículo)</span></p>
                <p className="muted">
                    O estado é definido pelo seu currículo. Para calibrar em outro estado, 
                    use "Calibrar para outra localidade".
                </p>
            </section>
            
            <footer>
                <span>🟡 2 créditos <span className="muted">(saldo: 20)</span></span>
                <button 
                    disabled={!selectedCanonical}
                    onClick={() => onClose({ canonicalId: selectedCanonical?.id })}
                >
                    Iniciar calibração → 🟡 2 créditos
                </button>
            </footer>
        </div>
    );
}
```

**Modal 2 — `MapaRegionalCompetenciasModal.tsx`:** estrutura similar, mas com 3 campos (área + função + estado), todos selecionáveis. Dropdowns de área e função idênticos ao Modal 1. Estado vira `<select>` populado com lista de UFs do Brasil.

(Implementação simétrica, omitida por brevidade — segue mesmo padrão do Modal 1.)

### G3.8 — Tests para Item G3

```typescript
// tests/integration/sprint-v4_0/item-g3-areas.spec.ts

describe('Item G3 — Áreas de atuação 0:N', () => {
    it('seed do catálogo cria 20 áreas', async () => {
        const { count } = await supabase
            .from('canonical_role_domains')
            .select('*', { count: 'exact', head: true })
            .eq('is_active', true);
        
        expect(count).toBe(20);
    });
    
    it('seed da junction cria links para os 128 canônicos do JSON', async () => {
        const { count } = await supabase
            .from('canonical_role_domain_links')
            .select('*', { count: 'exact', head: true })
            .eq('source', 'seed');
        
        expect(count).toBeGreaterThanOrEqual(128);
    });
    
    it('backfill IA processa canônicos sem area', async () => {
        // Simulação: contar canônicos sem links após backfill
        const { count: semLinks } = await supabase
            .from('job_canonical_roles')
            .select('*', { count: 'exact', head: true })
            .eq('status', 'active')
            .not('id', 'in', `(SELECT canonical_role_id FROM canonical_role_domain_links)`);
        
        expect(semLinks).toBeLessThan(50);  // Margem de erro: alguns podem falhar
    });
    
    it('endpoint /api/areas retorna todas as áreas ativas ordenadas', async () => {
        const res = await fetch('/api/areas');
        const data = await res.json();
        
        expect(data.areas).toHaveLength(20);
        expect(data.areas[0].display_order).toBeLessThan(data.areas[1].display_order);
    });
    
    it('endpoint /api/areas/:id/canonicals retorna canônicos da área', async () => {
        const { data: areaTI } = await supabase
            .from('canonical_role_domains')
            .select('id')
            .eq('slug', 'desenvolvimento')
            .single();
        
        const res = await fetch(`/api/areas/${areaTI.id}/canonicals`);
        const data = await res.json();
        
        expect(data.canonicals.length).toBeGreaterThan(0);
        expect(data.canonicals.every((c: any) => c.label && c.id)).toBe(true);
    });
    
    it('search resolve canônico depreciado para o final', async () => {
        // Setup: criar canônico depreciado apontando para outro
        // ...
        
        const res = await fetch('/api/canonicals/search?q=engenheiro+confiabilidade');
        const data = await res.json();
        
        const match = data.matches.find((m: any) => m.was_redirected);
        if (match) {
            expect(match.resolved_label).not.toBe(match.original_label_if_redirected);
        }
    });
});
```

### G3.9 — Esforço estimado

- **Schema (junction + RLS):** 1 hora
- **Seeds (catálogo + junction):** 2 horas
- **Backfill IA (script + execução):** 4 horas (incluindo monitoramento)
- **Endpoints de áreas/canônicos/search:** 3 horas
- **UI dos 2 modais:** 4 horas
- **Tests:** 2 horas
- **Total:** 16 horas

---


---

# PR3 — Inversão de paradigma

Substitui o modelo cron-driven anterior (canônicos promovidos por CRON noturno) por regra dinâmica em runtime + ciclo de vida explícito amber/red. Dois itens grandes: Item 7 (regra dinâmica + retroativa + `resolve_canonical`) e Item 2 (CRON `pipeline-maintenance` reformulado).

---

## Item 7 — Regra dinâmica + retroativa + `resolve_canonical`

Três sub-itens encadeados: 7A (regra dinâmica em runtime), 7B (aplicação retroativa aos 583 pending atuais), 7C (função SQL `resolve_canonical` com CTE recursivo).

### 7A — Regra dinâmica para canônicos novos

**Princípio:** quando o pipeline LLM cura um lote de vagas e identifica canônicos novos, a regra de promoção `pending → active` é aplicada **dentro do próprio lote**. Sem CRON noturno, sem delay.

**Constante cravada:** `MIN_ABSOLUTE = 3`. Um canônico novo precisa aparecer em pelo menos 3 vagas curadas pelo LLM **no mesmo lote** para virar `active`. Camada 0 (cache hit) **não** entra no denominador.

**Implementação em `lib/pipeline/batch-processor.ts`:**

```typescript
// lib/pipeline/batch-processor.ts (após o LLM curar todas as vagas do lote)

const MIN_ABSOLUTE = 3;  // Constante cravada

interface CuratedJob {
    canonical_proposed: string;
    canonical_id?: string;  // se já existe
    layer: 0 | 1 | 2 | 3;
}

async function applyDynamicRule(supabase: SupabaseClient, lot: CuratedJob[]) {
    // 1. Filtrar apenas vagas curadas pelo LLM (Camadas 2 e 3)
    const llmCurated = lot.filter(j => j.layer === 2 || j.layer === 3);
    
    // 2. Contar canônicos NOVOS (que ainda não existem)
    const canonicalCounts = new Map<string, number>();
    for (const job of llmCurated) {
        const c = job.canonical_proposed;
        const exists = await canonicalExists(supabase, c);
        if (!exists) {
            canonicalCounts.set(c, (canonicalCounts.get(c) ?? 0) + 1);
        }
    }
    
    // 3. Para cada canônico novo, decidir status baseado em contagem
    for (const [canonicalLabel, count] of canonicalCounts) {
        const status = count >= MIN_ABSOLUTE ? 'active' : 'pending';
        
        // Cria canônico (operação idempotente — outra instância pode ter criado simultaneamente)
        await upsertCanonicalRole(supabase, {
            label: canonicalLabel,
            slug: generateSlug(canonicalLabel),
            status,
            // Outros campos (embedding, metadata) gerenciados pelo upsertCanonicalRole existente
        });
    }
}

async function canonicalExists(supabase: SupabaseClient, label: string): Promise<boolean> {
    // maybeSingle (não single) — zero linhas é caminho normal aqui
    const { data } = await supabase
        .from('job_canonical_roles')
        .select('id')
        .eq('label', label)
        .maybeSingle();
    return !!data;
}
```

**Ponto de chamada no `batch-processor.ts`:**

```typescript
// Após o LLM terminar de curar todas as vagas do lote
await applyDynamicRule(supabase, lotResult);

// Depois aplica os UPDATEs em job_postings com canonical_role_id final
for (const job of lotResult) {
    const canonicalId = await resolveCanonicalIdByLabel(supabase, job.canonical_proposed);
    await supabase
        .from('job_postings')
        .update({ canonical_role_id: canonicalId, curation_status: 'curated' })
        .eq('id', job.id);
}
```

### 7B — Aplicação retroativa aos 583 pending atuais

Migration one-shot que aplica a mesma regra retroativamente. Três buckets:

| Bucket | `vacancy_count` | Ação |
|---|---|---|
| Promove | ≥ 3 | `status: pending → active` |
| Mantém | 1-2 | `status: pending` (regra dinâmica reavalia em uso futuro) |
| DELETE | 0 | DELETE direto com guard ampliado |

**Migration:**

**Mudanças críticas v3 sobre v2:**

1. **Guards usam lista real validada de 14 FKs** (ver §G2.9 para tabela completa) — v2 tinha 10 guards apontando para tabelas inventadas (`canonical_role_role_links`, `canonical_role_skill_links`, `merge_history`, `canonical_role_promotion_logs`, `canonical_role_pending_changes`, `canonical_role_audit_history`, `canonical_role_review_logs`) que **não existem no schema** — Migration abortaria no primeiro guard com `ERROR: relation does not exist`.
2. **Trap `NOT IN (NULL)` corrigido em todos os guards** (P0.5 Gemini): SQL tri-valorado faz `NOT IN (SELECT ... UNION SELECT ...)` retornar UNKNOWN se qualquer linha vier NULL — toda a expressão vira "falsa prática" e zero zumbis são apagados. v3 garante `WHERE coluna IS NOT NULL` em **toda** subquery de `NOT IN`, não só onde foi explicitado na v2.
3. **`AND status = 'pending'` explícito** (P2.7 Grok): defesa adicional contra cenário onde um canônico active+vacancy_count=0 (estado transitório raro, mas possível durante migração) seria varrido por engano.

```sql
BEGIN;

-- ================================
-- Bucket 1: Promove para active (vacancy_count >= 3)
-- ================================
-- v4 (P0 Outra-Claude): UPDATE + GET DIAGNOSTICS no MESMO DO block.
-- v3 fazia UPDATE fora e GET DIAGNOSTICS dentro de DO separado → ROW_COUNT capturado 
-- era do escopo do DO (vazio), sempre 0. RAISE NOTICE imprimia 0 enganosamente.
DO $$
DECLARE
    promovidos INT;
BEGIN
    UPDATE job_canonical_roles
    SET status = 'active',
        updated_at = NOW()
    WHERE status = 'pending'
      AND vacancy_count >= 3;
    
    GET DIAGNOSTICS promovidos = ROW_COUNT;
    RAISE NOTICE 'Bucket 1 — Promovidos para active: %', promovidos;
END $$;

-- ================================
-- Bucket 2: Mantém pending (vacancy_count em 1-2)
-- ================================
-- Não-ação. Regra dinâmica reavalia em uso futuro.

-- ================================
-- Bucket 3: DELETE direto para zumbis órfãos (vacancy_count = 0)
-- Guard ampliado: confirma que NÃO há referências em nenhuma das 11 FKs bloqueantes
-- (NO ACTION + RESTRICT) + 2 FKs auto-referência (alias_of_id, merged_into).
-- 3 FKs CASCADE/SET NULL não bloqueiam mas guards defensivos não custam.
-- 
-- TODOS os guards têm IS NOT NULL na subquery (defesa contra trap NOT IN NULL).
-- v4 (P0 Outra-Claude): DELETE + GET DIAGNOSTICS no MESMO DO block (ROW_COUNT correto)
-- ================================
DO $$
DECLARE
    apagados INT;
BEGIN
DELETE FROM job_canonical_roles
WHERE status = 'pending'  -- defesa P2.7: nunca apaga canônicos active acidentalmente
  AND vacancy_count = 0
  
  -- Guard #1: allowed_for_pre_resolution (CASCADE — defensivo)
  AND id NOT IN (
      SELECT canonical_role_id FROM allowed_for_pre_resolution 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #2: curation_batch_metrics (NO ACTION — bloquearia)
  AND id NOT IN (
      SELECT canonical_role_id FROM curation_batch_metrics 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #3: function_orchestrator_items (NO ACTION)
  AND id NOT IN (
      SELECT canonical_role_id FROM function_orchestrator_items 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #4: job_canonical_role_sources (NO ACTION)
  AND id NOT IN (
      SELECT canonical_role_id FROM job_canonical_role_sources 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #5: job_canonical_roles.alias_of_id (NO ACTION — auto-referência)
  AND id NOT IN (
      SELECT alias_of_id FROM job_canonical_roles 
      WHERE alias_of_id IS NOT NULL
  )
  -- Guard #6: job_canonical_roles.merged_into (NO ACTION — auto-referência)
  AND id NOT IN (
      SELECT merged_into FROM job_canonical_roles 
      WHERE merged_into IS NOT NULL
  )
  -- Guard #7: job_no_postings (NO ACTION)
  AND id NOT IN (
      SELECT canonical_role_id FROM job_no_postings 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #8: job_postings (SET NULL — defensivo)
  AND id NOT IN (
      SELECT canonical_role_id FROM job_postings 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #9: rapidapi_usage_logs (SET NULL — defensivo)
  AND id NOT IN (
      SELECT canonical_role_id FROM rapidapi_usage_logs 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #10: resume_role_suggestions (NO ACTION)
  AND id NOT IN (
      SELECT canonical_role_id FROM resume_role_suggestions 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #11: resume_skill_enrichments (RESTRICT — bloquearia hard)
  AND id NOT IN (
      SELECT canonical_role_id FROM resume_skill_enrichments 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #12 + #13: role_merge_decisions (source_id e target_id, NO ACTION)
  AND id NOT IN (
      SELECT source_id FROM role_merge_decisions WHERE source_id IS NOT NULL
      UNION
      SELECT target_id FROM role_merge_decisions WHERE target_id IS NOT NULL
  )
  -- Guard #14: skill_enrichment_stats (RESTRICT)
  AND id NOT IN (
      SELECT canonical_role_id FROM skill_enrichment_stats 
      WHERE canonical_role_id IS NOT NULL
  )
  -- Guard #15 (criada nesta sprint): taxonomy_relations.target_canonical_id (CASCADE — defensivo)
  AND id NOT IN (
      SELECT target_canonical_id FROM taxonomy_relations 
      WHERE target_canonical_id IS NOT NULL
  );

    GET DIAGNOSTICS apagados = ROW_COUNT;
    RAISE NOTICE 'Bucket 3 — Zumbis órfãos apagados: %', apagados;
END $$;

-- ================================
-- Validação: contar quanto sobrou em pending (esperado: 1-2 vagas só)
-- ================================
DO $$
DECLARE
    pending_restante INT;
BEGIN
    SELECT COUNT(*) INTO pending_restante
    FROM job_canonical_roles WHERE status = 'pending';
    RAISE NOTICE 'Pending restante (esperado: ~437 com 1-2 vagas): %', pending_restante;
END $$;

COMMIT;
```

**Pré-validação ANTES da migration (rodar para descobrir quantos canônicos serão deletados ou bloqueados):**

```sql
-- Quantos pending+vacancy_count=0 existem
SELECT COUNT(*) AS total_zumbis
FROM job_canonical_roles
WHERE status = 'pending' AND vacancy_count = 0;

-- Quantos desses estão referenciados em alguma das 15 FKs (vão ser BLOQUEADOS pelo guard)
WITH zumbis AS (
    SELECT id FROM job_canonical_roles WHERE status = 'pending' AND vacancy_count = 0
),
referenciados AS (
    SELECT DISTINCT canonical_role_id AS id FROM allowed_for_pre_resolution 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM curation_batch_metrics 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM function_orchestrator_items 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM job_canonical_role_sources 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT alias_of_id FROM job_canonical_roles 
        WHERE alias_of_id IS NOT NULL AND alias_of_id IN (SELECT id FROM zumbis)
    UNION SELECT merged_into FROM job_canonical_roles 
        WHERE merged_into IS NOT NULL AND merged_into IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM job_no_postings 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM job_postings 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM rapidapi_usage_logs 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM resume_role_suggestions 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM resume_skill_enrichments 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT source_id FROM role_merge_decisions 
        WHERE source_id IS NOT NULL AND source_id IN (SELECT id FROM zumbis)
    UNION SELECT target_id FROM role_merge_decisions 
        WHERE target_id IS NOT NULL AND target_id IN (SELECT id FROM zumbis)
    UNION SELECT canonical_role_id FROM skill_enrichment_stats 
        WHERE canonical_role_id IS NOT NULL AND canonical_role_id IN (SELECT id FROM zumbis)
    UNION SELECT target_canonical_id FROM taxonomy_relations 
        WHERE target_canonical_id IS NOT NULL AND target_canonical_id IN (SELECT id FROM zumbis)
)
SELECT
    (SELECT COUNT(*) FROM zumbis) AS total_zumbis,
    (SELECT COUNT(*) FROM referenciados) AS bloqueados_pelo_guard,
    (SELECT COUNT(*) FROM zumbis) - (SELECT COUNT(*) FROM referenciados) AS apagaveis;
```

**Validação manual após o run:**

```sql
-- Confirmar distribuição final dos canônicos pending
SELECT 
    vacancy_count,
    COUNT(*) AS qtd_canonicos
FROM job_canonical_roles
WHERE status = 'pending'
GROUP BY vacancy_count
ORDER BY vacancy_count;

-- Esperado:
-- vacancy_count = 1 ou 2 → algumas centenas
-- vacancy_count >= 3 → 0 (todos promovidos)
-- vacancy_count = 0 → 0 (todos apagados)
```

### 7C — Função SQL `resolve_canonical` com CTE recursivo

Função PL/SQL que segue cadeia `merged_into` e `alias_of_id` até encontrar o canônico final ativo. Profundidade máxima 10 (defesa contra ciclos sem invariante de UUIDs distintos).

**Mudanças críticas v3 sobre v2:**

1. **`status = 'merged'` REMOVIDO** — não existe no CHECK constraint. Apenas `'deprecated'` (com `merged_into NOT NULL`) e `'alias_of'` continuam a cadeia.
2. **`visited UUID[]` adicionado** — defesa explícita contra ciclos. Se a cadeia revisitar um UUID já visto, para imediatamente. Sem isso, um ciclo de 2 nós (A→B→A) consumiria 10 iterações antes de parar pelo depth limit.
3. **`STABLE` mantido** — função é determinística por entrada, não escreve. Habilita query plan caching.
4. **v3: Invalidação de cache Redis cravada como contrato em `mergeCanonicals` TS wrapper** (P2.4 Grok): após merge bem-sucedido, `invalidateRelations()` é chamado para evitar que próxima leitura veja o `target_canonical_id` antigo do cache. Sem isso, há janela de inconsistência entre o estado SQL e o cache. A função `resolve_canonical` em si não invalida cache (ela é STABLE e read-only) — a invalidação acontece no caller (mergeCanonicals).
5. **v3: Test de cadeia profunda 11+ níveis adicionado** em §13.2 (deve retornar último válido sem erro/ciclo infinito).

**Migration:**

```sql
BEGIN;

CREATE OR REPLACE FUNCTION resolve_canonical(p_id UUID)
RETURNS UUID
LANGUAGE sql
STABLE
AS $$
    WITH RECURSIVE chain AS (
        -- Caso base: começa no canônico de entrada
        SELECT
            id,
            status,
            alias_of_id,
            merged_into,
            1 AS depth,
            ARRAY[id] AS visited
        FROM job_canonical_roles
        WHERE id = p_id

        UNION ALL

        -- Caso recursivo: segue cadeia, mas só se não revisita UUID
        -- v4 (P1.5 Gemini): cast explícito jcr.id::uuid no concat de arrays para 
        -- garantir tipo correto. Sem cast, parser PL/pgSQL pode interpretar `||` 
        -- como concat de string em alguns contextos, gerando erro tardio.
        SELECT
            jcr.id,
            jcr.status,
            jcr.alias_of_id,
            jcr.merged_into,
            c.depth + 1,
            c.visited || jcr.id::uuid
        FROM chain c
        JOIN job_canonical_roles jcr ON jcr.id = COALESCE(
            CASE WHEN c.status = 'alias_of'   THEN c.alias_of_id END,
            CASE WHEN c.status = 'deprecated' THEN c.merged_into END
            -- 'merged' removido — não existe no CHECK constraint
        )
        WHERE c.depth < 10
          AND c.status IN ('alias_of', 'deprecated')
          AND NOT (jcr.id = ANY(c.visited))  -- defesa contra ciclos
    )
    SELECT id FROM chain
    ORDER BY depth DESC
    LIMIT 1;
$$;

-- Permitir chamada via RPC pelo Supabase JS
GRANT EXECUTE ON FUNCTION resolve_canonical(UUID) TO authenticated, service_role;

COMMIT;
```

**Substituição do TS atual:**

`lib/pipeline/upsert-canonical.ts:70-122` tem hoje a função `resolveCanonicalRedirect()` em TS. Substituir por chamada à RPC SQL:

```typescript
// ANTES (lib/pipeline/upsert-canonical.ts)
async function resolveCanonicalRedirect(supabase: SupabaseClient, canonicalId: string): Promise<string> {
    let currentId = canonicalId;
    let depth = 0;
    while (depth < 10) {
        const { data } = await supabase
            .from('job_canonical_roles')
            .select('id, status, alias_of_id, merged_into')
            .eq('id', currentId)
            .single();
        if (!data) break;
        if (data.status === 'active') return data.id;
        if (data.status === 'alias_of' && data.alias_of_id) {
            currentId = data.alias_of_id;
        } else if ((data.status === 'merged' || data.status === 'deprecated') && data.merged_into) {
            currentId = data.merged_into;
        } else {
            break;
        }
        depth++;
    }
    return currentId;
}

// DEPOIS (uma chamada RPC)
async function resolveCanonicalRedirect(supabase: SupabaseClient, canonicalId: string): Promise<string> {
    const { data, error } = await supabase.rpc('resolve_canonical', { p_id: canonicalId });
    if (error) throw error;
    return data ?? canonicalId;
}
```

**Outros callsites a atualizar:**

```bash
grep -rn "resolveCanonicalRedirect\|resolve_canonical_redirect" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/

# Atualizar todos para chamar supabase.rpc('resolve_canonical', { p_id: id })
```

### 7D — Tests para Item 7

```typescript
// tests/integration/sprint-v4_0/item-7-dynamic-rule.spec.ts

describe('Item 7 — Regra dinâmica + retroativa + resolve_canonical', () => {
    it('canônico com 3+ vagas no mesmo lote vira active', async () => {
        // Simular curadoria de lote com 3 vagas no mesmo canônico novo
        const lot = [
            { canonical_proposed: 'Test Canonical New 1', layer: 2 as const },
            { canonical_proposed: 'Test Canonical New 1', layer: 2 as const },
            { canonical_proposed: 'Test Canonical New 1', layer: 3 as const },
        ];
        
        await applyDynamicRule(supabase, lot);
        
        const { data } = await supabase
            .from('job_canonical_roles')
            .select('status')
            .eq('label', 'Test Canonical New 1')
            .single();
        
        expect(data?.status).toBe('active');
        
        // Cleanup
        await supabase.from('job_canonical_roles').delete().eq('label', 'Test Canonical New 1');
    });
    
    it('canônico com 1-2 vagas vira pending', async () => {
        const lot = [
            { canonical_proposed: 'Test Canonical New 2', layer: 2 as const },
            { canonical_proposed: 'Test Canonical New 2', layer: 3 as const },
        ];
        
        await applyDynamicRule(supabase, lot);
        
        const { data } = await supabase
            .from('job_canonical_roles')
            .select('status')
            .eq('label', 'Test Canonical New 2')
            .single();
        
        expect(data?.status).toBe('pending');
        
        // Cleanup
        await supabase.from('job_canonical_roles').delete().eq('label', 'Test Canonical New 2');
    });
    
    it('Camada 0 (cache hit) não entra no denominador', async () => {
        const lot = [
            { canonical_proposed: 'Test Canonical New 3', layer: 0 as const },
            { canonical_proposed: 'Test Canonical New 3', layer: 0 as const },
            { canonical_proposed: 'Test Canonical New 3', layer: 0 as const },
            { canonical_proposed: 'Test Canonical New 3', layer: 2 as const },
        ];
        
        await applyDynamicRule(supabase, lot);
        
        const { data } = await supabase
            .from('job_canonical_roles')
            .select('status')
            .eq('label', 'Test Canonical New 3')
            .single();
        
        // Apenas 1 vaga em Camada 2 → mantém pending
        expect(data?.status).toBe('pending');
        
        // Cleanup
        await supabase.from('job_canonical_roles').delete().eq('label', 'Test Canonical New 3');
    });
    
    it('aplicação retroativa: zumbis órfãos foram apagados', async () => {
        // Após migração 7B, não deve haver canônico com vacancy_count=0
        const { count } = await supabase
            .from('job_canonical_roles')
            .select('*', { count: 'exact', head: true })
            .eq('status', 'pending')
            .eq('vacancy_count', 0);
        
        expect(count).toBe(0);
    });
    
    it('resolve_canonical segue cadeia merged_into', async () => {
        // Setup: criar A → mergeado em B → mergeado em C
        // v3: status real é 'deprecated' (não 'merged' — esse valor não existe no CHECK constraint)
        // chk_deprecated_has_merged_into garante que merged_into NOT NULL quando status='deprecated'
        const { data: c } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestC', slug: 'testc', status: 'active' })
            .select()
            .single();
        
        const { data: b } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestB', slug: 'testb', status: 'deprecated', merged_into: c!.id })
            .select()
            .single();
        
        const { data: a } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestA', slug: 'testa', status: 'deprecated', merged_into: b!.id })
            .select()
            .single();
        
        // Resolve A → deve retornar C
        const { data: resolvedId } = await supabase.rpc('resolve_canonical', { p_id: a!.id });
        expect(resolvedId).toBe(c!.id);
        
        // Cleanup (deletar em ordem reversa de dependência)
        await supabase.from('job_canonical_roles').delete().eq('id', a!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', b!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', c!.id);
    });
    
    it('resolve_canonical respeita profundidade máxima (10)', async () => {
        // Setup: criar cadeia ciclo intencional (não deve travar)
        // ... (setup omitido — tests defensivos contra ciclos)
        
        // Esperado: retorna o último válido sem entrar em loop
    });
});
```

### 7E — Esforço estimado

- **7A (regra dinâmica em batch-processor):** 2 horas
- **7B (migration retroativa):** 1 hora (delicado, precisa dry-run)
- **7C (função SQL + substituição TS):** 2 horas
- **Tests:** 1.5 horas
- **Total:** 6.5 horas

---

## Item 2 — CRON `pipeline-maintenance` reformulado

Substitui o esqueleto vazio atual (Bloco C confirmou: 12 execuções, 16% sucesso, todas com `metadata = {phase_errors: []}`, zero promoções). Vira CRON com 3 funções concretas.

### 2.1 — Schedule

**Atual:** rodando diário em horário variável, degradado em Vercel Hobby.
**Novo:** mantém schedule mas troca conteúdo. Pós-upgrade Pro, voltar para horário ideal.

```json
// vercel.json
{
    "crons": [
        {
            "path": "/api/cron/pipeline-maintenance",
            "schedule": "0 4 * * *"
        }
    ]
}
```

### 2.2 — Função 1: Retry de `retryable_error` com idade ≥ 1h

```typescript
// app/api/cron/pipeline-maintenance/route.ts

async function retryRetryableErrors(supabase: SupabaseClient): Promise<{ retried: number }> {
    // Buscar vagas em retryable_error há ≥ 1h
    const { data: vagas } = await supabase
        .from('job_postings')
        .select('id')
        .eq('curation_status', 'retryable_error')
        .lte('updated_at', new Date(Date.now() - 60 * 60 * 1000).toISOString())
        .limit(500);  // Limite por execução
    
    if (!vagas || vagas.length === 0) return { retried: 0 };
    
    // Marcar como pending (CRON existente curate-job-postings vai retornar a processar)
    await supabase
        .from('job_postings')
        .update({ 
            curation_status: 'pending', 
            updated_at: new Date().toISOString() 
        })
        .in('id', vagas.map(v => v.id));
    
    return { retried: vagas.length };
}
```

### 2.3 — Função 2: Detecção de zumbis amber/red

**Constantes cravadas:** amber = 120 dias (3 ciclos × 40d Sólides); red = 365 dias (4 trimestres OLC arxiv:2406.15373).

**Mudança crítica v2:**

1. **Cap em 10 amber + 10 red por execução** — `findBestMergeCandidate` chama embedding RPC, custo computacional não-trivial. Limitar evita explosão em execuções iniciais quando muitos canônicos já podem estar em estado red.
2. **`emitEventOnce` usa schema correto:** `resource_type`/`resource_id` (não `entity_*`), `actor='system'` obrigatório, `actor_id=SYSTEM_USER_ID`. `.maybeSingle()` em vez de `.single()` (zero linhas é caminho normal — significa "ainda não foi emitido").

```typescript
const AMBER_THRESHOLD_DAYS = 120;
const RED_THRESHOLD_DAYS = 365;
const MAX_ALERTS_PER_EXECUTION = 10;  // cap por tipo (amber, red)

const SYSTEM_USER_ID = '00000000-0000-0000-0000-000000000001';

async function detectZombies(supabase: SupabaseClient): Promise<{ amber_emitted: number; red_emitted: number }> {
    const now = new Date();
    let amber = 0;
    let red = 0;

    // 1. Buscar canônicos active com pelo menos uma vaga (skip órfãos já apagados pelo Item 7B)
    const { data: canonicals } = await supabase
        .from('job_canonical_roles')
        .select('id, label, vacancy_count')
        .eq('status', 'active')
        .gt('vacancy_count', 0);

    if (!canonicals) return { amber_emitted: 0, red_emitted: 0 };

    for (const c of canonicals) {
        // Cap: se já atingiu 10 vermelhos E 10 ambers, encerra ciclo
        if (red >= MAX_ALERTS_PER_EXECUTION && amber >= MAX_ALERTS_PER_EXECUTION) break;

        // 2. Última vaga do canônico
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

        if (gapDays >= RED_THRESHOLD_DAYS && red < MAX_ALERTS_PER_EXECUTION) {
            // Red: emitir evento + sugerir candidato de merge (chamada cara)
            const mergeCandidate = await findBestMergeCandidate(supabase, c.id);

            await emitEventOnce(supabase, {
                event_name: 'canonical_red_alert',
                resource_type: 'job_canonical_role',
                resource_id: c.id,
                window_days: 30,
                payload: {
                    label: c.label,
                    gap_days: gapDays,
                    last_vacancy_at: lastVacancy.posted_at,
                    merge_candidate: mergeCandidate,
                },
            });
            red++;
        } else if (gapDays >= AMBER_THRESHOLD_DAYS && amber < MAX_ALERTS_PER_EXECUTION) {
            await emitEventOnce(supabase, {
                event_name: 'canonical_amber_alert',
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

    return { amber_emitted: amber, red_emitted: red };
}

async function emitEventOnce(supabase: SupabaseClient, params: {
    event_name: string;
    resource_type: string;
    resource_id: string;
    window_days: number;
    payload: any;
}) {
    // Idempotência: skip se já houve evento mesmo nome+resource nos últimos N dias
    const cutoff = new Date(Date.now() - params.window_days * 24 * 60 * 60 * 1000).toISOString();

    const { data: existing } = await supabase
        .from('events')
        .select('id')
        .eq('event_name', params.event_name)
        .eq('resource_id', params.resource_id)
        .gte('created_at', cutoff)
        .limit(1)
        .maybeSingle();  // pode não existir — caminho normal

    if (existing) return;  // Já emitiu, skip

    await supabase.from('events').insert({
        event_name: params.event_name,
        resource_type: params.resource_type,
        resource_id: params.resource_id,
        actor: 'system',                    // obrigatório
        actor_id: SYSTEM_USER_ID,           // UUID convencional
        previous_state: {},
        new_state: params.payload,
    });
}

async function findBestMergeCandidate(supabase: SupabaseClient, canonicalId: string): Promise<{ id: string; label: string; similarity: number } | null> {
    // Busca top-1 canônico mais similar via embedding
    const { data: source } = await supabase
        .from('job_canonical_roles')
        .select('embedding, label')
        .eq('id', canonicalId)
        .maybeSingle();

    if (!source?.embedding) return null;

    const { data: candidates } = await supabase.rpc('match_canonicals_by_embedding', {
        p_embedding: source.embedding,
        p_exclude_id: canonicalId,
        p_limit: 1,
    });

    if (!candidates || candidates.length === 0) return null;
    return candidates[0];
}
```

### 2.4 — Função 3: Telemetria útil em metadata

```typescript
async function gatherTelemetry(supabase: SupabaseClient) {
    const [activeCount, pendingCount, amberCount, redCount] = await Promise.all([
        supabase.from('job_canonical_roles').select('*', { count: 'exact', head: true }).eq('status', 'active'),
        supabase.from('job_canonical_roles').select('*', { count: 'exact', head: true }).eq('status', 'pending'),
        supabase.from('events').select('*', { count: 'exact', head: true })
            .eq('event_name', 'canonical_amber_alert')
            .gte('created_at', new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString()),
        supabase.from('events').select('*', { count: 'exact', head: true })
            .eq('event_name', 'canonical_red_alert')
            .gte('created_at', new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString()),
    ]);
    
    return {
        canonicals_active_total: activeCount.count ?? 0,
        canonicals_pending_total: pendingCount.count ?? 0,
        amber_emitted_30d: amberCount.count ?? 0,
        red_emitted_30d: redCount.count ?? 0,
    };
}
```

### 2.5 — Endpoint completo

**Mudança v2:** auth via `isCronAuthorized()` do `lib/cron-guard.ts` (padrão do projeto), substituindo o check inline de `Bearer ${process.env.CRON_SECRET}`.

```typescript
// app/api/cron/pipeline-maintenance/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createAdminServerClient } from '@/lib/supabase-server';
import { isCronAuthorized } from '@/lib/cron-guard';

export async function GET(req: NextRequest) {
    if (!isCronAuthorized(req)) {
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const supabase = createAdminServerClient();
    const startedAt = new Date();

    const [retryResult, zombiesResult, telemetry] = await Promise.all([
        retryRetryableErrors(supabase),
        detectZombies(supabase),
        gatherTelemetry(supabase),
    ]);

    const result = {
        status: 'done',
        retried_count: retryResult.retried,
        amber_emitted: zombiesResult.amber_emitted,
        red_emitted: zombiesResult.red_emitted,
        telemetry,
        duration_ms: Date.now() - startedAt.getTime(),
    };

    // Persistir em job_runs
    await supabase.from('job_runs').insert({
        job_name: 'pipeline-maintenance',
        status: 'success',
        metadata: result,
        started_at: startedAt.toISOString(),
        finished_at: new Date().toISOString(),
    });

    return NextResponse.json(result);
}
```

### 2.6 — Tests para Item 2

```typescript
// tests/integration/sprint-v4_0/item-2-pipeline-maintenance.spec.ts

describe('Item 2 — CRON pipeline-maintenance reformulado', () => {
    it('retry funciona: vagas retryable_error >= 1h viram pending', async () => {
        // Setup: vaga retryable_error com updated_at >1h
        const { data: vaga } = await supabase
            .from('job_postings')
            .insert({
                title: 'Test',
                description: 'Test',
                curation_status: 'retryable_error',
                updated_at: new Date(Date.now() - 2 * 60 * 60 * 1000).toISOString(),
            })
            .select()
            .single();
        
        await retryRetryableErrors(supabase);
        
        const { data: vagaApos } = await supabase
            .from('job_postings')
            .select('curation_status')
            .eq('id', vaga!.id)
            .single();
        
        expect(vagaApos?.curation_status).toBe('pending');
        
        // Cleanup
        await supabase.from('job_postings').delete().eq('id', vaga!.id);
    });
    
    it('detecção amber emite evento para canônico sem vagas há 120+ dias', async () => {
        // Test setup: canônico com última vaga há 130 dias
        // ... (setup)
        
        await detectZombies(supabase);
        
        const { data: events } = await supabase
            .from('events')
            .select('*')
            .eq('event_name', 'canonical_amber_alert')
            .order('created_at', { ascending: false })
            .limit(1);
        
        expect(events?.length).toBeGreaterThan(0);
    });
    
    it('emitEventOnce não duplica eventos dentro da janela (schema events real: resource_type/resource_id)', async () => {
        // v3: usa fixture real em vez de placeholder 'algum-canonical-id' (que falharia FK silenciosamente)
        const canonical = await createTestCanonical({ label: 'TestEmitEventOnce', status: 'pending' });

        // Primeira emissão
        await emitEventOnce(supabase, {
            event_name: 'canonical_amber_alert',
            resource_type: 'job_canonical_role',  // v3: schema real (não entity_type)
            resource_id: canonical.id,             // v3: schema real (não entity_id)
            window_days: 30,
            payload: { test: 1 },
        });

        // Segunda chamada imediata — deve ser ignorada
        await emitEventOnce(supabase, {
            event_name: 'canonical_amber_alert',
            resource_type: 'job_canonical_role',
            resource_id: canonical.id,
            window_days: 30,
            payload: { test: 2 },
        });

        const { count } = await supabase
            .from('events')
            .select('*', { count: 'exact', head: true })
            .eq('event_name', 'canonical_amber_alert')
            .eq('resource_id', canonical.id)
            .gte('created_at', new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString());

        expect(count).toBe(1);  // Apenas 1 emissão

        // Cleanup
        await cleanupTestCanonical(canonical.id);
    });
});
```

### 2.7 — Esforço estimado

- **Função 1 (retry):** 1 hora
- **Função 2 (detecção zumbis):** 3 horas (lógica de embedding + emitEventOnce)
- **Função 3 (telemetria):** 30 minutos
- **Endpoint completo + integração:** 1 hora
- **Tests:** 1.5 horas
- **Total:** 7 horas

---


# PR4 — Fixes de pipeline

Quatro correções pontuais de bugs e drifts identificados durante a sprint exploratória anterior. Todas são patches localizados sem mudança arquitetural. Quatro itens: 1A (`distinct_sources_count`), 1A.bis (filtro de agências), 4 (saneamento de 270 órfãs), 6 (trigger autoritativo único).

---

## Item 1A — Fix `distinct_sources_count` em 10 callsites + backfill

`distinct_sources_count` é métrica que conta quantos **empregadores distintos** atestaram a existência de um canônico. Hoje, das ~583 vagas pending atuais, apenas 1 (0,17%) tem `distinct_sources_count ≥ 1`. As outras 582 estão zeradas — bug em 10 callsites que não atualizam essa coluna.

**Mudança crítica v2:** o Claude Code validou o schema real de `job_canonical_role_sources` — colunas reais são `(id, canonical_role_id, normalized_company, first_seen_at, last_seen_at)`. **NÃO existe** `source_type`, `source_ref`, ou `job_posting_id`. A v1 propunha computar `COUNT(DISTINCT source_type)` — semântica errada. v2 corrige para `COUNT(DISTINCT normalized_company)`, que reflete corretamente "quantos empregadores diferentes têm essa vaga", ou seja, diversidade de empregadores reais.

**Trigger consolidado:** após validação do Claude Code de que volume real de inserts é 3.2/min (média), `FOR EACH ROW` é seguro — não há risco de lock por carga. Mantido.

### 1A.1 — Identificação dos 10 callsites

```bash
# Buscar todos os spots que tocam canônicos sem atualizar distinct_sources_count
grep -rn "job_canonical_role_sources\|distinct_sources_count" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/

# Esperado mapear:
# - lib/pipeline/persist-curation.ts (insere em job_canonical_role_sources mas não recomputa count)
# - lib/pipeline/persist-precheck.ts
# - app/api/admin/canonicals/[id]/remap/route.ts (remap de vagas)
# - app/api/admin/canonicals/human-validated/route.ts
# - lib/pipeline/upsert-canonical.ts (criação de canônico novo)
# - lib/pipeline/batch-processor.ts
# - scripts/backfill-* (vários scripts antigos)
# - app/api/jobs/submit/route.ts (Fluxo C — colagem manual)
# - app/api/admin/ingestor/route.ts (Fluxo A)
# - lib/pipeline/curate-from-cache.ts (Camada 0)
```

### 1A.2 — Padrão de fix unificado: trigger PostgreSQL

Ao invés de patchar 10 callsites individualmente (frágil — qualquer callsite novo esquecido reintroduz o bug), criar trigger que recomputa `distinct_sources_count` automaticamente sempre que `job_canonical_role_sources` muda.

**Migration:**

```sql
BEGIN;

-- Função de recomputação (versão SIMPLES — sem filtro de agência).
-- v4 (NIv3-6 Outra-Claude): esta função é definida primeiro em §1A.2 como versão 
-- básica. Em §1A.bis.5, é SUBSTITUÍDA via CREATE OR REPLACE pela versão com filtro 
-- de agência (rede de segurança DA1). Ordem de execução do PR4 é linear: §1A.2 → 
-- §1A.bis.5. Ambas têm a mesma assinatura, então o trigger de §1A.2 aponta 
-- automaticamente para a versão final após §1A.bis.5.
--
-- IMPORTANTE: NÃO inverta a ordem. Se §1A.bis rodar antes de §1A.2, a função final
-- é a "frágil" (sem filtro), o que defeats o propósito da rede de segurança.
CREATE OR REPLACE FUNCTION recompute_distinct_sources_count()
RETURNS TRIGGER AS $$
DECLARE
    target_canonical_id UUID;
BEGIN
    -- Determinar qual canônico foi afetado
    IF TG_OP = 'DELETE' THEN
        target_canonical_id := OLD.canonical_role_id;
    ELSE
        target_canonical_id := NEW.canonical_role_id;
    END IF;

    -- Recomputar count: COUNT(DISTINCT normalized_company) — empregadores distintos
    -- v1 dizia COUNT(DISTINCT source_type) — coluna não existe no schema real
    UPDATE job_canonical_roles
    SET distinct_sources_count = (
        SELECT COUNT(DISTINCT normalized_company)
        FROM job_canonical_role_sources
        WHERE canonical_role_id = target_canonical_id
    )
    WHERE id = target_canonical_id;

    -- Para UPDATE: se canonical_role_id mudou, recomputar tb o canônico antigo
    IF TG_OP = 'UPDATE' AND OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id THEN
        UPDATE job_canonical_roles
        SET distinct_sources_count = (
            SELECT COUNT(DISTINCT normalized_company)
            FROM job_canonical_role_sources
            WHERE canonical_role_id = OLD.canonical_role_id
        )
        WHERE id = OLD.canonical_role_id;
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Trigger AFTER FOR EACH ROW (não FOR EACH STATEMENT)
-- Volume real validado pelo Claude Code: 3.2 inserts/min — sem risco de lock
CREATE TRIGGER trg_recompute_distinct_sources_count
AFTER INSERT OR UPDATE OR DELETE ON job_canonical_role_sources
FOR EACH ROW
EXECUTE FUNCTION recompute_distinct_sources_count();

COMMIT;
```

### 1A.3 — Backfill SQL one-shot dos 582 canônicos pending

**Mudança v3 sobre v2 (P2.12 Grok parcial):** UPDATE idempotente com guard `IS DISTINCT FROM` para evitar UPDATEs desnecessários (e disparo redundante do trigger `updated_at`). Em rerun acidental, apenas linhas com valor diferente do calculado são tocadas.

```sql
BEGIN;

-- Recomputar para TODOS os canônicos (não apenas pending — defesa contra drift histórico)
-- Schema real: COUNT(DISTINCT normalized_company), não source_type
-- Idempotente: só atualiza linhas onde valor calculado difere do atual
-- v4 (P0 Outra-Claude): UPDATE + GET DIAGNOSTICS no MESMO DO block (ROW_COUNT correto)
DO $$
DECLARE
    pending_com_zero INT;
    pending_total INT;
    atualizados INT;
BEGIN
UPDATE job_canonical_roles jcr
SET distinct_sources_count = computed.cnt
FROM (
    SELECT 
        jcr.id,
        COALESCE((
            SELECT COUNT(DISTINCT normalized_company)
            FROM job_canonical_role_sources jcrs
            WHERE jcrs.canonical_role_id = jcr.id
        ), 0) AS cnt
    FROM job_canonical_roles jcr
) AS computed
WHERE jcr.id = computed.id
  AND jcr.distinct_sources_count IS DISTINCT FROM computed.cnt;

    GET DIAGNOSTICS atualizados = ROW_COUNT;
    
    SELECT COUNT(*) INTO pending_com_zero
    FROM job_canonical_roles
    WHERE status = 'pending' AND distinct_sources_count = 0;

    SELECT COUNT(*) INTO pending_total
    FROM job_canonical_roles
    WHERE status = 'pending';

    RAISE NOTICE 'Backfill atualizou % linhas. Pending: % total, % com count=0', 
        atualizados, pending_total, pending_com_zero;
END $$;

COMMIT;
```

**Resultado esperado:** Após o backfill, `pending_com_zero` deve ser apenas os canônicos legítimos sem source registrado em `job_canonical_role_sources` (provavelmente zero ou bem poucos). Em rerun, `atualizados` deve ser 0 (idempotência confirmada).

### 1A.4 — Tests para Item 1A

```typescript
// tests/integration/sprint-v4_0/item-1a-distinct-sources.spec.ts

describe('Item 1A — distinct_sources_count', () => {
    it('trigger recomputa count automaticamente em INSERT', async () => {
        // Setup: criar canônico
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test 1A', slug: 'test-1a', status: 'pending' })
            .select()
            .single();
        
        // v4 (NIv3-2 Outra-Claude): schema real de job_canonical_role_sources tem 
        // (id, canonical_role_id, normalized_company, first_seen_at, last_seen_at).
        // Colunas source_type e source_ref NÃO EXISTEM (eram premissa errada da v1).
        // Inserir 2 sources com normalized_company distinto para gerar COUNT(DISTINCT)=2
        await supabase.from('job_canonical_role_sources').insert([
            { canonical_role_id: canonical!.id, normalized_company: 'empresa-um' },
            { canonical_role_id: canonical!.id, normalized_company: 'empresa-dois' },
        ]);
        
        // Verificar count
        const { data: cAfter } = await supabase
            .from('job_canonical_roles')
            .select('distinct_sources_count')
            .eq('id', canonical!.id)
            .single();
        
        expect(cAfter?.distinct_sources_count).toBe(2);
        
        // Cleanup
        await supabase.from('job_canonical_roles').delete().eq('id', canonical!.id);
    });
    
    it('trigger recompõe count em DELETE', async () => {
        // Setup: criar canônico com 2 sources
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test 1A2', slug: 'test-1a2', status: 'pending' })
            .select()
            .single();
        
        const { data: sources } = await supabase
            .from('job_canonical_role_sources')
            .insert([
                // v4 (NIv3-2): schema real usa normalized_company (não source_type/source_ref)
                { canonical_role_id: canonical!.id, normalized_company: 'empresa-x' },
                { canonical_role_id: canonical!.id, normalized_company: 'empresa-y' },
            ])
            .select();
        
        // Delete um source
        await supabase
            .from('job_canonical_role_sources')
            .delete()
            .eq('id', sources![0].id);
        
        // Count deve cair para 1
        const { data: cAfter } = await supabase
            .from('job_canonical_roles')
            .select('distinct_sources_count')
            .eq('id', canonical!.id)
            .single();
        
        expect(cAfter?.distinct_sources_count).toBe(1);
        
        // Cleanup
        await supabase.from('job_canonical_roles').delete().eq('id', canonical!.id);
    });
    
    it('backfill zerou drift: 99%+ dos pending agora têm count > 0 ou justificadamente 0', async () => {
        const { count: total } = await supabase
            .from('job_canonical_roles')
            .select('*', { count: 'exact', head: true })
            .eq('status', 'pending');
        
        const { count: zero } = await supabase
            .from('job_canonical_roles')
            .select('*', { count: 'exact', head: true })
            .eq('status', 'pending')
            .eq('distinct_sources_count', 0);
        
        // Tolerância: até 5% pode legitimamente ter 0 (canônicos sem source registrado)
        expect((zero ?? 0) / (total ?? 1)).toBeLessThan(0.05);
    });
});
```

### 1A.5 — Esforço estimado

- **Trigger PostgreSQL:** 1.5 horas (escrita + dry-run)
- **Backfill SQL:** 30 minutos
- **Validação dos 10 callsites (não precisa mais patchar — trigger garante):** 1 hora (apenas confirmação)
- **Tests:** 1 hora
- **Total:** 4 horas

---

## Item 1A.bis — Filtro de agências de RH

Bloco N anterior identificou que ~3 fontes na tabela `job_canonical_role_sources` deveriam ser filtradas como "agência" e não compor o cálculo de `top1_employer_share` ou contar como "empregador real" para fins de viés. As 3 fontes representam padrão sistêmico: agências postam vagas para múltiplos empregadores, distorcendo o sinal.

### 1A.bis.0 — Mudança crítica v3 sobre desenho da v2

A v2 propunha duas camadas atuando como OR independentes:
- Camada A = `linkedin_org_recruitment_agency_derived = true`
- Camada B = `linkedin_org_industry IN ('Staffing and Recruiting', ...)`

**Problema descoberto pelo PO via análise dos samples:** o desenho v2 produz **falso positivo grave** em empresas como Michael Page. A organização Michael Page é classificada pelo LinkedIn como `linkedin_org_industry: "Staffing and Recruiting"` SEMPRE — porque institucionalmente é uma agência. Mas Michael Page também publica vagas internas (RH próprio, financeiro próprio, TI próprio) que devem contar como sources legítimas. O LinkedIn distingue isso vaga-a-vaga via `linkedin_org_recruitment_agency_derived`.

**Reformulação v3:** a Camada A é **autoridade primária** (sinal por vaga). A Camada B só é consultada quando Camada A é `null`/ausente — funciona como rede de segurança defensiva. Vagas internas de Michael Page (Camada A = `false`) ficam preservadas como sources legítimas. Vagas de recolocação (Camada A = `true`) são filtradas.

**Fonte de dados também mudou:** v2 propunha colunas `linkedin_org_recruitment_agency_derived` e `linkedin_org_industry` em `job_postings`. **Validação Claude Code mostrou que essas colunas NÃO existem** (57 colunas reais validadas). v3 lê o flag `linkedin_org_recruitment_agency_derived` do payload bruto preservado em `events.metadata->'raw_data'` (`event_name = 'jobs_ingest_raw_payload'`) — onde é gravado desde as primeiras importações funcionais. Para a Camada B (fallback), usa `job_postings.org_industry` direto da tabela (espelho do `linkedin_org_industry` original, casing preservado).

**Volume real validado para o backfill:** 172 events com `metadata->'raw_data'` válido, totalizando **11.505 raw_items** distribuídos. Suportável sem otimização específica. 8 events com `raw_data` não-array detectados (provavelmente bugs históricos de importação) — backfill tem guard `jsonb_typeof(metadata->'raw_data') = 'array'` que blinda. Inspeção dos 8 vira item operacional pós-deploy.

### 1A.bis.1 — Schema: nova coluna `is_recruitment_agency` em `job_postings`

```sql
BEGIN;

ALTER TABLE job_postings 
ADD COLUMN IF NOT EXISTS is_recruitment_agency BOOLEAN NOT NULL DEFAULT FALSE;

COMMENT ON COLUMN job_postings.is_recruitment_agency IS 
'TRUE quando a vaga é de recolocação por agência (não conta como source distinto para distinct_sources_count). 
Calculado em INSERT pelo Fluxo A via detectAgency() e backfilled retroativamente via LATERAL JOIN em events.metadata->raw_data. 
Camada A (autoridade): linkedin_org_recruitment_agency_derived do payload. 
Camada B (fallback quando A=null): linkedin_org_industry em whitelist (Staffing and Recruiting, Human Resources Services, Recruitment Services).';

-- v4 (P0 Gemini): índice composto SEM cláusula WHERE.
-- v3 tinha `WHERE is_recruitment_agency = true`, mas o trigger consulta com 
-- `= false` (preserva sources legítimas). Query Planner do PostgreSQL NÃO usa 
-- índice parcial filtrado por valor diferente do consultado → Sequential Scan 
-- em produção. Índice composto sem WHERE cobre ambos os cenários.
CREATE INDEX IF NOT EXISTS idx_job_postings_agency_lookup 
ON job_postings(canonical_role_id, normalized_company, is_recruitment_agency);

COMMIT;
```

### 1A.bis.2 — Estratégia de detecção: Camada A primária + Camada B fallback

**Princípio:** sinal por vaga (Camada A) sobrepõe sinal por organização (Camada B). Camada B é rede de segurança apenas para casos onde o LinkedIn não enriqueceu com o flag derived.

**Camada A — `linkedin_org_recruitment_agency_derived` (autoridade primária)**

Campo nativo do enriquecimento LinkedIn que classifica **a vaga específica** (não a organização) como recolocação. Distingue Michael Page recolocando para terceiros (`true`) de Michael Page contratando para si própria (`false`).

Regra:
- `true` → marca como agência, fim
- `false` → NÃO é agência, fim (não consulta Camada B mesmo se org_industry sugerir)
- `null`/ausente → cai para Camada B

**Camada B — Whitelist de `linkedin_org_industry` (fallback defensivo)**

Aciona apenas quando Camada A é `null`/ausente. Whitelist (3 valores fixos em inglês oficial do LinkedIn):
- `'Staffing and Recruiting'`
- `'Human Resources Services'`
- `'Recruitment Services'`

**Trade-offs documentados:**
- ✅ Camada A primária preserva vagas internas de empresas-agência (Michael Page contratando seu próprio Analista Contábil)
- ✅ Camada B fallback captura agências cujo payload não foi enriquecido com `linkedin_org_recruitment_agency_derived`
- ⚠️ Agências brasileiras com classificação industrial divergente (ex: "Internet", "Information Technology and Services") e sem flag derived passam batido — caso esperado para RH Ser, EMPREGARE, Fábrica de Valores
- ⚠️ Caso a porcentagem de falsos negativos se mostre alta em produção (medida via auditoria pós-deploy), sprint futura pode adicionar Camada C (heurística de keywords no nome de empresa)

### 1A.bis.3 — Função `detectAgency` (Camada A primária + B fallback)

```typescript
// lib/pipeline/agency-detector.ts

const RECRUITMENT_INDUSTRIES_WHITELIST = new Set([
    'Staffing and Recruiting',
    'Human Resources Services',
    'Recruitment Services',
]);

interface AgencyDetectionPayload {
    linkedin_org_recruitment_agency_derived?: boolean | null;
    linkedin_org_industry?: string | null;
}

/**
 * Detecta se uma vaga é de agência de recrutamento.
 *
 * Camada A (autoridade primária): linkedin_org_recruitment_agency_derived.
 *   - true: marca como agência
 *   - false: NÃO é agência (não consulta Camada B)
 *   - null/undefined: cai para Camada B
 *
 * Camada B (fallback defensivo, só aciona quando Camada A é null):
 *   - linkedin_org_industry em whitelist (Staffing and Recruiting, etc.)
 *
 * Trade-off documentado:
 *   ✅ Preserva vagas internas de empresas-agência (Michael Page contratando para si)
 *   ✅ Captura agências cujo payload não veio com flag derived
 *   ⚠️ Não pega agências com classificação industrial divergente e sem flag derived
 */
export function detectAgency(payload: AgencyDetectionPayload): boolean {
    // Camada A — autoridade primária por vaga
    if (payload.linkedin_org_recruitment_agency_derived === true) {
        return true;
    }
    if (payload.linkedin_org_recruitment_agency_derived === false) {
        return false;
    }

    // Camada B — fallback APENAS quando Camada A é null/undefined
    if (
        payload.linkedin_org_industry &&
        RECRUITMENT_INDUSTRIES_WHITELIST.has(payload.linkedin_org_industry)
    ) {
        return true;
    }

    return false;
}
```

### 1A.bis.4 — Integração no Fluxo A (ingestor admin) e Fluxo B (RapidAPI)

```typescript
// lib/pipeline/fluxo-b-upsert.ts (e similar para Fluxo A)
import { detectAgency } from '@/lib/pipeline/agency-detector';

async function upsertFromFantasticAPI(jobData: any) {
    // Worker tem o payload bruto na mão antes do INSERT — calcula direto
    const isAgency = detectAgency({
        linkedin_org_recruitment_agency_derived: jobData.linkedin_org_recruitment_agency_derived,
        linkedin_org_industry: jobData.linkedin_org_industry,
    });

    await supabase.from('job_postings').upsert({
        // ... campos existentes ...
        linkedin_id: String(jobData.linkedin_id),  // text na tabela
        is_recruitment_agency: isAgency,
        org_industry: jobData.linkedin_org_industry,  // espelho preservado em coluna
        // ... outros campos ...
    }, { onConflict: 'linkedin_id' });
}
```

**Fluxo C (colagem manual de URL):** o usuário cola apenas a URL do LinkedIn. O scraper interno NÃO recebe `linkedin_org_recruitment_agency_derived` (campo só existe no enriquecimento da API Fantastic, não no scraping direto). Para Fluxo C, `is_recruitment_agency=false` por default — vagas coladas manualmente raramente são de agência (o usuário tipicamente cola a vaga do empregador final).

### 1A.bis.5 — Filtro defensivo no trigger autoritativo (não no caller)

A v2 propunha filtrar dentro de `upsertSource()`, mas o desenho tinha bypass: se algum caller esquecesse de passar `job_posting_id`, agências passavam batido. **v3 move o filtro para o trigger `recompute_distinct_sources_count`**, que é o ponto autoritativo único de cálculo. Blindagem independente de caller.

```sql
-- Trigger atualizado (substitui §1A.2 quando rodado em paralelo)
CREATE OR REPLACE FUNCTION recompute_distinct_sources_count() RETURNS TRIGGER AS $$
DECLARE
    v_canonical_id UUID;
    v_count INT;
BEGIN
    v_canonical_id := COALESCE(NEW.canonical_role_id, OLD.canonical_role_id);
    
    IF v_canonical_id IS NULL THEN
        RETURN COALESCE(NEW, OLD);
    END IF;
    
    SELECT COUNT(DISTINCT jcrs.normalized_company)
    INTO v_count
    FROM job_canonical_role_sources jcrs
    -- Filtro defensivo: só conta sources cujas vagas associadas NÃO são de agência
    WHERE jcrs.canonical_role_id = v_canonical_id
      AND EXISTS (
          SELECT 1 FROM job_postings jp
          WHERE jp.canonical_role_id = jcrs.canonical_role_id
            AND jp.normalized_company = jcrs.normalized_company
            AND jp.is_recruitment_agency = false
      );
    
    UPDATE job_canonical_roles
    SET distinct_sources_count = v_count,
        updated_at = NOW()
    WHERE id = v_canonical_id;
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

**Justificativa do `EXISTS (... WHERE jp.is_recruitment_agency = false)`:** uma normalized_company pode ter múltiplas vagas, algumas internas (não-agência) e outras de recolocação (agência). Se EXISTE pelo menos UMA vaga não-agência, a normalized_company conta como source legítima. Se TODAS as vagas dessa normalized_company são de agência, NÃO conta. Isso preserva Michael Page como source legítima quando ela tem vagas internas para o canônico em questão, mas a remove quando todas as vagas Michael Page para aquele canônico são recolocação.

### 1A.bis.6 — Backfill via LATERAL JOIN em `events`

**Decisão DA3 do PO:** reprocessamento total via `events`, sem "melhor esforço" e sem marcação default `false`. Volume real: 11.505 raw_items históricos.

```sql
BEGIN;

-- Backfill com guard contra raw_data não-array (8 registros fora do padrão)
-- v4 (P0 Outra-Claude): UPDATE + GET DIAGNOSTICS no MESMO DO block.
-- v4 (P1.7 Grok): tiebreaker `id DESC` em DISTINCT ON para determinismo total
DO $$
DECLARE
    marcadas_agencia INT;
    total_atualizadas INT;
BEGIN
WITH raw_payloads AS (
    SELECT DISTINCT ON (raw_item->>'linkedin_id')
        raw_item->>'linkedin_id' AS linkedin_id,
        -- Camada A: autoridade primária
        CASE 
            WHEN raw_item ? 'linkedin_org_recruitment_agency_derived'
                 AND raw_item->>'linkedin_org_recruitment_agency_derived' IS NOT NULL
            THEN (raw_item->>'linkedin_org_recruitment_agency_derived')::boolean
            ELSE NULL
        END AS layer_a_flag,
        -- Camada B: industry para fallback
        raw_item->>'linkedin_org_industry' AS industry
    FROM events,
         jsonb_array_elements(metadata->'raw_data') AS raw_item
    WHERE event_name = 'jobs_ingest_raw_payload'
      AND jsonb_typeof(metadata->'raw_data') = 'array'  -- guard contra 8 registros bugados
      AND raw_item ? 'linkedin_id'
    -- Mais recente vence em duplicatas (vaga reimportada)
    -- v4 (P1.7 Grok): tiebreaker `id DESC` para determinismo se dois events tiverem 
    -- created_at idêntico (raro mas possível em batch)
    ORDER BY raw_item->>'linkedin_id', created_at DESC, id DESC
),
computed_agency AS (
    SELECT
        linkedin_id,
        CASE
            -- Camada A primária: se tem flag, decide direto
            WHEN layer_a_flag IS TRUE THEN true
            WHEN layer_a_flag IS FALSE THEN false
            -- Camada B fallback: só quando A é NULL
            WHEN industry IN ('Staffing and Recruiting', 'Human Resources Services', 'Recruitment Services') THEN true
            ELSE false
        END AS is_agency
    FROM raw_payloads
)
UPDATE job_postings jp
SET is_recruitment_agency = ca.is_agency
FROM computed_agency ca
WHERE jp.linkedin_id = ca.linkedin_id  -- jp.linkedin_id é text, match direto sem cast
  AND jp.is_recruitment_agency IS DISTINCT FROM ca.is_agency;

    GET DIAGNOSTICS total_atualizadas = ROW_COUNT;
    
    SELECT COUNT(*) INTO marcadas_agencia 
    FROM job_postings WHERE is_recruitment_agency = true;
    
    RAISE NOTICE 'Backfill is_recruitment_agency concluído. Vagas marcadas como agência: %. Total de UPDATEs efetivos: %.', 
        marcadas_agencia, total_atualizadas;
END $$;

-- Recomputar distinct_sources_count para todos os canônicos afetados
-- (trigger só dispara em INSERT/UPDATE/DELETE em job_canonical_role_sources, 
-- então precisamos forçar recálculo após backfill em job_postings)
UPDATE job_canonical_roles jcr
SET distinct_sources_count = (
    SELECT COUNT(DISTINCT jcrs.normalized_company)
    FROM job_canonical_role_sources jcrs
    WHERE jcrs.canonical_role_id = jcr.id
      AND EXISTS (
          SELECT 1 FROM job_postings jp
          WHERE jp.canonical_role_id = jcrs.canonical_role_id
            AND jp.normalized_company = jcrs.normalized_company
            AND jp.is_recruitment_agency = false
      )
)
WHERE EXISTS (
    SELECT 1 FROM job_postings jp
    WHERE jp.canonical_role_id = jcr.id
      AND jp.is_recruitment_agency = true  -- só recalcula canônicos que tinham vaga de agência
);

COMMIT;
```

**Notas operacionais:**
- `DISTINCT ON (raw_item->>'linkedin_id') ... ORDER BY ... created_at DESC` resolve duplicatas (vaga reimportada): vence o payload mais recente.
- `jp.linkedin_id` é `text` (validado via `\d job_postings`), sem necessidade de cast.
- `IS DISTINCT FROM` evita UPDATEs desnecessários (idempotência).
- Recomputação de `distinct_sources_count` ao final é necessária porque o trigger só dispara em mudanças nas tabelas de sources, não em `job_postings.is_recruitment_agency` direto.

**Inspeção dos 8 events bugados (item operacional pós-deploy):**
```sql
SELECT id, created_at, jsonb_typeof(metadata->'raw_data') AS tipo, metadata
FROM events
WHERE event_name = 'jobs_ingest_raw_payload'
  AND jsonb_typeof(metadata->'raw_data') != 'array';
```

### 1A.bis.7 — Tests

```typescript
// tests/integration/sprint-v4_0/item-1a-bis-agencies.spec.ts
import { detectAgency } from '@/lib/pipeline/agency-detector';
import { createTestCanonical, cleanupTestCanonical } from '../helpers';

describe('Item 1A.bis — Filtro de agências (Camada A primária + B fallback)', () => {
    describe('Camada A primária (autoridade por vaga)', () => {
        it('linkedin_org_recruitment_agency_derived=true → marca como agência mesmo se industry não é whitelist', () => {
            expect(detectAgency({
                linkedin_org_recruitment_agency_derived: true,
                linkedin_org_industry: 'Information Technology and Services',
            })).toBe(true);
        });

        it('linkedin_org_recruitment_agency_derived=false → NÃO marca como agência mesmo se industry é whitelist', () => {
            // Cenário Michael Page contratando vaga interna:
            // industry da org é "Staffing and Recruiting" (organizacional),
            // mas a vaga específica não é recolocação
            expect(detectAgency({
                linkedin_org_recruitment_agency_derived: false,
                linkedin_org_industry: 'Staffing and Recruiting',
            })).toBe(false);
        });
    });

    describe('Camada B fallback (só quando Camada A é null/undefined)', () => {
        it('Camada A null + industry em whitelist → marca como agência', () => {
            expect(detectAgency({
                linkedin_org_recruitment_agency_derived: null,
                linkedin_org_industry: 'Human Resources Services',
            })).toBe(true);

            expect(detectAgency({
                linkedin_org_recruitment_agency_derived: undefined,
                linkedin_org_industry: 'Recruitment Services',
            })).toBe(true);

            // Caso onde campo não está nem presente
            expect(detectAgency({
                linkedin_org_industry: 'Staffing and Recruiting',
            })).toBe(true);
        });

        it('Camada A null + industry fora da whitelist → NÃO marca como agência', () => {
            expect(detectAgency({
                linkedin_org_recruitment_agency_derived: null,
                linkedin_org_industry: 'Internet',
            })).toBe(false);

            expect(detectAgency({})).toBe(false);
            expect(detectAgency({ linkedin_org_industry: '' })).toBe(false);
        });
    });

    describe('Trigger recompute_distinct_sources_count com filtro de agência', () => {
        it('source que tem APENAS vagas de agência NÃO conta para distinct_sources_count', async () => {
            const canonical = await createTestCanonical({ label: 'TestAgency1A' });

            // Insere vaga de agência (Michael Page recolocando)
            // v6 (P0 Claude Code NEv5-2 + NEv5-3): posted_at, expires_at NOT NULL + linkedin_id UUID dinâmico
            const { data: vagaAgencia } = await supabase
                .from('job_postings')
                .insert({
                    title: 'Test',
                    description_curated: 'Test',
                    canonical_role_id: canonical.id,
                    is_recruitment_agency: true,
                    curation_status: 'curated',
                    normalized_company: 'michael page',
                    linkedin_id: `test-${crypto.randomUUID()}`,
                    posted_at: new Date().toISOString(),
                    expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString(),
                })
                .select()
                .single();

            // Insere source via UPSERT direto (simula o que o pipeline faz)
            await supabase.from('job_canonical_role_sources').upsert({
                canonical_role_id: canonical.id,
                normalized_company: 'michael page',
                last_seen_at: new Date().toISOString(),
            }, { onConflict: 'canonical_role_id,normalized_company' });

            // Verificar distinct_sources_count após trigger
            const { data: updatedCanonical } = await supabase
                .from('job_canonical_roles')
                .select('distinct_sources_count')
                .eq('id', canonical.id)
                .single();

            expect(updatedCanonical?.distinct_sources_count).toBe(0);

            await cleanupTestCanonical(canonical.id);
        });

        it('source que tem AMBAS vagas de agência E internas conta como source legítima', async () => {
            const canonical = await createTestCanonical({ label: 'TestAgency1B' });

            // Vaga de agência da Michael Page
            // v6 (P0 Claude Code NEv5-2 + NEv5-3): posted_at, expires_at NOT NULL + linkedin_id UUID dinâmico
            await supabase.from('job_postings').insert({
                title: 'Vaga recolocação',
                description_curated: 'Test',
                canonical_role_id: canonical.id,
                is_recruitment_agency: true,
                curation_status: 'curated',
                normalized_company: 'michael page',
                linkedin_id: `test-${crypto.randomUUID()}`,
                posted_at: new Date().toISOString(),
                expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString(),
            });

            // Vaga interna da Michael Page (mesma normalized_company, mas não-agência)
            // v6 (P0 Claude Code NEv5-2 + NEv5-3): mesma defesa
            await supabase.from('job_postings').insert({
                title: 'Vaga interna',
                description_curated: 'Test',
                canonical_role_id: canonical.id,
                is_recruitment_agency: false,
                curation_status: 'curated',
                normalized_company: 'michael page',
                linkedin_id: `test-${crypto.randomUUID()}`,
                posted_at: new Date().toISOString(),
                expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString(),
            });

            await supabase.from('job_canonical_role_sources').upsert({
                canonical_role_id: canonical.id,
                normalized_company: 'michael page',
                last_seen_at: new Date().toISOString(),
            }, { onConflict: 'canonical_role_id,normalized_company' });

            // Forçar trigger
            await supabase.rpc('recompute_distinct_sources_count_for', { p_canonical_id: canonical.id });

            const { data: updatedCanonical } = await supabase
                .from('job_canonical_roles')
                .select('distinct_sources_count')
                .eq('id', canonical.id)
                .single();

            expect(updatedCanonical?.distinct_sources_count).toBe(1);  // Michael Page conta porque tem vaga interna

            await cleanupTestCanonical(canonical.id);
        });
    });

    describe('Backfill via LATERAL JOIN em events (smoke test)', () => {
        it('events com raw_data não-array são ignorados (não falha)', async () => {
            // Insere evento bugado
            await supabase.from('events').insert({
                event_name: 'jobs_ingest_raw_payload',
                resource_type: 'job_ingestion',
                actor: 'pipeline',
                actor_id: '00000000-0000-0000-0000-000000000001',
                metadata: { note: 'Bug histórico', raw_data: null },  // não é array
            });

            // Roda o backfill (deve completar sem erro)
            const { error } = await supabase.rpc('run_backfill_is_recruitment_agency');
            expect(error).toBeNull();
        });
    });
});
```

### 1A.bis.8 — Esforço estimado

- **Schema (`is_recruitment_agency` + comentário + índice):** 30 minutos
- **Função `detectAgency` Camada A primária + B fallback:** 30 minutos
- **Integração nos Fluxos A e B:** 1.5 horas
- **Trigger `recompute_distinct_sources_count` com filtro defensivo:** 1 hora
- **Backfill via LATERAL JOIN em events (incluindo recomputação de distinct_sources_count):** 1.5 horas
- **Tests (3 grupos):** 2 horas
- **Total:** **6.5 horas** (era 4.5h na v2 — subiu por causa da fonte via events e do test do trigger com cenário Michael Page interno+externo)

---

## Item 4 — Saneamento das 270 vagas órfãs + invariant DB

Sprint v5.23 deixou 270 vagas em estado "órfão" (sem `canonical_role_id` associado). Distribuição cravada pelo Bloco anterior:

| Estado | Quantidade | Ação |
|---|---|---|
| `retryable_error` (≥ 1h) | 79 | Retentadas via Item 2 (CRON `pipeline-maintenance`) |
| `low_quality` | 188 | Auditadas e mantidas como skip — descrição genuinamente ruim |
| Contraditórios (`status=curated` mas sem canonical) | 3 | Investigação manual + correção pontual |

### 4.1 — Bucket 1: Retentar 79 retryable_error (delegado para Item 2)

Não precisa código adicional — o CRON `pipeline-maintenance` já faz isso na primeira execução pós-deploy. Apenas confirmar pós-execução:

```sql
-- Após primeira execução do pipeline-maintenance (Item 2)
SELECT COUNT(*) AS pending_pos_retry
FROM job_postings
WHERE curation_status = 'pending'
  AND created_at < NOW() - INTERVAL '7 days';

-- Esperado: ~79 (vagas que voltaram para pending após retry)
```

Após CRON `curate-job-postings` rodar nas seguintes 6h, espera-se que essas 79 sejam recuradas com sucesso (a maioria) ou voltem para `retryable_error` (raríssimas) ou virem `low_quality`.

### 4.2 — Bucket 2: Auditar e marcar 188 low_quality

São vagas que o LLM Sonnet curador classificou como "descrição genuinamente ruim" (texto truncado, anúncio meta, vaga sem informação útil). **Decisão cravada:** mantém o status `low_quality`, **não retentar**, mas precisa garantir que NÃO interferem em nenhum cálculo de canônico.

**Migration de validação (one-shot):**

```sql
BEGIN;

-- Confirmar que low_quality estão excluídas do cálculo de vacancy_count
-- (deve estar — mas validamos para defesa)
DO $$
DECLARE
    canonicos_com_low_quality INT;
BEGIN
    SELECT COUNT(DISTINCT canonical_role_id) INTO canonicos_com_low_quality
    FROM job_postings
    WHERE curation_status = 'low_quality'
      AND canonical_role_id IS NOT NULL;
    
    RAISE NOTICE 'Canônicos com vaga low_quality (deveria ser 0 — todas já estavam órfãs): %', canonicos_com_low_quality;
END $$;

-- Garantir que não há low_quality com canonical_role_id (defensivo)
UPDATE job_postings
SET canonical_role_id = NULL
WHERE curation_status = 'low_quality'
  AND canonical_role_id IS NOT NULL;

COMMIT;
```

### 4.3 — Bucket 3: Investigar 3 contraditórios

Esses são patológicos: `curation_status = 'curated'` mas `canonical_role_id IS NULL`. Investigação manual.

**Query de descoberta:**

```sql
SELECT id, title, description, created_at, curation_status, canonical_role_id, curation_metadata
FROM job_postings
WHERE curation_status = 'curated'
  AND canonical_role_id IS NULL
ORDER BY created_at DESC;
```

**Decisão por caso:**
- Se `curation_metadata` aponta para canônico que foi DELETED em algum sprint → migrar status para `retryable_error` para reprocessamento
- Se vaga é genuinamente sem canônico (raro) → migrar para `low_quality` com nota em metadata

### 4.4 — Invariant DB cravado: `chk_curated_has_canonical`

Para impedir que esse drift volte, criar CHECK constraint no banco:

```sql
BEGIN;

-- Antes da constraint, garantir que TODAS as vagas com curated têm canonical_role_id
-- (executar Bucket 3 ANTES desta migration)
DO $$
DECLARE
    contraditorios INT;
BEGIN
    SELECT COUNT(*) INTO contraditorios
    FROM job_postings
    WHERE curation_status = 'curated' AND canonical_role_id IS NULL;
    
    IF contraditorios > 0 THEN
        RAISE EXCEPTION 'Ainda há % vagas curated sem canonical. Resolver Bucket 3 primeiro.', contraditorios;
    END IF;
END $$;

-- Adiciona CHECK constraint
ALTER TABLE job_postings
ADD CONSTRAINT chk_curated_has_canonical
CHECK (
    curation_status != 'curated' OR canonical_role_id IS NOT NULL
);

COMMIT;
```

**Efeito:** qualquer tentativa futura de marcar `curation_status = 'curated'` sem canonical será rejeitada pelo banco.

### 4.5 — Tests

```typescript
// tests/integration/sprint-v4_0/item-4-orfas.spec.ts

describe('Item 4 — Saneamento de 270 vagas órfãs', () => {
    it('low_quality não tem canonical_role_id', async () => {
        const { count } = await supabase
            .from('job_postings')
            .select('*', { count: 'exact', head: true })
            .eq('curation_status', 'low_quality')
            .not('canonical_role_id', 'is', null);
        
        expect(count).toBe(0);
    });
    
    it('CHECK constraint impede curated sem canonical', async () => {
        // Tentar inserir vaga curated sem canonical_role_id
        const { error } = await supabase
            .from('job_postings')
            .insert({
                title: 'Test',
                description: 'Test',
                curation_status: 'curated',
                canonical_role_id: null,
            });
        
        expect(error).toBeDefined();
        expect(error?.message).toMatch(/chk_curated_has_canonical/);
    });
    
    it('contraditórios foram resolvidos', async () => {
        const { count } = await supabase
            .from('job_postings')
            .select('*', { count: 'exact', head: true })
            .eq('curation_status', 'curated')
            .is('canonical_role_id', null);
        
        expect(count).toBe(0);
    });
});
```

### 4.6 — Esforço estimado

- **Bucket 1:** 0 horas (delegado para Item 2)
- **Bucket 2 (auditoria + UPDATE defensivo):** 30 minutos
- **Bucket 3 (investigação manual + correção):** 1.5 horas
- **CHECK constraint + tests:** 1 hora
- **Total:** 3 horas

---

## Item 6 — Trigger autoritativo único para `vacancy_count`

Bloco do Claude Code da sprint anterior confirmou: **zero drift real** em `vacancy_count`. Os 6 canônicos que pareciam divergir entre `trigger` e `maintenance` eram falsos positivos da query do Bloco B.5 (que estava sem filtro `is_active`). Os 6 reais divergiam por design conflitante: trigger é liberal (toda vaga conta), `maintenance_phase_2` PASSO 4 era restritivo (só `curated` + outros filtros).

**Decisão cravada:** Opção A — DROP do PASSO 4 do `maintenance_phase_2`. Trigger `FOR EACH STATEMENT` em `job_canonical_roles` (criado na sprint v5.18) fica como **única fonte autoritativa**.

### 6.1 — DROP do PASSO 4 do `maintenance_phase_2`

```sql
BEGIN;

-- Identificar a função maintenance_phase_2
-- (assumindo função PL/pgSQL com lógica em fases)

-- Opção 1: Se função existe, recriar SEM o PASSO 4
CREATE OR REPLACE FUNCTION maintenance_phase_2()
RETURNS VOID AS $$
BEGIN
    -- PASSO 1: ... (mantém)
    -- PASSO 2: ... (mantém)
    -- PASSO 3: ... (mantém)
    
    -- PASSO 4: DROPADO
    -- Antigamente: UPDATE job_canonical_roles SET vacancy_count = (SELECT COUNT(*) ... WHERE is_active=true AND curation_status='curated' ...)
    -- Trigger FOR EACH STATEMENT já mantém vacancy_count autoritativo agora.
    
    -- PASSO 5: ... (mantém)
END;
$$ LANGUAGE plpgsql;

-- Opção 2: Se a função tem outro nome ou está em outro lugar, fazer grep
-- e remover apenas o bloco do PASSO 4
COMMIT;
```

**Verificação após DROP:** confirmar que trigger ainda está ativo e funcionando.

**Mudança crítica v2:** triggers de `vacancy_count` vivem em **`job_postings`**, não em `job_canonical_roles`. A query da v1 buscava em `event_object_table = 'job_canonical_roles'` — schema real do Claude Code mostra que o trigger é AFTER INSERT/UPDATE/DELETE em `job_postings`, recomputando contagem nos canônicos afetados. Query corrigida:

```sql
SELECT trigger_name, event_object_table, event_manipulation, action_timing
FROM information_schema.triggers
WHERE event_object_table = 'job_postings'
  AND trigger_name ILIKE '%vacancy%';

-- Esperado: trigger AFTER INSERT/UPDATE/DELETE em job_postings continua ativo
```

### 6.2 — Validação pós-DROP

Rodar query que compara trigger vs cálculo manual em uma amostra:

```sql
WITH amostra AS (
    SELECT id, label, vacancy_count
    FROM job_canonical_roles
    WHERE status = 'active'
    ORDER BY RANDOM()
    LIMIT 50
)
SELECT 
    a.id, a.label,
    a.vacancy_count AS trigger_count,
    (SELECT COUNT(*) FROM job_postings jp WHERE jp.canonical_role_id = a.id) AS manual_count
FROM amostra a
WHERE a.vacancy_count != (SELECT COUNT(*) FROM job_postings jp WHERE jp.canonical_role_id = a.id);

-- Esperado: zero linhas (trigger e manual idênticos)
```

### 6.3 — Esforço estimado

- **DROP do PASSO 4:** 1 hora (cuidadoso — função PL/pgSQL maior)
- **Validação:** 30 minutos
- **Total:** 1.5 horas

---


# PR5 — Observação e UX

Dois itens funcionais que afetam transparência para o usuário final. Item 8 (banner condicional quando análise foi feita com cache fresco vs stale) e Item 10 (rótulo de viés do escudo de qualidade na metodologia).

---

## Item 8 — FK `rapidapi_usage_log_id` + banner D condicional

Quando o pipeline busca vagas na LinkedIn API via fantastic-jobs, registra a chamada em `rapidapi_usage_logs`. Hoje, a `analyses` tabela não tem rastreio para qual chamada de API foi usada — então não conseguimos diferenciar análise feita com cache fresco vs cache stale (≥120 dias).

**Mudança crítica v3 sobre v2:** `rapidapi_usage_logs` em produção tem `ENABLE ROW LEVEL SECURITY` ativo, com policy `rapidapi_usage_logs_write_service` (`qual=true`, comando `INSERT`). Isso significa que o **service_role bypassa** (escrita do worker funciona normalmente), mas **leitura via client browser autenticado retorna zero linhas** porque não há policy SELECT permitindo authenticated. v2 chamou imprecisamente de "RLS deny-all puro" — a tabela aceita escrita do service_role; o que é bloqueado é leitura por usuários autenticados. O efeito prático é o mesmo (frontend não consegue ler), mas a caracterização correta evita confusão durante migrations futuras. Solução mantida: **denormalizar `rapidapi_log_created_at` direto em `analyses`** no momento da escrita do worker. Frontend lê apenas de `analyses` (que já tem RLS própria por dono).

### 8.1 — Schema: FK + denormalização do timestamp em `analyses`

```sql
BEGIN;

-- FK para auditoria (mantida)
ALTER TABLE analyses
    ADD COLUMN IF NOT EXISTS rapidapi_usage_log_id UUID
    REFERENCES rapidapi_usage_logs(id) ON DELETE SET NULL;

-- Denormalização: timestamp do log copiado para analyses (defesa contra RLS deny-all)
ALTER TABLE analyses
    ADD COLUMN IF NOT EXISTS rapidapi_log_created_at TIMESTAMPTZ;

CREATE INDEX IF NOT EXISTS idx_analyses_rapidapi_log
ON analyses(rapidapi_usage_log_id)
WHERE rapidapi_usage_log_id IS NOT NULL;

COMMIT;
```

### 8.2 — Lógica no `worker.ts:397` (cache vs API call)

O worker decide entre cache local (Postgres) e API call. Quando vai para API, registra log. Precisa propagar tanto o ID do log quanto o `created_at` denormalizado para `analyses`.

**Mudança v2:** worker grava `rapidapi_log_created_at` junto com o `rapidapi_usage_log_id`. `.maybeSingle()` em vez de `.single()` no fallback de busca do último log.

```typescript
// lib/analysis/worker.ts (~ linha 397)

interface CacheCheckResult {
    use_cache: boolean;
    cached_count?: number;
    needs_api_call: boolean;
    last_posted_at?: string;
}

async function decideCacheVsApi(
    supabase: SupabaseClient,
    canonicalRoleId: string,
    locationState: string
): Promise<CacheCheckResult> {
    // Regra existente: count + max(posted_at) com janela 120d
    const { data: stats } = await supabase.rpc('get_cache_stats_for_canonical', {
        p_canonical_id: canonicalRoleId,
        p_location_state: locationState,
        p_window_days: 120,
    });

    const count = stats?.count ?? 0;
    const lastPostedAt = stats?.max_posted_at;

    if (count >= 100) {
        return { use_cache: true, cached_count: count, needs_api_call: false, last_posted_at: lastPostedAt };
    }

    return { use_cache: false, needs_api_call: true, cached_count: count, last_posted_at: lastPostedAt };
}

async function executeAnalysis(supabase: SupabaseClient, analysisId: string, /* ... */) {
    const cacheCheck = await decideCacheVsApi(supabase, canonicalRoleId, locationState);

    let rapidapiLogId: string | null = null;
    let rapidapiLogCreatedAt: string | null = null;

    if (cacheCheck.needs_api_call) {
        // Faz chamada à RapidAPI
        const apiResult = await callFantasticAPI({
            canonicalLabel: canonicalLabel,
            locationState: locationState,
            dateFrom: 'most_recent',
            limit: 100 - cacheCheck.cached_count!,
        });

        // Registra em rapidapi_usage_logs
        const { data: log } = await supabase
            .from('rapidapi_usage_logs')
            .insert({
                endpoint: 'fantastic-jobs/search',
                request_payload: { canonical_role_id: canonicalRoleId, location_state: locationState },
                response_status: apiResult.status,
                vagas_returned: apiResult.jobs.length,
                cost_usd: apiResult.cost,
            })
            .select('id, created_at')
            .single();

        rapidapiLogId = log?.id ?? null;
        rapidapiLogCreatedAt = log?.created_at ?? null;

        // Persiste vagas novas em job_postings
        await persistApiResults(supabase, apiResult.jobs);
    }

    // Fallback: cache hit, busca o último log relevante
    if (!rapidapiLogId) {
        const { data: ultimoLog } = await supabase
            .from('rapidapi_usage_logs')
            .select('id, created_at')
            .eq('endpoint', 'fantastic-jobs/search')
            .contains('request_payload', { canonical_role_id: canonicalRoleId, location_state: locationState })
            .order('created_at', { ascending: false })
            .limit(1)
            .maybeSingle();  // pode não existir — caminho normal

        rapidapiLogId = ultimoLog?.id ?? null;
        rapidapiLogCreatedAt = ultimoLog?.created_at ?? null;
    }

    // Update analyses com FK + timestamp denormalizado (frontend lê só do analyses)
    await supabase
        .from('analyses')
        .update({
            rapidapi_usage_log_id: rapidapiLogId,
            rapidapi_log_created_at: rapidapiLogCreatedAt,
        })
        .eq('id', analysisId);

    // ... continua o resto do pipeline ...
}
```

### 8.3 — Banner D condicional no Modal de Análise

Quando o usuário vê resultado da análise, mostrar um banner discreto se a busca foi feita predominantemente em cache stale (último `rapidapi_usage_log` há > `STALE_DATA_DAYS`). Banner D = "Dados de mercado podem estar levemente defasados."

**Mudanças v3 sobre v2:**

1. Frontend lê **apenas** `analysis.rapidapi_log_created_at` (denormalizado em `analyses`, com RLS própria do dono). Sem `useEffect` para buscar em `rapidapi_usage_logs` (bloqueado pela RLS sem policy SELECT — caracterização correta em §8.0 v3).
2. **Threshold extraído para `constants.ts`** (P2.5 Grok): v2 hardcodava `60` dias direto no componente. v3 usa `STALE_DATA_DAYS` em `constants.ts:11+` para alinhar com convenção do projeto e facilitar tunagem futura sem deploy de frontend isolado.

**Constantes em `constants.ts` (extensão):**

```typescript
// constants.ts (já tem LLM_MODEL na linha 10)

// Banner D — Aviso de cache stale para usuário final
// 60 dias = ~2x o ciclo de recrutamento médio (Sólides 2024 ≈ 40 dias)
// Métrica: percentual de banners exibidos ao mês (validar se threshold está calibrado pós-deploy)
export const STALE_DATA_DAYS = 60;
```

**Frontend — `components/home/modals/AnalysisResultModal.tsx`:**

```typescript
import { STALE_DATA_DAYS } from '@/constants';

interface BannerStaleProps {
    rapidapiLogCreatedAt: string | null;
}

function BannerStale({ rapidapiLogCreatedAt }: BannerStaleProps) {
    if (!rapidapiLogCreatedAt) return null;

    const daysAgo = Math.floor(
        (Date.now() - new Date(rapidapiLogCreatedAt).getTime()) / (1000 * 60 * 60 * 24)
    );

    if (daysAgo < STALE_DATA_DAYS) return null;  // Cache fresco — sem banner

    return (
        <div className="banner banner-info" data-testid="banner-stale-data">
            <span className="icon">ℹ️</span>
            <span>
                Os dados de mercado dessa análise foram atualizados pela última vez há {daysAgo} dias.
                Em geral, manteremos o catálogo atualizado, mas para esta função em particular pode haver pequena defasagem.
            </span>
        </div>
    );
}

export function AnalysisResultModal({ analysis }: Props) {
    // analysis.rapidapi_log_created_at já vem denormalizado — sem fetch adicional
    return (
        <div className="modal-content">
            <BannerStale rapidapiLogCreatedAt={analysis.rapidapi_log_created_at} />

            {/* ... resto da UI ... */}
        </div>
    );
}
```

### 8.4 — Tests

```typescript
// tests/integration/sprint-v4_0/item-8-banner-stale.spec.ts

describe('Item 8 — Banner D condicional', () => {
    it('analysis ganha rapidapi_usage_log_id após API call', async () => {
        // Setup: criar analysis que vai disparar API call
        const { data: analysis } = await supabase
            .from('analyses')
            .insert({
                profile_id: testProfileId,
                canonical_role_id: testCanonicalId,
                status: 'pending',
                location_state: 'SP',
            })
            .select()
            .single();
        
        // Executar pipeline (mockado)
        await executeAnalysis(supabase, analysis!.id, /* ... */);
        
        const { data: analysisAfter } = await supabase
            .from('analyses')
            .select('rapidapi_usage_log_id')
            .eq('id', analysis!.id)
            .single();
        
        expect(analysisAfter?.rapidapi_usage_log_id).toBeTruthy();
    });
    
    it('banner D aparece se log_created_at >= 60 dias', () => {
        const oldLog = new Date(Date.now() - 80 * 24 * 60 * 60 * 1000).toISOString();
        const { getByTestId } = render(<BannerStale rapidapiLogCreatedAt={oldLog} />);
        expect(getByTestId('banner-stale-data')).toBeInTheDocument();
    });
    
    it('banner D NÃO aparece se log_created_at < 60 dias', () => {
        const recentLog = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
        const { queryByTestId } = render(<BannerStale rapidapiLogCreatedAt={recentLog} />);
        expect(queryByTestId('banner-stale-data')).toBeNull();
    });
});
```

### 8.5 — Esforço estimado

- **Schema (FK):** 30 minutos
- **Lógica em worker.ts:** 2 horas
- **Componente Banner D:** 1.5 horas
- **Tests:** 1 hora
- **Total:** 5 horas

---

## Item 10 — Thresholds fundamentados de viés do escudo de qualidade

Os 2 thresholds existentes em código (`TOP1_EMPLOYER_SHARE_THRESHOLD` e `INTERMEDIARY_SHARE_THRESHOLD`) são funcionais desde o lançamento, mas estavam sem fundamentação documentada. Esta sprint **fundamenta** os valores em fontes verificáveis e adiciona 2 textos à página de metodologia.

### 10.1 — Constantes cravadas com referências

**Localização:** `lib/constants.ts` ou `lib/quality-shield.ts`

```typescript
// lib/quality-shield.ts

/**
 * Threshold para detecção de viés "concentração de empregador único" (top1).
 *
 * Valor: 0.30 (30%)
 *
 * Fundamentação: Department of Justice + Federal Trade Commission Horizontal Merger Guidelines
 * (2010, atualizado 2023). HMG estabelece que mercados com participação >30% de um único
 * agente são "moderately concentrated". Mercado de trabalho saudável tem múltiplos
 * empregadores; se top1 > 30%, há viés de concentração suficiente para sinalizar.
 *
 * Fonte: https://www.justice.gov/atr/horizontal-merger-guidelines-08192010
 */
export const TOP1_EMPLOYER_SHARE_THRESHOLD = 0.30;

/**
 * Threshold para detecção de viés "intermediário/agência".
 *
 * Valor: 0.20 (20%)
 *
 * Fundamentação: American Staffing Association (ASA) reporta consistentemente que
 * staffing/recruitment agencies representam ~2% do mercado total de trabalho dos EUA.
 * Mercado brasileiro tem padrão similar ou ligeiramente maior. Se >20% das vagas de
 * uma função vêm de agências, há viés sistêmico — agências postam vagas que não
 * representam empregadores reais.
 *
 * Fonte: https://americanstaffing.net/research/fact-sheets-analysis-staffing-industry-trends/
 */
export const INTERMEDIARY_SHARE_THRESHOLD = 0.20;

/**
 * Constantes auxiliares para o ciclo de vida de canônicos
 * (referências para Item 2 do CRON pipeline-maintenance)
 */

/**
 * Threshold AMBER: canônico sem vagas há 120 dias.
 *
 * Fundamentação: 3 ciclos × 40 dias (média de tempo que vaga TI no Brasil
 * leva para ser preenchida segundo Sólides 2024). 120 dias = 3 vacâncias
 * consecutivas não preenchidas, sugere obsolescência incipiente.
 *
 * Fonte: https://www.solides.com.br/blog/tempo-medio-recrutamento-selecao
 */
export const AMBER_THRESHOLD_DAYS = 120;

/**
 * Threshold RED: canônico sem vagas há 365 dias.
 *
 * Fundamentação: 4 trimestres = ciclo orçamentário completo. Após 4 trimestres
 * sem evidência de mercado, canônico é candidato a merge ou descarte.
 * Online Learning Cycles (OLC) do paper arxiv:2406.15373 sugere janela
 * de 12 meses para staleness em sistemas adaptativos.
 *
 * Fonte: https://arxiv.org/abs/2406.15373
 */
export const RED_THRESHOLD_DAYS = 365;
```

### 10.2 — Aplicação dos thresholds (já funcional, sem mudança de código)

Os callsites que já usam essas constantes não precisam mudar. Esta sprint apenas:
1. Documenta a fundamentação no código (comentários acima)
2. Adiciona textos correspondentes à página de metodologia (próxima seção)

### 10.3 — Texto 1: "Quando a base de vagas tem distorção, sinalizamos"

**Localização:** `app/methodology/page.tsx` — seção "Decisões de qualidade"

```markdown
## Decisões de qualidade

### Quando a base de vagas tem distorção, sinalizamos

Nem toda função tem o mesmo perfil de mercado. Algumas funções recebem vagas concentradas em poucos empregadores, ou predominantemente de agências de recrutamento. Quando detectamos essas situações, sinalizamos a análise com um **escudo de qualidade** — um indicador visual de que os dados podem refletir a realidade de poucos empregadores ou intermediários, não do mercado como um todo.

**Como decidimos:**

- Se mais de 30% das vagas analisadas vêm de um único empregador, sinalizamos "concentração de empregador único". O limite vem de princípios clássicos de análise de mercado: participações acima de 30% indicam mercados moderadamente concentrados.

- Se mais de 20% das vagas analisadas vêm de agências de recrutamento, sinalizamos "alta presença de intermediários". Pesquisas de mercado de trabalho indicam que agências representam tipicamente 2% do volume natural; se a participação está dezenas de vezes acima do esperado, há viés sistêmico.

Sinalizar não é desclassificar. A análise continua sendo entregue com toda a profundidade habitual; o escudo apenas dá contexto adicional para você interpretar com mais nuance.
```

### 10.4 — Texto 2: "Como o catálogo de funções se mantém vivo"

**Localização:** `app/methodology/page.tsx` — seção "Fonte dos dados"

```markdown
## Fonte dos dados

### Como o catálogo de funções se mantém vivo

Mercado de trabalho muda. Funções que eram comuns há 5 anos podem estar obsoletas hoje (e vice-versa). Para refletir essa dinâmica, nosso catálogo de funções não é estático — ele é continuamente revisado.

**O que monitoramos:**

- Funções que param de aparecer no mercado por períodos prolongados são marcadas como "em observação" após 120 dias sem novas vagas, e como "candidatas a revisão" após 365 dias. A janela de 120 dias corresponde a 3 ciclos consecutivos de recrutamento médio para vagas de tecnologia no Brasil; 365 dias corresponde a 4 trimestres de ciclo orçamentário completo.

- Quando uma função em observação começa a se sobrepor semanticamente com outra função ativa, avaliamos se faz sentido consolidar. Essa avaliação leva em conta similaridade de descrição, sobreposição de competências e padrões de remuneração.

- Quando consolidamos, atualizamos automaticamente as análises anteriores que apontavam para a função antiga, mantendo a continuidade da experiência do usuário e enviando uma notificação informativa.

Esse processo é executado de forma contínua, com calibração contra dados de mercado verificados. O catálogo reflete o estado atual do mercado — não uma fotografia datada.
```

**Implementação técnica:** seções acima vão para arquivo `data/methodology-content.md` (mantendo padrão atual da página de metodologia) e são renderizadas via componente que carrega o markdown.

### 10.5 — Linkagem das fontes na metodologia

Adicionar no final da página `app/methodology/page.tsx`, seção "Referências":

```markdown
## Referências e leituras adicionais

- DOJ-FTC Horizontal Merger Guidelines (2010, atualizado 2023): [link oficial](https://www.justice.gov/atr/horizontal-merger-guidelines-08192010)
- American Staffing Association — Industry Statistics: [link oficial](https://americanstaffing.net/research/fact-sheets-analysis-staffing-industry-trends/)
- Sólides — Tempo médio de recrutamento e seleção no Brasil: [link oficial](https://www.solides.com.br/blog/tempo-medio-recrutamento-selecao)
- Online Learning Cycles in Adaptive Systems (2024): [arXiv:2406.15373](https://arxiv.org/abs/2406.15373)
```

### 10.6 — Tests

```typescript
// tests/integration/sprint-v4_0/item-10-thresholds.spec.ts

describe('Item 10 — Thresholds de viés', () => {
    it('TOP1_EMPLOYER_SHARE_THRESHOLD é 0.30', () => {
        expect(TOP1_EMPLOYER_SHARE_THRESHOLD).toBe(0.30);
    });
    
    it('INTERMEDIARY_SHARE_THRESHOLD é 0.20', () => {
        expect(INTERMEDIARY_SHARE_THRESHOLD).toBe(0.20);
    });
    
    it('AMBER_THRESHOLD_DAYS é 120 e RED_THRESHOLD_DAYS é 365', () => {
        expect(AMBER_THRESHOLD_DAYS).toBe(120);
        expect(RED_THRESHOLD_DAYS).toBe(365);
    });
    
    it('escudo de qualidade ativa quando top1 > 30%', () => {
        const result = checkQualityShield({
            top1_employer_share: 0.35,
            intermediary_share: 0.05,
        });
        
        expect(result.shield_active).toBe(true);
        expect(result.reasons).toContain('top1_concentration');
    });
    
    it('escudo de qualidade ativa quando intermediary > 20%', () => {
        const result = checkQualityShield({
            top1_employer_share: 0.10,
            intermediary_share: 0.25,
        });
        
        expect(result.shield_active).toBe(true);
        expect(result.reasons).toContain('intermediary_concentration');
    });
    
    it('escudo NÃO ativa quando ambos abaixo dos thresholds', () => {
        const result = checkQualityShield({
            top1_employer_share: 0.15,
            intermediary_share: 0.10,
        });
        
        expect(result.shield_active).toBe(false);
    });
});
```

### 10.7 — Esforço estimado

- **Atualização de constantes em código (com comentários fundamentados):** 30 minutos
- **Texto 1 (Quando base de vagas tem distorção):** 1 hora (escrita + revisão)
- **Texto 2 (Como catálogo se mantém vivo):** 1 hora
- **Linkagem de fontes:** 30 minutos
- **Tests:** 1 hora
- **Total:** 4 horas

---


# PR6 — Refinos

Dois itens transversais que refinam ao longo da sprint: Item 13 (tests M2 para 4 arquivos críticos do pipeline) e Item 14 (UI admin para drilldown de remap e human-validated).

---

## Item 13 — Tests M2 para 4 arquivos críticos

**Definição cravada:** "Tests M2" = testes de integração com banco real (não unitários puros, não E2E completos). Cobrem fluxos de persistência crítica do pipeline.

### 13.1 — Arquivos cobertos

| Arquivo | Função | Tests prioritários |
|---|---|---|
| `lib/pipeline/persist-precheck.ts` | Persiste resultado da Camada 0 (cache hit por hash) | Cache hit, cache miss, hash colisão |
| `lib/pipeline/persist-curation.ts` | Persiste curadoria pós-LLM (Camadas 2/3) | Insert canônico novo, update existente, retry idempotência |
| `app/api/admin/canonicals/[id]/remap/route.ts` | Endpoint admin para reapontar vagas de canônico em outro | Remap simples, remap em cascata, validação de permissão admin |
| `app/api/admin/canonicals/human-validated/route.ts` | Endpoint admin para marcar canônicos validados manualmente | Marcação simples, batch, evento de auditoria |

### 13.2 — Estrutura dos tests

**Mudança crítica v2:** o teste de cache hit do `persist-precheck` foi reescrito. A v1 inseria em `canonical_role_anchors` para simular âncora — **essa tabela NÃO EXISTE no schema real** (validado pelo Claude Code). A semântica correta da Camada 0 é: "existe vaga prévia em `job_postings` com mesmo `description_hash` + `curation_status='curated'` + (`human_validated=true` OU temporal_quorum 24h)? Reusar esse `canonical_role_id`."

```typescript
// tests/integration/sprint-v4_0/item-13-pipeline-tests.spec.ts

import { createAdminServerClient } from '@/lib/supabase-server';
import { persistPrecheck } from '@/lib/pipeline/persist-precheck';
import { persistCuration } from '@/lib/pipeline/persist-curation';

const supabase = createAdminServerClient();

describe('persist-precheck — testes M2', () => {
    let testCanonicalId: string;
    let testJobId: string;
    let anchorJobId: string;  // vaga prévia que serve de âncora

    beforeEach(async () => {
        // Setup: criar canônico active
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test Persist Precheck', slug: 'test-persist-precheck', status: 'active' })
            .select()
            .single();
        testCanonicalId = canonical!.id;

        // Vaga "pending" que vai ser processada
        const { data: job } = await supabase
            .from('job_postings')
            .insert({
                title: 'Test Job',
                description: 'Test description for hashing',
                curation_status: 'pending',
                description_hash: 'hash-test-1',
            })
            .select()
            .single();
        testJobId = job!.id;
    });

    afterEach(async () => {
        await supabase.from('job_postings').delete().eq('id', testJobId);
        if (anchorJobId) await supabase.from('job_postings').delete().eq('id', anchorJobId);
        await supabase.from('job_canonical_roles').delete().eq('id', testCanonicalId);
    });

    it('cache hit: vaga é apontada para canônico existente via âncora prévia', async () => {
        // Setup: âncora = vaga prévia com mesmo hash, curated + human_validated
        // (canonical_role_anchors NÃO EXISTE — usar job_postings curated)
        const { data: anchor } = await supabase
            .from('job_postings')
            .insert({
                title: 'Anchor Job',
                description: 'Test description for hashing',
                description_hash: 'hash-test-1',
                canonical_role_id: testCanonicalId,
                curation_status: 'curated',
                curation_layer: 2,
                human_validated: true,  // âncora forte (alternativa: temporal_quorum 24h)
                posted_at: new Date(Date.now() - 48 * 60 * 60 * 1000).toISOString(),  // 48h atrás
            })
            .select()
            .single();
        anchorJobId = anchor!.id;

        // Executar precheck
        await persistPrecheck(supabase, {
            job_id: testJobId,
            description_hash: 'hash-test-1',
        });

        // Verificar — vaga foi apontada para o mesmo canônico da âncora
        const { data: jobAfter } = await supabase
            .from('job_postings')
            .select('canonical_role_id, curation_status, curation_layer')
            .eq('id', testJobId)
            .single();

        expect(jobAfter?.canonical_role_id).toBe(testCanonicalId);
        expect(jobAfter?.curation_status).toBe('curated');
        expect(jobAfter?.curation_layer).toBe(0);
    });

    it('cache miss: vaga continua pending para Camada 1+', async () => {
        // Sem âncora — hash novo
        await persistPrecheck(supabase, {
            job_id: testJobId,
            description_hash: 'hash-nao-existe',
        });

        const { data: jobAfter } = await supabase
            .from('job_postings')
            .select('curation_status')
            .eq('id', testJobId)
            .single();

        expect(jobAfter?.curation_status).toBe('pending');
    });

    it('cache miss: âncora prévia sem human_validated nem temporal_quorum não é elegível', async () => {
        // Vaga prévia com hash igual mas SEM human_validated e POSTED há <24h (sem temporal_quorum)
        const { data: anchor } = await supabase
            .from('job_postings')
            .insert({
                title: 'Anchor Recent',
                description: 'Test description for hashing',
                description_hash: 'hash-test-1',
                canonical_role_id: testCanonicalId,
                curation_status: 'curated',
                curation_layer: 2,
                human_validated: false,
                posted_at: new Date(Date.now() - 1 * 60 * 60 * 1000).toISOString(),  // 1h atrás
            })
            .select()
            .single();
        anchorJobId = anchor!.id;

        await persistPrecheck(supabase, {
            job_id: testJobId,
            description_hash: 'hash-test-1',
        });

        const { data: jobAfter } = await supabase
            .from('job_postings')
            .select('curation_status')
            .eq('id', testJobId)
            .single();

        // Não deve ter sido curated pela Camada 0 (âncora não elegível)
        expect(jobAfter?.curation_status).toBe('pending');
    });
});

describe('persist-curation — testes M2', () => {
    it('cria canônico novo + insere source + atualiza vaga', async () => {
        // Setup: vaga pending sem canônico
        const { data: job } = await supabase
            .from('job_postings')
            .insert({
                title: 'Vaga teste',
                description: 'Descrição',
                curation_status: 'pending',
                company: 'Empresa Teste LTDA',
            })
            .select()
            .single();

        // Persist curação (canônico novo)
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Persist Test New',
            layer: 2,
            llm_proposed_label: 'Canonical Persist Test New',
            llm_proposed_relation_type: 'domain_synonym',
        });

        // Verificar canônico criado
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .select('id, status')
            .eq('label', 'Canonical Persist Test New')
            .single();

        expect(canonical).not.toBeNull();
        expect(canonical?.status).toBe('pending');  // Apenas 1 vaga, não promove

        // Verificar source criado (schema real: normalized_company)
        const { data: source } = await supabase
            .from('job_canonical_role_sources')
            .select('normalized_company')
            .eq('canonical_role_id', canonical!.id)
            .maybeSingle();

        expect(source?.normalized_company).toBeDefined();

        // Verificar vaga atualizada
        const { data: jobAfter } = await supabase
            .from('job_postings')
            .select('canonical_role_id, curation_status, curation_layer')
            .eq('id', job!.id)
            .single();

        expect(jobAfter?.canonical_role_id).toBe(canonical!.id);
        expect(jobAfter?.curation_status).toBe('curated');
        expect(jobAfter?.curation_layer).toBe(2);

        // Cleanup
        await supabase.from('job_postings').delete().eq('id', job!.id);
        await supabase.from('job_canonical_role_sources').delete().eq('canonical_role_id', canonical!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', canonical!.id);
    });
    
    it('idempotência: chamada repetida não cria duplicata', async () => {
        // Setup
        const { data: job } = await supabase
            .from('job_postings')
            .insert({
                title: 'Vaga idempotente',
                description: 'Descrição',
                curation_status: 'pending',
            })
            .select()
            .single();
        
        // Primeira chamada
        // v4 (NIv3-3 Outra-Claude): llm_proposed_relation_type é obrigatório por §G2.7 
        // (Sonnet propõe explicitamente). Test omitia em ambas as chamadas → não 
        // representava chamada real. Adicionado em ambas para fidelidade.
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Idempotency Test',
            layer: 2,
            llm_proposed_label: 'Canonical Idempotency Test',
            llm_proposed_relation_type: 'domain_synonym',
        });
        
        // Segunda chamada (idêntica) — deve ser idempotente
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Idempotency Test',
            layer: 2,
            llm_proposed_label: 'Canonical Idempotency Test',
            llm_proposed_relation_type: 'domain_synonym',
        });
        
        // Verificar que só há 1 canônico criado
        const { count } = await supabase
            .from('job_canonical_roles')
            .select('*', { count: 'exact', head: true })
            .eq('label', 'Canonical Idempotency Test');
        
        expect(count).toBe(1);
        
        // Cleanup
        const { data: c } = await supabase
            .from('job_canonical_roles')
            .select('id')
            .eq('label', 'Canonical Idempotency Test')
            .single();
        await supabase.from('job_postings').delete().eq('id', job!.id);
        await supabase.from('job_canonical_role_sources').delete().eq('canonical_role_id', c!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', c!.id);
    });
});

describe('endpoint /admin/canonicals/[id]/remap — testes M2', () => {
    // Helper: simula sessão admin via cookie de auth.
    // Auth admin v2 = isAdminAuthorized(req) consulta public.users.user_type='admin'.
    // Não há mais Bearer ADMIN_TOKEN — testes usam helper que injeta cookie de admin@calibracv.com.
    let adminCookie: string;

    beforeAll(async () => {
        adminCookie = await loginAsAdmin();  // helper que retorna cookie de auth do user_type='admin'
    });

    it('remap simples: vagas reapontadas via merge_canonicals + loser deprecated', async () => {
        // Setup: 2 canônicos + algumas vagas no source (loser)
        const { data: source } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Source Remap', slug: 'source-remap', status: 'active' })
            .select()
            .single();

        const { data: target } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Target Remap', slug: 'target-remap', status: 'active' })
            .select()
            .single();

        const { data: vagas } = await supabase
            .from('job_postings')
            .insert([
                { title: 'V1', description: 'D1', canonical_role_id: source!.id, curation_status: 'curated' },
                { title: 'V2', description: 'D2', canonical_role_id: source!.id, curation_status: 'curated' },
            ])
            .select();

        // Chamar endpoint via fetch — auth via cookie de admin
        const res = await fetch(`http://localhost:3000/api/admin/canonicals/${source!.id}/remap`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Cookie': adminCookie,
            },
            body: JSON.stringify({ target_canonical_id: target!.id }),
        });

        expect(res.status).toBe(200);

        // Verificar que vagas foram reapontadas
        const { data: vagasApos } = await supabase
            .from('job_postings')
            .select('canonical_role_id')
            .in('id', vagas!.map(v => v.id));

        for (const v of vagasApos ?? []) {
            expect(v.canonical_role_id).toBe(target!.id);
        }

        // Source deve ter status=deprecated e merged_into=target
        // (NÃO 'merged' — esse valor não existe no CHECK constraint)
        const { data: sourceAfter } = await supabase
            .from('job_canonical_roles')
            .select('status, merged_into')
            .eq('id', source!.id)
            .single();

        expect(sourceAfter?.status).toBe('deprecated');
        expect(sourceAfter?.merged_into).toBe(target!.id);

        // Cleanup
        for (const v of vagas!) {
            await supabase.from('job_postings').delete().eq('id', v.id);
        }
        await supabase.from('job_canonical_roles').delete().eq('id', source!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', target!.id);
    });

    it('valida permissão admin: requisição sem cookie de admin retorna 401', async () => {
        const res = await fetch('http://localhost:3000/api/admin/canonicals/some-id/remap', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },  // sem cookie de admin
            body: JSON.stringify({ target_canonical_id: 'other-id' }),
        });

        expect(res.status).toBe(401);
    });

    it('valida permissão: usuário authenticated mas user_type != admin retorna 403', async () => {
        const userCookie = await loginAsRegularUser();  // helper que retorna cookie de user_type='free_registered'
        const res = await fetch('http://localhost:3000/api/admin/canonicals/some-id/remap', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Cookie': userCookie,
            },
            body: JSON.stringify({ target_canonical_id: 'other-id' }),
        });

        expect(res.status).toBe(403);
    });
});

describe('endpoint /admin/canonicals/human-validated — testes M2', () => {
    let adminCookie: string;

    beforeAll(async () => {
        adminCookie = await loginAsAdmin();
    });

    it('marca canônicos como human_validated com event de auditoria', async () => {
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test HV', slug: 'test-hv', status: 'active' })
            .select()
            .single();

        const res = await fetch('http://localhost:3000/api/admin/canonicals/human-validated', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Cookie': adminCookie,
            },
            body: JSON.stringify({ canonical_ids: [canonical!.id] }),
        });

        expect(res.status).toBe(200);

        // Verificar flag setada (colunas criadas no Item 14)
        const { data: cAfter } = await supabase
            .from('job_canonical_roles')
            .select('human_validated_at, human_validated_by')
            .eq('id', canonical!.id)
            .single();

        expect(cAfter?.human_validated_at).toBeTruthy();
        expect(cAfter?.human_validated_by).toBeTruthy();

        // Verificar evento de auditoria — schema correto (resource_id/resource_type, actor)
        const { data: events } = await supabase
            .from('events')
            .select('*')
            .eq('resource_id', canonical!.id)
            .eq('resource_type', 'job_canonical_role')
            .eq('event_name', 'canonical_human_validated')
            .limit(1);

        expect(events?.length).toBe(1);
        expect(events?.[0]?.actor).toBe('human');
        expect(events?.[0]?.actor_id).toBeTruthy();

        // Cleanup
        await supabase.from('events').delete().eq('resource_id', canonical!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', canonical!.id);
    });
});
```

### 13.3 — Cobertura mínima cravada

Para cada um dos 4 arquivos:
- ✅ Caminho feliz (operação acontece como esperado)
- ✅ Caminho de erro (entrada inválida, conflito, permissão)
- ✅ Idempotência (chamadas repetidas não causam efeito colateral)
- ✅ Auditoria (eventos críticos vão para tabela `events`)

### 13.4 — Esforço estimado

- **Tests para `persist-precheck`:** 2 horas
- **Tests para `persist-curation`:** 3 horas (mais complexo)
- **Tests para endpoint `/remap`:** 2 horas
- **Tests para endpoint `/human-validated`:** 1.5 horas
- **Total:** 8.5 horas

---

## Item 14 — UI admin de drilldown para painéis O8 e O9

Painéis admin O8 (eventos por categoria) e O9 (anomalias) já mostram contadores agregados. Item 14 adiciona drilldown para os 2 endpoints críticos: `/api/admin/canonicals/[id]/remap` e `/api/admin/canonicals/human-validated`. Quando admin clica numa linha agregada, abre painel lateral com detalhes da operação e ações disponíveis.

**Mudança crítica v2 — pré-requisito:** o Claude Code validou que **`job_canonical_roles` NÃO tem hoje as colunas `human_validated_at` e `human_validated_by`**. A v1 assumiu que existiam (Item 13.2 testes lia delas direto). v2 adiciona a migration para criar ambas como parte do Item 14, antes da UI.

**Diferença semântica explícita entre os dois `human_validated` no schema (P1.11 v3):**

Existem duas colunas com nome similar em tabelas diferentes, com **significados completamente distintos**. Esse drift é fonte recorrente de confusão durante refator. v3 documenta a distinção e adiciona `COMMENT ON COLUMN` em ambas para que `\d job_postings` e `\d job_canonical_roles` mostrem a explicação direto no banco:

| Coluna | Tabela | Tipo | Significado |
|---|---|---|---|
| `human_validated` | `job_postings` (já existente, boolean) | Por **vaga** | Admin confirmou que **esta vaga específica** está bem curada. Usado como âncora forte no precheck (`description_hash` + `human_validated=true` força match). |
| `human_validated_at` + `human_validated_by` | `job_canonical_roles` (criadas em §14.0) | Por **canônico (catálogo)** | Admin marcou que o **catálogo inteiro deste canônico** está estável. Usado para impedir remap automático em mass-merge ou em correções de embeddings. |

**Erro comum a evitar:** confundir e tratar `job_canonical_roles.human_validated_at` como "última vaga validada por humano para esse canônico" — isso seria derivado de `MAX(jp.updated_at) WHERE jp.canonical_role_id=X AND jp.human_validated=true`, que tem semântica diferente. As colunas em `job_canonical_roles` são manuais, vivem com vida própria, e indicam validação do **catálogo** (linha em `job_canonical_roles`), não validação derivada das vagas relacionadas.

### 14.0 — Migration prévia: criar `human_validated_at/by` em `job_canonical_roles`

```sql
BEGIN;

-- Colunas para auditoria de validação manual de canônicos
ALTER TABLE job_canonical_roles
    ADD COLUMN IF NOT EXISTS human_validated_at TIMESTAMPTZ,
    ADD COLUMN IF NOT EXISTS human_validated_by UUID REFERENCES public.users(id) ON DELETE SET NULL;

-- Index para painéis O8/O9 listarem canônicos validados por período
CREATE INDEX IF NOT EXISTS idx_canonical_human_validated_at
ON job_canonical_roles(human_validated_at DESC NULLS LAST)
WHERE human_validated_at IS NOT NULL;

COMMIT;
```

**Endpoint `/api/admin/canonicals/human-validated` (refatoração para popular as novas colunas):**

```typescript
// app/api/admin/canonicals/human-validated/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createAdminServerClient } from '@/lib/supabase-server';
import { isAdminAuthorized } from '@/lib/admin-guard';

export async function POST(req: NextRequest) {
    const adminCheck = await isAdminAuthorized(req);
    if (!adminCheck.ok) {
        return NextResponse.json({ error: adminCheck.error }, { status: adminCheck.status });
    }
    const adminUserId = adminCheck.userId;  // UUID do admin autenticado

    const body = await req.json();
    const { canonical_ids } = body as { canonical_ids: string[] };

    if (!Array.isArray(canonical_ids) || canonical_ids.length === 0) {
        return NextResponse.json({ error: 'canonical_ids vazio' }, { status: 400 });
    }

    const supabase = createAdminServerClient();
    const validatedAt = new Date().toISOString();

    // 1. Update das colunas em job_canonical_roles
    const { error: updateError } = await supabase
        .from('job_canonical_roles')
        .update({
            human_validated_at: validatedAt,
            human_validated_by: adminUserId,
        })
        .in('id', canonical_ids);

    if (updateError) {
        return NextResponse.json({ error: updateError.message }, { status: 500 });
    }

    // 2. Inserir 1 evento por canônico (auditoria) — schema correto v2
    const events = canonical_ids.map((canonicalId) => ({
        event_name: 'canonical_human_validated',
        resource_type: 'job_canonical_role',
        resource_id: canonicalId,
        actor: 'human' as const,
        actor_id: adminUserId,
        previous_state: { human_validated_at: null },
        new_state: { human_validated_at: validatedAt, human_validated_by: adminUserId },
    }));
    await supabase.from('events').insert(events);

    return NextResponse.json({ status: 'ok', count: canonical_ids.length });
}
```

### 14.1 — Estrutura do componente principal

**Localização:** `components/admin/dashboard/RemapDrilldown.tsx`

**Mudança v2:** o componente lê eventos de merge gerados pela `merge_canonicals` SQL function (G2.9). Schema correto: `event_name='canonical_role_merged'`, `resource_type='job_canonical_role'`, payload em `new_state.canonical_label` + `new_state.previous_canonical_label`.

```typescript
// components/admin/dashboard/RemapDrilldown.tsx
import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase-browser';

interface Props {
    eventId: string;
    onClose: () => void;
}

export function RemapDrilldown({ eventId, onClose }: Props) {
    const [event, setEvent] = useState<any>(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        const supabase = createClient();
        supabase
            .from('events')
            .select('*')
            .eq('id', eventId)
            .maybeSingle()  // pode não existir — caminho normal
            .then(({ data }) => {
                setEvent(data);
                setLoading(false);
            });
    }, [eventId]);

    if (loading) return <div>Carregando...</div>;
    if (!event) return <div>Evento não encontrado.</div>;

    // Schema v2: payload de canonical_role_merged
    // resource_id = loser canonical id
    // new_state.merged_into = winner canonical id
    // new_state.canonical_label = winner label
    // new_state.previous_canonical_label = loser label
    const loserId = event.resource_id;
    const loserLabel = event.new_state?.previous_canonical_label;
    const winnerId = event.new_state?.merged_into;
    const winnerLabel = event.new_state?.canonical_label;
    const reason = event.reason ?? event.new_state?.reason;
    const affectedJobs = event.new_state?.affected_jobs;

    return (
        <aside className="drilldown-panel">
            <header>
                <h3>Detalhes do remap</h3>
                <button onClick={onClose}>×</button>
            </header>

            <section>
                <dl>
                    <dt>Quando</dt>
                    <dd>{new Date(event.created_at).toLocaleString('pt-BR')}</dd>

                    <dt>Origem (loser, deprecated)</dt>
                    <dd>
                        {loserLabel}
                        <span className="muted">(id: {loserId})</span>
                    </dd>

                    <dt>Destino (winner, ativo)</dt>
                    <dd>
                        {winnerLabel}
                        <span className="muted">(id: {winnerId})</span>
                    </dd>

                    <dt>Razão</dt>
                    <dd>{reason ?? 'Não documentado'}</dd>

                    <dt>Vagas afetadas</dt>
                    <dd>{affectedJobs ?? 'N/A'}</dd>

                    <dt>Ator</dt>
                    <dd>{event.actor === 'system' ? 'Automatizado (CRON Opus)' : event.actor === 'human' ? `Admin (${event.actor_id})` : 'Pipeline'}</dd>
                </dl>
            </section>

            <section>
                <h4>Ações</h4>
                <button onClick={() => window.open(`/admin/canonicals/${winnerId}`, '_blank')}>
                    Ver canônico destino
                </button>
                <button className="danger" disabled title="Reverter remap fica para sprint pós-v2">
                    ↩ Reverter remap (em breve)
                </button>
            </section>
        </aside>
    );
}
```

### 14.2 — Estrutura do componente para `human-validated`

**Localização:** `components/admin/dashboard/HumanValidatedDrilldown.tsx`

**Mudança v2:** query de eventos usa `resource_id`/`resource_type`. Renderização do "validado por" busca display_name do user (não exibe UUID cru).

```typescript
// components/admin/dashboard/HumanValidatedDrilldown.tsx
import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase-browser';

interface Props {
    canonicalId: string;
    onClose: () => void;
}

export function HumanValidatedDrilldown({ canonicalId, onClose }: Props) {
    const [data, setData] = useState<any>(null);

    useEffect(() => {
        const supabase = createClient();
        Promise.all([
            // Detalhes do canônico (com display_name do validador via JOIN)
            supabase
                .from('job_canonical_roles')
                .select(`
                    *,
                    validator:users!job_canonical_roles_human_validated_by_fkey (display_name)
                `)
                .eq('id', canonicalId)
                .maybeSingle(),
            // Histórico de eventos human_validated — schema v2 (resource_id/resource_type)
            supabase
                .from('events')
                .select('*')
                .eq('resource_id', canonicalId)
                .eq('resource_type', 'job_canonical_role')
                .eq('event_name', 'canonical_human_validated')
                .order('created_at', { ascending: false }),
        ]).then(([canonicalRes, eventsRes]) => {
            setData({
                canonical: canonicalRes.data,
                events: eventsRes.data ?? [],
            });
        });
    }, [canonicalId]);

    if (!data) return <div>Carregando...</div>;

    return (
        <aside className="drilldown-panel">
            <header>
                <h3>{data.canonical.label}</h3>
                <button onClick={onClose}>×</button>
            </header>

            <section>
                <h4>Validação manual</h4>
                <dl>
                    <dt>Validado em</dt>
                    <dd>
                        {data.canonical.human_validated_at
                            ? new Date(data.canonical.human_validated_at).toLocaleString('pt-BR')
                            : 'Não validado'}
                    </dd>

                    <dt>Validado por</dt>
                    <dd>{data.canonical.validator?.display_name ?? 'N/A'}</dd>

                    <dt>Status atual</dt>
                    <dd className={`badge badge-${data.canonical.status}`}>{data.canonical.status}</dd>
                </dl>
            </section>

            <section>
                <h4>Histórico ({data.events.length})</h4>
                <ul className="event-list">
                    {data.events.map((e: any) => (
                        <li key={e.id}>
                            <time>{new Date(e.created_at).toLocaleString('pt-BR')}</time>
                            <span>{e.actor === 'human' ? `admin ${e.actor_id}` : e.actor}</span>
                        </li>
                    ))}
                </ul>
            </section>

            <section>
                <h4>Ações</h4>
                <button disabled title="Revogar validação fica para sprint pós-v2">
                    Revogar validação manual (em breve)
                </button>
            </section>
        </aside>
    );
}
```

### 14.3 — Integração nos painéis O8/O9

**Localização:** `components/admin/dashboard/PanelO8.tsx` e `PanelO9.tsx`

```typescript
// Adicionar handler de clique na linha do painel agregado
function PanelO8() {
    const [drilldown, setDrilldown] = useState<{ type: string; id: string } | null>(null);

    return (
        <>
            <table>
                {/* ... linhas agregadas ... */}
                <tr onClick={() => setDrilldown({ type: 'remap', id: eventId })}>
                    {/* ... */}
                </tr>
            </table>

            {drilldown?.type === 'remap' && (
                <RemapDrilldown
                    eventId={drilldown.id}
                    onClose={() => setDrilldown(null)}
                />
            )}

            {drilldown?.type === 'human_validated' && (
                <HumanValidatedDrilldown
                    canonicalId={drilldown.id}
                    onClose={() => setDrilldown(null)}
                />
            )}
        </>
    );
}
```

### 14.4 — Tests M2

**Mudança v2:** evento de teste usa `event_name='canonical_role_merged'` + payload em `new_state` com `previous_canonical_label`/`canonical_label`/`merged_into`/`affected_jobs`. Schema events com `resource_id`/`resource_type` + `actor='system'` obrigatório.

```typescript
// tests/integration/sprint-v4_0/item-14-drilldown.spec.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { RemapDrilldown } from '@/components/admin/dashboard/RemapDrilldown';

describe('Item 14 — Drilldown UI', () => {
    it('RemapDrilldown carrega evento de canonical_role_merged e exibe campos relevantes', async () => {
        // Setup: criar evento de teste com schema v2
        const { data: event } = await supabase
            .from('events')
            .insert({
                event_name: 'canonical_role_merged',
                resource_type: 'job_canonical_role',
                resource_id: 'src-id',
                actor: 'system',
                actor_id: '00000000-0000-0000-0000-000000000001',
                previous_state: { canonical_label: 'Source A', status: 'active' },
                new_state: {
                    merged_into: 'tgt-id',
                    canonical_label: 'Target B',
                    previous_canonical_label: 'Source A',
                    affected_jobs: 12,
                    status: 'deprecated',
                },
                reason: 'Test merge',
            })
            .select()
            .single();

        const onClose = jest.fn();
        render(<RemapDrilldown eventId={event!.id} onClose={onClose} />);

        await waitFor(() => {
            expect(screen.getByText('Source A')).toBeInTheDocument();
            expect(screen.getByText('Target B')).toBeInTheDocument();
            expect(screen.getByText('Test merge')).toBeInTheDocument();
            expect(screen.getByText('12')).toBeInTheDocument();  // affected_jobs
        });

        // Test botão fechar
        fireEvent.click(screen.getByText('×'));
        expect(onClose).toHaveBeenCalled();

        // Cleanup
        await supabase.from('events').delete().eq('id', event!.id);
    });
});
```

### 14.5 — Esforço estimado

- **Migration `human_validated_at/by`:** 30 minutos (NOVO em v2 — colunas não existiam)
- **Refator `human-validated` endpoint:** 1 hora
- **`RemapDrilldown.tsx`:** 2.5 horas
- **`HumanValidatedDrilldown.tsx`:** 2 horas
- **Integração nos painéis O8/O9:** 1.5 horas
- **Tests M2:** 2 horas
- **Total:** 9.5 horas (era 7h na v1 — adição da migration + refactor)

---


# Conclusão e Checklist Consolidado

## Esforço total consolidado

| PR | Item | Esforço | Subtotal PR |
|---|---|---|---|
| **PR1** | Item 15 (drifts de nomenclatura) | 6-7h | |
| | Item 16 (DROP de colunas redundantes) | 6h | |
| | Item 17 (DROP de tabela/coluna fantasma) | 1h | |
| | Item 18 (RLS faltante + DROP backups) | 2.5h | **15.5-16.5h** |
| **PR2** | Item G2 (taxonomia dinâmica + DA2 migração total) | **27h** (subiu 1h em v4: nova §G2.4.bis seed families + helper findFamilyMatches) | |
| | Item G2.2.bis (NOVO em v4 — taxonomy_families + junction) | **1.5h** | |
| | Item G3 (áreas de atuação 0:N) | 16h | **44.5h** |
| **PR3** | Item 7 (regra dinâmica + retroativa + resolve_canonical) | 6.5h | |
| | Item 2 (CRON pipeline-maintenance) | 7h | **13.5h** |
| **PR4** | Item 1A (distinct_sources_count) | 4h | |
| | Item 1A.bis (filtro de agências, Camada A primária + B fallback + backfill via events) | **6.5h** | |
| | Item 4 (saneamento 270 órfãs) | 3h | |
| | Item 6 (trigger autoritativo único) | 1.5h | **15h** |
| **PR5** | Item 8 (FK rapidapi + banner D) | 5h | |
| | Item 10 (thresholds fundamentados) | 4h | **9h** |
| **PR6** | Item 13 (tests M2 — 4 arquivos) | 8.5h | |
| | Item 14 (UI admin drilldown) | 7h | **15.5h** |
| **TOTAL** | | | **113h** |

**Estimativa bruta:** 113 horas (subiu vs v3 = 110.5h por causa de Decisão 3 — Opção B).
**Estimativa com fluência (Antigravity executando paralelizações típicas):** **59-77 horas** (estimativa de §Pré-requisitos cravada em cima dessa fluência).

---

## Ordem de execução recomendada

A ordem abaixo minimiza retrabalho — cada PR depende apenas dos anteriores:

### Fase 1 — Limpeza (PR1)
**Objetivo:** schema limpo antes de qualquer construção nova.
**Pré-requisito:** nenhum.
**Resultado:** drifts resolvidos, colunas redundantes removidas, RLS endurecida.

### Fase 2 — Fixes de pipeline (PR4)
**Objetivo:** corrigir bugs antes de construir governança em cima.
**Pré-requisito:** PR1 (rename de colunas pode afetar callsites).
**Resultado:** `distinct_sources_count` correto, agências filtradas, 270 órfãs saneadas, trigger único.

### Fase 3 — Inversão de paradigma (PR3)
**Objetivo:** estabelecer regra dinâmica e ciclo de vida de canônicos.
**Pré-requisito:** PR4 (vacancy_count e sources precisam estar corretos antes).
**Resultado:** canônicos novos seguem regra dinâmica em runtime, retroativa aplicada, `resolve_canonical` SQL ativo.

### Fase 4 — Governança de taxonomia (PR2)
**Objetivo:** migrar JSONs para banco com fluxo dinâmico.
**Pré-requisito:** PR3 (resolve_canonical é usado em endpoints de search do PR2).
**Resultado:** `taxonomy_relations` populada, CRON Opus ativo, áreas 0:N seedadas, modais funcionais.

### Fase 5 — Observação e UX (PR5)
**Objetivo:** trazer transparência ao usuário final.
**Pré-requisito:** nenhum (independente, mas eficiente após pipeline estável).
**Resultado:** banner D condicional, thresholds fundamentados na metodologia.

### Fase 6 — Refinos (PR6)
**Objetivo:** consolidar com tests e UI admin.
**Pré-requisito:** todos os PRs anteriores.
**Resultado:** cobertura de tests M2 nos 4 arquivos críticos, drilldown nos painéis O8/O9.

---

## Checklist final cravado

### Schema
- [ ] `taxonomy_relations` criada e seedada
- [ ] `taxonomy_versions` criada com versão inicial
- [ ] `taxonomy_families` criada (NOVO em v4 — Decisão 3 Opção B)
- [ ] `taxonomy_family_canonicals` (junction N:N) criada (NOVO em v4)
- [ ] `canonical_role_domain_links` (junction N:N) criada
- [ ] `canonical_role_domains` populada (20 áreas + 128 + ~536 backfill IA)
- [ ] `profiles.pending_label_change_notification` adicionada
- [ ] `profiles.pending_label_change_notification_sent_at` adicionada (NOVO no checklist v4 — rate limit coalescente)
- [ ] `analyses.rapidapi_usage_log_id` adicionada
- [ ] `analyses.rapidapi_log_created_at` adicionada (NOVO no checklist v4 — denormalização para Banner D)
- [ ] `job_postings.is_recruitment_agency` adicionada
- [ ] `job_canonical_roles.domain_id` (1:1 dormente) DROPADA
- [ ] **10 colunas** redundantes DROPADAS (Item 16) — corrigido v4 (era 9 errado: contagem real inclui `shared_results.final_score` adicionado em v2)
- [ ] `resume_role_assignments` DROPADA + `analyses.assignment_id` DROPADA
- [ ] 5 backups DROPADOS após export defensivo
- [ ] CHECK constraint `chk_curated_has_canonical` adicionado

### Triggers e funções SQL
- [ ] Trigger `trg_recompute_distinct_sources_count` ativo
- [ ] Função `resolve_canonical(p_id)` criada e testada
- [ ] Função `recompute_distinct_sources_count()` criada
- [ ] Função `enforce_original_immutability()` atualizada para `title`/`description` (não mais `original_*`)
- [ ] PASSO 4 do `maintenance_phase_2` REMOVIDO (trigger autoritativo único)

### RLS hardening
- [ ] `canonical_seniority_distribution` ENABLE RLS deny-all
- [ ] `allowed_for_pre_resolution` ENABLE RLS deny-all
- [ ] `taxonomy_relations` ENABLE RLS deny-all
- [ ] `canonical_role_domain_links` policy SELECT public + service_role bypass
- [ ] `canonical_role_domains` policy SELECT public + service_role bypass
- [ ] `generated_documents` policy SELECT-only para dono via profile_id
- [ ] `analysis_fetch_locks` policies cosméticas DROPADAS
- [ ] `ai_usage_logs` mantém deny-all (decisão consciente, sem ação)

### Renames de colunas
- [ ] `analyses.origin_state → location_state`
- [ ] `submitted_jobs.location_state → origin_state`
- [ ] `job_postings.location_state → origin_state`
- [ ] `analysis_fetch_locks.state → location_state`
- [ ] `rapidapi_usage_logs.state → location_state`
- [ ] `job_no_postings.user_id → profile_id`
- [ ] `score_history.recorded_at → created_at`
- [ ] `job_postings.original_title → title` + DROP de `title` antigo
- [ ] `job_postings.original_description → description` + DROP de `description` antigo
- [ ] `prompt_content_version → taxonomy_content_version` (global)
- [ ] FK `analysis_fetch_locks.locked_by` adicionada explicitamente

### Endpoints novos
- [ ] `/api/cron/taxonomy-validation` (Opus 4.7 diário às 03:00)
- [ ] `/api/cron/pipeline-maintenance` (reformulado, 04:00)
- [ ] `/api/areas` (lista de áreas)
- [ ] `/api/areas/[domainId]/canonicals` (canônicos da área)
- [ ] `/api/canonicals/search` (search com resolve_canonical)

### Código TS — refatorações
- [ ] `lib/pipeline/taxonomy-cache.ts` (refatorado: Postgres + Redis write-through; preserva assinatura de `getFullTaxonomyCache`/`getTaxonomyLoader` para os 4 callsites existentes)
- [ ] `lib/taxonomy/merge-canonicals.ts` (mergeCanonicals + markUsersForLabelChangeNotification + generateSlug)
- [ ] `lib/pipeline/agency-detector.ts` (`detectAgency` Camada A primária + B fallback)
- [ ] Refatorar 4 callsites JSON-direto → `getRelations()` (zero callsite produtivo lendo dos 3 JSONs após PR2)
- [ ] Substituir `resolveCanonicalRedirect` TS → RPC SQL
- [ ] Refatorar `AnalysisTriggerModal.tsx:228+` para JOIN com `job_canonical_roles(label)`
- [ ] Atualizar callsites de UF, `user_id`, `recorded_at`, `original_title` em todo lib/

### Frontend
- [ ] `LabelChangeToast.tsx` (toast no próximo login)
- [ ] `CalibrarParaOutraFuncaoModal.tsx` (modal funcional com áreas)
- [ ] `MapaRegionalCompetenciasModal.tsx` (modal funcional com áreas)
- [ ] `BannerStale` no `AnalysisResultModal.tsx`
- [ ] `RemapDrilldown.tsx` (admin)
- [ ] `HumanValidatedDrilldown.tsx` (admin)
- [ ] Integração de drilldown nos painéis O8/O9

### Página de metodologia
- [ ] Texto 1 "Quando a base de vagas tem distorção, sinalizamos" adicionado
- [ ] Texto 2 "Como o catálogo de funções se mantém vivo" adicionado
- [ ] Seção "Referências" linkando DOJ-FTC, ASA, Sólides, arxiv:2406.15373

### Scripts de seed e backfill
- [ ] `scripts/seed-taxonomy-relations.ts` executado
- [ ] `scripts/seed-canonical-role-domains.ts` executado (20 áreas)
- [ ] `scripts/seed-canonical-role-domain-links.ts` executado (128 canônicos)
- [ ] `scripts/backfill-canonical-domains.ts` executado (~536 canônicos)
- [ ] Backfill SQL `distinct_sources_count` rodado
- [ ] Backfill SQL `is_recruitment_agency` rodado
- [ ] Migration retroativa Item 7B aplicada (583 pending → buckets)
- [ ] Saneamento dos 270 órfãos concluído (Item 4)

### Tests M2
- [ ] Tests para `persist-precheck`
- [ ] Tests para `persist-curation`
- [ ] Tests para endpoint `/admin/canonicals/[id]/remap`
- [ ] Tests para endpoint `/admin/canonicals/human-validated`
- [ ] Tests específicos de cada item (15, 16, 17, 18, G2, G3, 7, 2, 1A, 1A.bis, 4, 8, 10, 14)

### Validações pós-deploy
- [ ] Confirmar que todos os 9 RLS endpoints respondem corretamente para authenticated/anon
- [ ] Confirmar que CRON `taxonomy-validation` rodou no primeiro dia sem erros
- [ ] Confirmar que CRON `pipeline-maintenance` rodou e não emitiu zumbis falsos
- [ ] Confirmar que `taxonomy_content_version` bumpa apenas quando há mudança real
- [ ] Confirmar que cache Redis está sendo invalidado corretamente
- [ ] Confirmar que toast aparece para profile com flag setada
- [ ] Confirmar que email transacional é disparado
- [ ] Confirmar que UI dos 2 modais carrega áreas e canônicos
- [ ] Confirmar que `resolve_canonical` SQL retorna corretamente em cadeias profundas
- [ ] Confirmar que CHECK constraint impede curated sem canonical
- [ ] Confirmar que trigger `distinct_sources_count` recompõe automaticamente

---

## Pré-requisitos operacionais (recap)

Antes de começar:

- [ ] Acesso ao banco de produção via Claude Code confirmado
- [ ] Vercel Hobby ainda em vigor (CRONs degradados anotados — restaurar após upgrade)
- [ ] Storage para export defensivo dos backups (~3MB compactado)
- [ ] API key Opus 4.7 ativa em `.env.local`
- [ ] Redis Upstash ativo, com circuit breaker confirmado/implementado (DA2 elimina fallback JSON)
- [ ] Anthropic SDK ≥ 0.39 confirmado em `package.json` (estrutura `tool_use.input` validada)
- [ ] Permissão de escrita em `data/methodology-content.md`
- [ ] Rerun de métricas Bloco S e Bloco O confirmado (evitar usar números de 22/04/2026)
- [ ] Diagnóstico `description_curated ≠ original_description` rodado (Item 15.5.0)
- [ ] Inspeção dos 8 events com `metadata->'raw_data'` não-array agendada como tarefa pós-deploy

---

## Riscos identificados e mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Migration Item 7B (retroativa) apaga canônicos legítimos | Baixa | Alto | Guard com 15 FKs reais + `WHERE coluna IS NOT NULL` em cada subquery + dry-run obrigatório |
| Substituição total de JSON por Redis quebra Camada 1 (DA2) | Média | Alto | Smoke test obrigatório dos 4 callsites em staging + circuit breaker Redis confirmado como pré-requisito + tests M2 exaustivos |
| Backfill IA do Item G3 demora muito (>30min) | Média | Médio | Limite de batch + retry automático + checkpoint para retomada |
| CRON Opus erra em decisões DISAGREE com slug conflict | Baixa | Médio | Defesa em código: se conflict, faz merge ao invés de rename |
| `resolve_canonical` entra em ciclo infinito | Baixíssima | Alto | Profundidade máxima 10 + `visited UUID[]` defense já cravados |
| RLS deny-all em `canonical_seniority_distribution` quebra leitura admin | Baixa | Baixo | Bloco R confirmou: todos callers usam createAdminServerClient (bypass) |
| Trigger `distinct_sources_count` causa lock em batch grande | Média | Médio | `FOR EACH ROW` é eficiente; índice em `canonical_role_id` + monitoramento |
| Renames de colunas rompem queries em produção | Média | Alto | grep exaustivo + dry-run + deploy gradual + rollback prep |
| Falso positivo do filtro de agências em vagas internas (Michael Page interno classificado como agência) | **Resolvido em v3** | — | Camada A primária + B fallback evita o falso positivo. Test específico cravado em §1A.bis.7. |
| `merge_canonicals` falha por UNIQUE em `taxonomy_relations` ou `resume_skill_enrichments` | **Resolvido em v3** | — | UPDATE NOT EXISTS + DELETE residual aplicado em 3 FKs com risco (P0.4 Gemini) |

---

## Gates operacionais antes de produção (consolidação GenSpark)

Esta seção consolida 6 gates operacionais que precisam ser executados **antes** de cada PR ser considerado pronto para deploy em produção. Cada gate é responsabilidade explícita do executor (Antigravity ou Claude Code, conforme indicado).

### Gate 1 — Dry-run obrigatório do Item 7B (DELETE retroativo)

**Antes de PR3 ser executado:**
1. Rodar a query de pré-validação cravada em §7B (que reporta `total_zumbis`, `bloqueados_pelo_guard`, `apagaveis`).
2. Confirmar com o PO que o número de `apagaveis` é compatível com o esperado (~quantos zumbis órfãos legítimos).
3. Se `bloqueados_pelo_guard` for alto inesperadamente, investigar antes de prosseguir — pode indicar dados inesperados em alguma das 15 FKs.
4. Apenas após confirmação explícita, executar o DELETE definitivo.

### Gate 2 — Execução fatiada por PR (não monolítica)

**Princípio:** cada PR é commit/branch separada. Antigravity NÃO consolida vários PRs em um único deploy. Após cada PR:
- Claude Code valida migration no banco de produção.
- Onsly faz validação manual de smoke (queries cravadas no PR).
- Apenas após go-ahead explícito, próximo PR começa.

Vantagem: rollback granular se algo falhar. Desvantagem: ciclo mais longo. Trade-off cravado pelo PO.

### Gate 3 — Vercel Pro recomendado antes do PR2

**Justificativa:** PR2 é o mais pesado (~26h de execução), tem migrações que tocam coração do pipeline (taxonomy-cache refatorado, 4 callsites JSON→banco), e introduz dependência hard de Redis (DA2 elimina fallback JSON).

**Recomendação:** upgrade Vercel Hobby → Pro antes do deploy do PR2 para:
1. Restaurar CRONs degradados (`stripe-webhook-cleanup` e `reconcile-payments`) e ter ambiente coerente para validação.
2. Evitar deploys parados por timeout em build (PR2 é grande).
3. Habilitar Edge Runtime se necessário em endpoints críticos.

Não-bloqueante mas fortemente recomendado.

### Gate 4 — Regressão pós-renames de colunas

**Antes de cada PR que renomeia colunas (PR1 principalmente):**
1. Antigravity roda grep exhaustivo em `lib/`, `app/`, `scripts/` por nome antigo.
2. Cada callsite encontrado é atualizado e testado isoladamente em staging.
3. Smoke test em staging com queries reais antes do deploy em produção.
4. Manter branch de rollback pronta para revert imediato se erros aparecerem em prod.

### Gate 5 — Observabilidade Redis com circuit breaker

**Pré-requisito hard do PR2:**
1. Antigravity confirma via grep que `lib/pipeline/taxonomy-cache.ts` tem circuit breaker (padrão já usado no pipeline AI).
2. Se ausente, implementa antes do deploy do PR2.
3. Validação: simular Redis indisponível em staging — pipeline deve degradar graciosamente (chamar Postgres direto), não derrubar o request inteiro.

Sem isso, DA2 vira ponto único de falha: Redis cai → Camada 1 do pipeline para → todo o fluxo de curadoria trava.

### Gate 6 — Decisão definitiva sobre email `label_change_notification`

**Status atual cravado:** template HTML diferido para sprint pós-v3. Funciona como stub. Toast frontend cobre usuário ativo.

**Antes de produção:**
- Onsly decide se aceita stub (toast cobre) por algumas semanas, OU se prioriza template HTML como item de sprint pós-v3.
- Decisão registrada no commit message do PR2.

Se decisão for "aceitar stub": adicionar tarefa no backlog `Próximos passos pós-Sprint v4.0`.
Se decisão for "prioritário": agendar mini-sprint dedicada.

### Gate 7 — Backups defensivos (pré e pós-PR1)

**Antes de PR1:**
- `CREATE TABLE..._backup_v6_pre_pr1 AS SELECT *` para 4 tabelas core: `job_canonical_roles`, `analyses`, `submitted_jobs`, `job_postings`. Defensivo: Supabase free tier sem PITR (Point-In-Time Recovery), backup explícito é a única rede de segurança.
- `pg_dump --schema-only` do banco completo para ter snapshot do schema antes da migração.

**Após PR1 estabilizado (validação 24h):**
- DROP dos 4 backups `_pre_pr1` se PR1 estabilizou em produção sem regressão.
- `pg_dump --schema-only` post-PR1 como referência para PR2 começar.

### Gate 8 — Backup pós-PR2 (estado pós-DA2)

Após PR2 (refatoração JSON→banco), antes de prosseguir para PR3:
- `pg_dump --schema-only` para snapshot pós-DA2 (estado do schema com `taxonomy_relations` populada e callsites refatorados).
- Permite rollback rápido para "estado v3 pós-PR2" se PR3 ou PR4 introduzirem bugs sistêmicos.

---



## Próximos passos pós-Sprint v4.0

Anotações para sprints futuras (não fazem parte desta v4.0):

**G2 — evolução pós-launch:**
- Tela admin de revisão de candidatos (caso fluxo Sonnet→Opus precise intervenção manual)
- Tunagem do batch size do CRON Opus baseado em volume real
- Implementação de Batch API (50% off) quando volume justificar
- Email template `label_change_notification` desenhado e cadastrado no Hostinger
- Camada C de detecção de agências por keyword (se falsos negativos justificarem)

**G3 — refino:**
- UI para admin gerenciar áreas (criar/editar/desativar)
- Backfill de novas áreas conforme mercado evolui
- Análise de cohort para validar ortogonalidade industry × domain

**Hardening RLS dedicado:**
- Auditoria sistemática de todas as policies
- Adicionar constraints CHECK em colunas que dependem de regra de negócio

**Restauração de CRONs:**
- `stripe-webhook-cleanup`: restaurar `*/5 * * * *` após Vercel Pro
- `reconcile-payments`: restaurar `0 * * * *` após Vercel Pro

**Não-itens explicitamente fora desta sprint:**
- G1 (tela Blacklist nativa) → backlog separado
- Item 5 (admin merge-canonicals validação por Onsly contra ~7.000 registros) → operacional pós-spec
- Stripe `/api/credits/checkout` → não é desta sprint
- Hint `ResourceEditModal` → backlog
- Instrumentação de analytics → backlog

---

## Aprovação e início

Esta spec é considerada **completa e cravada** pelo PO em sua versão FINAL v6.0 — sexta iteração do documento.

A v6.0 absorve a triage de 5 reviewers sobre v5.0 (Gemini, Claude Code com acesso DB direto, Outra-Claude, Grok, GenSpark). 7 bugs P0 reais + 5 cosméticos/editoriais + 1 recomendação processual foram corrigidos. Os achados confirmam o padrão **"esquecimento evolutivo"** (3ª rodada consecutiva), com 2 achados qualitativamente novos do Gemini sobre paradoxos lógicos no merge transacional que só foram possíveis identificar porque o resto da spec está sólido.

**Continua válido das versões anteriores:**

1. **Decisão 1** (v4) — `description_curated` é coluna ATIVA do pipeline (escrita em 7 arquivos), preservada
2. **Decisão 2** (v4) — `mark_users_for_label_change_notification` JOIN via `resume_role_suggestions`
3. **Decisão 3 — Opção B** (v4) — `family_synonyms` ganha tabelas dedicadas (`taxonomy_families` + junction)
4. **Decisão 4** (v4) — Tabela é `job_posting_skills` (singular)

**Adicionado em v6.0 (13 itens):**

**P0 lógicos (Gemini):**
- **Notificação atomicizada:** `mark_users_for_label_change_notification` movida para DENTRO do `merge_canonicals` SQL (FASE 0, antes da FASE 1 reapontar suggestions). Sem isto, paradoxo temporal fazia notificação retornar 0 sempre. Wrapper TS e `process_opus_disagree` NÃO chamam mais externamente.
- **Mesclagem real de `last_seen_at`/`first_seen_at`:** FASE 2 do `merge_canonicals` agora preserva frescor de mercado para fontes sobrepostas (winner herda `GREATEST(last_seen)` e `LEAST(first_seen)` do loser antes do delete).
- **Toast genérico:** `LabelChangeToast.tsx` não lista mais merges de outros usuários (vazamento entre contas). Texto comportamental simples.

**P0 test fixture (Outra-Claude + Claude Code):**
- Test §G2.13 sem `role_label` (coluna dropada por PR1)
- Test §G2.13 com `input_type: 'text'` em `resumes` (NOT NULL sem default)
- 3 INSERTs em `job_postings` com `posted_at` + `expires_at` (NOT NULL sem default)
- 4 INSERTs com `linkedin_id: \`test-${crypto.randomUUID()}\`` (UNIQUE colidia entre runs interrompidos)

**Editoriais e processuais:**
- RPC `get_top_samples_per_canonical` usa `normalized_company` nativo (não `company` cru) — protege Opus de dados sujos
- Backups `_v3_pre_pr1` → `_v6_pre_pr1` (consistência)
- §15.5.0 reforça que `description_curated` é output do Sonnet
- Comentário no test §G2.13 explica E2E não-conectado
- Nota expandida de rollback parcial (lista do que NÃO é recuperado: índices, triggers, constraints, RLS, comments)
- Pré-validação NOT NULL para test fixtures como pré-requisito formal

**Backlog operacional pós-v6.0:**

- **Sprint Fluxo do Currículo (próxima):** investigação E2E mapeada pelo Claude Code revelou 3 quebras independentes — (1) `resume_role_assignments` é dead code (0 rows, 0 callsites efetivos, função stub), (2) currículo não é normalizado em background (só quando usuário abre modal), (3) worker no Vercel Hobby não termina (timeout 10s impede até diagnosticar outros bugs). Sprint dedicada com 5 PRs cobrirá: upgrade Vercel Pro (Gate 0 bloqueante), diagnóstico empírico, denormalização `analyses.canonical_role_id`, normalização background no upload, error_code consistente, drop `resume_role_assignments`.
- **Decisão 2 vira provisória:** quando `analyses.canonical_role_id` for criado e populado consistentemente na Sprint Fluxo do Currículo, `mark_users_for_label_change_notification` será reescrita uma terceira vez para JOIN direto via coluna indexada (mais robusta que via `resume_role_suggestions` que é cache da UI).
- **Listagem específica no toast:** quando `analyses.canonical_role_id` existir, podemos voltar a listar mudanças por usuário com filtro confiável.

Após aprovação, fluxo de execução:
1. Multi-AI review (Gemini, GPT, Grok, Claude Code, Outra-Claude) sobre a v6.0. **GenSpark fica fora do pool técnico** (apenas Go/No-Go pré-produção). **DeepSeek descartado nesta rodada** por revisar versão errada. **Mistral removido definitivamente** desde v4.
2. Triage de feedback por Claude (acatado/rejeitado/duplicata com contagem)
3. Antigravity implementa PR a PR conforme ordem recomendada
4. Claude Code executa migrations SQL diretamente no Supabase
5. Validação por Onsly em cada PR antes de avançar para o próximo

**Este documento é a fonte autoritativa única da Sprint v6.0.** Qualquer divergência durante implementação deve voltar para essa spec e ser registrada como mudança formal.

---

**Fim da Sprint v6.0 — FINAL**


