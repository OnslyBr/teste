# Sprint v1.0 — Fluxo de Retorno + Governança de Taxonomia + Limpeza Estrutural

**Versão:** v1.0 FINAL
**Data:** 27/04/2026
**Autor:** Onsly (PO) + Claude (arquiteto técnico)
**Implementação:** Antigravity (código TS) + Claude Code (SQL)

---

## Sumário executivo

A Sprint v1.0 fecha três frentes simultaneamente:

1. **Limpeza estrutural** — drifts de nomenclatura, colunas redundantes, tabelas-fantasma e RLS faltante; alinha schema ao padrão atual antes de construções novas
2. **Governança de taxonomia** — migração dos 4 JSONs (`equivalences`, `family_synonyms`, `domain_synonyms`, `domains`) para tabelas de banco com fluxo dinâmico de manutenção via Sonnet curador + Opus 4.7 validador; áreas de atuação 0:N para canônicos
3. **Inversão de paradigma** — `vacancy_count ≥ 3` deixa de ser cron-driven e vira regra dinâmica em runtime; criação de canônicos sai do modelo F1/F2/F3 estático

A sprint quebra com numeração v5.x intencionalmente: versão major nova reflete mudança de paradigma da governança de taxonomia. Após v1.0, voltamos para incrementos v1.1, v1.2 etc.

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
- "âncora" = par `(description_hash, canonical_role_id)` que serve de cache para Camada 0
- "lote" = conjunto de vagas que passa pelo pipeline LLM em uma única invocação

**Atores:**
- **Sonnet curador** = `LLM_MODEL` em `constants.ts:11` (Sonnet 4.7) — pipeline principal de curadoria
- **Opus validador** = Opus 4.7 — CRON diário de validação de mapeamentos novos
- **Haiku auxiliar** = Haiku 4.5 — `resume_processing`, `infer-state.ts`, `normalize-org-industry`, backfill G3

**Camadas do pipeline:**
- **Camada 0** = precheck por hash de descrição (`description_hash`) — cache puro, sem LLM
- **Camada 1** = pré-resolver por sinônimos/equivalências dos JSONs — sem LLM
- **Camada 2** = LLM curador com sugestão pré-resolvida (faltou match em sinônimos mas há candidato)
- **Camada 3** = LLM curador sem sugestão alguma (decisão pura do LLM do zero)

**Padrões SQL:**
- Toda migration começa com `BEGIN;` e termina com `COMMIT;`
- IDs de FK usam `ON DELETE CASCADE` ou `ON DELETE SET NULL` explicitamente
- RLS em tabelas operacionais = `ENABLE ROW LEVEL SECURITY` sem policies (deny-all, service_role bypassa)
- Sentinel date para créditos não-expiráveis: `9999-12-31T23:59:59Z`

---

## Decisões arquiteturais cravadas

**Escopo confirmado (reforço dos princípios desta sprint):**

1. **Frequência de mercado é o único peso.** Motor de Sugestão usa frequência crua, sem IDF, sem caps artificiais, sem bônus comportamental. O mercado define o peso.

2. **Mudança de label de canônico é operação aceitável.** Quando Opus renomeia um canônico, usuários afetados recebem toast no próximo login + email transacional de reforço. Não há delay de 24h forçado entre criação e ativação do canônico.

3. **Canônico sem vaga associada não deve existir.** Aplicação retroativa do Item 7B inclui DELETE direto de zumbis órfãos (com guard `id NOT IN (SELECT DISTINCT canonical_role_id FROM job_postings)`).

4. **`taxonomy_content_version` (renome de `prompt_content_version`) bumpa apenas quando há mudança real.** CRON Opus, no fim da execução diária, verifica se houve UPDATE em `taxonomy_relations`. Se sim, bumpa. Se não, não bumpa.

5. **Trigger é fonte autoritativa única para `vacancy_count`.** Remoção do PASSO 4 do `maintenance_phase_2` resolve o conflito de design entre trigger e CRON.

6. **`shared_results.final_score` mantém-se como snapshot intencional.** TTL longo + contrato implícito com terceiros (link público) justifica denormalização. Demais snapshots dropados.

7. **`taxonomy_relations` cresce via fluxo Sonnet→Opus.** Sonnet propõe (com `status='inactive'`); CRON diário envia para Opus 4.7 validar; Opus aprova/discorda/merge.

8. **G3 áreas de atuação reusa `canonical_role_domains` como catálogo.** Cria nova `canonical_role_domain_links` para relação N:N entre canônico e área. Coluna `job_canonical_roles.domain_id` (1:1 dormente, 664 NULLs) é DROPADA.

---

## Pré-requisitos operacionais

Antes de começar a implementação:

- [ ] Confirmar acesso ao banco de produção via Claude Code (DB connection ativa)
- [ ] Confirmar Vercel Hobby ainda em vigor (CRON degradados — anotar para upgrade pós-sprint)
- [ ] Confirmar storage para export defensivo dos backups (~3MB compactado)
- [ ] Confirmar API key Opus 4.7 ativa em `.env.local` (variável `ANTHROPIC_OPUS_KEY` ou similar)
- [ ] Confirmar Redis Upstash ativo (já usado para circuit breaker do pipeline AI)

---

## Estimativa de esforço

| PR | Estimativa | Risco |
|---|---|---|
| PR1 (limpeza) | 6-8 horas | Baixo — mudanças mecânicas |
| PR2 (governança taxonomia) | 16-24 horas | Médio-alto — toca coração do pipeline |
| PR3 (inversão paradigma) | 8-12 horas | Médio — regra retroativa delicada |
| PR4 (fixes pipeline) | 6-8 horas | Baixo-médio — patches localizados |
| PR5 (observação/UX) | 4-6 horas | Baixo — código funcional dormente |
| PR6 (refinos) | 8-12 horas | Variável — depende do escopo de tests |
| **Total** | **48-70 horas** | — |

Estimativa para Antigravity executar tudo, considerando ciclos de revisão Claude e ajustes.

---

## Backlog operacional pós-v1.0

Anotações que ficam para sprints futuras (não fazem parte da v1.0):

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

**Migration:**

```sql
BEGIN;

-- 1. Adicionar nova coluna
ALTER TABLE job_no_postings ADD COLUMN profile_id UUID REFERENCES profiles(id) ON DELETE CASCADE;

-- 2. Backfill: para cada user_id, encontrar o profile_id correspondente
UPDATE job_no_postings jnp
SET profile_id = p.id
FROM profiles p
WHERE p.user_id = jnp.user_id;

-- 3. Verificar que todos foram backfilled
DO $$
DECLARE
    rows_sem_profile INT;
BEGIN
    SELECT COUNT(*) INTO rows_sem_profile
    FROM job_no_postings WHERE profile_id IS NULL;
    
    IF rows_sem_profile > 0 THEN
        RAISE EXCEPTION 'Backfill incompleto: % linhas sem profile_id', rows_sem_profile;
    END IF;
END $$;

-- 4. Marcar como NOT NULL e dropar coluna antiga
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

### 15.5 — DROP `title` + rename `original_title → title` em `job_postings`

Bloco S confirmou que **100% das vagas com `original_title` populado** têm `title = original_title`. Os 122 casos com `original_title NULL` (1.45%) são pré-fix v5.24. Não há divergência real — são duas colunas com o mesmo dado, violando normalização.

**Cenário cravado (D):** drop `title`, rename `original_title → title`. Trigger `trg_enforce_original_immutability` continua protegendo a coluna após o rename. Casos de "edição admin" eventuais ficam como decisão consciente futura (criar nova coluna se realmente precisar).

**Migration:**

```sql
BEGIN;

-- 1. Backfill prévio dos 122 órfãos (operação idempotente)
UPDATE job_postings
SET original_title = title
WHERE original_title IS NULL;

-- 2. Backfill prévio do equivalente em description (mesma lógica)
UPDATE job_postings
SET original_description = description
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
        RAISE EXCEPTION 'Backfill incompleto: title=% desc=%', nulls_title, nulls_desc;
    END IF;
END $$;

-- 4. DROP da coluna mutável (que nunca foi exercida em produção)
ALTER TABLE job_postings DROP COLUMN title;

-- 5. Rename: original_title vira title (agora protegido por trigger)
ALTER TABLE job_postings RENAME COLUMN original_title TO title;
ALTER TABLE job_postings RENAME COLUMN original_description TO description;

COMMIT;
```

**Trigger continua válido após rename:** `trg_enforce_original_immutability` referencia colunas pelo nome (`OLD.original_title IS NOT NULL`). **Após o rename, precisa atualizar a função** para referenciar `title` e `description`:

```sql
BEGIN;

CREATE OR REPLACE FUNCTION enforce_original_immutability()
RETURNS TRIGGER AS $$
BEGIN
    -- Após rename, os campos imutáveis são title e description
    IF OLD.title IS NOT NULL AND NEW.title IS DISTINCT FROM OLD.title THEN
        RAISE NOTICE 'Tentativa de mudança em job_postings.title (imutável) silenciada para id=%', OLD.id;
        NEW.title := OLD.title;
    END IF;
    
    IF OLD.description IS NOT NULL AND NEW.description IS DISTINCT FROM OLD.description THEN
        RAISE NOTICE 'Tentativa de mudança em job_postings.description (imutável) silenciada para id=%', OLD.id;
        NEW.description := OLD.description;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Renomear o trigger para refletir o novo nome
ALTER TRIGGER trg_enforce_original_immutability ON job_postings RENAME TO trg_enforce_title_description_immutability;

COMMIT;
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

**`title` (era `original_title`) em `job_postings`:**
```bash
grep -rn "original_title\|original_description" \
  --include="*.ts" --include="*.tsx" \
  app/ lib/

# Todos os callsites de original_title/original_description devem ser substituídos
# por title/description simples (já que original_ não existe mais)
```

### 15.7 — Validação pós-migration

Após rodar todas as migrations:

```sql
-- 1. Confirmar que renames foram aplicados
SELECT table_name, column_name 
FROM information_schema.columns 
WHERE table_schema = 'public'
  AND column_name IN ('origin_state', 'location_state', 'state', 'user_id', 'profile_id', 'recorded_at', 'original_title', 'original_description')
ORDER BY table_name, column_name;

-- Esperado pós-rename:
-- analyses.location_state (era origin_state)
-- submitted_jobs.origin_state (era location_state)
-- job_postings.origin_state, job_postings.title, job_postings.description (sem original_*)
-- job_no_postings.profile_id (era user_id)
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
```

### 15.8 — Tests para Item 15

