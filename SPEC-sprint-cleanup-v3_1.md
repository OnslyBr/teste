# SPEC — Sprint Cleanup v3.1

**Projeto:** CalibraCV
**Data de fechamento da spec:** 2026-05-13
**Versão:** cleanup-v3.1
**MER de referência:** v37
**Ambiente alvo:** pré-produção, base operacional limpa (sem usuários ativos, sem dados históricos relevantes)
**Executor:** Antigravity (com acesso direto ao Supabase para migrations)

---

## §0 Convenções e diretrizes de execução

### §0.1 Convenção nominal

Esta sprint estabelece a convenção definitiva para nomes de objetos SQL e event_names:

| Padrão | Status | Racional |
|---|---|---|
| `*_role` / `*_skill` (qualificador semântico-explícito) | **Adotado** para entidades canônicas | Elimina ambiguidade de "canonical" sozinho |
| `_jcr` / `_jcs` (sufixo) e `jcr_*` / `jcs_*` (prefixo) | **Preservado** | Abreviações de tabela (`job_canonical_roles`, `job_canonical_skills`), informativas e consistentes |
| `_jp` / `_jps` / `_foi` | **Preservado** | Idem — abreviações de tabela |
| `canonical_role_*` / `canonical_skill_*` (qualificado) | **Preservado** | Não cria ambiguidade |
| `canonical_*` (genérico, sem qualificador) | **Eliminado** | Legado da fase pré-paridade-skills |

A função `retire_canonical(uuid, text [, text])` é exceção legítima ao último item — é dispatcher polimórfico intencional que aceita `entity_type` por parâmetro. Preservada com o nome atual.

### §0.2 Diretrizes de execução

1. **TodoWrite item-por-item antes de edição.** Para cada §X.Y, criar item na todo list e marcar concluído somente após validação contra o critério da §7.2. Validação de conformidade é DURANTE a execução, não depois.
2. **Validação ground truth obrigatória.** Antes de submeter PR, rodar os blocos de evidência da §7.1 contra o banco e o código real. Revisão estática multi-AI não substitui.
3. **Spec é fotografia.** Esta versão descreve o estado pretendido pós-implantação. Não há narrativa histórica, mapeamento de fixes acumulados, ou referência a versões anteriores no corpo.
4. **PT-BR puro em narrativa e UI.** Nomes técnicos (rotas, funções, parâmetros, arquivos) em inglês conforme convenção do projeto.
5. **Sem novos canais externos ou mecanismos assíncronos sem autorização.** Esta sprint não introduz SMS, e-mail externo, push, webhook externo, budget guard novo, kill-switch novo, rate limiter novo, monitor externo novo, ou alerta novo — qualquer adição desta natureza requer autorização expressa do Onsly (diretriz operacional permanente). **Telas, painéis admin, endpoints internos, modais e refactors de UI** estão dentro do escopo explícito desta sprint, listados nas seções §4 (endpoints) e §5 (frontend); não constituem violação deste ponto. O registry §2.17 (`admin_panel_functions`) é instrumentação de uso das telas admin existentes (não cria novos alertas ou notificações).
6. **Simetria skill↔role.** Toda adição que toca canonicals, decisões, merges, painéis ou telas administrativas precisa considerar paridade entre `role` e `skill`. Implementação prática: SQL via `UNION ALL` + `entity_type` derivado; endpoints aceitam `entity_type` em path ou payload; UI usa mesmo componente parametrizado. Assimetria justificada (limitação de produto ou domínio exclusivo de uma entidade) deve estar documentada em D-PS específica. A janela de simetrização é durante a sprint que toca o objeto — não há ciclo posterior dedicado. Mapa de simetria desta sprint em §0.3.

### §0.3 Mapa de simetria desta sprint

Auditoria das adições/alterações desta sprint quanto à paridade `role` ↔ `skill`:

| § | Conteúdo | Simetria | Mecanismo |
|---|---|---|---|
| §2.0a | `normalize_role_label` / `normalize_skill_label` (vivo) | ✓ par simétrico | função SQL IMMUTABLE por entidade + paridade TS (`normalizeTitle`/`normalizeSkill`) governada por D-PS-54 |
| §2.4 / §2.5 | `lookup_canonical_*_by_normalized_label` | ✓ par simétrico | 2 funções usando `normalize_*_label` correspondente; índice funcional dedicado em JCR criado em §2.5 |
| §2.6 / §2.6b | `lookup_canonical_*_by_normalized_alias` | ✓ par simétrico | 2 funções com filtro explícito `entity_type` |
| §2.7 + §2.8 / §2.8b | `resume_roles` (overhaul) / `resume_skills` (tabela nova) | ✓ par simétrico | confidence numeric(5,4), coluna gerada `*_normalized` STORED, UNIQUE regular, sem CHECK fixo de hard gate |
| §2.16 | `v_merge_audit_history` | ✓ UNION ALL com entity_type | role + skill em uma única view |
| §2.22 | `fn_flag_needs_opus_review_jcr/jcs` | ✓ par simétrico | trigger por tabela |
| §2.23 / §2.24 | `catchup_pending_role_promotions` / `_skill_` | ✓ par simétrico | 2 funções, idêntica lógica |
| §2.24a | `fn_promote_skill_on_threshold` | ✓ par com `fn_promote_role` | edição cirúrgica (4 alterações) |
| §2.26 | `admin_panel_custo_opus_por_canonical` | ✓ role + skill via UNION ALL | imputação dual A/B do par |
| §2.27 / §2.28 | merge auto vs manual / latest_posted | ✓ role + skill via UNION ALL / jsonb agregado | um endpoint, dois agregados |
| §2.29 | `admin_panel_suggestion_rejected_by_skill` | ✗ só skill | **D-PS-42 justifica** (produto não tem rejeição user-level de role) |
| §2.31 | `taxonomy_blacklist` | ✓ `entity_type CHECK ('role','skill')` | tabela única polimórfica; consultada em §2.13 e §2.14 antes de criar canonical (M-7) |
| §2.32 / §2.33 / §2.34 | painéis 11, 12, 13 | ✓ role + skill via UNION ALL | jsonb com chaves `role:{}` + `skill:{}` |
| §2.35 | `reject_and_blacklist_canonical_pair` | ✓ par dispatcher por `p_entity_type` | 1 RPC, 2 branches |
| §3.2 / §3.3 | `extractAndPersistResumeSkills` / `_Roles` | ✓ par simétrico | grava aprovados+reprovados; canonização em chunks de 5; dedup via `normalize*` |
| §3.6 fetch-refs | lê `resume_roles` + `resume_skills` com filtro de gate | ✓ par simétrico | duas queries paralelas, mesmo predicate |
| §4.3 endpoints decisão | `/merge-canonicals/ignore` + `/merge-skills/ignore` | ✓ par simétrico | mesmo handler, entity_type via path |
| §4.6 / §4.8 endpoints domain | apenas role | ✗ **D-PS-41 justifica** (domínios não se aplicam a skills) |
| §4.7 painéis | servem role+skill via RPCs subjacentes | ✓ painéis B7 e novos 11/12/13 | dados unificados na resposta |
| §5.2.x merge-canonicals UI | aba role + aba skill no mesmo refactor | ✓ par simétrico | mesma pilha-degrau |
| §5.3 `DomainRoleCascade` | apenas role | ✗ **D-PS-41 justifica** |
| §6.2 modais | inclui `IgnoreModal` para merge-canonicals E merge-skills | ✓ par migrado | duas fichas (6.2.2 + 6.2.3) |
| §3.14 24 chaves | 12 `role.*` + 12 `skill.*` simétricas | ✓ paridade total | seed +1 row por entidade |
| Orchestrator (pipeline upstream) | `function_orchestrator_items` rastreia só role | ✗ **D-PS-40 documenta débito** | sprint dedicada futura |
| Pipeline CV (input 1) | `resume_roles` / `resume_skills` sem stage tracking | ✗ **D-PS-43 documenta decisão arquitetural** | CV é fluxo unitário, não pipeline multi-stage |
| `confidence_median` JCR vs JCS | role lê de `function_orchestrator_items`; skill lê de `job_posting_skills` curated | ✗ **D-PS-49 documenta assimetria de fontes** | paridade estrutural mantida; assimetria de fonte intencional por robustez do sinal |
| `latest_posted_at` JCR vs JCS | JCR via trigger dedicado em §2.9; JCS via cálculo embutido em `fn_jps_recompute_jcs` (vivo) | ✓ paridade comportamental | mecanismos diferentes, ambos recomputam via `MAX()` filtrando `curation_status='curated'` |

Assimetrias documentadas (5): D-PS-40 (orchestrator), D-PS-41 (domínios só role), D-PS-42 (Painel B7 #5 só skill), D-PS-43 (CV sem orchestrator), D-PS-49 (fontes de confidence_median).

### §0.4 Out-of-scope explícito

- Frentes A1.1 (tabela áreas de atuação) e A1.2 (transição dos 4 JSONs para tabelas).
- Tela de blacklist/proibidos (movida para backlog pós-benchmark).
- Mudanças em fluxos de pagamento, B2B, fingerprint, LGPD.
- Sprint CBO v6.6.6 (já concluída em ramo separado).
- Backlog admin preços pós-D0.
- Backlog home pós-login.
- Mudança de fonte da mediana JCR (Opção D descartada — `function_orchestrator_items.confidence` já é confidence de extração semanticamente equivalente a `job_posting_skills.confidence`).

---

## §1 Objetivo e frentes da sprint

### §1.1 Frentes consolidadas

1. **Simetria comportamental JCR/JCS:** remoção do gate de `confidence_median ≥ auto_min_confidence` em promoção JCS (Opção C); correção do gap latente onde canonicals promovidos com mediana entre `hard_gate.min_confidence` e `auto_min_confidence` não entravam imediatamente na fila Opus.
2. **Rename atômico:** eliminação do cluster `canonical_*` genérico (sem qualificador) em funções, triggers, event_names e strings de metadata. Preservação dos sufixos `_jcr`/`_jcs` legítimos.
3. **Resolução dos 16 bloqueadores ground truth** identificados nas auditorias (A1-A10, B3-B10).
4. **Tela admin `/admin/pipeline-config` (Calibração de Limiares):** edição governada de `pipeline_config` com 3 KPIs no topo, filtros, tabela 6 colunas (incluindo coluna "Acompanhamento" com tooltips ricos por chave referenciando os 10 painéis existentes em `LimiaresTab`), modal de edição com box "Impacto estimado", modal de histórico com botão de rollback por linha, e log imutável via RLS FORCE em `pipeline_config_history`.
5. **Tela admin `/admin/merge-canonicals` (Unificação de Cadastros) — pilha-degrau de 3 cards:** Card 1 = auditoria integrada de decisões automáticas (substitui a página separada `/admin/merge-canonicals/audit`); Card 2 = unificação manual legada preservada; Card 3 = aguardando revisão humana (ativo por padrão). Animação FLIP entre cards. Abas Funções/Habilidades. Tradução de veredictos para 3 labels (UNIFICAR / MANTER AMBOS / REVISAR). Botão "Detalhes" abre painel lateral. Nomenclatura "IA" em vez de "Opus" na UI pública. Skills com badges `skill_type` e `CrossTypeConfirmModal` preservado.
6. **Frente A1.3 — backfill IA de `domain_id`** nos ~616 canonicals roles ativos via `canonical_role_domain_links`, e 2 dropdowns em cascata área→função para entrada manual.
7. **AdminModal compartilhado** (`components/ui/admin-modal.tsx`) com `useId()`, `useReducedMotion`, `useFocusTrap<T>(isOpen)`, suporte a `role='alertdialog'`, `closeOnEscape`, `closeDisabled`, e migração de 6 modais (`ResourceEditModal`, `IgnoreModal` em merge-canonicals, `IgnoreModal` em merge-skills com harmonização de API, `ComparisonModal`, `DuplicatesModal`, `PipelineModal`).
8. **Registry de funções admin** via tabela `admin_panel_functions` — 29 funções operacionais mapeadas em 5 panel_ids (pricing, merge_canonicals, campaigns, ingestor, pipeline_config) com criticidade e tags para telemetria e feature flags por função.
9. **Limpeza de event_names role-only:** rename de event_names role-only emitidos em TS (`canonical_creation_blocked_*` → `role_creation_blocked_*`) e remoção de dead filter na RPC `o6_recent_errors`.
10. **5 painéis B7 de observabilidade:** 4 RPCs SQL novos (`admin_panel_custo_opus_por_canonical`, `admin_panel_merge_auto_vs_manual`, `admin_panel_latest_posted_distribution`, `admin_panel_suggestion_rejected_by_skill`) + validador de view pré-existente (`v_opus_effectiveness`) + 5 endpoints `GET /api/admin/limiares/panel_b7_*`. Painéis adicionais aos 10 painéis já existentes em `LimiaresTab.tsx` — não substituem nem reordenam.
11. **Endpoints públicos de cascata área→função:** `GET /api/taxonomy/domains` e `GET /api/taxonomy/roles-by-domain?domain_id=...` sem autenticação, com edge cache de 300s, expondo apenas dados públicos (`id, name` para domínios; `id, label` para roles). Consumidos pelo `DomainRoleCascade` quando montado em página pública (formulário de criação de perfil profissional, fluxo "outra função").

### §1.2 Premissas operacionais

- Antigravity conecta direto ao Supabase e executa migrations sem etapa manual de SQL pelo Onsly.
- Base operacional limpa: nenhum UPDATE retroativo sobre eventos históricos é necessário (confirmado).
- Critério de "fechado" para cada §: conformidade item-por-item validada via TodoWrite + bloco de evidência da §7.1 retornando o resultado esperado.

---

## §2 Migrations SQL

Ordem de execução = ordem de numeração. Cada migration é um arquivo em `docs/migrations/sprint-cleanup-v3/NN_descricao.sql`. Todas com transação explícita (`BEGIN; ... COMMIT;`), `SET LOCAL statement_timeout` e snapshot prévio do estado das funções afetadas via `pg_get_functiondef`.

### §2.0 — Ordem crítica de dependência

Dependências fortes que não podem ser invertidas (a ordem dos arquivos respeita a notação `NN_*`, mas é útil ter os pontos abaixo explícitos para revisão e para qualquer aplicação parcial ou rollback granular):

```
§2.0a (normalize_role_label)
   ├─→ §2.5 (lookup_canonical_role_by_normalized_label) usa a função
   ├─→ §2.5 (índice uq_jcr_canonical_label_normalized) usa a função na expressão
   ├─→ §2.6  (lookup_canonical_role_by_normalized_alias) usa a função
   ├─→ §2.7  (resume_roles.role_normalized GENERATED) usa a função
   └─→ §2.13 (reconcile_canonical_role) usa a função (blacklist check)

§2.0a precisa estar criada e IMMUTABLE antes de qualquer GENERATED ALWAYS AS ou índice funcional usá-la, ou o CREATE falha com "functions in column generation must be marked IMMUTABLE".

§2.10 (auto_assign_family_to_role)
   └─→ §2.12 (fn_promote_role_on_threshold) faz PERFORM auto_assign_family_to_role(...)

§2.10 antes de §2.12 — promoção depende da função renomeada existir.

§2.13/§2.14 (reconcile_canonical_*)
   ├─→ §2.31 (taxonomy_blacklist) já existe (checada pelas reconcile)
   └─→ §3.2/§3.3 (extractAndPersist*) consomem via RPC

§2.31 antes ou junto de §2.13/§2.14 — a tabela blacklist precisa existir para o check IF EXISTS bater.

§2.20 (DROP TRIGGER trg_pipeline_config_audit)
   └─→ §2.20a (trigger BEFORE UPDATE/DELETE) cria o substituto

§2.20 antes de §2.20a — primeiro remove o trigger duplicador, depois cria o trigger de imutabilidade.
```

**Precheck operacional antes de aplicar qualquer migration desta sprint:**

```sql
-- (a) Conferir source values legados em JCS e JCR — orienta lista do novo CHECK em §2.1 e §2.2
SELECT 'jcs' AS tbl, source, COUNT(*) FROM job_canonical_skills GROUP BY source
UNION ALL
SELECT 'jcr', source, COUNT(*) FROM job_canonical_roles GROUP BY source
ORDER BY tbl, source;

-- (b) Confirmar nomes reais dos CHECK constraints (evita DROP no-op silencioso)
SELECT conrelid::regclass AS tbl, conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid IN ('job_canonical_skills'::regclass, 'job_canonical_roles'::regclass)
  AND contype = 'c'
  AND pg_get_constraintdef(oid) LIKE '%source%';

-- (c) Confirmar que trigger duplicador de pipeline_config existe (a ser dropado em §2.20)
SELECT tgname FROM pg_trigger
WHERE tgrelid = 'pipeline_config'::regclass
  AND tgname = 'trg_pipeline_config_audit';

-- (d) Confirmar que normalize_skill_label existe e é IMMUTABLE
SELECT proname, provolatile FROM pg_proc WHERE proname = 'normalize_skill_label';

-- (e) Confirmar schema atual de resume_roles (espera: confidence smallint 0-100; sem role_normalized)
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'resume_roles' AND table_schema = 'public'
ORDER BY ordinal_position;
```

Se qualquer um dos itens (a)-(e) retornar valor inesperado, **abortar a aplicação** e reavaliar a spec antes de prosseguir — premissas da v2.9 mudaram no banco entre fechamento e execução.

---

### §2.0a — Função `normalize_role_label` (paridade SQL↔TS com `normalize_skill_label`)

**Arquivo:** `00a_normalize_role_label.sql`
**Objetivo:** criar função imutável de normalização de label de role no SQL, espelhando a `normalize_skill_label` já existente, para servir de expressão canônica nos índices funcionais de JCR e na coluna gerada de `resume_roles`.
**Bloqueador resolvido:** sem essa função, §2.5 (lookup canonical role) cairia em sequencial scan no índice funcional ou inventaria uma terceira normalização inconsistente.

**Paridade TS:** `lib/pipeline/text-processing.ts` deve ganhar (ou já tem) função `normalizeTitle(input: string): string` que replica este comportamento. Diretriz D-PS-54 governa: alterar uma exige alterar a outra + rebuild dos índices que dependam dela.

**Decisões de design vs `normalize_skill_label`:**
- Mantém: lowercase, strip de acentos PT-BR via `translate`, colapso de separadores, trim final
- Remove: SYMBOL_LOOKUP (C++ → cpp, C# → csharp, etc.) — role não tem essa polissemia semântica
- Adiciona: nada (role é mais simples que skill)

```sql
BEGIN;

CREATE OR REPLACE FUNCTION normalize_role_label(input_text text)
RETURNS text
LANGUAGE plpgsql
IMMUTABLE PARALLEL SAFE
AS $$
DECLARE
  v_result text;
BEGIN
  IF input_text IS NULL OR input_text = '' THEN
    RETURN '';
  END IF;

  -- Lowercase primeiro (translate só precisa cobrir minúsculas)
  v_result := lower(input_text);

  -- Strip de acentos PT-BR latinos (mesma tabela de normalize_skill_label)
  v_result := translate(
    v_result,
    'áàâãäåéèêëíìîïóòôõöúùûüýÿçñ',
    'aaaaaaeeeeiiiiooooouuuuyycn'
  );

  -- Colapso de separadores comuns em títulos de cargo
  v_result := regexp_replace(v_result, '[\-_/]', ' ', 'g');

  -- Collapse whitespace + trim
  v_result := regexp_replace(v_result, '\s+', ' ', 'g');
  v_result := btrim(v_result);

  RETURN v_result;
END;
$$;

COMMENT ON FUNCTION normalize_role_label IS
'Normalização canônica de labels de role para lookup e indexação. Paridade obrigatória com normalizeTitle() em lib/pipeline/text-processing.ts (D-PS-54). Alterar esta função exige alterar a TS + rebuild do índice uq_jcr_canonical_label_normalized (§2.5).';

COMMIT;
```

**Validação pós-migration:**

```sql
-- A função existe e é IMMUTABLE
SELECT proname, provolatile
FROM pg_proc
WHERE proname = 'normalize_role_label';
-- Esperado: 1 linha — normalize_role_label, i

-- Comportamento básico
SELECT normalize_role_label('Analista de Sistemas Júnior');
-- Esperado: 'analista de sistemas junior'

SELECT normalize_role_label('Gestão de Projetos - PMO');
-- Esperado: 'gestao de projetos pmo'

SELECT normalize_role_label('  Desenvolvedor/Programador  ');
-- Esperado: 'desenvolvedor programador'
```

---

### §2.1 — CHECK constraint `job_canonical_skills_source_check` ampliado

**Arquivo:** `01_jcs_source_check.sql`
**Objetivo:** garantir que `job_canonical_skills.source` aceita os valores legados em produção + o novo valor `resume_extraction`.
**Ground truth:** nome real da constraint é `job_canonical_skills_source_check` (auto-gerado pela paridade-skills v11 §2.7 ao criar inline na ADD COLUMN). Valor `cbo_mte_2002_seed` populado em 53.478 linhas em produção — exclui-lo na nova constraint aborta a migration (`check constraint is violated by some row`).

**Precheck obrigatório antes de aplicar:**

```sql
SELECT source, COUNT(*) AS qtd
FROM job_canonical_skills
GROUP BY source
ORDER BY source;
-- Cada valor presente DEVE estar na lista do novo CHECK abaixo.
-- Se aparecer source desconhecido, abortar e ampliar a lista antes de prosseguir.
```

**Migration:**

```sql
BEGIN;

-- Dropar pelo nome REAL (auto-gerado, não o atalho jcs_source_check)
ALTER TABLE job_canonical_skills
  DROP CONSTRAINT IF EXISTS job_canonical_skills_source_check;

-- Recriar com TODOS os valores legados + o novo
ALTER TABLE job_canonical_skills
  ADD CONSTRAINT job_canonical_skills_source_check
  CHECK (source IN (
    'llm_extractor',       -- legado preservado (paridade-skills v11)
    'cbo_mte_2002_seed',   -- legado preservado (53.478 linhas em prod)
    'manual_admin',        -- legado preservado
    'opus_arbitration',    -- legado preservado
    'resume_extraction'    -- NOVO (objetivo desta migration)
  ));

COMMIT;
```

**Validação pós-migration:**

```sql
SELECT pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conname = 'job_canonical_skills_source_check';
-- Esperado: CHECK ((source = ANY (ARRAY['llm_extractor'::text, 'cbo_mte_2002_seed'::text,
--   'manual_admin'::text, 'opus_arbitration'::text, 'resume_extraction'::text])))
```

---

### §2.2 — CHECK constraint `job_canonical_roles_source_check` ampliado

**Arquivo:** `02_jcr_source_check.sql`
**Ground truth:** nome real é `job_canonical_roles_source_check`. Valor `cbo_mte_2002_seed` populado em 2.694 linhas em produção. Outros valores legados aceitos pela constraint atual: `seed`, `llm_extractor`. Excluir qualquer um deles aborta a migration ou (se o DROP falhar pelo nome errado) deixa constraint zumbi que bloqueia `resume_extraction`.

**Precheck obrigatório antes de aplicar:**

```sql
SELECT source, COUNT(*) AS qtd
FROM job_canonical_roles
GROUP BY source
ORDER BY source;
-- Cada valor presente DEVE estar na lista do novo CHECK abaixo.
```

**Migration:**

```sql
BEGIN;

-- Dropar pelo nome REAL
ALTER TABLE job_canonical_roles
  DROP CONSTRAINT IF EXISTS job_canonical_roles_source_check;

-- Recriar com TODOS os valores legados + o novo
ALTER TABLE job_canonical_roles
  ADD CONSTRAINT job_canonical_roles_source_check
  CHECK (source IN (
    'seed',                -- legado preservado
    'llm_extractor',       -- legado preservado
    'cbo_mte_2002_seed',   -- legado preservado (2.694 linhas em prod, sprint CBO v6.6.6)
    'manual_admin',        -- legado preservado
    'opus_arbitration',    -- legado preservado
    'resume_extraction'    -- NOVO (objetivo desta migration)
  ));

COMMIT;
```

**Validação pós-migration:**

```sql
SELECT pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conname = 'job_canonical_roles_source_check';
-- Esperado: 6 valores na lista.
```

---

### §2.3 — Coluna `resumes.skills_with_confidence jsonb`

**Arquivo:** `03_resumes_skills_with_confidence.sql`

```sql
BEGIN;

ALTER TABLE resumes ADD COLUMN IF NOT EXISTS skills_with_confidence jsonb;

COMMENT ON COLUMN resumes.skills_with_confidence IS
'Snapshot do conjunto de skills extraídas do CV com confidence numérica do LLM, antes do hard gate. Usado pelo pipeline de análise para auditoria e UI de explainability. Formato: [{"name": "...", "confidence": 0.92}, ...]';

COMMIT;
```

---

### §2.4 — `lookup_canonical_skill_by_normalized_label` (normalize_skill_label + status pending)

**Arquivo:** `04_lookup_canonical_skill_by_normalized_label.sql`
**Objetivo:** localizar canonical de skill pelo label normalizado, batendo no índice funcional `uq_jcs_label_normalized` existente em produção (`USING btree (normalize_skill_label(label)) WHERE status='active' AND merged_into IS NULL`). Aceita `pending` em adição a `active` para evitar duplicatas em janelas de pipeline.
**Ground truth:** `job_canonical_skills` **não tem coluna `employer_id`** (DeepSeek #1 era falso positivo após Q4). O parâmetro `p_employer_id` é mantido na assinatura apenas para simetria com `reconcile_canonical_skill` (que aceita contexto de employer para registro de evento), mas a função de lookup não usa.
**Expressão canônica:** usar `normalize_skill_label(p_label)` no WHERE — `lower(trim())` causaria miss em skills com acento ou símbolo (`C++`, `Integração Contínua`) porque o índice é populado via `normalize_skill_label`.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION lookup_canonical_skill_by_normalized_label(
  p_label text,
  p_employer_id uuid DEFAULT NULL  -- reservado para simetria; não usado no lookup
)
RETURNS uuid
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
BEGIN
  IF p_label IS NULL OR length(trim(p_label)) = 0 THEN
    RETURN NULL;
  END IF;

  v_normalized := normalize_skill_label(p_label);

  IF v_normalized = '' THEN
    RETURN NULL;
  END IF;

  -- Bate no índice uq_jcs_label_normalized via expressão canônica
  SELECT id INTO v_id
  FROM job_canonical_skills
  WHERE normalize_skill_label(label) = v_normalized
    AND status IN ('active', 'pending')
    AND merged_into IS NULL
  ORDER BY
    CASE WHEN status = 'active' THEN 0 ELSE 1 END,
    created_at ASC
  LIMIT 1;

  RETURN v_id;
END;
$$;

COMMIT;
```

**Validação pós-migration:**

```sql
-- Função usa o índice (EXPLAIN deve mostrar Index Scan, não Seq Scan)
EXPLAIN SELECT id FROM job_canonical_skills
WHERE normalize_skill_label(label) = normalize_skill_label('C++')
  AND status IN ('active','pending') AND merged_into IS NULL;
```

---

### §2.5 — `lookup_canonical_role_by_normalized_label` + índice funcional dedicado

**Arquivo:** `05_lookup_canonical_role_by_normalized_label.sql`
**Objetivo:** análogo simétrico a §2.4 para JCR, usando `normalize_role_label()` (§2.0a) como expressão canônica. Cria também o índice funcional que o lookup vai utilizar.
**Ground truth:** `job_canonical_roles` **não tem coluna `employer_id`** (Q4 confirmou ambas JCS e JCR). Parâmetro `p_employer_id` reservado para simetria com `reconcile_canonical_role` (registro de evento).
**Assimetria intencional vs JCS:** JCR usa `canonical_label` (D-PS-08 documenta).

```sql
BEGIN;

-- Índice funcional para o lookup (dedicado ao status='active' + merged_into IS NULL)
CREATE UNIQUE INDEX IF NOT EXISTS uq_jcr_canonical_label_normalized
ON job_canonical_roles
USING btree (normalize_role_label(canonical_label))
WHERE status = 'active' AND merged_into IS NULL;

CREATE OR REPLACE FUNCTION lookup_canonical_role_by_normalized_label(
  p_canonical_label text,
  p_employer_id uuid DEFAULT NULL  -- reservado para simetria; não usado no lookup
)
RETURNS uuid
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
BEGIN
  IF p_canonical_label IS NULL OR length(trim(p_canonical_label)) = 0 THEN
    RETURN NULL;
  END IF;

  v_normalized := normalize_role_label(p_canonical_label);

  IF v_normalized = '' THEN
    RETURN NULL;
  END IF;

  -- Bate no índice uq_jcr_canonical_label_normalized via expressão canônica
  SELECT id INTO v_id
  FROM job_canonical_roles
  WHERE normalize_role_label(canonical_label) = v_normalized
    AND status IN ('active', 'pending')
    AND merged_into IS NULL
  ORDER BY
    CASE WHEN status = 'active' THEN 0 ELSE 1 END,
    created_at ASC
  LIMIT 1;

  RETURN v_id;
END;
$$;

COMMIT;
```

**Validação pós-migration:**

```sql
-- O índice existe e a função o utiliza
SELECT indexname, indexdef FROM pg_indexes
WHERE tablename = 'job_canonical_roles'
  AND indexname = 'uq_jcr_canonical_label_normalized';
-- Esperado: 1 linha com expressão normalize_role_label(canonical_label)
```

---

### §2.6 — `lookup_canonical_role_by_normalized_alias` corrigido

**Arquivo:** `06_lookup_canonical_role_by_normalized_alias.sql`
**Bloqueador resolvido:** A6 — usa `target_role_id`, `source_term`, `status='active'` (não `target_canonical_id`, `relation_type`, `normalized_source_label`).
**Defensiva:** filtro explícito `entity_type='role'` para evitar dependência de XOR perfeito no schema polimórfico de `taxonomy_relations` (ChatGPT #11).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION lookup_canonical_role_by_normalized_alias(
  p_term text
)
RETURNS uuid
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
BEGIN
  IF p_term IS NULL OR length(trim(p_term)) = 0 THEN
    RETURN NULL;
  END IF;

  v_normalized := normalize_role_label(p_term);

  IF v_normalized = '' THEN
    RETURN NULL;
  END IF;

  SELECT target_role_id INTO v_id
  FROM taxonomy_relations
  WHERE source_term = v_normalized
    AND entity_type = 'role'
    AND status = 'active'
    AND target_role_id IS NOT NULL
  ORDER BY layer ASC NULLS LAST, created_at ASC
  LIMIT 1;

  RETURN v_id;
END;
$$;

COMMIT;
```

**Nota sobre `source_term` em `taxonomy_relations`:** o ingestor (`lib/pipeline/*`) é responsável por gravar `source_term` já normalizado (`normalize_role_label` para role, `normalize_skill_label` para skill) antes do upsert — paridade SQL↔TS coberta por D-PS-54. Q8 confirmou que a base atual não tem entries com acentos preservados em `source_term` (zero linhas via regex de acento), o que é consistente com normalização prévia já aplicada.

---

### §2.6b — `lookup_canonical_skill_by_normalized_alias` (simétrico)

**Arquivo:** `06b_lookup_canonical_skill_by_normalized_alias.sql`
**Atende:** simetria skill↔role para resolução via alias (taxonomy_relations).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION lookup_canonical_skill_by_normalized_alias(
  p_term text
)
RETURNS uuid
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
BEGIN
  IF p_term IS NULL OR length(trim(p_term)) = 0 THEN
    RETURN NULL;
  END IF;

  v_normalized := normalize_skill_label(p_term);

  IF v_normalized = '' THEN
    RETURN NULL;
  END IF;

  SELECT target_skill_id INTO v_id
  FROM taxonomy_relations
  WHERE source_term = v_normalized
    AND entity_type = 'skill'
    AND status = 'active'
    AND target_skill_id IS NOT NULL
  ORDER BY layer ASC NULLS LAST, created_at ASC
  LIMIT 1;

  RETURN v_id;
END;
$$;

COMMIT;
```

---

### §2.7 — `resume_roles`: canonical_role_id FK + ALTER confidence + coluna gerada + UNIQUE regular

**Arquivo:** `07_resume_roles_overhaul.sql`
**Bloqueadores resolvidos:**
- **C-3 (Claude Code v2.7.1):** `resume_roles.confidence` é `smallint NOT NULL CHECK (0-100)` em prod. §3.3 grava float 0-1 → cast silencioso para 0 ou 1, perde precisão. Migração para `numeric(5,4)` escala 0-1.
- **M-1 (DeepSeek):** `onConflict` em índice funcional não é detectado pelo Supabase JS. Substituir por **coluna gerada `STORED`** alimentada por `normalize_role_label` + UNIQUE regular sobre `(resume_id, role_normalized)`.
- **H-2 (ChatGPT #3):** CHECK fixo `confidence >= 0.70` em `passed_hard_gate=true` contradiz `pipeline_config.role.hard_gate.min_confidence` calibrável. Hard gate é filtro TS (§3.3), não DB constraint.
- **D-PS-53:** preserva índice parcial UNIQUE `idx_resume_roles_one_primary` existente (`(resume_id) WHERE is_primary = true`) — não tocado por esta migration.

**Ground truth:** CHECK existente é `chk_resume_roles_confidence` (CHECK (confidence >= 0 AND confidence <= 100)). Não há índice funcional pré-existente sobre `lower(trim(role))`. Não há UNIQUE pré-existente sobre `(resume_id, role)`.

```sql
BEGIN;

-- 1. FK canonical_role_id (cravando o que a v2.7.1 já planejava)
ALTER TABLE resume_roles
  ADD COLUMN IF NOT EXISTS canonical_role_id uuid
  REFERENCES job_canonical_roles(id) ON DELETE SET NULL;

CREATE INDEX IF NOT EXISTS idx_resume_roles_canonical_role_id
  ON resume_roles(canonical_role_id)
  WHERE canonical_role_id IS NOT NULL;

-- 2. ALTER confidence smallint(0-100) → numeric(5,4)(0-1)
--    Base resetada pós-benchmark — sem dados a migrar. CASE defensivo (B-1) tolera
--    qualquer linha legada se a base não estiver 100% limpa: valores >1 são tratados como
--    escala 0-100 e divididos; valores <=1 já estão em escala 0-1 (caso impossível em smallint
--    legado, mas safe). NULL preserva NULL.
ALTER TABLE resume_roles
  DROP CONSTRAINT IF EXISTS chk_resume_roles_confidence;

ALTER TABLE resume_roles
  ALTER COLUMN confidence TYPE numeric(5,4)
  USING (
    CASE
      WHEN confidence IS NULL THEN NULL
      WHEN confidence > 1 THEN (confidence::numeric / 100.0)::numeric(5,4)
      ELSE confidence::numeric(5,4)
    END
  );

ALTER TABLE resume_roles
  ADD CONSTRAINT chk_resume_roles_confidence
  CHECK (confidence >= 0 AND confidence <= 1);

-- 3. Coluna gerada role_normalized + UNIQUE regular (M-1)
ALTER TABLE resume_roles
  ADD COLUMN IF NOT EXISTS role_normalized text
  GENERATED ALWAYS AS (normalize_role_label(role)) STORED;

ALTER TABLE resume_roles
  ADD CONSTRAINT uq_resume_roles_resume_role_normalized
  UNIQUE (resume_id, role_normalized);

COMMIT;
```

**Validação pós-migration:**

```sql
-- confidence agora numeric(5,4)
SELECT data_type, numeric_precision, numeric_scale
FROM information_schema.columns
WHERE table_name = 'resume_roles' AND column_name = 'confidence';
-- Esperado: numeric, 5, 4

-- coluna gerada existe
SELECT column_name, is_generated, generation_expression
FROM information_schema.columns
WHERE table_name = 'resume_roles' AND column_name = 'role_normalized';
-- Esperado: role_normalized, ALWAYS, normalize_role_label(role)

-- UNIQUE constraint regular
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'resume_roles'::regclass AND contype = 'u';
-- Esperado: 1 linha — uq_resume_roles_resume_role_normalized UNIQUE (resume_id, role_normalized)
```

---

### §2.8 — `resume_roles ADD passed_hard_gate` (sem CHECK hardcoded)

**Arquivo:** `08_resume_roles_passed_hard_gate.sql`
**Bloqueador resolvido (H-2):** o CHECK fixo `confidence >= 0.70` contradiz `pipeline_config.role.hard_gate.min_confidence` calibrável. O hard gate é aplicado no TS (`§3.3 evaluateHardGate`), não como invariante de banco. A tabela armazena tanto registros que passaram quanto reprovados (decisão de produto cravada nesta sprint — auditoria simétrica skill↔role).

```sql
BEGIN;

ALTER TABLE resume_roles
  ADD COLUMN IF NOT EXISTS passed_hard_gate boolean NOT NULL DEFAULT false;

CREATE INDEX IF NOT EXISTS idx_resume_roles_passed_hard_gate
  ON resume_roles(resume_id, passed_hard_gate)
  WHERE passed_hard_gate = true;

COMMIT;
```

**Justificativa do índice parcial:** queries operacionais (motor de comparação, painel admin) filtram quase sempre por `passed_hard_gate = true`. Índice parcial reduz tamanho. Registros reprovados continuam consultáveis via seq scan (volume estimado <50/CV).

---

### §2.8b — `resume_skills` (tabela nova, espelha `resume_roles` corrigida)

**Arquivo:** `08b_resume_skills_table.sql`
**Atende:** simetria skill↔role na camada de extração do currículo. Tabela nasce já com schema corrigido (confidence numeric, coluna gerada, UNIQUE regular, sem CHECK hardcoded de hard gate).
**Distinção:** `analysis_skill_matches` é cache de match calculado por análise (com `frequency`, `matched`, `contribution`, `match_status` todos NOT NULL); `resume_skill_enrichments` é log user-facing de validação. Nenhuma das duas pode armazenar skills brutas pós-extração.

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS resume_skills (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  resume_id uuid NOT NULL REFERENCES resumes(id) ON DELETE CASCADE,
  skill text NOT NULL,
  skill_normalized text GENERATED ALWAYS AS (normalize_skill_label(skill)) STORED,
  confidence numeric(5,4) NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
  passed_hard_gate boolean NOT NULL DEFAULT false,
  canonical_skill_id uuid REFERENCES job_canonical_skills(id) ON DELETE SET NULL,
  source text NOT NULL DEFAULT 'resume_extraction',
  created_at timestamptz NOT NULL DEFAULT NOW(),

  CONSTRAINT uq_resume_skills_resume_skill_normalized
    UNIQUE (resume_id, skill_normalized)
);

CREATE INDEX IF NOT EXISTS idx_resume_skills_canonical_skill_id
  ON resume_skills (canonical_skill_id)
  WHERE canonical_skill_id IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_resume_skills_passed_hard_gate
  ON resume_skills (resume_id, passed_hard_gate)
  WHERE passed_hard_gate = true;

COMMENT ON TABLE resume_skills IS
'Skills extraídas do currículo (aprovados E reprovados pelo hard gate), paralela a resume_roles. Distinto de analysis_skill_matches (cache de match calculado por análise contra função) e de resume_skill_enrichments (log de validação user-facing). Coluna skill_normalized é STORED via normalize_skill_label() para suportar UNIQUE regular.';

COMMIT;
```

**Validação pós-migration:**

```sql
-- Schema confere
SELECT column_name, data_type, is_generated, generation_expression
FROM information_schema.columns
WHERE table_name = 'resume_skills'
ORDER BY ordinal_position;

-- UNIQUE regular (não funcional) sobre resume_id + skill_normalized
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'resume_skills'::regclass AND contype = 'u';
```

**Distinções importantes (evita confusão com tabelas vizinhas de skill):**
- `resume_skills` (esta) — skills brutas extraídas do currículo (aprovadas E reprovadas), com `canonical_skill_id` resolvido apenas para aprovadas. Análoga a `resume_roles`.
- `analysis_skill_matches` — cache de match calculado por análise contra função canonical (tem `frequency`, `matched`, `contribution`, `match_status`). Populada pelo motor de comparação, não pelo extrator.
- `resume_skill_enrichments` — log de enriquecimento user-facing (sugestões aceitas/rejeitadas pelo usuário na tela de análise). Tem `validation_status` consumido pelo Painel B7 #5.

---

### §2.9 — Trigger `latest_posted_at` em JCR (recompute via MAX, robusto a UPDATE/DELETE/reassign)

**Arquivo:** `09_jcr_latest_posted_at_trigger.sql`
**Nota descritiva:** primeira implementação do padrão `latest_posted_at` em JCR. JCS já usa cálculo embutido em `fn_jps_recompute_jcs` via `SELECT MAX(posted_at) ... WHERE curation_status='curated'` (Q-Bloco-3 confirmou). Esta migration introduz o mecanismo equivalente em JCR.
**Bloqueador resolvido (AL-4 / ChatGPT #4):** versão anterior usava `GREATEST(old, new)`, frágil a (1) UPDATE com posted_at menor, (2) reassign de canonical_role_id, (3) DELETE de vaga, (4) reverter `curation_status` para `pending`. Trigger novo recomputa via `SELECT MAX()` cobrindo OLD e NEW role_id, filtrando `curation_status='curated'` (simetria com fn_jps_recompute_jcs).

```sql
BEGIN;

ALTER TABLE job_canonical_roles
  ADD COLUMN IF NOT EXISTS latest_posted_at timestamptz;

CREATE OR REPLACE FUNCTION fn_jcr_recompute_latest_posted_at(p_role_id uuid)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
BEGIN
  IF p_role_id IS NULL THEN
    RETURN;
  END IF;

  UPDATE job_canonical_roles
  SET latest_posted_at = (
    SELECT MAX(jp.posted_at)
    FROM job_postings jp
    WHERE jp.canonical_role_id = p_role_id
      AND jp.curation_status = 'curated'
  )
  WHERE id = p_role_id;
END;
$$;

CREATE OR REPLACE FUNCTION fn_jcr_update_latest_posted_at()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
BEGIN
  -- INSERT: recomputa NEW.canonical_role_id
  IF TG_OP = 'INSERT' THEN
    PERFORM fn_jcr_recompute_latest_posted_at(NEW.canonical_role_id);
    RETURN NULL;
  END IF;

  -- DELETE: recomputa OLD.canonical_role_id
  IF TG_OP = 'DELETE' THEN
    PERFORM fn_jcr_recompute_latest_posted_at(OLD.canonical_role_id);
    RETURN NULL;
  END IF;

  -- UPDATE: cobre 4 cenários (posted_at mudou, canonical_role_id mudou, curation_status mudou, ou combinação).
  -- Recompute em OLD e NEW; se forem iguais, segunda chamada é idempotente.
  IF TG_OP = 'UPDATE' THEN
    IF OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id THEN
      PERFORM fn_jcr_recompute_latest_posted_at(OLD.canonical_role_id);
    END IF;
    PERFORM fn_jcr_recompute_latest_posted_at(NEW.canonical_role_id);
    RETURN NULL;
  END IF;

  RETURN NULL;
END;
$$;

DROP TRIGGER IF EXISTS trg_jp_update_jcr_latest_posted_at ON job_postings;

-- Dispara em todas as mutações relevantes; função decide o que recomputar
CREATE TRIGGER trg_jp_update_jcr_latest_posted_at
AFTER INSERT OR UPDATE OF posted_at, canonical_role_id, curation_status OR DELETE
ON job_postings
FOR EACH ROW
EXECUTE FUNCTION fn_jcr_update_latest_posted_at();

-- Backfill inicial — apenas vagas curadas
UPDATE job_canonical_roles jcr
SET latest_posted_at = (
  SELECT MAX(jp.posted_at)
  FROM job_postings jp
  WHERE jp.canonical_role_id = jcr.id
    AND jp.curation_status = 'curated'
);

COMMIT;
```

**Validação pós-migration:** 4 cenários cobertos pelo smoke test S-LP (§7.1.1).

---

### §2.10 — `auto_assign_family_to_role` (renomeada + SECURITY DEFINER + search_path + evento de sucesso)

**Arquivo:** `10_auto_assign_family_to_role.sql`
**Substitui:** `auto_assign_family_to_canonical`.
**Bloqueadores resolvidos:** S-SYM-3-BÔNUS (é **função regular** chamada via `PERFORM`, não trigger), parte do rename §2.21.
**Ordem crítica:** esta migration vem ANTES de `fn_promote_role_on_threshold` (§2.12) porque a função de promoção faz `PERFORM auto_assign_family_to_role(...)` em seu corpo.

Função regular (não trigger). Algoritmo família-por-membros preservado integralmente do banco:

1. Resolve `domain_id` do canonical via `canonical_role_domain_links` (primary primeiro, depois highest confidence)
2. Resolve `industry_normalized` dominante via `job_postings` → `employers.industry_normalized` (voto majoritário)
3. Encontra até 5 canonicals similares ao role de entrada (cosine ≥ 0.75)
4. Para cada família ativa com ao menos 1 canonical associado:
   - +1.0 se algum dos 5 similares já está nessa família
   - +0.5 se a família tem 2+ canonicals compartilhando o mesmo `domain_id` do role de entrada
   - +0.5 se a família tem canonicals com vagas no mesmo `industry_normalized` do role de entrada
5. Vencedora: score ≥ 1.5 → INSERT em `taxonomy_family_canonicals`
6. Falha: emite evento `role_orphan_no_family_assigned`

**Mudanças desta migration vs função viva:**
- Nome da função: `auto_assign_family_to_canonical` → `auto_assign_family_to_role`
- Nome do parâmetro: `p_canonical_id` → `p_role_id`
- Variáveis internas renomeadas para coerência semântica (`v_canonical_*` → `v_role_*`)
- Adicionado `SECURITY DEFINER`
- Adicionado `SET search_path TO 'public', 'pg_temp'`
- **Adicionada emissão de evento `family_assigned_auto`** no caminho de sucesso (instrumentação que faltava — função viva só emite no caminho de falha)
- Demais aspectos do algoritmo, queries, threshold, e branches de retorno: idênticos ao corpo vivo

```sql
BEGIN;

CREATE OR REPLACE FUNCTION auto_assign_family_to_role(
  p_role_id uuid,
  p_known_active boolean DEFAULT false
)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_status text;
  v_role_embedding vector(768);
  v_role_industry text;
  v_role_domain_id uuid;
  v_winning_family_id uuid;
  v_winning_score numeric;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
BEGIN
  -- 1. Carregar status + embedding do role
  SELECT status, embedding INTO v_status, v_role_embedding
  FROM job_canonical_roles WHERE id = p_role_id;

  IF NOT p_known_active AND v_status != 'active' THEN RETURN; END IF;
  IF v_role_embedding IS NULL THEN RETURN; END IF;

  -- 2. Resolver domain_id (primary primeiro, depois highest confidence)
  SELECT domain_id INTO v_role_domain_id
  FROM canonical_role_domain_links
  WHERE canonical_role_id = p_role_id AND is_primary = TRUE
  LIMIT 1;

  IF v_role_domain_id IS NULL THEN
    SELECT domain_id INTO v_role_domain_id
    FROM canonical_role_domain_links
    WHERE canonical_role_id = p_role_id
    ORDER BY confidence DESC NULLS LAST
    LIMIT 1;
  END IF;

  -- 3. Resolver industry dominante via vagas → employers (voto majoritário)
  SELECT e.industry_normalized INTO v_role_industry
  FROM job_postings jp
  JOIN employers e ON e.id = jp.employer_id
  WHERE jp.canonical_role_id = p_role_id
    AND e.industry_normalized IS NOT NULL
  GROUP BY e.industry_normalized
  ORDER BY COUNT(*) DESC
  LIMIT 1;

  -- 4. Calcular score por família e selecionar vencedora
  WITH similar_canonicals AS (
    SELECT jcr.id, 1 - (jcr.embedding <=> v_role_embedding) AS similarity
    FROM job_canonical_roles jcr
    WHERE jcr.id != p_role_id
      AND jcr.status = 'active'
      AND jcr.embedding IS NOT NULL
      AND 1 - (jcr.embedding <=> v_role_embedding) >= 0.75
    ORDER BY similarity DESC
    LIMIT 5
  ),
  family_scores AS (
    SELECT
      tf.id AS family_id,
      (CASE
        WHEN COUNT(DISTINCT tfc.canonical_role_id)
          FILTER (WHERE tfc.canonical_role_id IN (SELECT id FROM similar_canonicals)) > 0
        THEN 1.0
        ELSE 0.0
      END)
      +
      (CASE
        WHEN v_role_domain_id IS NOT NULL AND EXISTS (
          SELECT 1 FROM taxonomy_family_canonicals tfc2
          JOIN canonical_role_domain_links crdl ON crdl.canonical_role_id = tfc2.canonical_role_id
          WHERE tfc2.family_id = tf.id AND crdl.domain_id = v_role_domain_id
          GROUP BY tfc2.family_id HAVING COUNT(*) >= 2
        )
        THEN 0.5
        ELSE 0.0
      END)
      +
      (CASE
        WHEN v_role_industry IS NOT NULL AND EXISTS (
          SELECT 1 FROM taxonomy_family_canonicals tfc3
          JOIN job_postings jp ON jp.canonical_role_id = tfc3.canonical_role_id
          JOIN employers e ON e.id = jp.employer_id
          WHERE tfc3.family_id = tf.id AND e.industry_normalized = v_role_industry
          LIMIT 1
        )
        THEN 0.5
        ELSE 0.0
      END)
      AS score
    FROM taxonomy_families tf
    JOIN taxonomy_family_canonicals tfc ON tfc.family_id = tf.id
    WHERE tf.status = 'active'
    GROUP BY tf.id
  )
  SELECT family_id, score INTO v_winning_family_id, v_winning_score
  FROM family_scores
  WHERE score >= 1.5
  ORDER BY score DESC
  LIMIT 1;

  -- 5. Persistir resultado
  IF v_winning_family_id IS NOT NULL THEN
    INSERT INTO taxonomy_family_canonicals (family_id, canonical_role_id)
    VALUES (v_winning_family_id, p_role_id)
    ON CONFLICT (family_id, canonical_role_id) DO NOTHING;

    BEGIN
      INSERT INTO events (
        event_name, resource_type, resource_id,
        actor, actor_id, new_state, reason, metadata
      ) VALUES (
        'family_assigned_auto',
        'job_canonical_role',
        p_role_id,
        'system',
        v_actor_id,
        jsonb_build_object('family_id', v_winning_family_id),
        'Família atribuída automaticamente por score >= 1.5',
        jsonb_build_object(
          'score', v_winning_score,
          'threshold', 1.5,
          'has_domain', v_role_domain_id IS NOT NULL,
          'has_industry', v_role_industry IS NOT NULL
        )
      );
    EXCEPTION WHEN OTHERS THEN
      RAISE WARNING '[auto_assign_family_to_role] family_assigned_auto event insert failed: %', SQLERRM;
    END;
  ELSE
    BEGIN
      INSERT INTO events (
        event_name, resource_type, resource_id,
        actor, actor_id, previous_state, new_state, reason
      ) VALUES (
        'role_orphan_no_family_assigned',
        'job_canonical_role',
        p_role_id,
        'system',
        v_actor_id,
        jsonb_build_object('status', v_status),
        jsonb_build_object(
          'has_embedding', v_role_embedding IS NOT NULL,
          'has_domain', v_role_domain_id IS NOT NULL,
          'has_industry', v_role_industry IS NOT NULL,
          'best_score_below_threshold', COALESCE(v_winning_score, 0)
        ),
        'Score insuficiente para atribuição automática (threshold 1.5)'
      );
    EXCEPTION WHEN OTHERS THEN
      RAISE WARNING '[auto_assign_family_to_role] role_orphan_no_family_assigned event insert failed: %', SQLERRM;
    END;
  END IF;
END;
$$;

COMMENT ON FUNCTION auto_assign_family_to_role IS
'Tenta atribuir família ao role via algoritmo família-por-membros: encontra 5 canonicals similares (cosine >= 0.75), soma 1.0 se algum está na família + 0.5 se 2+ canonicals da família compartilham domain_id + 0.5 se família tem vagas no mesmo industry_normalized. Threshold 1.5. Em sucesso emite family_assigned_auto; em falha emite role_orphan_no_family_assigned. INSERTs em events envolvidos em BEGIN/EXCEPTION (H-1) para não abortar PERFORM caller em caso de FK/constraint violation no events.';

COMMIT;
```

**Comportamento esperado em base resetada:** com 0 canonicals ativos pós-reset, qualquer role novo terá 0 canonicals similares e nenhuma família terá membros para compartilhar domain/industry — todo role recém-promovido cai no caminho `role_orphan_no_family_assigned`. A cobertura de família passa a depender do Opus via `findOrCreateFamilyAndLink` (caminho independente, processadores `orphan-canonical` e `taxonomy-relation`). Esse é o comportamento operacional pretendido durante o cold start; não há ação corretiva necessária nesta sprint.

---

### §2.11 — `fn_retire_role_on_zero_vacancy` (renomeada + corrigida)

**Arquivo:** `11_fn_retire_role_on_zero_vacancy.sql`
**Bloqueador resolvido:** A5 (schema events correto), e parte do rename §2.21.
**Substitui:** `fn_retire_canonical_on_zero_vacancy`.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION fn_retire_role_on_zero_vacancy()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $$
BEGIN
  IF pg_trigger_depth() > 1 THEN
    RETURN NULL;
  END IF;

  IF NEW.vacancy_count != 0
     OR NEW.status NOT IN ('active', 'pending') THEN
    RETURN NULL;
  END IF;

  IF NEW.source = 'cbo_mte_2002_seed' THEN
    RETURN NULL;
  END IF;

  PERFORM retire_canonical(NEW.id, 'zero_vacancy_count');

  RETURN NULL;
END;
$$;

-- A criação/recriação do trigger associado fica em §2.21 (rename atômico)

COMMIT;
```

---

### §2.12 — `fn_promote_role_on_threshold` (renomeada, sem gate de mediana, corrigida)

**Arquivo:** `12_fn_promote_role_on_threshold.sql`
**Bloqueadores resolvidos:** A5 (schema events), S-SYM-3 (RETURN NULL ao invés de RETURN NEW), parte da Opção C (já era simétrica em não ter gate; preserva).
**Substitui:** `fn_promote_canonical_on_threshold`.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION fn_promote_role_on_threshold()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_min_vacancies int;
  v_min_employers int;
  v_lookback_days int;
  v_has_recent_vacancy boolean;
  v_is_resurrection boolean;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
BEGIN
  IF pg_trigger_depth() > 1 THEN
    RETURN NULL;
  END IF;

  IF NEW.status NOT IN ('pending', 'retired')
     OR OLD.status NOT IN ('pending', 'retired') THEN
    RETURN NULL;
  END IF;

  SELECT COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.min_vacancies'), 3),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.min_distinct_employers'), 2),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.lookback_days'), 180)
  INTO v_min_vacancies, v_min_employers, v_lookback_days;

  IF NEW.vacancy_count < v_min_vacancies
     OR NEW.distinct_sources_count < v_min_employers THEN
    RETURN NULL;
  END IF;

  SELECT EXISTS(
    SELECT 1 FROM job_postings jp
    WHERE jp.canonical_role_id = NEW.id
      AND jp.posted_at >= NOW() - make_interval(days => v_lookback_days)
      AND jp.curation_status = 'curated'   -- M-6: consistência com vacancy_count/distinct_sources_count
  ) INTO v_has_recent_vacancy;

  IF NOT v_has_recent_vacancy THEN
    BEGIN
      INSERT INTO events (
        event_name, resource_type, resource_id,
        actor, actor_id, previous_state, new_state, reason, metadata
      ) VALUES (
        'role_promotion_deferred_archaeological',
        'job_canonical_role',
        NEW.id,
        'system',
        v_actor_id,
        jsonb_build_object('status', OLD.status, 'vacancy_count', NEW.vacancy_count),
        jsonb_build_object('status', OLD.status, 'reason', 'no_recent_postings'),
        'Volume atingido mas nenhuma vaga recente — promoção deferida',
        jsonb_build_object(
          'lookback_days', v_lookback_days,
          'distinct_sources_count', NEW.distinct_sources_count
        )
      );
    EXCEPTION WHEN OTHERS THEN
      RAISE WARNING '[fn_promote_role_on_threshold] archaeological event insert failed: %', SQLERRM;
    END;
    RETURN NULL;
  END IF;

  v_is_resurrection := (OLD.status = 'retired');

  UPDATE job_canonical_roles
  SET status = 'active',
      promoted_at = COALESCE(promoted_at, NOW()),
      confidence_median_at_promotion = COALESCE(
        confidence_median_at_promotion, NEW.confidence_median
      ),
      vacancy_count_at_promotion = NEW.vacancy_count,
      distinct_sources_count_at_promotion = NEW.distinct_sources_count,
      retired_at = NULL,
      retire_reason = NULL
  WHERE id = NEW.id
    AND status IN ('pending', 'retired');

  BEGIN
    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, previous_state, new_state, reason, metadata
    ) VALUES (
      'role_promoted_dynamic',
      'job_canonical_role',
      NEW.id,
      'system',
      v_actor_id,
      jsonb_build_object('status', OLD.status),
      jsonb_build_object('status', 'active'),
      CASE WHEN v_is_resurrection THEN 'Ressuscitação automática por volume' ELSE 'Promoção automática por volume' END,
      jsonb_build_object(
        'vacancy_count', NEW.vacancy_count,
        'distinct_sources_count', NEW.distinct_sources_count,
        'confidence_median', NEW.confidence_median,
        'is_resurrection', v_is_resurrection
      )
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[fn_promote_role_on_threshold] role_promoted_dynamic event insert failed: %', SQLERRM;
  END;

  BEGIN
    PERFORM auto_assign_family_to_role(NEW.id, TRUE);
  EXCEPTION WHEN OTHERS THEN
    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, reason, metadata
    ) VALUES (
      'auto_assign_family_failed_in_promotion',
      'job_canonical_role',
      NEW.id,
      'system',
      v_actor_id,
      'Falha em auto_assign_family_to_role durante promoção',
      jsonb_build_object('error', SQLERRM)
    );
  END;

  RETURN NULL;
END;
$$;

-- A criação/recriação do trigger associado fica em §2.21

COMMIT;
```

**Nota crítica:** a função **não** seta `needs_opus_review` aqui. A flagging para auditoria Opus é responsabilidade dos triggers `fn_flag_needs_opus_review_jcr` modificados em §2.22 (que agora escutam mudanças de `status`, fechando o gap latente identificado).

---

### §2.13 — `reconcile_canonical_role` (blacklist check + sem default em p_source)

**Arquivo:** `13_reconcile_canonical_role.sql`
**Bloqueadores resolvidos:**
- A5 (schema events) — preservado da v2.7.1
- B6 — preservado (lookup usa parâmetro de contexto, não coluna armazenada)
- **M-7 (DeepSeek #11):** consulta `taxonomy_blacklist` antes de criar canonical novo. Se o `normalize_role_label` do label estiver banido, lança exceção `P0001 label_blacklisted` que o caller TS captura e descarta o item silenciosamente.
- **Q4 ground truth:** `job_canonical_roles` não tem coluna `employer_id`. INSERT removeu a coluna; parâmetro mantido para registro no evento.
- **CR-1 (ChatGPT #14, rodada v2.8):** `p_source` perdeu `DEFAULT` para evitar criação silenciosa de canonical com source desalinhado da constraint. A v2.8 tinha `DEFAULT 'job_posting'`, mas §2.2 nova lista não inclui `'job_posting'`. Em vez de adicionar valor semanticamente vazio ao CHECK ou cair para `'llm_extractor'` que viraria atribuição implícita, a decisão (Claude Code) é remover o default — caller obrigado a especificar. Caller esquecido recebe `RAISE EXCEPTION 'p_source is required...'` em vez de gravar source errado silenciosamente.
- **CR-3 (DeepSeek, rodada v3 → v3.1):** PostgreSQL exige que parâmetros sem default venham ANTES dos parâmetros com default na assinatura. A v3 trazia `p_canonical_label text, p_employer_id uuid DEFAULT NULL, p_source text` — `CREATE OR REPLACE FUNCTION` falharia com `input parameters after one with a default value must also have defaults`. Ordem corrigida para `p_canonical_label text, p_source text, p_employer_id uuid DEFAULT NULL`. Callers TS via Supabase RPC usam named args (`p_source: 'resume_extraction'`, `p_employer_id: null`), portanto ordem dos parâmetros é irrelevante para o callsite — apenas a definição SQL precisava ser corrigida.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION reconcile_canonical_role(
  p_canonical_label text,
  p_source text,                   -- OBRIGATÓRIO sem DEFAULT: caller deve especificar source explicitamente (CR-1 v2.9). Posicionado ANTES de p_employer_id porque PostgreSQL exige parâmetros sem default antes dos com default (CR-3 v3.1).
  p_employer_id uuid DEFAULT NULL  -- opcional
)
RETURNS uuid
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
BEGIN
  IF p_canonical_label IS NULL OR length(trim(p_canonical_label)) = 0 THEN
    RAISE EXCEPTION 'canonical_label cannot be null or empty';
  END IF;

  IF p_source IS NULL OR length(trim(p_source)) = 0 THEN
    RAISE EXCEPTION 'p_source is required (no default — caller must specify the source)';
  END IF;

  v_normalized := normalize_role_label(p_canonical_label);

  IF v_normalized = '' THEN
    RAISE EXCEPTION 'canonical_label normalized to empty';
  END IF;

  -- Tenta lookup direto (label normalizado)
  v_id := lookup_canonical_role_by_normalized_label(p_canonical_label, p_employer_id);
  IF v_id IS NOT NULL THEN
    RETURN v_id;
  END IF;

  -- Tenta lookup por alias (taxonomy_relations)
  v_id := lookup_canonical_role_by_normalized_alias(p_canonical_label);
  IF v_id IS NOT NULL THEN
    RETURN v_id;
  END IF;

  -- Antes de criar canonical novo, consulta blacklist
  IF EXISTS (
    SELECT 1 FROM taxonomy_blacklist
    WHERE entity_type = 'role'
      AND normalize_role_label(label) = v_normalized
  ) THEN
    RAISE EXCEPTION 'label_blacklisted: %', v_normalized USING ERRCODE = 'P0001';
  END IF;

  -- Criar canonical novo (sem coluna employer_id — não existe em JCR)
  INSERT INTO job_canonical_roles (
    canonical_label, status, source, vacancy_count, distinct_sources_count
  ) VALUES (
    trim(p_canonical_label), 'pending', p_source, 0, 0
  )
  RETURNING id INTO v_id;

  BEGIN
    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, new_state, reason, metadata
    ) VALUES (
      'canonical_role_created',
      'job_canonical_role',
      v_id,
      'system',
      v_actor_id,
      jsonb_build_object('status', 'pending', 'canonical_label', trim(p_canonical_label)),
      'Canonical criado por reconcile',
      jsonb_build_object('source', p_source, 'employer_id', p_employer_id)
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[reconcile_canonical_role] canonical_role_created event insert failed: %', SQLERRM;
  END;

  RETURN v_id;
END;
$$;

COMMIT;
```

---

### §2.14 — `reconcile_canonical_skill` (blacklist check + sem default em p_source)

**Arquivo:** `14_reconcile_canonical_skill.sql`
**Bloqueadores resolvidos:**
- A5, B6, A4 — preservados da v2.7.1
- **M-7:** simétrico a §2.13, consulta `taxonomy_blacklist` antes de criar.
- **Q4 ground truth:** `job_canonical_skills` não tem coluna `employer_id`.
- **CR-1 (ChatGPT #14, rodada v2.8):** simétrico a §2.13 — `p_source` sem `DEFAULT`. A v2.8 tinha `DEFAULT 'job_posting_skill'`, valor que NÃO está na nova lista do `job_canonical_skills_source_check` (§2.1). Caller obrigatório fail-fast em vez de criar silenciosamente com source errado.
- **CR-3 (DeepSeek, rodada v3 → v3.1):** simétrico a §2.13 — ordem corrigida para `p_label text, p_source text, p_employer_id uuid DEFAULT NULL` para respeitar regra do PostgreSQL (parâmetros sem default antes dos com default).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION reconcile_canonical_skill(
  p_label text,
  p_source text,                   -- OBRIGATÓRIO sem DEFAULT: caller deve especificar source explicitamente (CR-1 v2.9). Posicionado ANTES de p_employer_id porque PostgreSQL exige parâmetros sem default antes dos com default (CR-3 v3.1).
  p_employer_id uuid DEFAULT NULL  -- opcional
)
RETURNS uuid
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
  v_normalized text;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
BEGIN
  IF p_label IS NULL OR length(trim(p_label)) = 0 THEN
    RAISE EXCEPTION 'label cannot be null or empty';
  END IF;

  IF p_source IS NULL OR length(trim(p_source)) = 0 THEN
    RAISE EXCEPTION 'p_source is required (no default — caller must specify the source)';
  END IF;

  v_normalized := normalize_skill_label(p_label);

  IF v_normalized = '' THEN
    RAISE EXCEPTION 'label normalized to empty';
  END IF;

  -- Tenta lookup direto (label normalizado)
  v_id := lookup_canonical_skill_by_normalized_label(p_label, p_employer_id);
  IF v_id IS NOT NULL THEN
    RETURN v_id;
  END IF;

  -- Tenta lookup por alias (taxonomy_relations)
  v_id := lookup_canonical_skill_by_normalized_alias(p_label);
  IF v_id IS NOT NULL THEN
    RETURN v_id;
  END IF;

  -- Antes de criar canonical novo, consulta blacklist
  IF EXISTS (
    SELECT 1 FROM taxonomy_blacklist
    WHERE entity_type = 'skill'
      AND normalize_skill_label(label) = v_normalized
  ) THEN
    RAISE EXCEPTION 'label_blacklisted: %', v_normalized USING ERRCODE = 'P0001';
  END IF;

  -- Criar canonical novo (sem coluna employer_id — não existe em JCS)
  INSERT INTO job_canonical_skills (
    label, status, source, vacancy_count, distinct_sources_count
  ) VALUES (
    trim(p_label), 'pending', p_source, 0, 0
  )
  RETURNING id INTO v_id;

  BEGIN
    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, new_state, reason, metadata
    ) VALUES (
      'canonical_skill_created',
      'canonical_skill',
      v_id,
      'system',
      v_actor_id,
      jsonb_build_object('status', 'pending', 'label', trim(p_label)),
      'Canonical criado por reconcile',
      jsonb_build_object('source', p_source, 'employer_id', p_employer_id)
    );
  EXCEPTION WHEN OTHERS THEN
    RAISE WARNING '[reconcile_canonical_skill] canonical_skill_created event insert failed: %', SQLERRM;
  END;

  RETURN v_id;
END;
$$;

COMMIT;
```

---

### §2.15 — Adopção de `resolve_active_canonical_by_slug`

**Arquivo:** `15_resolve_active_canonical_by_slug.sql`

**Precheck obrigatório antes desta migration (D-3, Claude Code Ressalva 2):** a função aceita `entity_type='skill'` e referencia `job_canonical_skills.slug` e `job_canonical_skills.merged_into`. Confirmar via ground truth que ambas as colunas existem em JCS antes de criar a função — se não existirem, qualquer caller que invoque o branch `skill` recebe `ERROR: column "slug" does not exist` em runtime (erro técnico de Postgres, não erro semântico legível). Em paralelo, `lookup_canonical_skill_by_normalized_label` (§2.4) e `reconcile_canonical_skill` (§2.14) também usam `merged_into IS NULL` em JCS, portanto a premissa é compartilhada por várias migrations da sprint.

```sql
-- Precheck: confirmar colunas em JCS antes de prosseguir
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'job_canonical_skills'
  AND column_name IN ('slug', 'merged_into')
ORDER BY column_name;
-- Esperado: 2 linhas (slug text, merged_into uuid). Confirmadas via ground truth da
-- sprint paridade-skills v11 — JCS herda essas colunas do template comum a JCR.
-- Se retornar < 2 linhas, ABORTAR e investigar antes de aplicar §2.15, §2.4, §2.14.
```

```sql
BEGIN;

CREATE OR REPLACE FUNCTION resolve_active_canonical_by_slug(
  p_slug text,
  p_entity_type text DEFAULT 'role'
)
RETURNS uuid
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_id uuid;
BEGIN
  IF p_slug IS NULL OR length(trim(p_slug)) = 0 THEN
    RETURN NULL;
  END IF;

  IF p_entity_type = 'role' THEN
    SELECT id INTO v_id
    FROM job_canonical_roles
    WHERE slug = trim(p_slug)
      AND status IN ('active', 'pending')
      AND merged_into IS NULL
    ORDER BY CASE WHEN status = 'active' THEN 0 ELSE 1 END, created_at ASC
    LIMIT 1;
  ELSIF p_entity_type = 'skill' THEN
    SELECT id INTO v_id
    FROM job_canonical_skills
    WHERE slug = trim(p_slug)
      AND status IN ('active', 'pending')
      AND merged_into IS NULL
    ORDER BY CASE WHEN status = 'active' THEN 0 ELSE 1 END, created_at ASC
    LIMIT 1;
  ELSE
    RAISE EXCEPTION 'invalid entity_type: %', p_entity_type;
  END IF;

  RETURN v_id;
END;
$$;

COMMIT;
```

---

### §2.16 — View `v_merge_audit_history` UNION ALL

**Arquivo:** `16_v_merge_audit_history.sql`
**Bloqueador resolvido:** B10 + S-SYM-5 (UNION ALL real cobrindo role + skill com `entity_type` derivado).

**Caminho de join correto:** `opus_arbitration_outcomes.item_id` aponta para `canonical_*_merge_candidates.id` quando `item_type = 'merge_candidate'` (único valor válido para merges no CHECK constraint). Não aponta diretamente para `*_merge_decisions.id`. Para conectar uma decisão de merge à arbitragem Opus correspondente, é necessário pontear via `canonical_*_merge_candidates` usando a chave simétrica `(canonical_a_id, canonical_b_id)` que pode aparecer em qualquer ordem vs `(source_id, target_id)` em decisions. Além disso, `canonical_*_merge_candidates` já tem `opus_decision` e `opus_reasoning` embutidos — `opus_arbitration_outcomes` só agrega o que falta: `cost_usd`, `input_tokens`, `output_tokens`.

```sql
BEGIN;

DROP VIEW IF EXISTS v_merge_audit_history;

CREATE VIEW v_merge_audit_history AS
SELECT
  'role'::text AS entity_type,
  rmd.id,
  rmd.source_id,
  rmd.target_id,
  rmd.status,
  rmd.reason,
  rmd.actor_id,
  rmd.decided_at,
  crmc.id AS candidate_id,
  crmc.opus_decision,
  crmc.opus_reasoning,
  crmc.similarity AS opus_similarity,
  oao.cost_usd,
  oao.input_tokens,
  oao.output_tokens
FROM role_merge_decisions rmd
LEFT JOIN canonical_role_merge_candidates crmc
  ON  (crmc.canonical_a_id = rmd.source_id AND crmc.canonical_b_id = rmd.target_id)
   OR (crmc.canonical_a_id = rmd.target_id AND crmc.canonical_b_id = rmd.source_id)
LEFT JOIN opus_arbitration_outcomes oao
  ON oao.item_id = crmc.id
  AND oao.item_type = 'merge_candidate'

UNION ALL

SELECT
  'skill'::text AS entity_type,
  smd.id,
  smd.source_id,
  smd.target_id,
  smd.status,
  smd.reason,
  smd.actor_id,
  smd.decided_at,
  csmc.id AS candidate_id,
  csmc.opus_decision,
  csmc.opus_reasoning,
  csmc.similarity AS opus_similarity,
  oao.cost_usd,
  oao.input_tokens,
  oao.output_tokens
FROM skill_merge_decisions smd
LEFT JOIN canonical_skill_merge_candidates csmc
  ON  (csmc.canonical_a_id = smd.source_id AND csmc.canonical_b_id = smd.target_id)
   OR (csmc.canonical_a_id = smd.target_id AND csmc.canonical_b_id = smd.source_id)
LEFT JOIN opus_arbitration_outcomes oao
  ON oao.item_id = csmc.id
  AND oao.item_type = 'merge_candidate';

COMMENT ON VIEW v_merge_audit_history IS
'União simétrica de auditoria de merges (role + skill). entity_type derivado da fonte. opus_decision/opus_reasoning vêm de canonical_*_merge_candidates; cost_usd e tokens de opus_arbitration_outcomes. Merges manuais (sem candidato Opus prévio) terão crmc.* e oao.* NULL — LEFT JOIN preserva a linha.';

COMMIT;
```

**Performance:** o LEFT JOIN simétrico com OR pode não usar índice ótimo. Aceitável no MVP (volume baixo de merges/dia esperado). Otimização futura via convenção de ordenação canônica (sempre menor UUID como canonical_a_id) — fora do escopo desta sprint.

---

### §2.17 — Tabela `admin_panel_functions` (registry de funções admin)

**Arquivo:** `17_admin_panel_functions.sql`
**Objetivo:** registry de funções da UI admin para telemetria, controle de feature flags por função, e auditoria de uso. Substitui o termo legado "5 painéis B7" — esse era nome de outro conceito (5 SQL functions de BI agregadas no endpoint `/api/admin/limiares`). Aqui o escopo é diferente: instrumentação das telas admin operacionais.

O seed reflete as funções reais de cada tela admin existente (`/admin/pricing`, `/admin/merge-canonicals`, `/admin/campaigns`, `/admin/ingestor`) mais a tela `/admin/pipeline-config` que será construída nesta sprint.

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS admin_panel_functions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  panel_id text NOT NULL,
  panel_label text NOT NULL,
  function_id text NOT NULL,
  function_label text NOT NULL,
  function_kind text NOT NULL CHECK (function_kind IN ('action', 'view', 'export', 'config')),
  display_order int NOT NULL DEFAULT 100,
  is_enabled boolean NOT NULL DEFAULT true,
  criticality_level text NOT NULL DEFAULT 'low' CHECK (criticality_level IN ('low', 'medium', 'high')),
  panel_tags text[] DEFAULT ARRAY[]::text[],
  created_at timestamptz NOT NULL DEFAULT NOW(),
  updated_at timestamptz NOT NULL DEFAULT NOW(),
  UNIQUE (panel_id, function_id)
);

CREATE INDEX idx_admin_panel_functions_panel_id ON admin_panel_functions(panel_id);
CREATE INDEX idx_admin_panel_functions_enabled ON admin_panel_functions(is_enabled) WHERE is_enabled = true;

INSERT INTO admin_panel_functions (
  panel_id, panel_label, function_id, function_label,
  function_kind, display_order, criticality_level, panel_tags
) VALUES
  -- ── pricing (/admin/pricing) ─────────────────────────────────
  ('pricing', 'Preços', 'view_b2c_packages',          'Ver pacotes de créditos',            'view',   1, 'low',    ARRAY['pricing','b2c']),
  ('pricing', 'Preços', 'edit_package',                'Editar pacote de créditos',          'config', 2, 'high',   ARRAY['pricing','b2c','credits']),
  ('pricing', 'Preços', 'view_b2b_plans',              'Ver planos B2B',                     'view',   3, 'low',    ARRAY['pricing','b2b']),
  ('pricing', 'Preços', 'edit_plan',                   'Editar plano B2B',                   'config', 4, 'high',   ARRAY['pricing','b2b']),
  ('pricing', 'Preços', 'view_resources',              'Ver cardápio de recursos',           'view',   5, 'low',    ARRAY['pricing','resources']),
  ('pricing', 'Preços', 'edit_resource_cost',          'Editar custo de recurso',            'config', 6, 'high',   ARRAY['pricing','resources','credits']),
  ('pricing', 'Preços', 'toggle_resource_availability','Alternar disponibilidade de recurso','config', 7, 'medium', ARRAY['pricing','resources']),
  ('pricing', 'Preços', 'toggle_resource_active',      'Ativar/desativar recurso',           'config', 8, 'high',   ARRAY['pricing','resources']),

  -- ── merge_canonicals (/admin/merge-canonicals) ───────────────
  ('merge_canonicals', 'Unificação de Cadastros', 'view_merge_suggestions_roles',    'Ver sugestões (funções)',            'view',   1, 'low',    ARRAY['merge','roles','taxonomy']),
  ('merge_canonicals', 'Unificação de Cadastros', 'analyze_merge_impact',            'Analisar impacto de unificação',     'view',   2, 'low',    ARRAY['merge','roles']),
  ('merge_canonicals', 'Unificação de Cadastros', 'execute_merge_roles',             'Executar unificação de funções',     'action', 3, 'high',   ARRAY['merge','roles','taxonomy','irreversible']),
  ('merge_canonicals', 'Unificação de Cadastros', 'ignore_merge_suggestion',         'Ignorar sugestão (funções)',         'action', 4, 'medium', ARRAY['merge','roles']),
  ('merge_canonicals', 'Unificação de Cadastros', 'view_merge_suggestions_skills',   'Ver sugestões (habilidades)',        'view',   5, 'low',    ARRAY['merge','skills','taxonomy']),
  ('merge_canonicals', 'Unificação de Cadastros', 'execute_merge_skills',            'Executar unificação de habilidades', 'action', 6, 'high',   ARRAY['merge','skills','taxonomy','irreversible']),
  ('merge_canonicals', 'Unificação de Cadastros', 'ignore_merge_suggestion_skill',   'Ignorar sugestão (habilidades)',     'action', 7, 'medium', ARRAY['merge','skills']),

  -- ── campaigns (/admin/campaigns) ─────────────────────────────
  ('campaigns', 'Campanhas de Créditos', 'view_campaigns_list',         'Listar campanhas',                 'view',   1, 'low',    ARRAY['campaigns']),
  ('campaigns', 'Campanhas de Créditos', 'create_campaign',             'Criar nova campanha',              'action', 2, 'high',   ARRAY['campaigns','credits','distribution']),
  ('campaigns', 'Campanhas de Créditos', 'edit_campaign',               'Editar campanha agendada',         'config', 3, 'medium', ARRAY['campaigns']),
  ('campaigns', 'Campanhas de Créditos', 'cancel_campaign',             'Cancelar campanha',                'action', 4, 'high',   ARRAY['campaigns','irreversible']),
  ('campaigns', 'Campanhas de Créditos', 'resend_failed_participants',  'Reenviar participantes falhados',  'action', 5, 'medium', ARRAY['campaigns','dispatch']),
  ('campaigns', 'Campanhas de Créditos', 'view_campaign_participants',  'Ver participantes da campanha',    'view',   6, 'low',    ARRAY['campaigns']),

  -- ── ingestor (/admin/ingestor) ──────────────
  ('ingestor', 'Carga e Curadoria', 'ingest_json',         'Carregar lote (JSON colado)',     'action', 1, 'medium', ARRAY['ingest','curation']),
  ('ingestor', 'Carga e Curadoria', 'start_curation',      'Iniciar curadoria',                'action', 2, 'medium', ARRAY['ingest','curation','pipeline']),
  ('ingestor', 'Carga e Curadoria', 'stop_curation',       'Pausar/parar curadoria',           'action', 3, 'medium', ARRAY['ingest','curation','pipeline']),
  ('ingestor', 'Carga e Curadoria', 'view_import_report',  'Ver relatório de importação',      'view',   4, 'low',    ARRAY['ingest','curation','report']),

  -- ── pipeline_config (/admin/pipeline-config) — construído nesta sprint
  ('pipeline_config', 'Calibração de Limiares', 'view_config_keys',  'Ver chaves de configuração',    'view',   1, 'low',  ARRAY['pipeline','config']),
  ('pipeline_config', 'Calibração de Limiares', 'edit_threshold',    'Editar limiar de pipeline',     'config', 2, 'high', ARRAY['pipeline','config','threshold']),
  ('pipeline_config', 'Calibração de Limiares', 'view_history',      'Ver histórico de alterações',   'view',   3, 'low',  ARRAY['pipeline','config','history']),
  ('pipeline_config', 'Calibração de Limiares', 'rollback_change',   'Reverter alteração',            'action', 4, 'high', ARRAY['pipeline','config','threshold','rollback'])

ON CONFLICT (panel_id, function_id) DO NOTHING;

-- Nota: opus_review removido — sem tela /admin/opus-review correspondente.
-- A auditoria de decisões Opus está integrada na tela /admin/merge-canonicals
-- (Card 1 do layout pilha-degrau, ver §5.2).

COMMIT;
```

**Cobertura do registry:** cada uma das 29 linhas representa uma função admin real exposta em alguma tela. Quando uma função for adicionada/removida da UI, o seed deve ser atualizado no mesmo PR — registry sem cobertura é equivalente a não ter registry.

---

### §2.18 — Índices funcionais para lookups (vivos + criado em §2.5)

**Arquivo:** N/A — sem migration própria nesta sprint.
**Status:** os índices funcionais que sustentam os lookups de §2.4 e §2.5 já existem:
- **JCS:** `uq_jcs_label_normalized ON job_canonical_skills USING btree (normalize_skill_label(label)) WHERE (status='active' AND merged_into IS NULL)` — vivo desde a sprint paridade-skills v11. É o índice batido pela função `lookup_canonical_skill_by_normalized_label` (§2.4).
- **JCR:** `uq_jcr_canonical_label_normalized ON job_canonical_roles USING btree (normalize_role_label(canonical_label)) WHERE (status='active' AND merged_into IS NULL)` — criado em §2.5 desta sprint, simétrico ao vivo de JCS.

**Por que esta seção existe como referência:** versões anteriores da spec (v2.7.1 e v2.8) criavam aqui dois índices redundantes em `lower(label)` e `lower(canonical_label)`. Como as funções de lookup foram corrigidas em §2.4/§2.5 para usar `normalize_*_label` (alinhamento com os índices funcionais reais e com o ingestor TS), esses dois índices em `lower()` ficariam órfãos — nenhum caller bateria neles. **Removidos nesta sprint** (B-3 Claude). Não há migration `18_*.sql` correspondente; numeração mantida por compatibilidade com cross-references existentes (LK-PS, S2, etc.).

---

### §2.19 — `pipeline_config` ADD `criticality_level` + seed das 24 chaves

**Arquivo:** `19_pipeline_config_criticality.sql`
**Bloqueador resolvido:** B3.
**Ground truth (banco em pré-produção):** 26 chaves totais — 24 chaves simétricas `role.*` / `skill.*` de calibração + 2 chaves de sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) que ficam fora do escopo desta tela (ver §5.1.0).

```sql
BEGIN;

ALTER TABLE pipeline_config
  ADD COLUMN IF NOT EXISTS criticality_level text NOT NULL DEFAULT 'low'
  CHECK (criticality_level IN ('low', 'medium', 'high'));

-- ─────────────────────────────────────────────────────────────────────
-- HIGH (8 chaves) — gates de entrada e thresholds de promoção
-- exigem confirmação textual "PUBLICAR" no modal de edição (§5.1.6)
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET criticality_level = 'high'
  WHERE key IN (
    'role.hard_gate.min_confidence',
    'skill.hard_gate.min_confidence',
    'role.promotion.min_vacancies',
    'role.promotion.min_distinct_employers',
    'role.promotion.auto_min_confidence',
    'skill.promotion.min_vacancies',
    'skill.promotion.min_distinct_employers',
    'skill.promotion.auto_min_confidence'
  );

-- ─────────────────────────────────────────────────────────────────────
-- MEDIUM (8 chaves) — janelas móveis e cooldowns Opus
-- confirmação simples no modal (sem gate "PUBLICAR")
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET criticality_level = 'medium'
  WHERE key IN (
    'role.confidence.lookback_days',
    'skill.confidence.lookback_days',
    'role.promotion.lookback_days',
    'skill.promotion.lookback_days',
    'role.merge_candidate.lookback_days',
    'skill.merge_candidate.lookback_days',
    'role.opus_review.cooldown_days',
    'skill.opus_review.cooldown_days'
  );

-- ─────────────────────────────────────────────────────────────────────
-- LOW (8 chaves) — calibração fina: thresholds de similaridade,
-- contagens mínimas, gaps de aposentadoria, cooldowns secundários
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET criticality_level = 'low'
  WHERE key IN (
    'role.merge_candidate.cosine_threshold',
    'skill.merge_candidate.cosine_threshold',
    'role.confidence.min_count',
    'skill.confidence.min_count',
    'role.retirement.gap_days',
    'skill.retirement.gap_days',
    'role.merge_candidate.opus_review_cooldown_days',
    'skill.merge_candidate.opus_review_cooldown_days'
  );

-- Chaves de sistema (CURATE_PIPELINE_ENABLED, QUARANTINE_EXPIRY_DAYS)
-- permanecem com default criticality_level='low'.
-- Tela /admin/pipeline-config filtra essas chaves out via §5.1.0.

COMMIT;
```

**Decisão de classificação:** `hard_gate.min_confidence` é ALTO (gate de entrada que muda volume em produção imediatamente). `opus_review.cooldown_days` é MÉDIO (janela temporal anti-reflag, sem efeito imediato em dados). Enumeração explícita em vez de pattern `LIKE` — evita classificações acidentais quando novas chaves forem adicionadas ao banco no futuro. Cada chave entra com `criticality_level` consciente.

**Mapeamento de painéis afetados por chave:** vive exclusivamente em `lib/admin/pipeline-config-tooltips.ts` (§3.14), client-side. Não há coluna correspondente no banco — evita dupla source of truth.

---

### §2.19a — `pipeline_config` ADD `description` text + seed das 24 chaves

**Arquivo:** `19a_pipeline_config_description.sql`
**Objetivo:** alimentar a coluna "Chave + descrição" da tabela em `/admin/pipeline-config` (§5.1.4) e a caixa de descrição do modal de edição (§5.1.6). Cada chave tem um label curto explicativo que o admin lê antes de editar.
**Ground truth:** lista das 24 chaves obtida via `SELECT key FROM pipeline_config ORDER BY key` (excluindo as 2 chaves de sistema `CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`).

**Atenção operacional:** a coluna `description` já existe no banco em produção pré-MVP como `NOT NULL` (com 24/26 rows populadas). O `ALTER TABLE ... ADD COLUMN IF NOT EXISTS description text` desta migration vai pular silenciosamente (sem efeito). Os `UPDATE`s abaixo vão SOBRESCREVER os valores existentes pelos textos definitivos desta sprint. Importante: nenhum dos 24 UPDATEs pode resultar em `description = NULL` — todos os valores são strings não-vazias. Antigravity deve verificar com `SELECT COUNT(*) FROM pipeline_config WHERE description IS NULL OR description = ''` antes (esperado 2: as 2 chaves de sistema que não recebem UPDATE) e depois (esperado igual).

```sql
BEGIN;

ALTER TABLE pipeline_config ADD COLUMN IF NOT EXISTS description text;

-- ─────────────────────────────────────────────────────────────────────
-- ROLE (12 chaves)
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET description = 'Janela móvel para cálculo de confidence_median (funções)'
  WHERE key = 'role.confidence.lookback_days';

UPDATE pipeline_config SET description = 'Mínimo de amostras para confidence_median ser válido (funções)'
  WHERE key = 'role.confidence.min_count';

UPDATE pipeline_config SET description = 'Piso de aceitação para função nova entrar no pipeline'
  WHERE key = 'role.hard_gate.min_confidence';

UPDATE pipeline_config SET description = 'Similaridade mínima para detectar candidato de unificação (funções)'
  WHERE key = 'role.merge_candidate.cosine_threshold';

UPDATE pipeline_config SET description = 'Janela para varrer candidatos de unificação (funções)'
  WHERE key = 'role.merge_candidate.lookback_days';

UPDATE pipeline_config SET description = 'Cooldown anti-reflag de candidato de merge já arbitrado (funções)'
  WHERE key = 'role.merge_candidate.opus_review_cooldown_days';

UPDATE pipeline_config SET description = 'Cooldown geral anti-reflag pós-arbitragem Opus (funções)'
  WHERE key = 'role.opus_review.cooldown_days';

UPDATE pipeline_config SET description = 'Teto da Zona Opus — acima promove automático (funções)'
  WHERE key = 'role.promotion.auto_min_confidence';

UPDATE pipeline_config SET description = 'Janela para acumular vagas/empregadores para promoção (funções)'
  WHERE key = 'role.promotion.lookback_days';

UPDATE pipeline_config SET description = 'Mínimo de empregadores distintos para promoção pending→active (funções)'
  WHERE key = 'role.promotion.min_distinct_employers';

UPDATE pipeline_config SET description = 'Mínimo de vagas para promoção pending→active (funções)'
  WHERE key = 'role.promotion.min_vacancies';

UPDATE pipeline_config SET description = 'Gap máximo sem vagas antes de aposentar canonical (funções)'
  WHERE key = 'role.retirement.gap_days';

-- ─────────────────────────────────────────────────────────────────────
-- SKILL (12 chaves — espelhadas)
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET description = 'Janela móvel para cálculo de confidence_median (habilidades)'
  WHERE key = 'skill.confidence.lookback_days';

UPDATE pipeline_config SET description = 'Mínimo de amostras para confidence_median ser válido (habilidades)'
  WHERE key = 'skill.confidence.min_count';

UPDATE pipeline_config SET description = 'Piso de aceitação para habilidade nova entrar no pipeline'
  WHERE key = 'skill.hard_gate.min_confidence';

UPDATE pipeline_config SET description = 'Similaridade mínima para detectar candidato de unificação (habilidades)'
  WHERE key = 'skill.merge_candidate.cosine_threshold';

UPDATE pipeline_config SET description = 'Janela para varrer candidatos de unificação (habilidades)'
  WHERE key = 'skill.merge_candidate.lookback_days';

UPDATE pipeline_config SET description = 'Cooldown anti-reflag de candidato de merge já arbitrado (habilidades)'
  WHERE key = 'skill.merge_candidate.opus_review_cooldown_days';

UPDATE pipeline_config SET description = 'Cooldown geral anti-reflag pós-arbitragem Opus (habilidades)'
  WHERE key = 'skill.opus_review.cooldown_days';

UPDATE pipeline_config SET description = 'Teto da Zona Opus — acima promove automático (habilidades)'
  WHERE key = 'skill.promotion.auto_min_confidence';

UPDATE pipeline_config SET description = 'Janela para acumular vagas/empregadores para promoção (habilidades)'
  WHERE key = 'skill.promotion.lookback_days';

UPDATE pipeline_config SET description = 'Mínimo de empregadores distintos para promoção pending→active (habilidades)'
  WHERE key = 'skill.promotion.min_distinct_employers';

UPDATE pipeline_config SET description = 'Mínimo de vagas para promoção pending→active (habilidades)'
  WHERE key = 'skill.promotion.min_vacancies';

UPDATE pipeline_config SET description = 'Gap máximo sem vagas antes de aposentar canonical (habilidades)'
  WHERE key = 'skill.retirement.gap_days';

-- ─────────────────────────────────────────────────────────────────────
-- SISTEMA (2 chaves — fora do escopo da tela /admin/pipeline-config,
-- mas descritas por completude. Tela filtra essas chaves via §5.1.0.)
-- ─────────────────────────────────────────────────────────────────────
UPDATE pipeline_config SET description = 'Feature flag binária do CRON de curadoria'
  WHERE key = 'CURATE_PIPELINE_ENABLED';

UPDATE pipeline_config SET description = 'Janela de expiração para quarentena de candidatos'
  WHERE key = 'QUARANTINE_EXPIRY_DAYS';

COMMIT;
```

**Validação:** após migration, `SELECT COUNT(*) FROM pipeline_config WHERE description IS NULL` deve retornar `0`. Caso retorne >0, o seed está incompleto e bloqueia a §5.1 (evidence E12 da §7.1 falha).

---

### §2.20 — `set_pipeline_config_value` atualizada (com DROP do trigger duplicador)

**Arquivo:** `20_set_pipeline_config_value.sql`
**Bloqueadores resolvidos:**
- B4 — preservado (grava `pipeline_config_history` com nomes reais `changed_by`/`reason`, parâmetro real `p_value`)
- **M-2 (Manus #6 + Claude Code Q2):** trigger `trg_pipeline_config_audit` em `pipeline_config` (AFTER UPDATE) JÁ INSERIA em `pipeline_config_history` antes desta sprint, lendo do campo `NEW.last_changed_by_actor_id` que esta função NÃO atualiza. Resultado: cada chamada gerava 2 entradas — uma do trigger com `changed_by=SYSTEM_USER_ID` (errado) e uma da função com `changed_by=p_changed_by` (correto). A função agora assume controle total do histórico; o trigger é dropado nesta migration.

```sql
BEGIN;

-- DROP do trigger duplicador (Manus #6) — função assume controle do histórico
DROP TRIGGER IF EXISTS trg_pipeline_config_audit ON pipeline_config;
DROP FUNCTION IF EXISTS fn_pipeline_config_audit();

CREATE OR REPLACE FUNCTION set_pipeline_config_value(
  p_key text,
  p_value text,
  p_changed_by uuid,
  p_reason text DEFAULT NULL,
  p_confirmed boolean DEFAULT false
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_old_value text;
  v_criticality text;
BEGIN
  SELECT value, criticality_level
  INTO v_old_value, v_criticality
  FROM pipeline_config
  WHERE key = p_key;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'pipeline_config key not found: %', p_key;
  END IF;

  IF v_criticality = 'high' AND NOT p_confirmed THEN
    RAISE EXCEPTION 'high criticality change requires explicit confirmation (p_confirmed = true)';
  END IF;

  IF v_old_value = p_value THEN
    RETURN jsonb_build_object('changed', false, 'reason', 'value_unchanged');
  END IF;

  UPDATE pipeline_config SET value = p_value, updated_at = NOW() WHERE key = p_key;

  INSERT INTO pipeline_config_history (
    key, previous_value, new_value, changed_by, reason, changed_at
  ) VALUES (
    p_key, v_old_value, p_value, p_changed_by, p_reason, NOW()
  );

  RETURN jsonb_build_object(
    'changed', true,
    'key', p_key,
    'previous_value', v_old_value,
    'new_value', p_value,
    'criticality_level', v_criticality
  );
END;
$$;

COMMENT ON FUNCTION set_pipeline_config_value IS
'Atualiza chave em pipeline_config e grava histórico. Esta função é a ÚNICA fonte de escrita em pipeline_config_history (trigger trg_pipeline_config_audit foi dropado nesta migration para evitar duplicação). Bloqueia mudanças de criticidade high sem p_confirmed=true.';

COMMIT;
```

**Validação pós-migration:**

```sql
-- Trigger duplicador removido
SELECT COUNT(*) FROM pg_trigger
WHERE tgrelid = 'pipeline_config'::regclass
  AND tgname = 'trg_pipeline_config_audit';
-- Esperado: 0

-- Função set_pipeline_config_value existe
SELECT proname FROM pg_proc WHERE proname = 'set_pipeline_config_value';
-- Esperado: 1 linha
```

---

### §2.20a — Imutabilidade de `pipeline_config_history` (trigger BEFORE UPDATE/DELETE)

**Arquivo:** `20a_pipeline_config_history_immutable.sql`
**Objetivo:** garantir que histórico de alterações em `pipeline_config` é write-once. UPDATE/DELETE diretos na tabela são bloqueados por trigger, **independentemente do role** que executa (service_role, superuser, ou qualquer outro).
**Bloqueador resolvido (M-3 — ChatGPT #5 + Manus 3.1):** versão anterior usava `RLS FORCE` com premissa "SECURITY DEFINER bypassa RLS". A premissa é frágil — owner da função pode estar sujeito a RLS dependendo de configuração de role, e roles com `BYPASSRLS` ignoram FORCE também. Trigger `BEFORE UPDATE/DELETE` lançando exceção é o mecanismo robusto recomendado para imutabilidade de tabelas de auditoria. RLS de SELECT mantida para controle de leitura por admin.

```sql
BEGIN;

-- 1. SELECT permanece via RLS (admin pode ler)
ALTER TABLE pipeline_config_history ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS pipeline_config_history_select ON pipeline_config_history;
CREATE POLICY pipeline_config_history_select ON pipeline_config_history
  FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM profiles p
      WHERE p.id = auth.uid()
        AND p.role = 'admin'
    )
  );

-- 2. INSERT permitido apenas via RPC (que é SECURITY DEFINER em §2.20)
-- Política explícita para service_role; usuários comuns não inserem
DROP POLICY IF EXISTS pipeline_config_history_insert_service ON pipeline_config_history;
CREATE POLICY pipeline_config_history_insert_service ON pipeline_config_history
  FOR INSERT
  TO service_role
  WITH CHECK (true);

-- 3. Imutabilidade via TRIGGER (mecanismo robusto, não depende de role config)
CREATE OR REPLACE FUNCTION fn_pipeline_config_history_block_mutation()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  RAISE EXCEPTION 'pipeline_config_history é imutável: % bloqueado', TG_OP
    USING ERRCODE = 'P0001',
          HINT = 'Use set_pipeline_config_value() para alterar configuração — o histórico é write-once por design de auditoria.';
END;
$$;

DROP TRIGGER IF EXISTS trg_pipeline_config_history_block_update ON pipeline_config_history;
CREATE TRIGGER trg_pipeline_config_history_block_update
BEFORE UPDATE ON pipeline_config_history
FOR EACH ROW
EXECUTE FUNCTION fn_pipeline_config_history_block_mutation();

DROP TRIGGER IF EXISTS trg_pipeline_config_history_block_delete ON pipeline_config_history;
CREATE TRIGGER trg_pipeline_config_history_block_delete
BEFORE DELETE ON pipeline_config_history
FOR EACH ROW
EXECUTE FUNCTION fn_pipeline_config_history_block_mutation();

COMMIT;
```

**Validação (smoke S-RLS no §7.1.1):**

```sql
-- Cenário 1: SELECT como admin funciona
SET LOCAL ROLE authenticated;
SET LOCAL "request.jwt.claim.sub" = '<uuid-admin>';
SELECT COUNT(*) FROM pipeline_config_history;
RESET ROLE;

-- Cenário 2: UPDATE direto falha mesmo como service_role
SET LOCAL ROLE service_role;
UPDATE pipeline_config_history SET reason = 'tampered' WHERE id = (SELECT id FROM pipeline_config_history LIMIT 1);
-- Esperado: ERROR P0001 'pipeline_config_history é imutável: UPDATE bloqueado'
RESET ROLE;

-- Cenário 3: DELETE direto falha mesmo como service_role
SET LOCAL ROLE service_role;
DELETE FROM pipeline_config_history WHERE id = (SELECT id FROM pipeline_config_history LIMIT 1);
-- Esperado: ERROR P0001 'pipeline_config_history é imutável: DELETE bloqueado'
RESET ROLE;

-- Cenário 4: INSERT via RPC continua funcionando
SELECT set_pipeline_config_value('role.hard_gate.min_confidence', '0.65', '<uuid-admin>', 'teste smoke S-RLS');
SELECT * FROM pipeline_config_history WHERE key = 'role.hard_gate.min_confidence' ORDER BY changed_at DESC LIMIT 1;
-- Esperado: 1 linha inserida com changed_by = uuid-admin
```

---

### §2.21 — RENAME ATÔMICO

**Arquivo:** `21_rename_canonical_to_role.sql`
**Escopo:** rename de 4 funções regulares + 1 função wrapper de trigger + 3 triggers (consolidando 2 em 1) + 3 event_names + 1 string em metadata.origin + atualização da RPC `o6_recent_errors` para limpeza de event_names (3 edits cirúrgicos).

**Importante:** as funções `auto_assign_family_to_role`, `fn_retire_role_on_zero_vacancy`, `fn_promote_role_on_threshold` já foram criadas em §2.10, §2.11, §2.12 com os event_names corretos. Esta migration:

1. Atualiza o wrapper `trigger_auto_assign_family()` → `trigger_auto_assign_family_role()` para chamar a função renomeada.
2. Dropa triggers antigos.
3. Dropa funções antigas órfãs.
4. Cria triggers com nomes novos.
5. Consolida os 2 triggers `auto_assign_family_on_active` (UPDATE + INSERT) em **1 trigger** `AFTER INSERT OR UPDATE`.
6. Recria `o6_recent_errors` com a lista atualizada de event_names.

```sql
BEGIN;

-- ─── 1. Atualizar wrapper trigger function ───
DROP FUNCTION IF EXISTS trigger_auto_assign_family();

CREATE OR REPLACE FUNCTION trigger_auto_assign_family_role()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $$
BEGIN
  IF NEW.status = 'active'
     AND (TG_OP = 'INSERT' OR OLD.status IS DISTINCT FROM 'active') THEN
    PERFORM auto_assign_family_to_role(NEW.id);
  END IF;
  RETURN NULL;
END;
$$;

-- ─── 2. Dropar triggers antigos ───
DROP TRIGGER IF EXISTS trg_promote_on_threshold ON job_canonical_roles;
DROP TRIGGER IF EXISTS z_trg_retire_canonical_on_zero_vacancy ON job_canonical_roles;
DROP TRIGGER IF EXISTS auto_assign_family_on_active ON job_canonical_roles;
-- Nota: PostgreSQL exige nomes únicos de trigger por tabela. Se o schema atual
-- tem 2 triggers de auto-assign com nomes diferentes (ex: *_upd e *_ins),
-- o DROP acima precisa atingir os 2 nomes reais — Antigravity deve rodar
-- `SELECT tgname FROM pg_trigger WHERE tgname LIKE '%auto_assign_family%'`
-- imediatamente antes para confirmar e ajustar a lista de DROPs se necessário.

-- ─── 3. Dropar funções antigas (já substituídas) ───
DROP FUNCTION IF EXISTS fn_promote_canonical_on_threshold();
DROP FUNCTION IF EXISTS fn_retire_canonical_on_zero_vacancy();
DROP FUNCTION IF EXISTS auto_assign_family_to_canonical(uuid, boolean);

-- ─── 4. Criar triggers com nomes novos ───
CREATE TRIGGER trg_promote_role_on_threshold
AFTER UPDATE OF vacancy_count, distinct_sources_count, confidence_median ON job_canonical_roles
FOR EACH ROW
EXECUTE FUNCTION fn_promote_role_on_threshold();

CREATE TRIGGER z_trg_retire_role_on_zero_vacancy
AFTER UPDATE OF vacancy_count ON job_canonical_roles
FOR EACH ROW
EXECUTE FUNCTION fn_retire_role_on_zero_vacancy();

-- ─── 5. Consolidação de 2 triggers em 1 ───
-- Antes: 2 triggers separados auto_assign_family_on_active (UPDATE + INSERT)
-- Depois: 1 trigger AFTER INSERT OR UPDATE com guard via TG_OP no wrapper
CREATE TRIGGER auto_assign_family_on_role_active
AFTER INSERT OR UPDATE OF status ON job_canonical_roles
FOR EACH ROW
WHEN (NEW.status = 'active')
EXECUTE FUNCTION trigger_auto_assign_family_role();

-- ─── 6. Recriar RPC o6_recent_errors com limpeza de event_names aplicada ───
-- 3 edits cirúrgicos contra o corpo atual (schema.sql:5726-5780):
--   linha 5739 (CASE WHEN block): substituir os 2 event_names canonical_* + remover resolved_id_mismatch
--   linha 5749 (WHERE IN list): mesma substituição
--   linha 5756 (AND filter block): mesma substituição
CREATE OR REPLACE FUNCTION public.o6_recent_errors(
    p_filter text DEFAULT 'all',
    p_limit  integer DEFAULT 50
)
RETURNS TABLE(
    id           uuid,
    event_name   text,
    resource_type text,
    resource_id  uuid,
    category     text,
    metadata     jsonb,
    created_at   timestamp with time zone
)
LANGUAGE sql STABLE SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $function$
    SELECT
        e.id,
        e.event_name,
        e.resource_type,
        e.resource_id,
        CASE
            WHEN e.event_name IN ('sonnet_invalid_canonical_confidence', 'sonnet_invalid_skill_confidence', 'sonnet_output_schema_violation') THEN 'sonnet_parse'
            WHEN e.event_name IN ('role_creation_blocked_low_confidence', 'role_creation_blocked_missing_confidence') THEN 'hard_gate'
            WHEN e.event_name IN ('opus_arbitration_failed', 'opus_arbitration_quota_exceeded') THEN 'opus_rate_limit'
            WHEN e.event_name IN ('opus_cache_miss_unexpected', 'sonnet_cache_miss_unexpected') THEN 'cache_miss'
            ELSE 'other'
        END AS category,
        COALESCE(e.metadata, e.new_state) AS metadata,
        e.created_at
    FROM events e
    WHERE e.event_name IN (
        'sonnet_invalid_canonical_confidence', 'sonnet_invalid_skill_confidence', 'sonnet_output_schema_violation',
        'role_creation_blocked_low_confidence', 'role_creation_blocked_missing_confidence',
        'opus_arbitration_failed', 'opus_arbitration_quota_exceeded',
        'opus_cache_miss_unexpected', 'sonnet_cache_miss_unexpected'
    )
    AND (
        p_filter = 'all'
        OR (p_filter = 'sonnet_parse' AND e.event_name IN ('sonnet_invalid_canonical_confidence', 'sonnet_invalid_skill_confidence', 'sonnet_output_schema_violation'))
        OR (p_filter = 'hard_gate' AND e.event_name IN ('role_creation_blocked_low_confidence', 'role_creation_blocked_missing_confidence'))
        OR (p_filter = 'opus_rate_limit' AND e.event_name IN ('opus_arbitration_failed', 'opus_arbitration_quota_exceeded'))
        OR (p_filter = 'cache_miss' AND e.event_name IN ('opus_cache_miss_unexpected', 'sonnet_cache_miss_unexpected'))
    )
    ORDER BY e.created_at DESC
    LIMIT p_limit;
$function$;

COMMIT;
```

**Validação pós-aplicação:**
```sql
-- Confirma que o trigger consolidado existe e funciona em INSERT e UPDATE
SELECT tgname, pg_get_triggerdef(oid)
FROM pg_trigger
WHERE tgname = 'auto_assign_family_on_role_active';
-- Esperado: 1 linha com "AFTER INSERT OR UPDATE OF status"

-- Confirma que função wrapper foi renomeada
SELECT proname FROM pg_proc WHERE proname = 'trigger_auto_assign_family_role';
-- Esperado: 1 linha

SELECT proname FROM pg_proc WHERE proname = 'trigger_auto_assign_family';
-- Esperado: 0 linhas
```

---

### §2.22 — Triggers de flag escutam `OF status` + early-exit ajustado

**Arquivo:** `22_fn_flag_needs_opus_review_status.sql`
**Objetivo:** fechar o gap latente onde canonicals promovidos com mediana entre `hard_gate.min_confidence` e `auto_min_confidence` não ganhavam flag até a próxima vaga.
**Atende:** A10 = Opção C com correção arquitetural sugerida pelo Onsly (modificar trigger de flag, não trigger de promoção — preserva SRP).

**Nota sobre coluna `last_opus_review_at`:** existe em `job_canonical_roles` e `job_canonical_skills` (par com `needs_opus_review` boolean). Estas triggers leem `NEW.last_opus_review_at` para aplicar cooldown anti-reflag. **Importante:** `canonical_role_merge_candidates` e `canonical_skill_merge_candidates` NÃO têm essa coluna — usam `last_arbitration_attempt_at` (assimetria de schema, consultar §2.34). Migrations nesta sprint preservam essa convenção: triggers de JCR/JCS usam `last_opus_review_at`, painéis de merge_candidates usam `last_arbitration_attempt_at`.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcr()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_upper_threshold numeric;
  v_cooldown_days int;
BEGIN
  -- Early-exit ajustado: só sai cedo se NEM mediana NEM status mudaram
  IF NEW.confidence_median IS NOT DISTINCT FROM OLD.confidence_median
     AND NEW.status IS NOT DISTINCT FROM OLD.status THEN
    RETURN NEW;
  END IF;

  IF NEW.status != 'active' THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NULL THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review = TRUE THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::numeric FROM pipeline_config WHERE key='role.promotion.auto_min_confidence'), 0.85),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.opus_review.cooldown_days'), 90)
  INTO v_upper_threshold, v_cooldown_days;

  IF NEW.confidence_median < v_upper_threshold
     AND (NEW.last_opus_review_at IS NULL
          OR NEW.last_opus_review_at < NOW() - (v_cooldown_days || ' days')::interval) THEN
    NEW.needs_opus_review := TRUE;
  END IF;

  RETURN NEW;
END;
$$;

CREATE OR REPLACE FUNCTION fn_flag_needs_opus_review_jcs()
RETURNS trigger
LANGUAGE plpgsql
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_upper_threshold numeric;
  v_cooldown_days int;
BEGIN
  IF NEW.confidence_median IS NOT DISTINCT FROM OLD.confidence_median
     AND NEW.status IS NOT DISTINCT FROM OLD.status THEN
    RETURN NEW;
  END IF;

  IF NEW.status != 'active' THEN RETURN NEW; END IF;
  IF NEW.confidence_median IS NULL THEN RETURN NEW; END IF;
  IF NEW.needs_opus_review = TRUE THEN RETURN NEW; END IF;

  SELECT COALESCE((SELECT value::numeric FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='skill.opus_review.cooldown_days'), 90)
  INTO v_upper_threshold, v_cooldown_days;

  IF NEW.confidence_median < v_upper_threshold
     AND (NEW.last_opus_review_at IS NULL
          OR NEW.last_opus_review_at < NOW() - (v_cooldown_days || ' days')::interval) THEN
    NEW.needs_opus_review := TRUE;
  END IF;

  RETURN NEW;
END;
$$;

-- Recriar triggers com OF confidence_median, status (antes era OF confidence_median apenas).
-- Os 4 triggers do par (2 BEFORE recriados aqui + 2 AFTER _emit preservados intactos) seguem o padrão:
-- BEFORE seta a flag needs_opus_review (set/clear), AFTER emite o evento role/skill_flagged_low_confidence_residual
-- só quando a flag muda de FALSE→TRUE. Migrar os dois lados garante simetria e idempotência.

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcr ON job_canonical_roles;
CREATE TRIGGER trg_flag_needs_opus_review_jcr
BEFORE UPDATE OF confidence_median, status ON job_canonical_roles
FOR EACH ROW
EXECUTE FUNCTION fn_flag_needs_opus_review_jcr();

DROP TRIGGER IF EXISTS trg_flag_needs_opus_review_jcs ON job_canonical_skills;
CREATE TRIGGER trg_flag_needs_opus_review_jcs
BEFORE UPDATE OF confidence_median, status ON job_canonical_skills
FOR EACH ROW
EXECUTE FUNCTION fn_flag_needs_opus_review_jcs();

-- IMPORTANTE: os 2 triggers AFTER que emitem eventos (trg_flag_needs_opus_review_jcr_emit
-- e trg_flag_needs_opus_review_jcs_emit, junto com suas funções
-- fn_flag_needs_opus_review_jcr_emit_event / _jcs_emit_event) EXISTEM em produção e
-- DEVEM SER PRESERVADOS INTACTOS. Esta migration recria APENAS os triggers BEFORE.
-- NÃO inclua DROP dos triggers _emit ou de suas funções — eles seguem emitindo
-- role_flagged_low_confidence_residual / skill_flagged_low_confidence_residual quando
-- a flag muda de FALSE→TRUE. Validar pós-deploy:
--
--   SELECT tgname FROM pg_trigger WHERE tgrelid IN
--     ('job_canonical_roles'::regclass, 'job_canonical_skills'::regclass)
--     AND tgname LIKE 'trg_flag_needs_opus_review%';
--   -- Esperado: 4 linhas (2 BEFORE recriados + 2 AFTER _emit preservados).

COMMIT;
```

**Arquitetura "BEFORE seta flag + AFTER emite evento":** o par BEFORE/AFTER permite que a tabela tenha o estado consistente da flag (BEFORE) antes do AFTER ter chance de inspecionar `OLD.needs_opus_review` vs `NEW.needs_opus_review` para decidir se emite. Misturar set + emit em um único trigger BEFORE complicaria a lógica de detecção de transição. Os triggers `_emit` são existentes em produção desde a sprint paridade-skills v11 e foram explicitamente preservados — a presente sprint não os toca, apenas substitui o componente BEFORE que decide o valor da flag.

**Cobertura organica:**
- Promoção via trigger (`fn_promote_role_on_threshold` ou `fn_promote_skill_on_threshold` após §2.24): UPDATE interno SET status='active' dispara `trg_flag_needs_opus_review_*` em depth=2, processa, seta flag se aplicável.
- Promoção via CRON (`catchup_pending_role_promotions` ou `catchup_pending_skill_promotions`): UPDATE SET status='active' direto dispara o mesmo trigger, mesma cobertura.

---

### §2.23 — `catchup_pending_role_promotions` (renomeada)

**Arquivo:** `23_catchup_pending_role_promotions.sql`
**Substitui:** `catchup_pending_promotions`.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION catchup_pending_role_promotions()
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_min_vacancies int;
  v_min_employers int;
  v_lookback_days int;
  v_promoted int := 0;
  v_resurrected int := 0;
  v_archaeological int := 0;
  v_unchanged int := 0;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
  r record;
  v_has_recent boolean;
BEGIN
  SELECT COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.min_vacancies'), 3),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.min_distinct_employers'), 2),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='role.promotion.lookback_days'), 180)
  INTO v_min_vacancies, v_min_employers, v_lookback_days;

  FOR r IN
    SELECT jcr.id, jcr.status AS status_before, jcr.vacancy_count, jcr.distinct_sources_count, jcr.confidence_median
    FROM job_canonical_roles jcr
    WHERE jcr.status IN ('pending', 'retired')
      AND jcr.vacancy_count >= v_min_vacancies
      AND jcr.distinct_sources_count >= v_min_employers
      AND jcr.merged_into IS NULL
      AND jcr.source != 'cbo_mte_2002_seed'
  LOOP
    SELECT EXISTS(
      SELECT 1 FROM job_postings jp
      WHERE jp.canonical_role_id = r.id
        AND jp.posted_at >= NOW() - make_interval(days => v_lookback_days)
        AND jp.curation_status = 'curated'   -- A-1 (Gemini #1): paridade com §2.12 — recência só considera vagas curadas
    ) INTO v_has_recent;

    IF NOT v_has_recent THEN
      v_archaeological := v_archaeological + 1;
      CONTINUE;
    END IF;

    UPDATE job_canonical_roles SET
      status = 'active',
      promoted_at = COALESCE(promoted_at, NOW()),
      vacancy_count_at_promotion = r.vacancy_count,
      distinct_sources_count_at_promotion = r.distinct_sources_count,
      confidence_median_at_promotion = r.confidence_median,
      retired_at = NULL,
      retire_reason = NULL
    WHERE id = r.id
      AND status IN ('pending', 'retired');

    IF r.status_before = 'retired' THEN
      v_resurrected := v_resurrected + 1;
    ELSE
      v_promoted := v_promoted + 1;
    END IF;

    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, previous_state, new_state, reason, metadata
    ) VALUES (
      'role_promoted_dynamic',
      'job_canonical_role',
      r.id,
      'system',
      v_actor_id,
      jsonb_build_object('status', r.status_before),
      jsonb_build_object('status', 'active'),
      'Promoção por catchup CRON',
      jsonb_build_object(
        'origin', 'catchup_pending_role_promotions',
        'vacancy_count', r.vacancy_count,
        'distinct_sources_count', r.distinct_sources_count,
        'confidence_median', r.confidence_median
      )
    );

    -- Atribuição automática de família após promoção
    -- (restaura comportamento existente em catchup_pending_promotions, schema.sql:2617)
    BEGIN
      PERFORM auto_assign_family_to_role(r.id, TRUE);
    EXCEPTION WHEN OTHERS THEN
      INSERT INTO events (
        event_name, resource_type, resource_id,
        actor, actor_id, reason, metadata
      ) VALUES (
        'auto_assign_family_failed_in_catchup',
        'job_canonical_role',
        r.id,
        'system',
        v_actor_id,
        'Falha em auto_assign_family_to_role durante catchup',
        jsonb_build_object('error', SQLERRM)
      );
    END;
  END LOOP;

  INSERT INTO events (
    event_name, resource_type, resource_id,
    actor, actor_id, new_state, reason, metadata
  ) VALUES (
    'catchup_pending_role_promotions_executed',
    'system',
    gen_random_uuid(),
    'system',
    v_actor_id,
    jsonb_build_object(
      'promoted', v_promoted,
      'resurrected', v_resurrected,
      'archaeological', v_archaeological,
      'unchanged', v_unchanged
    ),
    'CRON catchup pending role promotions executado',
    jsonb_build_object('lookback_days', v_lookback_days)
  );

  RETURN jsonb_build_object(
    'promoted', v_promoted,
    'resurrected', v_resurrected,
    'archaeological', v_archaeological,
    'unchanged', v_unchanged
  );
END;
$$;

DROP FUNCTION IF EXISTS catchup_pending_promotions();

COMMIT;
```

---

### §2.24 — `catchup_pending_skill_promotions` (sem gate de mediana)

**Arquivo:** `24_catchup_pending_skill_promotions.sql`
**Atende:** A10 = Opção C — remove gate `confidence_median >= auto_min_confidence`.

**Nota — função NOVA, não substituição:** `catchup_pending_skill_promotions` não existe no banco antes desta sprint (padrão JCR só tem `catchup_pending_promotions` para roles, sem simétrico para skills). Por isso esta migration não tem `DROP FUNCTION IF EXISTS catchup_pending_skill_promotions()` na abertura — diferente de §2.23 que dropa `catchup_pending_promotions` antes do `CREATE OR REPLACE`. `CREATE OR REPLACE FUNCTION` aqui é efetivamente um `CREATE` puro.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION catchup_pending_skill_promotions()
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_min_vacancies int;
  v_min_employers int;
  v_lookback_days int;
  v_promoted int := 0;
  v_resurrected int := 0;
  v_archaeological int := 0;
  v_unchanged int := 0;
  v_actor_id uuid := '00000000-0000-0000-0000-000000000001'::uuid;
  r record;
  v_has_recent boolean;
BEGIN
  SELECT COALESCE((SELECT value::int FROM pipeline_config WHERE key='skill.promotion.min_vacancies'), 5),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='skill.promotion.min_distinct_employers'), 2),
         COALESCE((SELECT value::int FROM pipeline_config WHERE key='skill.promotion.lookback_days'), 180)
  INTO v_min_vacancies, v_min_employers, v_lookback_days;

  FOR r IN
    SELECT jcs.id, jcs.status AS status_before, jcs.vacancy_count, jcs.distinct_sources_count, jcs.confidence_median
    FROM job_canonical_skills jcs
    WHERE jcs.status IN ('pending', 'retired')
      AND jcs.vacancy_count >= v_min_vacancies
      AND jcs.distinct_sources_count >= v_min_employers
      AND jcs.merged_into IS NULL
    -- Opção C: NÃO há gate `confidence_median >= auto_min_confidence` aqui
  LOOP
    SELECT EXISTS(
      SELECT 1 FROM job_posting_skills jps
      JOIN job_postings jp ON jp.id = jps.job_posting_id
      WHERE jps.canonical_skill_id = r.id
        AND jp.posted_at >= NOW() - make_interval(days => v_lookback_days)
        AND jp.curation_status = 'curated'   -- A-1 (Gemini #1): simétrico ao §2.23 — recência só considera vagas curadas
    ) INTO v_has_recent;

    IF NOT v_has_recent THEN
      v_archaeological := v_archaeological + 1;
      CONTINUE;
    END IF;

    UPDATE job_canonical_skills SET
      status = 'active',
      promoted_at = COALESCE(promoted_at, NOW()),
      vacancy_count_at_promotion = r.vacancy_count,
      distinct_sources_count_at_promotion = r.distinct_sources_count,
      confidence_median_at_promotion = r.confidence_median,
      retired_at = NULL,
      retire_reason = NULL
    WHERE id = r.id
      AND status IN ('pending', 'retired');

    IF r.status_before = 'retired' THEN
      v_resurrected := v_resurrected + 1;
    ELSE
      v_promoted := v_promoted + 1;
    END IF;

    INSERT INTO events (
      event_name, resource_type, resource_id,
      actor, actor_id, previous_state, new_state, reason, metadata
    ) VALUES (
      'skill_promoted_dynamic',
      'canonical_skill',
      r.id,
      'system',
      v_actor_id,
      jsonb_build_object('status', r.status_before),
      jsonb_build_object('status', 'active'),
      'Promoção por catchup CRON',
      jsonb_build_object(
        'origin', 'catchup_pending_skill_promotions',
        'vacancy_count', r.vacancy_count,
        'distinct_sources_count', r.distinct_sources_count,
        'confidence_median', r.confidence_median
      )
    );
  END LOOP;

  INSERT INTO events (
    event_name, resource_type, resource_id,
    actor, actor_id, new_state, reason, metadata
  ) VALUES (
    'catchup_pending_skill_promotions_executed',
    'system',
    gen_random_uuid(),
    'system',
    v_actor_id,
    jsonb_build_object(
      'promoted', v_promoted,
      'resurrected', v_resurrected,
      'archaeological', v_archaeological,
      'unchanged', v_unchanged
    ),
    'CRON catchup pending skill promotions executado',
    jsonb_build_object('lookback_days', v_lookback_days)
  );

  RETURN jsonb_build_object(
    'promoted', v_promoted,
    'resurrected', v_resurrected,
    'archaeological', v_archaeological,
    'unchanged', v_unchanged
  );
END;
$$;

COMMIT;
```

**Nota Opção C — trigger `fn_promote_skill_on_threshold`:** o trigger correspondente precisa do mesmo tratamento. Como o corpo atual foi reproduzido pelo Claude Code (schema.sql:4179-4298), a edição é cirúrgica com **4 alterações localizadas** preservando todo o resto idêntico:

**Edit 1 — declarações (remover linha):**
```sql
-- ANTES (linha 4187):
DECLARE
  ...
  v_min_confidence NUMERIC;    -- ← REMOVER esta declaração
  ...

-- DEPOIS:
DECLARE
  ...
  -- v_min_confidence removido
  ...
```

**Edit 2 — SELECT de pipeline_config (remover linha):**
```sql
-- ANTES (linha 4201):
SELECT COALESCE((SELECT value::NUMERIC FROM pipeline_config WHERE key='skill.promotion.auto_min_confidence'), 0.85) INTO v_min_confidence;
-- ← REMOVER esta linha inteira
```

**Edit 3 — composição de `v_should_promote` (remover 2 linhas):**
```sql
-- ANTES (linhas 4203-4208):
v_should_promote := (
  NEW.vacancy_count >= v_min_vacancies AND
  NEW.distinct_sources_count >= v_min_employers AND
  NEW.confidence_median IS NOT NULL AND        -- ← REMOVER
  NEW.confidence_median >= v_min_confidence    -- ← REMOVER (também o AND da linha anterior)
);

-- DEPOIS:
v_should_promote := (
  NEW.vacancy_count >= v_min_vacancies AND
  NEW.distinct_sources_count >= v_min_employers
);
```

**Edit 4 — metadata do evento `skill_promoted_dynamic` (remover par chave-valor):**
```sql
-- ANTES (linha 4256, dentro do jsonb_build_object de thresholds):
'min_confidence', v_min_confidence,     -- ← REMOVER esta linha
```

Aplicar via `CREATE OR REPLACE FUNCTION` com o corpo completo da função em uma única migration. **Não há outras alterações além dessas 4.** Migration recomendada: `24a_fn_promote_skill_remove_median_gate.sql` (sub-arquivo da §2.24).

**Procedimento operacional para Antigravity (B1 resolvido):** o corpo completo da função (~120 linhas) não é replicado nesta spec por questão de tamanho — referência: `schema.sql:4179-4298` no repositório. O fluxo de execução:

1. **Antes da migration**, Antigravity executa via conexão Supabase:
   ```sql
   SELECT pg_get_functiondef(oid)
   FROM pg_proc
   WHERE proname = 'fn_promote_skill_on_threshold'
   LIMIT 1;
   ```
   Salva o resultado em buffer local.

2. **Aplica as 4 edições cirúrgicas** descritas acima (Edit 1 a Edit 4) no buffer — linhas a remover são explícitas. Não há linhas a adicionar.

3. **Substitui via `CREATE OR REPLACE FUNCTION`** com o buffer editado em transação:
   ```sql
   BEGIN;
   CREATE OR REPLACE FUNCTION fn_promote_skill_on_threshold() RETURNS trigger ... AS $$
     -- corpo editado completo
   $$;
   COMMIT;
   ```

4. **Validação pós-migration:** executar `pg_get_functiondef` de novo e confirmar via diff que apenas as 4 alterações foram aplicadas. Nenhuma outra linha deve diferir.

Esse fluxo respeita a diretriz de Antigravity conectar direto ao Supabase, sem necessidade de Onsly extrair manualmente o corpo da função.

---

### §2.25 — UNIQUE em `resume_roles` (já cravada em §2.7)

**Arquivo:** N/A — sem migration própria nesta sprint.
**Status:** a constraint `uq_resume_roles_resume_role_normalized UNIQUE (resume_id, role_normalized)` já é criada em §2.7 (overhaul), via coluna gerada `role_normalized` (M-1 do DeepSeek #3 — coluna `STORED` em vez de índice funcional). Esta seção é mantida apenas como ancoragem para referências cruzadas no spec; nenhum SQL adicional é necessário.

**Por que coluna gerada em vez de índice funcional:** Supabase JS não detecta conflito em `onConflict` baseado em índice funcional `lower(trim(...))`. Com coluna gerada `STORED` mais UNIQUE regular sobre essa coluna, o `upsert({...}, { onConflict: 'resume_id,role_normalized' })` em `§3.3` funciona corretamente.

---

### §2.26 — `admin_panel_custo_opus_por_canonical` (Painel B7 #1)

**Arquivo:** `26_admin_panel_custo_opus_por_canonical.sql`
**Atende:** B7 — Painel 1 do endpoint `/api/admin/limiares` (custo IA por canônico em janela de N dias).

**Modelo de atribuição:** `opus_arbitration_outcomes` registra custo POR ARBITRAGEM (par de canonicals avaliado). Cada arbitragem envolve dois canonicals (`canonical_a_id`, `canonical_b_id` no candidate). Decisão: atribuir o custo aos DOIS canonicals do par. Soma total fica duplicada (cada chamada Opus aparece em 2 linhas no resultado), mas reflete corretamente que cada canonical foi envolvido naquela arbitragem cara. Frontend pode filtrar por `entity_type` para ranking individualizado.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_custo_opus_por_canonical(p_days int)
RETURNS TABLE (
  canonical_id uuid,
  entity_type text,
  total_cost_usd numeric,
  call_count int
)
LANGUAGE sql STABLE AS $$
  WITH role_pairs AS (
    SELECT
      crmc.canonical_a_id,
      crmc.canonical_b_id,
      oao.cost_usd
    FROM opus_arbitration_outcomes oao
    JOIN canonical_role_merge_candidates crmc ON crmc.id = oao.item_id
    WHERE oao.item_type = 'merge_candidate'
      AND oao.decided_at >= NOW() - (p_days || ' days')::interval
  ),
  skill_pairs AS (
    SELECT
      csmc.canonical_a_id,
      csmc.canonical_b_id,
      oao.cost_usd
    FROM opus_arbitration_outcomes oao
    JOIN canonical_skill_merge_candidates csmc ON csmc.id = oao.item_id
    WHERE oao.item_type = 'merge_candidate'
      AND oao.decided_at >= NOW() - (p_days || ' days')::interval
  ),
  unified AS (
    SELECT canonical_a_id AS canonical_id, 'role'::text AS entity_type, cost_usd FROM role_pairs
    UNION ALL
    SELECT canonical_b_id, 'role'::text, cost_usd FROM role_pairs
    UNION ALL
    SELECT canonical_a_id, 'skill'::text, cost_usd FROM skill_pairs
    UNION ALL
    SELECT canonical_b_id, 'skill'::text, cost_usd FROM skill_pairs
  )
  SELECT
    canonical_id,
    entity_type,
    SUM(cost_usd) AS total_cost_usd,
    COUNT(*)::int AS call_count
  FROM unified
  WHERE canonical_id IS NOT NULL
  GROUP BY canonical_id, entity_type
  ORDER BY total_cost_usd DESC
  LIMIT 100;
$$;

COMMIT;
```

**Filtro discriminador:** o JOIN com `canonical_role_merge_candidates` discrimina runs de role automaticamente (só dá match se `oao.item_id` for um id real dessa tabela). Idem para skill. O CHECK constraint de `item_type` aceita `'merge_candidate'` como valor unificado — o tipo (role vs skill) vem da tabela de match, não do `item_type`.

**Trade-off do modelo:** soma total = 2× soma real. Aceitável porque (a) é ranking ordenado, não soma absoluta de "quanto gastei em Opus"; (b) interpretação correta é "este canonical foi envolvido em X chamadas que totalizaram Y USD"; (c) métrica agregada de custo absoluto vem do Painel B7 #2 ou de query direta em `opus_arbitration_outcomes`.

---

### §2.27 — `admin_panel_merge_auto_vs_manual` (Painel B7 #2)

**Arquivo:** `27_admin_panel_merge_auto_vs_manual.sql`
**Atende:** B7 — Painel 2 (proporção auto vs manual nas decisões de merge).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_merge_auto_vs_manual(p_days int)
RETURNS jsonb
LANGUAGE sql STABLE AS $$
  WITH base AS (
    SELECT
      CASE WHEN actor_id = '00000000-0000-0000-0000-000000000001'::uuid
           THEN 'auto' ELSE 'manual' END AS source,
      entity_type,
      status
    FROM (
      SELECT
        actor_id,
        'role'::text AS entity_type,
        status,
        decided_at
      FROM role_merge_decisions
      WHERE decided_at >= NOW() - (p_days || ' days')::interval
      UNION ALL
      SELECT
        actor_id,
        'skill'::text AS entity_type,
        status,
        decided_at
      FROM skill_merge_decisions
      WHERE decided_at >= NOW() - (p_days || ' days')::interval
    ) all_decisions
  )
  SELECT jsonb_build_object(
    'auto',       COUNT(*) FILTER (WHERE source = 'auto'),
    'manual',     COUNT(*) FILTER (WHERE source = 'manual'),
    'total',      COUNT(*),
    'auto_ratio', ROUND(COUNT(*) FILTER (WHERE source = 'auto')::numeric / NULLIF(COUNT(*), 0), 3),
    'by_entity',  jsonb_build_object(
      'role',  jsonb_build_object(
        'auto',   COUNT(*) FILTER (WHERE source = 'auto' AND entity_type = 'role'),
        'manual', COUNT(*) FILTER (WHERE source = 'manual' AND entity_type = 'role')
      ),
      'skill', jsonb_build_object(
        'auto',   COUNT(*) FILTER (WHERE source = 'auto' AND entity_type = 'skill'),
        'manual', COUNT(*) FILTER (WHERE source = 'manual' AND entity_type = 'skill')
      )
    )
  )
  FROM base;
$$;

COMMIT;
```

**Nota:** SYSTEM_USER_ID `00000000-0000-0000-0000-000000000001` identifica decisões automáticas (CRON/IA). Demais `actor_id` são humanos.

---

### §2.28 — `admin_panel_latest_posted_distribution` (Painel B7 #3)

**Arquivo:** `28_admin_panel_latest_posted_distribution.sql`
**Atende:** B7 — Painel 3 (distribuição de `latest_posted_at` em bandas temporais — termômetro de obsolescência).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_latest_posted_distribution()
RETURNS jsonb
LANGUAGE sql STABLE AS $$
  SELECT jsonb_build_object(
    'role', jsonb_build_object(
      'last_7d',  COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '7 days'),
      'last_30d', COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '30 days' AND latest_posted_at < NOW() - interval '7 days'),
      'last_60d', COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '60 days' AND latest_posted_at < NOW() - interval '30 days'),
      'older',    COUNT(*) FILTER (WHERE latest_posted_at < NOW() - interval '60 days'),
      'null',     COUNT(*) FILTER (WHERE latest_posted_at IS NULL)
    ),
    'skill', (
      SELECT jsonb_build_object(
        'last_7d',  COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '7 days'),
        'last_30d', COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '30 days' AND latest_posted_at < NOW() - interval '7 days'),
        'last_60d', COUNT(*) FILTER (WHERE latest_posted_at >= NOW() - interval '60 days' AND latest_posted_at < NOW() - interval '30 days'),
        'older',    COUNT(*) FILTER (WHERE latest_posted_at < NOW() - interval '60 days'),
        'null',     COUNT(*) FILTER (WHERE latest_posted_at IS NULL)
      )
      FROM job_canonical_skills WHERE status = 'active'
    )
  )
  FROM job_canonical_roles WHERE status = 'active';
$$;

COMMIT;
```

**Dependência:** consome `latest_posted_at` em ambas tabelas. Para skills, a coluna já existe; para roles é criada em §2.9.

---

### §2.29 — `admin_panel_suggestion_rejected_by_skill` (Painel B7 #5)

**Arquivo:** `29_admin_panel_suggestion_rejected_by_skill.sql`
**Atende:** B7 — Painel 5 (ranking de habilidades com sugestões rejeitadas pelo usuário — sinal de qualidade ruim de inferência).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_suggestion_rejected_by_skill(p_days int)
RETURNS TABLE (
  canonical_skill_id uuid,
  rejected_count int,
  skill_label text
)
LANGUAGE sql STABLE AS $$
  SELECT
    rse.canonical_skill_id,
    COUNT(*)::int AS rejected_count,
    jcs.label AS skill_label
  FROM resume_skill_enrichments rse
  JOIN job_canonical_skills jcs ON jcs.id = rse.canonical_skill_id
  WHERE rse.validation_status = 'rejected'
    AND rse.updated_at >= NOW() - (p_days || ' days')::interval
  GROUP BY rse.canonical_skill_id, jcs.label
  ORDER BY rejected_count DESC
  LIMIT 50;
$$;

COMMIT;
```

---

### §2.30 — Validação de `v_opus_effectiveness` (Painel B7 #4)

**Arquivo:** `30_validate_v_opus_effectiveness.sql`
**Atende:** B7 — Painel 4 (efetividade por verdict via view pré-existente).

A view `v_opus_effectiveness` é assumida pré-existente. Esta migration é **defensiva** — emite warning se a view não existir (sem abortar a migration), sinalizando que o Painel B7 #4 ficará inativo até que a view seja criada em sprint separada.

```sql
BEGIN;

DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.views
    WHERE table_name = 'v_opus_effectiveness'
  ) THEN
    RAISE WARNING
      'View v_opus_effectiveness não existe — Painel 4 do B7 inativo neste ambiente. Criar em sprint separada se necessário.';
  END IF;
END $$;

COMMIT;
```

**Nota:** o `RAISE WARNING` permite a migration continuar normalmente (não aborta a transação). Painel B7 #4 entra ativo junto aos outros 4 no SUB-PR-8a se a view existir. Em produção pré-base limpa, a view existe e retorna dados (agrega por `item_type` e `day` sobre `opus_arbitration_outcomes`).

---

### §2.31 — `taxonomy_blacklist` (tabela base — sprint futura completa a gestão)

**Arquivo:** `31_taxonomy_blacklist.sql`
**Atende:** suporte ao split-button "Rejeitar e banir" em §5.2.5. Tela completa de Blacklist (CRUD com paginação, filtros, importação em lote, histórico) fica em sprint futura — esta sprint cria apenas a tabela mínima para o fluxo contextual funcionar.

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS taxonomy_blacklist (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  label       text NOT NULL,
  entity_type text NOT NULL CHECK (entity_type IN ('role', 'skill')),
  reason      text,
  added_by    uuid REFERENCES profiles(id) ON DELETE SET NULL,
  added_at    timestamptz NOT NULL DEFAULT now(),
  source      text NOT NULL DEFAULT 'merge_review_contextual'
              CHECK (source IN ('merge_review_contextual', 'manual_admin', 'bulk_import'))
);

-- Dedup case-insensitive por entity_type
CREATE UNIQUE INDEX IF NOT EXISTS uq_taxonomy_blacklist_label_entity
  ON taxonomy_blacklist (lower(trim(label)), entity_type);

-- Lookup rápido por entity_type para validação durante criação de novos canonicals
CREATE INDEX IF NOT EXISTS idx_taxonomy_blacklist_entity_type
  ON taxonomy_blacklist (entity_type);

-- RLS standard (admin-only)
ALTER TABLE taxonomy_blacklist ENABLE ROW LEVEL SECURITY;

CREATE POLICY taxonomy_blacklist_admin_all ON taxonomy_blacklist
  FOR ALL
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'
    )
  );

COMMENT ON TABLE taxonomy_blacklist IS
  'Lista de labels banidos da criação automática de canonicals (roles/skills). Sprint atual: inserções via merge-canonicals split-button "Rejeitar e banir" (§2.35); consulta via reconcile_canonical_role/_skill (§2.13/§2.14). Sprint futura: CRUD completo via tela dedicada.';

COMMIT;
```

**Validação pós-migration:**

```sql
SELECT COUNT(*) FROM taxonomy_blacklist;
-- Esperado: 0 (tabela recém-criada, sem seed inicial)

SELECT indexname FROM pg_indexes WHERE tablename = 'taxonomy_blacklist';
-- Esperado: 3 (pk + uq + idx)

-- Confirma FK em added_by
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'taxonomy_blacklist'::regclass AND contype = 'f';
-- Esperado: 1 linha com REFERENCES profiles(id) ON DELETE SET NULL
```

**Decisão sobre nullable em `added_by` (M-3 Grok):** a coluna é `uuid` **sem `NOT NULL`** porque a FK usa `ON DELETE SET NULL`. Se o admin que adicionou o label for deletado de `profiles` no futuro, `added_by` vira NULL e o registro de blacklist sobrevive intacto (label continua banido, apenas perde-se a autoria). Alternativa `ON DELETE RESTRICT` impediria deletar admin que tenha qualquer banimento associado, criando débito operacional desnecessário.

**Integração com criação de canonicals:** já implementada nesta sprint — `reconcile_canonical_role` (§2.13) e `reconcile_canonical_skill` (§2.14) checam `taxonomy_blacklist` via `IF EXISTS` antes do INSERT e abortam com `RAISE EXCEPTION 'label_blacklisted'` (ERRCODE P0001). O caller TS captura e descarta o item silenciosamente (§3.2 e §3.3).

---

### §2.32 — `admin_panel_confidence_calculable` (Painel 11)

**Arquivo:** `32_admin_panel_confidence_calculable.sql`
**Atende:** cobre `role.confidence.min_count` e `skill.confidence.min_count` na §3.14.

**Base de contagem:** `vacancy_count` em `job_canonical_roles` e `job_canonical_skills` (coluna real confirmada). Não existe coluna dedicada `confidence_samples_count` — a base estatística de `confidence_median` é o próprio volume de vagas onde o canonical aparece. Em JCS adicionalmente existe `usage_count` (assimetria intencional vs JCR — registra frequência de uso em CVs/extrações além das vagas), mas para fins de "amostra suficiente para confidence_median ser estatisticamente válida", `vacancy_count` é a base canônica em ambas as tabelas.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_confidence_calculable(p_days int DEFAULT 30)
RETURNS jsonb
LANGUAGE sql STABLE AS $$
  WITH min_counts AS (
    SELECT
      (SELECT value::int FROM pipeline_config WHERE key = 'role.confidence.min_count')  AS role_min,
      (SELECT value::int FROM pipeline_config WHERE key = 'skill.confidence.min_count') AS skill_min
  ),
  role_buckets AS (
    SELECT
      COUNT(*) FILTER (WHERE jcr.vacancy_count >= mc.role_min) AS calculable,
      COUNT(*) FILTER (WHERE jcr.vacancy_count <  mc.role_min) AS not_calculable
    FROM job_canonical_roles jcr CROSS JOIN min_counts mc
    WHERE jcr.status = 'active'
  ),
  skill_buckets AS (
    SELECT
      COUNT(*) FILTER (WHERE jcs.vacancy_count >= mc.skill_min) AS calculable,
      COUNT(*) FILTER (WHERE jcs.vacancy_count <  mc.skill_min) AS not_calculable
    FROM job_canonical_skills jcs CROSS JOIN min_counts mc
    WHERE jcs.status = 'active'
  )
  SELECT jsonb_build_object(
    'snapshot', jsonb_build_object(
      'role',  (SELECT row_to_json(role_buckets.*)::jsonb  FROM role_buckets),
      'skill', (SELECT row_to_json(skill_buckets.*)::jsonb FROM skill_buckets)
    ),
    'thresholds', jsonb_build_object(
      'role_min_count',  (SELECT role_min FROM min_counts),
      'skill_min_count', (SELECT skill_min FROM min_counts)
    )
  );
$$;

COMMIT;
```

**Capacidade temporal:** painel é **snapshot puro** — mostra estado atual da proporção. Disponível em 24h/7d/30d com o mesmo dado (snapshot não muda por janela). Frontend exibe badge "snapshot — janela não aplicável" no canto do painel.

---

### §2.33 — `admin_panel_opus_review_queue` (Painel 12)

**Arquivo:** `33_admin_panel_opus_review_queue.sql`
**Atende:** cobre `role.opus_review.cooldown_days` e `skill.opus_review.cooldown_days` na §3.14.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_opus_review_queue(p_days int DEFAULT 7)
RETURNS jsonb
LANGUAGE sql STABLE AS $$
  WITH cooldowns AS (
    SELECT
      (SELECT value::int FROM pipeline_config WHERE key = 'role.opus_review.cooldown_days')  AS role_cooldown,
      (SELECT value::int FROM pipeline_config WHERE key = 'skill.opus_review.cooldown_days') AS skill_cooldown
  ),
  role_queue AS (
    SELECT
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.role_cooldown || ' days')::interval > NOW()
      ) AS in_cooldown,
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.role_cooldown || ' days')::interval <= NOW()
      ) AS eligible_for_rereview,
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.role_cooldown || ' days')::interval > NOW()
          AND oao.decided_at + (c.role_cooldown || ' days')::interval <= NOW() + (p_days || ' days')::interval
      ) AS exiting_cooldown_in_window
    FROM opus_arbitration_outcomes oao
    JOIN canonical_role_merge_candidates crmc ON crmc.id = oao.item_id
    CROSS JOIN cooldowns c
    WHERE oao.item_type = 'merge_candidate'
  ),
  skill_queue AS (
    SELECT
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.skill_cooldown || ' days')::interval > NOW()
      ) AS in_cooldown,
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.skill_cooldown || ' days')::interval <= NOW()
      ) AS eligible_for_rereview,
      COUNT(*) FILTER (
        WHERE oao.decided_at + (c.skill_cooldown || ' days')::interval > NOW()
          AND oao.decided_at + (c.skill_cooldown || ' days')::interval <= NOW() + (p_days || ' days')::interval
      ) AS exiting_cooldown_in_window
    FROM opus_arbitration_outcomes oao
    JOIN canonical_skill_merge_candidates csmc ON csmc.id = oao.item_id
    CROSS JOIN cooldowns c
    WHERE oao.item_type = 'merge_candidate'
  )
  SELECT jsonb_build_object(
    'snapshot', jsonb_build_object(
      'role',  (SELECT row_to_json(role_queue.*)::jsonb  FROM role_queue),
      'skill', (SELECT row_to_json(skill_queue.*)::jsonb FROM skill_queue)
    ),
    'window_days', p_days,
    'cooldowns', jsonb_build_object(
      'role_days',  (SELECT role_cooldown  FROM cooldowns),
      'skill_days', (SELECT skill_cooldown FROM cooldowns)
    )
  );
$$;

COMMIT;
```

**Lógica do painel:** 3 buckets por entidade — (a) `in_cooldown`: pares ainda dentro do cooldown, (b) `eligible_for_rereview`: pares onde cooldown já expirou (elegíveis para nova arbitragem), (c) `exiting_cooldown_in_window`: pares que VÃO sair do cooldown nos próximos `p_days` (janela prospectiva — informa carga de re-arbitragem esperada).

**Discriminação role/skill:** o `item_type = 'merge_candidate'` é unificado no CHECK constraint de `opus_arbitration_outcomes`. A separação entre arbitragens de role e de skill se dá pelo JOIN com `canonical_role_merge_candidates` (para `role_queue`) ou `canonical_skill_merge_candidates` (para `skill_queue`) — `oao.item_id` só dá match com a tabela correta. Padrão consistente com §2.16 e §2.26.

**Capacidade temporal:** disponível em 24h/7d/30d — `p_days` controla a janela prospectiva do bucket (c). Snapshot dos buckets (a) e (b) não muda por janela.

---

### §2.34 — `admin_panel_merge_review_queue` (Painel 13)

**Arquivo:** `34_admin_panel_merge_review_queue.sql`
**Atende:** cobre `role.merge_candidate.opus_review_cooldown_days` e `skill.merge_candidate.opus_review_cooldown_days` na §3.14.

**Esquema real de `canonical_role_merge_candidates` e `canonical_skill_merge_candidates`** (ambas têm o mesmo shape): `id`, `canonical_a_id`, `canonical_b_id`, `similarity`, `detected_at`, `resolved_at`, `opus_decision`, `opus_reasoning`, `arbitration_attempts`, `last_arbitration_attempt_at`. Note: os pares usam `canonical_a_id`/`canonical_b_id` (não `source_id`/`target_id` como em `*_merge_decisions`).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION admin_panel_merge_review_queue(p_days int DEFAULT 7)
RETURNS jsonb
LANGUAGE sql STABLE AS $$
  WITH cooldowns AS (
    SELECT
      (SELECT value::int FROM pipeline_config WHERE key = 'role.merge_candidate.opus_review_cooldown_days')  AS role_cooldown,
      (SELECT value::int FROM pipeline_config WHERE key = 'skill.merge_candidate.opus_review_cooldown_days') AS skill_cooldown
  ),
  role_queue AS (
    SELECT
      COUNT(*) FILTER (
        WHERE crmc.last_arbitration_attempt_at + (c.role_cooldown || ' days')::interval > NOW()
      ) AS in_cooldown,
      COUNT(*) FILTER (
        WHERE crmc.last_arbitration_attempt_at + (c.role_cooldown || ' days')::interval <= NOW()
      ) AS eligible_for_rereview,
      COUNT(*) FILTER (
        WHERE crmc.last_arbitration_attempt_at + (c.role_cooldown || ' days')::interval > NOW()
          AND crmc.last_arbitration_attempt_at + (c.role_cooldown || ' days')::interval <= NOW() + (p_days || ' days')::interval
      ) AS exiting_cooldown_in_window,
      COUNT(*) FILTER (
        WHERE crmc.arbitration_attempts >= 3
      ) AS attempt_limit_reached
    FROM canonical_role_merge_candidates crmc CROSS JOIN cooldowns c
    WHERE crmc.last_arbitration_attempt_at IS NOT NULL
      AND crmc.resolved_at IS NULL
  ),
  skill_queue AS (
    SELECT
      COUNT(*) FILTER (
        WHERE csmc.last_arbitration_attempt_at + (c.skill_cooldown || ' days')::interval > NOW()
      ) AS in_cooldown,
      COUNT(*) FILTER (
        WHERE csmc.last_arbitration_attempt_at + (c.skill_cooldown || ' days')::interval <= NOW()
      ) AS eligible_for_rereview,
      COUNT(*) FILTER (
        WHERE csmc.last_arbitration_attempt_at + (c.skill_cooldown || ' days')::interval > NOW()
          AND csmc.last_arbitration_attempt_at + (c.skill_cooldown || ' days')::interval <= NOW() + (p_days || ' days')::interval
      ) AS exiting_cooldown_in_window,
      COUNT(*) FILTER (
        WHERE csmc.arbitration_attempts >= 3
      ) AS attempt_limit_reached
    FROM canonical_skill_merge_candidates csmc CROSS JOIN cooldowns c
    WHERE csmc.last_arbitration_attempt_at IS NOT NULL
      AND csmc.resolved_at IS NULL
  )
  SELECT jsonb_build_object(
    'snapshot', jsonb_build_object(
      'role',  (SELECT row_to_json(role_queue.*)::jsonb  FROM role_queue),
      'skill', (SELECT row_to_json(skill_queue.*)::jsonb FROM skill_queue)
    ),
    'window_days', p_days,
    'cooldowns', jsonb_build_object(
      'role_days',  (SELECT role_cooldown  FROM cooldowns),
      'skill_days', (SELECT skill_cooldown FROM cooldowns)
    )
  );
$$;

COMMIT;
```

**Buckets agregados:** os 3 buckets do Painel 12 + `attempt_limit_reached` (pares que esgotaram as 3 tentativas de arbitragem e ficam fora do fluxo automático — útil para identificar pares que precisam intervenção manual). Filtro `resolved_at IS NULL` exclui pares já finalizados (merged/ignored/rejected) do snapshot da fila ativa.

**Capacidade temporal:** disponível em 24h/7d/30d (mesma lógica do Painel 12).

---

### §2.35 — RPC `reject_and_blacklist_canonical_pair` (split-button "Rejeitar e banir")

**Arquivo:** `35_reject_and_blacklist_canonical_pair.sql`
**Atende:** suporte ao endpoint `POST /api/admin/merge-canonicals/ignore` com parâmetro `add_to_blacklist: true` (§4.3). Substitui pseudocódigo do handler por RPC SECURITY DEFINER que garante atomicidade da transação (decisão + blacklist) num único round-trip.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION reject_and_blacklist_canonical_pair(
  p_candidate_id uuid,
  p_entity_type text,
  p_reason text,
  p_admin_id uuid
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'pg_temp'
AS $$
DECLARE
  v_canonical_a_id uuid;
  v_canonical_b_id uuid;
  v_a_label text;
  v_b_label text;
  v_blacklist_id uuid;
  v_decision_id uuid;
BEGIN
  IF p_entity_type NOT IN ('role', 'skill') THEN
    RAISE EXCEPTION 'invalid_entity_type: %', p_entity_type
      USING ERRCODE = 'invalid_parameter_value';
  END IF;

  -- 1. Resolve canonical IDs + labels do par via tabela correta
  IF p_entity_type = 'role' THEN
    SELECT crmc.canonical_a_id, crmc.canonical_b_id,
           jcr_a.canonical_label, jcr_b.canonical_label
    INTO v_canonical_a_id, v_canonical_b_id, v_a_label, v_b_label
    FROM canonical_role_merge_candidates crmc
    JOIN job_canonical_roles jcr_a ON jcr_a.id = crmc.canonical_a_id
    JOIN job_canonical_roles jcr_b ON jcr_b.id = crmc.canonical_b_id
    WHERE crmc.id = p_candidate_id;
  ELSE
    SELECT csmc.canonical_a_id, csmc.canonical_b_id,
           jcs_a.label, jcs_b.label
    INTO v_canonical_a_id, v_canonical_b_id, v_a_label, v_b_label
    FROM canonical_skill_merge_candidates csmc
    JOIN job_canonical_skills jcs_a ON jcs_a.id = csmc.canonical_a_id
    JOIN job_canonical_skills jcs_b ON jcs_b.id = csmc.canonical_b_id
    WHERE csmc.id = p_candidate_id;
  END IF;

  IF v_canonical_a_id IS NULL THEN
    RAISE EXCEPTION 'candidate_not_found: %', p_candidate_id
      USING ERRCODE = 'no_data_found';
  END IF;

  -- 2. Registra decisão de rejeição em *_merge_decisions
  IF p_entity_type = 'role' THEN
    INSERT INTO role_merge_decisions
      (source_id, target_id, status, actor_id, reason, decided_at)
    VALUES
      (v_canonical_a_id, v_canonical_b_id, 'rejected', p_admin_id, p_reason, NOW())
    RETURNING id INTO v_decision_id;
  ELSE
    INSERT INTO skill_merge_decisions
      (source_id, target_id, status, actor_id, reason, decided_at)
    VALUES
      (v_canonical_a_id, v_canonical_b_id, 'rejected', p_admin_id, p_reason, NOW())
    RETURNING id INTO v_decision_id;
  END IF;

  -- 3. Marca o candidato como resolvido
  IF p_entity_type = 'role' THEN
    UPDATE canonical_role_merge_candidates
    SET resolved_at = NOW()
    WHERE id = p_candidate_id;
  ELSE
    UPDATE canonical_skill_merge_candidates
    SET resolved_at = NOW()
    WHERE id = p_candidate_id;
  END IF;

  -- 4. INSERT na blacklist (canonical_a é o banido por convenção — ver D-PS-38)
  BEGIN
    INSERT INTO taxonomy_blacklist
      (label, entity_type, reason, added_by, source)
    VALUES
      (v_a_label, p_entity_type,
       format('Banido durante revisão em %s por %s — par: %s ↔ %s',
              to_char(NOW(), 'DD/MM/YYYY HH24:MI'),
              p_admin_id::text,
              v_a_label, v_b_label),
       p_admin_id, 'merge_review_contextual')
    RETURNING id INTO v_blacklist_id;
  EXCEPTION
    WHEN unique_violation THEN
      -- Label já banido — retorna info do registro existente para o frontend
      SELECT id INTO v_blacklist_id
      FROM taxonomy_blacklist
      WHERE lower(trim(label)) = lower(trim(v_a_label))
        AND entity_type = p_entity_type;

      RAISE EXCEPTION 'label_already_blacklisted: % (existing_id: %)',
        v_a_label, v_blacklist_id
        USING ERRCODE = 'unique_violation';
  END;

  RETURN jsonb_build_object(
    'decision', 'rejected',
    'decision_id', v_decision_id,
    'blacklisted', true,
    'blacklist_entry_id', v_blacklist_id,
    'banned_label', v_a_label
  );
END;
$$;

COMMENT ON FUNCTION reject_and_blacklist_canonical_pair IS
'Transação atômica: rejeita par + adiciona canonical_a à blacklist. Simétrico role/skill via p_entity_type. RAISE no UNIQUE violation aborta tudo (decision não persiste se blacklist falha). Chamado por POST /api/admin/merge-canonicals/ignore com add_to_blacklist=true.';

COMMIT;
```

**Tratamento do erro `label_already_blacklisted` no handler TS (§4.3):** o endpoint catch a exceção pelo `code='23505'` ou pela mensagem `'label_already_blacklisted'`, parseia o `existing_id` e retorna `409 Conflict { error: 'label_already_blacklisted', label, existing_blacklist_id }`. Frontend exibe mensagem específica permitindo "Apenas rejeitar" como fallback (sem reabrir o split-button).

---

## §3 TypeScript backend

### §3.1 — `lib/pipeline/_shared/evaluate-hard-gate.ts` (novo)

**Objetivo:** centralizar a avaliação do hard gate (lookup de `*.hard_gate.min_confidence` em `pipeline_config` + comparação).

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';

export type HardGateInput = {
  confidence: number | null | undefined;
};

export type HardGateResult = {
  passed: boolean;
  threshold: number;
  confidence: number | null;
};

export async function evaluateHardGate(
  supabase: SupabaseClient,
  configKey: 'role.hard_gate.min_confidence' | 'skill.hard_gate.min_confidence',
  input: HardGateInput,
): Promise<HardGateResult> {
  const { data, error } = await supabase
    .from('pipeline_config')
    .select('value')
    .eq('key', configKey)
    .maybeSingle();

  if (error) throw new Error(`pipeline_config lookup failed: ${error.message}`);

  const threshold = data ? Number(data.value) : 0.7;
  const confidence = input.confidence == null ? null : Number(input.confidence);

  return {
    passed: confidence != null && confidence >= threshold,
    threshold,
    confidence,
  };
}

export async function filterByHardGate<T extends HardGateInput>(
  supabase: SupabaseClient,
  configKey: 'role.hard_gate.min_confidence' | 'skill.hard_gate.min_confidence',
  items: T[],
): Promise<Array<T & { passedHardGate: boolean; hardGateThreshold: number }>> {
  if (items.length === 0) return [];

  const { data, error } = await supabase
    .from('pipeline_config')
    .select('value')
    .eq('key', configKey)
    .maybeSingle();

  if (error) throw new Error(`pipeline_config lookup failed: ${error.message}`);

  const threshold = data ? Number(data.value) : 0.7;

  return items.map((item) => {
    const confidence = item.confidence == null ? null : Number(item.confidence);
    return {
      ...item,
      passedHardGate: confidence != null && confidence >= threshold,
      hardGateThreshold: threshold,
    };
  });
}
```

---

### §3.2 — `lib/pipeline/extract-skills-from-resume.ts`

**Atende:** simetria com §3.3 — extração de skills do currículo, hard gate em batch, resolução de canonical_skill_id para aprovados, persistência em `resume_skills` (tabela criada em §2.8b) com **aprovados E reprovados** (decisão de produto desta sprint — auditoria simétrica skill↔role).

**Mudanças:**
- Função renomeada de `extractAndPersistAnalysisSkills` → `extractAndPersistResumeSkills` (simétrico a `extractAndPersistResumeRoles`).
- Parâmetro renomeado: `analysisId` → `resumeId`.
- Parâmetro `canonicalRoleId?` removido.
- Usa `filterByHardGate` em vez de leitura inline.
- **H-4 (decisão do Onsly):** grava **aprovados E reprovados** em `resume_skills`. `passed_hard_gate` mapeado corretamente para cada item. Canonização (`reconcile_canonical_skill`) só roda para aprovados.
- **B-6 (DeepSeek #10):** canonização paralela com chunks de 5 itens em vez de `Promise.all` ilimitado, evitando 50+ RPCs simultâneas em CVs com muitas skills.
- **C-4-b:** dedup usa `normalizeSkill()` de `lib/pipeline/text-processing.ts` (paridade SQL↔TS com `normalize_skill_label`), não `toLowerCase().trim()`. Garante que `C++` e `c++` sejam tratados como duplicatas; que `Integração` e `Integracao` colidem após normalização.
- UPSERT com `onConflict: 'resume_id,skill_normalized'` (coluna gerada criada em §2.8b — UNIQUE regular, detectável pelo Supabase JS).

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';
import { filterByHardGate } from './_shared/evaluate-hard-gate';
import { normalizeSkill } from './text-processing';

export type ExtractedSkill = {
  name: string;
  confidence: number;
};

export type ExtractSkillsResult = {
  hardGateMin: number;
  totalExtracted: number;
  passedHardGate: number;
  rejectedHardGate: number;
  canonicalsResolved: number;
};

const CANONIZATION_CHUNK_SIZE = 5;

async function resolveCanonicalsInChunks<T extends { name: string }>(
  supabase: SupabaseClient,
  items: T[],
): Promise<Array<T & { canonicalSkillId: string | null }>> {
  const results: Array<T & { canonicalSkillId: string | null }> = [];
  for (let i = 0; i < items.length; i += CANONIZATION_CHUNK_SIZE) {
    const chunk = items.slice(i, i + CANONIZATION_CHUNK_SIZE);
    const resolved = await Promise.all(
      chunk.map(async (s) => {
        try {
          const { data, error } = await supabase.rpc('reconcile_canonical_skill', {
            p_label: s.name,
            p_employer_id: null,
            p_source: 'resume_extraction',
          });
          if (error) {
            // P0001 'label_blacklisted' é caso esperado: descarta silenciosamente
            if (error.code === 'P0001' && error.message.includes('label_blacklisted')) {
              return { ...s, canonicalSkillId: null };
            }
            throw new Error(`reconcile_canonical_skill failed for "${s.name}": ${error.message}`);
          }
          return { ...s, canonicalSkillId: data as string };
        } catch (err) {
          if (err instanceof Error && err.message.includes('label_blacklisted')) {
            return { ...s, canonicalSkillId: null };
          }
          throw err;
        }
      }),
    );
    results.push(...resolved);
  }
  return results;
}

export async function extractAndPersistResumeSkills(
  supabase: SupabaseClient,
  resumeId: string,
  skills: ExtractedSkill[],
): Promise<ExtractSkillsResult> {
  const gated = await filterByHardGate(supabase, 'skill.hard_gate.min_confidence', skills);
  const threshold = gated[0]?.hardGateThreshold ?? 0.7;

  // Dedup global (aprovados + reprovados) usando normalizeSkill — paridade com normalize_skill_label
  // C++ e c++ → mesma key 'cpp'; Integração e Integracao → mesma key 'integracao'
  const dedupedByNormalized = new Map<string, typeof gated[number]>();
  for (const s of gated) {
    const normalized = normalizeSkill(s.name);
    if (!normalized) continue;
    const existing = dedupedByNormalized.get(normalized);
    // Em caso de duplicata, mantém o de maior confidence — preserva passed_hard_gate=true se um deles passou
    if (!existing || s.confidence > existing.confidence) {
      dedupedByNormalized.set(normalized, s);
    }
  }
  const dedupedAll = Array.from(dedupedByNormalized.values());

  // Separa para canonização (só aprovados resolvem canonical_skill_id)
  const approved = dedupedAll.filter((s) => s.passedHardGate);
  const rejected = dedupedAll.filter((s) => !s.passedHardGate);

  // Canonização em chunks de 5 (B-6)
  let resolvedApproved: Array<typeof approved[number] & { canonicalSkillId: string | null }> = [];
  if (approved.length > 0) {
    resolvedApproved = await resolveCanonicalsInChunks(supabase, approved);
  }

  const canonicalsResolved = resolvedApproved.filter((s) => s.canonicalSkillId !== null).length;

  // Monta linhas para upsert — aprovados + reprovados na mesma operação (H-4)
  const approvedRows = resolvedApproved.map((s) => ({
    resume_id: resumeId,
    skill: s.name,
    confidence: s.confidence,
    passed_hard_gate: true,
    canonical_skill_id: s.canonicalSkillId,
    source: 'resume_extraction',
  }));

  const rejectedRows = rejected.map((s) => ({
    resume_id: resumeId,
    skill: s.name,
    confidence: s.confidence,
    passed_hard_gate: false,
    canonical_skill_id: null,
    source: 'resume_extraction',
  }));

  const allRows = [...approvedRows, ...rejectedRows];

  if (allRows.length > 0) {
    const { error: upsertError } = await supabase
      .from('resume_skills')
      .upsert(allRows, { onConflict: 'resume_id,skill_normalized', ignoreDuplicates: false });

    if (upsertError) {
      throw new Error(`resume_skills upsert failed: ${upsertError.message}`);
    }
  }

  return {
    hardGateMin: threshold,
    totalExtracted: skills.length,
    passedHardGate: approved.length,
    rejectedHardGate: rejected.length,
    canonicalsResolved,
  };
}
```

**Caller upstream (`extract-skills-from-resume.ts:91` e `ingest-job-and-discover-skills.ts:82`):** correção do bug pré-existente apontado pelo Claude Code — substituir `skill.label.toLowerCase().trim()` por `normalizeSkill(skill.label)` ao popular `p_normalized` em `taxonomy_relations.source_term` e antes de salvar em `job_canonical_skills.label`. Sem essa correção, lookups falham silenciosamente para skills com acento ou símbolo (`C#`, `Integração Contínua`).

---

### §3.3 — `lib/pipeline/extract-roles-from-resume.ts` (novo)

**Atende:** N+1 ao canonizar roles (gate em batch antes do loop de canonização, simetria S-SYM-9).

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';
import { filterByHardGate } from './_shared/evaluate-hard-gate';
import { normalizeTitle } from './text-processing';

export type ExtractedRole = {
  role: string;
  confidence: number;
};

export type ExtractRolesResult = {
  hardGateMin: number;
  totalExtracted: number;
  passedHardGate: number;
  rejectedHardGate: number;
  canonicalsResolved: number;
};

const CANONIZATION_CHUNK_SIZE = 5;

async function resolveCanonicalsInChunks<T extends { role: string }>(
  supabase: SupabaseClient,
  items: T[],
): Promise<Array<T & { canonicalRoleId: string | null }>> {
  const results: Array<T & { canonicalRoleId: string | null }> = [];
  for (let i = 0; i < items.length; i += CANONIZATION_CHUNK_SIZE) {
    const chunk = items.slice(i, i + CANONIZATION_CHUNK_SIZE);
    const resolved = await Promise.all(
      chunk.map(async (r) => {
        try {
          const { data, error } = await supabase.rpc('reconcile_canonical_role', {
            p_canonical_label: r.role,
            p_employer_id: null,
            p_source: 'resume_extraction',
          });
          if (error) {
            // P0001 'label_blacklisted' é caso esperado: descarta silenciosamente
            if (error.code === 'P0001' && error.message.includes('label_blacklisted')) {
              return { ...r, canonicalRoleId: null };
            }
            throw new Error(`reconcile_canonical_role failed for "${r.role}": ${error.message}`);
          }
          return { ...r, canonicalRoleId: data as string };
        } catch (err) {
          if (err instanceof Error && err.message.includes('label_blacklisted')) {
            return { ...r, canonicalRoleId: null };
          }
          throw err;
        }
      }),
    );
    results.push(...resolved);
  }
  return results;
}

export async function extractAndPersistResumeRoles(
  supabase: SupabaseClient,
  resumeId: string,
  roles: ExtractedRole[],
): Promise<ExtractRolesResult> {
  const gated = await filterByHardGate(supabase, 'role.hard_gate.min_confidence', roles);
  const threshold = gated[0]?.hardGateThreshold ?? 0.7;

  // Dedup global (aprovados + reprovados) usando normalizeTitle — paridade com normalize_role_label.
  // C-4-b: corrige o bug pré-existente de usar toLowerCase().trim() que não strip acentos.
  const dedupedByNormalized = new Map<string, typeof gated[number]>();
  for (const r of gated) {
    const normalized = normalizeTitle(r.role);
    if (!normalized) continue;
    const existing = dedupedByNormalized.get(normalized);
    if (!existing || r.confidence > existing.confidence) {
      dedupedByNormalized.set(normalized, r);
    }
  }
  const dedupedAll = Array.from(dedupedByNormalized.values());

  // Separa aprovados/reprovados
  const approved = dedupedAll.filter((r) => r.passedHardGate);
  const rejected = dedupedAll.filter((r) => !r.passedHardGate);

  // Canonização em chunks de 5 (B-6) — só aprovados
  let resolvedApproved: Array<typeof approved[number] & { canonicalRoleId: string | null }> = [];
  if (approved.length > 0) {
    resolvedApproved = await resolveCanonicalsInChunks(supabase, approved);
  }

  const canonicalsResolved = resolvedApproved.filter((r) => r.canonicalRoleId !== null).length;

  // Monta linhas para upsert — aprovados + reprovados (H-4)
  const approvedRows = resolvedApproved.map((r) => ({
    resume_id: resumeId,
    role: r.role,
    confidence: r.confidence,
    passed_hard_gate: true,
    canonical_role_id: r.canonicalRoleId,
    source: 'resume_extraction', // requerido pelo CHECK constraint job_canonical_roles_source_check (§2.2)
  }));

  const rejectedRows = rejected.map((r) => ({
    resume_id: resumeId,
    role: r.role,
    confidence: r.confidence,
    passed_hard_gate: false,
    canonical_role_id: null,
    source: 'resume_extraction',
  }));

  const allRows = [...approvedRows, ...rejectedRows];

  if (allRows.length > 0) {
    const { error: upsertError } = await supabase
      .from('resume_roles')
      .upsert(allRows, { onConflict: 'resume_id,role_normalized', ignoreDuplicates: false });

    if (upsertError) {
      throw new Error(`resume_roles upsert failed: ${upsertError.message}`);
    }
  }

  return {
    hardGateMin: threshold,
    totalExtracted: roles.length,
    passedHardGate: approved.length,
    rejectedHardGate: rejected.length,
    canonicalsResolved,
  };
}
```

**Imports:** o módulo precisa importar `normalizeTitle` de `./text-processing` (paridade SQL↔TS com `normalize_role_label` — D-PS-54).

**Caller upstream:** mesma orientação de §3.2 — qualquer caller que gravava `taxonomy_relations.source_term` para roles via `lower(trim())` deve usar `normalizeTitle()` para garantir consistência com o lookup §2.6.

---

### §3.4 — `lib/resume-processor.ts`

**Mudanças:**
- Wiring de `extractAndPersistResumeRoles` (chamada após extração via LLM, linha ~195).
- Grava `skills_with_confidence` no `resumes` antes do gate.

**Formato esperado de `extractedSkills` (B5 confirmado):** vem do `ExtractedSkill[]` de §3.2, com tipo `{ name: string; confidence: number }`. Esse formato corresponde EXATAMENTE ao `COMMENT` declarado em §2.3 para a coluna `resumes.skills_with_confidence`:

```sql
COMMENT ON COLUMN resumes.skills_with_confidence IS
  'jsonb array: [{"name": "...", "confidence": 0.92}, ...]. Snapshot pré-gate.';
```

O update grava o array TypeScript diretamente — Postgres converte para `jsonb` no driver. Implementador NÃO deve renomear campos (`skillName` em vez de `name`, por exemplo) — quebra o consumo por queries SQL que esperam `->>'name'`. Tipo TypeScript correspondente em `lib/pipeline/extract-skills-from-resume.ts`:

```typescript
export type ExtractedSkill = {
  name: string;        // ← OBRIGATÓRIO este nome exato
  confidence: number;  // ← OBRIGATÓRIO este nome exato (0..1)
};
```

Edição cirúrgica — adicionar após o bloco que produz `extractedSkills` e `extractedRoles`:

```typescript
// Snapshot de skills com confidence para auditoria/explainability
await supabase
  .from('resumes')
  .update({ skills_with_confidence: extractedSkills })
  .eq('id', resumeId);

// Extração e persistência de roles (gate + canonização batch)
const rolesResult = await extractAndPersistResumeRoles(supabase, resumeId, extractedRoles);

console.log('[resume-processor] roles processed', {
  resumeId,
  ...rolesResult,
});
```

---

### §3.5 — `lib/analysis/compare/compare-fn.ts`

**Assinatura alvo de §3.2 pós-sprint:** `extractAndPersistResumeSkills(supabase, resumeId, skills)` — 3 parâmetros, sem `canonicalRoleId`.

**Localização da edição:** chamada existente de `extractAndPersistResumeSkills` em `compare-fn.ts`. Atualizar o call site para a nova assinatura: passar `resumeId` em vez de `analysisId` (renomeação) e remover o argumento `canonicalRoleId` (parâmetro suprimido em §3.2 — skill não pertence a uma role específica via análise; o vínculo skill↔role é derivado depois pelo motor de comparação a partir de `resume_roles`).

---

### §3.6 — `lib/analysis/compare/fetch-refs.ts`

**Mudança:** filtro `.eq('passed_hard_gate', true)` em queries que leem `resume_roles` e `resume_skills` (tabela criada em §2.8b) para evitar consumir registros que falharam no gate. Ambas tabelas têm a coluna `passed_hard_gate boolean NOT NULL DEFAULT false` com índice parcial em `WHERE passed_hard_gate = true` — query é eficiente. Anterior leitura em `analysis_skill_matches` para esse filtro era incorreta (tabela não tem a coluna; é cache de match calculado, conceitualmente downstream).

---

### §3.7 — 4 callers de hard gate migrados para shared

Substituir lookups inline de `pipeline_config` por chamadas a `evaluateHardGate` em:

1. `lib/pipeline/persist-curation/hard-gate.ts` — pipeline de curadoria de vagas (role).
2. `lib/pipeline/extract-skills-from-resume.ts` — já contemplado em §3.2.
3. `lib/pipeline/extract-roles-from-resume.ts` — já contemplado em §3.3.
4. `lib/analysis/compare/compare-fn.ts` — gate residual antes do scoring.

---

### §3.8 — `lib/pipeline/canonical-roles.ts`

**Mudança:** introduzir `resolveCanonicalRoleBySlug` via RPC `resolve_active_canonical_by_slug` (§2.15), substituindo lookups inline.

```typescript
export async function resolveCanonicalRoleBySlug(
  supabase: SupabaseClient,
  slug: string,
): Promise<string | null> {
  const { data, error } = await supabase.rpc('resolve_active_canonical_by_slug', {
    p_slug: slug,
    p_entity_type: 'role',
  });
  if (error) throw new Error(`resolve_active_canonical_by_slug failed: ${error.message}`);
  return (data as string) ?? null;
}
```

---

### §3.9 — `lib/admin/domain-backfill.ts` (novo)

**Frente A1.3.** Pattern CLI/Cron (não fire-and-forget Vercel — confirmação Onsly).
**Localização do arquivo:** `lib/admin/domain-backfill.ts` (segue convenção dos imports `@/lib/admin/*` já usados no projeto, ex.: `dashboard-day-aggregator`).
**Modelo:** `LLM_MODEL` (Sonnet 4.6) — Opus não se justifica para classificação batch sem decisão destrutiva.
**Threshold de aceite:** confidence ≥ 0.75 (`DOMAIN_CONFIDENCE_THRESHOLD`).
**Sanitização:** dupla — (a) slug retornado pelo LLM deve existir na lista carregada do banco; (b) `source = 'ai_backfill'` é rastreável e distinto dos demais valores válidos da coluna (`seed_cbo_mte_2002`, `manual_admin`, etc., dependendo dos seeds históricos).
**Decisão:** lista de domínios é carregada dinamicamente do banco no início do batch (resiliente a novos domínios adicionados por migration futura, sem necessidade de sincronizar hardcoded).

```typescript
import type { SupabaseClient } from '@supabase/supabase-js';
import { anthropic } from '@/lib/anthropic';
import { LLM_MODEL } from '@/lib/pipeline/constants';

const DOMAIN_CONFIDENCE_THRESHOLD = 0.75;
const BATCH_SIZE = 50;

interface DomainEntry {
  id: string;
  slug: string;
  name: string;
}

export interface DomainSuggestion {
  domain_id: string;
  slug: string;
  confidence: number;
}

export type BackfillResult = {
  processed: number;
  linked: number;
  skipped: number;
  failed: number;
};

/**
 * Sugere domínio para um cargo via LLM, com sanitização contra alucinação de slug.
 * Retorna null em três casos:
 *   1. confidence retornado < DOMAIN_CONFIDENCE_THRESHOLD
 *   2. JSON inválido ou campos ausentes na resposta
 *   3. Slug retornado não está na DOMAIN_LIST atual (alucinação)
 */
export async function suggestDomainForRole(
  roleLabel: string,
  domains: DomainEntry[],
): Promise<DomainSuggestion | null> {
  const domainMenu = domains.map((d) => `- ${d.slug}: ${d.name}`).join('\n');

  const response = await anthropic.messages.create({
    model: LLM_MODEL,
    max_tokens: 256,
    temperature: 0,
    system: `Você é um classificador de cargos corporativos brasileiros. Dado um cargo, identifique o domínio profissional mais adequado da lista abaixo e retorne um JSON com os campos domain_slug e confidence (0.0 a 1.0).

Domínios disponíveis:
${domainMenu}

Regras:
- Escolha apenas UM domínio — o mais específico possível.
- confidence >= 0.9: certeza clara. 0.75–0.89: plausível mas ambíguo. < 0.75: muito incerto (evite).
- Se o cargo claramente não pertence a nenhum domínio listado, retorne confidence: 0.
- Responda APENAS com JSON válido, sem texto adicional.

Formato de resposta:
{"domain_slug": "<slug>", "confidence": <0.0-1.0>}`,
    messages: [{ role: 'user', content: `Cargo: ${roleLabel}` }],
  });

  const text =
    response.content[0]?.type === 'text' ? response.content[0].text.trim() : '';

  let parsed: { domain_slug: string; confidence: number };
  try {
    parsed = JSON.parse(text);
  } catch {
    return null;
  }

  if (
    typeof parsed.domain_slug !== 'string' ||
    typeof parsed.confidence !== 'number' ||
    parsed.confidence < DOMAIN_CONFIDENCE_THRESHOLD
  ) {
    return null;
  }

  const domain = domains.find((d) => d.slug === parsed.domain_slug);
  if (!domain) return null;

  return {
    domain_id: domain.id,
    slug: domain.slug,
    confidence: parsed.confidence,
  };
}

export async function backfillCanonicalDomains(
  supabase: SupabaseClient,
  options: { dryRun?: boolean } = {},
): Promise<BackfillResult> {
  // 1. Carregar domínios ativos (dinâmico, não hardcoded)
  const { data: domains, error: domainsError } = await supabase
    .from('canonical_role_domains')
    .select('id, slug, name')
    .eq('is_active', true)
    .order('display_order');

  if (domainsError) throw new Error(`load domains failed: ${domainsError.message}`);
  if (!domains || domains.length === 0) {
    throw new Error('No active canonical_role_domains found — abort backfill');
  }

  // 2. Carregar canonicals ativos sem link em canonical_role_domain_links
  //    (M6: filtra explicitamente para idempotência — rerun não duplica nem incrementa
  //    failed indevidamente. Subquery via RPC porque PostgREST não suporta NOT IN com subquery.)
  const { data: alreadyLinked, error: linkedError } = await supabase
    .from('canonical_role_domain_links')
    .select('canonical_role_id');

  if (linkedError) throw new Error(`load existing links failed: ${linkedError.message}`);
  const linkedIds = new Set((alreadyLinked ?? []).map((l) => l.canonical_role_id));

  const { data: allActiveRoles, error: rolesError } = await supabase
    .from('job_canonical_roles')
    .select('id, canonical_label')
    .eq('status', 'active')
    .is('merged_into', null);

  if (rolesError) throw new Error(`load roles failed: ${rolesError.message}`);

  const roles = (allActiveRoles ?? []).filter((r) => !linkedIds.has(r.id));

  let processed = 0;
  let linked = 0;
  let skipped = 0;
  let failed = 0;

  for (let i = 0; i < roles.length; i += BATCH_SIZE) {
    const batch = roles.slice(i, i + BATCH_SIZE);

    for (const role of batch) {
      processed++;
      try {
        const suggestion = await suggestDomainForRole(
          role.canonical_label,
          domains,
        );

        if (!suggestion) {
          skipped++;
          continue;
        }

        if (options.dryRun) {
          linked++;
          continue;
        }

        const { error: linkError } = await supabase
          .from('canonical_role_domain_links')
          .insert({
            canonical_role_id: role.id,
            domain_id: suggestion.domain_id,
            is_primary: true,
            confidence: suggestion.confidence,
            source: 'ai_backfill',
          });

        if (linkError) {
          failed++;
          console.error('[domain-backfill] link insert failed', {
            roleId: role.id,
            error: linkError,
          });
          continue;
        }

        linked++;
      } catch (err) {
        failed++;
        console.error('[domain-backfill] processing failed', {
          roleId: role.id,
          err,
        });
      }
    }
  }

  return { processed, linked, skipped, failed };
}
```

**Endpoint:** expor via `app/api/admin/backfill/canonical-domains/route.ts` com guard de role admin + `dryRun` query param. Canonicals que ficarem sem domínio após o backfill (skipped) aparecem no Painel de Pendências para revisão manual.

---

### §3.10 — `lib/pipeline/persist-curation/constants.ts`

**Limpeza de event_names role-only:**

- Linha 22: `canonical_creation_blocked_missing_confidence` → `role_creation_blocked_missing_confidence`
- Linha 31: `canonical_creation_blocked_low_confidence` → `role_creation_blocked_low_confidence`

O `hard-gate.ts:80` faz pickup automático via `cfg.eventName` — nenhuma edição necessária ali.

---

### §3.11 — `app/api/cron/pipeline-maintenance/route.ts`

**Mudanças:**
- Linha 106: `supabase.rpc('catchup_pending_promotions')` → `supabase.rpc('catchup_pending_role_promotions')`
- Linha 108: atualizar string de log: `'[pipeline-maintenance] catchup_pending_role_promotions:'`
- Linhas 93, 99: atualizar comentários inline (cosmético, não funcional)

---

### §3.12 — `app/api/cron/audit-rpc-coverage/route.ts`

**Mudança:** linha 27 — atualizar whitelist removendo `catchup_pending_promotions` e adicionando `catchup_pending_role_promotions`. Adicionar também `auto_assign_family_to_role` se a whitelist incluir essa categoria.

---

### §3.13 — `tests/pipeline/sprint-v11-pr7-7a-integration.test.ts`

**Mudança:** linha 53 — atualizar `expect(sql).toContain('CREATE OR REPLACE FUNCTION fn_promote_role_on_threshold')`.

Se o teste verificava também o nome antigo via outro assert, atualizar. Se o arquivo de migration testado mudou de nome, atualizar referência.

---

### §3.14 — `lib/admin/pipeline-config-tooltips.ts` (novo)

> **Nota para extensão futura:** este mapeamento estático das 24 chaves de calibração será **reutilizado** pelo `IMPACT_SOURCES` da sprint orchestrator simétrico subsequente (que constrói o endpoint `GET /api/admin/pipeline-config/[key]/impact`). A próxima sprint deve **referenciar** este mapa, não duplicar a lista de chaves. Os nomes canônicos das 24 chaves estão cravados aqui — qualquer divergência futura entre `pipeline-config-tooltips.ts` e `IMPACT_SOURCES` é bug, não decisão.

**Objetivo:** mapeamento estático no frontend de `pipeline_config.key` → array de painéis afetados em `LimiaresTab`, com texto rico de tooltip por (chave, painel). Alimenta a coluna "Acompanhamento" da tabela em §5.1.4 e a lista de painéis na caixa "Impacto estimado" do modal de edição em §5.1.6.

**Ground truth:** mapeamento cobre as 24 chaves de calibração (12 `role.*` + 12 `skill.*`). As 2 chaves de sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) ficam de fora porque não aparecem na tela (§5.1.0). Todas as **24 chaves têm pelo menos 1 painel de validação** (cobertura 100% após inclusão dos painéis 11, 12 e 13 nesta sprint — sem chaves "às cegas").

**Estrutura do arquivo:**

```typescript
// lib/admin/pipeline-config-tooltips.ts

export type PanelHint = {
  panel: number;        // 1..10, mapeia para LimiaresTab panel_N
  label: string;        // título curto do painel (ex: "Hard Gate", "Zona Opus")
  text: string;         // tooltip rico explicando comportamento esperado pós-mudança
};

export const PIPELINE_CONFIG_TOOLTIPS: Record<string, PanelHint[]> = {
  // ───────────────────────────────────────────────────────────────────
  // HARD GATE — chaves que filtram entrada de canonicals
  // ───────────────────────────────────────────────────────────────────
  'skill.hard_gate.min_confidence': [
    { panel: 1, label: 'Hard Gate',     text: 'Taxa de descarte no Hard Gate. Subir o piso aumenta o volume rejeitado nas primeiras janelas após a publicação.' },
    { panel: 2, label: 'Zona Opus',     text: 'Volume entrando na Zona Opus tende a cair, já que o filtro anterior está mais restritivo. Esperar redução em 24-48h.' },
    { panel: 8, label: 'Distribuição',  text: 'Histograma de creation_confidence: a barra do piso para baixo deve concentrar o volume rejeitado pelo novo limiar.' },
  ],
  'role.hard_gate.min_confidence': [
    { panel: 4, label: 'Gate cumulativo', text: 'Backlog de funções pending cai porque novas funções abaixo do piso não entram mais no pipeline. Esperar redução em 24-48h.' },
    { panel: 8, label: 'Distribuição',    text: 'Histograma de creation_confidence (funções): a barra do piso para baixo deve concentrar o volume rejeitado pelo novo limiar.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Curva diária de rejeições cresce no dia da publicação; promoções caem em janela curta.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // PROMOÇÃO — auto_min_confidence (teto da Zona Opus)
  // ───────────────────────────────────────────────────────────────────
  'skill.promotion.auto_min_confidence': [
    { panel: 2, label: 'Zona Opus',    text: 'Após elevar o teto, mais habilidades passam a entrar na arbitragem Opus. A faixa entre o teto anterior e o novo deve concentrar volume crescente no horizonte de 7 dias.' },
    { panel: 7, label: 'Distribuição', text: 'Histograma de confidence_median: a barra imediatamente abaixo do novo teto deve subir, e a barra acima dele deve cair.' },
  ],
  'role.promotion.auto_min_confidence': [
    { panel: 7,  label: 'Distribuição',          text: 'Distribuição da confidence_median na promoção (funções). Subir o teto faz mais funções entrarem em Zona Opus em vez de auto-promoção.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Taxa de promoção automática cai; volume de arbitragem Opus sobe na janela imediata.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // PROMOÇÃO — min_vacancies e min_distinct_employers (gate cumulativo)
  // ───────────────────────────────────────────────────────────────────
  'role.promotion.min_vacancies': [
    { panel: 4,  label: 'Gate cumulativo', text: 'Taxa de promoção pending→active cai conforme exige mais vagas. Backlog de pendings cresce no curto prazo — acompanhar tendência.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Curva de promoções cai imediatamente; rejeições mantêm-se estáveis.' },
  ],
  'skill.promotion.min_vacancies': [
    { panel: 4,  label: 'Gate cumulativo', text: 'Taxa de promoção pending→active (habilidades) cai conforme exige mais vagas. Backlog cresce no curto prazo.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Curva de promoções de habilidades cai imediatamente; rejeições mantêm-se estáveis.' },
  ],
  'role.promotion.min_distinct_employers': [
    { panel: 4, label: 'Gate cumulativo', text: 'Complementa min_vacancies via AND — exige diversidade de empregadores além do volume. Backlog inclui pendings com vagas suficientes mas concentradas em poucos empregadores.' },
  ],
  'skill.promotion.min_distinct_employers': [
    { panel: 4, label: 'Gate cumulativo', text: 'Idem para habilidades — AND com min_vacancies. Endurece exigência de diversidade de empregadores.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // MERGE CANDIDATES — cosine_threshold (detecção)
  // ───────────────────────────────────────────────────────────────────
  'skill.merge_candidate.cosine_threshold': [
    { panel: 3, label: 'Unificação', text: 'Volume de merge candidates detectados (habilidades). Subir o threshold reduz pares sugeridos — a fila de revisão de unificação encolhe.' },
  ],
  'role.merge_candidate.cosine_threshold': [
    { panel: 3, label: 'Unificação', text: 'Volume de merge candidates detectados (funções). Subir o threshold reduz pares sugeridos. Habitualmente exige threshold mais alto que skills devido a labels mais ambíguos.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // LOOKBACK — janelas móveis (afetam universo de cálculo)
  // ───────────────────────────────────────────────────────────────────
  'role.confidence.lookback_days': [
    { panel: 7, label: 'Distribuição', text: 'Janela móvel de cálculo de confidence_median. Janelas curtas refletem ruído recente; longas suavizam mas demoram a reagir a mudanças.' },
  ],
  'skill.confidence.lookback_days': [
    { panel: 7, label: 'Distribuição', text: 'Janela móvel de cálculo de confidence_median (habilidades). Encurtar aumenta variância da mediana; alongar atrasa detecção de derivas.' },
  ],
  'role.promotion.lookback_days': [
    { panel: 4,  label: 'Gate cumulativo', text: 'Janela para acumular vagas/empregadores para promoção (funções). Encurtar reduz universo elegível — backlog cresce.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Curva de promoções cai inicialmente, recupera conforme novas vagas entram na janela.' },
  ],
  'skill.promotion.lookback_days': [
    { panel: 4,  label: 'Gate cumulativo', text: 'Idem para habilidades — encurtar reduz universo elegível para promoção pending→active.' },
    { panel: 10, label: 'Promoções vs Rejeições', text: 'Curva de promoções de habilidades reage com a mesma lógica das funções.' },
  ],
  'role.merge_candidate.lookback_days': [
    { panel: 3, label: 'Unificação', text: 'Janela para varrer candidatos de merge (funções). Encurtar reduz volume detectado mesmo com threshold fixo.' },
  ],
  'skill.merge_candidate.lookback_days': [
    { panel: 3, label: 'Unificação', text: 'Janela para varrer candidatos de merge (habilidades). Mesma dinâmica das funções.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // RETIREMENT — gap_days (aposentadoria por inatividade)
  // ───────────────────────────────────────────────────────────────────
  'role.retirement.gap_days': [
    { panel: 6, label: 'Aposentadoria por gap', text: 'Quanto menor o gap, mais funções saem por inatividade. Subir o gap retém mais canonicals ativos mesmo sem vagas recentes.' },
  ],
  'skill.retirement.gap_days': [
    { panel: 6, label: 'Aposentadoria por gap', text: 'Idem para habilidades — define quanto tempo sem latest_posted_at antes de aposentar.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // CONFIDENCE MIN_COUNT — Painel 11 (§2.32)
  // ───────────────────────────────────────────────────────────────────
  'role.confidence.min_count': [
    { panel: 11, label: 'Confidence calculável', text: 'Proporção de funções ativas com vacancy_count suficiente para confidence_median ser estatisticamente válida. Subir o mínimo encolhe o universo "calculável" — funções com poucas vagas passam para "não calculável" e ficam fora dos painéis 7/8.' },
  ],
  'skill.confidence.min_count': [
    { panel: 11, label: 'Confidence calculável', text: 'Mesma lógica para habilidades. Universo "calculável" cresce/encolhe conforme o mínimo. Subir demais pode esvaziar a base estatística dos painéis 7/8 do lado skill.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // OPUS REVIEW COOLDOWN (geral) — Painel 12 (§2.33)
  // ───────────────────────────────────────────────────────────────────
  'role.opus_review.cooldown_days': [
    { panel: 12, label: 'Fila de re-revisão Opus', text: 'Funções arbitradas pelo Opus aguardando cooldown expirar antes de poder ser re-arbitradas. Encurtar o cooldown libera pares mais rapidamente para nova rodada — fila "elegível" cresce, custo Opus subsequente tende a subir.' },
  ],
  'skill.opus_review.cooldown_days': [
    { panel: 12, label: 'Fila de re-revisão Opus', text: 'Idem para habilidades. Acompanhar fila "em cooldown" vs "elegível" antes e depois da mudança.' },
  ],

  // ───────────────────────────────────────────────────────────────────
  // MERGE CANDIDATE OPUS REVIEW COOLDOWN — Painel 13 (§2.34)
  // ───────────────────────────────────────────────────────────────────
  'role.merge_candidate.opus_review_cooldown_days': [
    { panel: 13, label: 'Fila de re-revisão de unificação', text: 'Pares de funções já arbitrados (decisão MERGE/KEEP_BOTH/NEEDS_HUMAN) aguardam expiração do cooldown antes de poder ser re-arbitrados como candidatos de unificação. Encurtar libera pares mais rapidamente.' },
  ],
  'skill.merge_candidate.opus_review_cooldown_days': [
    { panel: 13, label: 'Fila de re-revisão de unificação', text: 'Idem para habilidades. Útil quando admin observa que pares relevantes não voltam para revisão (cooldown longo demais) ou que pares já analisados re-aparecem cedo demais (cooldown curto demais).' },
  ],
};
```

**Decisão arquitetural:** manter no frontend em vez de coluna jsonb no banco porque (a) os textos são longos e específicos por (chave, painel), (b) mudam pouco, (c) evita 1 query extra por linha da tabela. Quando uma chave nova for adicionada a `pipeline_config`, o desenvolvedor adiciona o mapeamento aqui no mesmo PR.

**Gate de CI (recomendado):** teste unitário em `__tests__/pipeline-config-tooltips.spec.ts` valida que toda chave esperada (constante `PIPELINE_CONFIG_KEYS: readonly string[]` derivada do seed §2.19a) tem entrada em `PIPELINE_CONFIG_TOOLTIPS`, mesmo que array vazio. Sem I/O, sem servidor — bloqueia merge se desenvolvedor adicionar chave nova sem mapeamento correspondente.

**Mapa dos 12 painéis ativos em `LimiaresTab.tsx`:**

| # | Painel | Origem |
|---|---|---|
| 1 | Hard Gate | pré-existente |
| 2 | Promoção vs Zona Opus | pré-existente |
| 3 | Merge Candidate | pré-existente |
| 4 | Gate cumulativo (pending com volume) | pré-existente |
| 5 | Pending stuck (>30 dias) | pré-existente |
| 6 | Aposentadoria por gap | pré-existente |
| 7 | Distribuição confidence_median (promoção) | pré-existente |
| 8 | Distribuição creation_confidence | pré-existente |
| 10 | Promoções vs Rejeições por dia | pré-existente |
| 11 | Confidence calculável | §2.32 |
| 12 | Fila de re-revisão Opus | §2.33 |
| 13 | Fila de re-revisão de merge candidates | §2.34 |

Gap em `9` é intencional na numeração — preserva referências SQL existentes em `LimiaresTab.tsx` após reorganização. Total: 12 painéis ativos.

**Painel sem referência via tooltips:** panel_5 (Pending stuck) é multi-fator (pode ser stuck por falta de vagas, empregadores, samples...) — não responde a chave única. Mantido como alerta operacional ad-hoc.

**Cobertura final:** 24/24 chaves com pelo menos 1 painel de validação. Nenhuma chave editável fica sem feedback visual no painel — atende ao propósito da tela (calibração com feedback, não às cegas).

---

## §4 Endpoints admin

### §4.1 — `/api/admin/pipeline-config`

**Métodos primários:**
- `GET /api/admin/pipeline-config` — lista todas as chaves com filtros opcionais via query string (`scope`, `criticality`, `q`). Retorna: `key`, `value`, `description`, `criticality_level`, `updated_at`, `updated_by_label`. **Ground truth confirmado:** `pipeline_config.updated_by` é coluna `text` (não uuid) — armazena strings como `'system'` ou nome de actor já resolvido. Handler retorna `updated_by_label = row.updated_by` direto, sem JOIN com `profiles`. Quando `updated_by` é `NULL`, retornar `'sistema'` como fallback. Mapeamento de painéis afetados vem de `lib/admin/pipeline-config-tooltips.ts` (§3.14), aplicado client-side após a resposta — não no payload do banco.
- `GET /api/admin/pipeline-config/[key]/history` — retorna até 50 entradas de `pipeline_config_history` para a chave, ordenadas por `changed_at DESC`. Cada entrada inclui `previous_value`, `new_value`, `changed_by_label` (resolvido para email/nome), `reason`, `changed_at`, e flag `is_seed` (true quando `previous_value IS NULL`).
- `PATCH /api/admin/pipeline-config/[key]` — body: `{ value, reason, confirmed }`. Chama a RPC `set_pipeline_config_value` (§2.20). Retorna erro 400 se `criticality_level = 'high'` e `confirmed != true`.

  **Trecho TS literal do handler — parâmetros nomeados DEVEM bater com a RPC SQL de §2.20:**

  ```typescript
  const { data, error } = await supabase.rpc('set_pipeline_config_value', {
    p_key: params.key,                       // text (path param)
    p_value: body.value,                     // text — NÃO usar 'p_new_value'
    p_changed_by: adminUserId,               // uuid — NÃO usar 'p_actor_id'
    p_reason: body.reason ?? null,           // text | null
    p_confirmed: body.confirmed === true,    // boolean — OBRIGATÓRIO mesmo para low/medium
  });

  if (error) {
    // RPC retorna RAISE EXCEPTION para criticality high sem p_confirmed=true
    if (error.message.includes('high criticality change requires explicit confirmation')) {
      return NextResponse.json({ error: 'confirmation_required' }, { status: 400 });
    }
    if (error.message.includes('pipeline_config key not found')) {
      return NextResponse.json({ error: 'key_not_found' }, { status: 404 });
    }
    return NextResponse.json({ error: 'update_failed', detail: error.message }, { status: 500 });
  }
  ```

  **Bloqueador resolvido (CR-2 — Grok #2, rodada v2.8):** a versão anterior do §4.1 mencionava a chamada à RPC sem cravar os nomes dos parâmetros, deixando margem para o implementador chutar `p_new_value` / `p_actor_id` (nomes que NÃO existem em `set_pipeline_config_value`). PostgREST retornaria "Function not found" (mismatch de nomes) ou ignoraria o `p_confirmed` (default `false`), bloqueando toda edição de criticidade high. Os nomes corretos são os declarados em §2.20: `p_key`, `p_value`, `p_changed_by`, `p_reason`, `p_confirmed`. Implementador deve copiar o trecho acima literalmente.

**Endpoints derivados para KPIs (mockup):**
- `GET /api/admin/pipeline-config/summary` — KPI 1: total de chaves (`SELECT COUNT(*) FROM pipeline_config`). Resposta: `{ total: 24 }`.
- `GET /api/admin/pipeline-config/changes-summary?days=30` — KPI 2: breakdown de mudanças por criticidade nos últimos N dias. Resposta: `{ total: 7, by_criticality: { high: 5, medium: 1, low: 1 } }`. Lê de `pipeline_config_history` filtrando `changed_at >= NOW() - days::interval`.
- `GET /api/admin/pipeline-config/last-publication` — KPI 3: última entrada de `pipeline_config_history`. Resposta: `{ changed_at, changed_by_label, key, time_ago_label }`.

**Endpoint de rollback :**
- `POST /api/admin/pipeline-config/[key]/rollback` — body: `{ target_history_id, reason, confirmed }`. Lê a entrada `target_history_id` de `pipeline_config_history`, valida que pertence à chave, e chama `set_pipeline_config_value(key, target.previous_value, current_user, reason, confirmed)`. Funciona como qualquer outra edição — passa pelo mesmo gate de criticidade `high` exigindo `confirmed=true`.

  **Trecho TS literal do handler — parâmetros nomeados batem com §2.20 (paridade com PATCH):**

  ```typescript
  // 1. Buscar a entrada de histórico alvo
  const { data: target, error: targetErr } = await supabase
    .from('pipeline_config_history')
    .select('key, previous_value, new_value, changed_at')
    .eq('id', body.target_history_id)
    .eq('key', params.key)   // valida que a entrada pertence à chave do path
    .single();

  if (targetErr || !target) {
    return NextResponse.json({ error: 'history_entry_not_found' }, { status: 404 });
  }

  // 2. Guard contra rollback de seed inicial (previous_value IS NULL)
  if (target.previous_value === null) {
    return NextResponse.json({
      error: 'cannot_rollback_to_seed',
      message: 'Não é possível reverter para o estado anterior ao seed inicial (valor era NULL).',
    }, { status: 400 });
  }

  // 3. Chamar a RPC com nomes EXATOS de §2.20
  const { data, error } = await supabase.rpc('set_pipeline_config_value', {
    p_key: params.key,
    p_value: target.previous_value,         // text — reverter para o valor anterior
    p_changed_by: adminUserId,
    p_reason: body.reason ?? 'rollback via /api/admin/pipeline-config/[key]/rollback',
    p_confirmed: body.confirmed === true,
  });

  if (error) {
    if (error.message.includes('high criticality change requires explicit confirmation')) {
      return NextResponse.json({ error: 'confirmation_required' }, { status: 400 });
    }
    return NextResponse.json({ error: 'rollback_failed', detail: error.message }, { status: 500 });
  }
  ```

**Guard contra rollback de seed inicial:** o handler DEVE validar `if (!target.previous_value) return 400` antes de chamar a RPC (passo 2 acima). A entrada de seed inicial tem `previous_value IS NULL` (não havia valor anterior). Reverter para `NULL` quebraria todas as leituras que fazem `value::int` ou `value::numeric`. A UI bloqueia o botão de rollback no seed inicial (§5.1.7), mas o handler precisa do guard correspondente — chamada direta via curl contornaria a proteção do front. Erro 400 com mensagem: `"Não é possível reverter para o estado anterior ao seed inicial (valor era NULL)."`

Guard padrão: middleware de role admin em todas. Validação de tipo do valor (numérico para chaves numéricas, JSON para chaves estruturadas).

---

### §4.2 — `/api/admin/merge-canonicals/audit`

**Método:** `GET` — query params `entity_type` (`role`|`skill`|`all`), `decision` (`merged`|`ignored`|`rejected`|`MERGE`|`KEEP_BOTH`|`NEEDS_HUMAN`|`all`), `actor` (`ai`|`admin`|`all`), `from`, `to`, `limit` (default 20), `cursor`.

**Nota sobre o filtro `decision`:** o select da UI (§5.2.3) usa labels PT-BR (Unificar/Manter ambos/Rejeitar/Revisar), mapeados via `DECISION_LABELS` (§5.2.6) para os valores aceitos pelo backend. O endpoint aceita ambos os formatos (lowercase de `*_merge_decisions.status` para auditoria de decisões executadas; UPPERCASE de `opus_arbitration_outcomes.decision` para auditoria de recomendações Opus). Decisão arquitetural: filtro client-side traduz label → conjunto de valores backend antes de chamar o endpoint.

**Lógica de roteamento por coluna no handler (B4 fechado):** o `decision` recebido na query string mapeia para 1 das 2 colunas da view `v_merge_audit_history` conforme o valor:

| Valor recebido | Coluna filtrada | SQL aplicado |
|---|---|---|
| `merged`, `ignored`, `rejected`, `pending` (lowercase) | `status` | `WHERE status = $1` |
| `MERGE`, `KEEP_BOTH`, `NEEDS_HUMAN` (UPPERCASE) | `decision` | `WHERE decision = $1` |
| `all` ou ausente | nenhum filtro | sem WHERE |

Para o admin filtrar por "Manter ambos" no select da UI, o frontend traduz para `decision=ignored` (cobre todos os casos visualmente equivalentes onde `status='ignored'`). Para filtrar por "Rejeitar", traduz para `decision=rejected`. O label "Revisar" do select traduz para `decision=NEEDS_HUMAN` (UPPERCASE, vai para a outra coluna). Combinações (lowercase + UPPERCASE) não são suportadas no MVP — o filtro é singular.

Por simplicidade do handler, recomenda-se um helper `getDecisionFilter(value: string): { column: 'status' | 'decision'; value: string } | null` no endpoint.

Lê da view `v_merge_audit_history` (§2.16). Paginação por cursor em `decided_at`. Resposta inclui campos necessários para a tabela do Card 1 (§5.2.1): `when_label` ("há Xh"/"ontem"/"há X dias"), `decision_label` (PT-BR via tradução §5.2.4), `from_label`, `to_label`, `similarity` (%), `actor_label`, `cost_usd_formatted`.

**Endpoint de expansão (para linha "Expandir"):**
- `GET /api/admin/merge-canonicals/audit/[id]/payload` — query: `entity_type` (`role`|`skill`). Retorna JSON detalhado consolidando `role_merge_decisions` (ou `skill_merge_decisions`) + `canonical_role_merge_candidates` (ou `canonical_skill_merge_candidates`) + `opus_arbitration_outcomes` correspondente. Usado pela linha expandida (ed-row) do mockup.

---

### §4.3 — `/api/admin/merge-canonicals/manual` + endpoints de decisão

**Card 2 do mockup (unificação manual legada):**
- `POST /api/admin/merge-canonicals/manual/analyze` — body: `{ source: text, target: text, entity_type: 'role'|'skill' }`. Resolve ambos via `lookup_canonical_role_by_normalized_label`/`lookup_canonical_skill_by_normalized_label` e retorna preview de impacto (vínculos afetados, skill_type compatibility).
- `POST /api/admin/merge-canonicals/manual/execute` — body: `{ source_id, target_id, reason, entity_type, cross_type_confirmed? }`. Chama `merge_canonicals(...)` ou `merge_skills(..., p_cross_type_confirmed)`. Para skills cross-type sem `cross_type_confirmed=true`, retorna 409 com payload `{ requires_cross_type_confirmation: true, source_type, target_type }` (preserva contrato existente do `merge_skills`).

**Card 3 — endpoints de decisão sobre par sugerido pela IA:**
- `POST /api/admin/merge-canonicals/ignore` — body: `{ candidate_id, decision, reason?, entity_type, add_to_blacklist? }`. Valores válidos de `decision`: `'ignored'` (Manter ambos), `'rejected'` (Apenas rejeitar). Endpoint aceita os 2 valores via `decision || 'ignored'` default — sem alteração de schema.
- **Parâmetro `add_to_blacklist: boolean` (default `false`):** quando `true` e `decision='rejected'`, o handler executa transação atômica:

```typescript
// Pseudo-código do handler (apenas para clarificar o fluxo)
const result = await supabase.rpc('reject_and_blacklist_canonical_pair', {
  p_candidate_id: body.candidate_id,
  p_entity_type: body.entity_type,
  p_reason: body.reason,
  p_admin_id: ctx.user.id,
});
// RPC SECURITY DEFINER faz internamente, dentro de uma transação:
//   1. INSERT INTO *_merge_decisions (status='rejected', actor_id, reason)
//   2. SELECT canonical_a_id, canonical_b_id FROM canonical_*_merge_candidates WHERE id = p_candidate_id
//      → JOIN com job_canonical_roles/skills para obter labels reais
//   3. INSERT INTO taxonomy_blacklist (label, entity_type, reason, added_by, source)
//        VALUES (<a_label>, p_entity_type, 'Banido durante revisão em [data] por [admin] — par: [a_label] ↔ [b_label]', p_admin_id, 'merge_review_contextual')
//   4. Se INSERT da blacklist falhar (UNIQUE violation porque label já está banido), RAISE EXCEPTION 'already_blacklisted'
//      → transação inteira ROLLBACK
```

**Resposta:**
- `200 OK { decision: 'rejected', blacklisted: true, blacklist_entry_id: uuid }` em sucesso
- `409 Conflict { error: 'label_already_blacklisted', label, existing_blacklist_id }` se o label já estava banido — admin recebe mensagem específica e pode escolher "Apenas rejeitar" como fallback (sem reabrir o split-button)

**Determinação do label banido:** o esquema de `canonical_*_merge_candidates` usa `canonical_a_id`/`canonical_b_id` (simétrico, não há "source"/"target" como em `*_merge_decisions`). Convenção: bane o label associado a `canonical_a_id` (o "primeiro" do par). Caso o admin queira banir AMBOS, precisa de 2 cliques separados via tela própria de Blacklist (sprint futura).

---

### §4.4 — Endpoint do painel lateral de Detalhes (mockup §5.2.3 botão "Detalhes")

- `GET /api/admin/merge-canonicals/[id]/details` — query: `entity_type`. Retorna estrutura para o painel lateral: `opus_reasoning` completo do candidato + comparação lado-a-lado (vínculos por canonical, evolução temporal de `confidence_median`, top 5 vagas/CVs recentes por canonical).

---

### §4.5 — Endpoints dos painéis admin instrumentados (registry §2.17)

A tabela `admin_panel_functions` é registry para telemetria — não cria endpoints novos. As 29 funções listadas no seed mapeiam para endpoints já existentes ou criados nas §4.1-§4.4:

| panel_id | Base de endpoints |
|---|---|
| `pricing` | `/api/admin/pricing/*` (existente) |
| `merge_canonicals` | `/api/admin/merge-canonicals/*` (§4.2 + §4.3 + §4.4) |
| `campaigns` | `/api/admin/campaigns/*` (existente) |
| `ingestor` | `/api/admin/ingest/*` + `/api/admin/curate-job-postings/*` (existentes) |
| `pipeline_config` | `/api/admin/pipeline-config/*` (§4.1) |

Cada endpoint passa a declarar `panel_id` + `function_id` em metadata de log para auditoria de uso.

---

### §4.6 — A1.3 endpoints (canonical_role_domains)

- `GET /api/admin/role-domains` — lista domínios ativos (`SELECT id, name, slug FROM canonical_role_domains WHERE is_active = true ORDER BY display_order`).
- `GET /api/admin/canonical-roles-by-domain?domain_id=...` — lista canonicals roles linkadas ao domínio. Consumido pelo `DomainRoleCascade` (§5.3) em `mode='admin'`.

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const url = new URL(req.url);
  const domainId = url.searchParams.get('domain_id');

  if (!domainId) {
    return NextResponse.json({ error: 'domain_id required' }, { status: 400 });
  }

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase
    .from('canonical_role_domain_links')
    .select(`
      canonical_role_id,
      job_canonical_roles!inner (
        id,
        canonical_label,
        status,
        vacancy_count
      )
    `)
    .eq('domain_id', domainId);

  if (error) {
    return NextResponse.json({ error: 'query_failed' }, { status: 500 });
  }

  // Inclui canonicals em pending (com indicador visual no frontend) — diferente
  // do endpoint público §4.8.2 que retorna apenas active.
  const roles = (data ?? []).map(r => ({
    id: r.job_canonical_roles.id,
    label: r.job_canonical_roles.canonical_label,
    status: r.job_canonical_roles.status,
    vacancy_count: r.job_canonical_roles.vacancy_count,
  }));

  return NextResponse.json({ roles });
}
```

- `GET /api/admin/roles/[id]/domain-link` — link atual.
- `PUT /api/admin/roles/[id]/domain-link` — body: `{ domain_id, reason }`. Valida `domain_id` contra `canonical_role_domains.is_active = true`.

**Bloqueador resolvido (C-5 — Grok #1):** o endpoint `POST /api/admin/backfill/canonical-domains` foi **removido** desta sprint. Versão anterior dispararia `backfillCanonicalDomains` como fire-and-forget na Vercel — em ambiente serverless, a Promise morre quando o isolate é congelado após a response, então a função batch (que roda 1-2h em ~616 roles) NÃO completaria. §3.9 documenta o pattern correto: **CLI/Cron**.

**Como Antigravity dispara o backfill:**

```bash
# Da máquina do desenvolvedor ou EC2, conectando direto ao banco de produção
npx tsx scripts/run-domain-backfill.ts --dry-run     # primeiro: confirma escopo
npx tsx scripts/run-domain-backfill.ts               # depois: executa de verdade
```

O script `scripts/run-domain-backfill.ts` (entry-point CLI) importa `backfillCanonicalDomains` de `lib/admin/domain-backfill.ts` e invoca com `await` (não fire-and-forget). Conteúdo mínimo:

```typescript
// scripts/run-domain-backfill.ts
import { createSupabaseAdminClient } from '@/lib/supabase/admin';
import { backfillCanonicalDomains } from '@/lib/admin/domain-backfill';

const dryRun = process.argv.includes('--dry-run');

(async () => {
  const supabase = createSupabaseAdminClient();
  const result = await backfillCanonicalDomains(supabase, { dryRun });
  console.log(JSON.stringify(result, null, 2));
  process.exit(0);
})().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

- `GET /api/admin/backfill/canonical-domains/status` — acompanhamento de progresso (este SIM permanece como endpoint, é só consulta de SELECT). Retorna `{ total_roles_active, roles_with_domain, pct_covered }`. Implementação:

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const supabase = createSupabaseServerClient();

  const { count: roleCount } = await supabase
    .from('job_canonical_roles')
    .select('id', { count: 'exact', head: true })
    .eq('status', 'active');

  // Subquery via .in() resolveria N+1 em produção. Para o tamanho atual (~616
  // roles), inline é aceitável; refatorar para RPC dedicada se piorar.
  const { data: linkedRoleIds } = await supabase
    .from('canonical_role_domain_links')
    .select('canonical_role_id');

  const linkedSet = new Set((linkedRoleIds ?? []).map(r => r.canonical_role_id));
  const linkedCount = linkedSet.size;

  return NextResponse.json({
    total_roles_active: roleCount,
    roles_with_domain: linkedCount,
    pct_covered: roleCount ? Math.round((linkedCount / roleCount) * 100) : 0,
  });
}
```

**Tela §5.1 (Backfill IA section):** o botão "Disparar backfill" da tela admin **deve apenas exibir o comando CLI a copiar**, não fazer chamada HTTP — uma vez disparado da CLI, o admin acompanha o progresso via `GET .../status`. Alternativa para sprint futura: integração com Inngest, Trigger.dev, ou Vercel Cron Jobs com timeout estendido — fora do escopo desta cleanup.

---

### §4.7 — Endpoints `/api/admin/limiares` (5 painéis B7 do banco)

Cada painel SQL criado em §2.26-§2.30 é consumido por um handler TS no endpoint existente `/api/admin/limiares`. Os 5 painéis B7 são **adicionais** aos 10 painéis já existentes em `LimiaresTab.tsx` — não substituem nem reordenam.

#### §4.7.1 — Painel B7 #1 — `GET /api/admin/limiares/panel_b7_1_custo_opus`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 30);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_custo_opus_por_canonical', {
    p_days: days,
  });

  if (error) {
    logger.error('[panel_b7_1] RPC failed', { error });
    return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  }

  return NextResponse.json({
    panel: 'b7_1_custo_opus',
    days,
    rows: data ?? [],
    available_windows: ['7d', '30d'],
  });
}
```

#### §4.7.2 — Painel B7 #2 — `GET /api/admin/limiares/panel_b7_2_merge_auto_vs_manual`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 30);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_merge_auto_vs_manual', { p_days: days });
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'b7_2_merge_auto_vs_manual',
    days,
    data,
    available_windows: ['7d', '30d'],
  });
}
```

#### §4.7.3 — Painel B7 #3 — `GET /api/admin/limiares/panel_b7_3_latest_posted_distribution`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_latest_posted_distribution');
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'b7_3_latest_posted_distribution',
    data,
    available_windows: ['30d'],  // bandas temporais 7d/30d/60d são internas ao painel
  });
}
```

#### §4.7.4 — Painel B7 #4 — `GET /api/admin/limiares/panel_b7_4_opus_effectiveness`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const supabase = createSupabaseServerClient();

  // Lê view direta (não RPC) — view pré-existente assumida via §2.30 validator
  const { data, error } = await supabase
    .from('v_opus_effectiveness')
    .select('*');

  if (error) {
    if (error.code === '42P01') {
      // View não existe — §2.30 emite WARNING mas migration continua; aqui defendemos no runtime
      return NextResponse.json({ error: 'view_v_opus_effectiveness_missing' }, { status: 503 });
    }
    return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  }

  return NextResponse.json({
    panel: 'b7_4_opus_effectiveness',
    rows: data ?? [],
    available_windows: ['7d', '30d'],
  });
}
```

#### §4.7.5 — Painel B7 #5 — `GET /api/admin/limiares/panel_b7_5_suggestion_rejected_by_skill`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 30);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_suggestion_rejected_by_skill', {
    p_days: days,
  });
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'b7_5_suggestion_rejected_by_skill',
    days,
    rows: data ?? [],
    available_windows: ['7d', '30d'],
  });
}
```

**Nota:** todos os 5 endpoints B7 são `GET`, read-only, admin-only via `requireAdmin`. Cache HTTP curto (60s) recomendado no nível do CDN. Cada endpoint vive em arquivo `route.ts` próprio dentro de `app/api/admin/limiares/[slug]/route.ts` no Next.js App Router — não há roteamento por switch num único handler.

#### §4.7.6 — Painel 11 — `GET /api/admin/limiares/panel_11_confidence_calculable`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 30);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_confidence_calculable', {
    p_days: days,
  });
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'p11_confidence_calculable',
    days,
    data: data ?? null,
    available_windows: ['24h', '7d', '30d'],
  });
}
```

#### §4.7.7 — Painel 12 — `GET /api/admin/limiares/panel_12_opus_review_queue`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 7);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_opus_review_queue', {
    p_days: days,
  });
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'p12_opus_review_queue',
    days,
    data: data ?? null,
    available_windows: ['24h', '7d', '30d'],
  });
}
```

#### §4.7.8 — Painel 13 — `GET /api/admin/limiares/panel_13_merge_review_queue`

```typescript
export async function GET(req: Request) {
  await requireAdmin(req);
  const days = Number(new URL(req.url).searchParams.get('days') ?? 7);

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase.rpc('admin_panel_merge_review_queue', {
    p_days: days,
  });
  if (error) return NextResponse.json({ error: 'panel_unavailable' }, { status: 500 });
  return NextResponse.json({
    panel: 'p13_merge_review_queue',
    days,
    data: data ?? null,
    available_windows: ['24h', '7d', '30d'],
  });
}
```

#### §4.7.9 — Contrato de capacidade temporal por painel

Cada endpoint da §4.7 retorna `available_windows` no body. Frontend (`LimiaresTab.tsx`) consulta esse campo e renderiza placeholder quando a janela atual está fora da capacidade:

```typescript
type PanelResponse = {
  panel: string;
  days: number;
  data: unknown;
  available_windows: ('24h' | '7d' | '30d')[];
};

if (!response.available_windows.includes(currentWindow)) {
  return <PanelDisabled
    panelLabel="Pending stuck (>30 dias)"
    reason="Janela curta demais — selecione 30 dias para ver dados."
  />;
}
```

**Matriz declarativa de capacidade dos 12 painéis ativos:**

| # | Painel | 24h | 7d | 30d |
|---|---|---|---|---|
| 1 | Hard Gate | ✓ | ✓ | ✓ |
| 2 | Promoção vs Zona Opus | ✓ | ✓ | ✓ |
| 3 | Merge Candidate | ✓ | ✓ | ✓ |
| 4 | Gate cumulativo | ✗ | ✓ | ✓ |
| 5 | Pending stuck (>30d) | ✗ | ✗ | ✓ |
| 6 | Aposentadoria por gap | ✗ | ✗ | ✓ |
| 7 | Distribuição confidence_median (promoção) | ⚠ | ✓ | ✓ |
| 8 | Distribuição creation_confidence | ⚠ | ✓ | ✓ |
| 10 | Promoções vs Rejeições por dia | ✓ | ✓ | ✓ |
| 11 | Confidence calculável | ✓ | ✓ | ✓ |
| 12 | Fila de re-revisão Opus | ✓ | ✓ | ✓ |
| 13 | Fila de re-revisão de merge candidates | ✓ | ✓ | ✓ |

Painéis com ⚠ exibem o gráfico normalmente com badge "amostra reduzida" no canto.

**Estrutura de rotas (N9):** cada endpoint da §4.7.1 a §4.7.8 vive em arquivo `route.ts` próprio em `app/api/admin/limiares/[slug]/route.ts`. Total: 8 arquivos `route.ts` novos a serem criados no SUB-PR-11 (não 1 handler único com switch).

---

### §4.8 — Endpoints públicos de cascata área→função (A1.3)

Os endpoints `/api/admin/role-domains` (§4.6) são admin-only e usam tokens de admin. Para a cascata área→função ser consumida por **usuários finais** (formulário público de criação de perfil profissional, futuro fluxo de "outra função"), expomos endpoints públicos sem autenticação que retornam apenas dados já públicos.

#### §4.8.1 — `GET /api/taxonomy/domains`

```typescript
export async function GET() {
  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase
    .from('canonical_role_domains')
    .select('id, name')
    .eq('is_active', true)
    .order('display_order', { ascending: true });

  if (error) {
    logger.error('[taxonomy.domains] failed', { error });
    return NextResponse.json({ domains: [] }, { status: 500 });
  }

  return NextResponse.json({ domains: data ?? [] });
}
```

**Cache:** edge cache de 300s (`Cache-Control: public, s-maxage=300`). Lista muda raramente.

#### §4.8.2 — `GET /api/taxonomy/roles-by-domain?domain_id=...`

```typescript
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const domainId = searchParams.get('domain_id');
  if (!domainId) return NextResponse.json({ roles: [] });

  const supabase = createSupabaseServerClient();
  const { data, error } = await supabase
    .from('canonical_role_domain_links')
    .select(`
      canonical_role_id,
      job_canonical_roles!inner ( id, canonical_label, status )
    `)
    .eq('domain_id', domainId)
    .eq('job_canonical_roles.status', 'active')
    .order('job_canonical_roles.canonical_label');

  if (error) {
    logger.error('[taxonomy.roles-by-domain] failed', { error, domainId });
    return NextResponse.json({ roles: [] }, { status: 500 });
  }

  const roles = (data ?? []).map((r: any) => ({
    id: r.job_canonical_roles.id,
    label: r.job_canonical_roles.canonical_label,
  }));

  return NextResponse.json({ roles });
}
```

**Cache:** edge cache de 300s.

**Diferença vs §4.6:**

| Aspecto | §4.6 (admin) | §4.8 (público) |
|---|---|---|
| Auth | requireAdmin | nenhuma |
| Dados expostos | `id, name, slug, display_order, is_active` | apenas `id, name` (domínios) e `id, label` (roles) |
| Cache | curto/sem cache | edge 300s |
| Consumer | tela admin, ResourceEditModal | formulário público (DomainRoleCascade quando montado em página pública) |

---

## §5 Frontend

### §5.1 — `/admin/pipeline-config` (Calibração de Limiares)

Tela nova, baseada no mockup `admin-tuning-pipeline-config.html`. Rota: `/admin/pipeline-config` (separada de `/admin/dashboard?tab=limiares` — esta última já existe com 10 painéis de observabilidade construídos em `LimiaresTab`).

**Divergências mockup vs spec (implementar pela spec, não pelo mockup):**

1. **Filtro de Escopo:** o mockup mostra `<option value="opus.*">opus.*</option>` que captura 0 chaves. A spec §5.1.3 define `*.opus_review*` que captura corretamente 4 chaves. Implementação deve seguir a spec.

2. **"Impacto estimado" no modal:** o mockup exibe valores dinâmicos como `"~12 habilidades hoje em zona auto migrarão para arbitragem Opus"` e `"Custo Opus estimado: +$0,43/mês"`. A spec §5.1.6 e D-PS-33 são explícitos em proibir endpoint dinâmico no MVP. Implementação deve usar tooltips estáticos de `pipeline-config-tooltips.ts` (§3.14).

Mockup permanecerá com esses elementos como referência visual de layout; texto/dados serão substituídos pelos da spec na implementação real.

A relação entre as duas: `LimiaresTab` exibe métricas operacionais (Painéis 1-10). `/admin/pipeline-config` é a tela de calibração das chaves `pipeline_config` que **alimentam** esses 10 painéis. A integração visual entre elas é via tags na coluna "Acompanhamento" (§5.1.5) — cada chave de config aponta para os painéis afetados, com tooltip explicando o comportamento esperado.

#### §5.1.0 — Escopo: 24 chaves de calibração simétrica

A tabela `pipeline_config` tem 26 chaves no banco (ground truth). A tela `/admin/pipeline-config` exibe **apenas 24 chaves** — 12 `role.*` + 12 `skill.*` perfeitamente espelhadas. As outras 2 chaves (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) são **de sistema** (feature flag binária + janela de quarentena) e ficam fora desta tela por 3 motivos:

1. Não têm a estrutura `role.* | skill.*` simétrica da família de calibração — quebrariam o agrupamento visual.
2. `CURATE_PIPELINE_ENABLED` é binária — não cabe no modal de edição numérica (§5.1.6) e exigiria UI específica.
3. Ambas afetam comportamentos operacionais de CRON, não thresholds de pipeline — escopo conceitualmente distinto.

**Implementação do filtro:** o endpoint `GET /api/admin/pipeline-config` (§4.1) já filtra `WHERE key LIKE 'role.%' OR key LIKE 'skill.%'`. As 2 chaves de sistema continuam acessíveis via SQL direto ou tela operacional separada (fora do escopo desta sprint).

**KPI 1** ("Chaves do pipeline") mostra `24`, vindo de `SELECT COUNT(*) FROM pipeline_config WHERE key LIKE 'role.%' OR key LIKE 'skill.%'` — não é hardcoded.

#### §5.1.1 — Header

```
Calibração de Limiares
24 chaves de pipeline_config que governam os 10 painéis Limiares
```

Ícone à esquerda do título (gráfico de barras). Subtítulo em cinza claro abaixo, com referência a `pipeline_config` em fonte mono.

#### §5.1.2 — KPIs (3 cards no topo)

| KPI | Valor primário | Métrica secundária | Endpoint |
|---|---|---|---|
| **Chaves do pipeline** | total (ex: 24) | "configuráveis pelo admin" | `GET /api/admin/pipeline-config/summary` |
| **Mudanças (30d)** | count total (ex: 7) | breakdown por criticidade ("5 ALTO · 1 MÉDIO · 1 BAIXO") | `GET /api/admin/pipeline-config/changes-summary?days=30` |
| **Última publicação** | label relativo (ex: "há 3 dias") | autor + chave alterada (ex: "Onsly · skill.promotion.auto_min_confidence") | `GET /api/admin/pipeline-config/last-publication` |

#### §5.1.3 — Filtros

Linha horizontal acima da tabela:
- Input "Buscar chave..." (filtro client-side por `key` ou `description`).
- Select "Escopo": opções `Todos os escopos`, `role.*`, `skill.*`, `*.opus_review*` (regex contains; a última opção captura as 4 chaves de cooldown anti-reflag Opus: `role.opus_review.cooldown_days`, `skill.opus_review.cooldown_days`, `role.merge_candidate.opus_review_cooldown_days`, `skill.merge_candidate.opus_review_cooldown_days`).
- Select "Criticidade": opções `Todas criticidades`, `Alto`, `Médio`, `Baixo`.
- Contador à direita: "X de Y" (X = chaves após filtro, Y = total).

#### §5.1.4 — Tabela de chaves (6 colunas)

```
| Criticidade | Chave + descrição | Atual | Acompanhamento | Última alt. | Ações |
```

- **Criticidade:** badge colorido (vermelho/amarelo/verde para alto/médio/baixo). Centralizado.
- **Chave + descrição:** `key` em fonte mono na linha de cima, `description` em cinza menor na linha de baixo (vem de coluna `description` de `pipeline_config` — ver §2.19a).
- **Atual:** valor atual em fonte mono, alinhado à direita.
- **Acompanhamento:** tags `Painel X` separadas por `·`, cada uma com tooltip rico **hardcoded por chave** em `lib/admin/pipeline-config-tooltips.ts` (ver §3.14). Para chaves sem painel associado: `—`.
- **Última alt.:** ícone de relógio (SVG) + label relativo "há Xd". Toda a célula é clicável e abre o **modal de histórico** (§5.1.7).
- **Ações:** botão `Editar` que abre o **modal de edição** (§5.1.6). Sempre via modal — não há edição inline em nenhuma criticidade.

Legenda visível abaixo da tabela:
- ALTO: thresholds de gate/promotion — exige "Digite PUBLICAR"
- MÉDIO: janelas e cooldowns — confirmação simples
- BAIXO: descrições e labels

#### §5.1.5 — Coluna "Acompanhamento" — tooltips por chave

Cada chave em `pipeline_config` mapeia para 1..N painéis afetados em `LimiaresTab`. O mapeamento é **estático no frontend**, em `lib/admin/pipeline-config-tooltips.ts`. A definição canônica do arquivo (cobertura 24/24) está em **§3.14** — fonte única da verdade. Esta seção apenas ilustra o consumo.

**Exemplo de uso (extraído de §3.14):**

```typescript
import { PIPELINE_CONFIG_TOOLTIPS } from '@/lib/admin/pipeline-config-tooltips';

const hints = PIPELINE_CONFIG_TOOLTIPS[row.key] ?? [];
// hints = [
//   { panel: 1, label: 'Hard Gate', text: 'Taxa de descarte no Hard Gate...' },
//   { panel: 2, label: 'Zona Opus', text: 'Volume entrando na Zona Opus tende a cair...' },
//   { panel: 8, label: 'Distribuição', text: 'Histograma de creation_confidence...' },
// ]
```

A coluna "Acompanhamento" da tabela §5.1.4 renderiza, para cada `hint`, uma tag `Painel N` clicável (link para `/admin/dashboard?tab=limiares#panel-{N}`). O atributo `title` da tag carrega o `text` do hint — exibido como tooltip nativo on hover.

Para chaves com `hints = []`, a célula mostra `—` (em cinza claro). Justificativa de manter hardcoded em vez de coluna jsonb no banco: os textos são longos, específicos por (chave, painel) e mudam pouco. Constante no client evita 1 query extra por linha da tabela.



#### §5.1.6 — Modal de edição de chave

> **Nota para extensão futura:** a caixa "Impacto estimado" desta cleanup é **qualitativa** (lista de painéis afetados + texto explicativo + rodapé de horizonte de acompanhamento), alimentada pelos tooltips de §3.14. A sprint orchestrator simétrico subsequente **ADICIONA** componente `ImpactPreview` quantitativo (tabela de contagem + histograma de distribuição + seletor de janela 7d/30d/90d) ao MESMO modal — **sem substituir** esta caixa qualitativa. As duas convivem: a caixa qualitativa orienta o admin sobre quais painéis acompanhar; o `ImpactPreview` quantifica o impacto numérico da mudança proposta sobre o universo de items no banco. D-PS-33 (acima) foi atualizada com esta nota.

Abre ao clicar `Editar` em qualquer linha. Estrutura:

- **Header:** badge de criticidade + título "Editar valor" + `key` em fonte mono. Botão ✕ no canto.
- **Descrição:** caixa cinza com texto explicativo da chave (vem do mesmo `description` da §2.19a).
- **Comparação de valores:** grid 2 colunas com seta no meio. À esquerda "Valor atual" (somente leitura, fundo cinza). À direita "Novo valor" (input editável).
- **Impacto estimado (caixa com bullets):** **estática, alimentada pelos tooltips de §3.14** (não calculada no backend — sem endpoint de cálculo dinâmico no MVP). Para a chave em edição, a caixa lista:
  - Os painéis afetados (mesmas tags `Painel X` da coluna "Acompanhamento" da §5.1.4)
  - Para cada painel, o `text` do `PanelHint` correspondente — texto qualitativo descrevendo o efeito esperado pós-mudança (ex: "Taxa de descarte no Hard Gate cresce nas primeiras janelas após a publicação")
  - Recomendação de horizonte de acompanhamento (rodapé da caixa): "Acompanhe os painéis acima por 24-48h após a publicação para confirmar o efeito esperado."
- **Decisão arquitetural (D-PS-33 reafirmada):** sem cálculo dinâmico de "X canonicals afetados" ou "custo estimado +$Y" — isso exigiria endpoint dedicado e contagem em tempo real do banco, que adicionam complexidade sem benefício comparável ao texto qualitativo. Se em sprint futura quisermos números concretos, criamos `GET /api/admin/pipeline-config/[key]/impact?new_value=X`.
- **Box "Criticidade ALTA exige confirmação textual"** (apenas em `criticality_level='high'`):
  - Texto vermelho-claro: "Digite PUBLICAR para confirmar:"
  - Input vazio que aceita texto literal
  - Botão "Publicar" (estilo danger) — `disabled` até o texto = exatamente `PUBLICAR`. Ao clicar: `PATCH /api/admin/pipeline-config/[key]` com `confirmed=true`.
- Em `criticality_level='medium'` ou `'low'`: sem box vermelha, botão "Publicar" direto (mesmo endpoint com `confirmed=false`).

#### §5.1.7 — Modal de histórico por chave

Abre ao clicar no relógio "há Xd" de qualquer linha. **Modal centralizado, não drawer.** Estrutura:

- **Header:** ícone de relógio + "Histórico de alterações" + `key` em mono.
- **Descrição:** "Valor atual: `X` · `N` alterações registradas em `pipeline_config_history` (auditoria imutável via RLS FORCE)".
- **Tabela cronológica reversa**, 6 colunas:
  - `De` — valor anterior (em mono); `—` para entrada de seed inicial.
  - `→` — seta.
  - `Para` — valor novo (em mono).
  - `Autor` — `changed_by_label` (resolvido para nome/email).
  - `Quando` — label relativo "há Xd".
  - `Ação` — botão de rollback (ícone ↺) que abre o modal de edição preenchido com o `previous_value` daquela entrada como `new_value`. **Para seed inicial:** sem botão de rollback, texto "seed inicial" cinza.

**Como o frontend identifica uma entrada como "seed inicial" (D-1, Claude Code PO A):** a fonte da verdade é o campo `previous_value` da entrada de histórico — entrada com `previous_value IS NULL` é a inserção original durante a migration §2.19 (seed das 24 chaves), pois não havia valor anterior. Toda edição subsequente via `set_pipeline_config_value` grava `previous_value = current_value_before_change`, nunca NULL. Esse critério é simétrico ao guard do backend em `POST /[key]/rollback` (§4.1, passo 2 do trecho TS literal), portanto frontend e backend reconhecem "seed" pela mesma condição (`previous_value === null`). O endpoint `GET /api/admin/pipeline-config/[key]/history` retorna `is_seed: previous_value === null` no payload de cada entrada para conveniência do frontend, evitando duplicação da lógica no client. **Não usar `reason = 'seed inicial'`** como critério, pois `reason` é texto livre informado pelo admin nas edições — não confiável para detectar a entrada inicial.

**Endpoint usado:** `GET /api/admin/pipeline-config/[key]/history` (§4.1).
**Rollback flow:** clique no ↺ → modal de edição abre com `new_value = entry.previous_value` → admin confirma (gate "PUBLICAR" se criticidade alta) → `POST /api/admin/pipeline-config/[key]/rollback` com body `{ target_history_id, reason, confirmed }`. O endpoint dedicado é preferido sobre PATCH genérico porque (a) tem guard explícito contra `previous_value IS NULL` (§4.1), evitando reverter para `NULL` que quebra leituras `value::int`/`value::numeric`, e (b) registra `origin='rollback'` em metadata do evento, facilitando auditoria de "o que foi edição manual vs reversão histórica".

#### §5.1.8 — Coluna `description` em pipeline_config

Para alimentar §5.1.4 e §5.1.6, `pipeline_config` precisa ter coluna `description text` populada. A migration §2.19a cobre tanto a garantia da coluna quanto o seed completo das 24 chaves operacionais (textos canônicos cravados — implementação deve usar exatamente os valores definidos lá, sem reescrita). As 2 chaves de sistema (`CURATE_PIPELINE_ENABLED`, `QUARANTINE_EXPIRY_DAYS`) ficam fora do seed por convenção (§2.19a documenta).

---

### §5.2 — `/admin/merge-canonicals` (Unificação de Cadastros — pilha-degrau)

Refactor da tela atual, baseado no mockup `admin-merge-canonicals-fichas-v4.html` (versão atual com split-button + linguagem de produto). Rota: `/admin/merge-canonicals` (preservada).

#### §5.2.1 — Header

```
Unificação de Cadastros          [7 aguardando revisão]
Revisão humana, auditoria de decisões automáticas e ferramenta manual
```

- Título: ícone de árvore de dependências + "Unificação de Cadastros" (alinhar com label real do AdminNav, ver §5.4).
- Badge à direita: total agregado de itens em revisão humana (soma das duas abas), cor amarela (warn).
- Subtítulo: descrição funcional.

#### §5.2.2 — Abas e layout pilha-degrau

Duas abas no topo:
- **Funções** (count badge ao lado)
- **Habilidades** (count badge ao lado)

A escolha de aba persiste em `localStorage`. Cada aba renderiza um **stack de 3 cards** com layout em degraus visuais e interação FLIP.

**Estrutura visual de degraus:**
- Card 1 (topo): largura = `calc(100% - 36px)`, posição não-ativa
- Card 2 (meio): largura = `calc(100% - 18px)`, posição não-ativa
- Card 3 (base): largura = `100%`, **posição ativa por padrão**

Os 3 cards são empilhados verticalmente com `margin-bottom: -12px` nos não-ativos (sobreposição parcial) e `box-shadow` proeminente no ativo.

**Interação FLIP:**
- Clicar no header de um card não-ativo dispara `swapStack()`.
- O card clicado **desce** para a posição ativa (recebe `.active`, expande conteúdo).
- O card que era ativo **sobe** para a posição do clicado.
- Animação via técnica FLIP (First, Last, Invert, Play) — `getBoundingClientRect` antes, reordenação no DOM via `insertBefore`/`appendChild`, transform invertido, depois `transition: transform .45s cubic-bezier(0.4,0,0.2,1)`.
- Acessibilidade: cada header é `tabindex="0"` com `role="button"` e `aria-expanded`. Teclado: Enter/Space dispara o swap.
- `prefers-reduced-motion`: a animação é desabilitada via media query.

**Diretrizes CSS de performance (B-3 — Grok #4):** o payload da view `v_merge_audit_history` pode ser grande (centenas de decisões com `opus_reasoning` aninhado). Sem isolamento, a animação FLIP recalcula layout do conteúdo dos cards não-ativos enquanto eles fazem `transform` + `width` — engasgo perceptível em notebooks de admin. Mitigação:

```css
.stack-card {
  /* Container isolation: layout/paint dentro do card não afeta vizinhos */
  contain: layout paint;
}

.stack-card:not(.active) {
  /* Conteúdo dos não-ativos não interage durante animação */
  overflow: hidden;
  pointer-events: none;
}

.stack-card.animating {
  /* Hint ao compositor para usar GPU layer durante a animação;
     remover após a transição via .animating cleanup no JS */
  will-change: transform, width;
}

/* Cleanup do will-change após transitionend (no JS):
   element.classList.remove('animating'); */
```

`will-change` é aplicado apenas durante a animação (não constante) para evitar reservar GPU layer permanentemente — anti-pattern conhecido em FLIP-heavy UIs.

**Os 3 cards por aba:**

| Posição | Funções | Habilidades |
|---|---|---|
| Card 1 (topo) | Auditoria de decisões automáticas (roles) | Auditoria de decisões automáticas (skills) |
| Card 2 (meio) | Unificação manual de funções (legado) | Unificação manual de habilidades (legado) |
| Card 3 (base, ativo) | Aguardando revisão humana (roles) | Aguardando revisão humana (skills) |

#### §5.2.3 — Card 1 — Auditoria de decisões automáticas

Sub-header do card:
- Título: "Auditoria de decisões automáticas"
- Subtítulo: "Últimos 30 dias · N decisões · $X,YY" (contador + custo total)
- Badge à direita: total de decisões no período

Conteúdo (quando ativo):

**Filtros (linha horizontal):**
- Input "Buscar canônico..."
- Select "Decisão": Todas / Unificar / Manter ambos / Rejeitar / Revisar
- Select "Período": Últimos 7 dias / Últimos 30 dias / Últimos 90 dias / Todo o histórico
- Select "Ator": Todos / IA / Admin manual
- Para Habilidades: filtro adicional "Tipo" (Técnica / Comportamental / Híbrida)

**Tabela (7 colunas):**
- Quando — label relativo
- Decisão — badge colorido (verde UNIFICAR / cinza MANTER AMBOS / amarelo REVISAR)
- Tipo (apenas para Habilidades) — badge com cor de `skill_type`
- De → Para — labels dos canonicals
- Sim. — similaridade %
- Ator — badge (roxo IA / azul Admin)
- Custo — USD formatado (`$0,0143`) ou `—` para Admin manual
- Botão "Expandir" — abre linha colapsada com payload JSON detalhado

**Linha expandida (`ed-row`):**
- Background diferenciado (lilás claro)
- Conteúdo: payload JSON em fonte mono com syntax highlighting (keys roxas, strings verdes, números rosa) mostrando o consolidado:
  - `role_merge_decisions` (ou `skill_merge_decisions`): `id`, `status`, `similarity`, `actor_id`
  - `canonical_role_merge_candidates` (ou `canonical_skill_merge_candidates`): `opus_reasoning` completo
  - `opus_arbitration_outcomes`: `decision`, `confidence`, `cost_usd`, todas as flags `action_*` (true/false)
- Vem do endpoint `GET /api/admin/merge-canonicals/audit/[id]/payload` (§4.2).

**Paginação:** "Mostrando X de Y" + Anterior / Página A de B / Próxima.

**Note inferior (caixa azul-claro) — texto técnico para referência arquitetural:**
> Painel alimentado pela view `v_merge_audit_history` (§2.16) — UNION ALL cobrindo role + skill, com LEFT JOIN para `canonical_*_merge_candidates` (que aporta `opus_decision`, `opus_reasoning`, `opus_similarity`) e LEFT JOIN para `opus_arbitration_outcomes` (que aporta `cost_usd`, `input_tokens`, `output_tokens`). O endpoint separado `GET /api/admin/merge-canonicals/audit/[id]/payload` (§4.2) é usado on-demand ao expandir uma linha para entregar o `opus_reasoning` completo + comparação lado-a-lado, evitando inflar a listagem inicial. Merges manuais (`actor_id != SYSTEM_USER_ID`) não têm `opus_*` populado — os LEFT JOIN preservam a linha com esses campos NULL.

**Texto visível para o admin (caixa de baixo da tabela, linguagem de produto, ver mockup `admin-merge-canonicals-fichas-v4.html`):**
> Decisões processadas pela IA aparecem com custo em USD. Unificações que você fez manualmente entram aqui também, mas sem custo associado (não passam pela arbitragem).

#### §5.2.4 — Card 2 — Unificação manual (legado)

Sub-header:
- Título: "Unificação manual"
- Subtítulo: "Fluxo legado · use quando o automático não cobrir"
- Badge à direita: "legado" (cinza)

Conteúdo (quando ativo):

**Para Funções:**
- Campo "1 — Origem (função)": input para slug ou label
- Seta vertical
- Campo "2 — Destino (função)": input para slug ou label
- Botões: "Limpar" e "Analisar unificação" (primário)
- Hint (caixa tracejada): explica que merge manual grava `role_merge_decisions.reason = 'Merge manual via curadoria — <texto admin>'` com `actor_id = usuário logado`. `opus_arbitration_outcomes` permanece sem registro — não passa pelo CRON.

**Para Habilidades:** estrutura idêntica, com hint adicional:
> Se source e target tiverem `skill_type` diferente, RPC retorna 409 e dispara `CrossTypeConfirmModal`.

**Fluxo de "Analisar unificação"** → `POST /api/admin/merge-canonicals/manual/analyze` (§4.3) → exibe preview de impacto → botão "Executar" → `POST /api/admin/merge-canonicals/manual/execute`. Para skills cross-type sem confirmação: 409 → abre `CrossTypeConfirmModal` (já existente, ver V3 do Claude Code).

#### §5.2.5 — Card 3 — Aguardando revisão humana (ATIVO por padrão)

Sub-header:
- Título: "Aguardando revisão humana"
- Subtítulo: "IA indicou revisão humana"
- Badge à direita: contador de pendentes

Conteúdo: lista de itens (cada candidato pendente), cada item com grid 4 colunas:
- **Esquerda:** label do canônico origem + metadata (vínculos, status, flag de contexto)
- **Centro-esquerda:** ícone de swap (↔) + similaridade %
- **Centro-direita:** label do canônico destino + metadata
- **Direita:** botões `Detalhes` / `Unificar` / `Manter ambos` / `Rejeitar` (em habilidades, "Manter ambos" rotulado como "Manter")

**Para Habilidades adicionalmente:**
- Badge de `skill_type` ao lado de cada label (cor por tipo, ver TypeBadge.tsx mapeamento confirmado em V3)
- Itens cross-type têm `class="itm warn-bg"` (fundo rosado)
- Indicador `⚠ tipo divergente` ou `⚠ skills distintas` no item

**Botão "Detalhes":** abre **painel lateral** (não modal) com `opus_reasoning` completo + comparação lado a lado de vínculos, confidence e contexto recente. Endpoint: `GET /api/admin/merge-canonicals/[id]/details` (§4.4).

**Botão "Unificar":** dispara `POST /api/admin/merge-canonicals/manual/execute` com `source_id`, `target_id` da sugestão. Para skills cross-type sem `cross_type_confirmed=true`: 409 → abre `CrossTypeConfirmModal` (existente).

**Botão "Manter ambos" / "Manter":** dispara `POST /api/admin/merge-canonicals/ignore` (ou `merge-skills/ignore`) com `decision: 'ignored'` + `reason` opcional. Significa: "São canonicals distintos válidos — não sugerir de novo." Item some da lista via FLIP.

**Botão "Rejeitar ▼" — split-button com 2 opções:** o botão "Rejeitar" no Card 3 é um split-button — clicar no botão abre dropdown com 2 opções (não há ação default direta ao clicar no botão pai, para evitar erros não intencionais; admin sempre escolhe explicitamente). Layout final dos botões do Card 3:

```
[ Detalhes ]  [ Unificar ]  [ Manter ambos ]  [ Rejeitar ▼ ]
                                                     │
                                                     ▼ (ao clicar)
                                              ┌─────────────────────────────────┐
                                              │ Apenas rejeitar                 │
                                              │ Registra a decisão. Par some    │
                                              │ da lista; não vira sugestão     │
                                              │ de novo.                        │
                                              │ ─────────────────────────────── │
                                              │ Rejeitar e banir   (estilo      │
                                              │                     danger)     │
                                              │ Bane "{a_label}" — o primeiro   │
                                              │ canonical do par. O outro       │
                                              │ ({b_label}) NÃO é banido.       │
                                              └─────────────────────────────────┘
```

**Tooltip dinâmico ao hover/abrir o dropdown:** o item "Rejeitar e banir" mostra explicitamente qual label será banido (sempre `canonical_a` por convenção). Frontend resolve `a_label` e `b_label` a partir do candidato e injeta no texto. Se admin quiser banir `b_label` em vez de `a_label`, precisa usar a tela completa de Blacklist (sprint futura). MVP atual não expõe seletor A/B para preservar simplicidade do fluxo.

**Opção "Apenas rejeitar":** dispara `POST /.../ignore` com `decision: 'rejected'` + `reason` opcional. Apenas registra a decisão; mecanicamente equivalente a "Manter ambos" para o motor de sugestões (`schema.sql:4625` filtra `status IN ('ignored', 'rejected')`), mas registra intenção distinta em auditoria.

**Opção "Rejeitar e banir":** dispara `POST /api/admin/merge-canonicals/ignore` (ou `/merge-skills/ignore`) com body `{ candidate_id, decision: 'rejected', reason?, entity_type, add_to_blacklist: true }`. Backend executa transação atômica via RPC `reject_and_blacklist_canonical_pair` (§2.35): registra `decision='rejected'` em `*_merge_decisions` + INSERT em `taxonomy_blacklist` com o label do canonical_a do par. Caso o INSERT na blacklist falhe (label já banido), a transação inteira é abortada e o admin recebe erro 409 com mensagem específica.

**Atualizações dos hooks frontend (§3.x):**
- `app/admin/merge-canonicals/_components/use-merge-canonicals.ts:116` — antes hardcoded `decision: 'ignored'`. Atualizar para receber o valor do botão clicado: `decision: selectedDecision` onde `selectedDecision: 'ignored' | 'rejected'`. Body inclui `add_to_blacklist` quando split-button "Rejeitar e banir" foi escolhido.
- `components/admin/merge-skills/use-merge-skills.ts:167` — idem para skills.

**Mockup de referência:** `admin-merge-canonicals-fichas-v4.html` ilustra o split-button com dropdown, posicionamento (open-down, z-index alto), e estilo (`Apenas rejeitar` neutro + `Rejeitar e banir` rosa-danger).

**Tabela `taxonomy_blacklist` (escopo desta sprint):** o `INSERT` em `taxonomy_blacklist` exige que a tabela exista. Como a tela própria de Blacklist está prevista para sprint futura (memórias do projeto: "tela nativa, sempre comprometida, nunca foi JSON; menu 'Blacklist' já no dropdown admin"), a tabela é criada nesta sprint via §2.31 (ver abaixo). Mínimo: `id uuid PK`, `label text NOT NULL`, `entity_type text NOT NULL CHECK IN ('role','skill')`, `reason text`, `added_by uuid`, `added_at timestamptz`. UNIQUE em `(lower(trim(label)), entity_type)` para evitar duplicatas. RLS standard.

**Note inferior do Card 3** (texto visível ao admin, linguagem de produto — ver mockup v4):
> "Detalhes abre painel lateral com o raciocínio completo da IA + comparação lado a lado de vagas, confiança e contexto recente. Manter ambos = 'são funções diferentes mesmo'. Rejeitar tem duas variações: 'Apenas rejeitar' (registra a decisão) ou 'Rejeitar e banir' (rejeita + envia o termo para a Blacklist — usar quando o label é claramente lixo: 'asdf', 'teste', 'função xpto'). A Blacklist completa fica em tela própria no menu admin."

#### §5.2.6 — Tradução de veredictos (mapa client-side)

Mapeamento backend ↔ UI:

```typescript
// lib/admin/merge-decision-labels.ts
export const DECISION_LABELS: Record<string, { label: string; tone: 'success' | 'neutral' | 'warning' | 'danger' }> = {
  // *_merge_decisions.status (CHECK: pending, merged, ignored, rejected)
  merged:   { label: 'UNIFICAR',     tone: 'success' },
  ignored:  { label: 'MANTER AMBOS', tone: 'neutral' },
  rejected: { label: 'REJEITAR',     tone: 'danger'  },
  pending:  { label: 'PENDENTE',     tone: 'warning' },

  // opus_arbitration_outcomes.decision e canonical_*_merge_candidates.opus_decision
  MERGE:       { label: 'UNIFICAR',     tone: 'success' },
  KEEP_BOTH:   { label: 'MANTER AMBOS', tone: 'neutral' },
  NEEDS_HUMAN: { label: 'REVISAR',      tone: 'warning' },
};
```

**Semântica visível para o admin:**
- `MANTER AMBOS` (`ignored`) — "São canonicals distintos válidos; não sugerir de novo."
- `REJEITAR` (`rejected`) — "Sugestão foi erro de raiz; não fazia sentido nem sugerir."

Mecanicamente idênticos para o motor (`schema.sql:4625` filtra `status IN ('ignored', 'rejected')` igualmente). Diferem apenas na intenção registrada em auditoria, útil para tunar `cosine_threshold`.

#### §5.2.7 — Nomenclatura "IA" vs "Opus"

Em **todas** as strings exibidas na UI:
- "Opus" → "IA" (ex: badge `IA` no Card 1, "IA indicou revisão humana" no subtítulo do Card 3).
- Mantém **"Opus"** apenas em:
  - Logs e eventos (`event_name`, `metadata.origin`)
  - Nomes de colunas/funções SQL (`opus_arbitration_outcomes`, `opus_reasoning`)
  - Documentação interna do código (comentários, JSDoc)

---

### §5.3 — `DomainRoleCascade` (cascata área→função)

**Atende:** Frente A1.3.

Componente novo `components/admin/DomainRoleCascade.tsx`. Dois dropdowns encadeados:
1. **Área (Domínio):** lista de `canonical_role_domains` ativos.
2. **Função:** canonicals roles filtrados via `canonical_role_domain_links` pelo domínio selecionado.

Quando o admin troca o domínio, o dropdown de função recarrega. Inclui opção "Outra função" no dropdown de função para entrada livre (caso o canonical ainda não esteja linkado).

**Prop `mode`:** o componente aceita `mode: 'admin' | 'public'` que controla quais endpoints são chamados:

```tsx
type DomainRoleCascadeProps = {
  mode: 'admin' | 'public';
  selectedDomainId?: string;
  selectedRoleId?: string;
  onChange: (selection: { domainId?: string; roleId?: string; otherRole?: string }) => void;
};

// mode='admin' → GET /api/admin/role-domains + GET /api/admin/canonical-roles-by-domain (§4.6)
// mode='public' → GET /api/taxonomy/domains + GET /api/taxonomy/roles-by-domain (§4.8)
```

**Diferenças entre os endpoints:**

| Aspecto | `mode='admin'` (§4.6) | `mode='public'` (§4.8) |
|---|---|---|
| Auth | required (admin role) | público |
| Cache HTTP | sem cache | edge cache 300s |
| Campos retornados | `id, slug, name, is_active, display_order` (domains) / `id, label, status, vacancy_count` (roles) | `id, name` (domains) / `id, label` (roles) — apenas o estritamente necessário |
| Inclui canonicals em `status='pending'`? | sim (com indicador visual) | não — apenas `active` |

**Consumidores:**
- `mode='admin'`: `ResourceEditModal` quando o admin associa manualmente um canonical a um domínio; tela de criação manual de vaga no admin (futuro).
- `mode='public'`: formulário de criação de perfil profissional, fluxo "outra função" (Frente A1.3, parte pública).

A decisão de endpoint vive no próprio componente — fora dele, o consumidor só precisa passar `mode`. Isso evita duplicação de lógica entre o admin e o frontend público.

---

### §5.4 — AdminNav reordenado

Reordenação alfabética PT-BR do dropdown admin (conforme Onsly cravou em memória):

1. Calibração de Limiares (`/admin/pipeline-config`) — **nova entrada**
2. Campanhas de Créditos (`/admin/campaigns`)
3. Carga e Curadoria (`/admin/ingestor`)
4. Painel de Controle (`/admin/dashboard`)
5. Preços (`/admin/pricing`)
6. Unificação de Cadastros (`/admin/merge-canonicals`)

**Nota:** o item "Auditoria IA" não consta no menu — sem tela correspondente (`/admin/opus-review` não existe). Auditoria de decisões IA fica integrada como Card 1 de `/admin/merge-canonicals` (§5.2.3).

---

## §6 AdminModal compartilhado

### §6.1 — `components/ui/admin-modal.tsx`

API expandida para atender às demandas dos 6 modais: `ResourceEditModal` precisa de `role='alertdialog'` + `ariaDescribedBy` + size literal `'420px'`; `PipelineModal` precisa de `closeOnEscape` e `closeDisabled` (✕ desabilitado durante curation); `ComparisonModal` + `DuplicatesModal` precisam de pilha global para resolver bug de Escape duplo.

```tsx
import React, { useId, useRef, useEffect } from 'react';

type AdminModalProps = {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
  footer?: React.ReactNode;
  /** 'sm' | 'md' | 'lg' ou valor CSS literal ex.: '420px' */
  size?: 'sm' | 'md' | 'lg' | string;
  /** dialog (default) ou alertdialog (para confirmações destrutivas/críticas) */
  role?: 'dialog' | 'alertdialog';
  /** id de elemento descritivo, usado em alertdialog */
  ariaDescribedBy?: string;
  /** Default true. Quando false, ignora Escape (ex.: PipelineModal durante curation) */
  closeOnEscape?: boolean;
  /** Default false. Quando true, ✕ fica disabled + opacity-20 + cursor-not-allowed */
  closeDisabled?: boolean;
  /** Default true. Quando false, click no backdrop não fecha (ex.: PipelineModal) */
  closeOnBackdrop?: boolean;
};

// Pilha global de modais abertos — resolve bug de Escape duplo entre
// ComparisonModal (z-95) e DuplicatesModal (z-90)
const modalStack: Array<() => void> = [];

function useReducedMotion(): boolean {
  const [prefers, setPrefers] = React.useState(false);
  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    setPrefers(mq.matches);
    const handler = (e: MediaQueryListEvent) => setPrefers(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);
  return prefers;
}

function useFocusTrap<T extends HTMLElement>(isOpen: boolean) {
  const ref = useRef<T>(null);
  useEffect(() => {
    if (!isOpen || !ref.current) return;
    const focusableSelector =
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])';
    const focusables = ref.current.querySelectorAll<HTMLElement>(focusableSelector);
    const first = focusables[0];
    const last = focusables[focusables.length - 1];
    const handler = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last?.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first?.focus();
      }
    };
    ref.current.addEventListener('keydown', handler);
    first?.focus();
    // N1: capturar ref.current localmente — no momento do cleanup,
    // ref.current pode já ter sido nullified pelo React. Capturar evita removeEventListener silenciosamente no-op.
    const el = ref.current;
    return () => el?.removeEventListener('keydown', handler);
  }, [isOpen]);
  return ref;
}

// Guarda overflow original do body antes do primeiro modal abrir
let previousBodyOverflow: string | null = null;

export function AdminModal({
  isOpen,
  onClose,
  title,
  children,
  footer,
  size = 'md',
  role = 'dialog',
  ariaDescribedBy,
  closeOnEscape = true,
  closeDisabled = false,
  closeOnBackdrop = true,
}: AdminModalProps) {
  const titleId = useId();
  const reducedMotion = useReducedMotion();
  const modalRef = useFocusTrap<HTMLDivElement>(isOpen);

  // Pilha de Escape: só o último modal aberto responde
  useEffect(() => {
    if (!isOpen) return;

    modalStack.push(onClose);

    const handler = (e: KeyboardEvent) => {
      if (e.key !== 'Escape' || !closeOnEscape || closeDisabled) return;
      const topClose = modalStack[modalStack.length - 1];
      if (topClose === onClose) {
        e.stopPropagation();
        onClose();
      }
    };
    window.addEventListener('keydown', handler);

    return () => {
      window.removeEventListener('keydown', handler);
      const idx = modalStack.indexOf(onClose);
      if (idx !== -1) modalStack.splice(idx, 1);
    };
  }, [isOpen, onClose, closeOnEscape, closeDisabled]);

  // Guarda/restaura overflow do body corretamente (corrige bug PipelineModal)
  // N2: este useEffect DEPENDE da ordem do useEffect anterior (Escape stack). React garante que effects rodam em ordem de declaração — setup do Escape stack roda primeiro, populando modalStack ANTES deste setup ler modalStack.length. Cleanup é reverso: este cleanup roda PRIMEIRO (modalStack ainda contém o handler atual), depois o cleanup do Escape stack remove o handler.
  // Capturar modalStack.length em variável local no início de cada effect torna a dependência explícita.
  useEffect(() => {
    if (!isOpen) return;
    const stackSizeAtSetup = modalStack.length; // capturado após Escape setup adicionar onClose
    if (stackSizeAtSetup === 1) {
      previousBodyOverflow = document.body.style.overflow;
      document.body.style.overflow = 'hidden';
    }
    return () => {
      // Cleanup roda ANTES do cleanup do Escape useEffect remover onClose do stack
      const stackSizeAtCleanup = modalStack.length;
      if (stackSizeAtCleanup === 1 && previousBodyOverflow !== null) {
        document.body.style.overflow = previousBodyOverflow;
        previousBodyOverflow = null;
      }
    };
  }, [isOpen]);

  if (!isOpen) return null;

  const sizeClass =
    size === 'sm' ? 'max-w-md' :
    size === 'lg' ? 'max-w-3xl' :
    size === 'md' ? 'max-w-xl' : '';
  const sizeStyle = sizeClass === '' ? { maxWidth: size } : undefined;

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
      onClick={closeOnBackdrop && !closeDisabled ? onClose : undefined}
      role={role}
      aria-modal="true"
      aria-labelledby={titleId}
      aria-describedby={ariaDescribedBy}
    >
      <div
        ref={modalRef}
        className={`bg-white rounded-lg shadow-xl w-full ${sizeClass} ${reducedMotion ? '' : 'transition-transform'}`}
        style={sizeStyle}
        onClick={(e) => e.stopPropagation()}
      >
        <header className="px-6 py-4 border-b flex items-center justify-between">
          <h2 id={titleId} className="text-lg font-semibold">{title}</h2>
          <button
            type="button"
            onClick={closeDisabled ? undefined : onClose}
            disabled={closeDisabled}
            className={`text-gray-500 hover:text-gray-700 ${closeDisabled ? 'opacity-20 cursor-not-allowed' : ''}`}
            aria-label="Fechar"
          >
            ✕
          </button>
        </header>
        <div className="px-6 py-4 max-h-[70vh] overflow-y-auto">{children}</div>
        {footer && <footer className="px-6 py-4 border-t bg-gray-50">{footer}</footer>}
      </div>
    </div>
  );
}
```

### §6.2 — Plano de migração dos modais (6 fichas)

Cada modal foi auditado individualmente. As migrações **não** mudam o comportamento existente — apenas trocam a casca por `AdminModal` e harmonizam acessibilidade onde havia regressão.

#### 6.2.1 — `ResourceEditModal` (app/admin/pricing/_modals/ResourceEditModal.tsx)

**Multi-step real:** estado interno `view: 'edit' | 'alert' | 'review'`. Renderiza 3 instâncias do shell concorrentes.

**Plano:** manter 3 `AdminModal` concorrentes (cada um com `isOpen={isOpen && view === X}`). Substituir `ModalShell` por `AdminModal` em cada uma:

- **view "edit"** → `<AdminModal isOpen={isOpen && view==='edit'} onClose={onClose} title="..." size="md">`
- **view "alert"** → `<AdminModal isOpen={isOpen && view==='alert'} onClose={() => setView('edit')} title="..." role="alertdialog" ariaDescribedBy="alert-desc" size="420px">`. Note que `onClose` aqui não fecha o pai — volta ao `view='edit'` (preserva comportamento atual onde ✕ não fecha o review/alert).
- **view "review"** → `<AdminModal isOpen={isOpen && view==='review'} onClose={() => setView('edit')} title="Revisar e publicar" size="md">`. Gate textual "PUBLICAR" permanece dentro do footer/children; AdminModal não interfere.

Preservar `useEffect` que reseta estado em `resource?.id` e guard `isMounted.current`. `ModalShell` é deprecado após esta migração.

#### 6.2.2 — `IgnoreModal` em merge-canonicals (app/admin/merge-canonicals/_components/IgnoreModal.tsx)

**Atual:** pai renderiza condicionalmente, sem `isOpen`. Tem aria correto (`role="dialog"`, `aria-modal`, `aria-label`). `maxLength=500` no textarea, `autoFocus`.

**Plano:** refatorar interface para:
```tsx
type Props = {
  isOpen: boolean;
  ignoreModal: IgnoreModalState | null;
  ignoreReason: string;
  ignoreSending: boolean;
  onReasonChange: (reason: string) => void;
  onConfirm: () => void;
  onCancel: () => void;
};
```
e usar `<AdminModal isOpen={isOpen} onClose={onCancel} title="Ignorar sugestão de unificação" size="md">`. Body contém o textarea com `autoFocus` + `maxLength={500}`. Footer com botões Cancelar/Confirmar (Confirmar usa `loading={ignoreSending}`).

#### 6.2.3 — `IgnoreModal` em merge-skills (components/admin/merge-skills/IgnoreModal.tsx)

**Atual:** versão paralela ao 6.2.2 com **regressão de acessibilidade** (sem `role="dialog"`, `aria-modal`, `aria-label`), **sem** `maxLength`, e props com nomes divergentes (`modal` vs `ignoreModal`, `reason` vs `ignoreReason`, `sending` vs `ignoreSending`).

**Plano:** migrar para `AdminModal` com a **mesma API harmonizada** do 6.2.2 (props `ignoreModal/ignoreReason/ignoreSending/onReasonChange/onConfirm/onCancel`). Adicionar `maxLength=500` e `autoFocus`. A migração corrige a regressão a custo zero.

**Outputs do PR:** ambos os IgnoreModals deixam de duplicar código e podem ser consolidados em `components/ui/IgnoreReasonModal.tsx` (componente único compartilhado pelas duas telas). Decisão de consolidação pode ficar para sprint futura — esta apenas harmoniza.

#### 6.2.4 — `ComparisonModal` (components/admin/ingestor/ComparisonModal.tsx)

**Atual:** `z-[95]`, sem `role`/`aria`. Handler `Escape` via `window.addEventListener` (causa bug de fecho duplo com DuplicatesModal). `document.body.style.overflow = 'hidden'` com restauração para valor anterior (correto). Botão "Copiar JSON" → `navigator.clipboard` + toast.

**Plano:** trocar div custom por `AdminModal` com `size="lg"`. Pilha global de `AdminModal` (§6.1) **resolve por padrão** o bug de Escape duplo — só o modal no topo da pilha responde. Botão "Copiar JSON" e toast permanecem no body. Animação `animate-in fade-in duration-300` migra para `transition-opacity` padrão do AdminModal (já respeita `prefers-reduced-motion`).

#### 6.2.5 — `DuplicatesModal` (components/admin/ingestor/DuplicatesModal.tsx)

**Atual:** `z-[90]`, retorna null se sem report. Aciona `ComparisonModal` via `onSelectComparison`. Mesmo padrão de Escape/overflow.

**Plano:** trocar por `AdminModal` com `size="lg"`. Estado do `ComparisonModal` permanece gerenciado pelo pai (não pelo DuplicatesModal). Pilha global garante z-ordem correta sem hardcode de `z-[90]`/`z-[95]`.

#### 6.2.6 — `PipelineModal` (components/admin/ingestor/PipelineModal.tsx)

**Atual:** 17 props, rodapé condicional baseado em `pipelineProgress === 100 && !isCurationRunning`. Bloqueia Escape e desabilita ✕ durante `isCurationRunning`. `document.body.style.overflow` restaurado para `'auto'` hardcoded (**bug**: deveria restaurar valor anterior). `animate-in zoom-in-95`.

**Plano:** trocar por:
```tsx
<AdminModal
  isOpen={isPipelineOpen}
  onClose={onClose}
  title="Pipeline de Curadoria"
  size="lg"
  closeOnEscape={!isCurationRunning}
  closeDisabled={isCurationRunning}
  closeOnBackdrop={false}
  footer={<PipelineFooter ... />}
>
  <PipelineBody ... />
</AdminModal>
```
- `closeOnEscape={!isCurationRunning}` preserva o bloqueio intencional do Escape.
- `closeDisabled={isCurationRunning}` preserva ✕ disabled com opacity reduzida.
- `closeOnBackdrop={false}` preserva bloqueio de fechamento via backdrop.
- O bug de overflow não-restaurado é **corrigido automaticamente** pela pilha global.
- Sub-componente `StreamPanel` com `memo()` permanece dentro de `<PipelineBody>`, sem alteração.

#### Deprecação de `ModalShell`

Após as 6 migrações, remover `app/admin/pricing/_modals/ModalShell.tsx`. Verificar via `rg "from.*ModalShell"` que não há outros callers (auditoria 6.2 mostra que `ModalShell` é usado apenas em `ResourceEditModal.tsx` e `PackageEditModal.tsx`; o segundo não está no escopo desta sprint e usa o shell isoladamente — preservar até sprint futura migrá-lo).

---

## §7 Validação e critérios

### §7.1 — Evidence block (queries de validação)

Rodar após cada migration de SQL e antes de PR-merge:

```sql
-- E1: confirmar rename de funções
SELECT proname FROM pg_proc
WHERE proname IN (
  'fn_promote_role_on_threshold',
  'fn_retire_role_on_zero_vacancy',
  'catchup_pending_role_promotions',
  'auto_assign_family_to_role',
  'trigger_auto_assign_family_role' 
);
-- Esperado: 5 linhas

SELECT proname FROM pg_proc
WHERE proname IN (
  'fn_promote_canonical_on_threshold',
  'fn_retire_canonical_on_zero_vacancy',
  'catchup_pending_promotions',
  'auto_assign_family_to_canonical',
  'trigger_auto_assign_family'
);
-- Esperado: 0 linhas

-- E2: confirmar rename de triggers
SELECT tgname FROM pg_trigger WHERE tgname IN (
  'trg_promote_role_on_threshold',
  'z_trg_retire_role_on_zero_vacancy',
  'auto_assign_family_on_role_active'
);
-- Esperado: 3 linhas

-- E2.1: trigger consolidado — 1 row, NÃO 2
SELECT tgname, pg_get_triggerdef(oid) FROM pg_trigger
WHERE tgname = 'auto_assign_family_on_role_active'
  AND tgrelid = 'job_canonical_roles'::regclass;
-- Esperado: 1 linha; definição contém "AFTER INSERT OR UPDATE OF status"

-- E3: triggers de flag agora escutam status
SELECT tgname, pg_get_triggerdef(oid) FROM pg_trigger
WHERE tgname IN ('trg_flag_needs_opus_review_jcr', 'trg_flag_needs_opus_review_jcs');
-- Esperado: a definição de cada um deve incluir "UPDATE OF confidence_median, status"

-- E4: Opção C aplicada em JCS (catchup E trigger)
SELECT pg_get_functiondef(oid) FROM pg_proc
WHERE proname IN ('catchup_pending_skill_promotions', 'fn_promote_skill_on_threshold');
-- Esperado: nenhum dos 2 corpos deve conter "confidence_median >= v_min_confidence"
-- Esperado: nenhum dos 2 deve declarar variável "v_min_confidence"

-- E5: índices funcionais que sustentam lookups §2.4 e §2.5
SELECT indexname, indexdef FROM pg_indexes
WHERE indexname IN (
  'uq_jcs_label_normalized',                 -- vivo desde paridade-skills v11
  'uq_jcr_canonical_label_normalized'        -- criado em §2.5 desta sprint
);
-- Esperado: 2 linhas, ambos com a expressão funcional normalize_*_label(...) na definição.
-- IMPORTANTE: NÃO devem existir os índices em lower(label) / lower(canonical_label)
-- que apareciam em versões anteriores da spec (B-3 da v3 removeu dead code).
SELECT COUNT(*) FROM pg_indexes
WHERE indexname IN ('idx_jcs_lookup_label_lower', 'idx_jcr_lookup_canonical_label_lower');
-- Esperado: 0 — esses índices não devem ter sido criados nesta sprint.

-- E6: admin_panel_functions populada (registry de funções admin)
SELECT panel_id, COUNT(*) FROM admin_panel_functions GROUP BY panel_id ORDER BY panel_id;
-- Esperado: 5 panel_ids com contagem específica:
--   campaigns         | 6
--   ingestor          | 4
--   merge_canonicals  | 7
--   pipeline_config   | 4
--   pricing           | 8
-- Total: 29 funções. NÃO deve existir panel_id 'opus_review' (removido).

-- E6.1: criticality coverage no registry
SELECT panel_id, criticality_level, COUNT(*)
FROM admin_panel_functions
GROUP BY panel_id, criticality_level
ORDER BY panel_id, criticality_level;
-- Esperado: cada panel_id tem pelo menos 1 'high' (ações irreversíveis/financeiras).

-- E7: pipeline_config criticality_level seed
SELECT criticality_level, COUNT(*) FROM pipeline_config GROUP BY criticality_level;
-- Esperado: distribuição com pelo menos 6 'high' (promotion threshold keys)

-- E8: view v_merge_audit_history acessível
SELECT entity_type, COUNT(*) FROM v_merge_audit_history GROUP BY entity_type;
-- Esperado: roda sem erro (linhas podem ser 0 em base limpa)

-- limpeza aplicada em o6_recent_errors
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname = 'o6_recent_errors';
-- Esperado: corpo contém 'role_creation_blocked_low_confidence' e
-- 'role_creation_blocked_missing_confidence'; NÃO contém
-- 'canonical_creation_blocked_low_confidence',
-- 'canonical_creation_blocked_missing_confidence' nem
-- 'canonical_creation_blocked_resolved_id_mismatch'

-- E10: cadeia auto_assign_family conectada após rename
-- Verifica que o trigger wrapper chama a função regular renomeada
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname = 'trigger_auto_assign_family_role';
-- Esperado: corpo contém "PERFORM auto_assign_family_to_role(NEW.id)"
-- Esperado: corpo NÃO contém "auto_assign_family_to_canonical"

-- E11: catchup invoca auto_assign_family após promoção
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname = 'catchup_pending_role_promotions';
-- Esperado: corpo contém "PERFORM auto_assign_family_to_role(r.id, TRUE)"

-- E12: pipeline_config.description column criada e populada (§2.19a)
-- A coluna já existe NOT NULL em prod (§2.19a documenta) — ADD COLUMN IF NOT EXISTS é no-op,
-- os UPDATEs reescrevem os valores definitivos da sprint.
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'pipeline_config' AND column_name = 'description';
-- Esperado: 1 linha — description, text, NO

SELECT COUNT(*) FROM pipeline_config WHERE description IS NULL;
-- Esperado: 0 (seed completo é gate para §5.1 — ver §X pontos pendentes)

-- E13: RLS FORCE em pipeline_config_history (§2.20a)
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'pipeline_config_history';
-- Esperado: relrowsecurity = true AND relforcerowsecurity = true

-- E13.1: validar bloqueio de INSERT direto
-- (Executar como authenticated, não service_role)
INSERT INTO pipeline_config_history (key, previous_value, new_value, changed_by, reason, changed_at)
VALUES ('test.key', '0', '1', auth.uid(), 'test direct insert', NOW());
-- Esperado: erro de RLS (new row violates row-level security policy)

-- E14: RPC set_pipeline_config_value bypassa RLS via SECURITY DEFINER
SELECT prosecdef FROM pg_proc WHERE proname = 'set_pipeline_config_value';
-- Esperado: prosecdef = true

-- E15: admin_panel_functions UNIQUE constraint
SELECT pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'admin_panel_functions'::regclass
  AND contype = 'u';
-- Esperado: 1 linha — UNIQUE (panel_id, function_id)

-- E16: índice idx_admin_panel_functions_enabled (parcial)
SELECT indexdef FROM pg_indexes
WHERE indexname = 'idx_admin_panel_functions_enabled';
-- Esperado: contém "WHERE (is_enabled = true)"

-- E17: 4 RPCs B7 criados (§2.26-§2.29)
SELECT proname FROM pg_proc
WHERE proname IN (
  'admin_panel_custo_opus_por_canonical',
  'admin_panel_merge_auto_vs_manual',
  'admin_panel_latest_posted_distribution',
  'admin_panel_suggestion_rejected_by_skill'
);
-- Esperado: 4 linhas

-- E18: smoke do Painel B7 #1 (não deve dar erro mesmo em base vazia)
SELECT * FROM admin_panel_custo_opus_por_canonical(30) LIMIT 1;
-- Esperado: roda sem erro; pode retornar 0 linhas em base limpa

-- E19: smoke do Painel B7 #2
SELECT admin_panel_merge_auto_vs_manual(30);
-- Esperado: retorna jsonb (mesmo em base vazia, retorna {auto:0,manual:0,total:0,...})

-- E20: validação de v_opus_effectiveness (§2.30)
SELECT table_name FROM information_schema.views WHERE table_name = 'v_opus_effectiveness';
-- Resultado dita se Painel B7 #4 está ativo. Se 0 linhas, §2.30 deveria ter abortado.

-- E21: endpoints públicos de cascata estão acessíveis sem auth
-- (smoke manual via curl):
-- curl https://app.calibracv.com/api/taxonomy/domains          → 200 + { domains: [...] }
-- curl https://app.calibracv.com/api/taxonomy/roles-by-domain?domain_id=<uuid>  → 200 + { roles: [...] }

-- E22: smoke do Painel 11 (Confidence calculável)
SELECT admin_panel_confidence_calculable(30);
-- Esperado: jsonb { snapshot: { role: {calculable, not_calculable}, skill: {...} }, thresholds: {...} }

-- E23: smoke do Painel 12 (Fila de re-revisão Opus)
SELECT admin_panel_opus_review_queue(7);
-- Esperado: jsonb { snapshot: { role: {in_cooldown,...}, skill: {...} }, window_days: 7, cooldowns: {...} }

-- E24: smoke do Painel 13 (Fila de re-revisão de merge candidates)
SELECT admin_panel_merge_review_queue(7);
-- Esperado: jsonb similar ao E23 + attempt_limit_reached por entidade

-- E25: existência da RPC reject_and_blacklist_canonical_pair (não executar — efeito colateral)
SELECT proname, prosecdef AS security_definer
FROM pg_proc
WHERE proname = 'reject_and_blacklist_canonical_pair';
-- Esperado: 1 linha; security_definer = true
```

---

### §7.1.1 — Smoke tests granulares pós-aplicação

Procedimentos manuais executados após deploy, em ambiente pré-produção, para validar fluxos end-to-end que queries SQL puras não cobrem.

#### S1 — Confirmar todas as colunas, funções e views criadas

```sql
-- Colunas novas
SELECT table_name, column_name
FROM information_schema.columns
WHERE (table_name = 'resumes' AND column_name = 'skills_with_confidence')
   OR (table_name = 'resume_roles' AND column_name IN ('canonical_role_id', 'passed_hard_gate'))
   OR (table_name = 'job_canonical_roles' AND column_name = 'latest_posted_at')
   OR (table_name = 'pipeline_config' AND column_name = 'description');
-- Esperado: 5 linhas.

-- Funções novas
SELECT proname FROM pg_proc
WHERE proname IN (
  'lookup_canonical_role_by_normalized_label',
  'lookup_canonical_role_by_normalized_alias',
  'lookup_canonical_skill_by_normalized_label',
  'reconcile_canonical_role',
  'reconcile_canonical_skill',
  'fn_jcr_update_latest_posted_at',
  'admin_panel_custo_opus_por_canonical',
  'admin_panel_merge_auto_vs_manual',
  'admin_panel_latest_posted_distribution',
  'admin_panel_suggestion_rejected_by_skill',
  'admin_panel_confidence_calculable',
  'admin_panel_opus_review_queue',
  'admin_panel_merge_review_queue',
  'reject_and_blacklist_canonical_pair'
);
-- Esperado: 14 linhas.

-- View nova
SELECT table_name FROM information_schema.views WHERE table_name = 'v_merge_audit_history';
-- Esperado: 1 linha.

-- Tabela nova (§2.8b — simetria com resume_roles)
SELECT table_name FROM information_schema.tables
WHERE table_name = 'resume_skills' AND table_schema = 'public';
-- Esperado: 1 linha.

-- CHECK de confidence em resume_skills (escala 0-1, não hard gate fixo).
-- Versão v2.8 removeu o CHECK hardcoded em 0.70 (H-2 — hard gate é filtro TS, não DB invariant).
-- O único CHECK relevante hoje é o de range [0,1] criado inline em §2.8b.
SELECT conname, pg_get_constraintdef(oid) AS def
FROM pg_constraint
WHERE conrelid = 'resume_skills'::regclass
  AND contype = 'c'
  AND pg_get_constraintdef(oid) LIKE '%confidence%';
-- Esperado: 1 linha — CHECK ((confidence >= (0)::numeric) AND (confidence <= (1)::numeric)).
-- Garantia adicional: NÃO deve existir nenhum CHECK com 'confidence >= 0.70' (H-2 removeu).

-- Simetria: mesma validação em resume_roles
SELECT conname, pg_get_constraintdef(oid) AS def
FROM pg_constraint
WHERE conrelid = 'resume_roles'::regclass
  AND contype = 'c'
  AND pg_get_constraintdef(oid) LIKE '%confidence%';
-- Esperado: 1 linha — chk_resume_roles_confidence com range [0,1] (pós-§2.7).
-- NÃO deve existir CHECK hardcoded em 0.70 nem CHECK em 0-100 (era smallint antes da migration).

-- Índices em resume_skills (UNIQUE constraint regular + 2 parciais)
SELECT indexname FROM pg_indexes
WHERE tablename = 'resume_skills'
  AND indexname IN (
    'uq_resume_skills_resume_skill_normalized',
    'idx_resume_skills_canonical_skill_id',
    'idx_resume_skills_passed_hard_gate'
  );
-- Esperado: 3 linhas.

-- CHECKs atualizados
SELECT conname, pg_get_constraintdef(oid) AS def
FROM pg_constraint
WHERE conname IN ('job_canonical_skills_source_check', 'job_canonical_roles_source_check')
  AND pg_get_constraintdef(oid) LIKE '%resume_extraction%';
-- Esperado: 2 linhas.
```

#### S2 — Smoke test do fluxo CV → análise → STAR (end-to-end)

```sql
-- 1. Subir CV de teste via /api/resume/upload
-- 2. Após upload, verificar população das três camadas:
SELECT
  id,
  jsonb_array_length(skills_with_confidence) AS skills_audit_count,
  array_length(skills_extracted, 1) AS skills_operacional_count,
  jsonb_array_length(skills_with_confidence) - array_length(skills_extracted, 1) AS skills_below_gate
FROM resumes
WHERE id = '<resume_id_de_teste>';
-- Esperado: skills_audit_count >= skills_operacional_count (gate é monotônico).

-- 3. Verificar roles ingeridas
SELECT
  resume_id,
  count(*) AS total_roles,
  count(*) FILTER (WHERE passed_hard_gate) AS roles_above_gate,
  count(*) FILTER (WHERE canonical_role_id IS NOT NULL) AS roles_with_canonical,
  count(*) FILTER (WHERE source = 'resume_extraction') AS roles_with_source_set
FROM resume_roles
WHERE resume_id = '<resume_id_de_teste>'
GROUP BY resume_id;
-- Esperado:
--   roles_above_gate <= total_roles (alguns descartados se confidence baixa);
--   roles_with_canonical ≈ total_roles (resolução bem sucedida);
--   roles_with_source_set = total_roles (CHECK constraint job_canonical_roles_source_check §2.2 + §3.3 cravam source='resume_extraction').

-- 3b. Verificar skills ingeridas (simétrico a resume_roles — §2.8b)
SELECT
  resume_id,
  count(*) AS total_skills,
  count(*) FILTER (WHERE passed_hard_gate) AS skills_above_gate,
  count(*) FILTER (WHERE canonical_skill_id IS NOT NULL) AS skills_with_canonical,
  count(*) FILTER (WHERE source = 'resume_extraction') AS skills_with_source_set
FROM resume_skills
WHERE resume_id = '<resume_id_de_teste>'
GROUP BY resume_id;
-- Esperado: paralelo a S2 step 3 (skills_above_gate <= total_skills;
--   skills_with_canonical ≈ total_skills; skills_with_source_set = total_skills).

-- 4. Após análise, verificar resume_skill_enrichments populada
SELECT
  analysis_id,
  count(*) AS enrichment_rows,
  count(*) FILTER (WHERE canonical_skill_id IS NOT NULL) AS rows_with_canonical
FROM resume_skill_enrichments
WHERE analysis_id = '<analysis_id_de_teste>'
GROUP BY analysis_id;
-- Esperado: ambos > 0 (linhas-base criadas pelo wiring em compare-fn.ts).

-- 5. Usuário clica "Validar" em SkillOptimizeRow → pipeline STAR existente preenche campos
SELECT
  count(*) AS validated_rows,
  count(*) FILTER (WHERE star_situation IS NOT NULL) AS rows_with_situation,
  count(*) FILTER (WHERE cv_text_variant_1 IS NOT NULL) AS rows_with_variants
FROM resume_skill_enrichments
WHERE analysis_id = '<analysis_id_de_teste>';
-- Esperado: rows_with_variants > 0; rows_with_situation > 0 (STAR aplicado).
```

#### S3 — Confirmar uso da função compartilhada `evaluateHardGate`

```bash
# Antes da sprint: 4 callers com leitura inline de getConfigValue + comparison
git grep -nE "skill\.hard_gate\.min_confidence|role\.hard_gate\.min_confidence" -- '*.ts'
# Pós-sprint esperado:
# - Apenas 1 ocorrência: lib/pipeline/_shared/evaluate-hard-gate.ts (definição)
# - Chamadas via evaluateHardGate(...) ou filterByHardGate(...) nos 4 callers migrados
```

#### S4 — Confirmar adoção de `resolve_active_canonical_by_slug` no pipeline de funções

```bash
git grep -nE "resolve_active_canonical_by_slug" -- '*.ts' '*.tsx'
# Pós-sprint esperado: aparece em lib/pipeline/canonical-roles.ts
# (helper resolveCanonicalRoleBySlug) e em lib/pipeline/extract-roles-from-resume.ts.
```

#### S5 — Confirmar deprecação preparada de `ModalShell`

```bash
git grep -nE "ModalShell" -- 'app/admin/**' 'components/admin/**'
# Pós-sprint esperado: 2 ocorrências —
#   app/admin/pricing/_modals/ModalShell.tsx (arquivo do componente, preservado)
#   app/admin/pricing/_modals/PackageEditModal.tsx (consumer ainda fora do escopo desta sprint)
# Nenhuma OUTRA referência fora desses 2 arquivos.
```

#### S6 — Smoke da tela Calibração de Limiares

Procedimento manual via navegador em pré-produção:

1. Acessar `/admin/pipeline-config`.
2. Esperado: 3 KPIs no topo (Chaves do pipeline / Mudanças 30d / Última publicação); tabela renderizada com 24 linhas; coluna "Acompanhamento" mostra tags `Painel X` clicáveis com tooltip rico.
3. Clicar "Editar" em chave de criticidade `baixa` ou `média` — modal abre, edição habilitada, salva direto sem gate "PUBLICAR".
4. Salvar e validar:
   ```sql
   SELECT * FROM pipeline_config_history ORDER BY changed_at DESC LIMIT 1;
   ```
5. Clicar "Editar" em chave de criticidade `alta` — modal abre, box vermelho "Digite PUBLICAR" presente. Tentar salvar sem digitar → CTA desabilitado. Digitar "PUBLICAR" e salvar → publica.
6. Clicar relógio "há Xd" em qualquer linha — modal de histórico abre listando últimas alterações em ordem reversa, com botão de rollback ↺ em cada linha (exceto seed inicial).
7. Clicar rollback ↺ — modal de edição abre com `new_value = previous_value` da entrada; gate "PUBLICAR" aplica se criticidade alta.

#### S7 — Smoke do refactor `/admin/merge-canonicals`

1. Acessar a tela e confirmar:
   - 2 abas no topo (Funções / Habilidades) com contador de pendentes
   - 3 cards em pilha-degrau na ordem: Auditoria (topo), Manual (meio), Aguardando revisão humana (base, ativo)
   - Card ativo tem borda verde e sombra forte; cards inativos têm header completo visível
2. Clicar em card inativo:
   - Animação FLIP (~450ms): card clicado anima para a posição do ativo (largura cresce, posição desce); card antes ativo encolhe
   - `aria-expanded` atualiza corretamente em cada header
3. Navegar via teclado: Tab para um header inativo, Enter/Space dispara o swap
4. Card 1 (Auditoria): clicar "Expandir" em uma linha — payload JSON colapsado abre inline com `role_merge_decisions` + `canonical_role_merge_candidates` + `opus_arbitration_outcomes` formatado com syntax highlighting
5. Veredictos exibidos em PT-BR: UNIFICAR (verde) / MANTER AMBOS (cinza) / REVISAR (amarelo)
6. Card 3 (Aguardando revisão): atores `system_user_id` exibidos como badge "IA" (roxo); demais como "Admin manual" (azul)
7. Aba Habilidades: badges de skill_type (técnica/comportamental/híbrida) ao lado de cada label; itens cross-type têm fundo rosado (warn-bg); clicar "Unificar" em cross-type abre `CrossTypeConfirmModal` exigindo confirmação textual

#### S8 — Smoke de simetria CV-roles

1. Subir CV teste com mix de roles (1 com confidence > 80, 1 com confidence < 60, 1 com role nova não-CBO)
2. Verificar:
   ```sql
   SELECT
     resume_id, role, confidence,
     passed_hard_gate,
     canonical_role_id IS NOT NULL AS has_canonical
   FROM resume_roles
   WHERE resume_id = '<resume_test>'
   ORDER BY confidence DESC;
   ```
3. Esperado:
   - Todas as 3 linhas presentes
   - Role com confidence > 80: `passed_hard_gate = true`
   - Role com confidence < 60: `passed_hard_gate = false`
   - Role nova: `canonical_role_id` populada via INSERT em `job_canonical_roles` com `status='pending', source='resume_extraction'`
4. Verificar evento de auditoria:
   ```sql
   SELECT count(*) FROM job_canonical_roles
   WHERE source = 'resume_extraction' AND status = 'pending'
     AND created_at >= NOW() - interval '5 minutes';
   ```
   Esperado: contagem corresponde a roles novas criadas pelo teste.

#### S9 — Confirmar zero impacto de CV em vacancy_count / promoção

```sql
-- Após o smoke S8 (que cria role nova via CV), confirmar que NÃO incrementou:
SELECT id, vacancy_count, distinct_sources_count, status, source
FROM job_canonical_roles
WHERE source = 'resume_extraction'
  AND created_at >= NOW() - interval '10 minutes';
-- Esperado: distinct_sources_count = 0 ou unchanged.
-- CV não conta como fonte de promoção — D-PS-05.
```

#### S10 — Smoke dos 5 painéis B7

Procedimento via curl ou ferramenta de teste de API:

```bash
# Painel 1 — custo Opus por canônico (30d)
curl -H "Cookie: <admin_session>" \
  https://app.calibracv.com/api/admin/limiares/panel_b7_1_custo_opus?days=30
# Esperado: 200, JSON com { panel:'b7_1_custo_opus', days:30, rows:[...] }

# Painel 2 — auto vs manual
curl -H "Cookie: <admin_session>" \
  https://app.calibracv.com/api/admin/limiares/panel_b7_2_merge_auto_vs_manual?days=30
# Esperado: 200, JSON com data={ auto, manual, total, auto_ratio, by_entity }

# Painel 3 — distribuição latest_posted
curl -H "Cookie: <admin_session>" \
  https://app.calibracv.com/api/admin/limiares/panel_b7_3_latest_posted_distribution
# Esperado: 200, JSON com data={ role:{...}, skill:{...} }

# Painel 4 — opus effectiveness (depende de v_opus_effectiveness)
curl -H "Cookie: <admin_session>" \
  https://app.calibracv.com/api/admin/limiares/panel_b7_4_opus_effectiveness
# Esperado: 200 com rows OU 503 com error='view_v_opus_effectiveness_missing'

# Painel 5 — suggestion rejected
curl -H "Cookie: <admin_session>" \
  https://app.calibracv.com/api/admin/limiares/panel_b7_5_suggestion_rejected_by_skill?days=30
# Esperado: 200, JSON com rows=[...]
```

#### S11 — Smoke dos endpoints públicos de cascata

```bash
# Sem auth, deve retornar 200
curl https://app.calibracv.com/api/taxonomy/domains
# Esperado: { domains: [{id,name},...] }

# Cascata com domain_id válido
curl 'https://app.calibracv.com/api/taxonomy/roles-by-domain?domain_id=<uuid_valido>'
# Esperado: { roles: [{id,label},...] }

# Sem domain_id
curl https://app.calibracv.com/api/taxonomy/roles-by-domain
# Esperado: { roles: [] }
```

#### S12 — Smoke end-to-end "Rejeitar e banir"

```bash
# 1. Identificar um candidato role pendente (estado inicial)
CANDIDATE_ID=$(psql -t -c "SELECT id FROM canonical_role_merge_candidates
  WHERE resolved_at IS NULL LIMIT 1;")

# 2. Capturar label do canonical_a (que será banido)
A_LABEL=$(psql -t -c "SELECT jcr.canonical_label
  FROM canonical_role_merge_candidates crmc
  JOIN job_canonical_roles jcr ON jcr.id = crmc.canonical_a_id
  WHERE crmc.id = '$CANDIDATE_ID';")

# 3. Chamar endpoint com add_to_blacklist=true (autenticado como admin)
curl -X POST -H "Cookie: $ADMIN_COOKIE" -H "Content-Type: application/json" \
  -d "{\"candidate_id\":\"$CANDIDATE_ID\",\"decision\":\"rejected\",\"entity_type\":\"role\",\"add_to_blacklist\":true,\"reason\":\"smoke test S12\"}" \
  https://app.calibracv.com/api/admin/merge-canonicals/ignore
# Esperado: { decision: 'rejected', blacklisted: true, blacklist_entry_id: <uuid> }

# 4. Verificar persistência atômica
# role_merge_decisions não tem coluna candidate_id — reconstituir via JOIN simétrico
# em (source_id, target_id) ↔ (canonical_a_id, canonical_b_id)
psql -c "SELECT rmd.status
  FROM role_merge_decisions rmd
  JOIN canonical_role_merge_candidates crmc
    ON  (crmc.canonical_a_id = rmd.source_id AND crmc.canonical_b_id = rmd.target_id)
     OR (crmc.canonical_a_id = rmd.target_id AND crmc.canonical_b_id = rmd.source_id)
  WHERE crmc.id = '$CANDIDATE_ID';"
# Esperado: 1 linha com status='rejected'

psql -c "SELECT label, entity_type, source FROM taxonomy_blacklist WHERE label = '$A_LABEL';"
# Esperado: 1 linha com entity_type='role', source='merge_review_contextual'

# 5. Tentar banir o mesmo label de novo → deve falhar com 409
curl -X POST -H "Cookie: $ADMIN_COOKIE" -H "Content-Type: application/json" \
  -d "{\"candidate_id\":\"<outro_id>\",\"decision\":\"rejected\",\"entity_type\":\"role\",\"add_to_blacklist\":true}" \
  https://app.calibracv.com/api/admin/merge-canonicals/ignore
# Esperado: 409 { error: 'label_already_blacklisted', label, existing_blacklist_id }
```

#### S13 — Smoke dos 3 painéis novos

```bash
# Painel 11 — Confidence calculável (snapshot)
curl -H "Cookie: $ADMIN_COOKIE" \
  'https://app.calibracv.com/api/admin/limiares/panel_11_confidence_calculable?days=30'
# Esperado: { panel: 'p11_...', data: { snapshot: { role: {calculable,not_calculable}, skill: {...} }, thresholds: {...} }, available_windows: ['24h','7d','30d'] }

# Painel 12 — Fila Opus (janela 7d)
curl -H "Cookie: $ADMIN_COOKIE" \
  'https://app.calibracv.com/api/admin/limiares/panel_12_opus_review_queue?days=7'
# Esperado: { data: { snapshot: { role: {in_cooldown,eligible_for_rereview,exiting_cooldown_in_window}, skill: {...} } } }

# Painel 13 — Fila merge candidates (janela 30d)
curl -H "Cookie: $ADMIN_COOKIE" \
  'https://app.calibracv.com/api/admin/limiares/panel_13_merge_review_queue?days=30'
# Esperado: idem painel 12 + attempt_limit_reached por entidade
```

#### S-LP — Smoke `latest_posted_at` em JCR (4 cenários, atende §2.9 + AL-4)

Pré-condição: pelo menos 1 role ativo (`R1`) com 0 vagas e 1 role ativo (`R2`) com 0 vagas. Em base resetada, criar dois roles dummy via `INSERT` direto + `UPDATE status='active'`.

```sql
-- Cenário 1: INSERT vaga curada → role atualiza
INSERT INTO job_postings (id, canonical_role_id, curation_status, posted_at)
VALUES (gen_random_uuid(), '<R1_ID>', 'curated', '2026-05-01 10:00:00+00');
SELECT latest_posted_at FROM job_canonical_roles WHERE id = '<R1_ID>';
-- Esperado: 2026-05-01 10:00:00+00

-- Cenário 2: REASSIGN — vaga move de R1 para R2
UPDATE job_postings SET canonical_role_id = '<R2_ID>'
WHERE canonical_role_id = '<R1_ID>' AND posted_at = '2026-05-01 10:00:00+00';
SELECT latest_posted_at FROM job_canonical_roles WHERE id IN ('<R1_ID>', '<R2_ID>');
-- Esperado: R1 → NULL (perdeu a vaga única); R2 → 2026-05-01

-- Cenário 3: UPDATE para data MENOR (GREATEST falharia aqui — MAX funciona)
UPDATE job_postings SET posted_at = '2025-12-01 10:00:00+00'
WHERE canonical_role_id = '<R2_ID>';
SELECT latest_posted_at FROM job_canonical_roles WHERE id = '<R2_ID>';
-- Esperado: 2025-12-01 (refletindo a nova data menor, NÃO mantendo o GREATEST anterior)

-- Cenário 4: DELETE da única vaga curada → role volta a NULL
DELETE FROM job_postings WHERE canonical_role_id = '<R2_ID>';
SELECT latest_posted_at FROM job_canonical_roles WHERE id = '<R2_ID>';
-- Esperado: NULL
```

Se algum cenário falhar, o trigger ainda está usando `GREATEST(old, new)` em vez de `MAX()` — auditar `fn_jcr_recompute_latest_posted_at` no banco.

#### S-RLS — Smoke imutabilidade `pipeline_config_history` (atende §2.20a + M-3)

```sql
-- Cenário 1: SELECT como admin funciona
SET LOCAL ROLE authenticated;
SET LOCAL "request.jwt.claim.sub" = '<UUID_ADMIN>';
SELECT COUNT(*) FROM pipeline_config_history;
-- Esperado: count >= 0 sem erro
RESET ROLE;

-- Cenário 2: UPDATE direto bloqueado mesmo como service_role
SET LOCAL ROLE service_role;
DO $$ BEGIN
  UPDATE pipeline_config_history SET reason = 'tampered'
  WHERE id = (SELECT id FROM pipeline_config_history LIMIT 1);
  RAISE EXCEPTION 'UPDATE deveria ter falhado';
EXCEPTION WHEN OTHERS THEN
  IF SQLERRM LIKE '%pipeline_config_history é imutável%' THEN
    RAISE NOTICE 'OK: trigger de imutabilidade bloqueou UPDATE';
  ELSE
    RAISE;
  END IF;
END $$;
RESET ROLE;

-- Cenário 3: DELETE direto bloqueado mesmo como service_role
SET LOCAL ROLE service_role;
DO $$ BEGIN
  DELETE FROM pipeline_config_history
  WHERE id = (SELECT id FROM pipeline_config_history LIMIT 1);
  RAISE EXCEPTION 'DELETE deveria ter falhado';
EXCEPTION WHEN OTHERS THEN
  IF SQLERRM LIKE '%pipeline_config_history é imutável%' THEN
    RAISE NOTICE 'OK: trigger de imutabilidade bloqueou DELETE';
  ELSE
    RAISE;
  END IF;
END $$;
RESET ROLE;

-- Cenário 4: INSERT via RPC continua funcionando + sem duplicação de histórico
-- (trigger duplicador trg_pipeline_config_audit foi dropado em §2.20)
SELECT set_pipeline_config_value(
  'role.hard_gate.min_confidence', '0.65',
  '<UUID_ADMIN>', 'smoke S-RLS'
);

-- Conferir que somente UMA linha foi inserida no histórico (não duas)
SELECT COUNT(*) FROM pipeline_config_history
WHERE key = 'role.hard_gate.min_confidence'
  AND reason = 'smoke S-RLS';
-- Esperado: 1 (era 2 antes de §2.20 dropar trg_pipeline_config_audit)
```

#### S-LOOKUP-NORM — Smoke normalização de skills com símbolos/acentos (atende C-4)

Valida que `lookup_canonical_skill_by_normalized_label` encontra canonicals via `normalize_skill_label` em vez de `lower(trim())`.

```sql
-- Pré-condição: skill canonical existente para C++ (após paridade-skills v11)
-- Q1b confirmou 53.478 linhas todas em cbo_mte_2002_seed; assumir que pelo menos
-- um canonical normaliza para 'cpp' nesse conjunto, OU criar fixture local.
SELECT id, label FROM job_canonical_skills
WHERE normalize_skill_label(label) = 'cpp'
  AND status = 'active' AND merged_into IS NULL
LIMIT 1;

-- Cenário 1: input 'C++' resolve para o mesmo canonical (símbolo → cpp)
SELECT lookup_canonical_skill_by_normalized_label('C++', NULL) AS resolved_id;
-- Esperado: id do canonical da query anterior

-- Cenário 2: criar fixture local com acento e validar
INSERT INTO job_canonical_skills (label, status, source)
VALUES ('Integracao Continua', 'active', 'manual_admin')
ON CONFLICT DO NOTHING;
-- Nota: UNIQUE em normalize_skill_label, mas conflict resolution não suporta
-- expressão em ON CONFLICT — usar bloco DO NOTHING genérico ou validar via
-- SELECT antes do INSERT em produção. Para smoke local, esse INSERT é one-shot.

SELECT lookup_canonical_skill_by_normalized_label('Integração Contínua', NULL) AS resolved_with_accent;
-- Esperado: id do canonical 'Integracao Continua' (com acento normalizado pelo translate)
```

---

### §7.2 — Critérios de aceite

| Frente | Critério |
|---|---|
| Simetria comportamental | Teste de integração simula promoção de canonical com mediana 0.75 e valida que `needs_opus_review` fica TRUE imediatamente em ambos os lados (trigger escuta status) |
| Rename atômico | Objetos novos existem e antigos não existem; `audit-rpc-coverage` CRON roda sem erro |
| Tela calibração | Manualmente: editar chave de criticidade alta sem confirmar → erro; com "PUBLICAR" → sucesso; drawer de histórico mostra entrada; rollback via ↺ funciona com guard contra `previous_value IS NULL` |
| Tela merge-canonicals | Manualmente: processar candidato role → ficha some com animação FLIP; processar skill na outra aba; histórico mostra ambos |
| Split-button "Rejeitar e banir" | S12 passa: rejeição registrada em `*_merge_decisions` + INSERT em `taxonomy_blacklist` na mesma transação; tentativa de banir label já banido retorna 409 com `existing_blacklist_id` |
| `taxonomy_blacklist` schema | UNIQUE constraint `uq_taxonomy_blacklist_label_entity` ativa; RLS habilitada; policy `taxonomy_blacklist_admin_all` usa `profiles.role='admin'` (consistente com §2.20a) |
| A1.3 | Script CLI `npx tsx scripts/run-domain-backfill.ts --dry-run` retorna contagem; sem `--dry-run` popula `canonical_role_domain_links` |
| AdminModal | Os 6 modais migrados funcionam: focus trap, prefers-reduced-motion respeitado, useId gera aria-labelledby único |
| 12 painéis ativos | S13 passa: 3 endpoints novos retornam dado válido + `available_windows`; frontend renderiza `<PanelDisabled />` quando janela atual fora da capacidade; cobertura 24/24 chaves com tooltip→painel |
| Limpeza event_names | Grep em `canonical_creation_blocked_` no codebase TS retorna zero hits; RPC `o6_recent_errors` não menciona `resolved_id_mismatch` |

### §7.3 — Rollback

**Parcial:**
- Cada migration tem snapshot `pg_get_functiondef` prévio salvo em `docs/migrations/sprint-cleanup-v3/snapshots/NN_pre.sql`. Rollback de uma migration individual = aplicar o snapshot correspondente.
- Triggers podem ser revertidos via DROP + CREATE com nome antigo.

**Completo:**
- Rollback de toda a sprint = aplicar snapshots em ordem reversa (do 25 para o 01) + reverter mudanças TS via `git revert` do PR merged.
- Estimativa: ~30min em ambiente pré-produção limpo.

---

## §8 Sub-PRs breakdown

Sub-PRs por afinidade temática. Ordem de aplicação dentro de cada sub-PR é fixa quando há dependência funcional (caso explícito do SUB-PR-3 abaixo).

**Notas críticas de dependência:**

- **Dentro do SUB-PR-3**, a ordem de aplicação SQL é fixa: §2.10 (`auto_assign_family_to_role`) **antes** de §2.12 (`fn_promote_role_on_threshold`), porque a função de promoção faz `PERFORM auto_assign_family_to_role(...)` em seu corpo. A numeração dos arquivos (10 → 11 → 12 → 13 → 14) já reflete essa ordem.
- **SUB-PR-4 e SUB-PR-10 devem deployar juntos ou em deploy contíguo.** O CRON `pipeline-maintenance/route.ts:106` chama `supabase.rpc('catchup_pending_promotions')`. Após SUB-PR-4 (que dropa a função antiga), o CRON quebra até SUB-PR-10 atualizar para `catchup_pending_role_promotions`. **Protocolo operacional (D-2, Claude Code PO B) — duas opções, escolha do executor (Antigravity) explicitada no descritivo do PR:**
  - **(a) Deploy único combinado:** abrir um PR único que aplica simultaneamente as migrations §2.23/§2.24 (rename SQL) e os edits TS de §3.11 (rename callers). Risco: PR maior, review mais lento. Vantagem: zero janela de quebra do CRON.
  - **(b) Pausa orquestrada do CRON entre deploys:** antes do deploy de SUB-PR-4, pausar o CRON `pipeline-maintenance` via Vercel Cron config (`disabled: true` no `vercel.json` ou via UI Vercel Cron Jobs). Após SUB-PR-10 ter feito merge e deploy, reativar o CRON. Risco: dependência de operador manual lembrar do passo de pausa/reativação. Vantagem: PRs menores, deploys independentes.

  **Recomendação:** opção (a) — deploy único — porque base atual está em pré-MVP (sem usuários, sem ingestão de vagas via API ainda) e a janela de "perigo" da opção (b) só vale a pena com volume real em produção. Antigravity deve documentar a opção escolhida no corpo do PR (mensagem do commit ou descrição do MR) para auditoria posterior.
- **SUB-PR-9 deve preceder o endpoint A1.3 dentro de SUB-PR-11.** O endpoint `/api/admin/backfill/canonical-domains` importa `lib/admin/domain-backfill.ts` criado em §3.9.
- **SUB-PR-8 pode deployar isolado.** Nenhum endpoint atual da §4 e nenhuma página atual consome `v_merge_audit_history`, `reconcile_canonical_*` ou nova versão de `o6_recent_errors` (consumer real está em `app/api/admin/bi/recent-errors/route.ts`, que continua funcionando porque o filtro `'hard_gate'` permanece — só os event_names mudam, transparente para o caller).

```
SUB-PR-1: §2.0a (normalize_role_label), §2.1, §2.2, §2.3, §2.7, §2.8, §2.8b, §2.25, §2.31, §2.35 (schema additions sem dependências + RPC blacklist contextual + resume_skills paralela; §2.0a habilita índices de §2.5 e coluna gerada de §2.7/§2.8b)
   ↓
SUB-PR-2: §2.4, §2.5, §2.6, §2.6b, §2.15, §2.18 (lookups + índices funcionais, simétricos para role+skill)
   ↓
SUB-PR-3: §2.10 → §2.11 → §2.12 → §2.13 → §2.14 (funções na ordem correta de dependência)
   ↓
SUB-PR-4: §2.21 (rename atômico — depende de SUB-PR-3)
   ↓ ←─── SUB-PR-10 deve landar junto ou imediatamente após (CRON quebra senão)
SUB-PR-5: §2.22 (triggers de flag com OF status)
   ↓
SUB-PR-6: §2.23, §2.24 (catchups + Opção C — depende de SUB-PR-4)
   ↓
SUB-PR-7: §2.9 (trigger latest_posted_at JCR com MAX recompute)
   ↓
SUB-PR-8: §2.16, §2.17, §2.19, §2.19a, §2.20 (DROP trigger duplicador + RPC), §2.20a (trigger BEFORE UPDATE/DELETE)
   │       Inclui:
   │       - §2.17 registry com 29 linhas (5 panel_ids: pricing/merge_canonicals/campaigns/ingestor/pipeline_config)
   │       - §2.19a coluna description + seed 24 chaves
   │       - §2.20a imutabilidade via trigger (substituiu RLS FORCE)
   ↓
SUB-PR-8a: §2.26, §2.27, §2.28, §2.29, §2.30 (5 painéis B7) + §2.32, §2.33, §2.34 (3 painéis cobertura 100%)
   │       Pode landar em paralelo com SUB-PR-9 (sem dependência cruzada).
   │       §2.30 aborta se view v_opus_effectiveness não existir → Painel B7 #4 inativo nesse cenário (outros 4 OK).
   ↓
SUB-PR-9: §3.1–§3.9 (TypeScript backend — domain-backfill em §3.9 precede endpoint A1.3)
   ↓
SUB-PR-10: §3.10–§3.14 (rename TS callers + limpeza de event_names + pipeline-config-tooltips — DEVE landar junto com SUB-PR-4)
   │       Atualiza:
   │       - app/api/cron/pipeline-maintenance/route.ts:106 → catchup_pending_role_promotions
   │       - app/api/cron/audit-rpc-coverage/route.ts:27 → whitelist
   │       - tests/pipeline/sprint-v11-pr7-7a-integration.test.ts (linhas 5, 50, 53)
   │       - lib/pipeline/persist-curation/constants.ts:22,31 → role_creation_blocked_*
   │       - §3.14 lib/admin/pipeline-config-tooltips.ts (novo, alimenta §5.1)
   ↓
SUB-PR-11: §4 (endpoints admin — endpoint A1.3 depende de §3.9 em SUB-PR-9; endpoints B7 dependem de SUB-PR-8a)
   │       Inclui:
   │       - §4.1 (6 endpoints pipeline-config incluindo rollback)
   │       - §4.2 (audit + payload)
   │       - §4.3 (manual analyze/execute)
   │       - §4.4 (details panel lateral)
   │       - §4.6 (A1.3 admin domains: role-domains + canonical-roles-by-domain + status backfill + PUT/POST)
   │       - §4.7 (5 endpoints B7 — depende de SUB-PR-8a)
   │       - §4.8 (2 endpoints públicos cascata — taxonomy/domains e taxonomy/roles-by-domain)
   ↓
SUB-PR-12: §5.1 (tela pipeline-config — 6 colunas, modais edição/histórico/rollback)  ┐
SUB-PR-13: §5.2, §5.3 (merge-canonicals pilha-degrau + DomainRoleCascade)              ├ paralelos após SUB-PR-11
   │       (split-button "Rejeitar e banir" depende de taxonomy_blacklist em §2.31 — SUB-PR-1)
SUB-PR-14: §5.4 (AdminNav reordenado sem Auditoria IA)                                ┘
   ↓
SUB-PR-15: §6 (AdminModal + migração de 6 modais — pode iniciar em paralelo com SUB-PR-1, sem dependência de DB ou endpoints; recomendado paralelizar para reduzir clock wall time da sprint)
   │       Ordem de migração de modais sugerida (do simples para o complexo):
   │       1. IgnoreModal merge-canonicals (6.2.2 — baseline)
   │       2. IgnoreModal merge-skills com harmonização (6.2.3)
   │       3. ComparisonModal (6.2.4)
   │       4. DuplicatesModal (6.2.5)
   │       5. PipelineModal (6.2.6 — testar Escape bloqueado e ✕ disabled)
   │       6. ResourceEditModal (6.2.1 — multi-step, mais arriscado, por último)
```

**Deploy do SUB-PR-4 + SUB-PR-10:** Recomenda-se combinar em um único PR de coordenação (PR-4+10) que toca SQL + TS no mesmo merge. Alternativa: aplicar em janela de manutenção curta com CRON pausado entre os dois deploys.

---

## Anexo D-PS — Decisões padrão

- **D-PS-01:** Convenção nominal `*_role` / `*_skill` para entidades canônicas; `_jcr` / `_jcs` preservados como abreviações de tabela.
- **D-PS-02:** `retire_canonical(uuid, text [, text])` é dispatcher polimórfico legítimo; **não renomear**.
- **D-PS-03:** Famílias taxonômicas aplicam-se apenas a roles. Função `auto_assign_family_to_role` é role-specific.
- **D-PS-04:** Mediana JCR usa `function_orchestrator_items.confidence` (confidence de extração); mediana JCS usa `job_posting_skills.confidence`. Semanticamente equivalentes (ambas extração).
- **D-PS-05:** Promoção JCR e JCS usa apenas volume (vacancy_count + distinct_sources_count + has_recent). Gate de mediana removido em ambos (Opção C).
- **D-PS-06:** Fila Opus pós-promoção é responsabilidade exclusiva dos triggers `fn_flag_needs_opus_review_*`, que escutam `OF confidence_median, status`. Triggers de promoção **não** setam `needs_opus_review`.
- **D-PS-07:** Hard gate aplicado **antes** da canonização (gate batch via `filterByHardGate`).
- **D-PS-08:** JCS usa coluna `label`; JCR usa `canonical_label`. Assimetria intencional, lookups respeitam.
- **D-PS-09:** Hard gate threshold default = 0.70 (key `*.hard_gate.min_confidence`).
- **D-PS-10:** Auto-promotion threshold default = 0.85 (key `*.promotion.auto_min_confidence`). Usado apenas como upper threshold da zona Opus pós-promoção.
- **D-PS-11:** Eventos novos emitidos por triggers usam `actor = 'system'`, `actor_id = SYSTEM_USER_ID`.
- **D-PS-12:** SQL migrations executadas direto pelo Antigravity via conexão Supabase.
- **D-PS-13:** Sem UPDATE retroativo em eventos históricos no rename (base limpa).
- **D-PS-14:** Backfill de domain via IA usa pattern CLI/Cron (não fire-and-forget Vercel).
- **D-PS-15:** Sanitização obrigatória de `domain_id` retornado pelo LLM contra allowlist de domínios ativos.
- **D-PS-16:** AdminModal compartilhado usa `useId()`, `useReducedMotion`, `useFocusTrap<T>(isOpen)` retornando ref.
- **D-PS-17:** Reordenação alfabética PT-BR no AdminNav.
- **D-PS-18:** Modais admin não permitem fechar por click no backdrop quando em modo "edit unsaved" (preserva trabalho).
- **D-PS-19:** Mediana JCR não tem gate de promoção (preservado da v11; agora aplicado simetricamente em JCS via Opção C).
- **D-PS-20:** Pipeline config de criticidade `high` exige confirmação textual "PUBLICAR" + razão.
- **D-PS-21:** Sentinel date `9999-12-31T23:59:59Z` para créditos sem expiração (preservado).
- **D-PS-22:** Snapshots `pg_get_functiondef` pré-migration obrigatórios em `docs/migrations/sprint-cleanup-v3/snapshots/`.
- **D-PS-23:** Wrapper trigger `trigger_auto_assign_family` é renomeado para `trigger_auto_assign_family_role` em coerência com a convenção semântica.
- **D-PS-24:** Os 2 triggers `auto_assign_family_on_active` (AFTER UPDATE + AFTER INSERT) são consolidados em 1 trigger `AFTER INSERT OR UPDATE OF status` com guard via `TG_OP` no wrapper. Mantém cobertura de ambos os eventos com menos objetos no schema.
- **D-PS-25:** `suggestDomainForRole` carrega `DOMAIN_LIST` dinamicamente do banco no início do batch (não hardcoded), resiliente a novos domínios adicionados por migration futura sem necessidade de sincronização manual.
- **D-PS-26:** O `IgnoreModal` em `merge-skills` é incluído na sprint cleanup com harmonização de API ao padrão do `merge-canonicals` (corrige regressão de acessibilidade: `role="dialog"`, `aria-modal`, `aria-label`, `maxLength=500`).
- **D-PS-27:** Cadeia auto_assign_family após rename:  
  Trigger `auto_assign_family_on_role_active` → wrapper `trigger_auto_assign_family_role()` → função regular `auto_assign_family_to_role(uuid, boolean)`. Callers que invocam diretamente a função regular via `PERFORM`: `fn_promote_role_on_threshold` (§2.12) e `catchup_pending_role_promotions` (§2.23). Ambos chamam o nome novo após o rename.
- **D-PS-28:** SUB-PR-4 (rename SQL) e SUB-PR-10 (rename TS callers) devem ser coordenados em deploy único ou contíguo, sob risco de quebrar o CRON `pipeline-maintenance` que invoca `catchup_pending_promotions` por nome antigo.
- **D-PS-29 (registry §2.17):** O seed de `admin_panel_functions` cobre todas as funções operacionais expostas nas telas admin existentes (`/admin/pricing`, `/admin/merge-canonicals`, `/admin/campaigns`, `/admin/ingestor`) mais `/admin/pipeline-config` que entra nesta sprint. As 29 linhas refletem o estado pretendido pós-sprint. Quando uma função for adicionada/removida da UI, o seed deve ser atualizado no mesmo PR — registry sem cobertura é equivalente a não ter registry.
- **D-PS-30 (mapeamento de veredictos):** Tradução backend ↔ UI em `lib/admin/merge-decision-labels.ts`:
  - `*_merge_decisions.status`: `merged` → UNIFICAR (success), `ignored` → MANTER AMBOS (neutral), `rejected` → REJEITAR (danger), `pending` → PENDENTE (warning)
  - `opus_arbitration_outcomes.decision` e `canonical_*_merge_candidates.opus_decision`: `MERGE` → UNIFICAR, `KEEP_BOTH` → MANTER AMBOS, `NEEDS_HUMAN` → REVISAR
  - `rejected` tem label próprio `REJEITAR` (não é agrupado com MANTER AMBOS). Card 3 (§5.2.5) tem 3 botões de decisão — Unificar / Manter ambos / Rejeitar (este último em split-button por D-PS-38) — além do botão `Detalhes` (que é navegação, abre painel lateral). `ignored` e `rejected` permanecem mecanicamente idênticos para o motor de sugestões; distinção é exclusivamente de produto, visível em auditoria.
- **D-PS-31 (auditoria integrada):** Não há página separada `/admin/merge-canonicals/audit` — auditoria de decisões automáticas é Card 1 do layout pilha-degrau de `/admin/merge-canonicals` (§5.2.3). Reduz fricção de navegação e mantém contexto operacional no mesmo lugar.
- **D-PS-32 (rota separada):** `/admin/pipeline-config` é rota dedicada, separada de `/admin/dashboard?tab=limiares`. O `LimiaresTab` exibe os 12 painéis de observabilidade; a tela `/admin/pipeline-config` é dedicada à calibração das chaves que **alimentam** esses painéis. Integração visual via tags `Painel X` na coluna "Acompanhamento" da tabela §5.1.4.
- **D-PS-33 (tooltips client-side):** Os textos da coluna "Acompanhamento" em §5.1.4 vivem em `lib/admin/pipeline-config-tooltips.ts` (§3.14), não em coluna jsonb do banco. Justificativa: textos longos, mudam pouco, evita query extra por linha. Adicionar uma chave nova a `pipeline_config` requer adicionar o mapeamento aqui no mesmo PR. **Adição (status pós-cleanup):** a proibição original de "endpoint dinâmico de cálculo de impacto no MVP" valia apenas para o escopo desta cleanup v2.6. A sprint orchestrator simétrico subsequente reabre essa frente via `GET /api/admin/pipeline-config/[key]/impact?new_value=X&days=N`, com componente `ImpactPreview` (tabela de contagem + histograma + seletor 7d/30d/90d) **integrado** ao modal §5.1.6 — **sem substituir** a caixa qualitativa de painéis afetados desta cleanup. Ambas convivem: a caixa qualitativa orienta o admin sobre onde olhar (painéis a acompanhar); o `ImpactPreview` quantifica o impacto da mudança proposta sobre o universo de items no banco. Esta decisão de integração (não substituição) é explicitada também no apêndice "Interfaces para a sprint orchestrator" ao final desta spec.
- **D-PS-34 (RLS FORCE):** `pipeline_config_history` recebe `FORCE ROW LEVEL SECURITY` (§2.20a) — mesmo o owner não consegue INSERT/UPDATE/DELETE direto. Apenas a RPC `set_pipeline_config_value` (SECURITY DEFINER) grava. Garante auditoria imutável como descrito no mockup.
- **D-PS-35 (5 painéis B7 incluídos no escopo):** os 4 RPCs SQL B7 (`admin_panel_custo_opus_por_canonical`, `admin_panel_merge_auto_vs_manual`, `admin_panel_latest_posted_distribution`, `admin_panel_suggestion_rejected_by_skill`) + validador `v_opus_effectiveness` entram nesta sprint como migrations §2.26-§2.30 com endpoints §4.7. Diretriz operacional: spec é fotografia do estado pretendido, incluindo todo o conteúdo de implementação (bodies SQL, snippets TS, validações).
- **D-PS-36 (endpoints públicos cascata):** `/api/taxonomy/domains` e `/api/taxonomy/roles-by-domain` (§4.8) são distintos dos endpoints admin homônimos em §4.6. Diferenças: sem auth, cache 300s, dados reduzidos (sem `slug`, `is_active`, `display_order`). Permitem montagem do `DomainRoleCascade` em fluxos públicos (formulário de criação de perfil profissional). Sem esses endpoints, a cascata fica restrita ao admin e a frente A1.3 entrega só metade.
- **D-PS-38 (split-button "Rejeitar e banir" no Card 3):** o split-button do Card 3 da §5.2.5 captura o caso de uso "admin vê par claramente lixo no fluxo de revisão e quer banir o termo sem trocar de tela", sem violar a decisão de manter a Blacklist como tela própria. Justificativa: gestão completa da Blacklist (CRUD, paginação, filtros, importação em lote, histórico, estatísticas) é tema de sprint futura via menu admin dedicado — mas a ação contextual de "rejeitar este par específico e banir o canonical_a" cabe no fluxo de unificação atual. Trade-off: a tabela `taxonomy_blacklist` é criada nesta sprint (§2.31) apenas com schema mínimo + RLS, sem CRUD via API admin. Endpoint `POST /.../ignore` recebe parâmetro `add_to_blacklist: boolean`; transação atômica via RPC `reject_and_blacklist_canonical_pair` (§2.35) garante consistência ou ROLLBACK em caso de erro.
- **D-PS-39 (threshold 1.5 e algoritmo família-por-membros em `auto_assign_family_to_role`):** o algoritmo da §2.10 é **família-por-membros**, não família-por-centróide. A função encontra até 5 canonicals similares ao role de entrada (cosine ≥ 0.75 entre embeddings de canonicals), e pontua cada família ativa pela presença desses similares dentro dela: +1.0 se algum dos 5 está na família, +0.5 se a família tem 2+ canonicals compartilhando o mesmo domain_id do role de entrada, +0.5 se a família tem canonicals com vagas no mesmo industry_normalized. Threshold 1.5. Sinais derivados via JOINs: domain via `canonical_role_domain_links` (primary, fallback highest confidence); industry via `job_postings` → `employers.industry_normalized` (voto majoritário). O threshold é intencionalmente restritivo: só atribui família quando há evidência convergente (vizinhança semântica + contexto domain ou industry). Cenários em que a função não atinge o threshold (incluindo cold start em base resetada com 0 canonicals ativos) caem em `role_orphan_no_family_assigned` e são cobertos pelo fluxo Opus via `findOrCreateFamilyAndLink` (caminho independente, processadores `orphan-canonical` e `taxonomy-relation`).
- **D-PS-40 (instrumentação assimétrica do pipeline upstream — débito reconhecido):** as tabelas `function_orchestrator_runs` e `function_orchestrator_items` rastreiam o pipeline de curadoria de vagas com fidelidade granular para role (`canonical_role_proposed`, `canonical_status`, `pipeline_stage`, `error_type`, `action_required`) mas apenas com agregado mínimo para skills (`skills_count smallint`). O `DiscoverResult { discovered, reused, pending_created }` é calculado em `safeDiscoverAndLinkSkills` mas descartado (void return). Impacto: debugging skill-por-skill no contexto de um run é inviável; alertas de qualidade não separam degradação de extração de role vs de skill; rastreabilidade de "esta skill virou pending nesse run" requer path indireto via timestamps. **Esta sprint NÃO fecha esse gap — foco em aparato pós-canonical.** Frente dedicada futura: criar `function_orchestrator_skill_items` paralela, adicionar contadores `skills_extracted/reused/pending_created/gate_rejected/failed` em `function_orchestrator_runs`, modificar `safeDiscoverAndLinkSkills` para retornar `DiscoverResult` em vez de void.
- **D-PS-41 (A1.3 assimetria intencional — domínios só para roles):** as estruturas de domínio (`canonical_role_domains`, `canonical_role_domain_links`) e os endpoints/UI relacionados (§3.9 backfill, §4.6 admin endpoints, §4.8 endpoints públicos, §5.3 `DomainRoleCascade`) são exclusivos do lado role. Justificativa: o domínio é uma categoria operacional ligada à área profissional ("Tecnologia", "Recursos Humanos", "Marketing"...) que se aplica naturalmente a uma função/cargo. Skills (habilidades) são atributos transversais a múltiplas áreas — "Liderança" pertence tanto a "Gerente de RH" (RH) quanto a "Tech Lead" (Tecnologia). Não há valor de produto em atribuir uma única área a uma skill; o vínculo natural skill→área se dá indiretamente via as funções onde a skill aparece. Esta assimetria é de produto, não de spec — não precisa ser corrigida e não tem janela de simetrização nesta sprint.
- **D-PS-42 (Painel B7 #5 `admin_panel_suggestion_rejected_by_skill` assimetria intencional — só skill):** o Painel 5 do B7 (§2.29) mostra ranking de habilidades com sugestões rejeitadas pelo usuário via `resume_skill_enrichments.validation_status = 'rejected'`. Não há equivalente para roles porque o produto atual não tem fluxo de "rejeição de role inferida pelo usuário" — a função canonical apresentada ao usuário na análise é tratada como fato definitivo, sem botão de "essa não é minha função". Skills, em contrapartida, têm fluxo explícito ("Não tenho essa habilidade") que alimenta `resume_skill_enrichments`. Não existe tabela `resume_role_enrichments` no banco. Caso o produto adicione fluxo de rejeição de role no futuro, a simetria do Painel 5 deve ser implementada na mesma sprint (criação simultânea de `resume_role_enrichments` + RPC `admin_panel_suggestion_rejected_by_role` + endpoint correspondente). Esta sprint reconhece e documenta a assimetria; **não** abre frente para mudar produto.
- **D-PS-43 (Pipeline CV fora do escopo de orchestrator — decisão arquitetural):** o input 1 (currículo) não tem instrumentação stage-by-stage equivalente a `function_orchestrator_items`/`function_orchestrator_runs` (que cobrem o input 2 — vagas — para os fluxos A, B, C). A decisão é **estrutural, não conjuntural**: extração de CV é fluxo unitário (1 chamada LLM → hard gate → resolução de canonical → INSERT em `resume_roles`/`resume_skills`), não pipeline multi-camada com cache/dict_match/sugestão/LLM-puro/fallback como o pipeline de vagas. As tabelas `resume_roles` e `resume_skills` (§2.7, §2.8, §2.8b) cobrem 100% da rastreabilidade necessária via colunas `passed_hard_gate`, `canonical_*_id`, `source`, `confidence`. Métricas agregadas relevantes (taxa de hard_gate reject, % com canonical resolvido, throughput por dia, distribuição de confidence) são calculáveis diretamente dessas tabelas sem instrumentação adicional. Trade-off explícito: perde-se decomposição por estágio interno do LLM (tempo no LLM vs tempo no gate vs tempo no resolve) — não considerada essencial porque o pipeline CV não otimiza por camada. Caso futuramente o pipeline CV evolua para multi-stage (extração em lote, cache de embeddings de CV, sugestão por similaridade entre CVs, etc.), reabrir esta decisão em sprint dedicada. **Esta justificativa NÃO se baseia em volume** (volume pode crescer); baseia-se na natureza arquitetural distinta entre input 1 (unitário, síncrono, user-driven) e input 2 (em lote, assíncrono, system-driven).
- **D-PS-49 (assimetria de fontes para `confidence_median` entre role e skill — documentação retrospectiva via ground truth):** o comentário inline da função `fn_jps_recompute_jcs` afirma "paridade com `fn_recompute_jcr_confidence_median` na mig 29". Ground truth via `pg_proc` revela que a paridade entre as duas funções é **estrutural** (mesmo cálculo `PERCENTILE_CONT(0.5)`, mesmos parâmetros `lookback_days`/`min_count` lidos de `pipeline_config`, mesma escrita condicional em rows com `status IN ('active', 'pending')`), **não de fonte de dados**:

  - **Role:** `fn_recompute_jcr_confidence_median` calcula mediana sobre `function_orchestrator_items.confidence` (filtrada por `function_orchestrator_runs.started_at >= NOW() - lookback`). Fonte = TODA chamada de curadoria via LLM, incluindo vagas que depois foram descartadas. Disparada por 3 triggers (`trg_foi_jcr_confidence_insert/update/delete`) em `function_orchestrator_items`, que chamam as funções `fn_jcr_confidence_median_insert/update/delete`, que por sua vez delegam a `fn_recompute_jcr_confidence_median`.

  - **Skill:** `fn_jps_recompute_jcs` calcula mediana sobre `job_posting_skills.skill_confidence` filtrada por `job_postings.curation_status = 'curated'` (e mesmo lookback). Fonte = APENAS matches que sobreviveram à curadoria completa.

  **Justificativa para manter a divergência:**
  1. As fontes têm propósitos diferentes. Role precisa de sinal de extração contínua para promoção; skill precisa de sinal robusto de uso real em vagas validadas.
  2. A robustez do sinal skill (filtro curated) é trade-off intencional — descarta ruído de vagas em processamento. Replicar isso para role exigiria estrutura análoga a `job_posting_skills` que não existe para roles hoje (não há `job_posting_roles` enquanto tabela paralela).
  3. `confidence_median` é input para promoção em ambos os lados; o threshold de promoção é calibrado independentemente via `role.promotion.auto_min_confidence` (default 0.85) e `skill.promotion.auto_min_confidence` (default 0.85) — pode ser ajustado por chave caso a característica das fontes torne necessário.

  **Defaults relacionados** (cravados em `pipeline_config` via §2.19a):
  - `role.confidence.lookback_days = 120`, `role.confidence.min_count = 5`
  - `skill.confidence.lookback_days = 120`, `skill.confidence.min_count = 3`
  - Diferença `min_count` (5 vs 3) reflete o fato de skill ter base amostral menor pós-filtro curated.

  **Implicação para sprint orchestrator simétrico (próxima sprint):** a tabela `function_orchestrator_skill_items` (criada lá) tem coluna `skill_confidence` semanticamente análoga a `function_orchestrator_items.confidence`. Há tentação arquitetural de migrar o cálculo de `confidence_median` skill para usar essa nova fonte (paridade total de fonte), mas a decisão registrada aqui é **MANTER `fn_jps_recompute_jcs` apontando para `job_posting_skills`** — a robustez do sinal pós-curadoria não deve ser sacrificada por simetria formal. A sprint orchestrator simétrico **NÃO TOCA** o circuito de confidence_median: nem para skill (já tem fonte estabelecida em job_posting_skills) nem para role (já tem triggers em FOI/FORI que delegam a `fn_recompute_jcr_confidence_median`).

  **Não há duplicação no lado role:** ground truth confirmou (Queries A, B, C) que apenas `fn_recompute_jcr_confidence_median` é escritora ativa de `job_canonical_roles.confidence_median`. Demais funções (`fn_promote_canonical_on_threshold`, `catchup_pending_promotions`, `maintenance_phase_1`, `reset_taxonomy_core`, `process_opus_create_new`) leem para gates de promoção ou apagam em reset, sem competir como escritoras.
- **D-PS-50 (contadores `canonical_created` e `canonical_promoted` em `function_orchestrator_runs` são role-only por convenção pré-existente):** essas duas colunas, sem prefixo `role_`, acumulam apenas novos canonicals de role criados e promovidos no run. A nomenclatura sem prefixo é herança histórica de quando a tabela `function_orchestrator_runs` foi originalmente criada apenas para curadoria de role. Não há equivalente skill nesta cleanup; a sprint orchestrator simétrico subsequente adicionará 5 contadores `skills_*` independentes em `function_orchestrator_runs` (`skills_extracted`, `skills_reused`, `skills_pending_created`, `skills_gate_rejected`, `skills_failed`) cobrindo o ciclo de processamento, mas **não tocará** `canonical_created`/`canonical_promoted` que permanecem role-only. Renomeação para `role_canonicals_created`/`role_canonicals_promoted` para simetria nominal completa é dívida adiada conscientemente — esses campos podem ter dependências externas (telemetria, dashboards, scripts de operação) que mapeamento até esta cleanup não capturou. Quem ler runs em consulta agregada deve saber que `canonical_created` e `canonical_promoted` refletem apenas role; promoção de skill, quando ocorrer no mesmo run, ficará visível em `events` (`event_name = 'skill_promoted_dynamic'`) e nos próprios `job_canonical_skills.promoted_at`.
- **D-PS-51 (`fallback_ratio` em `function_orchestrator_runs` é role-only por construção):** a fórmula é `curated_fallback / NULLIF(total, 0)`, operando exclusivamente em contadores de role (`curated_fallback` registra vagas que viraram canonical genérico conservador no fluxo de role; `total` é total de vagas processadas, métrica de vaga e não de skill). O alerta automático "fallback_ratio > 0.20" continua role-only após a sprint orchestrator subsequente. Não há intenção de criar coluna `skills_fallback_ratio`: a métrica equivalente é calculável on-the-fly a partir dos contadores adicionados pela próxima sprint (`skills_failed / NULLIF(skills_extracted, 0)`), sem necessidade de coluna agregada adicional. Painéis operacionais que precisem dessa razão devem computá-la diretamente na query, não esperar coluna materializada.

- **D-PS-52 (nomenclatura Opus vs IA — banco/código vs UI):** no banco e no código backend, mantém-se `Opus` como nome do modelo arbitral (event_names como `opus_arbitration_*`, tabela `opus_arbitration_outcomes`, comentários técnicos, logs). Na UI pública e em qualquer texto exposto ao usuário final ou ao admin, usa-se `IA`. Essa fronteira permite alterar o modelo (Opus → Sonnet → outro) no futuro sem rebatizar event_names ou colunas, mantendo a UI estável. Reviewers que sugerirem unificar para `IA` em todos os lugares (ChatGPT #18) devem ser orientados a respeitar essa decisão. Implementador: ao criar string user-facing, sempre `IA`; ao criar nome de função/coluna/evento, sempre `opus_*` ou termo neutro.

- **D-PS-53 (índice parcial UNIQUE em `resume_roles` para `is_primary`):** existe pré-sprint o índice `idx_resume_roles_one_primary UNIQUE btree (resume_id) WHERE (is_primary = true)`, garantindo uma role primária por currículo. Não é tocado por esta cleanup (preservado intacto). Callers que precisem marcar uma role como primária devem fazê-lo em transação: primeiro `UPDATE resume_roles SET is_primary = false WHERE resume_id = $1`, depois `UPDATE resume_roles SET is_primary = true WHERE id = $2`. Inserção via §3.3 não seta `is_primary` (default `false`), portanto não colide. Documentado aqui para callers futuros que possam tentar marcar primária e bater no constraint.

- **D-PS-54 (paridade SQL↔TS de funções de normalização):** as funções de normalização de label existem em **dois lados** e **devem ser mantidas em paridade**:
  - SQL: `normalize_skill_label(text)` (vivo desde paridade-skills v11) e `normalize_role_label(text)` (criada nesta sprint em §2.0a).
  - TS: `normalizeSkill(text: string)` e `normalizeTitle(text: string)` em `lib/pipeline/text-processing.ts`.

  Alterar qualquer uma das quatro exige alterar a **par direta** (SQL ↔ TS) E **rebuild dos índices** que dependem da função SQL alterada (`uq_jcs_label_normalized` para skill, `uq_jcr_canonical_label_normalized` para role, mais coluna gerada `resume_skills.skill_normalized` e `resume_roles.role_normalized` que dependem das funções `STORED` — coluna gerada não recomputa automaticamente em mudança de função, requer `ALTER TABLE ... DROP COLUMN ... ADD COLUMN ...` para repopular). Lookups e índices DEVEM usar a mesma função; senão produzem falsos negativos em produção (skill `C++` é armazenada como `cpp` no índice; busca com `lower(trim('C++'))` retorna `c++` e dá miss). Sprint que mexer em qualquer uma das quatro funções é responsável por inventariar todos os índices/colunas geradas e fazer rebuild simétrico.

- **D-PS-55 (`admin_panel_functions.is_enabled` é metadado informativo nesta sprint):** a coluna `is_enabled boolean` registrada na tabela `admin_panel_functions` (§2.17) **não é lida nem respeitada pelos endpoints `/api/admin/limiares/...` ou pelas telas admin nesta sprint**. Serve apenas como metadado declarativo descrevendo o estado conceitual do painel/função (útil para inventário, mapeamentos cross-doc, ou futura ativação). Reviewers que apontaram ambiguidade (ChatGPT #15 v2.8): declarado aqui explicitamente — se uma sprint futura quiser converter `is_enabled` em feature flag real, deve ser autorizada (diretriz de controles operacionais — D-PS-9) e implementada com smoke tests de cobertura. Implementadores não devem ramificar lógica em `is_enabled` nesta sprint.

- **D-PS-56 (contrato API↔RPC `set_pipeline_config_value` — parâmetros nomeados literais):** os nomes exatos dos parâmetros da RPC `set_pipeline_config_value` (§2.20) são `p_key`, `p_value`, `p_changed_by`, `p_reason`, `p_confirmed`. Qualquer chamada TS via `supabase.rpc('set_pipeline_config_value', {...})` (PATCH §4.1, rollback §4.1) deve usar exatamente esses nomes — PostgREST faz match por nome, não por posição. Reviewers que sugiram `p_new_value`, `p_actor_id`, ou que omitam `p_confirmed` (CR-2 da rodada v2.8 — Grok #2) introduzem bug invisível: PostgREST responde "Function not found" para nomes diferentes, ou usa default `false` quando `p_confirmed` é omitido, bloqueando toda edição de criticality high. Trecho TS literal cravado em §4.1 PATCH é a referência canônica.

- **D-PS-57 (branch arqueológico é barreira de segurança, não redundância):** o trigger §2.12 e os CRONs §2.23/§2.24 implementam um `SELECT EXISTS` por vagas recentes (`posted_at >= NOW() - lookback_days AND curation_status = 'curated'`) e abortam a promoção com evento `role_promotion_deferred_archaeological` (ou contador `v_archaeological` no CRON) quando nenhuma vaga recente curada é encontrada. **Esse branch é intencional e ativo, não redundante.** Embora `vacancy_count` e `distinct_sources_count` em produção sejam alimentados por agregados de vagas curadas (e portanto refletem volume real), eles não carregam temporalidade — um canonical com 50 vagas todas de 2 anos atrás teria volume alto mas nenhum sinal de mercado vivo. O branch arqueológico é a única defesa contra "ressurreição morta" de canonicals que tinham relevância histórica mas não têm mais. O custo de CPU é desprezível (`EXISTS` com índice em `(canonical_role_id, posted_at, curation_status)` retorna em microssegundos) frente ao custo de promover artefato arqueológico que vai recolher imediatamente para Opus arbitration. Não confundir com D-PS-39 (que descreve o algoritmo família-por-membros em `auto_assign_family_to_role` — função distinta da promoção e sem branch arqueológico, pois algoritmo de família não tem semântica temporal).

---

## Anexo LK-PS — Cross-references

- **LK-PS-01:** §2.11 (fn_promote_role) consome `pipeline_config` chaves `role.promotion.min_vacancies`, `min_distinct_employers`, `lookback_days`. Seed em §2.19 confirma criticidade `high`.
- **LK-PS-02:** §2.22 (triggers de flag) consome `role.promotion.auto_min_confidence` e `skill.promotion.auto_min_confidence` como upper threshold da zona Opus.
- **LK-PS-03:** §3.9 (backfill canonical domains) usa `canonical_role_domains` (tabela existente, ver D5) e popula `canonical_role_domain_links`.
- **LK-PS-04:** §5.3 (DomainRoleCascade) consumido por `ResourceEditModal` (§6.2) e pela tela de criação manual de vaga.
- **LK-PS-05:** §3.10 (constants.ts — limpeza de event_names) afeta `hard-gate.ts:80` indiretamente via `cfg.eventName`. §2.21 atualiza RPC `o6_recent_errors` no banco.
- **LK-PS-06:** §2.16 (v_merge_audit_history) consumido por §4.2 (endpoint audit) e §5.2.3 (Card 1 da tela merge-canonicals — auditoria integrada).
- **LK-PS-07:** §2.17 (admin_panel_functions) é registry de telemetria — endpoints §4.5 declaram `panel_id` + `function_id` em metadata. Não cria endpoints novos.
- **LK-PS-08:** §3.1 (evaluateHardGate) consumido por §3.2, §3.3, §3.7.
- **LK-PS-09:** §6.1 (AdminModal) consumido por §6.2 (6 modais migrados) e por modais futuros que respeitem D-PS-16.
- **LK-PS-10:** §2.21 (rename atômico) afeta callers TS listados em §3.11, §3.12, §3.13 — todos devem ser atualizados no mesmo PR ou em PR imediatamente subsequente.
- **LK-PS-11:** §2.19a (coluna `description`) consumido por §5.1.4 (coluna "Chave + descrição" da tabela) e §5.1.6 (caixa de descrição do modal de edição). Endpoint §4.1 `GET /api/admin/pipeline-config` inclui `description` no payload.
- **LK-PS-12:** §2.20a (RLS FORCE) protege `pipeline_config_history` — RPC §2.20 (`set_pipeline_config_value`, SECURITY DEFINER) é o único caminho de gravação. Endpoint §4.1 `POST /[key]/rollback` chama essa mesma RPC.
- **LK-PS-13:** §3.14 (`pipeline-config-tooltips.ts`) consumido por §5.1.4 (coluna "Acompanhamento") e §5.1.6 (lista de painéis na caixa "Impacto estimado"). Os números 1-10 referenciam painéis existentes em `LimiaresTab.tsx` (já construídos, não criados nesta sprint).
- **LK-PS-14:** §5.2.6 (`merge-decision-labels.ts`) consumido por §5.2.3 (Card 1 — coluna "Decisão" da tabela de auditoria) e §5.2.5 (Card 3 — texto dos botões "Manter ambos" / "Unificar").
- **LK-PS-15:** §2.26-§2.29 (4 RPCs B7) consumidos pelos endpoints §4.7.1-§4.7.3 e §4.7.5 (Painel B7 #4 lê view direta, não RPC). §2.30 valida pré-existência de `v_opus_effectiveness` para o Painel B7 #4. Endpoints chamados manualmente em S10 (§7.1.1).
- **LK-PS-16:** §4.8 (endpoints públicos `/api/taxonomy/*`) consumidos pelo `DomainRoleCascade` (§5.3) quando montado em página pública. Endpoints admin §4.6 são consumidos pelo mesmo componente em telas admin. O componente decide qual endpoint chamar via prop `mode='public'|'admin'`.
- **LK-PS-17:** §7.1 + §7.1.1 (evidence post-deploy + smoke tests granulares) rodam após cada SUB-PR conforme cabíveis. Saídas armazenadas em `docs/migrations/sprint-cleanup-v3/evidence-post-deploy.sql.out` para auditoria.

---

## Apêndice — Interfaces previstas para a sprint orchestrator subsequente

Esta cleanup v2.6 deixa 3 pontos de extensão para a sprint orchestrator simétrico subsequente (paralela em fase 1, sequencial em fase 2). **Não há modificação retroativa esperada nesta cleanup** — todas as integrações da próxima sprint são aditivas. Este apêndice documenta as interfaces para que a próxima sessão de spec-writing tenha contexto explícito do que esperar.

### Interface 1 — §3.14 `pipeline-config-tooltips.ts` como fonte canônica das 24 chaves

A sprint orchestrator constrói `IMPACT_SOURCES` (mapeamento chave → fonte de dados para cálculo de impacto) referenciando este arquivo, não duplicando a lista de chaves. Cobertura efetiva permanece 24/26 (excluindo as 2 de sistema `CURATE_PIPELINE_ENABLED` e `QUARANTINE_EXPIRY_DAYS`, que não aparecem na tela `/admin/pipeline-config`). Os nomes canônicos das chaves são autoridade desta cleanup — qualquer divergência futura em IMPACT_SOURCES (chaves inexistentes, nomes incorretos, omissões) é bug a corrigir alinhando à §3.14.

Chaves canônicas das 24 chaves de calibração:

| # | Chave role | Chave skill |
|---|---|---|
| 1 | `role.confidence.lookback_days` | `skill.confidence.lookback_days` |
| 2 | `role.confidence.min_count` | `skill.confidence.min_count` |
| 3 | `role.hard_gate.min_confidence` | `skill.hard_gate.min_confidence` |
| 4 | `role.merge_candidate.cosine_threshold` | `skill.merge_candidate.cosine_threshold` |
| 5 | `role.merge_candidate.lookback_days` | `skill.merge_candidate.lookback_days` |
| 6 | `role.merge_candidate.opus_review_cooldown_days` | `skill.merge_candidate.opus_review_cooldown_days` |
| 7 | `role.opus_review.cooldown_days` | `skill.opus_review.cooldown_days` |
| 8 | `role.promotion.auto_min_confidence` | `skill.promotion.auto_min_confidence` |
| 9 | `role.promotion.lookback_days` | `skill.promotion.lookback_days` |
| 10 | `role.promotion.min_distinct_employers` | `skill.promotion.min_distinct_employers` |
| 11 | `role.promotion.min_vacancies` | `skill.promotion.min_vacancies` |
| 12 | `role.retirement.gap_days` | `skill.retirement.gap_days` |

### Interface 2 — §5.1.6 modal: caixa qualitativa + `ImpactPreview` quantitativo, convivência

A caixa "Impacto estimado" desta cleanup é qualitativa por design (D-PS-33). A sprint orchestrator ADICIONA componente `ImpactPreview` ao mesmo modal: tabela de contagem (items atual vs. proposto, delta) + histograma de distribuição (10 buckets) + seletor de janela [7d / 30d / 90d] dentro do próprio componente. As duas visualizações convivem no mesmo modal e são complementares: qualitativa orienta onde olhar pós-mudança; quantitativa diz quantos itens são afetados pela mudança proposta.

A sprint orchestrator NÃO deve substituir, mover ou modificar a caixa qualitativa desta cleanup. Caso seja necessário reorganizar visualmente o modal para acomodar ambas, isso é responsabilidade da próxima sprint e deve estar explícito na spec dela (não na cleanup).

### Interface 3 — `function_orchestrator_*` evolução completa

A sprint orchestrator simétrico subsequente executa os seguintes ajustes na infraestrutura de orchestrator (em base de pré-produção sem rows históricos relevantes):

1. **Rename de `function_orchestrator_items` → `function_orchestrator_role_items`.** Inclui rename de tabela, indices, triggers (`trg_foi_jcr_confidence_*` → `trg_fori_jcr_confidence_*`), constraints. Funções de trigger (`fn_jcr_confidence_median_insert/update/delete`) NÃO precisam rename — já são agnostic do nome da tabela.

2. **Atualização do corpo de `fn_recompute_jcr_confidence_median`.** Esta função referencia `function_orchestrator_items` por nome no SQL (`FROM function_orchestrator_items foi`). Sem atualizar corpo via `CREATE OR REPLACE FUNCTION`, a função quebra pós-rename. D-PS-49 desta cleanup documenta o circuito completo de `confidence_median`.

3. **Criação de `function_orchestrator_skill_items`** espelhando schema de `function_orchestrator_role_items` semanticamente. Tabela paralela, não substitui nenhuma estrutura existente.

4. **Adição de 5 colunas `skills_*` em `function_orchestrator_runs`:** `skills_extracted`, `skills_reused`, `skills_pending_created`, `skills_gate_rejected`, `skills_failed`. Default 0, NOT NULL.

5. **`canonical_created` e `canonical_promoted` permanecem role-only** (D-PS-50 desta cleanup). Não são renomeados nem ganham equivalentes skill nesta evolução. Promoções de skill ficam visíveis em `events` (`event_name = 'skill_promoted_dynamic'`) e em `job_canonical_skills.promoted_at`.

6. **`fallback_ratio` permanece role-only** (D-PS-51 desta cleanup). Métrica equivalente skill é calculada on-the-fly em queries, sem coluna materializada.

7. **`confidence_median` em ambos os lados permanece intocado** (D-PS-49 desta cleanup). Sprint orchestrator NÃO cria trigger novo em `function_orchestrator_skill_items` para popular `job_canonical_skills.confidence_median` — `fn_jps_recompute_jcs` continua sendo a fonte canônica a partir de `job_posting_skills` curated.

Sequenciamento sugerido (paralelismo controlado β-light): SUB-PRs de criação de tabela + colunas + auditoria são paralelos com cleanup v2.6 (zero overlap de arquivos); SUB-PRs de rename + refator TS + endpoint + integração de UI são sequenciais após cleanup v2.6 fechar.


**Fim da SPEC-sprint-cleanup-v3.1.**