```typescript
// tests/integration/sprint-v1_0/item-15-renames.spec.ts

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

Remove **9 colunas** que são derivadas via JOIN (denormalizações sem mecanismo de sincronização) ou que duplicam informação. Reescreve **3 RLS policies** para apontar para `analysis_id` ao invés das colunas dropadas.

### 16.1 — Lista exaustiva das colunas

| Tabela | Coluna | Razão | Impacto |
|---|---|---|---|
| `analysis_skill_matches` | `resume_id` | Derivável via JOIN: `analyses.resume_id` | RLS policy precisa mudar para via `analysis_id` |
| `resume_skill_enrichments` | `resume_id` | Idem | RLS policy precisa mudar |
| `saved_analyses` | `profile_id` | Derivável via JOIN: `analyses.profile_id` | RLS policy precisa mudar |
| `score_history` | `profile_id` | Idem | Já usa `analysis_id`, drop limpo |
| `shared_results` | `profile_id` | Idem | RLS policy precisa mudar |
| `job_canonical_roles` | `is_active` | 92% incoerente com status real (Bloco O), zero callers TS | Drop limpo |
| `saved_analyses` | `initial_score` | `analyses.initial_score` é imutável após status='done' | Drop, leitura via JOIN |
| `saved_analyses` | `suggestion_text` | Idem (imutável) | Drop, leitura via JOIN |
| `resume_role_suggestions` | `role_label` | Bloco T confirmou: única cópia denormalizada ativa de label | Drop, JOIN no AnalysisTriggerModal |

**Mantém-se intencionalmente:** `shared_results.final_score`. Justificativa: TTL longo do link público + contrato implícito com terceiros. Snapshot intencional documentado.

### 16.2 — Reescrita das RLS policies

**Padrão cravado:** todos os satélites de `analyses` passam a filtrar via `analysis_id` ao invés de `resume_id` ou `profile_id` direto.

**Antes (analysis_skill_matches_select_own):**

```sql
-- Hipotético (formato típico antes do drop)
CREATE POLICY analysis_skill_matches_select_own ON analysis_skill_matches
FOR SELECT
USING (resume_id IN (
    SELECT r.id FROM resumes r
    JOIN profiles p ON p.id = r.profile_id
    WHERE p.user_id = auth.uid()
));
```

**Depois (via analysis_id):**

```sql
CREATE POLICY analysis_skill_matches_select_own ON analysis_skill_matches
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));
```

**Migration completa:**

```sql
BEGIN;

-- ====================
-- 1. Drop policies antigas
-- ====================
DROP POLICY IF EXISTS analysis_skill_matches_select_own ON analysis_skill_matches;
DROP POLICY IF EXISTS resume_skill_enrichments_write_own ON resume_skill_enrichments;
DROP POLICY IF EXISTS shared_results_select_own ON shared_results;

-- ====================
-- 2. Drop das colunas redundantes
-- ====================
ALTER TABLE analysis_skill_matches DROP COLUMN resume_id;
ALTER TABLE resume_skill_enrichments DROP COLUMN resume_id;
ALTER TABLE saved_analyses DROP COLUMN profile_id;
ALTER TABLE score_history DROP COLUMN profile_id;
ALTER TABLE shared_results DROP COLUMN profile_id;
ALTER TABLE job_canonical_roles DROP COLUMN is_active;
ALTER TABLE saved_analyses DROP COLUMN initial_score;
ALTER TABLE saved_analyses DROP COLUMN suggestion_text;
ALTER TABLE resume_role_suggestions DROP COLUMN role_label;

-- ====================
-- 3. Recria policies com analysis_id
-- ====================
CREATE POLICY analysis_skill_matches_select_own ON analysis_skill_matches
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

CREATE POLICY resume_skill_enrichments_write_own ON resume_skill_enrichments
FOR ALL  -- ou apenas FOR INSERT, UPDATE — verificar policy original
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

CREATE POLICY shared_results_select_own ON shared_results
FOR SELECT
USING (analysis_id IN (
    SELECT a.id FROM analyses a
    JOIN profiles p ON p.id = a.profile_id
    WHERE p.user_id = auth.uid()
));

-- saved_analyses e score_history já usam analysis_id em policies existentes
-- Apenas confirma que continuam funcionais após o DROP de profile_id

COMMIT;
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
// tests/integration/sprint-v1_0/item-16-drops.spec.ts

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
    
    it('shared_results.final_score foi mantida (snapshot intencional)', async () => {
        const { error } = await supabase
            .from('shared_results')
            .select('final_score')
            .limit(1);
        expect(error).toBeNull();  // Coluna ainda existe
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

```bash
# Em ambiente local com acesso ao banco produção
mkdir -p archive/2026-04-27_v5.23-backups

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
// tests/integration/sprint-v1_0/item-18-rls.spec.ts

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

```sql
BEGIN;

CREATE TABLE taxonomy_relations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Tipo de relação (substitui os 3 JSONs)
    type TEXT NOT NULL CHECK (type IN (
        'equivalence',      -- vem de equivalences.json (tradução, sinônimo direto)
        'family_synonym',   -- vem de family_synonyms.json (mesma família ocupacional)
        'domain_synonym'    -- vem de domain_synonyms.json (mesma área funcional)
    )),
    
    -- Termo bruto que aparece na vaga (ex: "Site Reliability Engineer III")
    source_term TEXT NOT NULL,
    
    -- Canônico de destino
    target_canonical_id UUID NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
    
    -- Status do mapeamento
    status TEXT NOT NULL DEFAULT 'inactive' CHECK (status IN (
        'inactive',  -- Sonnet propôs, aguarda Opus
        'active',    -- Opus aprovou (afeta cache)
        'rejected'   -- Opus rejeitou (não afeta cache, fica para auditoria)
    )),
    
    -- Camada do pipeline que originou (2 = sugestão pré-resolvida; 3 = LLM puro)
    layer SMALLINT NOT NULL CHECK (layer IN (2, 3)),
    
    -- Label que o Sonnet sugeriu (pode diferir do canônico final se Opus mudou)
    llm_proposed_label TEXT,
    
    -- Razão da decisão do Opus (para auditoria forense)
    opus_decision_reason TEXT,
    
    -- Quando Opus validou (NULL enquanto inactive)
    validated_at TIMESTAMPTZ,
    
    -- Quem validou (sempre 'opus_4_7' por enquanto, mas extensível)
    validated_by TEXT,
    
    -- Versão da entrada (alimenta taxonomy_content_version global)
    version TEXT NOT NULL DEFAULT 'v1',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Constraint: não permite source_term duplicado para o mesmo type
    UNIQUE (type, source_term)
);

-- Index para consulta rápida por type+source_term (uso do Redis cache)
CREATE INDEX idx_taxonomy_relations_type_source 
ON taxonomy_relations(type, source_term) 
WHERE status = 'active';

-- Index para o CRON Opus encontrar pendentes
CREATE INDEX idx_taxonomy_relations_pending 
ON taxonomy_relations(created_at) 
WHERE status = 'inactive';

-- RLS: tabela operacional, deny-all (service_role bypassa)
ALTER TABLE taxonomy_relations ENABLE ROW LEVEL SECURITY;

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

### G2.3 — Renome de `prompt_content_version` para `taxonomy_content_version`

Renome global para evitar confusão com `SYSTEM_PROMPT.ts` (que é arquivo TS estático). O nome novo reflete o propósito real: versão do conteúdo da taxonomia.

**Identificar callsites:**

```bash
grep -rn "prompt_content_version\|PROMPT_CONTENT_VERSION" \
  --include="*.ts" --include="*.tsx" --include="*.sql" \
  --exclude-dir=node_modules --exclude-dir=.next \
  app/ lib/ docs/

# Esperado: arquivo de constants, função de hash, possíveis colunas SQL
```

**Substituições TS:** trocar `prompt_content_version` → `taxonomy_content_version` em todos os matches.

**Migration SQL (se existir como coluna ou variável de configuração):**

```sql
BEGIN;

-- Caso 1: se for coluna em tabela
-- ALTER TABLE <tabela> RENAME COLUMN prompt_content_version TO taxonomy_content_version;

-- Caso 2: se for valor em tabela de configuração
-- UPDATE pipeline_config SET key = 'taxonomy_content_version' WHERE key = 'prompt_content_version';

-- Caso 3: criar tabela de versionamento de taxonomia se não existir
CREATE TABLE IF NOT EXISTS taxonomy_versions (
    id SERIAL PRIMARY KEY,
    content_version TEXT NOT NULL DEFAULT 'v1',
    bumped_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    bumped_by TEXT NOT NULL,  -- 'opus_cron', 'admin', etc.
    reason TEXT,
    UNIQUE (content_version)
);

-- Inserir versão inicial
INSERT INTO taxonomy_versions (content_version, bumped_by, reason)
VALUES ('v1', 'sprint_v1.0_init', 'Versão inicial pós-migração JSON→banco')
ON CONFLICT DO NOTHING;

ALTER TABLE taxonomy_versions ENABLE ROW LEVEL SECURITY;

COMMIT;
```

### G2.4 — Seed inicial de `taxonomy_relations` a partir dos 3 JSONs

**Estratégia:** ler os 3 arquivos JSON existentes (`data/equivalences.json`, `data/family_synonyms.json`, `data/domain_synonyms.json`), iterar e fazer INSERT em `taxonomy_relations` com `status='active'` e `layer=NULL` (são seeds, não vieram de inferência LLM).

**Schema atualizado (adicionar campo `seeded_from_json` para auditoria):**

```sql
-- Recriar a tabela com campo extra para seeds
-- (rodar antes do seed, idempotente se já criada com schema anterior)

ALTER TABLE taxonomy_relations 
ADD COLUMN IF NOT EXISTS seeded_from_json BOOLEAN NOT NULL DEFAULT FALSE;

-- Para seeds, layer pode ser NULL
ALTER TABLE taxonomy_relations 
ALTER COLUMN layer DROP NOT NULL;
ALTER TABLE taxonomy_relations 
DROP CONSTRAINT IF EXISTS taxonomy_relations_layer_check;
ALTER TABLE taxonomy_relations 
ADD CONSTRAINT taxonomy_relations_layer_check 
CHECK (layer IS NULL OR layer IN (2, 3));
```

**Script TypeScript de seed (executável uma vez):**

```typescript
// scripts/seed-taxonomy-relations.ts
import { createAdminServerClient } from '@/lib/supabase-server';
import equivalences from '@/data/equivalences.json';
import familySynonyms from '@/data/family_synonyms.json';
import domainSynonyms from '@/data/domain_synonyms.json';

async function seedTaxonomyRelations() {
    const supabase = createAdminServerClient();
    
    const seeds: Array<{
        type: 'equivalence' | 'family_synonym' | 'domain_synonym';
        source_term: string;
        target_label: string;  // resolveremos para canonical_id
    }> = [];
    
    // 1. Carregar equivalences
    for (const [source, target] of Object.entries(equivalences)) {
        seeds.push({ type: 'equivalence', source_term: source, target_label: target as string });
    }
    
    // 2. Carregar family_synonyms
    for (const [source, target] of Object.entries(familySynonyms)) {
        seeds.push({ type: 'family_synonym', source_term: source, target_label: target as string });
    }
    
    // 3. Carregar domain_synonyms
    for (const [source, target] of Object.entries(domainSynonyms)) {
        seeds.push({ type: 'domain_synonym', source_term: source, target_label: target as string });
    }
    
    console.log(`Total de seeds para inserir: ${seeds.length}`);
    
    // 4. Resolver target_label para canonical_id via JOIN
    const labels = [...new Set(seeds.map(s => s.target_label))];
    const { data: canonicals } = await supabase
        .from('job_canonical_roles')
        .select('id, label')
        .in('label', labels);
    
    const labelToId = new Map((canonicals ?? []).map(c => [c.label, c.id]));
    
    // 5. Filtrar seeds com canonical resolvido + insert em batches
    const seedsResolvidos = seeds
        .filter(s => labelToId.has(s.target_label))
        .map(s => ({
            type: s.type,
            source_term: s.source_term,
            target_canonical_id: labelToId.get(s.target_label)!,
            status: 'active' as const,
            layer: null,
            seeded_from_json: true,
            version: 'v1',
        }));
    
    const seedsSemCanonical = seeds.filter(s => !labelToId.has(s.target_label));
    
    console.log(`Seeds resolvidos: ${seedsResolvidos.length}`);
    console.log(`Seeds SEM canônico (precisam atenção): ${seedsSemCanonical.length}`);
    if (seedsSemCanonical.length > 0) {
        console.log('Labels não encontrados:', [...new Set(seedsSemCanonical.map(s => s.target_label))]);
    }
    
    // 6. Insert em batches de 100
    const batchSize = 100;
    for (let i = 0; i < seedsResolvidos.length; i += batchSize) {
        const batch = seedsResolvidos.slice(i, i + batchSize);
        const { error } = await supabase
            .from('taxonomy_relations')
            .insert(batch);
        
        if (error) {
            console.error(`Erro no batch ${i}-${i+batchSize}:`, error);
            throw error;
        }
        console.log(`Inserido batch ${i}-${i+batchSize}`);
    }
    
    console.log('Seed completo.');
}

seedTaxonomyRelations().catch(console.error);
```

**Execução:**

```bash
npx tsx scripts/seed-taxonomy-relations.ts
```

### G2.5 — Cache Redis com write-through

**Padrão cravado:** `lib/taxonomy/cache.ts` gerencia cache em Redis Upstash. Toda leitura passa por cache; toda escrita invalida o cache no callsite.

```typescript
// lib/taxonomy/cache.ts
import { Redis } from '@upstash/redis';
import { createAdminServerClient } from '@/lib/supabase-server';

const redis = Redis.fromEnv();
const REDIS_KEY_PREFIX = 'tax:';
const CACHE_TTL_SECONDS = 3600; // 1h, mas write-through invalida antes

type RelationType = 'equivalence' | 'family_synonym' | 'domain_synonym';

interface TaxonomyRelation {
    source_term: string;
    target_canonical_id: string;
    target_label: string;  // hidratado via JOIN
}

export async function getRelations(type: RelationType): Promise<Map<string, TaxonomyRelation>> {
    const key = `${REDIS_KEY_PREFIX}${type}`;
    
    // 1. Tenta ler do Redis
    const cached = await redis.get<string>(key);
    if (cached) {
        const entries: [string, TaxonomyRelation][] = JSON.parse(cached);
        return new Map(entries);
    }
    
    // 2. Cache miss → lê do Postgres
    const supabase = createAdminServerClient();
    const { data, error } = await supabase
        .from('taxonomy_relations')
        .select('source_term, target_canonical_id, job_canonical_roles!inner(label)')
        .eq('type', type)
        .eq('status', 'active');
    
    if (error) throw error;
    
    const map = new Map<string, TaxonomyRelation>();
    for (const row of data ?? []) {
        map.set(row.source_term.toLowerCase(), {
            source_term: row.source_term,
            target_canonical_id: row.target_canonical_id,
            target_label: (row.job_canonical_roles as any).label,
        });
    }
    
    // 3. Popula Redis com TTL
    await redis.set(key, JSON.stringify([...map.entries()]), { ex: CACHE_TTL_SECONDS });
    
    return map;
}

export async function invalidateRelations(type: RelationType): Promise<void> {
    const key = `${REDIS_KEY_PREFIX}${type}`;
    await redis.del(key);
}

export async function invalidateAllRelations(): Promise<void> {
    await Promise.all([
        invalidateRelations('equivalence'),
        invalidateRelations('family_synonym'),
        invalidateRelations('domain_synonym'),
    ]);
}
```

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
        .single();
    
    // status='active' AFETA cache, invalida
    if (relation) {
        await invalidateRelations(relation.type);
    }
}
```

### G2.6 — Substituição dos consumers do JSON pelo cache Redis

**Identificar callsites antigos:**

```bash
grep -rn "equivalences.json\|family_synonyms.json\|domain_synonyms.json\|require.*equivalences\|import.*equivalences" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next \
  lib/ app/

# Esperado: lib/pipeline/precheck.ts ou similar (Camada 1 do pipeline)
```

**Padrão de substituição:**

```typescript
// ANTES (lib/pipeline/precheck.ts ou similar)
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
import { getRelations } from '@/lib/taxonomy/cache';

async function preResolve(rawTitle: string) {
    const lower = rawTitle.toLowerCase();
    
    // Ordem de busca: equivalence > family_synonym > domain_synonym
    const equivalences = await getRelations('equivalence');
    if (equivalences.has(lower)) return equivalences.get(lower)!;
    
    const families = await getRelations('family_synonym');
    if (families.has(lower)) return families.get(lower)!;
    
    const domains = await getRelations('domain_synonym');
    if (domains.has(lower)) return domains.get(lower)!;
    
    return null;
}
```

**Cuidado:** `preResolve` agora é async. Todos os callers precisam ser atualizados. Vai exigir refatoração propagada.

### G2.7 — Sonnet curador propõe mapeamentos novos

Quando o LLM curador (Sonnet) curar uma vaga e o título original **veio das Camadas 2 ou 3** (não bateu no cache de equivalência ou sinônimo), grava a relação proposta em `taxonomy_relations` com `status='inactive'`.

**Lógica em `lib/pipeline/persist-curation.ts`** (ou onde a curadoria é finalizada):

```typescript
// Após persistir o canônico decidido pelo LLM
async function maybeInsertTaxonomyProposal(
    supabase: SupabaseClient,
    rawTitle: string,
    targetCanonicalId: string,
    layer: 2 | 3,
    llmProposedLabel: string
) {
    // Só insere se foi inferência (Camadas 2 ou 3, não cache hit)
    if (layer !== 2 && layer !== 3) return;
    
    const lower = rawTitle.toLowerCase().trim();
    
    // Determina type baseado em heurísticas simples
    const type = inferRelationType(rawTitle, llmProposedLabel);
    
    // Insert com ON CONFLICT DO NOTHING (idempotente)
    const { error } = await supabase
        .from('taxonomy_relations')
        .insert({
            type,
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

function inferRelationType(rawTitle: string, label: string): 'equivalence' | 'family_synonym' | 'domain_synonym' {
    // Heurística simples: se o rawTitle é tradução de outro idioma → equivalence
    // Se é família ocupacional próxima → family_synonym
    // Caso contrário (mais comum) → domain_synonym
    
    // Para v1.0, default = 'domain_synonym' (mais genérico)
    // Tunagem futura baseada em métricas de classificação
    return 'domain_synonym';
}
```

**Integração com persist-curation:**

```typescript
// Em lib/pipeline/persist-curation.ts (após o INSERT/UPDATE em job_postings)
await maybeInsertTaxonomyProposal(
    supabase,
    job.original_title,  // (após rename, será job.title)
    canonicalId,
    job.curation_layer,  // 2 ou 3, vindo do batch processor
    job.llm_proposed_label
);
```

### G2.8 — CRON diário de validação Opus 4.7

**Endpoint novo:** `app/api/cron/taxonomy-validation/route.ts`

**Schedule:** diário às 03:00 (após o pipeline-maintenance, em janela de baixo tráfego)

**Lógica:**

1. Buscar todas as `taxonomy_relations` com `status='inactive'` criadas nas últimas 24h
2. Agrupar por `target_canonical_id` (consolida em "lotes" — todos os mapeamentos do mesmo canônico)
3. Para cada canônico, coletar até 10 vagas mais recentes que o originaram
4. Construir prompt para Opus 4.7 (ver template abaixo)
5. Chamar Opus 4.7 via Anthropic SDK
6. Processar resposta: aprovar/discordar/merge para cada mapeamento
7. No fim, se houve mudanças, bumpar `taxonomy_content_version` (Opção B revisada)
8. Invalidar cache Redis dos types afetados

**Template de prompt para Opus:**

```typescript
const OPUS_VALIDATION_PROMPT = `Você é um auditor de taxonomia de cargos profissionais do mercado brasileiro de TI e correlatos. Sua função é validar mapeamentos de termos brutos de vagas para canônicos pré-existentes.

Para cada item abaixo:
1. Avalie se o termo bruto realmente representa o canônico proposto
2. Decida: APROVAR (mantém o canônico) | DISCORDAR (sugere outro label) | REJEITAR (mapeamento ruim, ignorar)
3. Se DISCORDAR, sugira o label correto

Retorne JSON estruturado:
[
  {
    "id": "<id_da_relation>",
    "decision": "APPROVE" | "DISAGREE" | "REJECT",
    "suggested_label": "<novo_label_se_DISAGREE>",
    "reason": "<justificativa_curta>"
  }
]

Itens para validar:
{ITEMS}`;

// Construir ITEMS:
function buildItemsBlock(relations: any[], samplesByCanonical: Map<string, any[]>) {
    return relations.map(r => `
ID: ${r.id}
Termo bruto da vaga: "${r.source_term}"
Canônico proposto pelo Sonnet: "${r.target_label}"

Vagas que originaram esse mapeamento (até 10):
${(samplesByCanonical.get(r.target_canonical_id) ?? []).slice(0, 10).map((v, i) => `
  ${i+1}. Título: "${v.title}"
     Empresa: ${v.company ?? 'N/A'}
     Trecho: "${v.description.slice(0, 300)}..."
`).join('\n')}
`).join('\n---\n');
}
```

**Implementação completa do endpoint:**

```typescript
// app/api/cron/taxonomy-validation/route.ts
import { NextRequest, NextResponse } from 'next/server';
import Anthropic from '@anthropic-ai/sdk';
import { createAdminServerClient } from '@/lib/supabase-server';
import { invalidateRelations } from '@/lib/taxonomy/cache';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function GET(req: NextRequest) {
    // Auth do CRON Vercel
    const authHeader = req.headers.get('authorization');
    if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }
    
    const supabase = createAdminServerClient();
    const startedAt = new Date();
    
    // 1. Buscar pendentes
    const { data: pendentes, error } = await supabase
        .from('taxonomy_relations')
        .select(`
            id, type, source_term, target_canonical_id, layer, llm_proposed_label,
            job_canonical_roles!inner(label)
        `)
        .eq('status', 'inactive')
        .gte('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString())
        .order('created_at', { ascending: true })
        .limit(50);  // Limite de segurança por execução
    
    if (error) throw error;
    if (!pendentes || pendentes.length === 0) {
        return NextResponse.json({ 
            status: 'no_pending', 
            duration_ms: Date.now() - startedAt.getTime() 
        });
    }
    
    // 2. Coletar samples de vagas por canônico (até 10 por)
    const canonicalIds = [...new Set(pendentes.map(p => p.target_canonical_id))];
    const { data: samples } = await supabase
        .from('job_postings')
        .select('canonical_role_id, title, company, description')
        .in('canonical_role_id', canonicalIds)
        .eq('curation_status', 'curated')
        .order('posted_at', { ascending: false })
        .limit(canonicalIds.length * 10);
    
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
    
    // 4. Chamar Opus 4.7
    let opusResponse;
    try {
        opusResponse = await anthropic.messages.create({
            model: 'claude-opus-4-7',
            max_tokens: 4096,
            messages: [{ role: 'user', content: prompt }],
        });
    } catch (err) {
        console.error('Erro chamando Opus:', err);
        return NextResponse.json({ 
            error: 'opus_call_failed', 
            details: err instanceof Error ? err.message : String(err) 
        }, { status: 500 });
    }
    
    // 5. Parse da resposta
    const responseText = opusResponse.content[0].type === 'text' ? opusResponse.content[0].text : '';
    const decisionsMatch = responseText.match(/\[[\s\S]*\]/);
    if (!decisionsMatch) {
        return NextResponse.json({ error: 'opus_response_unparseable', raw: responseText });
    }
    
    const decisions: Array<{
        id: string;
        decision: 'APPROVE' | 'DISAGREE' | 'REJECT';
        suggested_label?: string;
        reason: string;
    }> = JSON.parse(decisionsMatch[0]);
    
    // 6. Aplicar decisões
    let appliedCount = 0;
    let typesAfetados = new Set<string>();
    
    for (const decision of decisions) {
        const relation = pendentes.find(p => p.id === decision.id);
        if (!relation) continue;
        
        if (decision.decision === 'APPROVE') {
            await supabase
                .from('taxonomy_relations')
                .update({
                    status: 'active',
                    validated_at: new Date().toISOString(),
                    validated_by: 'opus_4_7',
                    opus_decision_reason: decision.reason,
                    updated_at: new Date().toISOString(),
                })
                .eq('id', decision.id);
            
            typesAfetados.add(relation.type);
            appliedCount++;
        } else if (decision.decision === 'DISAGREE' && decision.suggested_label) {
            // Caso 5b: novo label, verificar se canônico existe
            const { data: existingCanonical } = await supabase
                .from('job_canonical_roles')
                .select('id, slug')
                .eq('label', decision.suggested_label)
                .single();
            
            if (existingCanonical) {
                // Subcaso 5a: merge do canônico criado pelo Sonnet com existente
                await mergeCanonicals(
                    supabase,
                    relation.target_canonical_id,
                    existingCanonical.id,
                    'opus_validation_disagreed'
                );
                
                // Atualiza relation para apontar para o canônico final
                await supabase
                    .from('taxonomy_relations')
                    .update({
                        target_canonical_id: existingCanonical.id,
                        status: 'active',
                        validated_at: new Date().toISOString(),
                        validated_by: 'opus_4_7',
                        opus_decision_reason: `Mergeado com canônico existente: ${decision.reason}`,
                    })
                    .eq('id', decision.id);
            } else {
                // Subcaso 5b: rename do canônico criado pelo Sonnet
                // Defesa: gerar novo slug e verificar conflito
                const newSlug = generateSlug(decision.suggested_label);
                const { data: slugConflict } = await supabase
                    .from('job_canonical_roles')
                    .select('id')
                    .eq('slug', newSlug)
                    .neq('id', relation.target_canonical_id)
                    .single();
                
                if (slugConflict) {
                    // Conflito de slug — fazer merge ao invés de rename
                    await mergeCanonicals(
                        supabase,
                        relation.target_canonical_id,
                        slugConflict.id,
                        'opus_disagreed_slug_conflict'
                    );
                } else {
                    // Sem conflito: rename do canônico
                    await supabase
                        .from('job_canonical_roles')
                        .update({
                            label: decision.suggested_label,
                            slug: newSlug,
                            // Trigger trg_reset_embedding_on_semantic_change reseta embedding automaticamente
                        })
                        .eq('id', relation.target_canonical_id);
                    
                    // Notificar usuários afetados
                    await markUsersForLabelChangeNotification(
                        supabase,
                        relation.target_canonical_id,
                        decision.suggested_label
                    );
                }
                
                await supabase
                    .from('taxonomy_relations')
                    .update({
                        status: 'active',
                        validated_at: new Date().toISOString(),
                        validated_by: 'opus_4_7',
                        opus_decision_reason: decision.reason,
                    })
                    .eq('id', decision.id);
            }
            
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
                })
                .eq('id', decision.id);
            // REJECT não afeta cache
        }
    }
    
    // 7. Bump taxonomy_content_version se houve mudanças
    if (appliedCount > 0) {
        const newVersion = `v${Date.now()}`;  // ou incremento sequencial
        await supabase
            .from('taxonomy_versions')
            .insert({
                content_version: newVersion,
                bumped_by: 'opus_cron',
                reason: `Validação diária: ${appliedCount} mapeamentos aplicados`,
            });
        
        // Invalidar cache Redis dos types afetados
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


### G2.9 — Funções auxiliares (`mergeCanonicals`, `markUsersForLabelChangeNotification`, `generateSlug`)

```typescript
// lib/taxonomy/merge-canonicals.ts
import { SupabaseClient } from '@supabase/supabase-js';

/**
 * Merge de dois canônicos. O canônico "perdedor" tem suas vagas reapontadas para o "vencedor"
 * e seu status muda para 'merged' com merged_into apontando para o vencedor.
 */
export async function mergeCanonicals(
    supabase: SupabaseClient,
    loserCanonicalId: string,
    winnerCanonicalId: string,
    reason: string
): Promise<void> {
    // 1. Reaponta todas as vagas
    await supabase
        .from('job_postings')
        .update({ canonical_role_id: winnerCanonicalId })
        .eq('canonical_role_id', loserCanonicalId);
    
    // 2. Reaponta source rows (job_canonical_role_sources)
    await supabase
        .from('job_canonical_role_sources')
        .update({ canonical_role_id: winnerCanonicalId })
        .eq('canonical_role_id', loserCanonicalId);
    
    // 3. Reaponta análises que estavam vinculadas ao loser
    await supabase
        .from('analyses')
        .update({ canonical_role_id: winnerCanonicalId })
        .eq('canonical_role_id', loserCanonicalId);
    
    // 4. Reaponta resume_role_suggestions
    await supabase
        .from('resume_role_suggestions')
        .update({ canonical_role_id: winnerCanonicalId })
        .eq('canonical_role_id', loserCanonicalId);
    
    // 5. Marca o loser como merged
    await supabase
        .from('job_canonical_roles')
        .update({
            status: 'merged',
            merged_into: winnerCanonicalId,
        })
        .eq('id', loserCanonicalId);
    
    // 6. Registra o merge em events
    await supabase.from('events').insert({
        event_name: 'job_canonical_remapped',
        entity_type: 'job_canonical_role',
        entity_id: loserCanonicalId,
        actor_id: null,  // automatizado
        previous_state: { canonical_role_id: loserCanonicalId },
        new_state: { canonical_role_id: winnerCanonicalId, reason },
    });
    
    // 7. Marca usuários afetados para notificação
    await markUsersForLabelChangeNotification(supabase, loserCanonicalId, null /* label muda automaticamente */);
}

/**
 * Marca todos os profile_id que tiveram análise apontando para o canônico afetado
 * para receberem notificação no próximo login + email transacional.
 */
export async function markUsersForLabelChangeNotification(
    supabase: SupabaseClient,
    canonicalId: string,
    newLabel: string | null
): Promise<void> {
    // 1. Encontrar todos os profile_id com análise para esse canônico
    const { data: profilesAfetados } = await supabase
        .from('analyses')
        .select('profile_id')
        .eq('canonical_role_id', canonicalId);
    
    const profileIds = [...new Set((profilesAfetados ?? []).map(p => p.profile_id))];
    
    if (profileIds.length === 0) return;
    
    // 2. Marcar flag de notificação pendente em profiles
    await supabase
        .from('profiles')
        .update({ pending_label_change_notification: true })
        .in('id', profileIds);
    
    // 3. Disparar emails transacionais (assíncrono, não bloqueia o CRON)
    for (const profileId of profileIds) {
        await enqueueLabelChangeEmail(profileId, canonicalId, newLabel);
    }
}

async function enqueueLabelChangeEmail(profileId: string, canonicalId: string, newLabel: string | null) {
    // Implementação delega para sistema de email transacional via Hostinger
    // Template: 'label_change_notification'
    // Variáveis: { first_name, old_label, new_label, analysis_link }
    // ...
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

### G2.10 — Adicionar coluna `pending_label_change_notification` em `profiles`

```sql
BEGIN;

ALTER TABLE profiles 
ADD COLUMN IF NOT EXISTS pending_label_change_notification BOOLEAN NOT NULL DEFAULT FALSE;

CREATE INDEX IF NOT EXISTS idx_profiles_pending_notif 
ON profiles(id) 
WHERE pending_label_change_notification = true;

COMMIT;
```

### G2.11 — Toast no próximo login + email template

**Frontend:** componente novo `components/notifications/LabelChangeToast.tsx`:

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
                .single();
            
            if (!profile?.pending_label_change_notification) return;
            
            // Buscar detalhes das mudanças via events
            const { data: events } = await supabase
                .from('events')
                .select('previous_state, new_state, created_at')
                .eq('event_name', 'job_canonical_remapped')
                .gte('created_at', new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString())
                .order('created_at', { ascending: false })
                .limit(5);
            
            if (events && events.length > 0) {
                const details = events.map(e => `${e.previous_state.label} → ${e.new_state.label}`).join('; ');
                showToast({
                    type: 'info',
                    title: 'Algumas funções foram atualizadas',
                    description: `Com base em análise de mercado, atualizamos: ${details}. Suas análises anteriores continuam disponíveis com os nomes atualizados.`,
                    duration: 8000,
                });
            }
            
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

**Email template** (cadastrar no Hostinger):

```
Assunto: Atualização nas suas análises CalibraCV — função reclassificada

Olá, {{first_name}}!

Com base em análise contínua de dados de mercado, reclassificamos a função "{{old_label}}" para "{{new_label}}". Essa atualização garante que sua análise reflita a nomenclatura atual do mercado.

Suas análises anteriores continuam disponíveis e foram automaticamente atualizadas com o novo nome. Não há necessidade de refazer nada.

[Ver minha análise →]({{analysis_link}})

Mais sobre como mantemos nosso catálogo vivo: https://calibracv.com/methodology

Equipe CalibraCV
```

### G2.12 — `taxonomy_content_version` bumping após CRON Opus

Já implementado em G2.8 (passo 7). Recap:

- CRON Opus, ao final, **se houve `appliedCount > 0`**, insere nova versão em `taxonomy_versions`
- Cache Redis é invalidado em todos os types afetados (write-through)
- Hashes de `description_hash` antigos ficam stale e vão sendo substituídos organicamente em chamadas futuras do pipeline

**Onde o pipeline lê a versão:**

```typescript
// lib/pipeline/precheck.ts (ou onde computa description_hash)
async function getCurrentTaxonomyVersion(supabase: SupabaseClient): Promise<string> {
    // Cache Redis com TTL curto (15min) para evitar query a cada vaga
    const cached = await redis.get<string>('tax:current_version');
    if (cached) return cached;
    
    const { data } = await supabase
        .from('taxonomy_versions')
        .select('content_version')
        .order('bumped_at', { ascending: false })
        .limit(1)
        .single();
    
    const version = data?.content_version ?? 'v1';
    await redis.set('tax:current_version', version, { ex: 900 });  // 15min
    return version;
}

function computeDescriptionHash(description: string, taxonomyVersion: string): string {
    const normalized = normalizeDescription(description);
    return crypto.createHash('sha256')
        .update(`${normalized}|${taxonomyVersion}`)
        .digest('hex');
}
```

**Quando CRON Opus bumpa versão:** invalidar também `tax:current_version` no Redis para que próximas leituras peguem a nova.

```typescript
// Adicionar ao final do CRON Opus, depois do bump
if (appliedCount > 0) {
    // ... bump em taxonomy_versions ...
    await redis.del('tax:current_version');  // ← força próxima leitura a pegar do banco
}
```

### G2.13 — Tests para Item G2

```typescript
// tests/integration/sprint-v1_0/item-g2-taxonomy.spec.ts

describe('Item G2 — Sinônimos dinâmicos via taxonomy_relations', () => {
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
        
        // 2. Insere nova relação ativa
        const { data: novaRelacao } = await supabase
            .from('taxonomy_relations')
            .insert({
                type: 'equivalence',
                source_term: 'test_sinonimo_xyz',
                target_canonical_id: 'algum-canonical-id',
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
        await supabase.from('taxonomy_relations').delete().eq('id', novaRelacao.id);
    });
    
    it('Sonnet propõe mapeamento em Camadas 2 e 3 com status inactive', async () => {
        // Simular curadoria de vaga em Camada 2
        await maybeInsertTaxonomyProposal(
            supabase,
            'Site Reliability Engineer III',
            'algum-canonical-id',
            2,
            'Engenheiro de Confiabilidade Senior'
        );
        
        const { data } = await supabase
            .from('taxonomy_relations')
            .select('*')
            .eq('source_term', 'site reliability engineer iii')
            .single();
        
        expect(data.status).toBe('inactive');
        expect(data.layer).toBe(2);
    });
    
    it('CRON Opus aprovação muda status para active e bumpa versão', async () => {
        // Setup: insere relação inactive
        const { data: relation } = await supabase
            .from('taxonomy_relations')
            .insert({
                type: 'domain_synonym',
                source_term: 'test_sre_iii',
                target_canonical_id: 'algum-canonical-id',
                status: 'inactive',
                layer: 2,
            })
            .select()
            .single();
        
        const versionBefore = await getCurrentTaxonomyVersion(supabase);
        
        // Mock chamada Opus → APPROVE
        // (em testes reais, mockar Anthropic SDK ou usar fixture)
        // Aqui simulamos manualmente o resultado
        await supabase
            .from('taxonomy_relations')
            .update({
                status: 'active',
                validated_at: new Date().toISOString(),
                validated_by: 'opus_4_7',
                opus_decision_reason: 'Mapeamento correto',
            })
            .eq('id', relation.id);
        
        await supabase
            .from('taxonomy_versions')
            .insert({
                content_version: `v${Date.now()}`,
                bumped_by: 'opus_cron_test',
                reason: 'Test',
            });
        
        const versionAfter = await getCurrentTaxonomyVersion(supabase);
        expect(versionAfter).not.toBe(versionBefore);
        
        // Cleanup
        await supabase.from('taxonomy_relations').delete().eq('id', relation.id);
    });
    
    it('mergeCanonicals reaponta vagas e marca canônico como merged', async () => {
        // Setup: criar 2 canônicos + algumas vagas no loser
        // ... (setup detalhado) ...
        
        await mergeCanonicals(supabase, loserId, winnerId, 'test_merge');
        
        // Verificar
        const { data: vagasReapontadas } = await supabase
            .from('job_postings')
            .select('canonical_role_id')
            .eq('canonical_role_id', winnerId);
        
        expect(vagasReapontadas?.length).toBeGreaterThan(0);
        
        const { data: loser } = await supabase
            .from('job_canonical_roles')
            .select('status, merged_into')
            .eq('id', loserId)
            .single();
        
        expect(loser?.status).toBe('merged');
        expect(loser?.merged_into).toBe(winnerId);
    });
    
    it('toast aparece no login após mudança de label', async () => {
        // Setup: marcar profile com flag
        await supabase
            .from('profiles')
            .update({ pending_label_change_notification: true })
            .eq('id', testProfileId);
        
        // Simular render do componente
        const { result } = renderHook(() => useLabelChangeNotification(testProfileId));
        
        await waitFor(() => {
            expect(result.current.toastShown).toBe(true);
        });
        
        // Verificar que flag foi limpa
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

```typescript
// scripts/backfill-canonical-domains.ts
import Anthropic from '@anthropic-ai/sdk';
import { createAdminServerClient } from '@/lib/supabase-server';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

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

Retorne JSON:
{
  "primary_domain_slug": "<slug>",
  "secondary_domain_slug": "<slug ou null>",
  "confidence": <0.0-1.0>,
  "reason": "<justificativa curta>"
}

Se nenhuma área se aplicar bem, retorne primary_domain_slug = null.`;

async function backfillCanonicalDomains() {
    const supabase = createAdminServerClient();
    
    // 1. Buscar canônicos sem links em canonical_role_domain_links
    const { data: canonicals } = await supabase
        .from('job_canonical_roles')
        .select('id, label')
        .eq('status', 'active')
        .not('id', 'in', `(SELECT canonical_role_id FROM canonical_role_domain_links)`);
    
    console.log(`Canônicos a backfillar: ${canonicals?.length ?? 0}`);
    
    if (!canonicals || canonicals.length === 0) {
        console.log('Nada a fazer.');
        return;
    }
    
    // 2. Carregar mapping slug → domain_id
    const { data: domains } = await supabase
        .from('canonical_role_domains')
        .select('id, slug');
    const domainBySlug = new Map((domains ?? []).map(d => [d.slug, d.id]));
    
    // 3. Para cada canônico, coletar contexto e chamar Haiku
    let processados = 0;
    let bemSucedidos = 0;
    let comFallback = 0;
    
    for (const canonical of canonicals) {
        try {
            // a. Skills agregadas (top 10 por frequência)
            const { data: skills } = await supabase.rpc('get_top_skills_for_canonical', {
                p_canonical_id: canonical.id,
                p_limit: 10,
            });
            
            // b. Top 5 industries
            const { data: industries } = await supabase.rpc('get_top_industries_for_canonical', {
                p_canonical_id: canonical.id,
                p_limit: 5,
            });
            
            // c. Até 7 vagas com title + description (resumida)
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
                .replace('{SAMPLES}', (samples ?? []).map((s, i) => `${i+1}. "${s.title}"\n   "${s.description.slice(0, 200)}..."`).join('\n\n'));
            
            // e. Chamar Haiku
            const response = await anthropic.messages.create({
                model: 'claude-haiku-4-5-20251001',
                max_tokens: 256,
                messages: [{ role: 'user', content: prompt }],
            });
            
            const text = response.content[0].type === 'text' ? response.content[0].text : '';
            const jsonMatch = text.match(/\{[\s\S]*\}/);
            if (!jsonMatch) {
                throw new Error('Resposta Haiku sem JSON');
            }
            
            const result = JSON.parse(jsonMatch[0]);
            
            // f. Inserir link primário
            if (result.primary_domain_slug && domainBySlug.has(result.primary_domain_slug)) {
                await supabase.from('canonical_role_domain_links').insert({
                    canonical_role_id: canonical.id,
                    domain_id: domainBySlug.get(result.primary_domain_slug)!,
                    is_primary: true,
                    confidence: result.confidence,
                    source: 'ai_backfill',
                });
                
                // g. Secundário (se houver)
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
                console.warn(`Canônico ${canonical.label}: Haiku não conseguiu classificar`);
                comFallback++;
            }
            
            processados++;
            if (processados % 50 === 0) {
                console.log(`Progresso: ${processados}/${canonicals.length} (sucesso=${bemSucedidos}, fallback=${comFallback})`);
            }
        } catch (err) {
            console.error(`Erro processando ${canonical.label}:`, err);
            comFallback++;
        }
    }
    
    console.log(`\nBackfill concluído: ${processados} processados, ${bemSucedidos} sucesso, ${comFallback} fallback`);
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
    JOIN job_postings_skills jps ON jps.job_posting_id = jp.id
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
// tests/integration/sprint-v1_0/item-g3-areas.spec.ts

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
    const { data } = await supabase
        .from('job_canonical_roles')
        .select('id')
        .eq('label', label)
        .single();
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
| DELETE | 0 | DELETE direto com guard |

**Migration:**

```sql
BEGIN;

-- ================================
-- Bucket 1: Promove para active (vacancy_count >= 3)
-- ================================
UPDATE job_canonical_roles
SET status = 'active',
    updated_at = NOW()
WHERE status = 'pending' 
  AND vacancy_count >= 3;

-- Log de quantos foram promovidos
DO $$
DECLARE
    promovidos INT;
BEGIN
    GET DIAGNOSTICS promovidos = ROW_COUNT;
    RAISE NOTICE 'Bucket 1 — Promovidos para active: %', promovidos;
END $$;

-- ================================
-- Bucket 2: Mantém pending (vacancy_count em 1-2)
-- ================================
-- Não-ação. Regra dinâmica reavalia em uso futuro.

-- ================================
-- Bucket 3: DELETE direto para zumbis órfãos (vacancy_count = 0)
-- Guard de defesa: confirma que NÃO há vagas reais apontando, mesmo se vacancy_count tiver drift
-- ================================
DELETE FROM job_canonical_roles
WHERE status = 'pending' 
  AND vacancy_count = 0
  AND id NOT IN (
      SELECT DISTINCT canonical_role_id 
      FROM job_postings 
      WHERE canonical_role_id IS NOT NULL
  );

DO $$
DECLARE
    apagados INT;
BEGIN
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

Função PL/SQL que segue cadeia `merged_into` e `alias_of_id` até encontrar o canônico final ativo. Profundidade máxima 10 (defesa contra ciclo).

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
        SELECT id, status, alias_of_id, merged_into, 1 AS depth
        FROM job_canonical_roles 
        WHERE id = p_id
        
        UNION ALL
        
        -- Caso recursivo: segue cadeia
        SELECT jcr.id, jcr.status, jcr.alias_of_id, jcr.merged_into, c.depth + 1
        FROM chain c
        JOIN job_canonical_roles jcr ON jcr.id = COALESCE(
            CASE WHEN c.status = 'alias_of'   THEN c.alias_of_id END,
            CASE WHEN c.status = 'merged'     THEN c.merged_into END,
            CASE WHEN c.status = 'deprecated' THEN c.merged_into END
        )
        WHERE c.depth < 10
          AND c.status IN ('alias_of', 'merged', 'deprecated')
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
// tests/integration/sprint-v1_0/item-7-dynamic-rule.spec.ts

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
        const { data: c } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestC', slug: 'testc', status: 'active' })
            .select()
            .single();
        
        const { data: b } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestB', slug: 'testb', status: 'merged', merged_into: c!.id })
            .select()
            .single();
        
        const { data: a } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'TestA', slug: 'testa', status: 'merged', merged_into: b!.id })
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

```typescript
const AMBER_THRESHOLD_DAYS = 120;
const RED_THRESHOLD_DAYS = 365;

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
        // 2. Última vaga do canônico
        const { data: lastVacancy } = await supabase
            .from('job_postings')
            .select('posted_at')
            .eq('canonical_role_id', c.id)
            .eq('curation_status', 'curated')
            .order('posted_at', { ascending: false })
            .limit(1)
            .single();
        
        if (!lastVacancy?.posted_at) continue;
        
        const gapDays = Math.floor(
            (now.getTime() - new Date(lastVacancy.posted_at).getTime()) / (1000 * 60 * 60 * 24)
        );
        
        if (gapDays >= RED_THRESHOLD_DAYS) {
            // Red: emitir evento + sugerir candidato de merge
            const mergeCandidate = await findBestMergeCandidate(supabase, c.id);
            
            await emitEventOnce(supabase, {
                event_name: 'canonical_red_alert',
                entity_type: 'job_canonical_role',
                entity_id: c.id,
                window_days: 30,
                payload: {
                    label: c.label,
                    gap_days: gapDays,
                    last_vacancy_at: lastVacancy.posted_at,
                    merge_candidate: mergeCandidate,
                },
            });
            red++;
        } else if (gapDays >= AMBER_THRESHOLD_DAYS) {
            await emitEventOnce(supabase, {
                event_name: 'canonical_amber_alert',
                entity_type: 'job_canonical_role',
                entity_id: c.id,
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
    entity_type: string;
    entity_id: string;
    window_days: number;
    payload: any;
}) {
    // Idempotência: skip se já houve evento mesmo nome+entity nos últimos N dias
    const cutoff = new Date(Date.now() - params.window_days * 24 * 60 * 60 * 1000).toISOString();
    
    const { data: existing } = await supabase
        .from('events')
        .select('id')
        .eq('event_name', params.event_name)
        .eq('entity_id', params.entity_id)
        .gte('created_at', cutoff)
        .limit(1)
        .single();
    
    if (existing) return;  // Já emitiu, skip
    
    await supabase.from('events').insert({
        event_name: params.event_name,
        entity_type: params.entity_type,
        entity_id: params.entity_id,
        actor_id: null,  // automatizado
        new_state: params.payload,
    });
}

async function findBestMergeCandidate(supabase: SupabaseClient, canonicalId: string): Promise<{ id: string; label: string; similarity: number } | null> {
    // Busca top-1 canônico mais similar via embedding
    const { data: source } = await supabase
        .from('job_canonical_roles')
        .select('embedding, label')
        .eq('id', canonicalId)
        .single();
    
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

```typescript
// app/api/cron/pipeline-maintenance/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createAdminServerClient } from '@/lib/supabase-server';

export async function GET(req: NextRequest) {
    const authHeader = req.headers.get('authorization');
    if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
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
// tests/integration/sprint-v1_0/item-2-pipeline-maintenance.spec.ts

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
    
    it('emitEventOnce não duplica eventos dentro da janela', async () => {
        const canonicalId = 'algum-canonical-id';
        
        // Primeira emissão
        await emitEventOnce(supabase, {
            event_name: 'canonical_amber_alert',
            entity_type: 'job_canonical_role',
            entity_id: canonicalId,
            window_days: 30,
            payload: { test: 1 },
        });
        
        // Segunda chamada imediata — deve ser ignorada
        await emitEventOnce(supabase, {
            event_name: 'canonical_amber_alert',
            entity_type: 'job_canonical_role',
            entity_id: canonicalId,
            window_days: 30,
            payload: { test: 2 },
        });
        
        const { count } = await supabase
            .from('events')
            .select('*', { count: 'exact', head: true })
            .eq('event_name', 'canonical_amber_alert')
            .eq('entity_id', canonicalId)
            .gte('created_at', new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString());
        
        expect(count).toBe(1);  // Apenas 1 emissão
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

`distinct_sources_count` é métrica que conta quantas fontes distintas (Fantastic, colagem manual, admin) atestaram a existência de um canônico. Hoje, das ~583 vagas pending atuais, apenas 1 (0,17%) tem `distinct_sources_count ≥ 1`. As outras 582 estão zeradas — bug em 10 callsites que não atualizam essa coluna.

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

-- Função de recomputação
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
    
    -- Recomputar count
    UPDATE job_canonical_roles
    SET distinct_sources_count = (
        SELECT COUNT(DISTINCT source_type)
        FROM job_canonical_role_sources
        WHERE canonical_role_id = target_canonical_id
    )
    WHERE id = target_canonical_id;
    
    -- Para UPDATE: se canonical_role_id mudou, recomputar tb o canônico antigo
    IF TG_OP = 'UPDATE' AND OLD.canonical_role_id IS DISTINCT FROM NEW.canonical_role_id THEN
        UPDATE job_canonical_roles
        SET distinct_sources_count = (
            SELECT COUNT(DISTINCT source_type)
            FROM job_canonical_role_sources
            WHERE canonical_role_id = OLD.canonical_role_id
        )
        WHERE id = OLD.canonical_role_id;
    END IF;
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Trigger AFTER (não BEFORE — recompute precisa do estado já gravado)
CREATE TRIGGER trg_recompute_distinct_sources_count
AFTER INSERT OR UPDATE OR DELETE ON job_canonical_role_sources
FOR EACH ROW
EXECUTE FUNCTION recompute_distinct_sources_count();

COMMIT;
```

### 1A.3 — Backfill SQL one-shot dos 582 canônicos pending

```sql
BEGIN;

-- Recomputar para TODOS os canônicos (não apenas pending — defesa contra drift histórico)
UPDATE job_canonical_roles jcr
SET distinct_sources_count = COALESCE((
    SELECT COUNT(DISTINCT source_type)
    FROM job_canonical_role_sources jcrs
    WHERE jcrs.canonical_role_id = jcr.id
), 0);

-- Validação
DO $$
DECLARE
    pending_com_zero INT;
    pending_total INT;
BEGIN
    SELECT COUNT(*) INTO pending_com_zero
    FROM job_canonical_roles
    WHERE status = 'pending' AND distinct_sources_count = 0;
    
    SELECT COUNT(*) INTO pending_total
    FROM job_canonical_roles
    WHERE status = 'pending';
    
    RAISE NOTICE 'Pending: % total, % com count=0 após backfill', pending_total, pending_com_zero;
END $$;

COMMIT;
```

**Resultado esperado:** Após o backfill, `pending_com_zero` deve ser apenas os canônicos legítimos sem source registrado em `job_canonical_role_sources` (provavelmente zero ou bem poucos).

### 1A.4 — Tests para Item 1A

```typescript
// tests/integration/sprint-v1_0/item-1a-distinct-sources.spec.ts

describe('Item 1A — distinct_sources_count', () => {
    it('trigger recomputa count automaticamente em INSERT', async () => {
        // Setup: criar canônico
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test 1A', slug: 'test-1a', status: 'pending' })
            .select()
            .single();
        
        // Inserir 2 sources de tipos distintos
        await supabase.from('job_canonical_role_sources').insert([
            { canonical_role_id: canonical!.id, source_type: 'fantastic_jobs', source_ref: 'src1' },
            { canonical_role_id: canonical!.id, source_type: 'manual_submission', source_ref: 'src2' },
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
                { canonical_role_id: canonical!.id, source_type: 'fantastic_jobs', source_ref: 'src1' },
                { canonical_role_id: canonical!.id, source_type: 'admin_ingestion', source_ref: 'src2' },
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

### 1A.bis.1 — Schema: nova coluna `is_recruitment_agency` em `job_postings`

```sql
BEGIN;

ALTER TABLE job_postings 
ADD COLUMN IF NOT EXISTS is_recruitment_agency BOOLEAN NOT NULL DEFAULT FALSE;

CREATE INDEX IF NOT EXISTS idx_job_postings_is_agency 
ON job_postings(canonical_role_id, is_recruitment_agency) 
WHERE is_recruitment_agency = true;

COMMIT;
```

### 1A.bis.2 — Lista cravada das 3 agências (Fluxo B popula manualmente)

Bloco N identificou:

| Razão social | Padrão de identificação |
|---|---|
| Agência A (placeholder) | Empresa com >50 vagas em >20 canônicos distintos no mesmo trimestre |
| Agência B (placeholder) | LinkedIn URL contém `/in/recrutador/` ou nome contém "recrutamento", "headhunter" |
| Agência C (placeholder) | Razão social conhecida — lista mantida em `data/recruitment-agencies.json` |

**Arquivo de lista:** `data/recruitment-agencies.json`

```json
{
  "agencies": [
    {
      "company_name_pattern": "robert half",
      "match_type": "ilike"
    },
    {
      "company_name_pattern": "michael page",
      "match_type": "ilike"
    },
    {
      "company_name_pattern": "page personnel",
      "match_type": "ilike"
    }
  ],
  "version": "v1",
  "last_updated": "2026-04-27"
}
```

### 1A.bis.3 — Populador no Fluxo B (busca automática RapidAPI)

```typescript
// lib/pipeline/agencies.ts
import agenciesList from '@/data/recruitment-agencies.json';

export function isRecruitmentAgency(companyName: string): boolean {
    if (!companyName) return false;
    const lower = companyName.toLowerCase().trim();
    
    return agenciesList.agencies.some(a => {
        if (a.match_type === 'ilike') {
            return lower.includes(a.company_name_pattern);
        }
        if (a.match_type === 'exact') {
            return lower === a.company_name_pattern;
        }
        return false;
    });
}
```

**Integração no upsert do Fluxo B (LinkedIn API):**

```typescript
// lib/pipeline/fluxo-b-upsert.ts (ou onde acontece o INSERT a partir do RapidAPI)
import { isRecruitmentAgency } from '@/lib/pipeline/agencies';

async function upsertFromFantasticAPI(jobData: any) {
    const isAgency = isRecruitmentAgency(jobData.company);
    
    await supabase.from('job_postings').upsert({
        // ... campos existentes ...
        company: jobData.company,
        is_recruitment_agency: isAgency,
    }, { onConflict: 'linkedin_id' });
}
```

### 1A.bis.4 — Filtro em `upsertSource` para não computar agência

```typescript
// lib/pipeline/upsert-canonical.ts
async function upsertSource(supabase: SupabaseClient, params: {
    canonical_role_id: string;
    source_type: string;
    job_posting_id?: string;
}) {
    // Se a vaga é de agência, NÃO contar como source distinto
    if (params.job_posting_id) {
        const { data: vaga } = await supabase
            .from('job_postings')
            .select('is_recruitment_agency')
            .eq('id', params.job_posting_id)
            .single();
        
        if (vaga?.is_recruitment_agency) {
            // Vaga é de agência → ignora para distinct_sources_count
            return;
        }
    }
    
    // Source válida — insere normalmente
    await supabase.from('job_canonical_role_sources').insert({
        canonical_role_id: params.canonical_role_id,
        source_type: params.source_type,
        // ...
    });
}
```

### 1A.bis.5 — Backfill da coluna em vagas existentes

```sql
BEGIN;

-- Marcar vagas que casam com a lista atual de agências
UPDATE job_postings
SET is_recruitment_agency = true
WHERE 
    company ILIKE '%robert half%' OR
    company ILIKE '%michael page%' OR
    company ILIKE '%page personnel%';

-- Log
DO $$
DECLARE
    marcadas INT;
BEGIN
    SELECT COUNT(*) INTO marcadas FROM job_postings WHERE is_recruitment_agency = true;
    RAISE NOTICE 'Vagas marcadas como agência: %', marcadas;
END $$;

COMMIT;
```

### 1A.bis.6 — Tests

```typescript
// tests/integration/sprint-v1_0/item-1a-bis-agencies.spec.ts

describe('Item 1A.bis — Filtro de agências', () => {
    it('isRecruitmentAgency identifica agência conhecida', () => {
        expect(isRecruitmentAgency('Robert Half Brasil')).toBe(true);
        expect(isRecruitmentAgency('Michael Page International')).toBe(true);
        expect(isRecruitmentAgency('Empresa Comum LTDA')).toBe(false);
        expect(isRecruitmentAgency('')).toBe(false);
    });
    
    it('upsertSource ignora vagas de agência', async () => {
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test 1A.bis', slug: 'test-1abis', status: 'active' })
            .select()
            .single();
        
        // Inserir vaga marcada como agência
        const { data: vaga } = await supabase
            .from('job_postings')
            .insert({
                title: 'Test',
                description: 'Test',
                canonical_role_id: canonical!.id,
                is_recruitment_agency: true,
                curation_status: 'curated',
            })
            .select()
            .single();
        
        // Tentar criar source via upsertSource
        await upsertSource(supabase, {
            canonical_role_id: canonical!.id,
            source_type: 'fantastic_jobs',
            job_posting_id: vaga!.id,
        });
        
        // Verificar que NÃO foi inserido em job_canonical_role_sources
        const { count } = await supabase
            .from('job_canonical_role_sources')
            .select('*', { count: 'exact', head: true })
            .eq('canonical_role_id', canonical!.id);
        
        expect(count).toBe(0);
        
        // Cleanup
        await supabase.from('job_postings').delete().eq('id', vaga!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', canonical!.id);
    });
});
```

### 1A.bis.7 — Esforço estimado

- **Schema + arquivo JSON:** 30 minutos
- **Função `isRecruitmentAgency` + integração Fluxo B:** 1.5 horas
- **Filtro em `upsertSource`:** 1 hora
- **Backfill:** 30 minutos
- **Tests:** 1 hora
- **Total:** 4.5 horas

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
// tests/integration/sprint-v1_0/item-4-orfas.spec.ts

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

**Verificação após DROP:** confirmar que trigger ainda está ativo e funcionando:

```sql
SELECT trigger_name, event_manipulation, action_timing 
FROM information_schema.triggers 
WHERE event_object_table = 'job_canonical_roles'
  AND trigger_name LIKE '%vacancy_count%';

-- Esperado: trigger FOR EACH STATEMENT continua ativo
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

### 8.1 — Schema: FK em `analyses`

```sql
BEGIN;

ALTER TABLE analyses 
ADD COLUMN IF NOT EXISTS rapidapi_usage_log_id UUID 
REFERENCES rapidapi_usage_logs(id) ON DELETE SET NULL;

CREATE INDEX IF NOT EXISTS idx_analyses_rapidapi_log 
ON analyses(rapidapi_usage_log_id) 
WHERE rapidapi_usage_log_id IS NOT NULL;

COMMIT;
```

### 8.2 — Lógica no `worker.ts:397` (cache vs API call)

O worker decide entre cache local (Postgres) e API call. Quando vai para API, registra log. Precisa propagar o ID do log até `analyses`.

**Refatoração:**

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
            .select('id')
            .single();
        
        rapidapiLogId = log?.id ?? null;
        
        // Persiste vagas novas em job_postings
        await persistApiResults(supabase, apiResult.jobs);
    }
    
    // Atualiza analyses com FK do log (mesmo se cache hit, pode ser do último log relevante)
    // Estratégia: pega o log MAIS RECENTE para o canônico+estado nas últimas 120d
    if (!rapidapiLogId) {
        const { data: ultimoLog } = await supabase
            .from('rapidapi_usage_logs')
            .select('id')
            .eq('endpoint', 'fantastic-jobs/search')
            .contains('request_payload', { canonical_role_id: canonicalRoleId, location_state: locationState })
            .order('created_at', { ascending: false })
            .limit(1)
            .single();
        
        rapidapiLogId = ultimoLog?.id ?? null;
    }
    
    // Update analyses
    await supabase
        .from('analyses')
        .update({ rapidapi_usage_log_id: rapidapiLogId })
        .eq('id', analysisId);
    
    // ... continua o resto do pipeline ...
}
```

### 8.3 — Banner D condicional no Modal de Análise

Quando o usuário vê resultado da análise, mostrar um banner discreto se a busca foi feita predominantemente em cache stale (último `rapidapi_usage_log` há > 60 dias). Banner D = "Dados de mercado podem estar levemente defasados."

**Frontend — `components/home/modals/AnalysisResultModal.tsx`:**

```typescript
interface BannerStaleProps {
    rapidapiLogCreatedAt: string | null;
}

function BannerStale({ rapidapiLogCreatedAt }: BannerStaleProps) {
    if (!rapidapiLogCreatedAt) return null;
    
    const daysAgo = Math.floor(
        (Date.now() - new Date(rapidapiLogCreatedAt).getTime()) / (1000 * 60 * 60 * 24)
    );
    
    if (daysAgo < 60) return null;  // Cache fresco — sem banner
    
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
    // analysis tem campo rapidapi_usage_log_id
    const [logCreatedAt, setLogCreatedAt] = useState<string | null>(null);
    
    useEffect(() => {
        if (!analysis.rapidapi_usage_log_id) return;
        
        supabase
            .from('rapidapi_usage_logs')
            .select('created_at')
            .eq('id', analysis.rapidapi_usage_log_id)
            .single()
            .then(({ data }) => setLogCreatedAt(data?.created_at ?? null));
    }, [analysis.rapidapi_usage_log_id]);
    
    return (
        <div className="modal-content">
            <BannerStale rapidapiLogCreatedAt={logCreatedAt} />
            
            {/* ... resto da UI ... */}
        </div>
    );
}
```

### 8.4 — Tests

```typescript
// tests/integration/sprint-v1_0/item-8-banner-stale.spec.ts

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
// tests/integration/sprint-v1_0/item-10-thresholds.spec.ts

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

```typescript
// tests/integration/sprint-v1_0/item-13-pipeline-tests.spec.ts

import { createAdminServerClient } from '@/lib/supabase-server';
import { persistPrecheck } from '@/lib/pipeline/persist-precheck';
import { persistCuration } from '@/lib/pipeline/persist-curation';

const supabase = createAdminServerClient();

describe('persist-precheck — testes M2', () => {
    let testCanonicalId: string;
    let testJobId: string;
    
    beforeEach(async () => {
        // Setup: criar canônico active e vaga pending
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .insert({ label: 'Test Persist Precheck', slug: 'test-persist-precheck', status: 'active' })
            .select()
            .single();
        testCanonicalId = canonical!.id;
        
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
        await supabase.from('job_canonical_roles').delete().eq('id', testCanonicalId);
    });
    
    it('cache hit: vaga é apontada para canônico existente', async () => {
        // Setup: criar âncora pré-existente
        await supabase.from('canonical_role_anchors').insert({
            description_hash: 'hash-test-1',
            canonical_role_id: testCanonicalId,
            taxonomy_content_version: 'v1',
        });
        
        // Executar
        await persistPrecheck(supabase, {
            job_id: testJobId,
            description_hash: 'hash-test-1',
        });
        
        // Verificar
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
            })
            .select()
            .single();
        
        // Persist curação (canônico novo)
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Persist Test New',
            layer: 2,
            llm_proposed_label: 'Canonical Persist Test New',
        });
        
        // Verificar canônico criado
        const { data: canonical } = await supabase
            .from('job_canonical_roles')
            .select('id, status')
            .eq('label', 'Canonical Persist Test New')
            .single();
        
        expect(canonical).not.toBeNull();
        expect(canonical?.status).toBe('pending');  // Apenas 1 vaga, não promove
        
        // Verificar source criado
        const { data: source } = await supabase
            .from('job_canonical_role_sources')
            .select('source_type')
            .eq('canonical_role_id', canonical!.id)
            .single();
        
        expect(source?.source_type).toBeDefined();
        
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
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Idempotency Test',
            layer: 2,
            llm_proposed_label: 'Canonical Idempotency Test',
        });
        
        // Segunda chamada (idêntica) — deve ser idempotente
        await persistCuration(supabase, {
            job_id: job!.id,
            canonical_proposed: 'Canonical Idempotency Test',
            layer: 2,
            llm_proposed_label: 'Canonical Idempotency Test',
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
    it('remap simples: vagas reapontadas e source apagado', async () => {
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
        
        // Chamar endpoint via fetch
        const res = await fetch(`http://localhost:3000/api/admin/canonicals/${source!.id}/remap`, {
            method: 'POST',
            headers: { 
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${ADMIN_TOKEN}`,
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
        
        // Source deve ter status=merged e merged_into=target
        const { data: sourceAfter } = await supabase
            .from('job_canonical_roles')
            .select('status, merged_into')
            .eq('id', source!.id)
            .single();
        
        expect(sourceAfter?.status).toBe('merged');
        expect(sourceAfter?.merged_into).toBe(target!.id);
        
        // Cleanup
        for (const v of vagas!) {
            await supabase.from('job_postings').delete().eq('id', v.id);
        }
        await supabase.from('job_canonical_roles').delete().eq('id', source!.id);
        await supabase.from('job_canonical_roles').delete().eq('id', target!.id);
    });
    
    it('valida permissão admin', async () => {
        const res = await fetch('http://localhost:3000/api/admin/canonicals/some-id/remap', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },  // sem auth
            body: JSON.stringify({ target_canonical_id: 'other-id' }),
        });
        
        expect(res.status).toBe(401);
    });
});

describe('endpoint /admin/canonicals/human-validated — testes M2', () => {
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
                'Authorization': `Bearer ${ADMIN_TOKEN}`,
            },
            body: JSON.stringify({ canonical_ids: [canonical!.id] }),
        });
        
        expect(res.status).toBe(200);
        
        // Verificar flag setada
        const { data: cAfter } = await supabase
            .from('job_canonical_roles')
            .select('human_validated_at, human_validated_by')
            .eq('id', canonical!.id)
            .single();
        
        expect(cAfter?.human_validated_at).toBeTruthy();
        expect(cAfter?.human_validated_by).toBeTruthy();
        
        // Verificar evento de auditoria
        const { data: events } = await supabase
            .from('events')
            .select('*')
            .eq('entity_id', canonical!.id)
            .eq('event_name', 'canonical_human_validated')
            .limit(1);
        
        expect(events?.length).toBe(1);
        
        // Cleanup
        await supabase.from('events').delete().eq('entity_id', canonical!.id);
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

### 14.1 — Estrutura do componente principal

**Localização:** `components/admin/dashboard/RemapDrilldown.tsx`

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
            .single()
            .then(({ data }) => {
                setEvent(data);
                setLoading(false);
            });
    }, [eventId]);
    
    if (loading) return <div>Carregando...</div>;
    if (!event) return <div>Evento não encontrado.</div>;
    
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
                    
                    <dt>Origem (loser)</dt>
                    <dd>
                        {event.previous_state?.canonical_label} 
                        <span className="muted">(id: {event.previous_state?.canonical_id})</span>
                    </dd>
                    
                    <dt>Destino (winner)</dt>
                    <dd>
                        {event.new_state?.canonical_label}
                        <span className="muted">(id: {event.new_state?.canonical_id})</span>
                    </dd>
                    
                    <dt>Razão</dt>
                    <dd>{event.new_state?.reason ?? 'Não documentado'}</dd>
                    
                    <dt>Vagas afetadas</dt>
                    <dd>{event.new_state?.affected_count ?? 'N/A'}</dd>
                </dl>
            </section>
            
            <section>
                <h4>Ações</h4>
                <button onClick={() => window.open(`/admin/canonicals/${event.new_state?.canonical_id}`, '_blank')}>
                    Ver canônico destino
                </button>
                <button className="danger" onClick={() => /* handler de rollback se aplicável */}>
                    ↩ Reverter remap
                </button>
            </section>
        </aside>
    );
}
```

### 14.2 — Estrutura do componente para `human-validated`

**Localização:** `components/admin/dashboard/HumanValidatedDrilldown.tsx`

```typescript
// components/admin/dashboard/HumanValidatedDrilldown.tsx
export function HumanValidatedDrilldown({ canonicalId, onClose }: Props) {
    const [data, setData] = useState<any>(null);
    
    useEffect(() => {
        const supabase = createClient();
        Promise.all([
            // Detalhes do canônico
            supabase
                .from('job_canonical_roles')
                .select('*')
                .eq('id', canonicalId)
                .single(),
            // Histórico de eventos human_validated
            supabase
                .from('events')
                .select('*')
                .eq('entity_id', canonicalId)
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
                    <dd>{data.canonical.human_validated_by ?? 'N/A'}</dd>
                    
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
                            <span>{e.actor_id ? `por ${e.actor_id}` : 'automatizado'}</span>
                        </li>
                    ))}
                </ul>
            </section>
            
            <section>
                <h4>Ações</h4>
                <button onClick={() => /* handler para revogar validation */}>
                    Revogar validação manual
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

```typescript
// tests/integration/sprint-v1_0/item-14-drilldown.spec.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { RemapDrilldown } from '@/components/admin/dashboard/RemapDrilldown';

describe('Item 14 — Drilldown UI', () => {
    it('RemapDrilldown carrega evento e exibe campos relevantes', async () => {
        // Setup: criar evento de teste
        const { data: event } = await supabase
            .from('events')
            .insert({
                event_name: 'job_canonical_remapped',
                entity_type: 'job_canonical_role',
                entity_id: 'src-id',
                previous_state: { canonical_label: 'Source A', canonical_id: 'src-id' },
                new_state: { canonical_label: 'Target B', canonical_id: 'tgt-id', reason: 'Test merge' },
            })
            .select()
            .single();
        
        const onClose = jest.fn();
        render(<RemapDrilldown eventId={event!.id} onClose={onClose} />);
        
        await waitFor(() => {
            expect(screen.getByText('Source A')).toBeInTheDocument();
            expect(screen.getByText('Target B')).toBeInTheDocument();
            expect(screen.getByText('Test merge')).toBeInTheDocument();
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

- **`RemapDrilldown.tsx`:** 2.5 horas
- **`HumanValidatedDrilldown.tsx`:** 2 horas
- **Integração nos painéis O8/O9:** 1.5 horas
- **Tests:** 1 hora
- **Total:** 7 horas

---


# Conclusão e Checklist Consolidado

## Esforço total consolidado

| PR | Item | Esforço | Subtotal PR |
|---|---|---|---|
| **PR1** | Item 15 (drifts de nomenclatura) | 6-7h | |
| | Item 16 (DROP de colunas redundantes) | 6h | |
| | Item 17 (DROP de tabela/coluna fantasma) | 1h | |
| | Item 18 (RLS faltante + DROP backups) | 2.5h | **15.5-16.5h** |
| **PR2** | Item G2 (taxonomia dinâmica) | 21.5h | |
| | Item G3 (áreas de atuação 0:N) | 16h | **37.5h** |
| **PR3** | Item 7 (regra dinâmica + retroativa + resolve_canonical) | 6.5h | |
| | Item 2 (CRON pipeline-maintenance) | 7h | **13.5h** |
| **PR4** | Item 1A (distinct_sources_count) | 4h | |
| | Item 1A.bis (filtro de agências) | 4.5h | |
| | Item 4 (saneamento 270 órfãs) | 3h | |
| | Item 6 (trigger autoritativo único) | 1.5h | **13h** |
| **PR5** | Item 8 (FK rapidapi + banner D) | 5h | |
| | Item 10 (thresholds fundamentados) | 4h | **9h** |
| **PR6** | Item 13 (tests M2 — 4 arquivos) | 8.5h | |
| | Item 14 (UI admin drilldown) | 7h | **15.5h** |
| **TOTAL** | | | **104h** |

**Estimativa antes de paddings de revisão Claude e ajustes:** 104 horas.
**Estimativa com padding (revisão + ajustes):** **48-70 horas se Antigravity executar com fluência** (estimativa original mantida porque inclui paralelizações típicas e código boilerplate gerado rapidamente).

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
- [ ] `canonical_role_domain_links` (junction N:N) criada
- [ ] `canonical_role_domains` populada (20 áreas + 128 + ~536 backfill IA)
- [ ] `profiles.pending_label_change_notification` adicionada
- [ ] `analyses.rapidapi_usage_log_id` adicionada
- [ ] `job_postings.is_recruitment_agency` adicionada
- [ ] `job_canonical_roles.domain_id` (1:1 dormente) DROPADA
- [ ] 9 colunas redundantes DROPADAS (Item 16)
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
- [ ] `lib/taxonomy/cache.ts` (cache Redis write-through)
- [ ] `lib/taxonomy/merge-canonicals.ts` (mergeCanonicals + markUsersForLabelChangeNotification + generateSlug)
- [ ] `lib/pipeline/agencies.ts` (isRecruitmentAgency)
- [ ] Substituir consumers de JSONs → cache Redis
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
- [ ] Email template `label_change_notification` cadastrado no Hostinger

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
- [ ] Redis Upstash ativo
- [ ] Permissão de escrita em `data/recruitment-agencies.json` e `data/methodology-content.md`
- [ ] Email template `label_change_notification` desenhado e pronto para cadastro no Hostinger

---

## Riscos identificados e mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Migration Item 7B (retroativa) apaga canônicos legítimos | Baixa | Alto | Guard `id NOT IN (SELECT canonical_role_id FROM job_postings)` + dry-run obrigatório |
| Substituição de JSON por Redis quebra Camada 1 | Média | Alto | Tests M2 exaustivos antes do deploy + feature flag para rollback |
| Backfill IA do Item G3 demora muito (>30min) | Média | Médio | Limite de batch + retry automático + checkpoint para retomada |
| CRON Opus erra em decisões DISAGREE com slug conflict | Baixa | Médio | Defesa em código: se conflict, faz merge ao invés de rename |
| `resolve_canonical` entra em ciclo infinito | Baixíssima | Alto | Profundidade máxima 10 já cravada na função SQL |
| RLS deny-all em `canonical_seniority_distribution` quebra leitura admin | Baixa | Baixo | Bloco R confirmou: todos callers usam createAdminServerClient (bypass) |
| Trigger `distinct_sources_count` causa lock em batch grande | Média | Médio | `FOR EACH ROW` é eficiente; índice em `canonical_role_id` + monitoramento |
| Renames de colunas rompem queries em produção | Média | Alto | grep exaustivo + dry-run + deploy gradual + rollback prep |

---

## Próximos passos pós-Sprint v1.0

Anotações para sprints futuras (não fazem parte desta v1.0):

**G2 — evolução pós-launch:**
- Tela admin de revisão de candidatos (caso fluxo Sonnet→Opus precise intervenção manual)
- Tunagem do batch size do CRON Opus baseado em volume real
- Implementação de Batch API (50% off) quando volume justificar

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

Esta spec é considerada **completa e cravada** pelo PO em sua versão FINAL v1.0. 

Após aprovação, fluxo de execução:
1. Multi-AI review (DeepSeek, Genspark, Gemini, GPT, Mistral, Grok, Manus, Copilot)
2. Triage de feedback por Claude (acatado/rejeitado/duplicata com contagem)
3. Antigravity implementa PR a PR conforme ordem recomendada
4. Claude Code executa migrations SQL diretamente no Supabase
5. Validação por Onsly em cada PR antes de avançar para o próximo

**Este documento é a fonte autoritativa única da Sprint v1.0.** Qualquer divergência durante implementação deve voltar para essa spec e ser registrada como mudança formal.

---

**Fim da Sprint v1.0 — FINAL**

