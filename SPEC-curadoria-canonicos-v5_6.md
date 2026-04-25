# SPEC — Curadoria de Canônicos e Prevenção Sistêmica de Erros de Classificação LLM

**Projeto:** CalibraCV
**Versão:** 5.6
**Data:** 24 de abril de 2026
**Destinatário primário:** Claude Code (execução)
**Status:** pronta para execução pelo Claude Code

---
## 1. Sumário executivo

**Objetivo:** reduzir estruturalmente a taxa de mis-classificação de canônicos funcionais no pipeline de curadoria de vagas, eliminando o efeito de não-determinismo do LLM observado empiricamente, ativando a infraestrutura dos 4 JSONs de taxonomia que hoje é código morto, e introduzindo camadas determinísticas de pré-resolução que complementam (sem substituir) o motor LLM.

**Evidência empírica consolidada (medida em 22/04/2026 — rerun obrigatório no dia do PR1):**

| Métrica | Valor |
|---|---|
| Vagas curadas no catálogo | ~8.034 |
| Vagas em grupos de descrição duplicada | ~839 (~10,4%) |
| Grupos distintos de duplicatas | ~263 |
| Vagas em grupos com divergência de canônico | ~122 |
| Grupos com divergência de canônico | ~31 |
| Percentual de chamadas LLM evitáveis em grupos duplicados | ~68,7% |
| Chamadas LLM historicamente desperdiçadas | ~576 |

**Ganho esperado de economia LLM — calibração de expectativa (v5.4, GS-R6-5):**

A Camada 0 captura ~68,7% dos duplicados detectados, mas duplicados representam ~10,4% do catálogo. A economia real agregada de chamadas LLM no catálogo atual é **~7% do total** (10,4% × 68,7%). **Esta é a faixa esperada — não 30-50%.** O ganho estrutural cresce se o mix futuro de vagas tiver maior taxa de duplicação (ex: empresas usando templates de descrição). A Camada 1 (domain_synonyms) adiciona economia incremental variável (esperado 5-15% adicional, dependendo de quantos títulos casam em synonyms), mas isso é ganho probabilístico, não medido empiricamente no catálogo atual. Total projetado combinado: **12-22% no catálogo atual**. Sprints futuras (near-duplicate via MinHash, embedding fallback) podem ampliar para 30%+ se monitoramento indicar necessidade.

**Entregáveis (agrupados em 2 PRs sequenciais):**

**PR1 — Conteúdo (risco baixo, impacto imediato):**
1. Patch no `domain_synonyms.json` v2.0 → v3.0 com 68 entradas novas. Baseline real de 113 entradas existentes. Total final: 181.
2. Reescrita parcial do `SYSTEM_PROMPT.ts` com regras A-E incorporadas, Etapas 0-5 renumeradas, micro-regra contextual Support vs Service.
3. Ajuste no `generate-prompt-version.ts` para emitir `PROMPT_STRUCTURE_VERSION` e `PROMPT_CONTENT_VERSION` separadamente.
4. Ajuste no `buildUserPrompt` para aceitar atributos XML extras com escape de aspas.
5. Nenhuma mudança em `equivalences.json`, `family_synonyms.json` ou `domains.json`.

**PR2 — Infraestrutura (risco médio, impacto estrutural):**
6. Camada 0: SHA-256 da descrição normalizada + clonagem de 5 outputs + quórum 2-fases + guarda de 80 palavras.
7. Camada 1: `domain_synonyms.json` lookup pre-LLM com regex compilado e heurística de tokens + posição, sem race condition em workers concorrentes.
8. Camada 2 simplificada: overlap de tokens com `family_synonyms` e `domains` in-memory. Sem embedding.
9. Preload de `validCanonicalLabels`, `allowedForPreResolution` e `vacancyCountByLabel` no boot do `getFullTaxonomyCache` (Camadas 1 e 2 operam 100% in-memory, zero query per-call).
10. Migração SQL com 10 novas colunas em `job_postings`: `description_hash`, `curation_source`, `prompt_structure_version`, `prompt_content_version`, `human_validated`, `canonical_resolved_at`, `canonical_role_label`, `layer_2_hint_count`, `original_title`, `original_description` (v5.6 — ChatGPT L1: contagem corrigida; v5.5 e anteriores diziam 8 omitindo as 2 colunas `original_*` adicionadas pela v5.4).
11. Backfill estimado em 40 segundos.
12. Função `normalizeResumeText` extraída para `lib/minhash.ts` como server-safe.
13. Modificação em `persistCuratedJob` para gravar `description_hash`, `prompt_structure_version`, `prompt_content_version`, `curation_source` nas vagas curadas pelo LLM (pool de âncoras da Camada 0 cresce organicamente).
14. Endpoint admin para setar/revogar `human_validated` com trilha de auditoria em `events`.
15. Função `stablePercent(jobId)` para rollout percentual determinístico baseado em hash de ID.

### 1.3 Não-objetivos

Esta v5.6 **não** resolve os seguintes problemas, deliberadamente:

1. **Quase-duplicatas com descrições levemente diferentes.** Camada 0 captura apenas duplicatas exatas via SHA-256. Near-duplicates (Jaccard 0.85-0.99) ficam para eventual sprint posterior se monitoramento §8.4 indicar necessidade. Erros sistêmicos de empresas que variam descrição mínima (mudar cidade, empresa, 1 palavra) não são endereçados nesta versão.
2. **Promoção automática de canônicos `pending` para `active`.** 533 canônicos em pending hoje. v5.5 **aceita pending no lookup** (Camada 1 emite flag) mas **exige aprovação explícita do Onsly** (via §2.4 + whitelist `allowedForPreResolution`) para auto-decisão sem LLM. Automação de promoção baseada em vacancy_count ou quórum temporal fica para sprint separada.
3. **Revisão humana obrigatória para domínios novos.** Erros Tipo 4 do diagnóstico (~12%) — vagas cujo domínio não tem canônico correspondente — continuam dependendo de decisão manual do Onsly. Pipeline v5.5 não cria canônicos automaticamente para domínios inéditos; marca como `pending` e aguarda curadoria.
4. **Convergência semântica em títulos ambíguos fora do patch §6.1.** As 68 entradas novas cobrem os casos de alto volume identificados empiricamente. Títulos ambíguos não cobertos (ex: "Solution Consultant" — delivery vs pre-sales) continuam dependendo do LLM guiado pelas Regras A-E.
5. **Auditoria retroativa do catálogo legado.** Backfill da v5.5 grava `prompt_structure_version='legacy'` em vagas anteriores (exceto `human_validated=true`), marcando-as como não-elegíveis para servir de âncora na Camada 0. Correção retroativa de erros sistêmicos pré-v5.5 é trabalho manual do Onsly (procedimento análogo ao Canonical GSI) ou sprint futura de batch recuration.
6. **Consolidação das CHECK constraints duplicadas em `curation_status`.** Conhecido mas não endereçado — v5.5 usa apenas a interseção efetiva dos dois CHECKs existentes.

---

## 2. Pré-requisitos operacionais obrigatórios

**Esta seção é convertida em TodoWrite pelo executor como primeira ação. Cada item tem critério de aceite binário. Se qualquer falhar, executor para e reporta ao Onsly.**

### 2.1 Pré-requisitos de colunas no banco

Validar que as colunas listadas abaixo existem em produção. Essas colunas são usadas pelas Camadas 0 e 1 da v5.1 (em especial `original_title` que serve de âncora para o guard de título da Camada 0) e precisam estar presentes antes do deploy do PR2.

**Critério de aceite binário:** a validação é feita via `information_schema.columns` (acessível via supabase-js). A tabela interna `supabase_migrations.schema_migrations` não é exposta via API, portanto usar apenas a query abaixo:

**v5.6 (ChatGPT C1) — correção de pré-requisito fantasma:** a v5.5 pedia validação de 5 colunas em `job_postings` + `analyses.error_code` + `ai_usage_logs.metadata`, mas apenas `original_title` e `original_description` são criadas em §5.1. As outras 5 referências (`original_language`, `curation_in_progress`, `curation_lock_acquired_at`, `analyses.error_code`, `ai_usage_logs.metadata`) não existem no schema atual e **nunca são criadas** pela spec. Eram resquício de convenção de outro projeto. A v5.6 remove essas referências: §2.1 valida apenas as 2 colunas que a spec de fato cria.

Colunas esperadas em `job_postings` (criadas pela migration §5.1):

- `original_title text`
- `original_description text`

**Validação SQL:**

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'job_postings'
  AND column_name IN ('original_title', 'original_description');
-- Esperado: 2 linhas retornadas ANTES da migration (zero) e APÓS (2).
-- Esta validação serve como check pós-migration §5.1, não como pré-requisito.
```

**Estado esperado ANTES do deploy PR2:** 0 linhas (colunas ainda não existem). A §5.1 é quem as cria.

**Estado esperado APÓS Passo 12.2 (§5.1):** 2 linhas (ambas colunas criadas + backfilled).

### 2.2 `ingest-validation.ts` plugado nos callers

**Critério de aceite binário:** grep a seguir retorna ≥1 import fora do próprio arquivo.

```bash
grep -rn "validateAndNormalizeJobText" --include="*.ts" --exclude-dir=node_modules
```

Se retornar apenas `lib/pipeline/ingest-validation.ts` (zero imports externos), o código está órfão. Plugar nos 3 fluxos de origem de vagas (A: admin ingestor, B: RapidAPI CRON, C: usuário colando vaga individual) antes de prosseguir.

### 2.3 Canônicos-alvo criados via seed

**Decisão de produto (não empírica):** os 5 canônicos C-level são criados como siglas em inglês (CTO, CEO, COO, CMO, CFO).

**Rationale da decisão:** o volume de vagas C-level no catálogo é ínfimo (2-6 por cargo), insuficiente para decisão empírica reproduzível. Decisão é por consistência operacional e alinhamento com convenção estabelecida de termos técnicos incorporados ao português (DevOps, SRE, Product Owner, Scrum Master).

SQL com colunas NOT NULL obrigatórias do schema real + cláusula idempotente:

```sql
-- v5.6 (ChatGPT G4): source='seed' alinhado com o título da seção
-- ("Canônicos-alvo criados via seed"). 'seed' e 'manual' são ambos válidos
-- no CHECK source IN ('seed', 'llm_extractor', 'manual', 'merge'), mas 'seed'
-- facilita auditoria futura para identificar canônicos criados na inicialização
-- da v5.6 vs adicionados manualmente pelo admin posteriormente.
INSERT INTO job_canonical_roles (id, slug, label, status, source)
VALUES
    (gen_random_uuid(), 'cto', 'CTO', 'active', 'seed'),
    (gen_random_uuid(), 'ceo', 'CEO', 'active', 'seed'),
    (gen_random_uuid(), 'coo', 'COO', 'active', 'seed'),
    (gen_random_uuid(), 'cmo', 'CMO', 'active', 'seed'),
    (gen_random_uuid(), 'cfo', 'CFO', 'active', 'seed')
ON CONFLICT (slug) DO NOTHING;
```

Campos omitidos (`is_active`, `is_emerging`, `usage_count`, `vacancy_count`, `created_at`) usam DEFAULTs do schema. A cláusula `ON CONFLICT (slug) DO NOTHING` torna o seed idempotente — reexecuções em re-deploys não criam duplicatas.

**Critério de aceite binário:**

```sql
SELECT COUNT(*) FROM job_canonical_roles
WHERE slug IN ('cto', 'ceo', 'coo', 'cmo', 'cfo') AND status = 'active';
-- Esperado: 5
```

### 2.4 Canônicos destino do patch — decisão de produto

Executar validação de existência antes do PR1:

```sql
-- v5.6 (Claude Code #1): query atualizada para refletir labels já cravados
-- no Anexo B. "Analista de Sales Operations" virou "Analista de Operações de
-- Vendas" (rename 24/04). "Representante de Pré-Venda" foi merged para "SDR"
-- (Anexo B operação 1). Adicionado SDR e BDR como labels canônicos finais.
SELECT label, status, vacancy_count
FROM job_canonical_roles
WHERE label IN (
    'Gerente de Produto',
    'Gerente de Marketing',
    'Diretor de Produto',
    'Diretor de Marketing',
    'Gerente de Finanças',
    'Analista de Customer Service',
    'Analista de Customer Experience',
    'Analista de Operações de Vendas',
    'Analista de Marketing Digital',
    'Analista Tributário',
    'Analista de Compliance',
    'Analista de Compras',
    'Analista de Recrutamento e Seleção',
    'Analista de Operações Financeiras',
    'Coordenador de Projetos',
    'Gerente de Engenharia',
    'Gerente de TI',
    'Gerente de Operações',
    'Gerente de Customer Success',
    'Diretor de Finanças',
    'Diretor de Customer Success',
    'SDR',
    'BDR',
    'Analista de Recursos Humanos',
    'Analista de Marketing'
)
ORDER BY status, label;
```

Classificar resultado em 3 grupos:

1. **`status = 'active'`:** ação zero. Entra automaticamente em `allowedForPreResolution`.
2. **`status = 'pending'`:** **decisão de produto obrigatória** do Onsly para cada um. Três opções:
   - **(a) Promover para `active`:** UPDATE manual. Passa a ser auto-decidível.
   - **(b) Aceitar em `pending` com auto-decisão:** adicionar explicitamente à whitelist `allowedForPreResolution`. Camada 1 pode emitir flag e servidor sobrescreve. Registrar na tabela de auditoria (ver abaixo).
   - **(c) Aceitar em `pending` sem auto-decisão:** canônico permanece visível para lookup geral mas **fora** de `allowedForPreResolution`. Se Camada 1 casar por regex, o resultado é descartado (miss) e LLM decide em branco. Apropriado para canônicos pending questionáveis ou experimentais.
3. **Canônico não retornado pela query:** não existe. Executor reporta como anomalia. Decisão do Onsly: criar via seed manual, ou remover do patch §6.1, ou aceitar criação via `upsertCanonicalRole` na primeira curadoria (cria como `pending` sem auto-decisão).

**Tabela de auditoria da whitelist:**

Criar tabela `allowed_for_pre_resolution` via migração §5.4:

```sql
CREATE TABLE IF NOT EXISTS allowed_for_pre_resolution (
    canonical_role_id uuid NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
    added_at timestamptz NOT NULL DEFAULT now(),
    added_by uuid NOT NULL,
    reason text,
    PRIMARY KEY (canonical_role_id)
);

COMMENT ON TABLE allowed_for_pre_resolution IS
    'Subconjunto de job_canonical_roles que podem ser escolhidos automaticamente por Camada 0/1 sem LLM. Canônicos active são sempre elegíveis; canônicos pending só são elegíveis se estiverem nesta tabela.';
```

Para cada canônico que o Onsly decidir opção (b) acima, adicionar:

```sql
INSERT INTO allowed_for_pre_resolution (canonical_role_id, added_by, reason)
SELECT id, '00000000-0000-0000-0000-000000000004', 'Aprovado pelo Onsly em 23/04/2026 — volume suficiente, sem ambiguidade'
FROM job_canonical_roles
WHERE label = '<canônico aprovado>'
ON CONFLICT (canonical_role_id) DO NOTHING;
```

**`allowedForPreResolution` = canônicos `active` ∪ canônicos `pending` que estão na whitelist.** Populado pelo `getFullTaxonomyCache` no boot como `Set<string>`.

**Critério de aceite binário:** executor apresenta a classificação ao Onsly e **aguarda resposta explícita** sobre cada canônico em `pending` com escolha (a), (b) ou (c). Executor não decide sozinho. Resposta esperada em formato:

```
Para cada canônico pending listado:
- (a) Promover para active: [lista]
- (b) Aceitar pending com auto-decisão (whitelist): [lista]
- (c) Aceitar pending sem auto-decisão: [lista]
```

Se algum canônico listado em §6.1 (Categoria A, B, C ou D) não retornar na query acima nem estiver na lista de seed §2.3, executor reporta como anomalia e aguarda decisão.

### 2.5 Confirmações de estado

Checklist TodoWrite binário:

- [ ] Plano Vercel atual reportado (Hobby ou Pro).
- [ ] `LLM_BATCH_SIZE=38` confirmado em `constants.ts`.
- [ ] `temperature=0` confirmado em `batch-processor.ts:51`, `curate-job-postings/route.ts:60`, `llm-call.ts:35`.
- [ ] Hash atual em `prompt-version.generated.ts` reportado (valor livre).
- [ ] Conteúdo atual de `domain_synonyms.json` sem alteração desde 22/04/2026.
- [ ] Contagem atual de entradas em `domain_synonyms.json` validada via `jq '.data | length' domain_synonyms.json` (esperado: 113).
- [ ] Volume atual de vagas `curation_status='curated'` reportado.
- [ ] 5 canônicos C-level (§2.3) criados.
- [ ] Classificação §2.4 apresentada ao Onsly e resposta recebida.
- [ ] Medição de tokens do novo SYSTEM_PROMPT executada com `tiktoken` ou equivalente e valor reportado (esperado: 1.400-1.800 tokens adicionais; se ultrapassar, plano de mitigação documentado).

Apenas após todos os itens marcados, executor prossegue com o PR1.

---

## 3. Diagnóstico empírico do pipeline atual

### 3.1 Taxonomia dos erros observados

Classificação cross-canônico dos aproximadamente 80 erros identificados na investigação:

**Erro Tipo 1 — Hierarquia bilíngue colapsada em Analista (~38% dos erros)**
Manager/Head/Director/Specialist/Lead traduzidos incorretamente para Analista. Exemplos: "Head FP&A" → Analista Financeiro (correto: Diretor de Finanças); "Engineering Manager" → Engenheiro de Software (correto: Gerente de Engenharia); "Regional Support Manager" → Analista de Field Service (correto: Gerente de TI).

**Erro Tipo 2 — Qualificador de domínio em títulos Operations/Analytics (~31% dos erros)**
LLM vê "Operations" ou "Specialist" no título e mapeia para o canônico mais genérico disponível, ignorando o qualificador que especifica o domínio. Exemplos: "Sales Operations Analyst" → Analista de Operações (correto: Analista de Sales Operations); "People Operations Specialist" → Analista de Operações (correto: Analista de Recursos Humanos); "Tax Analyst" → Analista Financeiro (correto: Analista Tributário).

**Erro Tipo 3 — Support/Service bilíngue ambíguo (~19% dos erros)**
"Support" em inglês frequentemente designa atendimento ao cliente, não suporte técnico. Exemplos: "Customer Service Representative" → Analista de Suporte (correto: Analista de Customer Service); "Client Service Specialist" → Analista de Suporte (correto: Analista de Customer Service).

**Erro Tipo 4 — Domínio semântico novo sem canônico adequado (~12% dos erros)**
Vagas cujo domínio não tem canônico correspondente no catálogo atual e precisam de decisão humana ou criação de canônico novo. Não endereçável via automação.

### 3.2 Descoberta crítica — não-determinismo do LLM com temperature=0

Durante análise do canônico "Partner Sales Director GSI" da Canonical (7 vagas aparentemente duplicadas), validou-se por `linkedin_id` e URL Greenhouse que eram vagas reais distintas, ingeridas em momentos diferentes. A classificação divergiu entre três canônicos: 2 Executivo de Contas, 2 Gerente de Contas, 3 Diretor de Vendas.

A auditoria confirmou:
- `temperature=0` está corretamente setado nos três locais do pipeline.
- `cache_control: ephemeral` para prompt caching é funcional.
- Não existe pre-check por hash antes da chamada LLM.
- `LLM_BATCH_SIZE=38` implica que cada vaga é processada num batch com outras 37 vagas vizinhas.

**Causa raiz (primária):** efeito de posição no batch (batch position effect). `temperature=0` garante determinismo por prompt completo, não por vaga individual. Para cargos ambíguos na tradução BR com múltiplos equivalentes válidos, a diferença de contexto entre batches é suficiente para gerar classificações divergentes mesmo com temperatura zero.

**Causa raiz (secundária, identificada pela pt. 15 em 24/04/2026):** drift de prompt sem mecanismo de reaferição. A sessão pt. 15 descobriu empiricamente que os 4 casos cross-type em `canonical_skills` foram classificados entre 14:49-14:52 UTC de 02/04/2026 sob prompt `8cb04a9`, enquanto a regra que teria classificado corretamente (Regra 5: "atendimento + contexto = hybrid") entrou no prompt às 18:30 UTC do mesmo dia — 3h40 depois. Nenhum mecanismo reavaliou aqueles canônicos após a regra entrar. Isso significa que parte do problema observado no catálogo **não é não-determinismo**, mas classificações corretas sob prompt imaturo que ficaram cristalizadas sem reaferição. A v5.4 adiciona coluna `prompt_version` em `job_canonical_roles` (§5.5) para permitir detecção e triagem retroativa na sprint pt. 15.

O caso Canonical GSI foi corrigido manualmente em sessão anterior (todas as 7 vagas agora apontam para canônico único `ed2cd343-925e-442d-9f26-c5a462a37e81`). Mas o padrão persiste em outros grupos do catálogo.

**Implicação para os critérios de aceite (Seção 9):** por ser não-determinístico por vaga individual, qualquer teste de regressão baseado em "N de M vagas mantêm canônico" deve ser executado múltiplas vezes com batches embaralhados e os resultados avaliados em média + desvio padrão, não em limiar único.

### 3.3 Quantificação empírica do problema

> **Nota de rodapé:** os números desta seção foram medidos em **22/04/2026 — rerun obrigatório no dia do PR1**. Ordem de grandeza e direção das conclusões são estáveis; números exatos atualizáveis. As variações entre medições em dias próximos refletem chegada de novas vagas via Fluxo B (CRON RapidAPI), não mudança arquitetural.

Três queries executadas no banco de produção para medir o volume e a qualidade do problema:

**Validação 1 — Duplicatas de descrição (md5 sobre `requirements->>'description'` normalizado):**

| Métrica | Valor 22/04/2026 (baseline) |
|---|---|
| Total de grupos de duplicatas (≥2 vagas com mesma descrição) | 263 |
| Total de vagas em grupos duplicados | 839 |
| Percentual do catálogo curado | ~10,4% |
| Grupos com divergência de canônico | 31 |
| Vagas em grupos com divergência de canônico | 122 |

**Validação 2 — Caso Canonical GSI investigado:**

As 7 vagas da Canonical têm:
- `desc_md5` idêntico em todas (descrição byte-a-byte igual)
- `content_hash` distinto em todas (metadata além de descrição varia)
- `canonical_role_id` idêntico agora (corrigido manualmente)

Conclusão: o caso seminal era duplicata exata de descrição, não quase-duplicata. SHA-256 simples resolveria.

**Validação 3 — Distribuição por tamanho de grupo:**

| Faixa | Grupos | Vagas | Divergentes | % Divergência |
|---|---|---|---|---|
| 2-3 vagas | 214 | 461 | 26 | ~12% |
| 4-7 vagas | 28 | 144 | 1 | ~4% |
| 8-15 vagas | 19 | 193 | 2 | ~11% |
| 16+ vagas | 2 | 41 | 2 | 100% |

Padrão bimodal: grupos pequenos pulverizados e 2 grupos gigantes (Stone, AgileEngine) com 100% de divergência. Rollout gradual por tamanho não faz sentido — ativação flat.

**Validação 4 — Projeção de economia LLM:**

| Métrica | Valor |
|---|---|
| Vagas âncora (primeira ocorrência, curadas pelo LLM) | 263 |
| Vagas que Camada 0 teria interceptado | 576 |
| Percentual de chamadas LLM evitáveis em grupos duplicados | ~68,7% |

**v5.5 (ChatGPT bloco 1) — Query SQL para reproduzir as métricas acima** (aide-mémoire — executor pode rodar em staging/produção para re-validar em datas futuras):

```sql
-- Métrica 1: total de vagas em grupos duplicados (denominador do 10,4%)
WITH grupos AS (
    SELECT
        description_hash,
        COUNT(*) AS vagas_no_grupo
    FROM job_postings
    WHERE curation_status = 'curated'
      AND description_hash IS NOT NULL
    GROUP BY description_hash
    HAVING COUNT(*) >= 2
)
SELECT
    (SELECT SUM(vagas_no_grupo) FROM grupos) AS total_em_duplicados,
    (SELECT COUNT(*) FROM grupos) AS total_grupos,
    (SELECT COUNT(*) FROM job_postings WHERE curation_status = 'curated') AS total_curadas;

-- Métrica 2: chamadas LLM evitáveis (576) — N-1 por grupo
-- Para cada grupo de N duplicatas, LLM é chamado 1x (âncora) e N-1 seriam
-- interceptadas pela Camada 0.
SELECT
    SUM(vagas_no_grupo - 1) AS llm_calls_eviteis,
    SUM(vagas_no_grupo - 1)::numeric / SUM(vagas_no_grupo) AS pct_eviteis_em_grupos
FROM (
    SELECT description_hash, COUNT(*) AS vagas_no_grupo
    FROM job_postings
    WHERE curation_status = 'curated'
      AND description_hash IS NOT NULL
    GROUP BY description_hash
    HAVING COUNT(*) >= 2
) sub;

-- Métrica 3: grupos com divergência de canônico (122 vagas)
SELECT
    COUNT(DISTINCT description_hash) AS grupos_divergentes,
    SUM(vagas_do_grupo) AS vagas_em_grupos_divergentes
FROM (
    SELECT
        description_hash,
        COUNT(*) AS vagas_do_grupo,
        COUNT(DISTINCT canonical_role_id) AS canonicos_distintos
    FROM job_postings
    WHERE curation_status = 'curated'
      AND description_hash IS NOT NULL
    GROUP BY description_hash
    HAVING COUNT(*) >= 2
       AND COUNT(DISTINCT canonical_role_id) > 1
) sub;
```

**Nota:** essas queries dependem da coluna `description_hash` já populada (pós-backfill §7.6). Antes do backfill, usar `md5(requirements->>'description')` como proxy — resultado aproximado.

### 3.4 Estado do catálogo de canônicos

Dados do banco de produção em 22/04/2026:

| Métrica | Valor |
|---|---|
| `job_postings` total | ~8.662 |
| `curation_status = 'curated'` | ~8.034 |
| `job_canonical_roles` total | 652 |
| status='active' | 31 (~5%) |
| status='pending' | 533 (~82%) |
| status='deprecated' | 19 |
| status='rejected' | 69 |

Observação crítica: apenas ~5% dos canônicos estão `active`, mas ~94% das vagas têm `canonical_role_id` populado — a grande maioria aponta para canônicos em estado pending. A v5.1 relaxa a guarda de pre-check para aceitar `status IN ('active', 'pending')` dado esse quadro, mas introduz whitelist `allowedForPreResolution` (§2.4) para restringir auto-decisão apenas aos canônicos aprovados pelo Onsly. Auto-promoção automática de pending para active é débito conhecido endereçável em sprint separada.

---

## 4. Arquitetura-alvo: pipeline reestruturado em 5 camadas

### 4.1 Fluxo canônico vaga-a-vaga

```
[Vaga chega em processBatch com curation_status='pending']
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 0 — Pre-check por SHA-256 da descrição                │
│ • Calcula SHA-256 de normalizeDescriptionForHash(description)│
│ • Guarda: ≥80 palavras na descrição normalizada. Se menor,   │
│   description_hash=NULL e Camada 0 passa direto              │
│ • Busca vagas curadas com mesmo description_hash             │
│ • Filtros: status canônico ∈ {'active','pending'},           │
│   curated_at dentro de TTL 30 dias,                          │
│   prompt_structure_version igual ao atual                    │
│ • SE hit com quórum ≥ 1 human_validated=true OU             │
│   ≥ 2 âncoras com gap temporal ≥ 24h:                        │
│     → CLONA 5 outputs da âncora: canonical_role_id, skills,  │
│       seniority_inferred, work_model, description_curated    │
│     → curation_source = 'precheck_description_hash'          │
│     → SKIP integral (não chama LLM)                          │
│ • SE miss: guarda hash gerado, segue                         │
└─────────────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 1 — domain_synonyms lookup pre-LLM                    │
│ • Normaliza título via normalizeTitle                        │
│ • matchAll com regex clonado por chamada (sem estado global) │
│ • Múltiplos matches: mais tokens > menos tokens;             │
│   empate → match mais à esquerda no título vence             │
│ • Valida canônico destino consultando Set<string> preloaded  │
│   (labels em ('active', 'pending')). In-memory, zero query.  │
│ • SE hit válido: canonical_already_resolved = label          │
│ • SE hit inválido ou miss: LLM decide em branco              │
└─────────────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 2 — Montagem de suggested_roles                       │
│ • SE Camada 1 resolveu: suggested_roles vazio                │
│ • SE Camada 1 miss:                                          │
│   1. Tenta family match: overlap de tokens do título com     │
│      tokens do nome da família. Família com mais overlap     │
│      vence. Retorna canônicos da família (cap 12)            │
│   2. Senão tenta domain match: overlap de tokens com nome    │
│      do domain. Retorna canônicos do domain ranqueados por   │
│      vacancy_count DESC (cap 12)                             │
│   3. Senão suggested_roles vazio (LLM decide em branco)      │
│ • SEM fallback por embedding nesta v5.6                      │
└─────────────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 3 — Chamada ao Sonnet 4.6                             │
│ • callLLM com batch de até LLM_BATCH_SIZE=38 vagas            │
│ • temperature=0, cache_control ephemeral preservado          │
│ • buildUserPrompt monta tag <job id="..."                    │
│     [canonical_already_resolved="<label>"]?                  │
│     [suggested_roles="A | B | C"]?                           │
│   > com escape de aspas em valores                           │
│ • Output: skills, seniority_inferred, canonical_role,        │
│   work_model, description_curated                            │
└─────────────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ CAMADA 4 — Pós-processamento                                 │
│ • equivalences.json lookup (preservado)                      │
│ • upsertCanonicalRole: blacklist, redirect chain             │
│ • Persist em job_postings com curation_status='curated',     │
│   curation_source ∈ {'precheck_description_hash',            │
│   'layer_1_domain_synonyms', 'llm_direct',                   │
│   'llm_equivalences_redirect', 'manual_remap',               │
│   'quarantined_llm_output'}  ← v5.6 (ChatGPT G2)             │
│ • description_hash populado em todas as vagas novas          │
└─────────────────────────────────────────────────────────────┘
                  │
                  ▼
[Vaga com curation_status='curated', canonical_role_id, skills,
 description_hash, curation_source, prompt_structure_version,
 prompt_content_version]
```

### 4.2 Ordem de avaliação em colisão

**Camada 0:** sempre primeiro. Se hit com quórum, nenhuma camada posterior executa.

**Camada 1:** opera sobre título, independente da Camada 0. Se Camada 0 resolveu, Camada 1 é pulada.

**Camada 2:** executa se o LLM vai ser chamado. Para vagas com Camada 1 resolvida, `suggested_roles` é vazio.

**Múltiplos matches em `domain_synonyms`:** heurística por número de tokens matching, não comprimento em caracteres. Exemplo: "Sales Operations Manager" casa em `"sales operations"` (2 tokens, posição 0) e `"operations manager"` (2 tokens, posição 1). Empate por tokens. Desempate por posição: `"sales operations"` vence. Correto semanticamente — "Sales Operations Manager" é Analista de Operações de Vendas.

---

## 5. Migração SQL

**v5.3 (C5) — nota sobre numeração:** as subseções desta seção aparecem na ordem §5.1, §5.4, §5.2, §5.3 (§5.4 foi inserida na v5.1 entre §5.1 e §5.2). A numeração é preservada por estabilidade de referências cruzadas em toda a spec — renumerar mudaria dezenas de citações. A ordem **de execução** do PR2 é independente da ordem textual: §11.3 Passo 12 especifica a sequência correta (colunas → índices → whitelist → triggers → backfill). Ao ler linearmente, a ordem sugerida é §5.0 → §5.1 → §5.2 → §5.3 → §5.4.

### 5.0 Backup pré-migration (v5.4)

**Contexto:** o plano atual do Supabase utilizado pelo CalibraCV é o gratuito, que não oferece backup automatizado nem Point-in-Time Recovery. A migration §5.1 adiciona 10 colunas em `job_postings` (8.308 linhas) e executa UPDATE inline para popular `original_title` e `original_description`. Em caso de erro operacional durante a execução, não há rollback nativo — precisa ser reconstruído manualmente.

**Mitigação v5.4:** criar tabelas espelho no mesmo banco **antes** de qualquer ALTER TABLE ou UPDATE. Custo: ~10-15 MB de storage temporário (job_postings tem ~8k linhas, job_canonical_roles tem ~9k linhas, skill_merge_decisions tem ~200 linhas, role_merge_decisions tem dezenas de linhas). Aceitável no plano atual.

**v5.4 (pt. 15 — feedback externo):** backup ampliado para 4 tabelas (antes eram 2). As duas tabelas `*_merge_decisions` estão em escopo porque o Passo 12 pode executar merges via tela admin, e o log dessas decisões mora nessas tabelas. Em caso de rollback, recuperar o log é essencial para auditoria forense.

```sql
-- EXECUTAR ANTES de qualquer statement da §5.1
-- 4 tabelas críticas (v5.4 pós-pt.15):
-- - job_postings: recebe colunas novas + backfill (§5.1)
-- - job_canonical_roles: recebe merge SDR e renames do Anexo B
-- - skill_merge_decisions: log de auditoria (não alterado pela v5.4 diretamente,
--   mas incluído para proteção cruzada com a sprint pt. 15 se executarem em ordem apertada)
-- - role_merge_decisions: log de auditoria das operações do Anexo B
--
-- v5.5 (DeepSeek #10): skill_merge_decisions e role_merge_decisions podem
-- ainda não existir — são criadas na sprint pt. 15 de governança. O DO block
-- abaixo usa condicional IF para backup apenas se a tabela existir.
--
-- v5.6 (Gemini #2): CREATE TABLE AS SELECT * sem colunas extras —
-- backup com schema idêntico ao original permite INSERT INTO ... SELECT *
-- direto no rollback, sem precisar adaptar lista de colunas em emergência P0.
-- Timestamp de quando foi feito fica em COMMENT ON TABLE, não em coluna.
--
-- v5.6 (ChatGPT G3) — LIMITAÇÕES CONHECIDAS DO BACKUP:
-- CREATE TABLE ... AS SELECT * copia APENAS dados + definição de colunas.
-- NÃO copia: índices, triggers, constraints (CHECK, FK, UNIQUE), RLS policies,
-- COMMENTs, default values complexos, ownership/grants.
-- Em rollback catastrófico, restaurar dados via backup NÃO reconstrói esses
-- artefatos. O runbook de rollback precisa incluir:
-- 1. DROP das colunas novas via §5.1 rollback
-- 2. TRUNCATE + INSERT INTO ... SELECT * FROM _backup_v5_5
-- 3. Re-aplicar índices, triggers, constraints se tiverem sido alterados
-- No plano atual (Supabase Free sem PITR), isso é o melhor que dá.
-- Qualquer restore via snapshot manual passa por Onsly abrir ticket no Supabase.

CREATE TABLE job_postings_backup_v5_5 AS
    SELECT * FROM job_postings;
COMMENT ON TABLE job_postings_backup_v5_5 IS
    'Backup tirado em [timestamp no momento de aplicação] como safety net para rollback da v5.6';

CREATE TABLE job_canonical_roles_backup_v5_5 AS
    SELECT * FROM job_canonical_roles;
COMMENT ON TABLE job_canonical_roles_backup_v5_5 IS
    'Backup tirado em [timestamp no momento de aplicação] como safety net para rollback da v5.6';

-- Backup condicional de skill_merge_decisions (só se existir)
DO $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public' AND table_name = 'skill_merge_decisions'
    ) THEN
        EXECUTE 'CREATE TABLE skill_merge_decisions_backup_v5_5 AS
                 SELECT * FROM skill_merge_decisions';
        EXECUTE 'COMMENT ON TABLE skill_merge_decisions_backup_v5_5 IS ''Backup para rollback v5.6''';
        RAISE NOTICE 'Backup skill_merge_decisions criado';
    ELSE
        RAISE NOTICE 'Tabela skill_merge_decisions não existe — backup pulado (escopo pt. 15)';
    END IF;
END $$;

-- Backup condicional de role_merge_decisions (só se existir)
DO $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public' AND table_name = 'role_merge_decisions'
    ) THEN
        EXECUTE 'CREATE TABLE role_merge_decisions_backup_v5_5 AS
                 SELECT * FROM role_merge_decisions';
        EXECUTE 'COMMENT ON TABLE role_merge_decisions_backup_v5_5 IS ''Backup para rollback v5.6''';
        RAISE NOTICE 'Backup role_merge_decisions criado';
    ELSE
        RAISE NOTICE 'Tabela role_merge_decisions não existe — backup pulado (escopo pt. 15)';
    END IF;
END $$;

-- Validação pós-backup: contagens devem bater 1:1 com originais (nas que existem)
DO $$
DECLARE
    orig_jp bigint;       bkp_jp bigint;
    orig_jcr bigint;      bkp_jcr bigint;
    smd_exists boolean;   rmd_exists boolean;
    orig_smd bigint;      bkp_smd bigint;
    orig_rmd bigint;      bkp_rmd bigint;
BEGIN
    SELECT COUNT(*) INTO orig_jp FROM job_postings;
    SELECT COUNT(*) INTO bkp_jp FROM job_postings_backup_v5_5;
    SELECT COUNT(*) INTO orig_jcr FROM job_canonical_roles;
    SELECT COUNT(*) INTO bkp_jcr FROM job_canonical_roles_backup_v5_5;

    IF orig_jp  != bkp_jp  THEN RAISE EXCEPTION 'Backup job_postings incompleto: orig=%, bkp=%', orig_jp, bkp_jp; END IF;
    IF orig_jcr != bkp_jcr THEN RAISE EXCEPTION 'Backup job_canonical_roles incompleto: orig=%, bkp=%', orig_jcr, bkp_jcr; END IF;

    -- Validação condicional das tabelas de merge
    SELECT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public' AND table_name = 'skill_merge_decisions_backup_v5_5'
    ) INTO smd_exists;

    IF smd_exists THEN
        SELECT COUNT(*) INTO orig_smd FROM skill_merge_decisions;
        SELECT COUNT(*) INTO bkp_smd FROM skill_merge_decisions_backup_v5_5;
        IF orig_smd != bkp_smd THEN RAISE EXCEPTION 'Backup skill_merge_decisions incompleto: orig=%, bkp=%', orig_smd, bkp_smd; END IF;
    END IF;

    SELECT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public' AND table_name = 'role_merge_decisions_backup_v5_5'
    ) INTO rmd_exists;

    IF rmd_exists THEN
        SELECT COUNT(*) INTO orig_rmd FROM role_merge_decisions;
        SELECT COUNT(*) INTO bkp_rmd FROM role_merge_decisions_backup_v5_5;
        IF orig_rmd != bkp_rmd THEN RAISE EXCEPTION 'Backup role_merge_decisions incompleto: orig=%, bkp=%', orig_rmd, bkp_rmd; END IF;
    END IF;

    RAISE NOTICE 'Backup validado: job_postings=%, job_canonical_roles=%, skill_merge_decisions=%, role_merge_decisions=%',
        bkp_jp, bkp_jcr,
        CASE WHEN smd_exists THEN bkp_smd::text ELSE 'N/A' END,
        CASE WHEN rmd_exists THEN bkp_rmd::text ELSE 'N/A' END;
END $$;
```

**Procedimento de rollback caso §5.1 ou §5.3 falhem catastroficamente:**

```sql
-- CENÁRIO 1: migration §5.1 falhou no meio, tabelas em estado inconsistente
-- Restaurar do backup (perde dados curados entre o backup e o momento do rollback,
-- que deve ser < 5 minutos se executado imediatamente)

BEGIN;

-- Drop das colunas adicionadas pela §5.1 (volta ao schema pré-v5.4)
ALTER TABLE job_postings
    DROP COLUMN IF EXISTS description_hash,
    DROP COLUMN IF EXISTS curation_source,
    DROP COLUMN IF EXISTS prompt_structure_version,
    DROP COLUMN IF EXISTS prompt_content_version,
    DROP COLUMN IF EXISTS canonical_resolved_at,
    DROP COLUMN IF EXISTS human_validated,
    DROP COLUMN IF EXISTS canonical_role_label,
    DROP COLUMN IF EXISTS layer_2_hint_count,
    DROP COLUMN IF EXISTS original_title,
    DROP COLUMN IF EXISTS original_description;

-- Restaurar dados se houve UPDATE parcial em linhas existentes (improvável mas possível)
-- Esta parte só é necessária se algum dado existente foi modificado, o que não ocorre
-- na §5.1 (tudo é ADD COLUMN). Mas a redundância não custa.
-- Se precisar, rodar:
-- TRUNCATE job_postings; INSERT INTO job_postings SELECT ... FROM job_postings_backup_v5_5;
-- [adaptar à lista de colunas original]

COMMIT;
```

**Remoção dos backups:** após validação de estabilidade do ambiente pós-rollout completo (Passo 15 em 100% + ausência de anomalias em `events` forensics por janela razoável — a definir pelo Onsly no momento do deploy, dado que o produto ainda não está em produção assistida), remover manualmente:

```sql
DROP TABLE IF EXISTS job_postings_backup_v5_5;
DROP TABLE IF EXISTS job_canonical_roles_backup_v5_5;
DROP TABLE IF EXISTS skill_merge_decisions_backup_v5_5;
DROP TABLE IF EXISTS role_merge_decisions_backup_v5_5;
```

**Nota operacional:** este backup é sincronicamente consistente (snapshot único da tabela no momento do CREATE TABLE AS SELECT). Não é backup incremental — se entre o backup e o rollback novas curagens chegaram, elas são perdidas no rollback. Por isso o backup deve ser feito imediatamente antes da migration, e o rollback deve ser decidido em minutos, não horas.

### 5.1 Colunas novas em `job_postings`

```sql
BEGIN;

-- Hash SHA-256 da descrição normalizada (64 chars hex)
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS description_hash text;

-- Dimensão ortogonal ao status — registra canal de curadoria
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS curation_source text
    CHECK (
        curation_source IS NULL OR
        curation_source IN (
            'precheck_description_hash',
            'layer_1_domain_synonyms',
            'llm_direct',
            'llm_equivalences_redirect',
            'manual_remap',
            'quarantined_llm_output'  -- v5.5 (Claude-2 médio): valor explícito facilita queries do painel admin
        )
    );

-- Versão estrutural do prompt (schema do output: campos e tipos)
-- Hash muda quando campos do output mudam
-- Pre-check invalida quando esta versão muda
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS prompt_structure_version text;

-- Versão de conteúdo do prompt (regras e exemplos)
-- Hash muda a cada ajuste de regra. Não invalida pre-check
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS prompt_content_version text;

-- Timestamp de quando canonical_role_id foi resolvido deterministicamente
-- (Camada 0 OU Camada 1). Nome coerente com uso em ambas as camadas.
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS canonical_resolved_at timestamptz;

-- Flag booleana — vaga passou por validação humana explícita
-- Usado para quórum da Camada 0 (1 âncora human_validated resolve)
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS human_validated boolean NOT NULL DEFAULT false;

-- Label denormalizado do canônico (evita JOIN repetido em queries de leitura)
-- Preenchido em todos os canais (precheck, Camada 1, LLM)
-- Mantido sincronizado com job_canonical_roles.label
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS canonical_role_label text;

-- Observabilidade causal da Camada 2
-- Número de sugestões que foram passadas ao LLM via suggested_roles
-- NULL = Camada 2 não rodou (pre-check ou Camada 1 resolveu)
-- 0 = Camada 2 rodou mas não encontrou sugestões
-- >0 = Camada 2 passou N sugestões ao LLM
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS layer_2_hint_count smallint;

-- v5.4: colunas para preservação do título/descrição brutos (pré-LLM).
-- Referenciadas pelo filtro N7 da Camada 0 (guard de título exige original_title
-- populado). Previamente listadas em §2.1 como pré-requisito mas nunca criadas
-- pela migration — gap identificado pelo Claude Code no pré-check §2.1.
--
-- original_title: título bruto vindo do scraper, antes de qualquer processamento
-- do LLM (que pode reescrever em description_curated). É a fonte de verdade
-- para overlap de tokens do title guard.
--
-- original_description: descrição bruta pré-LLM. Disponível hoje em
-- job_postings.requirements->>'description' (JSONB escrito por
-- lib/analysis/insert-jobs.ts:64). Backfill move para coluna dedicada abaixo.
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS original_title text;

ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS original_description text;

-- v5.4: backfill determinístico das 8.308 vagas existentes.
-- Todas as vagas atuais têm origin_mode='market' (100% Fluxo C/RapidAPI),
-- title é NOT NULL, requirements->>'description' tem fill rate 100%.
-- Nenhuma vaga de Fluxo A (admin manual) em produção — backfill é homogêneo.
-- Custo: subsegundo (8.308 linhas).
--
-- v5.5 (Claude-2 #3): UPDATEs separados por coluna, cada um idempotente.
-- Previne sobrescrever valores já populados por pipeline novo que rodou
-- entre a migration §5.1 e uma re-execução acidental deste script.
UPDATE job_postings
SET original_title = title
WHERE original_title IS NULL;

UPDATE job_postings
SET original_description = requirements->>'description'
WHERE original_description IS NULL;

-- v5.5 (Claude-2 #1): validação de fill-rate pós-UPDATE.
-- Se alguma linha permanecer NULL após o backfill, é indício de bug
-- (ex: requirements sem chave 'description' ou title NULL — improvável mas
-- verificável). Falhar aqui evita continuar com catálogo parcial.
DO $$
DECLARE
    null_title_count bigint;
    null_desc_count bigint;
    total_rows bigint;
BEGIN
    SELECT COUNT(*) INTO total_rows FROM job_postings;
    SELECT COUNT(*) INTO null_title_count FROM job_postings WHERE original_title IS NULL;
    SELECT COUNT(*) INTO null_desc_count FROM job_postings WHERE original_description IS NULL;

    IF null_title_count > 0 THEN
        RAISE EXCEPTION 'Backfill inline falhou: % vagas com original_title NULL de % totais',
            null_title_count, total_rows;
    END IF;

    IF null_desc_count > 0 THEN
        RAISE WARNING 'Backfill inline: % vagas com original_description NULL de % totais — investigar requirements JSONB sem chave description',
            null_desc_count, total_rows;
        -- NOTICE em vez de EXCEPTION: original_description pode legitimamente
        -- estar NULL se requirements JSONB não tiver a chave 'description'
        -- para alguma vaga antiga. Warning permite continuar mas sinaliza.
    END IF;

    RAISE NOTICE 'Backfill validado: % vagas, original_title 100%% populado, original_description com % NULL', total_rows, null_desc_count;
END $$;

COMMIT;
```

**Nota de lock:** esses `ALTER TABLE ... ADD COLUMN` com default NULL (exceto `human_validated` com DEFAULT false) fazem metadata-only change em PostgreSQL 11+ — lock curto, praticamente instantâneo em tabelas do tamanho atual (~8.6K). A adição de `human_validated NOT NULL DEFAULT false` exigiria rewrite da tabela em versões antigas, mas PostgreSQL 12+ faz isso sem rewrite para DEFAULT constante.

### 5.4 Nova tabela: `allowed_for_pre_resolution`

**v5.4 (GS-R6-12) — nome:** renomeada de `canonical_role_autodecision_whitelist` (v5.1-v5.3) para `allowed_for_pre_resolution`. Motivo: o nome antigo era confuso em dashboards — "whitelist de autodecisão" ambíguo (whitelist de quê? Camada 0? 1? promoção?). O nome novo descreve exatamente a função: canônicos aprovados para pré-resolução determinística pelas Camadas 0 e 1 antes do LLM. Troca feita antes do deploy — depois seria migração. Referências no código e na spec (14 ocorrências) atualizadas.

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS allowed_for_pre_resolution (
    canonical_role_id uuid NOT NULL REFERENCES job_canonical_roles(id) ON DELETE CASCADE,
    added_at timestamptz NOT NULL DEFAULT now(),
    added_by uuid NOT NULL,
    reason text,
    PRIMARY KEY (canonical_role_id)
);

COMMENT ON TABLE allowed_for_pre_resolution IS
    'Subconjunto de job_canonical_roles que podem ser escolhidos automaticamente por Camada 0/1 sem LLM. Canônicos active são sempre elegíveis; canônicos pending só são elegíveis se estiverem nesta tabela.';

CREATE INDEX IF NOT EXISTS idx_allowed_for_pre_resolution_added_at
    ON allowed_for_pre_resolution (added_at DESC);

COMMIT;
```

### 5.1.1 Ingestão popula `original_title` e `original_description` diretamente (v5.4)

A migration acima cobre o legado via backfill. Para vagas novas, o pipeline de ingestão em `lib/analysis/insert-jobs.ts` deve passar a popular as duas colunas diretamente em vez de depender do backfill periódico.

**Mudança mecânica em `insert-jobs.ts`** (linha ~53-64, onde acontece o INSERT):

```typescript
// Antes (v5.3):
const { data, error } = await supabase.from('job_postings').insert({
    title: rawJob.title,
    requirements: { description: descText },
    // ... outros campos
});

// Depois (v5.4):
const { data, error } = await supabase.from('job_postings').insert({
    title: rawJob.title,
    original_title: rawJob.title,           // v5.4 — mesma string de title na ingestão
    original_description: descText,          // v5.4 — mesma descrição bruta
    requirements: { description: descText }, // mantido para compatibilidade com callers legados
    // ... outros campos
});
```

**Observação sobre `requirements.description`:** continua sendo gravado durante a v5.4 para não quebrar callers que ainda leem de lá. Após validação de estabilidade pós-deploy (janela a ser definida pelo Onsly — ambiente ainda não está em produção assistida), `requirements.description` vira candidato a remoção em sprint futura — mas não é escopo da v5.4.

**Risco sinalizado:** se no futuro o Fluxo A (admin manual via `/admin/ingestor`) entrar em produção, o PR que adicionar esse fluxo **deve** popular `original_title`/`original_description` na inserção. Caso contrário, vagas de admin manual ficam com essas colunas NULL e são excluídas do pool de âncoras pela Camada 0 (filtro N7). Adicionar item ao checklist do PR do Fluxo A.

### 5.1.2 Trigger de imutabilidade de `original_title` e `original_description` (v5.5 — DeepSeek #1)

**Problema:** a Camada 0 depende de `original_title` da âncora refletir o título bruto **no momento da ingestão**. Se um admin editar o campo `title` de uma vaga via endpoint admin futuro, ou se um pipeline de re-ingestão/sincronização executar UPSERT sobrescrevendo todas as colunas, `original_title` pode acabar dessincronizado — quebrando o propósito do guard de título (que compara `candidateTitle` de vaga nova com `original_title` da âncora).

**Mitigação v5.6 (Gemini #1) — silenciamento em vez de EXCEPTION:** a v5.5 original levantava `RAISE EXCEPTION` quando `original_title` mudava. Em produção, isso quebraria **toda a transação de ingestão** caso um pipeline de sincronização (ex: re-sync com Gupy/Greenhouse após correção de título na origem) faça UPSERT natural e tente reescrever todas as colunas. Em vez de quebrar o batch de ingestão, a v5.6 usa **silenciamento defensivo**: o trigger restaura `NEW := OLD` e opcionalmente emite `RAISE NOTICE` para observabilidade. Preserva o valor original sem impacto na operação de UPDATE/UPSERT.

```sql
BEGIN;

CREATE OR REPLACE FUNCTION enforce_original_immutability()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    -- v5.6 (Gemini #1): silenciar mudança em vez de RAISE EXCEPTION.
    -- Preserva valor original sem quebrar batch de ingestão quando pipelines
    -- de sincronização fazem UPSERT natural reescrevendo todas as colunas.
    IF OLD.original_title IS NOT NULL
       AND NEW.original_title IS DISTINCT FROM OLD.original_title THEN
        RAISE NOTICE 'Tentativa de alterar original_title silenciada na vaga %: de % para %',
            OLD.id, OLD.original_title, NEW.original_title;
        NEW.original_title := OLD.original_title;  -- restaura valor original
    END IF;

    IF OLD.original_description IS NOT NULL
       AND NEW.original_description IS DISTINCT FROM OLD.original_description THEN
        RAISE NOTICE 'Tentativa de alterar original_description silenciada na vaga %',
            OLD.id;
        NEW.original_description := OLD.original_description;
    END IF;

    RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS trigger_enforce_original_immutability ON job_postings;
DROP TRIGGER IF EXISTS trg_enforce_original_immutability ON job_postings;

-- v5.6 (Claude Code): nome alinhado à convenção trg_* do projeto
CREATE TRIGGER trg_enforce_original_immutability
    BEFORE UPDATE ON job_postings
    FOR EACH ROW
    WHEN (OLD.original_title IS NOT NULL OR OLD.original_description IS NOT NULL)
    EXECUTE FUNCTION enforce_original_immutability();

COMMENT ON FUNCTION enforce_original_immutability IS
    'v5.6 (DeepSeek #1 + Gemini #1) — garante imutabilidade de original_title e original_description após a primeira gravação. Silencia mudanças em vez de bloquear, para não quebrar batches de ingestão que fazem UPSERT natural. RAISE NOTICE emitido para observabilidade via logs do Postgres.';

COMMIT;
```

**Casos cobertos:**
- INSERT inicial com `original_title` populado → permitido (OLD é NULL, trigger WHEN não dispara)
- UPDATE que altera só `title` → permitido, `original_title` intacto
- UPDATE que tenta alterar `original_title` → **silenciado**, valor original preservado, NOTICE no log
- UPSERT de re-sincronização reescrevendo todas as colunas → `original_title` protegido sem quebrar transação
- Backfill inline da §5.1 (NULL → valor) → permitido (OLD.original_title IS NULL)

**Semântica declarada:** `original_title` é o título bruto do momento da ingestão, nunca deve ser alterado após a inserção. O título visível no produto (`title`) pode ser alterado livremente sem afetar `original_title`. Admin que quiser "corrigir" um título errado edita `title`; a âncora mantém o original para fins de guard de título.

### 5.2 Índices

```sql
BEGIN;

-- Suporta query principal da Camada 0
-- Composto em (description_hash, prompt_structure_version, curated_at DESC)
-- Permite filtragem combinada sem seq scan
CREATE INDEX IF NOT EXISTS idx_job_postings_description_hash_precheck
    ON job_postings (description_hash, prompt_structure_version, curated_at DESC)
    WHERE curation_status = 'curated'
      AND description_hash IS NOT NULL;

-- Suporta filtro human_validated para quórum Camada 0 com 1 âncora
CREATE INDEX IF NOT EXISTS idx_job_postings_human_validated
    ON job_postings (canonical_role_id)
    WHERE human_validated = true AND curation_status = 'curated';

-- Suporta queries admin de curation_source distribuição
CREATE INDEX IF NOT EXISTS idx_job_postings_curation_source
    ON job_postings (curation_source)
    WHERE curation_source IS NOT NULL;

COMMIT;
```

### 5.3 Backfill de `description_hash`, `prompt_structure_version`, `prompt_content_version`

Executar após migração de schema **e antes de qualquer rollout de tráfego**. SHA-256 é rápido — tempo estimado para ~8.034 vagas: 40-60 segundos.

Implementação em `scripts/backfill-job-description-hash.ts` (Seção 7.6).

### 5.5 Coluna `prompt_version` em `job_canonical_roles` (v5.4 pós-pt.15)

**Contexto:** a sessão pt. 15 (sprint de governança de taxonomia) descobriu empiricamente que parte significativa das classificações erradas em `canonical_skills` vem de drift de prompt — canônicos criados sob uma versão do prompt, regras adicionadas depois, canônicos existentes nunca reaferidos. Evidência: 4 cross-type de skills foram classificados entre 14:49-14:52 UTC de 02/04/2026 sob prompt `8cb04a9`; a regra que teria classificado corretamente (Regra 5, "atendimento + contexto = hybrid") entrou às 18:30 UTC do mesmo dia. 3h40 de janela, nenhum mecanismo de reaferição.

**Problema análogo em `job_canonical_roles`:** hoje existem ~9.000 canônicos de roles ativos+pending, **nenhum deles** carrega a versão do prompt que os gerou. Se a v5.4 deploya prompt novo (reescrita do `SYSTEM_PROMPT.ts` via PR1), todos os canônicos existentes continuam classificados sob prompt antigo e ninguém tem como saber quais.

**Mitigação v5.4:** adicionar coluna `prompt_version` em `job_canonical_roles` **agora**, durante a migration §5.1. O pipeline de curadoria passa a gravar `prompt_version` em todo INSERT novo de canônico (via `upsertCanonicalRole` em §7.11). Canônicos existentes recebem `prompt_version = 'legacy'` no backfill — marcador explícito de "classificado sob prompt pré-v5.4, pode precisar de reaferição".

```sql
-- Parte da migration §5.1, mesmo BEGIN/COMMIT
ALTER TABLE job_canonical_roles
    ADD COLUMN IF NOT EXISTS prompt_version text;

-- Backfill: todos os canônicos existentes viram 'legacy'
UPDATE job_canonical_roles
SET prompt_version = 'legacy'
WHERE prompt_version IS NULL;

-- NOT NULL enforcement após backfill
ALTER TABLE job_canonical_roles
    ALTER COLUMN prompt_version SET NOT NULL;

ALTER TABLE job_canonical_roles
    ALTER COLUMN prompt_version SET DEFAULT 'unspecified';
-- DEFAULT 'unspecified' para proteger contra INSERTs do pipeline antigo
-- que não foi atualizado. Monitoramento pós-deploy deve alertar se
-- aparecer canônico com 'unspecified' — é sinal de regressão.

-- v5.6 (ChatGPT G5): documentar os 3 tiers semânticos da coluna
-- para a sprint pt. 15 (e qualquer auditoria futura) entender a triagem.
COMMENT ON COLUMN job_canonical_roles.prompt_version IS
    'v5.6 — coluna com 3 tiers semânticos:
     1) ''legacy'' — canônico criado ANTES da v5.6 (backfill). Risco alto de
        ter sido classificado sob prompt imaturo. Prioridade da pt. 15 para
        reaferição manual ou batch recuration.
     2) ''unspecified'' — INSERT do pipeline novo SEM o valor de
        PROMPT_STRUCTURE_VERSION setado. Indica BUG: env var não injetada,
        bootstrap mal-feito, ou caller esqueceu de passar. Query de
        monitoramento §8.5.1 alerta se aparecer em janela 24h.
     3) hash real (ex: ''9a68cc80'') — canônico criado pela v5.6+ sob prompt
        com versão rastreável. Status ideal.
     Triagem da pt. 15: filtro útil é prompt_version != ''legacy'' AND
     prompt_version != ''unspecified'' para encontrar canônicos confiáveis.';
```

**Mudança em `upsertCanonicalRole` (§7.11):** ao criar canônico novo, popular `prompt_version` com o valor do `PROMPT_STRUCTURE_VERSION` em uso naquele momento. v5.5 detalha o código exato e adiciona contador de regressão.

**v5.5 (Claude-2 #4) — implementação completa do upsert:**

```typescript
// lib/pipeline/persist-curation.ts — upsertCanonicalRole
// Localização: dentro do INSERT em job_canonical_roles quando canônico novo
// é detectado (source='llm_extractor'). O INSERT existente precisa ganhar a
// coluna prompt_version com o valor atual em uso pelo pipeline.

const { data: newCanonical, error: insertError } = await supabase
    .from('job_canonical_roles')
    .insert({
        label: normalizedLabel,
        slug: slugify(normalizedLabel),
        status: 'pending',
        source: 'llm_extractor',
        distinct_sources_count: 1,
        // ... campos existentes ...
        // v5.5 (Claude-2 #4) — rastreabilidade de drift
        prompt_version: PROMPT_STRUCTURE_VERSION,
    })
    .select()
    .single();

// v5.5 (Claude-2 #4) — detecção de regressão pós-deploy
// Se por algum motivo PROMPT_STRUCTURE_VERSION estiver undefined ou null no
// momento do INSERT, a coluna cai no DEFAULT 'unspecified' da migration §5.5.
// Isso é sinal de bug: prompt não foi carregado, env var não foi injetada,
// ou hot-reload falhou. Contador específico permite alertar via dashboard.
if (!PROMPT_STRUCTURE_VERSION) {
    if (counters) {
        counters.canonicalCreatedWithUnspecifiedPromptVersion++;
    }
    await supabase.from('events').insert({
        event_name: 'prompt_version_unspecified_on_canonical_insert',
        resource_type: 'job_canonical_roles',
        resource_id: newCanonical.id,
        actor: 'pipeline',
        actor_id: SYSTEM_USER_ID,
        reason: 'PROMPT_STRUCTURE_VERSION undefined no momento do INSERT — investigar bootstrap do pipeline',
        metadata: {
            canonical_label: normalizedLabel,
            env_NODE_ENV: process.env.NODE_ENV,
        }
    });
}
```

**Contador novo em `RunCounters`:**

```typescript
interface RunCounters {
    // ... campos existentes v5.4 ...
    llmOutputQuarantined: number;  // v5.4 (GS-R6-1)
    canonicalCreatedWithUnspecifiedPromptVersion: number;  // v5.5 (Claude-2 #4)
}
```

**Inicialização em `createRunCounters()`:** `canonicalCreatedWithUnspecifiedPromptVersion: 0`.

**Critério de aceite novo:** teste unitário valida que INSERT em `job_canonical_roles` inclui `prompt_version` com o valor correto do env. Query de monitoramento em §8.5.1: `SELECT COUNT(*) FROM job_canonical_roles WHERE prompt_version = 'unspecified' AND created_at > NOW() - INTERVAL '24 hours'` deve retornar 0 — qualquer resultado > 0 é regressão.

**Implicação para a sprint pt. 15:** com `prompt_version` populado em todo canônico novo a partir da v5.4, a pt. 15 pode filtrar canônicos "legacy" (criados antes da v5.4) como prioridade de reaferição. Canônicos `prompt_version = 'v1'` (ou versão vigente) vêm sob prompt maduro e têm risco menor. Sem essa coluna, a pt. 15 teria que cruzar `created_at` com git log do prompt — análise cara e aproximada.

**Sem isso, qual o gap?** O mesmo problema dos 4 cross-type de skills reaparece em roles daqui a alguns meses. A v5.4 deploya prompt novo, canônicos novos são criados sob o prompt novo, mas não há como distinguir de canônicos antigos no banco. Daqui a 2 meses, quando houver nova rodada de ajuste de regras, a pt. 15 vai ter que rodar a mesma investigação arqueológica de novo.

### 5.6 Coluna `curation_source` + endpoint admin de remap + trigger N4 (v5.5 — greenfield)

**Contexto:** Claude Code confirmou em 24/04/2026 que o codebase atual **não tem** coluna `curation_source` em `job_postings`, **não tem** endpoint admin de remap que altere `canonical_role_id` de uma vaga específica, e **não tem** lógica que sete `curation_source='manual_remap'`. A v5.4 original propunha o trigger N4 reagindo a esse valor, mas sem criar os pré-requisitos — trigger nasceria inerte.

**Decisão v5.5 (Decisão 4 do Onsly):** implementar os 3 pré-requisitos nesta sprint como greenfield. Não há débito técnico — é funcionalidade nova.

#### 5.6.1 Coluna `curation_source` em `job_postings`

**Já coberta** pela migration §5.1 da v5.4/v5.5 (a coluna é uma das 10 listadas). O CHECK constraint original da §5.1 lista os valores válidos:

```sql
-- Fragmento já presente na §5.1 (v5.1+), replicado aqui para referência:
-- v5.6 (Claude Code): alinhado com §5.1 que inclui 'quarantined_llm_output'
ALTER TABLE job_postings
    ADD COLUMN IF NOT EXISTS curation_source text
    CHECK (
        curation_source IS NULL OR
        curation_source IN (
            'precheck_description_hash',
            'layer_1_domain_synonyms',
            'llm_direct',
            'llm_equivalences_redirect',
            'manual_remap',
            'quarantined_llm_output'
        )
    );
```

**v5.5:** o valor `'manual_remap'` já está no CHECK da v5.4. O que falta é o endpoint que o seta e o trigger N4 que reage a ele.

#### 5.6.2 Endpoint admin de remap — novo arquivo

**Path:** `app/api/admin/jobs/[id]/remap/route.ts`

**Função:** permitir que admin altere `canonical_role_id` de uma vaga específica (correção manual) setando `curation_source='manual_remap'` no mesmo UPDATE atômico. Isso dispara o trigger N4 que revoga `human_validated` da vaga se estava populada.

**v5.6 (ChatGPT C2+C3+C4 / Claude Code) — reescrita completa:** a v5.5 tinha 6 bugs no código proposto:
1. `createAdminSupabaseClient` de `@/lib/supabase/admin` — função inexistente (nome inventado)
2. `requireAdminAuth` de `@/lib/auth/admin-guard` — arquivo e função inexistentes (helper imaginário)
3. `SYSTEM_USER_ID` de `@/lib/constants` — path errado (o real é `@/lib/pipeline/constants`)
4. Select em `archived_at` em `job_canonical_roles` — coluna inexistente (só existe em `profiles`)
5. `actor: 'admin'` viola CHECK constraint `events_actor_check CHECK (actor IN ('system', 'pipeline', 'human'))`
6. `actor_id: auth.userId` com shape que não existe (shape real é `authData.user.id`)

Claude Code auditou 20+ endpoints admin reais do projeto e confirmou:
- Padrão de cliente admin: `createAdminServerClient` de `@/lib/supabase-server`
- Padrão de auth: **inline de 6 linhas replicado em 28+ endpoints** — não existe helper. A convenção do projeto é cookie client para validar sessão + admin client para queries. Não criar helper agora introduziria inconsistência com o resto do codebase.
- Status de canônicos em vez de `archived_at`: usar `status IN ('deprecated', 'rejected')` como guard.

Template corrigido seguindo convenção do projeto:

```typescript
// app/api/admin/jobs/[id]/remap/route.ts
// v5.6 — greenfield, não havia endpoint equivalente no codebase
// Padrão de auth e cliente alinhados com os 28+ endpoints admin existentes

import { createAdminServerClient } from '@/lib/supabase-server';
import { NextRequest, NextResponse } from 'next/server';

interface RemapRequestBody {
    new_canonical_role_id: string;
    reason: string;  // texto livre — por que o remap está sendo feito
}

export async function POST(
    req: NextRequest,
    { params }: { params: { id: string } }
): Promise<NextResponse> {
    // ── Auth inline (padrão do projeto, replicado em 28+ endpoints admin) ──
    const cookieSupabase = await import('@/lib/supabase/server').then(m => m.createClient());
    const { data: authData } = await (await cookieSupabase).auth.getUser();
    if (!authData?.user) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });

    const supabaseAdmin = createAdminServerClient();
    const { data: userData } = await supabaseAdmin
        .from('users')
        .select('user_type')
        .eq('id', authData.user.id)
        .single();
    if (userData?.user_type !== 'admin') {
        return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
    }

    // ── Validação do body ──────────────────────────────────────
    const body = (await req.json()) as RemapRequestBody;
    const { new_canonical_role_id, reason } = body;

    if (!new_canonical_role_id || !reason || reason.trim().length < 10) {
        return NextResponse.json(
            { error: 'new_canonical_role_id e reason (mínimo 10 chars) são obrigatórios' },
            { status: 400 }
        );
    }

    // ── Validar canônico destino (usar status em vez de archived_at) ──
    // v5.6 (ChatGPT C3): archived_at não existe em job_canonical_roles.
    // Guard contra canônicos inativos via status ∈ {deprecated, rejected}.
    const { data: destCanonical, error: destErr } = await supabaseAdmin
        .from('job_canonical_roles')
        .select('id, label, status')
        .eq('id', new_canonical_role_id)
        .single();

    if (destErr || !destCanonical) {
        return NextResponse.json({ error: 'Canônico destino não encontrado' }, { status: 404 });
    }

    if (destCanonical.status === 'deprecated' || destCanonical.status === 'rejected') {
        return NextResponse.json(
            { error: `Canônico destino tem status=${destCanonical.status}. Remap rejeitado.` },
            { status: 400 }
        );
    }

    // ── Ler estado atual da vaga para evento forense ──────────
    const { data: currentJob, error: readErr } = await supabaseAdmin
        .from('job_postings')
        .select('id, canonical_role_id, canonical_role_label, human_validated, curation_source')
        .eq('id', params.id)
        .single();

    if (readErr || !currentJob) {
        return NextResponse.json({ error: 'Vaga não encontrada' }, { status: 404 });
    }

    // ── UPDATE atômico (trigger N4 reage em BEFORE UPDATE) ────
    const { error: updateErr } = await supabaseAdmin
        .from('job_postings')
        .update({
            canonical_role_id: new_canonical_role_id,
            canonical_role_label: destCanonical.label,
            curation_source: 'manual_remap',  // v5.6 — gatilho do trigger N4
            canonical_resolved_at: new Date().toISOString(),
        })
        .eq('id', params.id);

    if (updateErr) {
        return NextResponse.json({ error: updateErr.message }, { status: 500 });
    }

    // ── Evento forense imutável ────────────────────────────────
    // v5.6 (ChatGPT C4): actor='human' (não 'admin') — CHECK constraint
    //                    events_actor_check só aceita {system, pipeline, human}.
    // v5.6: actor_id é o user real do admin logado (authData.user.id),
    //       não SYSTEM_USER_ID — rastreabilidade de quem executou a ação.
    await supabaseAdmin.from('events').insert({
        event_name: 'job_canonical_remapped',
        resource_type: 'job_posting',
        resource_id: params.id,
        actor: 'human',                  // ← NÃO 'admin' (viola CHECK)
        actor_id: authData.user.id,      // ← user real logado
        previous_state: {
            canonical_role_id: currentJob.canonical_role_id,
            canonical_role_label: currentJob.canonical_role_label,
            human_validated: currentJob.human_validated,
            curation_source: currentJob.curation_source,
        },
        new_state: {
            canonical_role_id: new_canonical_role_id,
            canonical_role_label: destCanonical.label,
            curation_source: 'manual_remap',
            human_validated: false,  // revogado pelo trigger N4
        },
        reason,
        metadata: {
            revoked_human_validated: currentJob.human_validated === true,
        }
    });

    return NextResponse.json({
        success: true,
        job_id: params.id,
        new_canonical: {
            id: new_canonical_role_id,
            label: destCanonical.label,
        },
        human_validated_revoked: currentJob.human_validated === true,
    });
}
```

**Autorização:** seguindo convenção do projeto — auth inline de 6 linhas (cookie client para validar sessão + admin client para consultar `user_type`). Admin logado passa, qualquer outro 403. Convenção confirmada em 28+ endpoints admin existentes.

#### 5.6.3 Trigger N4 — revogação de `human_validated` em `manual_remap`

**v5.6 (ChatGPT C4 / Claude Code) — correções críticas:**
1. `actor='trigger'` violava CHECK `events_actor_check` (só aceita `{system, pipeline, human}`). Correto: `'system'`.
2. Evento forense não capturava `canonical_role_id` antigo — rastreabilidade perdida. Adicionado ao `previous_state`.
3. Nome do trigger alinhado com convenção `trg_*` do projeto (14 triggers existentes no schema usam esse prefixo).

**Path da migration:** `docs/migrations/20260424_02_revoke_human_validated_trigger.sql` (data atual, sequencial novo).

```sql
BEGIN;

CREATE OR REPLACE FUNCTION revoke_human_validated_on_manual_remap()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    -- Reage apenas quando curation_source muda para 'manual_remap'
    -- E a vaga tinha human_validated=true antes da mudança
    IF NEW.curation_source = 'manual_remap'
       AND (OLD.curation_source IS NULL OR OLD.curation_source != 'manual_remap')
       AND OLD.human_validated = true
    THEN
        NEW.human_validated := false;

        -- Evento forense dentro da trigger
        -- v5.6 (ChatGPT C4): actor='system' (não 'trigger' que viola CHECK)
        -- v5.6 (Claude Code): canonical_role_id antigo em previous_state
        --                     para rastreabilidade do remap no histórico forense
        INSERT INTO events (
            event_name,
            resource_type,
            resource_id,
            actor,
            actor_id,
            reason,
            previous_state,
            new_state
        ) VALUES (
            'human_validated_revoked_by_remap',
            'job_posting',
            NEW.id,
            'system',                                         -- ← NÃO 'trigger' (viola CHECK)
            '00000000-0000-0000-0000-000000000001',           -- SYSTEM_USER_ID
            'Trigger N4: curation_source mudou para manual_remap em vaga com human_validated=true',
            jsonb_build_object(
                'human_validated', OLD.human_validated,
                'canonical_role_id', OLD.canonical_role_id    -- ← rastreabilidade do remap
            ),
            jsonb_build_object(
                'human_validated', false,
                'canonical_role_id', NEW.canonical_role_id
            )
        );
    END IF;

    RETURN NEW;
END;
$$;

-- v5.6 (Claude Code): nome do trigger alinhado à convenção trg_* do projeto
DROP TRIGGER IF EXISTS trigger_revoke_human_validated_before_update ON job_postings;
DROP TRIGGER IF EXISTS trg_revoke_human_validated_on_remap ON job_postings;

CREATE TRIGGER trg_revoke_human_validated_on_remap
    BEFORE UPDATE ON job_postings
    FOR EACH ROW
    WHEN (
        NEW.curation_source IS DISTINCT FROM OLD.curation_source
        AND NEW.curation_source = 'manual_remap'
    )
    EXECUTE FUNCTION revoke_human_validated_on_manual_remap();

COMMENT ON FUNCTION revoke_human_validated_on_manual_remap IS
    'v5.6 (Decisão 4 — greenfield): quando admin faz remap manual de uma vaga previamente human_validated, a validação humana é revogada automaticamente. Remap implica que a validação anterior estava errada — manter human_validated=true propagaria o erro pela Camada 0.';

COMMIT;
```

**Casos cobertos:**
- Admin remapeia vaga sem `human_validated` → trigger não dispara (WHEN filter)
- Admin remapeia vaga com `human_validated=true` → `human_validated` vira false + evento gravado
- Admin remapeia mesma vaga 2x seguidas → segundo remap não dispara (OLD.curation_source já é 'manual_remap')

**Caso não coberto (débito consciente):** admin edita `canonical_role_id` via SQL direto sem passar pelo endpoint → `curation_source` permanece no valor anterior → trigger não dispara → `human_validated` não é revogado. **Mitigação documentada no runbook:** admin é instruído a usar sempre o endpoint. Se SQL direto for usado em emergência, admin precisa também setar `curation_source='manual_remap'` explicitamente no UPDATE (DeepSeek #11 — item 23).

#### 5.6.4 Referências cruzadas

A v5.4 original mencionava trigger N4 na §7.15 como planejada. v5.5 substitui aquela referência por ponteiro para §5.6.3 (implementação completa). A §7.15 não é mais necessária como seção separada.

---

## 6. Patches de conteúdo

### 6.1 `domain_synonyms.json` — bump 2.0 → 3.0 com 68 entradas novas

Entradas a adicionar ao objeto `data`, preservando as **113 entradas existentes** intactas. Bump do campo `version` de `"2.0"` para `"3.0"`. Total final: 181 entradas.

**Contagem por categoria: 23 + 14 + 7 + 14 + 10 = 68 entradas novas.**

**Categoria A — Hierarquia bilíngue (23 entradas novas)**

```json
"chief financial officer": "CFO",
"chief technology officer": "CTO",
"chief executive officer": "CEO",
"chief operating officer": "COO",
"chief marketing officer": "CMO",
"chief customer officer": "Diretor de Customer Success",
"head of finance": "Diretor de Finanças",
"head of fp&a": "Diretor de Finanças",
"head of engineering": "Gerente de Engenharia",
"head of it": "Gerente de TI",
"head of operations": "Gerente de Operações",
"head of customer success": "Diretor de Customer Success",
"head of marketing": "Diretor de Marketing",
"head of product": "Diretor de Produto",
"finance manager": "Gerente de Finanças",
"engineering manager": "Gerente de Engenharia",
"software engineering manager": "Gerente de Engenharia",
"it director": "Gerente de TI",
"finance director": "Diretor de Finanças",
"operations director": "Gerente de Operações",
"marketing director": "Diretor de Marketing",
"marketing manager": "Gerente de Marketing",
"product manager": "Gerente de Produto"
```

**Decisão explícita de rejeitar siglas de 3-4 letras:** `"cto"`, `"ceo"`, `"coo"`, `"cmo"`, `"cfo"`, `"cco"` INTENCIONALMENTE não entram. Rationale: word boundary mitiga falsos positivos, mas CCO tem ambiguidade tripla real (Chief Customer / Chief Commercial / Chief Content Officer); por uniformidade de política, rejeitamos todas as 6 siglas e aceitamos apenas as formas por extenso. Apenas canônicos DESTINO estão em sigla (CTO, CEO, COO, CMO, CFO) — as siglas entram no catálogo via seed da §2.3, não via patch do JSON.

**Categoria B — Qualificador Operations e domínios funcionais (14 entradas novas)**

```json
"sales ops": "Analista de Operações de Vendas",
"revenue operations": "Analista de Operações de Vendas",
"revops": "Analista de Operações de Vendas",
"people operations": "Analista de Recursos Humanos",
"people ops": "Analista de Recursos Humanos",
"hr operations": "Analista de Recursos Humanos",
"ad operations": "Analista de Marketing Digital",
"ad ops": "Analista de Marketing Digital",
"marketing operations": "Analista de Marketing",
"marketing ops": "Analista de Marketing",
"payments operations": "Analista de Operações Financeiras",
"financial operations": "Analista de Operações Financeiras",
"tax analyst": "Analista Tributário",
"tax specialist": "Analista Tributário"
```

**Débito conhecido em `hr operations`:** mapeamento para `Analista de Recursos Humanos` é um achatamento funcional (HR Operations em mercado é função específica de payroll/benefícios/compliance trabalhista). O canônico `Analista de Departamento Pessoal` hoje tem volume baixo e o Sonnet pode afinar via descrição. Reavaliar em sprint separada se volume de HR Operations crescer.

**Categoria C — Variantes de Customer Service/Experience (7 entradas novas)**

```json
"customer service representative": "Analista de Customer Service",
"customer service specialist": "Analista de Customer Service",
"customer service officer": "Analista de Customer Service",
"client service specialist": "Analista de Customer Service",
"customer care specialist": "Analista de Customer Service",
"customer experience specialist": "Analista de Customer Experience",
"cx specialist": "Analista de Customer Experience"
```

**Categoria D — Procurement, Compliance, Projetos e Cargos US ambíguos (14 entradas novas)**

```json
"compliance officer": "Analista de Compliance",
"compliance specialist": "Analista de Compliance",
"procurement analyst": "Analista de Compras",
"procurement specialist": "Analista de Compras",
"recruitment specialist": "Analista de Recrutamento e Seleção",
"recruiting analyst": "Analista de Recrutamento e Seleção",
"talent acquisition specialist": "Analista de Recrutamento e Seleção",
"social media analyst": "Analista de Marketing Digital",
"project administrator": "Coordenador de Projetos",
"project coordinator": "Coordenador de Projetos",
"sdr": "SDR",
"sales development representative": "SDR",
"bdr": "BDR",
"business development representative": "BDR"
```

**v5.5 — SDR e BDR como canônicos distintos:** a v5.4 original apontava ambas as siglas para o mesmo canônico `Representante de Pré-Venda`. A v5.5 reflete a nova taxonomia decidida no Anexo B (§2.4 cravado): `Representante de Pré-Venda` é merged para `SDR` (UUID `d780aacf` absorvido pelo UUID do SDR active), e `Analista de Desenvolvimento de Negócios` é renomeado para `BDR` (UUID `b2aa7f8e` preservado). Isso corrige a diferença semântica real entre SDR (inbound, qualificação de leads) e BDR (outbound, prospecção ativa) que o catálogo agora honra. Ambos os canônicos são `active` e entram em `allowed_for_pre_resolution`. Word boundary previne falsos positivos em telecom (SDR = Software Defined Radio).

**Entradas intencionalmente excluídas do patch (ambiguidade contextual):**

`"solution consultant"`, `"solutions consultant"`, `"partner sales director"`, `"account director"`, `"alliance director"`, `"technical account manager"`, `"customer success manager"`, `"user experience manager"` — ambiguidade delivery vs pre-sales, IC vs manager real. Sonnet decide com apoio das regras A-E.

**Entradas já existentes no arquivo v2.0 (não contadas no patch):**

`"project manager"`, `"operations manager"`, `"compliance analyst"`, `"talent acquisition"`, `"recruiter"`, `"field service technician"`, `"field engineer"`, `"operations analyst"`, `"sales operations analyst"`, `"training specialist"`, `"l&d"` e outras — já estavam no arquivo, retiradas do patch após validação cruzada contra as 113 entradas existentes.

**Categoria E — Sinônimos decorrentes das operações de taxonomia cravadas no Anexo B (v5.5 — 10 entradas novas)**

```json
"representante de pré-venda": "SDR",
"representante de pre-venda": "SDR",
"pré-vendas": "SDR",
"pre-vendas": "SDR",
"analista de pré-vendas": "SDR",
"analista de pre-vendas": "SDR",
"business development associate": "BDR",
"bda": "BDR",
"analista de desenvolvimento de negócios": "BDR",
"operações de vendas": "Analista de Operações de Vendas"
```

**Justificativa da Category E:** como o merge (Representante de Pré-Venda → SDR) e os renames (Desenvolvimento de Negócios → BDR, Sales Operations → Operações de Vendas) executam antes do PR1 (v5.5 — ordem unificada de deploy), essas entradas cobrem as variações PT mais comuns dos títulos brutos que agora devem apontar aos canônicos renomeados. Sem elas, títulos como "Analista de Pré-Vendas" cairiam no LLM em vez de resolver na Camada 1.

**Contagem v5.5:** categorias A (23) + B (14) + C (7) + D (14) + E (10) = **68 entradas novas**. Total final: 113 existentes + 68 novas = **181 entradas**.

### 6.2 `equivalences.json` — papel pós-LLM preservado

**Nenhuma mudança de conteúdo na v5.1.** Continua operando como hoje: lookup pós-LLM via exact match em `upsertCanonicalRole:141-164`, gerando eventos `canonical_redirected_by_equivalence` ou `canonical_ambiguous_equivalence`.

**v5.4 (GS-R6-7) — precedência Camada 1 vs Camada 4:** vagas que passam pela Camada 1 e têm `pre_resolved_canonical_label` populado **pulam `upsertCanonicalRole` completamente** pelo override server-side de `persistCuratedJob` (§7.11). Isso significa que entradas em `equivalences.json` cujo **destino** é um canônico também resolvido via `domain_synonyms.json` ficam inativas para o caminho pré-resolvido.

Exemplo: se `equivalences.json` tem `"Gerente Financeiro": "Gerente de Finanças"` e `domain_synonyms.json` v3.0 tem `"head of finance"` → `"Gerente de Finanças"`, vagas com título "Head of Finance" passam pela Camada 1 e vão direto para "Gerente de Finanças" — a regra de redirecionamento "Gerente Financeiro → Gerente de Finanças" em equivalences só opera se o LLM retornar literalmente "Gerente Financeiro", o que acontece apenas quando a Camada 1 não resolveu antes.

**Ação operacional:** auditar `equivalences.json` antes do PR2 procurando entradas com origem/destino presentes em `domain_synonyms.json` ou em `allowedForPreResolution`. Se houver conflito de semântica (não apenas sobreposição), reavaliar se a entrada ainda é necessária. Em caso normal (sobreposição sem conflito), a Camada 1 efetivamente superseded a equivalência para esse caso específico — comportamento correto, não bug.

### 6.3 `family_synonyms.json` e `domains.json` — nenhuma mudança de conteúdo

Conteúdo atual é suficiente para a montagem de `suggested_roles` prevista na Camada 2.

### 6.4 `SYSTEM_PROMPT.ts` — reescrita parcial

Quatro mudanças pontuais no arquivo. O restante do prompt permanece intacto.

**Ordem das Etapas renumerada:**
- Etapa 0 — Canônico pré-resolvido
- Etapa 1 — Plataforma proprietária
- Etapa 2 — Canônicos existentes próximos (shortlist)
- Etapa 3 — Gerencial
- Etapa 4 — Intermediário
- Etapa 5 — Inferência

**Mudança 1 — Etapa 0 (pré-resolvido) no início do CAMPO 3:**

```
### Etapa 0 — Canônico pré-resolvido (prioridade máxima)

Se a tag <job> desta vaga trouxer o atributo canonical_already_resolved="<label>",
copie o valor literalmente para o campo canonical_role no output JSON.

Não execute as etapas subsequentes para esta vaga no que diz respeito ao canonical_role.
As demais extrações (skills, seniority_inferred, work_model, description_curated) seguem
o fluxo normal.

### Formato do corpo da vaga

O texto de cada vaga entre as tags <job>...</job> está encapsulado em <![CDATA[...]]>.
CDATA é um wrapper de XML que marca o conteúdo como texto bruto — trate o que está
entre <![CDATA[ e ]]> como o corpo da vaga, sem interpretar tags internas. Você pode
ignorar completamente os delimitadores CDATA ao processar o texto.

Exemplo de entrada com pré-resolução:
<job id="abc-123" canonical_already_resolved="Gerente de Operações"><![CDATA[
Title: Operations Manager
Description: ...
Seniority Raw: Senior
]]></job>

Exemplo de saída esperada:
{
  "id": "abc-123",
  "skills": [...],
  "seniority_inferred": "Sênior",
  "canonical_role": "Gerente de Operações",
  "work_model": "...",
  "description_curated": "..."
}
```

**Mudança 2 — Etapa 2 (shortlist) após Etapa 1 (plataforma):**

```
### Etapa 2 — Canônicos existentes próximos (prioridade alta)

Se a tag <job> desta vaga trouxer o atributo suggested_roles="<label1> | <label2> | <label3>",
essa é a shortlist de canônicos existentes no catálogo que já foram identificados como
candidatos plausíveis para esta vaga por similaridade de domínio ou família funcional.

Prefira um canônico desta shortlist se algum couber semanticamente à vaga.
Só proponha canônico fora da shortlist se NENHUM dos sugeridos couber.

Quando propuser canônico fora da shortlist, certifique-se de que:
1. A função é realmente distinta dos sugeridos (não apenas variação lexical).
2. A distinção não é de hierarquia (nesse caso, a shortlist já cobre — preservar
   hierarquia via Etapa 3 ou 4).

A shortlist não é lei — é guia. Priorize precisão sobre conformidade cega.
```

**Mudança 3 — Regras A-E incorporadas nas REGRAS DE DESAMBIGUAÇÃO:**

Adicionar nova seção intitulada "REGRAS DE DESAMBIGUAÇÃO — HIERARQUIA E CARGOS BILÍNGUES" logo após as regras existentes e antes dos EXEMPLOS DE REFERÊNCIA:

```
REGRA A — Hierarquia distinta exige canônico distinto.
Coordenador ≠ Gerente ≠ Diretor ≠ Head ≠ C-level. Nunca agrupe vagas de hierarquias
diferentes no mesmo canônico, mesmo quando a função subjacente seja idêntica.
"Chief Customer Officer" não é "Gerente de Customer Success" — é "Diretor de Customer
Success". Se o canônico do nível hierárquico identificado não existir no catálogo de
referência, sinalize canonical_role com o canônico da função no nível mais próximo
disponível, mas nunca colapse níveis hierárquicos distintos em um mesmo canônico.

REGRA B — Função primária identificada por tokens de domínio, não por posição.
Em títulos em português, a primeira palavra usualmente identifica a função principal:
"Analista de Projetos e Customer Success" é Analista de Projetos. Em títulos em inglês,
o padrão é invertido: "Customer Success Operations Manager" tem função principal
"Operations Manager" modificada pelo domínio "Customer Success" — não é Customer
Success Manager. Use os tokens do título para identificar a função primária,
considerando ambos os padrões (PT antepõe função, EN pospõe).

REGRA C — Delivery sobre pre-sales quando há ambiguidade.
"Solution Consultant", "Solutions Engineer", "Technical Consultant" em descrições que
mencionam implementação, customização, integração, onboarding ou entrega de projeto
são funções de delivery (Consultor, Engenheiro de Implantação). Quando a descrição
enfatiza venda, demonstração, discovery ou proposta comercial, são funções de pre-sales
(Consultor Pré-Vendas). Na ausência de sinal claro, assuma delivery — é o caso mais
comum no catálogo brasileiro.

REGRA D — Título IC prevalece sobre seniority hierárquico aparente.
"Senior Developer", "Lead Engineer" com responsabilidade técnica individual são
Desenvolvedor Sênior, Desenvolvedor Lead — NÃO Gerente de Engenharia. Títulos IC
(Individual Contributor) não mudam de canônico funcional por senioridade. Só migre
para canônico gerencial quando a descrição mencionar explicitamente gestão de pessoas,
orçamento, headcount, processo de contratação, avaliação de desempenho, mentoring
estruturado com cadência formal.

REGRA E — Cargos US sem equivalente BR mantêm o termo em inglês.
"Product Owner", "Scrum Master", "DevOps Engineer", "SRE", "Data Scientist",
"Growth Hacker", "Customer Success Manager" são termos incorporados ao mercado brasileiro
e devem ser mantidos em inglês mesmo quando o título da vaga venha em PT. Não force
tradução para "Dono de Produto", "Mestre Scrum" ou similar. Exceções naturais: "Developer"
vira "Desenvolvedor", "Engineer" vira "Engenheiro", "Analyst" vira "Analista" — o catálogo
usa termos em PT nesses casos.

Siglas C-level (CTO, CEO, COO, CMO, CFO) são canônicos em inglês, como termos
incorporados. CCO é intencionalmente rejeitado por ambiguidade tripla (Chief Customer,
Chief Commercial, Chief Content Officer) — "Chief Customer Officer" mapeia para "Diretor
de Customer Success" (ver REGRA A).
```

**Mudança 4 — Micro-regra contextual Support vs Service (adicionar após as regras A-E):**

```
MICRO-REGRA CONTEXTUAL — SUPPORT/SERVICE BILÍNGUE

"Customer Support" ou "Customer Service" em empresa brasileira (sem outro
qualificador) → "Analista de Customer Service" ou "Analista de Customer Experience",
NUNCA "Analista de Suporte".

"Support" em inglês em contexto de clientes finais, pedidos, entregas, reservas, SAC
→ "Analista de Customer Service".

"Suporte" em português aplica-se apenas quando for a software, sistema ou produto
técnico com chamados, incidentes ou troubleshooting de TI.
```

**Overhead de tokens — medição obrigatória pré-PR1:**

Executor roda `tiktoken` (ou equivalente) no novo SYSTEM_PROMPT e reporta valor exato. Estimativa prévia: 1.400-1.800 tokens adicionados (regras A-E + Etapa 0 + Etapa 2 + micro-regra Support), passando prompt total de ~5.500 para ~7.300 tokens.

**Se medição real passar de 1.800 tokens**, considerar uma das mitigações:

1. Dividir regras A-E em versão concisa no prompt base + apêndice carregado apenas quando aplicável (ex: apenas em vagas com sinal claro de hierarquia).
2. Mover exemplos numerados para arquivo externo referenciado como "see appendix", reduzindo overhead sem perder guia.

Qualquer mitigação adicionada precisa de validação A/B antes de ir para produção.

---

## 7. Código novo esperado

### 7.1 Nova função em `lib/minhash.ts`: extração de `normalizeResumeText` + nova `normalizeDescriptionForHash`

A função `normalizeResumeText` hoje vive em `components/landing/analyze-modal-utils/text-processor.ts`. Extrair para `lib/minhash.ts` como função pura server-safe. Os 3 callers existentes do landing/upload atualizam import.

```typescript
// lib/minhash.ts (adição)

const STOPWORDS_PT_BR = new Set([
    'de', 'da', 'do', 'das', 'dos', 'e', 'o', 'a', 'os', 'as',
    'em', 'na', 'no', 'nas', 'nos', 'para', 'por', 'com', 'sem',
    'um', 'uma', 'uns', 'umas', 'que', 'se', 'ao', 'à', 'ou',
    'foi', 'são', 'ser', 'ter', 'está', 'mais', 'também',
    'sobre', 'entre', 'até', 'desde', 'durante', 'quando',
    'como', 'sua', 'seu', 'suas', 'seus'
]);

export function normalizeResumeText(text: string): string {
    if (!text) return '';

    return text
        .toLowerCase()
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/[-=*_]{3,}/g, ' ')
        // v5.1 — inclui aspas tipográficas (comuns em textos copiados do Word/LinkedIn)
        .replace(/[,.\-()\[\]{}\/\\:;!?"'#$%&*+<>\u2018\u2019\u201C\u201D]/g, ' ')
        .replace(/\b\d+\b/g, ' ')
        .split(/\s+/)
        .filter(token => token && !STOPWORDS_PT_BR.has(token))
        .join(' ')
        .replace(/\s+/g, ' ')
        .trim();
}

/**
 * Variante mais conservadora para hashing exato de descrições.
 * v5.1 — strip de HTML tags (descrições de scrapers frequentemente contêm <p>, <br>, etc.)
 * v5.3 (GS3) — strip de pontuação não-alfanumérica antes do hash.
 *              Scrapers diferentes (Gupy, Greenhouse, LinkedIn) geram bullets
 *              distintos para a mesma vaga ('•' vs '-' vs '*' vs '◦'). Sem
 *              essa normalização, hashes SHA-256 divergem para conteúdo
 *              semanticamente idêntico, reduzindo hit rate da Camada 0.
 *              Usamos \p{L}\p{N}\s com flag /u para preservar acentos PT-BR.
 * Não remove stopwords nem números — apenas trim, lowercase, collapse de whitespace.
 */
export function normalizeDescriptionForHash(text: string): string {
    if (!text) return '';
    return text
        // v5.1 — strip HTML tags antes de qualquer coisa
        // Scrapers Greenhouse/Gupy trazem <p>, <br>, <ul>, etc.
        // Sem isso, descrição idêntica com/sem formatação gerava hashes diferentes.
        .replace(/<[^>]*>/g, ' ')
        // v5.1 — HTML entities comuns
        .replace(/&nbsp;/g, ' ')
        .replace(/&amp;/g, '&')
        .replace(/&lt;/g, '<')
        .replace(/&gt;/g, '>')
        .replace(/&quot;/g, '"')
        .replace(/&#39;/g, "'")
        .trim()
        .toLowerCase()
        .replace(/[\r\n\t\u00a0]/g, ' ')  // v5.1 — inclui non-breaking space
        // v5.3 (GS3) — strip de pontuação não-alfanumérica Unicode-aware.
        // Preserva letras (\p{L}), números (\p{N}) e whitespace (\s).
        // Remove bullets ('•', '◦', '▪'), hífens, asteriscos, barras, parênteses,
        // aspas, pontos finais, vírgulas, etc. — tudo que varia entre scrapers.
        .replace(/[^\p{L}\p{N}\s]/gu, ' ')
        .replace(/\s+/g, ' ')
        .trim();
}
```

**Nota sobre B5 de backfill (v5.2) e GS3 (v5.3):** a mudança em `normalizeDescriptionForHash` altera os hashes de **todas** as descrições — incluindo as que o backfill já populou com a normalização antiga. Na prática não é um problema operacional porque:
- Vagas backfilled com `prompt_structure_version='legacy'` **não servem de âncora** (filtro da Camada 0 via `.neq('prompt_structure_version', 'legacy')`). Hash antigo fica no banco mas não é consultado.
- Vagas `human_validated=true` (pool inicial ~100) **têm hash recomputado** no PR2 ao passarem pelo pipeline novo na primeira curagem via Camada 0, porque o hash correto é calculado no momento do lookup (`precheckDescriptionHash` chama `normalizeDescriptionForHash` do código atual).
- Vagas curadas pós-deploy já usam a normalização correta desde o início.

Portanto, não é necessário re-rodar o backfill. Se houver dúvida em staging, executar a query `SELECT COUNT(*) FROM job_postings WHERE description_hash IS NOT NULL AND human_validated = true` antes e depois da primeira passagem pipeline — os hashes dessas vagas devem ser atualizados conforme elas passam pela Camada 0 como candidatos (mas não como âncoras, porque Camada 0 não escreve hash em vagas já curadas).


**Migração dos callers:** `components/landing/analyze-modal-utils/text-processor.ts` agora importa `normalizeResumeText` de `@/lib/minhash` e re-exporta para compatibilidade com código cliente.

**v5.5 (Mistral B1) — Paridade de contrato entre backfill e runtime:** tanto `scripts/backfill-job-description-hash.ts` (§7.6) quanto `lib/pipeline/precheck-description-hash.ts` (§7.3) **importam a mesma função `normalizeDescriptionForHash` de `lib/minhash.ts`**. Não há implementação duplicada em SQL ou em outro arquivo — o runtime e o backfill compartilham o mesmo contrato de normalização por construção. Qualquer alteração futura em `normalizeDescriptionForHash` afeta ambos simultaneamente, sem risco de dessincronização. Teste de paridade explícito não é necessário porque não existem duas implementações separadas para reconciliar.

### 7.2 Nova função em `lib/pipeline/taxonomy-cache.ts`: `getFullTaxonomyCache` com preload de labels válidos e gestão de erro

Função separada de `getTaxonomyLoader` existente. Retorna os 4 JSONs em memória mais um `Set<string>` de labels elegíveis, preloaded uma vez por processo. Gestão de erro registra falha e permite fail-fast com mensagem acionável.

```typescript
// lib/pipeline/taxonomy-cache.ts (adição)

import fs from 'fs';
import path from 'path';
import { SupabaseClient } from '@supabase/supabase-js';

interface FullTaxonomyCache {
    equivalences: Record<string, string>;
    familySynonyms: Record<string, string[]>;
    domainSynonyms: Record<string, string>;
    domains: Record<string, string[]>;
    validCanonicalLabels: Set<string>;        // active + pending (lookup geral)
    allowedForPreResolution: Set<string>;     // v5.1 — subset aprovado para auto-decisão
    vacancyCountByLabel: Map<string, number>;
    roleIdByLabel: Map<string, string>;       // v5.2 — label → id para override B3
}

let cached: FullTaxonomyCache | null = null;
let cachedAt: number = 0;
const TTL_MS = 5 * 60 * 1000; // 5 min, alinhado com padrão canonical-skills.ts

export async function getFullTaxonomyCache(
    supabase: SupabaseClient
): Promise<FullTaxonomyCache> {
    const now = Date.now();
    if (cached && (now - cachedAt) < TTL_MS) {
        return cached;
    }

    const baseDir = path.join(process.cwd(), 'data');

    const loadJson = (filename: string) => {
        const filepath = path.join(baseDir, filename);
        try {
            const content = fs.readFileSync(filepath, 'utf-8');
            const parsed = JSON.parse(content);
            return parsed.data ?? {};
        } catch (err) {
            throw new Error(
                `getFullTaxonomyCache: failed to load ${filename} from ${filepath}. ` +
                `Original error: ${err instanceof Error ? err.message : String(err)}`
            );
        }
    };

    let equivalences: Record<string, string>;
    let familySynonyms: Record<string, string[]>;
    let domainSynonyms: Record<string, string>;
    let domains: Record<string, string[]>;

    try {
        equivalences = loadJson('equivalences.json');
        familySynonyms = loadJson('family_synonyms.json');
        domainSynonyms = loadJson('domain_synonyms.json');
        domains = loadJson('domains.json');
    } catch (err) {
        // Registra erro em events para observabilidade
        // v5.6 (ChatGPT C5): actor='system' implica actor_id=SYSTEM_USER_ID (0001),
        // não 0004 (que é o admin Onsly). Eventos do pipeline são do "sistema",
        // não do usuário admin que decidiu cadastrar uma whitelist.
        await supabase.from('events').insert({
            event_name: 'taxonomy_cache_load_failed',
            resource_type: 'pipeline',
            actor: 'system',
            actor_id: '00000000-0000-0000-0000-000000000001',
            reason: err instanceof Error ? err.message : String(err),
            metadata: {}
        });
        throw err;
    }

    // Preload de labels válidos (active + pending) + vacancy_count
    // Substitui query per-call em Camadas 1 e 2
    const { data: validRoles, error: rolesError } = await supabase
        .from('job_canonical_roles')
        .select('id, label, status, vacancy_count')
        .in('status', ['active', 'pending']);

    if (rolesError) {
        await supabase.from('events').insert({
            event_name: 'taxonomy_cache_valid_labels_failed',
            resource_type: 'pipeline',
            actor: 'system',
            actor_id: '00000000-0000-0000-0000-000000000001',  // v5.6 (ChatGPT C5): SYSTEM_USER_ID
            reason: rolesError.message,
            metadata: {}
        });
        throw new Error(`Failed to load valid canonical labels: ${rolesError.message}`);
    }

    const validCanonicalLabels = new Set<string>(
        (validRoles ?? []).map(r => r.label)
    );

    const vacancyCountByLabel = new Map<string, number>(
        (validRoles ?? []).map(r => [r.label, r.vacancy_count ?? 0])
    );

    // v5.2 — B3: mapa label → id para override server-side em persistCuratedJob.
    // Permite resolver UUID a partir do label sem query extra ao banco.
    const roleIdByLabel = new Map<string, string>(
        (validRoles ?? []).map(r => [r.label, r.id])
    );

    // v5.1 — carregar whitelist de auto-decisão
    // Canônicos `active` entram automaticamente.
    // Canônicos `pending` só entram se estiverem na tabela allowed_for_pre_resolution.
    const { data: whitelistRows, error: whitelistError } = await supabase
        .from('allowed_for_pre_resolution')
        .select('canonical_role_id');

    if (whitelistError) {
        console.error('Failed to load autodecision whitelist:', whitelistError.message);
        // Fallback defensivo: em erro, tratar TODO canônico active como elegível e NENHUM pending.
        // Falha-segura: em caso de dúvida, deixa o LLM decidir.
    }

    const whitelistedIds = new Set<string>(
        (whitelistRows ?? []).map((r: { canonical_role_id: string }) => r.canonical_role_id)
    );

    const allowedForPreResolution = new Set<string>(
        (validRoles ?? [])
            .filter(r => r.status === 'active' || whitelistedIds.has(r.id))
            .map(r => r.label)
    );

    cached = {
        equivalences,
        familySynonyms,
        domainSynonyms,
        domains,
        validCanonicalLabels,
        allowedForPreResolution,
        vacancyCountByLabel,
        roleIdByLabel    // v5.2
    };
    cachedAt = now;

    return cached;
}

/**
 * Força recompilação — usar apenas em testes ou após reload manual dos JSONs.
 */
export function invalidateFullTaxonomyCache(): void {
    cached = null;
    cachedAt = 0;
}
```

**Decisão de arquitetura:** TTL de 5 minutos alinha com o padrão existente em `canonical-skills.ts`. Se canônicos forem promovidos de pending para active via admin, a mudança propaga em até 5 min.

**v5.4 (GS-R6-3) — invalidação opcional no rename de canônicos:** o trigger N3 propaga `canonical_role_label` em vagas via âncora quando admin renomeia um canônico. Mas `getFullTaxonomyCache` tem TTL de 5 min, e o `roleIdByLabel` dentro dele é consultado pela Camada 0 via `hashResult.canonical_role_label`. Se admin renomeia "Analista de Vendas" para "Analista Comercial" e uma vaga duplicada chega 30s depois, a Camada 0 pode escrever o label antigo (do cache) antes do trigger reescrever via âncora para o novo label. O trigger corrige a inconsistência em seguida, então a janela de observação é curta (~5 min em pior caso) e auto-cura.

**Mitigação opcional:** o endpoint admin de rename (`PUT /api/admin/canonical-roles/[id]`) pode chamar `invalidateFullTaxonomyCache()` imediatamente após o UPDATE para forçar recarga do cache no próximo acesso. Trivial de adicionar, elimina a janela. Não é bloqueante — fica como débito observável se admin reclamar de inconsistência visível em produção.

**v5.5 (Mistral B3) — Evolução futura: chave de versão composta para cache robusto.** O TTL de 5 min cobre 95% dos casos, mas não protege contra misturas de versão dentro do mesmo lote quando múltiplos workers processam vagas simultaneamente e um invalida o cache entre eles. Para cenários de produção intensa com paralelismo alto, a evolução recomendada é mudar de TTL puro para chave composta `taxonomy_cache_key = PROMPT_CONTENT_VERSION + domain_synonyms_version + canonical_roles_schema_version`. Qualquer uma das três mudando força recarga — sem janelas de inconsistência de 5 min. Escopo pós-v5.5, quando o produto atingir volume que justifique. Hoje (E2E ainda em estabilização, zero tráfego real), TTL puro é suficiente.

**Convivência com `lib/pipeline/canonical-skills.ts` (sprint 0.x, finalizada em 23/04/2026):** a v5.4 adiciona `normalizeTitle` em `lib/pipeline/text-processing.ts`, mesmo arquivo onde a sprint 0 adicionou `normalizeSkill` com `SYMBOL_LOOKUP`. São funções independentes e coexistem sem conflito. Executor do PR2 deve **preservar** `normalizeSkill` ao adicionar `normalizeTitle` — ambas exportadas, escopos ortogonais.

### 7.3 Novo arquivo: `lib/pipeline/precheck-description-hash.ts`

Implementa a Camada 0 do pipeline reestruturado.

```typescript
import crypto from 'crypto';
import { SupabaseClient } from '@supabase/supabase-js';
import { normalizeDescriptionForHash } from '@/lib/minhash';
import { normalizeTitle } from '@/lib/pipeline/text-processing';  // v5.2 — import estático
import { MissReason } from '@/lib/pipeline/types';  // v5.5 (Grok #4) — tipo unificado de miss reason

export interface DescriptionHashPreCheckHit {
    hit: true;
    canonical_role_id: string;
    canonical_role_label: string;
    skills: string[];
    seniority_inferred: string | null;
    work_model: string | null;
    description_curated: string | null;
    generated_hash: string;                    // v5.1 — hash calculado, necessário para persistir
    matched_source_ids: string[];
    anchor_title: string;                      // v5.1 — para guard de título
    // v5.4 (GS-R6-10): optimistic lock — caller propaga para persistPrecheckResult,
    // que re-lê a âncora e aborta se updated_at mudou entre lookup e persist.
    anchor_job_id: string;                     // UUID da âncora principal (primeira da shortlist)
    anchor_updated_at: string;                 // ISO timestamp da âncora no momento do lookup
    reason: 'human_validated_anchor' | 'temporal_quorum';
}

export type DescriptionHashPreCheckMissReason = MissReason;

export interface DescriptionHashPreCheckMiss {
    hit: false;
    generated_hash: string | null;
    miss_reason?: DescriptionHashPreCheckMissReason;
}

const TTL_DAYS = 30;
const TEMPORAL_QUORUM_MIN_GAP_HOURS = 24;
const MIN_WORDS = 80;
const MIN_TITLE_TOKEN_OVERLAP = 2;  // Guard anti-boilerplate multi-cargo

/**
 * v5.1 — Camada 0 com 4 correções arquiteturais:
 * (a) generated_hash retornado também no hit (necessário para persistência)
 * (b) allowedForPreResolution restringe auto-decisão (não basta ser "conhecido")
 * (c) conflict detection quando 2+ canônicos satisfazem quórum → miss + evento
 * (d) title guard — hash idêntico + título com overlap < 2 tokens → miss (boilerplate)
 */
export async function precheckDescriptionHash(
    description: string,
    candidateTitle: string,                      // v5.1 — título da vaga nova
    currentPromptStructureVersion: string,
    allowedForPreResolution: Set<string>,        // v5.1 — whitelist de canônicos elegíveis
    supabase: SupabaseClient
): Promise<DescriptionHashPreCheckHit | DescriptionHashPreCheckMiss> {
    const normalized = normalizeDescriptionForHash(description);

    const words = normalized.split(/\s+/).filter(Boolean);
    if (words.length < MIN_WORDS) {
        return { hit: false, generated_hash: null, miss_reason: 'short_description' };
    }

    const descriptionHash = crypto
        .createHash('sha256')
        .update(normalized)
        .digest('hex');

    const { data: candidates, error } = await supabase
        .from('job_postings')
        .select(`
            id,
            canonical_role_id,
            original_title,
            skills,
            seniority_inferred,
            work_model,
            description_curated,
            curated_at,
            updated_at,
            human_validated
        `)
        .eq('curation_status', 'curated')
        .eq('description_hash', descriptionHash)
        .eq('prompt_structure_version', currentPromptStructureVersion)
        .neq('prompt_structure_version', 'legacy')  // v5.1 — vagas backfilled não viram âncora
        .gte(
            'curated_at',
            new Date(Date.now() - TTL_DAYS * 24 * 60 * 60 * 1000).toISOString()
        )
        .not('canonical_role_id', 'is', null)
        .not('original_title', 'is', null)         // v5.2 — N7: guard de título exige original_title populado
        .order('human_validated', { ascending: false })
        .order('curated_at', { ascending: false })
        .limit(200);  // v5.1 — aumentado de 50 para não perder âncoras human_validated antigas

    if (error || !candidates || candidates.length === 0) {
        return { hit: false, generated_hash: descriptionHash, miss_reason: 'no_match_in_db' };
    }

    const candidateRoleIds = Array.from(
        new Set(candidates.map(c => c.canonical_role_id).filter(Boolean))
    );

    const { data: roles } = await supabase
        .from('job_canonical_roles')
        .select('id, label, status')
        .in('id', candidateRoleIds)
        .in('status', ['active', 'pending']);

    if (!roles || roles.length === 0) {
        return { hit: false, generated_hash: descriptionHash, miss_reason: 'no_match_in_db' };
    }

    const roleLabelMap = new Map(roles.map(r => [r.id, r.label]));

    // v5.1 — filtra apenas roles que estão em allowedForPreResolution
    const eligibleRoleIds = new Set(
        roles
            .filter(r => allowedForPreResolution.has(r.label))
            .map(r => r.id)
    );

    if (eligibleRoleIds.size === 0) {
        return { hit: false, generated_hash: descriptionHash, miss_reason: 'canonical_not_allowed' };
    }

    const byCanonical = new Map<string, typeof candidates>();
    for (const c of candidates) {
        if (!c.canonical_role_id || !eligibleRoleIds.has(c.canonical_role_id)) continue;
        const list = byCanonical.get(c.canonical_role_id) ?? [];
        list.push(c);
        byCanonical.set(c.canonical_role_id, list);
    }

    // v5.1 — coletar TODOS os canônicos que satisfazem quórum antes de decidir
    type QuorumMatch = {
        canonicalId: string;
        anchor: typeof candidates[0];
        matchedSources: typeof candidates;
        reason: 'human_validated_anchor' | 'temporal_quorum';
    };

    const quorumMatches: QuorumMatch[] = [];

    for (const [canonicalId, matches] of byCanonical.entries()) {
        // Quórum A: pelo menos 1 âncora human_validated
        const humanValidated = matches.filter(m => m.human_validated === true);
        if (humanValidated.length >= 1) {
            quorumMatches.push({
                canonicalId,
                anchor: humanValidated[0],
                matchedSources: humanValidated.slice(0, 3),
                reason: 'human_validated_anchor'
            });
            continue;
        }

        // Quórum B: ≥ 2 âncoras com gap temporal ≥ 24h
        if (matches.length >= 2) {
            const timestamps = matches
                .map(m => new Date(m.curated_at).getTime())
                .sort((a, b) => a - b);
            const gapHours =
                (timestamps[timestamps.length - 1] - timestamps[0]) / (1000 * 60 * 60);

            if (gapHours >= TEMPORAL_QUORUM_MIN_GAP_HOURS) {
                const anchor = [...matches].sort(
                    (a, b) =>
                        new Date(a.curated_at).getTime() - new Date(b.curated_at).getTime()
                )[0];

                quorumMatches.push({
                    canonicalId,
                    anchor,
                    matchedSources: matches.slice(0, 3),
                    reason: 'temporal_quorum'
                });
            }
        }
    }

    if (quorumMatches.length === 0) {
        return { hit: false, generated_hash: descriptionHash, miss_reason: 'no_quorum' };
    }

    // v5.1 — detecção de conflito: 2+ canônicos satisfazendo quórum
    if (quorumMatches.length > 1) {
        // Prioridade: human_validated > temporal
        const humanValidatedGroups = quorumMatches.filter(m => m.reason === 'human_validated_anchor');

        // Se só 1 canônico tem human_validated, ele vence mesmo com outros temporais
        if (humanValidatedGroups.length === 1) {
            const winner = humanValidatedGroups[0];
            return buildHitResult(winner, roleLabelMap, descriptionHash, candidateTitle);
        }

        // Se múltiplos têm human_validated OU nenhum tem e múltiplos são temporais → conflito
        await supabase.from('events').insert({
            event_name: 'precheck_conflict_detected',
            resource_type: 'job_posting',
            reason: `${quorumMatches.length} canônicos satisfazem quórum para hash ${descriptionHash.slice(0, 16)}`,
            actor: 'pipeline',
            actor_id: '00000000-0000-0000-0000-000000000001',
            metadata: {
                canonical_role_ids: quorumMatches.map(m => m.canonicalId),
                canonical_role_labels: quorumMatches.map(
                    m => roleLabelMap.get(m.canonicalId) ?? '?'
                ),
                reasons: quorumMatches.map(m => m.reason),
                description_hash: descriptionHash
            }
        });
        return {
            hit: false,
            generated_hash: descriptionHash,
            miss_reason: 'conflict_detected'
        };
    }

    // Apenas 1 canônico satisfaz quórum
    const winner = quorumMatches[0];
    return buildHitResult(winner, roleLabelMap, descriptionHash, candidateTitle);
}

/**
 * v5.1 — Constrói resultado Hit aplicando guarda de título.
 * Se overlap de tokens do candidateTitle com anchor.original_title < 2, retorna miss.
 * Previne propagação de boilerplate multi-cargo (descrições iguais, cargos diferentes).
 */
function buildHitResult(
    match: {
        canonicalId: string;
        anchor: { id: string; original_title: string | null; skills: string[] | null;
                  seniority_inferred: string | null; work_model: string | null;
                  description_curated: string | null;
                  updated_at: string };  // v5.4 (GS-R6-10) — optimistic lock
        matchedSources: Array<{ id: string }>;
        reason: 'human_validated_anchor' | 'temporal_quorum';
    },
    roleLabelMap: Map<string, string>,
    descriptionHash: string,
    candidateTitle: string
): DescriptionHashPreCheckHit | DescriptionHashPreCheckMiss {
    const anchorTitle = match.anchor.original_title ?? '';

    // v5.2 — B6: guard usa titleGuardPasses que trata exceção C-level
    if (!titleGuardPasses(candidateTitle, anchorTitle)) {
        return {
            hit: false,
            generated_hash: descriptionHash,
            miss_reason: 'title_guard_failed'
        };
    }

    return {
        hit: true,
        canonical_role_id: match.canonicalId,
        canonical_role_label: roleLabelMap.get(match.canonicalId) ?? '',
        skills: match.anchor.skills ?? [],
        seniority_inferred: match.anchor.seniority_inferred,
        work_model: match.anchor.work_model,
        description_curated: match.anchor.description_curated,
        generated_hash: descriptionHash,
        matched_source_ids: match.matchedSources.map(m => m.id),
        anchor_title: anchorTitle,
        // v5.4 (GS-R6-10): propaga id e timestamp da âncora para o optimistic
        // lock do persistPrecheckResult — re-lê âncora e aborta se updated_at mudou.
        anchor_job_id: match.anchor.id,
        anchor_updated_at: match.anchor.updated_at,
        reason: match.reason
    };
}

/**
 * v5.1 — Normaliza títulos para tokens comparáveis e conta overlap.
 * Usa o mesmo normalizeTitle das Camadas 1 e 2.
 * Tokens genéricos comuns em títulos não contam (ex: "de", "e", "a").
 *
 * v5.2 — B6: cargos C-level curtos (CTO, CEO, SDR, QA, UX) têm apenas 1 token
 * após stopwords removidas. Regra estrita de 2 tokens bloqueava esses casos
 * legítimos. Nova exceção: se ambos os títulos têm ≤ 2 tokens após normalização,
 * aceita overlap ≥ 1 (em vez de ≥ 2). Não afeta casos genéricos longos.
 *
 * v5.5 (Grok #4): EXPORTADA para testes.
 */
export function computeTitleTokenOverlap(a: string, b: string): number {
    if (!a || !b) return 0;
    const STOPWORDS = new Set(['de', 'da', 'do', 'em', 'para', 'com', 'e', 'a', 'o', 'of', 'in', 'for']);

    const tokenize = (s: string) =>
        new Set(
            normalizeTitle(s)
                .split(/\s+/)
                .filter((t: string) => t.length > 0 && !STOPWORDS.has(t))
        );

    const setA = tokenize(a);
    const setB = tokenize(b);
    let overlap = 0;
    for (const token of setA) {
        if (setB.has(token)) overlap++;
    }
    return overlap;
}

/**
/**
 * v5.4 (GS-R6-4): verifica se o guard de título aprova considerando exceção C-level
 * restrita a lista explícita de cargos conhecidos.
 *
 * v5.5 (Grok #4): EXPORTADA para ser testável diretamente em tests/ via import.
 * A v5.4 tinha declarada como `function` sem export, quebrando os testes de
 * §9.2.3 que fazem `import { titleGuardPasses } from '@/lib/pipeline/precheck-description-hash'`.
 */
export function titleGuardPasses(candidateTitle: string, anchorTitle: string): boolean {
    if (!candidateTitle || !anchorTitle) return false;

    const STOPWORDS = new Set(['de', 'da', 'do', 'em', 'para', 'com', 'e', 'a', 'o', 'of', 'in', 'for']);
    const tokenize = (s: string) =>
        normalizeTitle(s).split(/\s+/).filter(t => t.length > 0 && !STOPWORDS.has(t));

    const tokensCandidate = tokenize(candidateTitle);
    const tokensAnchor = tokenize(anchorTitle);

    const overlap = computeTitleTokenOverlap(candidateTitle, anchorTitle);

    // v5.4 (GS-R6-4): Exceção C-level via whitelist explícita.
    // Siglas curtas que identificam cargos únicos — nelas, 1 token de overlap é
    // semanticamente suficiente porque o próprio token já carrega todo o sentido
    // hierárquico ("CTO" é só CTO, não existe "CTO Sênior" vs "CTO Júnior").
    const CLEVEL_TOKENS = new Set([
        // C-suite tradicional
        'cto', 'ceo', 'coo', 'cfo', 'cmo', 'cro', 'cpo', 'ciso', 'chro',
        // Leadership
        'vp', 'evp', 'svp',
        // Sales/BD
        'sdr', 'bdr', 'ae', 'csm',
        // Tech
        'qa', 'ux', 'ui', 'pm', 'po', 'tl', 'tpm', 'sre', 'devops',
    ]);

    const bothShortAndClevel =
        tokensCandidate.length <= 2 &&
        tokensAnchor.length <= 2 &&
        tokensCandidate.some(t => CLEVEL_TOKENS.has(t)) &&
        tokensAnchor.some(t => CLEVEL_TOKENS.has(t));

    if (bothShortAndClevel) {
        return overlap >= 1;
    }

    // Regra padrão: exige overlap ≥ 2.
    // Vale também para "Analista Sênior" vs "Analista Júnior" (ambos 2 tokens
    // mas sem C-level token) — overlap 1 em "analista" é insuficiente.
    return overlap >= MIN_TITLE_TOKEN_OVERLAP;
}
```

Notas de implementação:

- Parâmetro `supabase` recebido via ctx do caller, não via factory interna.
- Clonagem dos 5 outputs garante vaga nova entrar no banco com estado completo idêntico ao da âncora.
- Em empate temporal, usa âncora mais antiga como fonte de verdade (escolha determinística).
- **v5.1:** conflito entre canônicos (2+ satisfazendo quórum sem critério de desempate) gera `precheck_conflict_detected` em `events` e a vaga segue para Camadas 1/2/LLM normalmente.
- **v5.1:** `anchor_title` é retornado no hit para o caller poder usar no `curation_metadata` de auditoria.
- **v5.1:** `canonical_not_allowed` é retornado quando o canônico existe mas não está em `allowedForPreResolution` — isso evita auto-decisão para canônicos pending que o Onsly ainda não aprovou explicitamente para whitelist.
- **v5.4 (GS-R6-4):** `titleGuardPasses` usa **lista C-level explícita** (CTO, CEO, COO, CFO, CMO, VP, SDR, BDR, QA, UX, PM, PO, TL, SRE, etc.) em vez da regra frouxa "≤ 2 tokens". Corrige falso positivo crítico: "Analista Sênior" vs "Analista Júnior" (ambos 2 tokens, overlap 1 em "analista") passava com `true` no v5.3 e permitiria Camada 0 clonar cross-seniority em empresas que usam templates de descrição. Agora cai na regra padrão ≥ 2 e retorna `false`.
- **v5.2 (N7):** query de candidates adiciona `.not('original_title', 'is', null)` — garante que apenas vagas com título populado servem como âncora. Evita degradação do guard de título por dados faltantes em âncoras antigas.
- **v5.2:** `normalizeTitle` agora é import estático no topo do arquivo (era `require()` dinâmico). Remove inconsistência de estilo e previne edge cases em build ESM.

### 7.4 Novo arquivo: `lib/pipeline/domain-synonyms-lookup.ts`

Implementa a Camada 1 com heurística de tokens + posição, regex sem estado global compartilhado, e validação in-memory de canônico destino.

#### 7.4.0 Contrato de `normalizeTitle` — especificação formal (v5.1)

`normalizeTitle` já existe em `lib/pipeline/text-processing.ts`, mas a v5 não a especificava. Como a confiabilidade das entradas do `domain_synonyms.json` depende 100% de como `normalizeTitle` transforma o título bruto, a v5.1 formaliza o contrato:

```typescript
// lib/pipeline/text-processing.ts

/**
 * Normaliza título de vaga para comparação léxica determinística.
 *
 * Operações aplicadas, nesta ordem:
 *   1. lowercase
 *   2. Unicode NFD + strip de diacríticos (remove acentos)
 *   3. substitui caracteres de separação por espaço: / \ - _ · •
 *   4. substitui "&" por " and " (preserva conector em "fp&a" → "fp and a")
 *   5. remove pontuação restante: , . : ; ! ? " ' ( ) [ ] { } < > # $ % * +
 *   6. colapsa whitespace em espaço único
 *   7. trim
 *
 * NÃO remove números nem stopwords — preserva contexto útil para matching.
 *
 * Keys em domain_synonyms.json DEVEM estar escritas no formato produzido
 * por essa função. Exemplos de transformações canônicas:
 *   "Head of FP&A"                → "head of fp and a"
 *   "Sales Ops / Revenue Ops"     → "sales ops revenue ops"
 *   "CX Specialist - Sênior"      → "cx specialist senior"
 *   "Customer Success Manager"    → "customer success manager"
 *   "Analista Tributário"         → "analista tributario"
 */
export function normalizeTitle(rawTitle: string): string {
    if (!rawTitle) return '';

    return rawTitle
        .toLowerCase()
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/[\/\\\-_·•]/g, ' ')
        .replace(/&/g, ' and ')
        .replace(/[,.:;!?"'()\[\]{}<>#$%*+]/g, ' ')
        .replace(/\s+/g, ' ')
        .trim();
}
```

**Implicação operacional:** ao editar `domain_synonyms.json`, o autor DEVE escrever as keys como a saída dessa função. Para as 68 entradas novas do patch §6.1, isso já foi validado: `"head of fp&a"` vira `"head of fp and a"` na normalização, então a key precisa estar escrita como `"head of fp&a"` no JSON (a transformação `&` → ` and ` ocorre no runtime). Mais simples: keys no JSON podem manter `&` original porque a normalização converte runtime. **O que não pode**: key com acento, com caixa mista, ou com barras/hífens — porque essas formas não sobrevivem à normalização.

**Teste obrigatório no PR1 (tests/normalize-title.test.ts):**

```typescript
describe('normalizeTitle', () => {
    it.each([
        ['Head of FP&A', 'head of fp and a'],
        ['Sales Ops / Revenue Ops', 'sales ops revenue ops'],
        ['CX Specialist - Sênior', 'cx specialist senior'],
        ['Customer Success Manager', 'customer success manager'],
        ['Analista Tributário', 'analista tributario'],
        ['  Espaços   duplos  ', 'espacos duplos'],
        ['', ''],
    ])('normalizes "%s" → "%s"', (input, expected) => {
        expect(normalizeTitle(input)).toBe(expected);
    });
});
```

#### 7.4.0.1 Teste de higiene de keys no prebuild (v5.2 — N1)

Criar `tests/domain-synonyms-hygiene.test.ts`. Roda no prebuild e falha o build se detectar keys malformadas.

```typescript
import { describe, it, expect } from 'vitest';
import { normalizeTitle } from '@/lib/pipeline/text-processing';
import domainSynonymsJson from '@/data/domain_synonyms.json';

describe('domain_synonyms.json — higiene de keys (v5.2 — N1)', () => {
    it('todas as keys sobrevivem a normalizeTitle sem perder conteúdo semântico', () => {
        // Objetivo: detectar key com pontuação/caracteres que normalizeTitle
        // elimina completamente (ex: "!!!" → ""), tornando a key match-impossível.
        const entries = Object.entries(domainSynonymsJson.data as Record<string, string>);

        const problems: Array<{ key: string; normalized: string; reason: string }> = [];

        for (const [key, _label] of entries) {
            const normalized = normalizeTitle(key);

            // Regra 1: key normalizada não pode ser vazia
            if (!normalized) {
                problems.push({ key, normalized, reason: 'normalização produz string vazia' });
                continue;
            }

            // Regra 2: key normalizada não pode perder > 50% dos caracteres
            // (indica que tinha muita pontuação/caracteres que foram strip-ados)
            const lossRatio = 1 - (normalized.length / key.length);
            if (lossRatio > 0.5) {
                problems.push({
                    key,
                    normalized,
                    reason: `normalização remove ${Math.round(lossRatio * 100)}% dos caracteres`
                });
            }
        }

        if (problems.length > 0) {
            const report = problems
                .map(p => `  - "${p.key}" → "${p.normalized}" (${p.reason})`)
                .join('\n');
            throw new Error(
                `Detectadas ${problems.length} keys malformadas em domain_synonyms.json:\n${report}\n\n` +
                `Ação: reescreva as keys em formato compatível com normalizeTitle. ` +
                `Keys devem usar espaços, palavras alfanuméricas e caracteres comuns de título ` +
                `(ex: "Head of FP&A", não "Head-of-FP&A!!!").`
            );
        }

        expect(problems).toHaveLength(0);
    });

    it('não há duplicatas após normalização', () => {
        // Duas keys distintas que normalizam para o mesmo valor são conflito
        // silencioso (a última vence no Map, primeira é ignorada).
        const entries = Object.entries(domainSynonymsJson.data as Record<string, string>);
        const normalizedMap = new Map<string, string>();
        const duplicates: Array<[string, string]> = [];

        for (const [key, _label] of entries) {
            const norm = normalizeTitle(key);
            if (normalizedMap.has(norm)) {
                duplicates.push([normalizedMap.get(norm)!, key]);
            } else {
                normalizedMap.set(norm, key);
            }
        }

        expect(duplicates).toEqual([]);
    });

    it('todos os labels existem como canônicos conhecidos', () => {
        // Validação offline — executor pode pular se canonicalLabels não disponível
        // (esta verificação é feita no runtime via allowedForPreResolution).
        const entries = Object.values(domainSynonymsJson.data as Record<string, string>);
        expect(entries.every(label => typeof label === 'string' && label.length > 0)).toBe(true);
    });

    // v5.5 (Mistral B7): fixtures negativos por categoria de entrada nova.
    // Objetivo: garantir que sinônimos novos não overmatcham títulos que
    // deveriam cair no LLM (precisão). Teste falha se algum título do array
    // NEGATIVE_TITLES for resolvido pela Camada 1.
    it('não overmatcham em títulos semanticamente distintos', async () => {
        const { lookupDomainSynonyms } = await import('@/lib/pipeline/domain-synonyms-lookup');
        const { buildDomainSynonymsCache } = await import('@/lib/pipeline/taxonomy-cache');

        const cache = await buildDomainSynonymsCache();

        const NEGATIVE_TITLES = [
            // Categoria A — C-level sem hierarquia
            ['Analista Tributário Sênior', 'não é Tax Analyst de C-level'],
            // Categoria B — Operations em contexto diferente
            ['Operations Research Analyst', 'não é Sales Ops / RevOps'],
            // Categoria C — Support vs Service
            ['Technical Support Engineer', 'não é Customer Service Representative'],
            // Categoria D — Development em contexto não-Sales
            ['Frontend Developer', 'não é Business Development Representative'],
            // Categoria E (v5.5) — Pré-Vendas telecom
            ['Técnico de Pré-Vendas de Telecom', 'não é SDR mesmo com "pré-vendas" na string'],
            // Siglas ambíguas
            ['SDR (Software Defined Radio) Engineer', 'não é SDR de vendas'],
        ];

        for (const [title, reason] of NEGATIVE_TITLES) {
            const result = lookupDomainSynonyms(title, cache);
            if (result.hit) {
                throw new Error(
                    `Overmatch detectado: título "${title}" foi resolvido como ` +
                    `"${result.canonical_label}" pela Camada 1, mas deveria cair no LLM. ` +
                    `Razão: ${reason}`
                );
            }
        }
    });
});
```

**Configuração do prebuild:**

```json
// package.json
{
  "scripts": {
    "prebuild": "npm run test:hygiene && node scripts/generate-prompt-version.ts",
    "test:hygiene": "vitest run tests/domain-synonyms-hygiene.test.ts tests/normalize-title.test.ts"
  }
}
```

Se qualquer teste de higiene falhar, o build de produção é bloqueado. Evita deploy de keys que nunca matcham silenciosamente.

#### 7.4.1 Implementação do lookup

```typescript
import crypto from 'crypto';
import { SupabaseClient } from '@supabase/supabase-js';
import { getFullTaxonomyCache } from '@/lib/pipeline/taxonomy-cache';
import { normalizeTitle } from '@/lib/pipeline/text-processing';

export interface DomainSynonymsHit {
    hit: true;
    matched_key: string;
    canonical_role_label: string;
}

export interface DomainSynonymsMiss {
    hit: false;
}

interface CompiledMeta {
    patternSource: string;
    patternFlags: string;
    keyMap: Map<string, { label: string; tokenCount: number }>;
    compiledFromKeysHash: string;  // v5.2 — N6: hash SHA-256 das keys normalizadas ordenadas
}

let compiledMeta: CompiledMeta | null = null;

/**
 * v5.2 — N6: computa hash SHA-256 das keys normalizadas ordenadas.
 * Usado como invalidador do cache compilado. Permite detectar qualquer
 * mudança no conjunto de keys (inserção, remoção, substituição) mesmo
 * quando a contagem total permanece igual.
 */
function computeKeysHash(normalizedKeys: string[]): string {
    const sorted = [...normalizedKeys].sort();
    return crypto
        .createHash('sha256')
        .update(sorted.join('\u0000'))  // separador null para evitar colisões
        .digest('hex');
}

/**
 * Compila metadados uma vez por processo.
 * Cada chamada cria RegExp local a partir de source+flags para evitar
 * estado compartilhado de lastIndex em workers concorrentes.
 *
 * v5.1 — recompila se número de keys mudou (detecta reload pelo taxonomy-cache TTL).
 * v5.2 — B1: keys do JSON são normalizadas via normalizeTitle antes de compilar
 *        regex. Sem isso, keys com `&`, hífens, acentos ou caixa mista nunca
 *        matchavam porque o input normalizado tem forma diferente das keys cruas.
 * v5.2 — N6: invalidador trocado de contagem para hash SHA-256 das keys ordenadas.
 *        Detecta substituição de key (mesma contagem, conteúdo diferente).
 */
function ensureMeta(domainSynonyms: Record<string, string>): CompiledMeta {
    const rawKeys = Object.keys(domainSynonyms);

    // v5.2 — B1: normalizar cada key via normalizeTitle (mesma função usada no input)
    // keyToNormalizedMap preserva a key original apenas para debugging
    const keyToNormalizedMap = new Map<string, string>();
    const normalizedKeys: string[] = [];

    for (const rawKey of rawKeys) {
        const normKey = normalizeTitle(rawKey);
        if (normKey) {
            keyToNormalizedMap.set(rawKey, normKey);
            normalizedKeys.push(normKey);
        }
    }

    // v5.2 — N6: hash-based cache invalidation
    const currentKeysHash = computeKeysHash(normalizedKeys);

    if (compiledMeta && compiledMeta.compiledFromKeysHash === currentKeysHash) {
        return compiledMeta;
    }

    // Escape regex metachars nas keys já normalizadas
    const escaped = normalizedKeys.map(k => k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'));
    const patternSource = `\\b(?:${escaped.join('|')})\\b`;
    const patternFlags = 'gi';

    // keyMap usa key normalizada como chave (para match após normalizeTitle do input)
    const keyMap = new Map<string, { label: string; tokenCount: number }>();
    for (const rawKey of rawKeys) {
        const normKey = keyToNormalizedMap.get(rawKey);
        if (!normKey) continue;

        keyMap.set(normKey, {
            label: domainSynonyms[rawKey],
            tokenCount: normKey.split(/\s+/).filter(Boolean).length
        });
    }

    compiledMeta = {
        patternSource,
        patternFlags,
        keyMap,
        compiledFromKeysHash: currentKeysHash
    };
    return compiledMeta;
}

/**
 * v5.1 — aceita `allowedForPreResolution` como parâmetro explícito.
 * Antes recebia `validCanonicalLabels` implicitamente do cache, o que
 * permitia auto-decisão para canônicos pending não aprovados.
 */
export async function lookupDomainSynonyms(
    rawTitle: string,
    allowedForPreResolution: Set<string>,
    supabase: SupabaseClient
): Promise<DomainSynonymsHit | DomainSynonymsMiss> {
    if (!rawTitle) return { hit: false };

    // v5.5 (DeepSeek #17) — nota de performance: `getFullTaxonomyCache`
    // é chamado a cada invocação desta função. Em batches de 38 vagas,
    // isso gera 38 chamadas sequenciais ao cache. O TTL de 5min e o retorno
    // por referência mitigam o custo (primeiro acesso popula, demais 37
    // retornam ref do Map cacheado). Otimização futura: aceitar `cache`
    // como parâmetro opcional e cair no `getFullTaxonomyCache` só se não
    // for passado. Ganho marginal (<5ms em batch) — backlog.
    const { domainSynonyms } = await getFullTaxonomyCache(supabase);
    const normalized = normalizeTitle(rawTitle);
    const { patternSource, patternFlags, keyMap } = ensureMeta(domainSynonyms);

    // Regex local por chamada — evita estado compartilhado de lastIndex.
    //
    // v5.3 (GS4) — trade-off documentado: `matchAll` em JavaScript moderno
    // NÃO sofre mutação de `lastIndex` em regex compartilhada (retorna
    // iterator independente). Então tecnicamente poderíamos armazenar a
    // `compiledRegex` em `CompiledMeta` e reutilizar entre chamadas,
    // economizando custo marginal de GC em batches de 38 vagas.
    //
    // Mantemos `new RegExp(...)` por chamada por DEFESA EM PROFUNDIDADE:
    // (a) protege contra bugs futuros se alguém trocar `matchAll` por
    //     `exec`/`match` com flag `/g` (que sofrem mutação de `lastIndex`);
    // (b) o custo real em V8 é baixo — compilação de regex é cacheada
    //     internamente quando `patternSource` é idêntico entre instâncias.
    //
    // Benchmark interno estimado: <1ms adicional por batch de 38 vagas.
    // Ganho de reutilização: marginal (<0.1% do custo total de curagem).
    // Custo de bug futuro por race condition oculta: alto (dias de debug).
    //
    // Se performance virar gargalo, mover compilação para `ensureMeta` e
    // revisitar com benchmarks concretos.
    const localRegex = new RegExp(patternSource, patternFlags);

    const matches: Array<{ key: string; position: number; tokenCount: number; label: string }> = [];

    // matchAll retorna iterator sem mutação externa de estado
    for (const m of normalized.matchAll(localRegex)) {
        // v5.2 — B1: key já está na forma normalizada (mesmo que input)
        const key = m[0].toLowerCase();
        const info = keyMap.get(key);
        if (!info) continue;
        if (m.index === undefined) continue;

        matches.push({
            key,
            position: m.index,
            tokenCount: info.tokenCount,
            label: info.label
        });
    }

    if (matches.length === 0) return { hit: false };

    // Heurística: mais tokens > menos tokens; em empate, menor posição vence
    matches.sort((a, b) => {
        if (a.tokenCount !== b.tokenCount) return b.tokenCount - a.tokenCount;
        return a.position - b.position;
    });

    const winner = matches[0];

    // v5.1 — valida canônico destino contra allowedForPreResolution
    // (subconjunto de canônicos aprovados para auto-decisão)
    if (!allowedForPreResolution.has(winner.label)) {
        return { hit: false };
    }

    return {
        hit: true,
        matched_key: winner.key,
        canonical_role_label: winner.label
    };
}

/**
 * Força recompilação — usar apenas em testes.
 * Nota v5.1: em produção, a recompilação é automática quando o número de
 * keys em domain_synonyms.json muda (via check em ensureMeta).
 */
export function invalidateDomainSynonymsLookupCache(): void {
    compiledMeta = null;
}
```

**Notas críticas:**

- **Sem race condition:** cada chamada cria `localRegex` a partir de `patternSource + patternFlags`. `matchAll` é iterator sem estado compartilhado. Workers concorrentes não interferem uns com os outros.
- **Zero query per-call:** validação de canônico destino é `Set.has()` em memória, preloaded pelo `getFullTaxonomyCache`.
- **Heurística correta:** número de tokens matching é o critério principal; posição no título é desempate.

### 7.5 Novo arquivo: `lib/pipeline/suggested-roles-builder.ts`

Implementa a Camada 2 simplificada — apenas overlap de tokens, sem embedding.

```typescript
import { SupabaseClient } from '@supabase/supabase-js';
import { getFullTaxonomyCache } from '@/lib/pipeline/taxonomy-cache';
import { normalizeTitle } from '@/lib/pipeline/text-processing';

const MAX_SUGGESTED_ROLES = 12;

export async function buildSuggestedRoles(
    rawTitle: string,
    supabase: SupabaseClient
): Promise<string[]> {
    const normalized = normalizeTitle(rawTitle);
    const {
        familySynonyms,
        domains,
        validCanonicalLabels,
        vacancyCountByLabel
    } = await getFullTaxonomyCache(supabase);

    const familyMatch = findFamilyMatch(
        normalized,
        familySynonyms,
        validCanonicalLabels,
        vacancyCountByLabel
    );
    if (familyMatch && familyMatch.length > 0) {
        return familyMatch.slice(0, MAX_SUGGESTED_ROLES);
    }

    const domainMatch = findDomainMatch(
        normalized,
        domains,
        validCanonicalLabels,
        vacancyCountByLabel
    );
    if (domainMatch && domainMatch.length > 0) {
        return domainMatch.slice(0, MAX_SUGGESTED_ROLES);
    }

    return [];
}

/**
 * Encontra família pelo overlap de tokens únicos do título,
 * filtra canônicos inválidos, e ranqueia por vacancy_count DESC.
 *
 * v5.1 — usa Set de tokens únicos do título para evitar contagem inflada
 * quando palavras se repetem (ex: "Engenheiro Engenheiro Engenheiro").
 */
function findFamilyMatch(
    normalizedTitle: string,
    families: Record<string, string[]>,
    validLabels: Set<string>,
    vacancyCount: Map<string, number>
): string[] | null {
    const uniqueTokens = new Set(normalizedTitle.split(/\s+/).filter(Boolean));

    let bestFamily: string | null = null;
    let bestScore = 0;

    for (const [familyName, _canonicalList] of Object.entries(families)) {
        const familyTokens = expandFamilyTokens(familyName);
        let overlap = 0;
        for (const t of uniqueTokens) {
            if (familyTokens.has(t)) overlap++;
        }

        if (overlap > bestScore) {
            bestScore = overlap;
            bestFamily = familyName;
        }
    }

    if (!bestFamily) return null;
    // v5.3 (C4): piso implícito é `bestScore > 0` — bestFamily só sai de null
    // se algum overlap foi encontrado. Diferente do findDomainMatch que tem
    // piso explícito `< 2`, famílias são categorias mais amplas onde 1 token
    // único já é informativo (ex: token "engenharia" casa com família
    // Engenharia mesmo sem contexto adicional). Ver §7.5 rationale.

    // v5.1 — shortlist usa validCanonicalLabels (active + pending visíveis).
    // NÃO filtra por allowedForPreResolution — Camada 2 é SUGESTÃO ao LLM,
    // não auto-decisão. LLM pode escolher livremente da shortlist.
    return families[bestFamily]
        .filter(label => validLabels.has(label))
        .sort((a, b) => (vacancyCount.get(b) ?? 0) - (vacancyCount.get(a) ?? 0));
}

/**
 * Encontra domain pelo overlap de tokens únicos, filtra canônicos inválidos,
 * e ranqueia por vacancy_count DESC — tudo em memória, zero query.
 *
 * v5.1 — dedup de tokens + piso de overlap >= 2 tokens para evitar shortlist
 * enviesada por token genérico (ex: "Analista" sozinho casaria com muitas famílias).
 */
function findDomainMatch(
    normalizedTitle: string,
    domains: Record<string, string[]>,
    validLabels: Set<string>,
    vacancyCount: Map<string, number>
): string[] | null {
    const uniqueTokens = new Set(normalizedTitle.split(/\s+/).filter(Boolean));

    let bestDomain: string | null = null;
    let bestScore = 0;

    for (const [domainName, _canonicalList] of Object.entries(domains)) {
        const domainTokens = expandFamilyTokens(domainName);
        let overlap = 0;
        for (const t of uniqueTokens) {
            if (domainTokens.has(t)) overlap++;
        }

        if (overlap > bestScore) {
            bestScore = overlap;
            bestDomain = domainName;
        }
    }

    if (!bestDomain) return null;

    // v5.1 — piso: aceitar match apenas se houver ao menos 2 tokens de overlap.
    // Evita shortlist baseada em token único genérico.
    //
    // v5.2 — B2: implementação ajustada para `< 2` (estava `< 1` na v5.1,
    // contradizendo o comentário). Rationale: token único genérico
    // ("analista", "engenheiro", "coordenador") sozinho casa com dezenas de
    // domains distintos — disparar shortlist por ele enviesa para canônicos
    // populares via vacancy_count. Exigir ≥ 2 tokens força match mais específico.
    //
    // findFamilyMatch mantém `bestScore > 0` (sem piso explícito) porque
    // famílias são categorias mais amplas (ex: "Engenharia") onde token único
    // é informativo — e se não houver nenhum match, retorna null naturalmente.
    if (bestScore < 2) return null;

    return domains[bestDomain]
        .filter(label => validLabels.has(label))
        .sort((a, b) => (vacancyCount.get(b) ?? 0) - (vacancyCount.get(a) ?? 0));
}

/**
 * Expande tokens da família/domain, tratando abreviações conhecidas.
 */
function expandFamilyTokens(name: string): Set<string> {
    const baseTokens = name
        .toLowerCase()
        .replace(/-/g, ' ')
        .split(/\s+/)
        .filter(Boolean);

    const expansions: Record<string, string[]> = {
        'rh': ['recursos', 'humanos'],
        'td': ['desenvolvimento'],
        'bi': ['business', 'intelligence'],
        'ti': ['tecnologia', 'informacao']
    };

    const expanded = new Set<string>(baseTokens);
    for (const token of baseTokens) {
        const exp = expansions[token];
        if (exp) exp.forEach(t => expanded.add(t));
    }

    return expanded;
}
```

**Correções v5 nesta seção:**

- **Zero query no caminho-quente:** ranking por `vacancy_count DESC` usa `Map<string, number>` preloaded em `getFullTaxonomyCache`. Camada 2 agora é 100% in-memory.
- **Consistência active+pending:** `findFamilyMatch` e `findDomainMatch` usam `validCanonicalLabels` (que inclui ambos) — mesmo critério da Camada 1. Na v4, Camada 2 filtrava só `active`, inconsistente com Camada 1 que aceitava ambos.
- **Função `findDomainMatch` não precisa mais ser async** — removeu `await supabase.from(...)`.

### 7.6 Novo script: `scripts/backfill-job-description-hash.ts`

Script standalone para popular `description_hash`, `prompt_structure_version`, `prompt_content_version` em todas as vagas curadas existentes.

**v5.1 — decisão arquitetural crítica:** vagas curadas ANTES do deploy do PR2 foram classificadas com o prompt antigo (pré-v5.1). Se elas fossem carimbadas com o `PROMPT_STRUCTURE_VERSION` atual (`'v1'`), passariam a ser âncoras elegíveis para a Camada 0 — propagando erros sistêmicos do prompt antigo de forma determinística. A v5.1 marca todas as vagas legadas com `prompt_structure_version = 'legacy'`; a query da Camada 0 filtra explicitamente `!= 'legacy'`. Apenas vagas novas (curadas pós-deploy do PR2, via pipeline com prompt novo) viram âncoras. O pool de âncoras cresce organicamente a partir do LLM populando os novos campos em `persistCuratedJob` (§7.11).

**v5.1 — script idempotente:** condição de seleção é `prompt_structure_version IS NULL OR description_hash IS NULL` (OR em vez de AND). Reexecuções não reprocessam vagas já tratadas mas cobrem casos de falha parcial em que uma coluna ficou populada e a outra não.

```typescript
import { createClient } from '@supabase/supabase-js';
import crypto from 'crypto';
import { normalizeDescriptionForHash } from '../lib/minhash';
import {
    PROMPT_STRUCTURE_VERSION,
    PROMPT_CONTENT_VERSION
} from '../lib/pipeline/prompt-version.generated';

const BATCH_SIZE = 200;
const DELAY_BETWEEN_BATCHES_MS = 100;
const MIN_WORDS = 80;
const MAX_RETRY_ATTEMPTS = 3;

// v5.1 — marker explícito para vagas pré-v5.1
// Camada 0 filtra este valor via .neq('prompt_structure_version', 'legacy')
const LEGACY_MARKER = 'legacy';

async function sleep(ms: number) {
    return new Promise(r => setTimeout(r, ms));
}

async function updateWithBackoff(
    supabase: ReturnType<typeof createClient>,
    jobId: string,
    payload: Record<string, unknown>
): Promise<{ success: boolean; error?: string }> {
    for (let attempt = 0; attempt < MAX_RETRY_ATTEMPTS; attempt++) {
        const { error } = await supabase
            .from('job_postings')
            .update(payload)
            .eq('id', jobId);

        if (!error) return { success: true };

        // Exponential backoff: 500ms, 2s, 8s
        if (attempt < MAX_RETRY_ATTEMPTS - 1) {
            await sleep(500 * Math.pow(4, attempt));
        } else {
            return { success: false, error: error.message };
        }
    }
    return { success: false, error: 'Max retries exhausted' };
}

async function main() {
    // v5.6 (ChatGPT G1): NEXT_PUBLIC_SUPABASE_URL é a env padrão do projeto
    // (confirmado em lib/supabase/admin.ts, lib/audit/auth-events.ts,
    //  lib/supabase-server.ts, lib/supabase/server.ts — 4+ ocorrências).
    // SUPABASE_URL sem prefixo não existe em nenhum lugar e o script abortaria
    // no guard inicial se usasse essa env.
    const supabase = createClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.SUPABASE_SERVICE_ROLE_KEY!
    );

    // v5.2 — B7: captura timestamp de cutoff no início da execução.
    // Vagas com curated_at >= cutoff NÃO são tocadas pelo backfill — elas
    // foram curadas pelo código novo (pós-deploy) e já têm os campos
    // corretos populados por persistCuratedJob. Isso previne race entre
    // deploy e backfill marcando vagas novas incorretamente como 'legacy'.
    const DEPLOY_CUTOFF_TIMESTAMP = new Date().toISOString();

    console.log('Iniciando backfill de description_hash e prompt_versions (v5.2)...');
    console.log(`Marker para vagas legadas: prompt_structure_version='${LEGACY_MARKER}'`);
    console.log(`Cutoff temporal: vagas com curated_at >= ${DEPLOY_CUTOFF_TIMESTAMP} serão puladas.`);
    console.log(`Exceção: vagas human_validated=true recebem prompt_structure_version='${PROMPT_STRUCTURE_VERSION}'.`);

    let totalProcessed = 0;
    let totalSkippedShort = 0;
    let totalSkippedPostDeploy = 0;  // v5.2 — B7
    let totalHumanValidated = 0;     // v5.2 — B5
    let totalFailed = 0;
    let hasMore = true;
    let lastId: string | null = null;
    const startTime = Date.now();

    while (hasMore) {
        // v5.1 — condição idempotente (OR em vez de AND)
        // v5.2 — B7: filtra apenas vagas pré-cutoff
        // v5.2 — B5: seleciona human_validated para tratamento especial
        // v5.5 (DeepSeek #26 / GenSpark #2): CLEANUP_MODE inverte a lógica
        // temporal — em vez de pegar vagas pré-cutoff, pega vagas DENTRO do
        // intervalo cleanup (órfãs entre deploy e fim do backfill).
        let query = supabase
            .from('job_postings')
            .select('id, requirements, human_validated, curated_at')
            .eq('curation_status', 'curated')
            .or('description_hash.is.null,prompt_structure_version.is.null')
            .order('id', { ascending: true })
            .limit(BATCH_SIZE);

        if (process.env.CLEANUP_MODE === 'true') {
            const cleanupMin = process.env.CLEANUP_MIN_CURATED_AT;
            const cleanupMax = process.env.CLEANUP_MAX_CURATED_AT;
            if (!cleanupMin || !cleanupMax) {
                console.error('CLEANUP_MODE=true requer CLEANUP_MIN_CURATED_AT e CLEANUP_MAX_CURATED_AT');
                process.exit(1);
            }
            query = query.gte('curated_at', cleanupMin).lte('curated_at', cleanupMax);
            console.log(`Modo cleanup: filtra curated_at BETWEEN ${cleanupMin} AND ${cleanupMax}`);
        } else {
            query = query.lt('curated_at', DEPLOY_CUTOFF_TIMESTAMP);
        }

        if (lastId) query = query.gt('id', lastId);

        const { data: batch, error } = await query;

        if (error) {
            console.error('Erro na busca:', error);
            process.exit(1);
        }

        if (!batch || batch.length === 0) {
            hasMore = false;
            break;
        }

        for (const job of batch) {
            try {
                // v5.2 — B7: double-check do cutoff (defesa contra race de leitura)
                // v5.6 (Gemini #3): usar .getTime() em vez de .toISOString() para
                // comparação. Strings ISO podem divergir em formatação de milissegundos
                // (`.000Z` vs `.Z`) dependendo do driver, produzindo comparação
                // lexicográfica errada silenciosamente. Comparação numérica é inequívoca.
                if (job.curated_at && new Date(job.curated_at).getTime() >= new Date(DEPLOY_CUTOFF_TIMESTAMP).getTime()) {
                    totalSkippedPostDeploy++;
                    continue;
                }

                // Cast explícito de Json | null para shape esperado
                const requirements = job.requirements as { description?: string } | null;
                const description = requirements?.description ?? '';
                const normalized = normalizeDescriptionForHash(description);
                const words = normalized.split(/\s+/).filter(Boolean);

                let descriptionHash: string | null = null;
                if (words.length >= MIN_WORDS) {
                    descriptionHash = crypto
                        .createHash('sha256')
                        .update(normalized)
                        .digest('hex');
                } else {
                    totalSkippedShort++;
                }

                // v5.2 — B5: exceção para human_validated
                // Vagas validadas manualmente (Canonical GSI, Stone, AgileEngine, etc.)
                // são confiáveis mesmo tendo sido curadas pelo prompt antigo. Elas
                // devem servir como âncoras da Camada 0 desde o dia 1 do rollout.
                // Receberem PROMPT_STRUCTURE_VERSION atual (não 'legacy') permite isso.
                const structureVersion = job.human_validated === true
                    ? PROMPT_STRUCTURE_VERSION
                    : LEGACY_MARKER;

                if (job.human_validated === true) {
                    totalHumanValidated++;
                }

                const result = await updateWithBackoff(supabase, job.id, {
                    description_hash: descriptionHash,
                    prompt_structure_version: structureVersion,
                    prompt_content_version: PROMPT_CONTENT_VERSION
                });

                if (!result.success) {
                    totalFailed++;
                    console.error(`Erro definitivo no ID ${job.id} após retries:`, result.error);
                }
            } catch (err) {
                totalFailed++;
                console.error(`Exceção no ID ${job.id}:`, err);
                // Continua o loop — não aborta backfill por uma vaga
            }
        }

        totalProcessed += batch.length;
        lastId = batch[batch.length - 1].id;

        // v5.1 — log a cada 1000 registros para visibilidade em backfills grandes
        if (totalProcessed % 1000 === 0 || batch.length < BATCH_SIZE) {
            const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
            console.log(
                `[${elapsed}s] Processadas: ${totalProcessed} | ` +
                `Curtas (hash=NULL): ${totalSkippedShort} | ` +
                `Pós-deploy (puladas): ${totalSkippedPostDeploy} | ` +
                `Human_validated (v1): ${totalHumanValidated} | ` +
                `Falhas: ${totalFailed}`
            );
        }

        if (batch.length < BATCH_SIZE) hasMore = false;
        else await sleep(DELAY_BETWEEN_BATCHES_MS);
    }

    const totalTime = ((Date.now() - startTime) / 1000).toFixed(1);
    console.log(`\nBackfill concluído em ${totalTime}s.`);
    console.log(`Total: ${totalProcessed} | Curtas: ${totalSkippedShort} | Falhas: ${totalFailed}`);

    if (totalFailed > 0) {
        console.warn('\n⚠ Há falhas. Reexecute o script — é idempotente e cobre apenas falhas pendentes.');
        process.exit(1);
    }
}

main().catch(err => {
    console.error('Fatal:', err);
    process.exit(1);
});
```

**Ordem de execução crítica (ver §8.1):** backfill roda **após** a migração SQL e **antes** de qualquer ativação de tráfego na Camada 0. Sem essa ordem, primeiras horas de 10% tráfego têm Camada 0 com 0% hit-rate silencioso.

**Reexecução segura:** o script pode ser executado múltiplas vezes. Cada execução processa apenas vagas que ainda têm `description_hash IS NULL` ou `prompt_structure_version IS NULL`. Em backfill interrompido, basta reexecutar.

**v5.2 — Exceção `human_validated` (B5):** vagas com `human_validated=true` (marcadas manualmente pelo Onsly antes do deploy via procedimento análogo ao Canonical GSI: Stone, AgileEngine, etc.) recebem `prompt_structure_version=PROMPT_STRUCTURE_VERSION` (valor atual, ex: `'v1'`) em vez de `'legacy'`. Isso garante que essas vagas sirvam como âncoras da Camada 0 desde o dia 1 do rollout — sem elas, o pool inicial seria zero e o estágio 10% teria hit rate artificialmente em zero. Volume esperado: ~100 vagas inicialmente (Canonical GSI + Stone + AgileEngine).

**v5.2 — Cutoff temporal (B7):** o script captura `DEPLOY_CUTOFF_TIMESTAMP` no início da execução e filtra apenas vagas com `curated_at < cutoff`. Vagas curadas após o deploy (passo 13) mas antes do backfill (passo 14) **não são tocadas** — elas já foram processadas pelo código novo e têm os campos populados via `persistCuratedJob` (§7.11). Sem esse cutoff, vagas curadas nesse intervalo podem receber `legacy` por engano, tornando-se inelegíveis como âncoras.

### 7.7 Modificação em `lib/pipeline/batch-processor.ts`

Integração das Camadas 0, 1, 2 antes da chamada LLM.

**Pré-requisitos desta seção (correções descobertas na revisão v4):**

1. **Estender `PreparedJob` em `lib/pipeline/types.ts`** — adicionar 3 campos opcionais.
2. **Estender `RunCounters` em `lib/pipeline/types.ts`** — adicionar `preCheckHit: number`.
3. **Atualizar `createRunCounters()` em `lib/pipeline/types.ts`** — inicializar `preCheckHit: 0`.
4. **Criar `lib/pipeline/persist-precheck.ts`** — função dedicada `persistPrecheckResult` (ver §7.10 abaixo).
5. **Usar constante real `SYSTEM_USER_ID`** importada de `constants.ts`.
6. **Feature flag posicionada antes do loop de camadas** — ver snippet.

#### 7.7.1 Mudanças em `lib/pipeline/types.ts`

```typescript
// Adicionar 3 campos opcionais em PreparedJob
export interface PreparedJob {
    id: string;
    text: string;
    truncated: boolean;
    title?: string;
    company_name?: string;
    // v5.4: descrição bruta (pré-LLM) propagada ao persistCuratedJob
    // para popular job_postings.original_description. Durante transição
    // (v5.4 → sprint de remoção de requirements), pode vir de
    // requirements.description ou de original_description diretamente.
    rawDescription?: string | null;
    // Novos campos v5 — populados pelas Camadas 0/1/2
    canonical_already_resolved?: string | null;
    suggested_roles?: string[];
    generated_description_hash?: string | null;
    precheck_hit?: boolean;
    curation_source?: string;
    // v5.1 — modo debug
    precheck_only_miss?: boolean;
    // v5.2 — B3: campos para override server-side em persistCuratedJob
    pre_resolved_canonical_label?: string | null;
    pre_resolved_canonical_role_id?: string | null;
}

// v5.4 (GS-R6-8): enum centralizado de miss reasons — factory de contadores
// e union type do PreCheck usam a mesma fonte, evitando drift silencioso.
export const MISS_REASONS = [
    'short_description',
    'no_match_in_db',
    'no_quorum',
    'conflict_detected',
    'title_guard_failed',
    'canonical_not_allowed',
] as const;

export type MissReason = typeof MISS_REASONS[number];

// Adicionar contadores em RunCounters
export interface RunCounters {
    // ... campos existentes ...
    preCheckHit: number;              // novo v5
    preCheckMiss: number;             // novo v5.1 — para taxa de hit
    preCheckConflict: number;         // novo v5.1 — detecções de conflito
    layer1Hit: number;                // novo v5.1 — hits em domain_synonyms
    layer2HintCount: number;          // novo v5.1 — total de sugestões enviadas ao LLM
    precheckUpdateConflict: number;   // novo v5.1 — races de estado detectadas
    // v5.4 (GS-R6-8): tipado via MissReason, factory via Object.fromEntries
    preCheckMissByReason: Record<MissReason, number>;
    // v5.2 — adicional: LLM desobedecendo pré-resolução
    llmDisobeyedPreResolution: number;
    // v5.4 (GS-R6-1): vagas com canonical retornado pelo LLM inválido
    llmOutputQuarantined: number;
}

// Atualizar factory para inicializar
export function createRunCounters(): RunCounters {
    return {
        // ... campos existentes zerados ...
        preCheckHit: 0,
        preCheckMiss: 0,
        preCheckConflict: 0,
        layer1Hit: 0,
        layer2HintCount: 0,
        precheckUpdateConflict: 0,
        // v5.4 (GS-R6-8): gerado via Object.fromEntries para sincronia
        // automática com MISS_REASONS — novo reason adicionado ao enum
        // gera automaticamente contador zerado aqui, eliminando drift.
        preCheckMissByReason: Object.fromEntries(
            MISS_REASONS.map(k => [k, 0])
        ) as Record<MissReason, number>,
        llmDisobeyedPreResolution: 0,
        llmOutputQuarantined: 0,  // v5.4 (GS-R6-1)
    };
}
```

**Tabela contador → ponto de incremento (atualizada v5.4):** a tabela abaixo ganha duas linhas:

| Contador | Arquivo | Localização | Condição |
|---|---|---|---|
| `llmOutputQuarantined` | `persist-curation.ts` §7.11.1 | dentro de `if (!canonicalRole)` após `upsertCanonicalRole` | LLM retornou canonical não-resolvível |

#### 7.7.1.1 Tabela contador → ponto exato de incremento (v5.2 — N8)

Mapeamento normativo de cada contador ao local no código onde é incrementado. Implementador deve seguir essa tabela exatamente — dashboards são desenhados em cima e contadores não incrementados aparecem como zero no painel sem que ninguém perceba.

| Contador | Arquivo | Localização | Condição |
|---|---|---|---|
| `preCheckHit` | `batch-processor.ts` §7.7.2 | após `persistPrecheckResult` com `result.success === true` | hit path da Camada 0 |
| `preCheckMiss` | `batch-processor.ts` §7.7.2 | após `precheckDescriptionHash` retornar `hit: false` | miss path da Camada 0 (qualquer razão) |
| `preCheckConflict` | `batch-processor.ts` §7.7.2 | dentro do miss path quando `hashResult.miss_reason === 'conflict_detected'` | registro secundário do conflito |
| `preCheckMissByReason.<reason>` | `batch-processor.ts` §7.7.2 | dentro do miss path, switch sobre `hashResult.miss_reason` | categorização para `precheck_miss_summary` |
| `layer1Hit` | `batch-processor.ts` §7.7.2 | após `lookupDomainSynonyms` retornar `hit: true` | hit path da Camada 1 |
| `layer2HintCount` | `batch-processor.ts` §7.7.2 | após `buildSuggestedRoles` retornar array não-vazio | somar `suggestedRoles.length` (não +1) |
| `precheckUpdateConflict` | `batch-processor.ts` §7.7.2 | dentro de `if (result.concurrent_update_detected)` | race no `persistPrecheckResult` |
| `llmDisobeyedPreResolution` | `persist-curation.ts` §7.11 | dentro de `if (normalized.canonicalRole !== pre_resolved_canonical_label)` | antes de gravar evento `llm_disobeyed_pre_resolution` |

**Teste de fumaça:** depois do PR2 deployado com rollout=10%, rodar batch com 100 vagas sintéticas e verificar que todos os contadores relevantes foram incrementados alguma vez (exceto os que dependem de condições raras como `preCheckConflict` ou `precheckUpdateConflict`).

#### 7.7.2 Snippet de integração em `batch-processor.ts`

```typescript
import { precheckDescriptionHash } from '@/lib/pipeline/precheck-description-hash';
import { lookupDomainSynonyms } from '@/lib/pipeline/domain-synonyms-lookup';
import { buildSuggestedRoles } from '@/lib/pipeline/suggested-roles-builder';
import { persistPrecheckResult } from '@/lib/pipeline/persist-precheck';
import { isJobInRollout } from '@/lib/pipeline/rollout-sampling';  // v5.1
import { getFullTaxonomyCache } from '@/lib/pipeline/taxonomy-cache';
import { SYSTEM_USER_ID } from '@/lib/pipeline/constants';
import {
    PROMPT_STRUCTURE_VERSION,
    PROMPT_CONTENT_VERSION
} from '@/lib/pipeline/prompt-version.generated';

// ============================================================
// v5.1 — rollout gradual via PIPELINE_V3_ROLLOUT_PERCENT
// Valores: 0 (off), 10, 25, 50, 75, 100 (all)
// Hash determinístico por ID garante que mesma vaga fica do mesmo lado
// ============================================================
const precheckOnly = process.env.PIPELINE_V3_PRECHECK_ONLY === 'true';

// Pre-load cache uma vez por batch (não por vaga)
// v5.3 (C3): destructuring inclui roleIdByLabel para uso direto no hit path
// da Camada 1 — elimina função auxiliar lookupRoleIdByLabel e chamada await
// redundante ao cache (que, mesmo com TTL, é custo desnecessário por vaga)
const { allowedForPreResolution, roleIdByLabel } = await getFullTaxonomyCache(ctx.supabase);

// Classificar jobs: dentro do rollout vs fora
const jobsInRollout = jobs.filter(j => isJobInRollout(j.id));
const jobsLegacy = jobs.filter(j => !isJobInRollout(j.id));

for (const job of jobsInRollout) {
    // Cast explícito para JSONB (requirements: Json | null)
    const requirements = job.requirements as { description?: string } | null;
    const description = requirements?.description ?? '';
    const candidateTitle = job.title ?? '';

    // ═══════════════ CAMADA 0 ═══════════════
    const hashResult = await precheckDescriptionHash(
        description,
        candidateTitle,
        PROMPT_STRUCTURE_VERSION,
        allowedForPreResolution,
        ctx.supabase
    );

    if (hashResult.hit) {
        const result = await persistPrecheckResult(ctx.supabase, {
            jobId: job.id,
            canonicalRoleId: hashResult.canonical_role_id,
            canonicalRoleLabel: hashResult.canonical_role_label,
            skills: hashResult.skills,
            seniorityInferred: hashResult.seniority_inferred,
            workModel: hashResult.work_model,
            descriptionCurated: hashResult.description_curated,
            descriptionHash: hashResult.generated_hash,
            promptStructureVersion: PROMPT_STRUCTURE_VERSION,
            promptContentVersion: PROMPT_CONTENT_VERSION,
            curationSource: 'precheck_description_hash',
            canonicalResolvedAt: new Date().toISOString(),
            // v5.4 (GS-R6-10): optimistic lock propagado do hit
            anchorJobId: hashResult.anchor_job_id,
            anchorUpdatedAt: hashResult.anchor_updated_at
        });

        if (result.success) {
            // Evento + audit metadata
            await ctx.supabase.from('events').insert({
                event_name: 'canonical_role_curated_via_precheck',
                resource_type: 'job_posting',
                resource_id: job.id,
                actor: 'pipeline',
                actor_id: SYSTEM_USER_ID,
                new_state: {
                    canonical_role_id: hashResult.canonical_role_id,
                    canonical_role_label: hashResult.canonical_role_label,
                    matched_source_ids: hashResult.matched_source_ids,
                    anchor_title: hashResult.anchor_title,
                    reason: hashResult.reason
                },
                run_id: ctx.jobRunId
            });

            // v5.2 — N9: trilha forense em `events` (não em `ai_usage_logs`,
            // que é semanticamente para chamadas a modelos de IA). Camada 0
            // não invoca IA — usa apenas hash + lookup em banco.
            // O evento canonical_role_curated_via_precheck acima já grava a
            // decisão operacional; este evento adicional carrega metadata
            // rica para forense pós-incidente.
            await ctx.supabase.from('events').insert({
                event_name: 'precheck_hit_forensic',
                resource_type: 'job_posting',
                resource_id: job.id,
                actor: 'pipeline',
                actor_id: SYSTEM_USER_ID,
                metadata: {
                    layer: 0,
                    reason: hashResult.reason,
                    matched_source_ids: hashResult.matched_source_ids,
                    anchor_title: hashResult.anchor_title,
                    candidate_title: candidateTitle,
                    canonical_role_label: hashResult.canonical_role_label,
                    prompt_structure_version: PROMPT_STRUCTURE_VERSION,
                    description_hash: hashResult.generated_hash
                }
            });

            ctx.counters.preCheckHit++;
            job.precheck_hit = true;
        } else if (result.concurrent_update_detected) {
            // v5.3 (GS8): race condition detectada — vaga foi curada por outro
            // caminho (manual, CRON concorrente, reprocessing) entre o hit da
            // Camada 0 e o UPDATE. Não é falha operacional — é um sinal
            // esperado de contenção. Incrementar contador e gravar evento
            // forense específico evita que races virem perda silenciosa de
            // trilha de auditoria.
            ctx.counters.precheckUpdateConflict++;

            await ctx.supabase.from('events').insert({
                event_name: 'precheck_update_conflict',
                resource_type: 'job_posting',
                resource_id: job.id,
                actor: 'pipeline',
                actor_id: SYSTEM_USER_ID,
                reason: 'Camada 0 gerou hit mas UPDATE rejeitado pelo guard de curation_status=pending — vaga foi curada por outro caminho no meio do batch',
                metadata: {
                    layer: 0,
                    intended_canonical_role_id: hashResult.canonical_role_id,
                    intended_canonical_role_label: hashResult.canonical_role_label,
                    reason_hit: hashResult.reason,
                    run_id: ctx.jobRunId
                }
            });

            // Vaga já está curada em outro caminho — não reprocessar via LLM.
            // Ela sairá do conjunto `jobsForLLM` naturalmente porque o status
            // no banco não é mais 'pending'.
        }
        continue;
    }

    // Miss — guarda hash gerado para persistir com resultado do LLM
    job.generated_description_hash = hashResult.generated_hash;

    // v5.2 — N2: categoriza miss_reason para evento agregado ao fim do batch
    ctx.counters.preCheckMiss++;
    if (hashResult.miss_reason) {
        ctx.counters.preCheckMissByReason[hashResult.miss_reason]++;

        // Contador secundário dedicado para conflict (já era rastreado isoladamente)
        if (hashResult.miss_reason === 'conflict_detected') {
            ctx.counters.preCheckConflict++;
        }
    }

    // v5.1 — `precheck_only` real: miss também pula LLM (modo debug)
    if (precheckOnly) {
        job.precheck_only_miss = true;
        continue;
    }

    // ═══════════════ CAMADA 1 ═══════════════
    const domainResult = await lookupDomainSynonyms(
        candidateTitle,
        allowedForPreResolution,  // v5.1 — passa whitelist
        ctx.supabase
    );

    if (domainResult.hit) {
        // Nomes de campo SEM prefixo __ — devem bater com buildUserPrompt
        job.canonical_already_resolved = domainResult.canonical_role_label;
        job.curation_source = 'layer_1_domain_synonyms';
        // v5.2 — B3: contadores e campos para override em persistCuratedJob
        ctx.counters.layer1Hit++;
        job.pre_resolved_canonical_label = domainResult.canonical_role_label;
        // v5.3 (C3): lookup direto no Map do cache (já carregado no topo do batch).
        // Sem await, sem função auxiliar, sem chamada redundante a getFullTaxonomyCache.
        job.pre_resolved_canonical_role_id = roleIdByLabel.get(domainResult.canonical_role_label) ?? null;
    }

    // ═══════════════ CAMADA 2 ═══════════════
    if (!domainResult.hit) {
        job.suggested_roles = await buildSuggestedRoles(
            candidateTitle,
            ctx.supabase
        );
        // v5.2 — N8: incrementa layer2HintCount pelo TAMANHO da shortlist
        if (job.suggested_roles && job.suggested_roles.length > 0) {
            ctx.counters.layer2HintCount += job.suggested_roles.length;
        }
    } else {
        job.suggested_roles = [];
    }
}

// ============================================================
// v5.2 — N2: evento agregado de miss_reason ao fim do batch
// Permite observabilidade sobre por que a Camada 0 não está funcionando
// sem descartar a informação em cada miss individual.
// ============================================================
if (jobsInRollout.length > 0 && ctx.counters.preCheckMiss > 0) {
    await ctx.supabase.from('events').insert({
        event_name: 'precheck_miss_summary',
        resource_type: 'batch_run',
        resource_id: ctx.jobRunId,
        actor: 'pipeline',
        actor_id: SYSTEM_USER_ID,
        metadata: {
            run_id: ctx.jobRunId,
            jobs_in_rollout: jobsInRollout.length,
            total_misses: ctx.counters.preCheckMiss,
            miss_by_reason: { ...ctx.counters.preCheckMissByReason }
        }
    });
}

// Filtrar vagas que precisam de LLM:
// - jobsLegacy: todas (pipeline sem Camadas)
// - jobsInRollout: as que não tiveram hit Camada 0 e não são precheck_only_miss
const jobsForLLM = [
    ...jobsLegacy,
    ...jobsInRollout.filter(j => !j.precheck_hit && !j.precheck_only_miss)
];

if (jobsForLLM.length > 0) {
    const userPrompt = buildUserPrompt(jobsForLLM);
    const response = await callLLM(userPrompt);
    // Processamento do response segue fluxo normal da Camada 3/4.
    // ATENÇÃO v5.2 (B3): persistCuratedJob (§7.11) DEVE receber PersistOptions com:
    //   - description_hash: job.generated_description_hash
    //   - prompt_structure_version: PROMPT_STRUCTURE_VERSION
    //   - prompt_content_version: PROMPT_CONTENT_VERSION
    //   - curation_source: job.curation_source ?? 'llm_direct'
    //   - canonical_resolved_at: job.canonical_already_resolved ? new Date().toISOString() : null
    //   - layer_2_hint_count: job.suggested_roles?.length ?? null
    //   - pre_resolved_canonical_label: job.pre_resolved_canonical_label
    //   - pre_resolved_canonical_role_id: job.pre_resolved_canonical_role_id
    // Sem os 2 últimos, override server-side não funciona quando LLM desobedece
    // pre-resolução da Camada 1.
}
```

**v5.3 (C3) — Inline direto no hit path da Camada 1.**

O hit path da Camada 1 resolve o `pre_resolved_canonical_role_id` diretamente via `roleIdByLabel.get()`, usando o Map já preloaded pelo destructuring no topo do batch (`const { allowedForPreResolution, roleIdByLabel } = await getFullTaxonomyCache(...)`).

Ganhos vs a abordagem da v5.2 (função auxiliar `lookupRoleIdByLabel` com `await`):
- Zero chamada `await` redundante ao cache TTL-based (mesmo sendo cache hit, o overhead de Promise existe)
- Zero função auxiliar vivendo isolada em `batch-processor.ts`
- Código mais direto: 1 linha `roleIdByLabel.get(label) ?? null` vs 5 linhas

**Nota:** `roleIdByLabel` é adicionado a `getFullTaxonomyCache` em §7.2 — um `Map<string, string>` com label → id, preloaded junto com `validCanonicalLabels` e `vacancyCountByLabel`. Zero query extra.

**Notas críticas (correções v5 e v5.1):**

- **Nomes de campo sem prefixo `__`:** `canonical_already_resolved`, `suggested_roles`, `generated_description_hash`, `precheck_hit`, `curation_source`. Esses nomes batem exatamente com o que `buildUserPrompt` lê (§7.8).
- **v5.1 rollout percentual:** via `isJobInRollout(job.id)` com SHA-256 dos últimos 2 bytes. Comportamento determinístico por ID.
- **v5.1 precheck_only real:** misses na Camada 0 agora também pulam LLM quando flag ativa. Vaga fica em `curation_status='pending'` para processamento posterior.
- **v5.1 allowedForPreResolution:** passado tanto para Camada 0 quanto Camada 1. Canônicos fora da whitelist geram miss, nunca auto-decisão.
- **v5.1 candidateTitle no precheck:** necessário para guarda de título anti-boilerplate.
- **Cast explícito de `requirements` como JSONB** em strict mode.
- **`persistPrecheckResult` é função dedicada (§7.10),** NÃO reusa `persistCuratedJob`.
- **`SYSTEM_USER_ID` (não `SYSTEM_ACTOR_ID`):** constante real em `lib/pipeline/constants.ts:81`.
- **`ctx.counters.preCheckHit`:** campo novo em `RunCounters` (§7.7.1).
- **`job.precheck_only_miss`:** novo campo em `PreparedJob` — sinaliza modo debug, não é hit real.

### 7.8 Modificação em `lib/pipeline/text-processing.ts` — `buildUserPrompt` ajustado

**Antes (assinatura original):**

```typescript
export function buildUserPrompt(jobs: Array<{ id: string; text: string }>): string {
    const items = jobs.map(j => `<job id="${j.id}">\n${j.text}\n</job>`).join('\n\n');
    return `Analyze the following ${jobs.length} job postings:\n\n${items}`;
}
```

**Depois (v5.1 — escape XML completo + CDATA no corpo):**

```typescript
/**
 * v5.1 — Escape XML completo dos 4 caracteres críticos.
 * Ordem importa: & primeiro para não duplicar entities.
 */
function escapeXmlAttr(s: string): string {
    return s
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;');
}

export function buildUserPrompt(jobs: Array<{
    id: string;
    text: string;
    canonical_already_resolved?: string | null;
    suggested_roles?: string[];
}>): string {
    const items = jobs.map(j => {
        // v5.4 (GS-R6-11): guard de tamanho patológico.
        // Texto bruto que venha com 50k+ chars indica scraper quebrado,
        // input adversário, ou bug upstream. buildUserPrompt não é lugar
        // para tratamento silencioso — falhar aqui protege o batch inteiro.
        // 50.000 chars ≈ 12.500 tokens ≈ 7× descrição típica (1.800 chars).
        if (j.text.length > 50_000) {
            throw new Error(
                `Job ${j.id} has text length ${j.text.length} chars (max 50000). ` +
                `Likely scraper issue or adversarial input. Investigate before retry.`
            );
        }

        const attrs: string[] = [`id="${escapeXmlAttr(j.id)}"`];

        if (j.canonical_already_resolved) {
            attrs.push(
                `canonical_already_resolved="${escapeXmlAttr(j.canonical_already_resolved)}"`
            );
        }

        if (j.suggested_roles?.length) {
            attrs.push(
                `suggested_roles="${escapeXmlAttr(j.suggested_roles.join(' | '))}"`
            );
        }

        // v5.1 — CDATA protege o corpo do texto contra quebra de framing
        // Descrições que contêm tags HTML, pseudo-XML, ou o próprio "</job>"
        // não corrompem a estrutura do prompt.
        // Edge case: se o texto contiver literalmente "]]>", dividir em 2 CDATA.
        const safeText = j.text.includes(']]>')
            ? j.text.replace(/\]\]>/g, ']]]]><![CDATA[>')
            : j.text;

        return `<job ${attrs.join(' ')}><![CDATA[\n${safeText}\n]]></job>`;
    }).join('\n\n');

    return `Analyze the following ${jobs.length} job postings:\n\n${items}`;
}
```

**Arquivos que chamam `buildUserPrompt` (precisam receber novos campos, mas não precisam mudar se não passarem):**

- `lib/pipeline/batch-processor.ts:53` — passa `canonical_already_resolved` e `suggested_roles`
- `lib/pipeline/llm-call.ts:39` — pode manter chamada simples (retrocompatível)
- Demais callers (testes, scripts pontuais) — retrocompatíveis

**Mudança de contrato:** novos campos são opcionais (`?`). Callers antigos que não passam os campos têm comportamento idêntico ao atual. Mesmo assim, executor deve listar os 4 callers na lista de arquivos modificados.

**v5.1 — notas sobre CDATA:**

- CDATA é um bloco bruto — nenhum escape dentro é interpretado, o que protege contra qualquer conteúdo da descrição (HTML, pseudo-XML, caracteres de controle).
- O único caractere sequencial proibido dentro de CDATA é `]]>`. v5.1 detecta e quebra em duas seções CDATA se aparecer.
- SYSTEM_PROMPT orienta o LLM a ler o conteúdo entre `<![CDATA[` e `]]>` como o texto da vaga.
- Atributos XML continuam precisando de escape (CDATA só funciona em content, não em attributes).

#### 7.8.1 Teste unitário obrigatório (v5.2 — B4)

Criar `tests/build-user-prompt.test.ts` com os seguintes casos:

```typescript
import { describe, it, expect } from 'vitest';
import { buildUserPrompt } from '@/lib/pipeline/text-processing';

describe('buildUserPrompt', () => {
    it('envelopa corpo em CDATA', () => {
        const out = buildUserPrompt([{ id: 'abc', text: 'Texto simples' }]);
        expect(out).toContain('<![CDATA[');
        expect(out).toContain(']]>');
        expect(out).toContain('Texto simples');
    });

    it('escapa atributos XML corretamente (4 caracteres críticos)', () => {
        const out = buildUserPrompt([{
            id: 'xyz',
            text: 'corpo',
            canonical_already_resolved: 'A & B > C < D " E'
        }]);
        expect(out).toContain('A &amp; B &gt; C &lt; D &quot; E');
        expect(out).not.toMatch(/canonical_already_resolved="[^"]*[<>&"][^"]*"/);
    });

    it('preserva texto contendo HTML sem quebrar framing', () => {
        const htmlText = '<p>Vaga para <b>Desenvolvedor</b></p>';
        const out = buildUserPrompt([{ id: 'abc', text: htmlText }]);
        expect(out).toContain(htmlText);  // HTML preservado literalmente dentro de CDATA
        // Deve ter exatamente 1 <job e 1 </job>
        expect((out.match(/<job /g) ?? []).length).toBe(1);
        expect((out.match(/<\/job>/g) ?? []).length).toBe(1);
    });

    it('preserva texto contendo pseudo-XML de outras tags', () => {
        const pseudoXml = 'Requisito: <job>tag interna</job> no texto';
        const out = buildUserPrompt([{ id: 'abc', text: pseudoXml }]);
        // Job interna não deve quebrar o framing — fica dentro do CDATA
        expect(out).toContain(pseudoXml);
        expect((out.match(/<\/job>/g) ?? []).length).toBe(1);  // só a tag de fechamento real
    });

    it('trata "]]>" no corpo sem quebrar CDATA', () => {
        // Este é o único caractere sequencial proibido dentro de CDATA
        const textWithCdataEnd = 'Foo ]]> Bar';
        const out = buildUserPrompt([{ id: 'abc', text: textWithCdataEnd }]);

        // O texto tem que aparecer de alguma forma (split em 2 CDATA)
        expect(out).toContain('Foo');
        expect(out).toContain('Bar');

        // Verificação de correção do split: substituição ']]>' por ']]]]><![CDATA[>'
        // cria duas seções CDATA consecutivas, o que é XML válido
        expect(out).toContain(']]]]><![CDATA[>');

        // Ainda deve ter apenas 1 tag </job> — framing preservado
        expect((out.match(/<\/job>/g) ?? []).length).toBe(1);
    });

    it('trata múltiplas ocorrências de "]]>" no mesmo texto', () => {
        const text = 'a]]>b]]>c';
        const out = buildUserPrompt([{ id: 'abc', text }]);
        expect(out).toContain('a');
        expect(out).toContain('b');
        expect(out).toContain('c');
        // 2 splits, 3 pedaços de conteúdo envelopados em 3 CDATAs (ou equivalente)
        expect((out.match(/]]]]><!\[CDATA\[>/g) ?? []).length).toBe(2);
    });

    it('gera prompt para múltiplas vagas com separadores corretos', () => {
        const out = buildUserPrompt([
            { id: '1', text: 'A' },
            { id: '2', text: 'B' },
            { id: '3', text: 'C' },
        ]);
        expect((out.match(/<job /g) ?? []).length).toBe(3);
        expect(out).toContain('Analyze the following 3 job postings');
    });

    it('escapa ID se contiver caracteres especiais', () => {
        // UUIDs normais são seguros mas defender contra edge cases
        const out = buildUserPrompt([{ id: 'abc"xyz', text: 'corpo' }]);
        expect(out).toContain('abc&quot;xyz');
    });
});
```

**Critério de aceite (PR1):** todos os 8 casos passam.

### 7.9 Modificação em `scripts/generate-prompt-version.ts`

Dividir geração de hash em duas dimensões com estratégia robusta.

**Decisão arquitetural (correção de bug descoberto na revisão v4):** a v4 propunha extrair um bloco do SYSTEM_PROMPT via regex marcador `SCHEMA JSON DE SAÍDA`. Verificação no repositório mostrou que esse marcador **não existe** no arquivo real. A regex sempre retornava `null`, caía no fallback `else content`, e `structureHash === contentHash` sempre — anulando a inovação arquitetural central do PR1.

**Correção:** `PROMPT_STRUCTURE_VERSION` passa a ser constante hardcoded no próprio arquivo gerado. É incrementada manualmente quando campos do schema JSON de output mudam (raro, evento consciente). `PROMPT_CONTENT_VERSION` continua sendo SHA-256 do arquivo inteiro (muda a cada ajuste de regra).

```typescript
import crypto from 'crypto';
import fs from 'fs';
import path from 'path';

const SYSTEM_PROMPT_PATH = path.join(process.cwd(), 'lib/pipeline/SYSTEM_PROMPT.ts');
const OUTPUT_PATH = path.join(process.cwd(), 'lib/pipeline/prompt-version.generated.ts');

/**
 * PROMPT_STRUCTURE_VERSION: incrementado manualmente quando o schema JSON de
 * output muda (adicionar/remover/renomear campo, mudar tipo de valor).
 *
 * Eventos que DEVEM incrementar:
 * - Adicionar novo campo ao output (ex: sentiment_score)
 * - Remover campo existente
 * - Renomear campo (canonical_role → role_canonical)
 * - Mudar tipo (seniority_inferred: string → seniority_inferred: number)
 *
 * Eventos que NÃO devem incrementar (usam prompt_content_version):
 * - Ajustar regra de desambiguação (A-E)
 * - Adicionar exemplo
 * - Modificar micro-regra Support
 * - Mudar Etapa 2 shortlist rules
 * - Ajustar tom de instrução
 *
 * Formato: 'v<N>' onde N é inteiro monotônico. Ao alterar, incrementar N.
 */
const STRUCTURE_VERSION = 'v1';

function main() {
    const content = fs.readFileSync(SYSTEM_PROMPT_PATH, 'utf-8');

    // Content hash: SHA-256 do arquivo inteiro, primeiros 8 chars hex
    const contentHash = crypto
        .createHash('sha256')
        .update(content)
        .digest('hex')
        .slice(0, 8);

    const output = `// Gerado automaticamente por scripts/generate-prompt-version.ts
// Não editar manualmente.

export const PROMPT_STRUCTURE_VERSION = '${STRUCTURE_VERSION}';
export const PROMPT_CONTENT_VERSION = '${contentHash}';

// Alias mantido para compatibilidade com código legado.
// IMPORTANTE: PROMPT_VERSION legacy aponta para STRUCTURE, não para CONTENT.
// Ver lib/pipeline/precheck-description-hash.ts — filtra por estrutura, não conteúdo.
export const PROMPT_VERSION = PROMPT_STRUCTURE_VERSION;
`;

    fs.writeFileSync(OUTPUT_PATH, output, 'utf-8');
    console.log('prompt-version.generated.ts atualizado.');
    console.log(`STRUCTURE: ${STRUCTURE_VERSION} (hardcoded)`);
    console.log(`CONTENT:   ${contentHash} (sha256 do arquivo)`);
}

main();
```

**Motivação dupla:**

1. `precheck-description-hash.ts` filtra por `prompt_structure_version`. Quando apenas conteúdo do prompt muda (ajuste de regra A-E por feedback de produção), `STRUCTURE_VERSION` permanece igual e pre-check continua funcionando. Só mudança de estrutura do output JSON invalida o pre-check, obrigando backfill explícito.

2. Geração via constante hardcoded elimina fragilidade de regex sobre markdown. Dev consciente incrementa quando necessário.

**Teste de disciplina (opcional, recomendado):** criar `tests/prompt-version.test.ts` que computa SHA-256 do bloco entre `Output schema` e `Output example` (linhas delimitadas no próprio SYSTEM_PROMPT) e compara contra snapshot em `tests/__snapshots__/prompt-structure.snap`. Se o SHA divergir sem alguém ter incrementado `STRUCTURE_VERSION`, teste falha. Isso pega "mudança estrutural acidental que esqueceu de incrementar versão". Fica como débito se não for implementado no PR1.

### 7.10 Novo arquivo: `lib/pipeline/persist-precheck.ts`

Função dedicada para persistir o resultado da Camada 0 sem passar pelo pipeline de `persistCuratedJob`.

**Motivação (correção de bug descoberto na revisão v4):** `persistCuratedJob` em `lib/pipeline/persist-curation.ts:40-44` espera um `NormalizedCurationResult` com campos em camelCase (`canonicalRole`, `skillItems`, `seniorityInferred`, `workModel`). Os `skillItems` são do tipo `LLMSkillItem[]` (com `name`, `type`, `confidence`), não `string[]`. O objeto retornado por `precheckDescriptionHash` tem shape incompatível — `skills` é `string[]` porque já está na forma persistida em `job_postings.skills`. Tentar adaptar exigiria remontar `LLMSkillItem[]` a partir de strings e chamar `mapSkillsToCanonical`, o que é desnecessário: as skills já foram mapeadas para canônicos quando a vaga âncora foi originalmente curada. A Camada 0 apenas clona o resultado final.

Solução arquitetural limpa: função nova com escopo mínimo que faz apenas UPDATE direto em `job_postings`, sem chamar `upsertCanonicalRole` (canônico já existe) e sem remapear skills.

**v5.5 (Claude-2 #6) — refactor recomendado pós-implementação:** o guard `curation_status='pending'` aparece em `persistPrecheckResult` (aqui) e em `persistCuratedJob` (§7.11). É a mesma lógica duplicada. Pós-implementação, extrair função utilitária `withPendingGuard(jobId, updatePayload, supabase)` em `lib/pipeline/persist-helpers.ts` que encapsula o UPDATE com WHERE `curation_status='pending'` AND optimistic lock. Ambos os callers passam a chamar essa função. Refactor é de qualidade de código, não bloqueante — pode entrar no mesmo PR ou em PR subsequente de limpeza.

```typescript
import { SupabaseClient } from '@supabase/supabase-js';

export interface PersistPrecheckInput {
    jobId: string;
    canonicalRoleId: string;
    canonicalRoleLabel: string;
    skills: string[];                          // clonado de job_postings.skills
    seniorityInferred: string | null;
    workModel: string | null;
    descriptionCurated: string | null;
    descriptionHash: string;                   // hash gerado na detecção
    promptStructureVersion: string;
    promptContentVersion: string;
    curationSource: 'precheck_description_hash';
    canonicalResolvedAt: string;               // ISO timestamp

    // v5.4 (GS-R6-10): optimistic lock na âncora.
    // Caller passa o id e updated_at da âncora que produziu o hit.
    // persistPrecheckResult re-lê a âncora e aborta se updated_at mudou.
    anchorJobId?: string;
    anchorUpdatedAt?: string;                  // ISO timestamp
}

export interface PersistPrecheckResult {
    success: boolean;
    error?: string;
    concurrent_update_detected?: boolean;      // true quando guard de estado falha
}

/**
 * Persiste o resultado de uma vaga que teve hit na Camada 0.
 *
 * Diferente de persistCuratedJob, não chama upsertCanonicalRole nem
 * mapSkillsToCanonical — o canônico já existe (referenciado pela âncora)
 * e as skills já estão na forma persistida (string[]).
 *
 * Campos mutados em job_postings:
 * - canonical_role_id
 * - canonical_role_label
 * - skills (array de strings)
 * - seniority_inferred
 * - work_model
 * - description_curated
 * - description_hash
 * - prompt_structure_version
 * - prompt_content_version
 * - curation_source
 * - canonical_resolved_at
 * - curation_status ('curated')
 * - curated_at (NOW())
 *
 * Todos os campos são gravados em um único UPDATE atômico.
 *
 * GUARDA DE ESTADO: UPDATE só afeta vagas em `curation_status='pending'`.
 * Previne sobrescrever vaga já curada manualmente, por outro caminho do
 * pipeline, ou por race concorrente. Se UPDATE não afetar 1 linha, retorna
 * erro com flag `concurrent_update_detected=true`.
 *
 * v5.4 (GS-R6-10) — OPTIMISTIC LOCK NA ÂNCORA:
 * Além de validar que a vaga nova está pending, validar que a âncora que
 * produziu o hit ainda tem o mesmo updated_at de quando foi lida. Se admin
 * fez manual_remap na âncora entre o lookup e este UPDATE (cenário raro mas
 * real: endpoint HTTP + CRON worker concorrentes), o trigger N4 pode ter
 * revogado human_validated e a âncora que estamos clonando já não é mais
 * válida. Descartar o hit nesse caso é mais seguro que propagar estado stale.
 */
export async function persistPrecheckResult(
    supabase: SupabaseClient,
    input: PersistPrecheckInput
): Promise<PersistPrecheckResult> {
    // v5.4 (GS-R6-10): re-ler a âncora e validar que o updated_at não mudou
    // desde o lookup original. Se mudou, admin interveio entre leitura e
    // persistência — abortar e deixar a vaga cair em Camada 1/LLM.
    if (input.anchorUpdatedAt) {
        const { data: anchorNow, error: anchorErr } = await supabase
            .from('job_postings')
            .select('updated_at, human_validated')
            .eq('id', input.anchorJobId)
            .single();

        if (anchorErr || !anchorNow) {
            return {
                success: false,
                error: `Anchor ${input.anchorJobId} disappeared between lookup and persist`,
                concurrent_update_detected: true
            };
        }

        if (anchorNow.updated_at !== input.anchorUpdatedAt) {
            // Âncora foi modificada entre lookup e persist. Gravar evento
            // e não persistir — a vaga nova segue para Camada 1/LLM.
            await supabase.from('events').insert({
                event_name: 'precheck_anchor_stale_abort',
                resource_type: 'job_posting',
                resource_id: input.jobId,
                actor: 'pipeline',
                actor_id: '00000000-0000-0000-0000-000000000001',
                reason: `Anchor ${input.anchorJobId} updated_at mudou entre lookup e persist — hit descartado por defesa`,
                metadata: {
                    anchor_id: input.anchorJobId,
                    updated_at_at_lookup: input.anchorUpdatedAt,
                    updated_at_now: anchorNow.updated_at,
                    anchor_human_validated_now: anchorNow.human_validated
                }
            });

            return {
                success: false,
                error: `Anchor changed between lookup and persist for job ${input.jobId}`,
                concurrent_update_detected: true
            };
        }
    }

    const { data, error } = await supabase
        .from('job_postings')
        .update({
            canonical_role_id: input.canonicalRoleId,
            canonical_role_label: input.canonicalRoleLabel,
            skills: input.skills,
            seniority_inferred: input.seniorityInferred,
            work_model: input.workModel,
            description_curated: input.descriptionCurated,
            description_hash: input.descriptionHash,
            prompt_structure_version: input.promptStructureVersion,
            prompt_content_version: input.promptContentVersion,
            curation_source: input.curationSource,
            canonical_resolved_at: input.canonicalResolvedAt,
            curation_status: 'curated',
            curated_at: new Date().toISOString()
        })
        .eq('id', input.jobId)
        .eq('curation_status', 'pending')  // GUARDA — previne sobrescrever vaga já curada
        .select('id');

    if (error) {
        return {
            success: false,
            error: `persistPrecheckResult failed for job ${input.jobId}: ${error.message}`
        };
    }

    // Se 0 linhas afetadas, vaga foi curada por outro caminho entre leitura e write
    if (!data || data.length === 0) {
        // Registrar evento forense mas NÃO falhar o batch
        await supabase.from('events').insert({
            event_name: 'precheck_update_conflict',
            resource_type: 'job_posting',
            resource_id: input.jobId,
            actor: 'pipeline',
            actor_id: '00000000-0000-0000-0000-000000000001',
            reason: 'Job não estava em curation_status=pending ao tentar persistir precheck hit',
            metadata: { curation_source: input.curationSource }
        });

        return {
            success: false,
            error: `Concurrent update detected for job ${input.jobId} — curation_status was not pending`,
            concurrent_update_detected: true
        };
    }

    return { success: true };
}
```

**Observações:**

- **Campos em camelCase na interface do TypeScript, snake_case na persistência** — convenção padrão do Supabase.
- **v5.1 — guarda de estado:** UPDATE só afeta vagas em `curation_status='pending'`. Detecção de concorrência gera evento forense `precheck_update_conflict`.
- **Chamado exclusivamente de `batch-processor.ts`:** função não é API pública, não precisa de validação de input via Zod ou similar. Caller é confiável.

### 7.11 Modificação em `lib/pipeline/persist-curation.ts` — CRÍTICA

**Correção de bloqueante descoberto na terceira rodada de revisão.** A v5.0 especificava `persistPrecheckResult` apenas para a Camada 0, mas **todas as vagas curadas pelo LLM continuam passando por `persistCuratedJob`**, que na v5.0 não foi modificada para gravar os 4 novos campos. Consequência silenciosa:

- Vagas novas curadas pelo LLM ficariam com `description_hash = NULL`
- Pool de âncoras da Camada 0 nunca cresceria além do backfill inicial
- Após TTL de 30 dias, Camada 0 viraria inerte e economia projetada de 68,7% se perderia progressivamente

**v5.2 — B3: contrato agora executável.** A v5.1 usava `expectedPreResolvedLabel` e `preresolvedRoleId` sem defini-los em lugar nenhum. O implementador não conseguia executar sem inventar a origem dessas variáveis. Correção: adicionar `pre_resolved_canonical_label` e `pre_resolved_canonical_role_id` ao `PersistOptions`, populados pelo caller em `batch-processor.ts` a partir de `job.pre_resolved_canonical_label` e `job.pre_resolved_canonical_role_id` (campos de `PreparedJob`).

**Modificação obrigatória em `persistCuratedJob`:**

A função existente recebe `NormalizedCurationResult` + `PersistOptions`. v5.3 estende `PersistOptions` com **8 campos novos opcionais** (6 da v5.1: `description_hash`, `prompt_structure_version`, `prompt_content_version`, `curation_source`, `canonical_resolved_at`, `layer_2_hint_count`; 2 da v5.2: `pre_resolved_canonical_label`, `pre_resolved_canonical_role_id`). Todos retrocompatíveis — callers antigos funcionam sem passar nada:

```typescript
// lib/pipeline/persist-curation.ts (modificação v5.2)

export interface PersistOptions {
    // ... opções existentes ...

    // v5.1 — campos de versão e origem do pipeline
    description_hash?: string | null;          // calculado antes da chamada LLM, passado como opt
    prompt_structure_version?: string;         // PROMPT_STRUCTURE_VERSION em uso
    prompt_content_version?: string;           // PROMPT_CONTENT_VERSION em uso
    curation_source?: 'llm_direct' | 'layer_1_domain_synonyms' | 'llm_equivalences_redirect';
    canonical_resolved_at?: string | null;     // populado se Camada 1 resolveu antes do LLM
    layer_2_hint_count?: number | null;        // observabilidade causal da Camada 2

    // v5.2 — B3: override server-side explícito (pré-resolução da Camada 1)
    // Quando o caller sabe qual canônico foi pré-resolvido, passa label + id aqui.
    // persistCuratedJob usa ambos como verdade — LLM é contexto, servidor é autoritativo.
    //
    // v5.5 (Claude-2 polimento) — convenção de nomenclatura:
    // - `pre_resolved_canonical_label` é o nome usado no CÓDIGO TypeScript
    //   (campo de PersistOptions, variáveis locais, logs, eventos).
    // - `canonical_already_resolved` é o nome usado no PROMPT XML enviado
    //   ao Sonnet (Etapa 0, atributo da tag <job>).
    // Os dois referem-se à mesma coisa mas ficam em domínios separados:
    // TS usa snake_case descritivo, XML usa nome semanticamente intuitivo
    // para o LLM. Não unificar — dessincronizar os domínios tornaria código
    // ou prompt menos claros.
    pre_resolved_canonical_label?: string | null;
    pre_resolved_canonical_role_id?: string | null;

    // v5.4: popula colunas original_* em vagas novas (vagas legadas cobertas
    // pelo backfill inline da migration §5.1). Propagado pelo caller em
    // batch-processor.ts a partir de PreparedJob.title (bruto, pré-LLM).
    // Sem isso, filtro N7 da Camada 0 (original_title NOT NULL) exclui
    // vagas novas do pool de âncoras e o pool não cresce.
    originalTitle?: string | null;
    originalDescription?: string | null;
}

export async function persistCuratedJob(
    supabase: SupabaseClient,
    normalized: NormalizedCurationResult,
    options: PersistOptions = {},
    counters?: RunCounters  // v5.2 — N8: aceita counters para incrementar llmDisobeyedPreResolution
): Promise<PersistResult> {
    // ... código existente de mapSkillsToCanonical e upsertCanonicalRole ...

    // v5.2 — B3: sobrescrita server-side com variáveis vindas de PersistOptions
    // Se pre_resolved_canonical_label está populado, significa Camada 1 já
    // determinou o canônico. Confiar no servidor, não no LLM (LLM pode
    // desobedecer o atributo XML canonical_already_resolved).
    let finalCanonicalRoleId = mappedRoleId;
    let finalCanonicalRoleLabel = mappedLabel;

    const hasPreResolution =
        options.curation_source === 'layer_1_domain_synonyms' &&
        options.pre_resolved_canonical_label &&
        options.pre_resolved_canonical_role_id;

    if (hasPreResolution) {
        const expectedLabel = options.pre_resolved_canonical_label!;
        const expectedRoleId = options.pre_resolved_canonical_role_id!;

        // Checa se LLM respeitou a pré-resolução
        if (normalized.canonicalRole !== expectedLabel) {
            // v5.2: LLM desobedeceu — gravar evento forense e contador
            // v5.3 (G16): metadata expandido com pre_resolved_canonical_role_id
            // para trilha forense completa (permite rastrear qual canônico
            // específico foi sobrescrito sem precisar cruzar com outra tabela)
            await supabase.from('events').insert({
                event_name: 'llm_disobeyed_pre_resolution',
                resource_type: 'job_posting',
                resource_id: normalized.jobId,
                actor: 'pipeline',
                actor_id: SYSTEM_USER_ID,
                reason: `LLM retornou '${normalized.canonicalRole}' mas Camada 1 havia pré-resolvido para '${expectedLabel}'`,
                metadata: {
                    llm_canonical: normalized.canonicalRole,
                    preresolved_canonical: expectedLabel,
                    pre_resolved_canonical_role_id: expectedRoleId,  // v5.3 (G16)
                    job_id: normalized.jobId
                }
            });

            // v5.2 — N8: incrementar contador se disponível
            if (counters) {
                counters.llmDisobeyedPreResolution++;
            }
        }

        // Usa o valor pré-resolvido como verdade (não o LLM)
        finalCanonicalRoleId = expectedRoleId;
        finalCanonicalRoleLabel = expectedLabel;
    }

    // UPDATE final com 4 novos campos
    const { error } = await supabase
        .from('job_postings')
        .update({
            // ... campos existentes (skills, seniority, work_model, etc.) ...
            canonical_role_id: finalCanonicalRoleId,
            canonical_role_label: finalCanonicalRoleLabel,

            // v5.1 — novos campos
            description_hash: options.description_hash ?? null,
            prompt_structure_version: options.prompt_structure_version ?? null,
            prompt_content_version: options.prompt_content_version ?? null,
            curation_source: options.curation_source ?? 'llm_direct',
            canonical_resolved_at: options.canonical_resolved_at ?? null,
            layer_2_hint_count: options.layer_2_hint_count ?? null,

            curation_status: 'curated',
            curated_at: new Date().toISOString()
        })
        .eq('id', normalized.jobId)
        .eq('curation_status', 'pending')  // v5.1 guard de estado
        .select('id');

    // ... resto do código existente ...
}
```

**Caller em `batch-processor.ts` precisa passar os options:**

```typescript
// Após parsing do response do LLM:
for (const job of jobsForLLM) {
    const llmResult = parseSonnetResponse(response, job.id);
    const normalized = normalizeCurationResult(llmResult);

    // v5.1 — determinar curation_source baseado em como a vaga chegou ao LLM
    const curationSource: PersistOptions['curation_source'] =
        job.curation_source === 'layer_1_domain_synonyms'
            ? 'layer_1_domain_synonyms'
            : 'llm_direct';

    await persistCuratedJob(ctx.supabase, normalized, {
        description_hash: job.generated_description_hash,
        prompt_structure_version: PROMPT_STRUCTURE_VERSION,
        prompt_content_version: PROMPT_CONTENT_VERSION,
        curation_source: curationSource,
        canonical_resolved_at: job.canonical_already_resolved
            ? new Date().toISOString()
            : null,
        layer_2_hint_count: job.suggested_roles?.length ?? null,
        // v5.2 — B3: campos explícitos para override server-side
        pre_resolved_canonical_label: job.pre_resolved_canonical_label ?? null,
        pre_resolved_canonical_role_id: job.pre_resolved_canonical_role_id ?? null,
        // v5.4: propaga título e descrição brutos do PreparedJob para persistência
        // nas colunas dedicadas. PreparedJob.title vem do scraper via insert-jobs.ts.
        // PreparedJob.rawDescription (ver tipo em §7.7.1) vem de
        // requirements.description quando ingestão ainda não foi migrada, ou
        // direto de original_description quando ingestão já foi migrada (§5.1.1).
        originalTitle: job.title ?? null,
        originalDescription: job.rawDescription ?? null
    }, ctx.counters);  // v5.2 — passa counters para incrementar llmDisobeyedPreResolution
}
```

**Impacto:** essa modificação é **a mais crítica da v5.1/v5.2**. Sem ela, a arquitetura não fecha operacionalmente — Camada 0 morre em 30 dias e override server-side não funciona. Executor deve tratar como item de mais alta prioridade no PR2.

### 7.11.1 Comportamento `quarantined_llm_output` no pipeline novo

**v5.4 (GS-R6-1) — bug da sprint anterior formalizado na spec.**

Contexto: em 23/04/2026, sprint anterior identificou bug crítico em `persist-curation.ts:126` onde o path `curated_fallback` gravava vagas com `curation_status='curated'` mesmo quando o canônico retornado pelo LLM era blacklisted ou não-resolvível via `upsertCanonicalRole`. Commit `2c26ce6` corrigiu via novo status `quarantined_llm_output` — vagas nesse path ficam invisíveis para o pipeline de matching até que admin as revise manualmente.

**Problema na v5.3:** a spec define o happy path (Camada 0 hit, Camada 1 override, LLM normal) mas não define o que acontece quando `upsertCanonicalRole` retorna `null` **dentro do pipeline novo** com as 8 colunas novas em jogo. Risco: vagas quarentenadas gravadas sem `description_hash`/`prompt_structure_version`/`curation_source` populados contaminam o pool de âncoras legadas ou ficam órfãs do schema novo.

**Regra normativa v5.4:** o path `quarantined_llm_output` em `persist-curation.ts` deve gravar **todos os 8 campos novos** do `PersistOptions` mesmo não tendo canônico resolvido:

```typescript
// Dentro de persistCuratedJob, quando upsertCanonicalRole retorna null:
if (!canonicalRole) {
    // v5.4 (GS-R6-1) — quarentena com os 8 campos novos populados.
    // Mesmo sem canônico resolvido, a vaga teve hash calculado, passou por
    // determinada versão do prompt, e tem origem rastreável — tudo isso
    // deve ficar no registro para auditoria forense e para que o backfill
    // não tente re-processar essa vaga no futuro.

    const { error } = await supabase
        .from('job_postings')
        .update({
            curation_status: 'quarantined_llm_output',
            canonical_role_id: null,
            canonical_role_label: null,
            canonical_resolved_at: null,           // null explícito — vaga não foi resolvida
            curation_source: 'llm_direct',         // v5.2 — passou pelo LLM, apenas não resolveu canônico
            curated_at: new Date().toISOString(),  // timestamp da tentativa
            skills: normalized.skillItems ?? [],
            seniority_inferred: normalized.seniorityInferred ?? null,
            work_model: normalized.workModel ?? null,
            description_curated: normalized.descriptionCurated ?? null,
            // v5.4 — sempre grava os 4 campos de versão do pipeline novo
            // v5.5 (Grok #2): CORREÇÃO de naming — PersistOptions usa snake_case,
            // não camelCase. A v5.4 tinha bug funcional que gravava null em
            // description_hash de toda vaga quarentenada, contradizendo exatamente
            // a motivação declarada em §7.11.1.
            description_hash: options.description_hash ?? null,
            prompt_structure_version: options.prompt_structure_version ?? PROMPT_STRUCTURE_VERSION,
            prompt_content_version: options.prompt_content_version ?? PROMPT_CONTENT_VERSION,
            layer_2_hint_count: options.layer_2_hint_count ?? null,
            // v5.4 — popula original_title/original_description se disponíveis
            // (originalTitle/originalDescription são camelCase propositalmente
            //  porque foram adicionados v5.4 com essa convenção; manter consistência
            //  com o tipo declarado em §7.11):
            original_title: options.originalTitle ?? null,
            original_description: options.originalDescription ?? null,
            // Fontes de pré-resolução ficam null — LLM deu o resultado quarentenado:
            // (pre_resolved_canonical_label/role_id não aplicáveis)
        })
        .eq('id', normalized.jobId);

    if (error) throw error;

    // Evento forense específico para quarentena no pipeline novo
    await supabase.from('events').insert({
        event_name: 'llm_output_quarantined',
        resource_type: 'job_posting',
        resource_id: normalized.jobId,
        actor: 'pipeline',
        actor_id: SYSTEM_USER_ID,
        reason: `LLM retornou canonical '${normalized.canonicalRole}' que não resolveu em upsertCanonicalRole (blacklist/redirect/missing)`,
        metadata: {
            llm_canonical_raw: normalized.canonicalRole,
            description_hash: options.description_hash,  // v5.5 (Grok #2)
            prompt_structure_version: options.prompt_structure_version  // v5.5 (Grok #2)
        }
    });

    if (counters) {
        counters.llmOutputQuarantined++;
    }

    return;  // Não continua para o path de curated_fallback nem para o override server-side
}
```

**Por que gravar `description_hash` e `prompt_structure_version` mesmo em quarentena:**
- **Idempotência do backfill:** sem esses campos, o backfill pode tentar re-processar a vaga posteriormente. Com eles populados, o filtro `description_hash IS NULL` no backfill pula a vaga corretamente.
- **Rastreabilidade de prompt:** admin que revisar a vaga no futuro pode ver qual versão do prompt a classificou como quarentenada. Se o prompt mudou entre versões, vale re-rodar a vaga manualmente.
- **Não contaminar pool de âncoras:** Camada 0 só considera âncoras com `curation_status='curated'` (filtro em §7.3). Vagas `quarantined_llm_output` nunca viram âncoras, independente dos campos populados.

**Contador novo em `RunCounters`:**

```typescript
interface RunCounters {
    // ... campos existentes ...
    llmOutputQuarantined: number;  // v5.4 (GS-R6-1)
}
```

**Inicialização em `createRunCounters()`:** `llmOutputQuarantined: 0`.

**Critério de aceite §9.2.5 (novo):** teste unitário valida que vaga com canonical retornado pelo LLM que está em blacklist resulta em:
- `curation_status = 'quarantined_llm_output'`
- `description_hash` populado (do cálculo prévio)
- `prompt_structure_version` populado
- Evento `llm_output_quarantined` gravado com metadata completo
- `canonical_role_id` e `canonical_resolved_at` ambos NULL

### 7.12 Novo arquivo: `lib/pipeline/rollout-sampling.ts`

Implementa rollout percentual real. v5.0 usava flag booleana que não permitia amostragem.

```typescript
import crypto from 'crypto';

/**
 * Retorna percentual estável (0-99) baseado no jobId.
 * Mesmo jobId sempre retorna mesmo valor → rollout determinístico.
 *
 * v5.1: usava últimos 2 hex (1 byte = 256 valores). `256 % 100` tem viés
 * nos primeiros 56 valores (aparecem 3×, os 44 restantes aparecem 2×).
 * Para `% 100`, isso significa ~17% de viés no pior caso.
 *
 * v5.2 (N10): usa 4 hex chars (2 bytes = 65536 valores). `65536 % 100`
 * tem viés de apenas ~0.05% — negligível para qualquer percentual de
 * rollout, inclusive valores não-padrão (ex: 37%, 83%).
 *
 * Exemplo: PIPELINE_V3_ROLLOUT_PERCENT=37 roteia 37,00% ± 0.05% do
 * tráfego para o pipeline novo, de forma determinística por job.id.
 */
export function stablePercent(jobId: string): number {
    const hash = crypto.createHash('sha256').update(jobId).digest('hex');
    // v5.2 — 4 hex chars em vez de 2 (65536 valores em vez de 256)
    const bucket = parseInt(hash.slice(-4), 16);
    return bucket % 100;
}

/**
 * Checa se a vaga está no rollout baseado em PIPELINE_V3_ROLLOUT_PERCENT.
 * Valores válidos: 0 (nenhuma vaga), 10, 25, 50, 75, 100 (todas).
 * Qualquer valor é aceito; a granularidade é de 1%.
 */
export function isJobInRollout(jobId: string): boolean {
    const rolloutPercent = Number(process.env.PIPELINE_V3_ROLLOUT_PERCENT ?? '0');
    if (rolloutPercent <= 0) return false;
    if (rolloutPercent >= 100) return true;
    return stablePercent(jobId) < rolloutPercent;
}
```

**Uso em `batch-processor.ts`:**

```typescript
import { isJobInRollout } from '@/lib/pipeline/rollout-sampling';

// Substituir o check anterior de PIPELINE_V3_ENABLED===true por:
for (const job of jobs) {
    if (!isJobInRollout(job.id)) {
        // Vaga fora do rollout → pipeline legado (sem Camadas 0/1/2)
        continue;
    }
    // ... código das camadas ...
}
```

**Teste de distribuição (v5.2):**

```typescript
// tests/rollout-sampling.test.ts
import { describe, it, expect } from 'vitest';
import { stablePercent } from '@/lib/pipeline/rollout-sampling';
import crypto from 'crypto';

describe('stablePercent (v5.2 — 4 hex chars)', () => {
    it('é determinístico', () => {
        const id = crypto.randomUUID();
        expect(stablePercent(id)).toBe(stablePercent(id));
    });

    it('distribui uniformemente em 10000 IDs', () => {
        const counts = Array(100).fill(0);
        for (let i = 0; i < 10000; i++) {
            const id = crypto.randomUUID();
            counts[stablePercent(id)]++;
        }
        // Cada bucket tem média esperada de 100; desvio padrão << 30
        const mean = counts.reduce((a, b) => a + b, 0) / counts.length;
        expect(mean).toBeCloseTo(100, 0);

        const maxDeviation = Math.max(...counts.map(c => Math.abs(c - 100)));
        expect(maxDeviation).toBeLessThan(50);  // muito conservador; real << 30
    });

    it('não concentra em decil inicial (validação anti-viés)', () => {
        const counts = Array(10).fill(0);
        for (let i = 0; i < 10000; i++) {
            const id = crypto.randomUUID();
            counts[Math.floor(stablePercent(id) / 10)]++;
        }
        // Cada decil tem média esperada de 1000; desvio < 15% por decil
        for (const c of counts) {
            expect(c).toBeGreaterThan(850);
            expect(c).toBeLessThan(1150);
        }
    });

    // v5.4 (GS-R6-9): em produção, batches de 38 vagas podem ter IDs próximos
    // por timestamp (CRON processa em janelas). Se SHA-256 tiver correlação
    // fraca entre UUIDs consecutivos, um batch inteiro pode ir 100% para
    // rollout ou 0% para rollout, enviesando métricas do estágio 10%.
    // Este teste simula 100 batches de 38 UUIDs cada e valida que a
    // distribuição dentro de cada batch fica dentro de ±15% do
    // PIPELINE_V3_ROLLOUT_PERCENT esperado.
    it('distribui corretamente em batches consecutivos de 38 UUIDs', () => {
        const ROLLOUT = 25;  // teste com 25%
        const BATCH_SIZE = 38;
        const NUM_BATCHES = 100;
        const batchInclusionRates: number[] = [];

        for (let b = 0; b < NUM_BATCHES; b++) {
            const ids = Array.from({ length: BATCH_SIZE }, () => crypto.randomUUID());
            const includedInRollout = ids.filter(id => stablePercent(id) < ROLLOUT).length;
            batchInclusionRates.push(includedInRollout / BATCH_SIZE);
        }

        const mean = batchInclusionRates.reduce((a, b) => a + b, 0) / NUM_BATCHES;
        expect(mean).toBeCloseTo(ROLLOUT / 100, 1);

        // Nenhum batch deve ter >60% ou <5% (expectativa em 25%)
        // Tolerância ampla porque batches pequenos têm variância natural alta
        const outliers = batchInclusionRates.filter(r => r > 0.6 || r < 0.05);
        expect(outliers.length).toBeLessThan(3);  // <3% dos batches podem ser outliers extremos
    });
});
```

**Env vars finais:**

- `PIPELINE_V3_ROLLOUT_PERCENT` (0-100, default 0): percentual de tráfego que entra no novo pipeline.
- `PIPELINE_V3_PRECHECK_ONLY` (boolean, default `false`): em vagas dentro do rollout, se `true`, apenas Camada 0 executa; misses pulam LLM (modo debug).

### 7.13 Novo endpoint admin: `app/api/admin/jobs/[id]/human-validated/route.ts`

Endpoint para setar e revogar `human_validated` com trilha de auditoria.

**v5.6 (Claude Code — 5 correções críticas):** auditoria do Claude Code encontrou 5 bugs na v5.5 desta seção:
1. `requireAdmin` de `@/lib/auth/require-admin` — função e arquivo inexistentes (nome diferente de `requireAdminAuth` usado em §5.6.2 — inconsistência interna da própria spec)
2. `const supabase = createClient();` sem `await` — `createClient` em `@/lib/supabase/server` é **async**, retorna Promise. `supabase.from()` quebra com "is not a function"
3. Cookie client (`NEXT_PUBLIC_SUPABASE_ANON_KEY`) faz UPDATE em `job_postings` — RLS está habilitada, request é **silenciosamente rejeitado** (retorna `{data: [], error: null}`). Bug invisível: sucesso aparente + nada persistido
4. `actor='admin'` viola CHECK constraint `events.actor` (só aceita `{system, pipeline, human}`)
5. Referência "v5.1 backlog" sobre revogação automática contradiz §5.6.3 (trigger N4 já é obrigatório, não backlog)

Reescrita completa seguindo padrão do projeto (idêntico ao §5.6.2):

```typescript
// app/api/admin/jobs/[id]/human-validated/route.ts
// v5.6 — padrão alinhado com 28+ endpoints admin existentes

import { createAdminServerClient } from '@/lib/supabase-server';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(
    req: NextRequest,
    { params }: { params: { id: string } }
) {
    // ── Auth inline (padrão do projeto, idêntico ao §5.6.2) ──
    const cookieSupabase = await import('@/lib/supabase/server').then(m => m.createClient());
    const { data: authData } = await (await cookieSupabase).auth.getUser();
    if (!authData?.user) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });

    const supabaseAdmin = createAdminServerClient();
    const { data: userData } = await supabaseAdmin
        .from('users')
        .select('user_type')
        .eq('id', authData.user.id)
        .single();
    if (userData?.user_type !== 'admin') {
        return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
    }

    // ── Validação do body ────────────────────────────────
    const body = await req.json();
    const { validated, reason } = body as { validated: boolean; reason?: string };

    if (typeof validated !== 'boolean') {
        return NextResponse.json({ error: 'validated must be boolean' }, { status: 400 });
    }

    // ── Ler estado atual (admin client bypassa RLS) ──────
    // v5.6 (Claude Code): usar supabaseAdmin em vez de cookie client.
    // job_postings tem RLS habilitada — cookie client resultaria em no-op silencioso.
    const { data: current } = await supabaseAdmin
        .from('job_postings')
        .select('human_validated, canonical_role_id, canonical_role_label')
        .eq('id', params.id)
        .single();

    if (!current) {
        return NextResponse.json({ error: 'Job not found' }, { status: 404 });
    }

    // No-op se estado já bate
    if (current.human_validated === validated) {
        return NextResponse.json({ success: true, noop: true });
    }

    // ── UPDATE (admin client) ────────────────────────────
    const { error: updError } = await supabaseAdmin
        .from('job_postings')
        .update({ human_validated: validated })
        .eq('id', params.id);

    if (updError) {
        return NextResponse.json({ error: updError.message }, { status: 500 });
    }

    // ── Evento forense (actor='human', não 'admin') ──────
    // v5.6 (ChatGPT C4): CHECK constraint events.actor só aceita
    // {system, pipeline, human}. Admin real é actor='human' + user_id real.
    await supabaseAdmin.from('events').insert({
        event_name: validated ? 'human_validated_set' : 'human_validated_revoked',
        resource_type: 'job_posting',
        resource_id: params.id,
        actor: 'human',                   // ← NÃO 'admin' (viola CHECK)
        actor_id: authData.user.id,       // ← user real logado
        previous_state: {
            human_validated: current.human_validated,
            canonical_role_id: current.canonical_role_id,
            canonical_role_label: current.canonical_role_label
        },
        new_state: { human_validated: validated },
        reason: reason ?? null
    });

    return NextResponse.json({ success: true });
}
```

**v5.6 (Claude Code — item 40c):** a seção "Políticas de revogação automática (opcional, v5.1 backlog)" da v5.5 descrevia a revogação automática como backlog, mas a §5.6.3 da mesma v5.5 já implementa isso como trigger N4 obrigatório. Contradição interna removida na v5.6 — a trigger é feature oficial, não backlog.

**Populamento inicial (executar no PR2 após §2.4):**

O Onsly marca manualmente via SQL as vagas que foram validadas em sessões anteriores — tipicamente os 3 grupos conhecidos (Canonical GSI, Stone, AgileEngine):

```sql
-- Exemplo: marcar Canonical GSI como human_validated
UPDATE job_postings
SET human_validated = true
WHERE canonical_role_id = 'ed2cd343-925e-442d-9f26-c5a462a37e81'
  AND linkedin_id IN (...lista de IDs conhecidos...);
```

### 7.14 Trigger PostgreSQL: sincronia de `canonical_role_label` (v5.2 — N3)

**Motivação:** `canonical_role_label` em `job_postings` é denormalizado de `job_canonical_roles.label`. Se um admin renomear um canônico via `UPDATE job_canonical_roles SET label = 'Novo Label' WHERE id = ...`, o label antigo sobreviveria em todas as vagas apontando para aquele canônico. Trigger PostgreSQL propaga a mudança automaticamente.

**Arquivo:** `docs/migrations/20260423_04_canonical_role_label_sync_trigger.sql`

```sql
-- ============================================================================
-- v5.2 — N3: sincronia automática de canonical_role_label
--
-- Quando o label de um canônico em job_canonical_roles muda, todas as vagas
-- (job_postings) que apontam para esse canônico têm seu canonical_role_label
-- atualizado automaticamente. Elimina drift silencioso entre o catálogo
-- canônico e a denormalização em vagas.
-- ============================================================================

CREATE OR REPLACE FUNCTION sync_canonical_role_label_on_update()
RETURNS TRIGGER AS $$
BEGIN
    -- Só atualiza se label realmente mudou (evita trigger ciclo em outras mudanças)
    IF NEW.label IS DISTINCT FROM OLD.label THEN
        UPDATE job_postings
        SET canonical_role_label = NEW.label
        WHERE canonical_role_id = NEW.id
          AND canonical_role_label IS DISTINCT FROM NEW.label;

        -- Evento de auditoria
        INSERT INTO events (
            event_name, resource_type, resource_id,
            actor, actor_id, reason, metadata
        ) VALUES (
            'canonical_role_label_synced',
            'job_canonical_role',
            NEW.id,
            'system',
            '00000000-0000-0000-0000-000000000001',
            format('Label renomeado de %L para %L — sincronização em vagas propagada', OLD.label, NEW.label),
            jsonb_build_object(
                'old_label', OLD.label,
                'new_label', NEW.label
            )
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trigger_sync_canonical_role_label_after_update
ON job_canonical_roles;

CREATE TRIGGER trigger_sync_canonical_role_label_after_update
AFTER UPDATE ON job_canonical_roles
FOR EACH ROW
EXECUTE FUNCTION sync_canonical_role_label_on_update();

COMMENT ON TRIGGER trigger_sync_canonical_role_label_after_update
ON job_canonical_roles IS
    'v5.2 — N3: propaga rename do label para canonical_role_label em job_postings.';
```

**Notas operacionais:**

- **Escopo:** apenas UPDATE de `label`. INSERT e DELETE não precisam de sincronia (INSERT não tem vagas ainda; DELETE é bloqueado se houver FK ativa).
- **Idempotência:** o UPDATE interno tem `IS DISTINCT FROM NEW.label` para evitar escrita desnecessária em vagas já sincronizadas.
- **Performance:** em admin UI, um rename atinge ≤ poucas centenas de vagas em casos típicos. Quando um canônico tem milhares de vagas (ex: "Analista de Vendas"), o UPDATE é batch single-statement — barato.
- **Evento único por rename:** um evento `canonical_role_label_synced` por operação de rename (não um por vaga atualizada).

### 7.15 Trigger PostgreSQL: revogação de `human_validated` após `manual_remap` — movido para §5.6.3 (v5.5)

**v5.5 (Decisão 4 — greenfield):** esta seção originalmente descrevia apenas o trigger N4, mas não os pré-requisitos (coluna `curation_source`, endpoint admin de remap). Claude Code confirmou em 24/04/2026 que nenhum dos 3 componentes existia no codebase — era greenfield completo, não integração.

A v5.5 consolidou as 3 partes (coluna + endpoint + trigger) em **§5.6** para que a implementação seja tratada como unidade coesa, não como 3 itens espalhados. A implementação do trigger está em §5.6.3, do endpoint em §5.6.2, e a coluna em §5.6.1 (que reusa o CHECK constraint já presente na §5.1).

**Referência:** ver §5.6 para implementação completa. Esta seção §7.15 é mantida na numeração para preservar referências cruzadas de versões anteriores da spec, mas o conteúdo foi migrado.

---

## 8. Plano de rollout

### 8.1 Ordem de execução — crítica

**v5.6 (ChatGPT I1) — Esclarecimento sobre "Fase 1" vs "Fase 2":** as nomenclaturas Fase 1 e Fase 2 abaixo referem-se à **organização lógica dos artefatos** (PR1 = conteúdo, PR2 = infraestrutura), não a fases temporais separadas no deploy. **Os dois PRs saem em deploy unificado** (decisão Onsly v5.5), conforme detalhado abaixo. O texto está dividido por PR para clareza de revisão e responsabilidade de código, mas Claude Code faz merge dos dois PRs em sequência imediata, sem janela de espera entre eles.

**Fase 0 — Pré-requisitos (§2):**
- Todos os itens da §2.1–§2.5 marcados e confirmados
- Resposta do Onsly sobre §2.4 recebida

**Fase 1 — PR1 (Conteúdo e prompt) — preparação e merge:**

1. Medir overhead de tokens com `tiktoken` (referência: §6.4 overhead esperado 1.400-1.800; se ultrapassar, executar mitigação documentada)
2. Aplicar patch em `domain_synonyms.json` (v2.0 → v3.0 com 68 entradas; total final: 181)
3. Aplicar mudanças 1-4 no `SYSTEM_PROMPT.ts` (Etapa 0, Etapa 2, Regras A-E, micro-regra Support)
4. Renumerar todas as referências internas do prompt para nova ordem de Etapas
5. Atualizar `scripts/generate-prompt-version.ts` para duas dimensões de hash (§7.9)
6. Atualizar `buildUserPrompt` (§7.8) — mesmo que no PR1 ainda não passe novos atributos, muda contrato
7. Rodar prebuild
8. Confirmar novos hashes em `prompt-version.generated.ts`
9. Merge do PR1 (não há deploy independente — segue para Fase 2 imediatamente)
10. Validação imediata em staging: disparar curadoria manual de 50 vagas pending aleatórias via admin
11. Executar validação descrita em §9

**Fase 2 — PR2 (Infraestrutura pre-LLM) — preparação, merge e deploy unificado:**

12. **Passo 12 (migração SQL):** aplicar migração §5.1 (colunas em `job_postings`) + §5.2 (índices) + §5.4 (tabela `allowed_for_pre_resolution`). Popular a whitelist com as decisões do Onsly obtidas em §2.4. Não aplicar código ainda.
13. **Passo 13 (deploy do código):**
    - Estender `lib/pipeline/types.ts` (`PreparedJob` com 8 campos novos opcionais — 6 da v5.1 + 2 da v5.2 para override server-side; `RunCounters` com 6 contadores novos + `preCheckMissByReason` + `llmDisobeyedPreResolution`)
    - Criar arquivos novos: `precheck-description-hash.ts`, `domain-synonyms-lookup.ts`, `suggested-roles-builder.ts`, `persist-precheck.ts`, `rollout-sampling.ts`, `app/api/admin/jobs/[id]/human-validated/route.ts`
    - Modificar `persist-curation.ts` (§7.11) para aceitar os 8 novos `PersistOptions` e sobrescrever canônico server-side quando pré-resolvido
    - Atualizar `taxonomy-cache.ts` (§7.2) com preload de `allowedForPreResolution` e `vacancyCountByLabel`
    - Extrair `normalizeResumeText` para `lib/minhash.ts`; adicionar `normalizeDescriptionForHash` com HTML strip
    - Adicionar `normalizeTitle` em `text-processing.ts` (§7.4.0) com teste unitário
    - Modificar `batch-processor.ts` com snippet v5.1 (§7.7.2): rollout via `isJobInRollout`, `precheckOnly` real, passagem de `candidateTitle` e `allowedForPreResolution` para Camada 0
    - Atualizar `buildUserPrompt` (§7.8) com escape XML completo e CDATA no corpo
    - Atualizar `scripts/generate-prompt-version.ts` (§7.9) com `STRUCTURE_VERSION` hardcoded
    - **Adicionar env vars:** `PIPELINE_V3_ROLLOUT_PERCENT=0` e `PIPELINE_V3_PRECHECK_ONLY=false`
    - Deploy com rollout em 0% (equivalente a OFF)
14. **Passo 14 (backfill):** executar `npx tsx scripts/backfill-job-description-hash.ts`. Tempo: 40-60s. Verifica que vagas elegíveis (≥80 palavras) têm `description_hash` populado; demais têm NULL; **todas têm `prompt_structure_version='legacy'`** (marker v5.1) e `prompt_content_version` populado.
15. **Passo 15 (rollout gradual):** subir `PIPELINE_V3_ROLLOUT_PERCENT` gradualmente: `10` → `25` → `50` → `75` → `100`, com 48h entre etapas (exceto o passo 100, que fica 7 dias antes de remover a flag). Amostragem é determinística por hash do `job.id`.
16. **Passo 16 (monitoramento em cada etapa):** ver §8.5 metas operacionais. Gatilhos de rollback definidos lá.
17. **Validação crítica pós-10%:** rodar a query da §8.5.2 para garantir que as Camadas 1 e 2 estão efetivamente alimentando o LLM. Se `curation_source='layer_1_domain_synonyms'` retornar zero após 48h, suspender rollout e investigar integração Camada 1 ↔ `buildUserPrompt`.
18. Remover flag de rollout (setar `PIPELINE_V3_ROLLOUT_PERCENT=100` permanente e remover do código) após 7 dias estáveis em 100%.

**Por que essa ordem é crítica:** se backfill rodar depois de rollout, primeiras horas de 10% tráfego têm `prompt_structure_version=NULL` em todas as vagas curadas históricas, e a query do `precheckDescriptionHash` nunca retorna hits porque `.eq('prompt_structure_version', currentHash)` sempre falha. Resultado: Camada 0 com 0% hit-rate silencioso, débito escondido até alguém investigar os contadores.

### 8.2 Rollback

**Rollback do PR1:**
- Reverter commit. Hashes voltam ao valor anterior.
- Cache ephemeral Anthropic invalida naturalmente.
- Nenhuma mudança em banco de dados.

**Rollback do PR2:**
- Setar `PIPELINE_V3_ROLLOUT_PERCENT=0`. Pipeline reverte instantaneamente para comportamento pré-v5.1.
- Migrations SQL não são revertidas (seguras — colunas vazias não afetam pipeline antigo).
- Vagas já curadas via Camada 0/1 podem precisar de remediação — ver §8.6.

### 8.3 Feature flags

- `PIPELINE_V3_ROLLOUT_PERCENT` (integer 0-100, default `0`): percentual de vagas que entram no novo pipeline. Amostragem determinística por hash do `job.id`.
- `PIPELINE_V3_PRECHECK_ONLY` (boolean, default `false`): se `true`, vagas dentro do rollout só passam pela Camada 0. Misses pulam LLM (modo debug). Vagas ficam em `curation_status='pending'`.

**Nota:** a v5.0 tinha `PIPELINE_V3_ENABLED` boolean. v5.1 substituiu por `PIPELINE_V3_ROLLOUT_PERCENT` para viabilizar rollout gradual real. Migração de env vars:
- `PIPELINE_V3_ENABLED=false` → `PIPELINE_V3_ROLLOUT_PERCENT=0`
- `PIPELINE_V3_ENABLED=true` → `PIPELINE_V3_ROLLOUT_PERCENT=100`

### 8.4 Observabilidade pós-deploy — detecção de regressão silenciosa

**Motivação:** SHA-256 captura duplicatas exatas. Se empresas começarem a variar descrição minimamente (mudar nome da cidade, trocar 1 palavra, reformatar), o mesmo grupo divergente pode reaparecer escapando da Camada 0.

**Script de monitoramento:** após 2 semanas em 100% de rollout, executar job semanal que:

1. Seleciona amostra aleatória de 500 vagas novas curadas na última semana
2. Gera MinHash (**256 permutações** — v5.1: erro ~6% com threshold 0.94) via `lib/minhash.ts`
3. Computa Jaccard par-a-par em memória (125.250 pares)
4. Conta quantos pares têm Jaccard ≥ 0.94 mas SHA-256 diferente
5. **v5.5 (GenSpark #3) — filtro anti-falso-positivo de recrutamento massivo regional:** descarta pares onde `company_name` é idêntica entre as duas vagas mas divergem em `city` ou `state`. Esses pares representam empresas que publicam a mesma vaga em diversas praças (Nubank "Analista Financeiro Sênior" em SP, RJ, BH, etc.) — texto é quase idêntico mas a divergência de canônico não é bug semântico. Filtro aplica `WHERE NOT (company_a = company_b AND (city_a != city_b OR state_a != state_b))` na contagem do numerador.
6. Entre os pares restantes após filtro (5), conta quantos têm `canonical_role_id` divergente

**Critério de alerta:** se `divergentes_em_jaccard_alto_filtrados / total_pares_jaccard_alto_filtrados > 5%` por 2 semanas consecutivas, reabrir discussão de adicionar MinHash como Camada 0.5 (entre 0 e 1). Item fica como backlog observável, não como débito cego.

Implementação em `scripts/monitor-precheck-regression.ts` — criar junto com o PR2 mas agendar execução apenas após Passo 18. Cron: `0 2 * * 0` (domingo 02:00 UTC).

### 8.5 Metas operacionais de sucesso e gatilhos de rollback (v5.2)

Esta subseção define **o que significa sucesso em produção** pós-rollout. Sem faixas alvo explícitas, a equipe monitora tudo e ainda diverge sobre se o rollout foi bem. Com faixas, qualquer SRE/dev vê o dashboard e decide.

#### 8.5.1 Faixas alvo por estágio de rollout

**v5.2 (B5) — importante sobre o estágio 10%:** o pool de âncoras da Camada 0 é construído durante o rollout a partir das vagas `human_validated=true` backfilled com `prompt_structure_version=PROMPT_STRUCTURE_VERSION` (~100 vagas) + vagas novas curadas pelo LLM (que crescem orgânicamente). Nas primeiras 48h do estágio 10%, apenas ~100 âncoras existem contra um fluxo esperado de ~500-800 curagens novas/dia. Hit rate inicial é matematicamente baixo (~1-2%) e não representa degradação. Por isso **a Camada 0 não tem gatilho de rollback no estágio 10%** — os gatilhos só ativam a partir do estágio 25% (quando o pool já acumulou 1-2 semanas de âncoras).

| Métrica | Fonte | 10% | 25% | 50% | 100% | Gatilho rollback |
|---|---|---|---|---|---|---|
| Hit rate da Camada 0 | `preCheckHit / (preCheckHit + preCheckMiss)` | observacional | ≥ 3% | ≥ 6% | ≥ 7% | < 3% por 48h **a partir do estágio 25%** |
| Hit rate da Camada 1 | `layer1Hit / (jobs entrando no LLM)` | ≥ 6% | ≥ 7% | ≥ 8% | ≥ 10% | < 4% por 48h seguidas |
| Média `layer_2_hint_count` quando populada | `AVG(layer_2_hint_count) WHERE layer_2_hint_count > 0` | 3-10 | 3-10 | 3-10 | 3-10 | < 2 ou > 12 |
| Conflitos da Camada 0 | `preCheckConflict / preCheckHit` | observacional | < 15% | < 10% | < 5% | > 20% sustentado (a partir de 25%) |
| Redução de chamadas LLM | vagas curadas via Camada 0 / total curadas | observacional | ≥ 3% | ≥ 6% | ≥ 7% | — (métrica observacional) |
| Taxa de `llm_disobeyed_pre_resolution` | `events WHERE event_name='llm_disobeyed_pre_resolution'` | < 5% dos hits Camada 1 | < 4% | < 3% | < 2% | > 10% indica problema no prompt |
| Taxa de parse errors (LLM) | `ai_usage_logs WHERE event_name='prompt_parse_failed'` | baseline atual ± 20% | idem | idem | idem | > 2× baseline |
| Latência média de batch | medição do próprio `processBatch` | 1.0-1.2× baseline | 1.0-1.2× | 1.0-1.2× | 1.0-1.2× | > 1.5× baseline |
| Race conditions `precheck_update_conflict` | contador `precheckUpdateConflict` | < 1% dos hits | < 0.5% | < 0.3% | < 0.2% | > 2% |

**Legenda "observacional":** no estágio 10%, a métrica é registrada e observada mas **não dispara rollback**. Útil para baseline histórica.

**v5.3 (G6) — thresholds numéricos esperados no estágio 10%:**

O pool inicial de âncoras é composto pelas ~100 vagas `human_validated=true` backfilled com `prompt_structure_version=PROMPT_STRUCTURE_VERSION` (Canonical GSI + Stone + AgileEngine). Contra um fluxo de ~500-800 curagens novas por dia em 10% do tráfego (~50-80 vagas/dia entram no pipeline novo), a expectativa matemática é:

- **Hit rate Camada 0 esperado no estágio 10%:** entre **0.5% e 2%** (varia com a taxa de repetição de empresas que publicam vagas similares à Canonical GSI/Stone/AgileEngine — baixa inicialmente, cresce à medida que o pool se forma organicamente).
- **Se hit rate for 0% por 48h consecutivas:** não é motivo para rollback, mas é motivo para investigação manual. Possíveis causas: pool não foi corretamente marcado como `human_validated=true`, backfill não executou a exceção B5, ou o filtro `.neq('prompt_structure_version', 'legacy')` está excluindo as vagas que deveria incluir. Query diagnóstica:

```sql
-- Validar que pool inicial existe e está elegível
SELECT COUNT(*) AS pool_inicial
FROM job_postings
WHERE human_validated = true
  AND prompt_structure_version != 'legacy'
  AND curation_status = 'curated'
  AND description_hash IS NOT NULL;
-- Esperado: >= 80 linhas (tolerância para vagas muito curtas sem hash)
```

- **Se hit rate > 5% no estágio 10%:** também não é problema — indica que as vagas de entrada têm alta taxa de repetição das empresas do pool inicial. Apenas monitorar se o pool está sendo usado de forma saudável (não concentrado em 1-2 canônicos).

Gatilhos de rollback só ativam a partir do estágio 25% porque nesse ponto o pool já acumulou 1-2 semanas de curagens novas via pipeline e a base estatística é significativa.

#### 8.5.2 Validação pós-rollout: duas validações complementares (v5.2 — N5)

A v5.1 tinha uma única query SQL como validação pós-10%. Com a sobrescrita server-side em `persistCuratedJob` (§7.11), essa query sozinha perdeu poder de detectar bug na integração Camada 1 ↔ `buildUserPrompt`: `curation_source='layer_1_domain_synonyms'` pode aparecer mesmo quando o atributo XML nunca chegou ao LLM, porque o servidor sobrescreve no persist.

v5.2 separa a validação em duas:

**Validação A — Distribuição operacional por `curation_source` (SQL)**

Verifica se os caminhos do pipeline estão sendo exercitados. Roda 48h após cada estágio de rollout.

```sql
SELECT
    curation_source,
    COUNT(*) AS total,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS percent
FROM job_postings
WHERE curated_at >= NOW() - INTERVAL '48 hours'
  AND curation_status = 'curated'
GROUP BY curation_source
ORDER BY total DESC;

-- Resultado esperado em estágio 50% (aproximado):
--  curation_source             | total | percent
-- -----------------------------+-------+---------
--  llm_direct                  |  XXX  |  75-85%
--  layer_1_domain_synonyms     |  XX   |   5-12%
--  precheck_description_hash   |  XX   |   3-10%
--  llm_equivalences_redirect   |   X   |   1-3%

-- Gatilho de investigação (estágio 25% em diante):
-- • Se layer_1_domain_synonyms == 0 → rodar Validação B para diagnosticar
-- • Se precheck_description_hash == 0 e backfill rodou → bug na Camada 0 ou índice
```

**Validação B — Integração Camada 1 ↔ `buildUserPrompt` (teste unitário)**

Verifica que o atributo XML `canonical_already_resolved` é de fato incluído no prompt quando a Camada 1 pré-resolve. Não depende de runtime de produção; é executado no PR2 como teste unitário obrigatório.

```typescript
// tests/buildUserPrompt-layer1-integration.test.ts
import { describe, it, expect } from 'vitest';
import { buildUserPrompt } from '@/lib/pipeline/text-processing';

describe('buildUserPrompt — integração Camada 1', () => {
    it('inclui canonical_already_resolved quando populado', () => {
        const out = buildUserPrompt([{
            id: 'job-001',
            text: 'Descrição da vaga X',
            canonical_already_resolved: 'Diretor de Finanças'
        }]);

        // Atributo DEVE estar presente
        expect(out).toMatch(/canonical_already_resolved="Diretor de Finanças"/);
    });

    it('omite canonical_already_resolved quando ausente', () => {
        const out = buildUserPrompt([{
            id: 'job-002',
            text: 'Descrição X'
        }]);

        expect(out).not.toContain('canonical_already_resolved');
    });

    it('inclui suggested_roles quando populado', () => {
        const out = buildUserPrompt([{
            id: 'job-003',
            text: 'Descrição Y',
            suggested_roles: ['Analista Financeiro', 'Controller', 'Gerente de FP&A']
        }]);

        expect(out).toContain('suggested_roles="Analista Financeiro | Controller | Gerente de FP&amp;A"');
    });
});
```

**Resultado combinado:**
- Ambas passam → integração Camada 1 ↔ prompt está ok + distribuição em produção está saudável
- A falha, B passa → integração no código está ok mas em produção a Camada 1 não está sendo invocada (possivelmente flag desligada, queries de cache falhando, whitelist vazia)
- A passa, B falha → bug no código do `buildUserPrompt` (não deveria acontecer se tests rodaram no PR2)
- Ambas falham → bug no código OU nenhuma vaga entrou pela Camada 1 ainda (possível em estágio 10% se pool inicial for muito pequeno)

#### 8.5.3 Dashboard mínimo

Adicionar ao `/admin/dashboard` (ou similar) painel com 4 gráficos por estágio de rollout:

1. **Timeline de `curation_source`** nos últimos 7 dias (stacked area).
2. **Hit rate Camada 0 e Camada 1** nos últimos 7 dias (line chart).
3. **Top 10 canônicos por `curation_source='precheck_description_hash'`** — permite detectar se a Camada 0 está concentrando em poucos canônicos ou distribuindo bem.
4. **Contador de eventos forenses:** `precheck_conflict_detected`, `precheck_update_conflict`, `llm_disobeyed_pre_resolution`, `taxonomy_cache_load_failed`.

Dashboard pode ser criado em sprint paralela ao PR2. Ausência não bloqueia rollout mas dificulta validação rigorosa.

#### 8.5.4 Protocolo de decisão

- **Todas as métricas dentro das faixas-alvo por 48h:** avançar para próximo estágio.
- **1 métrica fora mas tendência de correção:** aguardar mais 24h e reavaliar.
- **1+ métrica em gatilho de rollback:** reduzir rollout para o estágio anterior, abrir incidente, investigar.
- **2+ métricas em gatilho de rollback:** rollback imediato para 0%, abrir P0.

### 8.6 Remediação de dados após rollback (v5.3)

A §8.2 especifica como **desligar** o pipeline (flag para 0%). Esta subseção especifica como **limpar** dados eventualmente corrompidos por uma janela problemática antes do rollback.

#### 8.6.1 Diagnóstico — identificar vagas afetadas

Antes de qualquer remediação, delimitar escopo. Exemplo: rollback às 14h00 após 6h de problema detectado às 08h00.

```sql
-- Passo 1: quais vagas foram curadas pelo pipeline v5.3 durante a janela?
SELECT
    id,
    curation_source,
    canonical_role_id,
    canonical_role_label,
    curated_at,
    description_hash,
    prompt_structure_version
FROM job_postings
WHERE curated_at BETWEEN '2026-XX-XX 08:00:00+00' AND '2026-XX-XX 14:00:00+00'
  AND curation_source IN ('precheck_description_hash', 'layer_1_domain_synonyms')
  AND curation_status = 'curated'
ORDER BY curated_at;

-- Passo 2: agrupar por canônico para ver se há concentração suspeita
SELECT
    canonical_role_label,
    curation_source,
    COUNT(*) AS total
FROM job_postings
WHERE curated_at BETWEEN '2026-XX-XX 08:00:00+00' AND '2026-XX-XX 14:00:00+00'
  AND curation_source IN ('precheck_description_hash', 'layer_1_domain_synonyms')
GROUP BY canonical_role_label, curation_source
ORDER BY total DESC;
```

#### 8.6.2 Remediação — reabrir vagas para recuradoria

Opção A: **remediação cirúrgica** (apenas canônicos específicos problemáticos):

```sql
UPDATE job_postings
SET
    curation_status = 'pending',
    canonical_role_id = NULL,
    canonical_role_label = NULL,
    canonical_resolved_at = NULL,
    curation_source = NULL,
    curated_at = NULL,
    skills = NULL,
    seniority_inferred = NULL,
    work_model = NULL,
    description_curated = NULL,
    layer_2_hint_count = NULL,
    human_validated = FALSE         -- v5.3 (G25): validação anterior não sobrevive a rollback
    -- NÃO limpar description_hash, prompt_structure_version, prompt_content_version
    -- (esses campos refletem apenas a versão do pipeline; permanecem válidos)
WHERE curated_at BETWEEN '2026-XX-XX 08:00:00+00' AND '2026-XX-XX 14:00:00+00'
  AND curation_source = 'precheck_description_hash'
  AND canonical_role_label IN ('<lista de canônicos problemáticos>');
```

Opção B: **remediação da janela inteira** (todas as vagas da janela, mais conservador):

```sql
UPDATE job_postings
SET
    curation_status = 'pending',
    canonical_role_id = NULL,
    canonical_role_label = NULL,
    canonical_resolved_at = NULL,
    curation_source = NULL,
    curated_at = NULL,
    skills = NULL,
    seniority_inferred = NULL,
    work_model = NULL,
    description_curated = NULL,
    layer_2_hint_count = NULL,
    human_validated = FALSE         -- v5.3 (G25): validação anterior não sobrevive a rollback
WHERE curated_at BETWEEN '2026-XX-XX 08:00:00+00' AND '2026-XX-XX 14:00:00+00'
  AND curation_source IN ('precheck_description_hash', 'layer_1_domain_synonyms');
```

**Nota v5.3 (G13 + G25):** a lista de campos resetados agora cobre `layer_2_hint_count`, `canonical_resolved_at` e `human_validated` — garantindo que nenhum metadado órfão da curagem anterior sobreviva ao rollback. Se a vaga era `human_validated=true` antes do incidente, a validação precisa ser refeita após a recuragem (a confiança na validação estava atrelada ao canônico antigo que foi revogado).

#### 8.6.3 Reprocessamento

Vagas reabertas serão naturalmente re-curadas pelo próximo ciclo de `/api/cron/curate-job-postings` (fluxo B) ou pelo admin manualmente (fluxo A). Com `PIPELINE_V3_ROLLOUT_PERCENT=0`, elas passam apenas pelo fluxo legado sem Camadas. Após problema identificado e corrigido, subir rollout gradualmente de novo.

#### 8.6.4 Registro de auditoria

Para qualquer remediação executada, registrar em `events` por registro afetado:

```sql
-- v5.6 (ChatGPT C5): actor='system' implica actor_id=SYSTEM_USER_ID (0001).
-- Remediação automática é ação do pipeline, não do admin Onsly (0004).
INSERT INTO events (event_name, resource_type, resource_id, actor, actor_id, reason, metadata)
SELECT
    'job_reopened_for_recuration',
    'job_posting',
    id,
    'system',
    '00000000-0000-0000-0000-000000000001',
    'Remediação pós-rollback de v5.3 — janela 08h-14h com problema em Camada 0',
    jsonb_build_object(
        'previous_canonical_role_label', canonical_role_label,
        'previous_curation_source', curation_source,
        'previous_curated_at', curated_at
    )
FROM job_postings
WHERE /* ... mesmos filtros da remediação ... */;
```

Sem esse registro, investigação futura não consegue reconstruir o que aconteceu.

---

## 9. Critérios de aceite

Critérios binários para merge de cada PR. **v5.1:** critérios de regressão usam seed determinístico para reprodutibilidade; métricas em média + desvio padrão de 3 execuções embaralhadas. **v5.2:** incorpora critérios para as 7 correções bloqueantes + 10 não-bloqueantes da 4ª rodada de revisão.

### 9.1 PR1 (conteúdo)

- [ ] Todos os testes unitários existentes passam.
- [ ] `domain_synonyms.json` v3.0 valida via Zod schema existente sem erro.
- [ ] Contagem: `jq '.data | length'` retorna 181 (113 + 68).
- [ ] `PROMPT_STRUCTURE_VERSION` (string `'v1'`) e `PROMPT_CONTENT_VERSION` (hash SHA-256 primeiros 8 chars) gerados pelo prebuild têm valores distintos e não-vazios.
- [ ] Medição de tokens adicionados no SYSTEM_PROMPT fica ≤1.800 (ou plano de mitigação documentado se ultrapassar).
- [ ] **Teste `tests/normalize-title.test.ts`:** todos os 7 casos-exemplo (§7.4.0) passam.
- [ ] **v5.2 — Teste `tests/domain-synonyms-hygiene.test.ts` (N1):** todas as 181 keys sobrevivem a `normalizeTitle` sem perder mais de 50% dos caracteres; não há duplicatas após normalização; todos os labels são strings não-vazias.
- [ ] **v5.2 — Teste `tests/build-user-prompt.test.ts` (B4):** os 8 casos (§7.8.1) passam, incluindo o caso `]]>` no corpo com split correto em duas seções CDATA.
- [ ] **Teste de regressão (3 execuções embaralhadas):**
    - 30 vagas aleatórias em canônicos saudáveis (Desenvolvedor Full Stack, Desenvolvedor Backend, Analista de Dados)
    - Seed determinístico via `crypto.createHash('sha256').update('regression-pr1').digest()` para shuffle reprodutível
    - Critério: **taxa média ≥ 90%** de consistência com canônico original; **desvio padrão ≤ 5 pontos percentuais** entre execuções.
- [ ] **Teste de correção (3 execuções embaralhadas):**
    - 20 vagas representativas dos 4 Tipos de erro do Diagnóstico
    - Mesmo seed scheme
    - Critério: **taxa média ≥ 85%** de canônico correto; **desvio padrão ≤ 8 pontos percentuais**.
- [ ] **v5.2 — Teste de sanidade Camada 1 (6 casos — B1):** vagas com títulos específicos que DEVEM bater no `domain_synonyms.json` após correção de normalização de keys:
    - "Head of Finance" → `Gerente de Finanças` (v5.5 — seed novo criado via §2.3 bis)
    - "Sales Operations Manager" → `Analista de Operações de Vendas` (v5.5 — após rename 24/04)
    - "Customer Service Representative" → `Analista de Customer Service`
    - "SDR" → `SDR` (v5.5 — após merge + rename no Passo 12)
    - "Tax Analyst" → `Analista Tributário`
    - **"Head of FP&A" → `Gerente de Finanças`** (caso crítico de `&` — teste que detecta B1 não corrigido)
    - Critério: **todos os 6 casos** retornam o canônico esperado via `lookupDomainSynonyms` em execução isolada (fora do LLM).

### 9.2 PR2 (infraestrutura)

#### 9.2.1 Build e migração

- [ ] TypeScript compila sem erros (strict mode).
- [ ] Migração SQL executa sem erro em staging e produção.
- [ ] Todas as 8 colunas novas em `job_postings` presentes (validação via `information_schema`).
- [ ] Tabela `allowed_for_pre_resolution` criada.
- [ ] Todos os 3 índices da §5.2 criados.
- [ ] **v5.2 — Trigger `trigger_sync_canonical_role_label_after_update` (N3):** criado; UPDATE em `job_canonical_roles.label` propaga para `job_postings.canonical_role_label`; evento `canonical_role_label_synced` é gravado.
- [ ] **v5.2 — Trigger `trigger_revoke_human_validated_before_update` (N4):** criado; UPDATE em `job_postings.canonical_role_id` com `curation_source='manual_remap'` revoga `human_validated` e grava evento `human_validated_revoked_by_manual_remap`.

#### 9.2.2 Backfill

- [ ] `scripts/backfill-job-description-hash.ts` completa em menos de 2 minutos para ~8.034 vagas.
- [ ] Vagas curadas pré-v5.2 **sem `human_validated`** têm `prompt_structure_version='legacy'`.
- [ ] **v5.2 — Vagas `human_validated=true` (B5):** têm `prompt_structure_version=PROMPT_STRUCTURE_VERSION` (valor atual, não `'legacy'`). Validação: `SELECT COUNT(*) FROM job_postings WHERE human_validated=true AND prompt_structure_version='legacy'` retorna 0 após backfill.
- [ ] **v5.2 — Cutoff temporal (B7):** vagas com `curated_at >= DEPLOY_CUTOFF_TIMESTAMP` **não são modificadas** pelo backfill. Verificação: antes do backfill, criar vaga de teste com `curated_at = NOW() + INTERVAL '1 minute'`; depois do backfill, confirmar que `prompt_structure_version` dessa vaga segue igual ao valor pré-backfill.
- [ ] Vagas com < 80 palavras têm `description_hash=NULL` e `prompt_content_version` populado.
- [ ] Vagas com ≥ 80 palavras têm `description_hash` populado (64 chars hex).
- [ ] Script é idempotente: rodar 2× não reprocessa nada novo.

#### 9.2.3 Funções da Camada 0

- [ ] `normalizeDescriptionForHash` produz output determinístico em 100 inputs de teste.
- [ ] `normalizeDescriptionForHash` trata HTML: `<p>X</p>` e `X` produzem mesmo hash.
- [ ] **v5.3 (GS3) — `normalizeDescriptionForHash` remove pontuação:** mesma descrição com bullets diferentes (`•`, `-`, `*`) produz hash idêntico. Teste: 3 variantes do mesmo texto (copiadas de Gupy, Greenhouse, LinkedIn) devem produzir o mesmo SHA-256.
- [ ] **v5.3 (GS3) — `normalizeDescriptionForHash` preserva acentos:** "Análise" e "Analise" produzem hashes **diferentes** (Unicode \p{L} preserva acentos; PT-BR é relevante semanticamente).
- [ ] `precheckDescriptionHash` retorna `hit=false, miss_reason='no_match_in_db'` para vaga nunca vista antes.
- [ ] `precheckDescriptionHash` retorna `hit=true` com 5 outputs clonados E `generated_hash` populado para vaga duplicada de uma `human_validated=true` (whitelistada).
- [ ] `precheckDescriptionHash` retorna `hit=false, miss_reason='canonical_not_allowed'` para vaga cujo canônico está em `validCanonicalLabels` mas não em `allowedForPreResolution`.
- [ ] `precheckDescriptionHash` retorna `hit=false, miss_reason='conflict_detected'` quando 2 canônicos distintos satisfazem quórum (gera evento `precheck_conflict_detected`).
- [ ] `precheckDescriptionHash` retorna `hit=false, miss_reason='title_guard_failed'` quando hash bate mas overlap de tokens do título < 2 (no caso geral).
- [ ] **v5.2 — Guard de título para C-level (B6):** vaga com título "CTO" como candidato E âncora com `original_title="CTO"` resulta em `hit=true` (overlap exato de 1 token suficiente porque ambos ≤ 2 tokens). Vaga "Chief Technology Officer" vs âncora "CTO" resulta em `hit=false` (candidato tem 3 tokens, cai em regra padrão ≥ 2).
- [ ] **v5.3 (G2) — Teste unitário explícito C-level:** criar `tests/precheck-description-hash.test.ts` com casos:
    - `titleGuardPasses("CTO", "CTO")` → `true` (ambos ≤ 2 tokens, ambos têm token C-level, overlap 1)
    - `titleGuardPasses("CEO", "Chief Executive Officer")` → `false` (candidate tem 1 token, âncora tem 3 tokens; não ambos ≤ 2; regra padrão exige ≥ 2)
    - `titleGuardPasses("Product Manager", "Product Manager")` → `true` (overlap 2, regra padrão)
    - `titleGuardPasses("Analista de Dados", "Desenvolvedor de Software")` → `false` (overlap 1 em "de", mas "de" é stopword e não conta)
    - **v5.4 (GS-R6-4)** `titleGuardPasses("Analista Sênior", "Analista Júnior")` → `false` (ambos 2 tokens e overlap 1 em "analista", **mas nenhum token pertence à lista C-level** — cai na regra padrão ≥ 2, retorna `false`). Teste crítico: regressão deste caso significa Camada 0 clonando cross-seniority.
    - **v5.4 (GS-R6-4)** `titleGuardPasses("Gerente Sênior", "Gerente Pleno")` → `false` (mesmo cenário da linha acima)
    - `titleGuardPasses("UX Designer", "UX Designer")` → `true` (overlap 2, regra padrão)
    - `titleGuardPasses("UX", "UX")` → `true` (ambos 1 token, ambos têm "ux" em C-level, overlap 1)
    - `titleGuardPasses("SDR", "BDR")` → `false` (overlap 0 — nenhum token em comum)
    - **v5.4 (GS-R6-4)** `titleGuardPasses("PM", "CEO")` → `false` (ambos 1 token, ambos C-level, mas overlap 0)
- [ ] **v5.2 — Filtro `original_title NOT NULL` (N7):** query de candidates da Camada 0 exclui vagas com `original_title=NULL`. Teste: criar âncora válida sem `original_title`; confirmar que vaga nova com hash idêntico retorna miss `no_match_in_db`.
- [ ] Vagas curadas pré-v5.2 com `human_validated=false` (`prompt_structure_version='legacy'`) NUNCA aparecem como âncoras.

#### 9.2.4 Funções das Camadas 1 e 2

- [ ] **v5.2 — Normalização de keys (B1):** mesmo teste do 9.1 repetido aqui com `lookupDomainSynonyms` real — 6 casos incluindo "Head of FP&A" retornam canônico esperado.
- [ ] **v5.2 — Piso de match na Camada 2 (B2):** título com 1 token genérico (ex: "Analista") passado a `findDomainMatch` retorna `null` (não dispara shortlist). Título com 2 tokens ("Analista Financeiro") passados retorna shortlist não-vazia.
- [ ] **v5.2 — Invalidação de cache por hash das keys (N6):** simular mudança de 1 key no `domain_synonyms.json` (mesma contagem total) e confirmar que `ensureMeta` recompila o regex (novo padrão reflete nova key).
- [ ] `lookupDomainSynonyms` — teste de consistência sob concorrência: **mesma entrada produz mesmo resultado em 100 chamadas `Promise.allSettled`** (valida ausência de race em `lastIndex` de regex).
- [ ] `lookupDomainSynonyms` — valida destino via `allowedForPreResolution` in-memory (zero queries extras por chamada).
- [ ] Regex de `domain-synonyms-lookup.ts` com heurística de tokens não tem falsos positivos em 50 títulos de teste manual (listados em `tests/domain-synonyms-false-positives.test.ts`).
- [ ] `buildSuggestedRoles` — 0 queries no caminho-quente quando cache está quente (testável via `@supabase/supabase-js` mock contando chamadas).
- [ ] `findFamilyMatch` e `findDomainMatch` — tokens duplicados no título não inflam score (teste com título `"Engenheiro Engenheiro de Software"`).

#### 9.2.5 Persistência

- [ ] `persistPrecheckResult` só afeta vagas em `curation_status='pending'` (teste: UPDATE em vaga já curada retorna `concurrent_update_detected=true`).
- [ ] `persistCuratedJob` grava `description_hash`, `prompt_structure_version`, `prompt_content_version`, `curation_source`, `canonical_resolved_at`, `layer_2_hint_count` quando recebidos em `PersistOptions`.
- [ ] **v5.2 — Override server-side executável (B3):** `persistCuratedJob` recebendo `pre_resolved_canonical_label='Diretor de Finanças'` + `pre_resolved_canonical_role_id='<uuid>'` + `normalized.canonicalRole='Analista Financeiro'` (LLM desobedeceu) grava `canonical_role_id` = `'<uuid>'` (valor pré-resolvido) e registra evento `llm_disobeyed_pre_resolution`.
- [ ] **v5.2 — Contador `llmDisobeyedPreResolution` (N8):** incrementado pelo `persistCuratedJob` quando há divergência entre LLM e pré-resolução.

#### 9.2.6 Rollout

- [ ] `stablePercent(jobId)` retorna o mesmo valor para o mesmo ID em execuções sucessivas.
- [ ] **v5.2 — Distribuição com 4 hex chars (N10):** em 10.000 IDs aleatórios, nenhum decil tem mais de 1.150 nem menos de 850 resultados (média esperada 1.000 ± 15%). Teste em `tests/rollout-sampling.test.ts`.
- [ ] Tempo médio de batch com Camadas 0+1+2 ativas não excede 1.2× o tempo do batch sem elas.
- [ ] **v5.2 — Validação A pós-rollout (N5-A):** após 48h em 25% do tráfego, query da §8.5.2 mostra `curation_source='layer_1_domain_synonyms'` > 0. No estágio 10% esta verificação é observacional (pode ser 0 legitimamente por cold start).
- [ ] **v5.2 — Validação B integração (N5-B):** `tests/buildUserPrompt-layer1-integration.test.ts` passa, confirmando que o atributo XML `canonical_already_resolved` é incluído corretamente quando populado em `PreparedJob`.

#### 9.2.7 Observabilidade (v5.2)

- [ ] **N2 — Evento `precheck_miss_summary`:** ao fim de cada batch com `preCheckMiss > 0`, 1 evento agregado é inserido em `events` com categorias (`short_description`, `no_match_in_db`, `no_quorum`, `conflict_detected`, `title_guard_failed`, `canonical_not_allowed`) e seus contadores.
- [ ] **N8 — Contadores mapeados:** rodar smoke test de 100 vagas e confirmar que `preCheckHit`, `preCheckMiss`, `layer1Hit`, `layer2HintCount` foram incrementados alguma vez (outros são esperados em condições raras).
- [ ] **N9 — Trilha forense em `events`:** hits da Camada 0 geram evento `precheck_hit_forensic` em `events` (não em `ai_usage_logs`).

#### 9.2.8 Endpoint admin

- [ ] `POST /api/admin/jobs/[id]/human-validated` com `{ validated: true }` atualiza flag e cria evento `human_validated_set`.
- [ ] Mesmo endpoint com `{ validated: false }` reverte e cria evento `human_validated_revoked`.
- [ ] Chamada sem auth de admin retorna 401.

---

## 10. Fora do escopo

Itens deliberadamente excluídos desta v5.4. Esta seção é a lista operacional; a §1.3 é o resumo para stakeholders.

### 10.1 Itens pertencentes à sprint de governança de taxonomia (pt. 15)

A sessão paralela "CalibraCV pt. 15 — Adequação de Habilidades" está no planejamento de uma sprint única de governança que absorve os itens abaixo. Executor do PR2 da v5.4 **não** deve implementar nada desta lista — se algum vier como sugestão de reviewer, redirecionar para a sprint pt. 15.

- **RPC `merge_canonical_roles` (paridade com `merge_canonical_skills`).** A v5.4 usa a tela admin de merge de canonicals de roles que já existe no CalibraCV. Criar RPC formal é escopo da pt. 15.
- **Aba de histórico/auditoria nas telas admin (skills + roles).** Consulta de decisões de merge/archive com filtros (actor, data, tipo). Hoje as decisões ficam apenas em `events` sem UI dedicada.
- **Cron de curadoria contínua usando modelo de raciocínio forte (Opus/GPT/Gemini — benchmark dentro da sprint pt. 15).** Detecta pares de canônicos com alta similaridade, canônicos com `skill_type` inconsistente entre sinônimos, e propõe merges/reclassificações. Threshold de confiança calibrado para auto-apply vs revisão humana.
- **Re-classificação retroativa do catálogo de skills (~9.000 canônicos).** Primeira passada com Sonnet + prompt calibrado, segunda passada com Opus sobre resíduo ambíguo. Corrige drift histórico de classificação.
- **Re-classificação retroativa do catálogo de roles (~9.000 canônicos).** v5.4 deploya prompt novo (PR1). Canônicos existentes continuam classificados sob prompt antigo (`prompt_version = 'legacy'` via §5.5). Reaferição com prompt maduro é escopo da pt. 15 — o filtro `WHERE prompt_version = 'legacy'` é query triagem direta.
- **Revisão da definição de `skill_type` no `SYSTEM_PROMPT.ts`.** A pt. 15 identificou que a definição de `hybrid` é circular. Reescrita faz parte do escopo dela.
- **Dedupe de 41 duplicatas em `skill_merge_decisions`, novo UNIQUE expression-based, normalização retroativa de convenção source=absorbed, tratamento dos 4 cross-type da R2.** Todo o lado skills.
- **Telas admin para aprovação pending → active em roles.** Hoje é manual via SQL ou via endpoint sem UI dedicada.
- **Auto-promoção de canônicos pending para active.** 533 canônicos em pending é débito bloqueante para qualidade do catálogo. v5.4 oferece whitelist `allowed_for_pre_resolution` como paliativo (controle manual). Automação baseada em `distinct_sources_count`, `vacancy_count` ou cron Opus é escopo da pt. 15.

**Dependência temporal:** a pt. 15 declarou textualmente que depende da v5.4 estar deployada antes de começar. Como o ambiente ainda não está em produção assistida (produto não é funcional E2E conforme contexto do Onsly em 24/04/2026), a janela de "estabilidade pós-deploy" antes de iniciar a pt. 15 é definida pelo próprio Onsly — pode ser imediata após validação técnica dos testes que ele pretende rodar antes do MVP, não exige período fixo em produção assistida.

### 10.2 Backlog operacional geral

**v5.6 (ChatGPT ✓3) — Item "Agendamento de cron já implementado:** o `vercel.json` já registra o cron `/api/cron/curate-job-postings` com schedule definido. Item removido do backlog porque está em produção. Histórico: estava listado na v5.5 como "independente das mudanças desta spec" mas Claude Code confirmou via auditoria do `vercel.json` que já foi feito.

**Bug pré-existente das 2 CHECK constraints duplicadas em `curation_status` (v5.4 pós-pt.15).** A pt. 15 reportou via Claude Code: `docs/supabase-schema.sql` linhas 567 e 569 definem duas CHECKs para `curation_status` com interseção `{pending, curated, low_quality, retryable_error, quarantined_llm_output}`. Valores `curated_fallback` e `llm_extractor` aparecem em uma das CHECKs mas não na outra, o que pode causar bloqueio de UPDATE se o schema for aplicado exatamente dessa forma em staging/prod fresh. **Não é escopo da v5.4** porque ela não introduz valores novos em `curation_status` (ela adiciona colunas novas). Se executor notar erro de CHECK constraint violation durante UPDATE do Passo 12, investigar essa duplicação como causa candidata. Consolidação em uma única definição autoritativa é escopo da sprint pt. 15 ou sprint separada de limpeza de schema.

**Migração dos 4 JSONs de taxonomia para tabelas em DB com admin UI.** Já planejada como sprint separada, independente da pt. 15.

**Tabela central `prompt_versions`.** Backlog pós-v5.4.

**Detecção automática de divergência sistêmica em hashes idênticos.** Item §8.4 cobre detecção pós-deploy via monitoramento; automação de remediação é sprint futura.

**Backlog de ~12 vagas em status REQUER HUMANO.** Criação de canônicos novos é decisão humana.

**Trigger `canonical_role_id_changed`.** Removido em relação à v2. Rastreabilidade via `curation_source` + `events`.

**Embedding fallback na Camada 2.** Removido por simplicidade. Fica como sprint pós-v5.4 se monitoramento da §8.4 indicar necessidade.

**Correção SQL manual dos 2 grupos de 16+ vagas com 100% divergência (Stone, AgileEngine).** Onsly executará antes do PR2, análogo ao procedimento com Canonical GSI. Essas vagas podem ser marcadas como `human_validated=true` via endpoint admin após correção manual. A Validação GS-R6-14 no Passo 12 garante convergência.

**Índice cobrindo também `prompt_structure_version` no predicado (otimização):** seq scan atual é barato dado baixa cardinalidade por hash. Fica como nota observável se `prompt_structure_version` mudar com alta frequência.

**Detecção automática de "LLM desobedeceu pré-resolução" como sinal de degradação do prompt.** Evento `llm_disobeyed_pre_resolution` é gravado pela v5.4 mas não há gatilho automático de alerta. Monitoramento via dashboard da §8.5.3.

**Métricas de ROI de custo LLM.** A v5.4 mede `preCheckHit`, `layer1Hit`, `llmOutputQuarantined` mas não calcula custo evitado em USD. Pode ser adicionado em dashboard posterior — o cálculo é `(preCheckHit + layer1Hit) × custo_médio_por_chamada_Sonnet`.

**Remoção de `requirements.description` como campo JSONB.** Após v5.4 popular `original_description` diretamente, `requirements.description` vira storage morto. Remoção fica para sprint de limpeza pós-rollout estável — janela definida pelo Onsly dado estágio do produto.

**Quebra do documento em ADR + Spec + Runbook.** Sugerido por reviewer mas adiado: v5.4 é otimizada para execução única pelo Claude Code. Reestruturação em 3 artefatos faz sentido pós-rollout, quando o documento vira peça de manutenção histórica.

---

## 11. Checklist de execução para Claude Code

### 11.1 Confirmações pré-execução (TodoWrite obrigatório — §2)

- [ ] **v5.6 (P3 confirmado pelo Onsly) — Vercel Pro: upgrade agendado pós-MVP.** O handler `curate-job-postings` declara `maxDuration = 300s` mas plano Hobby tem cap de 10s. Durante o período pré-MVP em Hobby, 3 CRONs estão degradados (documentado em `project_vercel_hobby_upgrade_backlog.md`). Pipeline novo pode rodar em staging/dev com `PIPELINE_V3_ROLLOUT_PERCENT=0` sem impacto. Rollout para >0% em produção **só após upgrade**. Enquanto isso, Fluxo B via CRON pode falhar em batches grandes — aceita-se como limitação conhecida do período pré-MVP. Executor NÃO bloqueia execução por este item; apenas documenta estado atual do ambiente no TodoWrite.
- [ ] §2.1 — Colunas pré-requisitas presentes em produção (validação SQL via `information_schema.columns` retorna 2 linhas em `job_postings`: `original_title` e `original_description`). v5.6 (ChatGPT C1) — pré-requisito corrigido: a v5.5 pedia 5 colunas em `job_postings` + colunas em `analyses`/`ai_usage_logs`, mas essas 5 referências eram resquício órfão que nunca foi criado pela spec
- [ ] §2.2 — `ingest-validation.ts` plugado nos 3 fluxos A/B/C (grep retorna callers em paths nomeados)
- [ ] §2.3 — 5 canônicos C-level criados via seed idempotente (`ON CONFLICT (slug) DO NOTHING`); SELECT retorna 5
- [ ] §2.4 — Query de validação executada; classificação apresentada ao Onsly; **resposta em formato (a)/(b)/(c) para cada canônico pending recebida**; whitelist `allowed_for_pre_resolution` populada conforme decisão
- [ ] §2.5 — Todos os itens de confirmação de estado reportados. v5.6 (ChatGPT I2) — `LLM_BATCH_SIZE` é env var com default 38, não constante hardcoded; confirmar valor efetivo em runtime (`process.env.LLM_BATCH_SIZE`), não só em `constants.ts`

**v5.6 (Gemini #4) — Aviso operacional sobre rename de canônicos massivos:**

A trigger N3 de sincronia de `canonical_role_label` (§7.14) propaga o novo label para todas as vagas com `canonical_role_id` correspondente via UPDATE. Para canônicos com muitas vagas atreladas (ex: "Desenvolvedor Full Stack" com 5.000+ vagas), esse UPDATE adquire `RowExclusiveLock` em milhares de linhas simultaneamente. Se o pipeline de ingestão (Fluxo B CRON) ou curadoria estiver rodando no mesmo momento, pode haver gargalo de performance ou deadlock pontual.

**Recomendação operacional:** renomeações de canônicos com mais de 1.000 vagas atreladas devem preferencialmente ser executadas **fora do horário de pico dos CRONs de ingestão** (janela noturna ou madrugada). Para canônicos com menos de 1.000 vagas, impacto é desprezível. A arquitetura não precisa de mudança — apenas disciplina operacional.

### 11.2 Execução PR1 (Conteúdo)

- [ ] Medir overhead de tokens do request COMPLETO (SYSTEM_PROMPT + 38 vagas médias + output) com `tiktoken` e reportar número
- [ ] Validar contagem atual de `domain_synonyms.json` via `jq '.data | length'` (esperado: 113)
- [ ] Aplicar patch em `domain_synonyms.json` com bump 2.0 → 3.0 e 68 entradas novas
- [ ] Validar contagem final: 113 + 68 = 181
- [ ] Validar JSON via Zod schema existente
- [ ] Aplicar Mudança 1 (Etapa 0) no `SYSTEM_PROMPT.ts`
- [ ] Aplicar Mudança 2 (Etapa 2 renumerada) no `SYSTEM_PROMPT.ts`
- [ ] Aplicar Mudança 3 (Regras A-E) no `SYSTEM_PROMPT.ts`
- [ ] Aplicar Mudança 4 (micro-regra Support) no `SYSTEM_PROMPT.ts`
- [ ] Renumerar referências internas do prompt para nova ordem
- [ ] Adicionar `normalizeTitle` em `lib/pipeline/text-processing.ts` (§7.4.0) com teste `tests/normalize-title.test.ts`
- [ ] **v5.2 — Criar `tests/domain-synonyms-hygiene.test.ts` (§7.4.0.1, N1):** valida higiene de keys no prebuild
- [ ] **v5.2 — Criar `tests/build-user-prompt.test.ts` (§7.8.1, B4):** 8 casos incluindo `]]>` split em CDATA
- [ ] **v5.2 — Adicionar `test:hygiene` ao `package.json` como passo de prebuild:** `"prebuild": "npm run test:hygiene && node scripts/generate-prompt-version.ts"`
- [ ] Atualizar `scripts/generate-prompt-version.ts` com `STRUCTURE_VERSION='v1'` hardcoded + hash SHA-256 para `CONTENT`
- [ ] Atualizar `buildUserPrompt` em `lib/pipeline/text-processing.ts` (§7.8) com escape XML completo + CDATA
- [ ] Rodar prebuild; confirmar `PROMPT_STRUCTURE_VERSION='v1'` e `PROMPT_CONTENT_VERSION=<hash>` distintos em `prompt-version.generated.ts`
- [ ] Commit único: `feat(curation): domain_synonyms v3.0 + SYSTEM_PROMPT A-E rules + CDATA prompt + hygiene tests (v5.2 PR1)`
- [ ] Deploy em produção
- [ ] Executar validação §9.1 (teste de regressão, teste de correção, teste de sanidade Camada 1 com 6 casos incluindo "Head of FP&A" — todos com seed determinístico)
- [ ] Reportar resultados em issue de tracking

### 11.3 Execução PR2 (Infraestrutura pre-LLM)

**v5.5:** deploy unificado com PR1, sem janela de espera de 7 dias (ambiente não está em produção assistida). Passo 12 deve rodar antes ou em conjunto com Passo 13 — os dois saem juntos.

#### 11.3.0 Ordem canônica do Passo 12 (v5.5 — Claude-2 #2)

A v5.4 tinha a ordem distribuída entre o §11.3 (checklist linear), o Anexo B (ordem narrada), e a §5 (ordem textual das migrations). A v5.5 consolida em **uma única tabela autoritativa** referenciada por todos os outros lugares. Executor segue apenas esta tabela.

| # | Sub-passo | Descrição | Referência |
|---|---|---|---|
| 12.1 | **Backup** | Executar `CREATE TABLE ... _backup_v5_5 AS SELECT *` para as 4 tabelas (condicional nas 2 de merge_decisions) + DO block de validação de contagens | §5.0 |
| 12.2 | **Migration §5.1 — colunas novas `job_postings`** | 10 colunas incluindo `original_title`, `original_description` com backfill inline e validação fill-rate pós-UPDATE | §5.1 |
| 12.3 | **Validação §5.1** | `SELECT COUNT(*) FROM job_postings WHERE original_title IS NULL OR original_description IS NULL` deve retornar 0 | §5.1 |
| 12.4 | **Migration §5.1.2 — trigger imutabilidade** | Criar função `enforce_original_immutability()` e trigger `BEFORE UPDATE` em `job_postings` | §5.1.2 |
| 12.5 | **Migration §5.5 — coluna `prompt_version`** | `ALTER TABLE job_canonical_roles` + backfill `'legacy'` + constraint NOT NULL DEFAULT `'unspecified'` | §5.5 |
| 12.6 | **Migration §5.2 — índices** | 3 índices compostos + parciais para performance da Camada 0 | §5.2 |
| 12.7 | **Migration §5.4 — whitelist** | Criar tabela `allowed_for_pre_resolution` com FK para `job_canonical_roles` | §5.4 |
| 12.8 | **Migration §5.6 — coluna `curation_source` + trigger N4** | v5.5 (Decisão 4): criar coluna `curation_source` com CHECK, criar endpoint admin de remap, criar trigger N4 que reage a `curation_source='manual_remap'` | §5.6 |
| 12.9 | **Trigger N3 — sincronia `canonical_role_label`** | Aplicar `docs/migrations/20260423_04_canonical_role_label_sync_trigger.sql` | §7.14 |
| 12.10 | **Operações de taxonomia** | Via tela admin atual: merge "Representante de Pré-Venda" → "SDR", rename "Desenvolvimento de Negócios" → "BDR", archive "Supervisor de Pré-Vendas", setar "Executivo de Pré-Venda" como `pending` sem whitelist | Anexo B |
| 12.11 | **Seeds** | Seed 5 C-level (CTO, CEO, COO, CMO, CFO) + seed "Gerente de Finanças" como `active`, ambos com `prompt_version = PROMPT_STRUCTURE_VERSION` | §2.3, Anexo B |
| 12.12 | **Whitelist populada** | INSERTs em `allowed_for_pre_resolution` para 14 (a) + 4 (b) = 18 canônicos (Anexo B) | Anexo B |
| 12.13 | **`human_validated = true`** | Marcar vagas de pool inicial (Canonical GSI, Stone, AgileEngine) | §2.5 |
| 12.14 | **Validação triggers** | `SELECT tgname FROM pg_trigger WHERE tgname IN ('trg_sync_canonical_role_label', 'trg_revoke_human_validated_on_remap', 'trg_enforce_original_immutability')` retorna **3 linhas** (v5.6) | §11.3 |
| 12.15 | **Validação convergência (GS-R6-14)** | Query de convergência `human_validated` retorna 0 linhas | §11.3 |
| 12.16 | **Validação pool inicial (GS-R6-6)** | `pool_efetivo_inicial >= 80` (v5.5 — gate elevado de 50 para 80) | §11.3 |

**Validação cruzada:** executor NÃO pula sub-passos. Se qualquer sub-passo falhar, rollback via §5.0 e re-execução a partir do ponto de falha (a partir de 12.1 se foi corrupção de schema, a partir do sub-passo específico se foi falha pontual).

**Passo 12 — Migração SQL (checklist linearizado, segue tabela 11.3.0):**
- [ ] **12.1 (v5.5) — Backup pré-migration (§5.0):** executar os `CREATE TABLE ... _backup_v5_5 AS SELECT *` para `job_postings`, `job_canonical_roles`, e condicional para `skill_merge_decisions` e `role_merge_decisions`. Validar via DO block que contagens batem. Sem plano pago do Supabase, este é o único safety net para rollback operacional
- [ ] **12.2** — Aplicar migração §5.1 (10 colunas novas em `job_postings`, incluindo `canonical_role_label`, `layer_2_hint_count`, `canonical_resolved_at`, **`original_title`, `original_description`**)
- [ ] **12.3** — Validar que backfill inline da §5.1 populou `original_title` em 100% das vagas existentes (v5.5: validação fill-rate já está no DO block da §5.1)
- [ ] **12.4 (v5.5 — DeepSeek #1)** — Aplicar migração §5.1.2: função `enforce_original_immutability()` + trigger `BEFORE UPDATE` em `job_postings`. Testar rejeição: `UPDATE job_postings SET original_title = 'teste' WHERE id = <qualquer>` deve lançar EXCEPTION
- [ ] **12.5 (v5.4 pós-pt.15)** — Aplicar migração §5.5: coluna `prompt_version` com backfill `'legacy'` + NOT NULL DEFAULT `'unspecified'`. Validar: `SELECT COUNT(*) FROM job_canonical_roles WHERE prompt_version IS NULL` retorna 0
- [ ] **12.6** — Aplicar migração §5.2 (3 índices)
- [ ] **12.7** — Aplicar migração §5.4 (tabela `allowed_for_pre_resolution`)
- [ ] **12.8 (v5.5 — Decisão 4 greenfield)** — Aplicar migração §5.6: coluna `curation_source` com CHECK constraint, criação do endpoint `/api/admin/jobs/[id]/remap`, trigger N4 que reage a `curation_source='manual_remap'`
- [ ] **12.9** — Aplicar trigger N3 (sincronia `canonical_role_label`)
- [ ] **12.10 (v5.5 — ordem unificada)** — Executar operações de taxonomia via tela admin: merge Pré-Venda → SDR, rename Desenvolvimento → BDR, archive Supervisor, pending-c Executivo
- [ ] **12.11** — Seeds: 5 C-level + Gerente de Finanças
- [ ] **12.12** — Popular whitelist com decisões do Onsly (§2.4 cravado no Anexo B): 18 canônicos (14 a + 4 b)
- [ ] **12.13** — Marcar `human_validated=true` em vagas de pool inicial (Canonical GSI, Stone, AgileEngine)
- [ ] **12.14** — Validar triggers: `SELECT tgname FROM pg_trigger WHERE tgname IN ('trg_sync_canonical_role_label', 'trg_revoke_human_validated_on_remap', 'trg_enforce_original_immutability')` retorna **3 linhas** (v5.6 — Claude Code: v5.5 dizia "4 linhas" contando N4 e "revogação" como distintos, mas são o mesmo trigger)
- [ ] **12.15** — Validar todas as colunas via `information_schema.columns`
- [ ] **12.16 (v5.6 — ChatGPT G6)** — **Validação pool inicial (GS-R6-6):** rodar query do pool inicial e confirmar `pool_efetivo_inicial >= 80`. Sub-passo estava na tabela §11.3.0 mas tinha sumido do checklist linear da v5.5 — reinserido.

- [ ] **v5.4 (GS-R6-14) — Validar convergência de `human_validated`:** rodar query abaixo para garantir que nenhum grupo de vagas com mesma descrição e marcadas como human_validated aponta para canônicos distintos. Query deve retornar **zero linhas** — se retornar algo, há inconsistência no pool inicial (Stone/AgileEngine) que contamina a Camada 0 com âncoras divergentes:

```sql
SELECT
    description_hash,
    COUNT(DISTINCT canonical_role_id) AS distinct_canonicals,
    ARRAY_AGG(DISTINCT canonical_role_label) AS labels,
    COUNT(*) AS total_jobs
FROM job_postings
WHERE human_validated = true
  AND description_hash IS NOT NULL
GROUP BY description_hash
HAVING COUNT(DISTINCT canonical_role_id) > 1;
-- Esperado: 0 linhas. Se retornar linhas, revisar com Onsly.
```

- [ ] **v5.5 (GS-R6-6 + Claude-2 #5) — Gate de pool inicial elevado para 80:** rodar query abaixo e confirmar que o pool efetivo de âncoras elegíveis é **≥ 80**. A v5.4 tinha gate em 50; v5.5 eleva para 80 porque 50 era aceitável apenas em modo observacional — para rollout efetivo com hit rate mensurável, 80 âncoras é o piso recomendado. Se pool < 80, não subir rollout para 10% — fazer curadoria manual adicional de vagas de empresas com templates repetitivos (Canonical GSI, Stone, AgileEngine, Nubank, iFood) até atingir o mínimo:

```sql
SELECT COUNT(*) AS pool_efetivo_inicial
FROM job_postings
WHERE human_validated = true
  AND prompt_structure_version != 'legacy'
  AND curation_status = 'curated'
  AND description_hash IS NOT NULL
  AND original_title IS NOT NULL;  -- v5.2 N7: guard de título exige título populado
-- Gate v5.5 (Claude-2 #5):
--   >= 80: subir para 10% com confiança
--   < 80: BLOQUEAR rollout — curadoria manual adicional obrigatória
```

**Passo 13 — Deploy do código com rollout em 0%:**

*Tipos e estrutura:*
- [ ] **Estender `PreparedJob` em `lib/pipeline/types.ts`** (§7.7.1): 8 campos opcionais (6 da v5.1 + 2 novos v5.2: `pre_resolved_canonical_label`, `pre_resolved_canonical_role_id`)
- [ ] **Estender `RunCounters` em `lib/pipeline/types.ts`**: 6 contadores da v5.1 + `preCheckMissByReason` (objeto com 6 categorias) + `llmDisobeyedPreResolution` (v5.2)
- [ ] **Atualizar `createRunCounters()` em `lib/pipeline/types.ts`**: inicializar todos os contadores em 0, incluindo objeto `preCheckMissByReason`
- [ ] **v5.2 — Adicionar §7.7.1.1 na prática:** confirmar que cada contador é incrementado no ponto especificado (tabela contador → incremento)

*Normalizações:*
- [ ] Extrair `normalizeResumeText` para `lib/minhash.ts` (§7.1) com aspas tipográficas e nbsp
- [ ] Adicionar `normalizeDescriptionForHash` em `lib/minhash.ts` (§7.1) com HTML strip completo
- [ ] Atualizar 3 callers do landing/upload para importar de `@/lib/minhash`

*Cache de taxonomia:*
- [ ] Atualizar `getFullTaxonomyCache` em `lib/pipeline/taxonomy-cache.ts` (§7.2) com preload de `validCanonicalLabels`, `allowedForPreResolution` (via JOIN com whitelist), `vacancyCountByLabel`, **`roleIdByLabel` (v5.2)**

*Novos arquivos Camadas 0/1/2:*
- [ ] Criar `lib/pipeline/precheck-description-hash.ts` (§7.3) com `generated_hash` no hit, guard de título **com exceção C-level (B6)**, detecção de conflito, `allowedForPreResolution` param, filtro `!= 'legacy'`, **import estático de `normalizeTitle`**, **filtro `.not('original_title', 'is', null)` (N7)**
- [ ] Criar `lib/pipeline/domain-synonyms-lookup.ts` (§7.4.1) — regex sem estado global, **keys normalizadas via `normalizeTitle` antes de compilar regex (B1)**, **invalidação de cache via hash SHA-256 das keys (N6)**, `allowedForPreResolution` param
- [ ] Criar `lib/pipeline/suggested-roles-builder.ts` (§7.5) — zero query, ranking in-memory, dedup de tokens, **piso `< 2` em `findDomainMatch` (B2)**

*Persistência:*
- [ ] Criar `lib/pipeline/persist-precheck.ts` (§7.10) — função `persistPrecheckResult` com guard de estado (`.eq('curation_status', 'pending')`) e detecção de race
- [ ] **Modificar `lib/pipeline/persist-curation.ts` (§7.11) — CRÍTICO**: `persistCuratedJob` aceita 8 campos novos em `PersistOptions` (6 da v5.1 + `pre_resolved_canonical_label` e `pre_resolved_canonical_role_id` da v5.2) + sobrescrita server-side via B3 + evento `llm_disobeyed_pre_resolution` + parâmetro `counters` opcional para incrementar `llmDisobeyedPreResolution`

*Rollout e endpoint:*
- [ ] Criar `lib/pipeline/rollout-sampling.ts` (§7.12) com `stablePercent()` **usando 4 hex chars (N10)** e `isJobInRollout()`
- [ ] **v5.2 — Criar `tests/rollout-sampling.test.ts`:** 3 casos (determinismo, distribuição uniforme em 10k IDs, anti-viés por decil)
- [ ] **v5.2 — Criar `tests/buildUserPrompt-layer1-integration.test.ts` (§8.5.2, N5-B):** 3 casos validando atributos XML do prompt
- [ ] Criar `app/api/admin/jobs/[id]/human-validated/route.ts` (§7.13) com autenticação admin e auditoria

*Integração:*
- [ ] Modificar `batch-processor.ts` (§7.7.2) com snippet v5.3: rollout via `isJobInRollout`, `precheckOnly` real, passagem de `candidateTitle` e `allowedForPreResolution` para Camada 0, **destructuring com `roleIdByLabel` para inline de override server-side (C3)**, `allowedForPreResolution` para Camada 1, **categorização de `miss_reason` via `ctx.counters.preCheckMissByReason[reason]++` (N2)**, **evento agregado `precheck_miss_summary` ao fim do batch (N2)**, **hit path da Camada 1 preenche `job.pre_resolved_canonical_label` + `pre_resolved_canonical_role_id` inline via `roleIdByLabel.get()` (B3/C3)**, **incremento de `layer2HintCount` pelo tamanho da shortlist (N8)**, **evento `precheck_hit_forensic` em `events` no lugar de `ai_usage_logs` (N9)**, **branch `else if (result.concurrent_update_detected)` com contador `precheckUpdateConflict` + evento `precheck_update_conflict` (GS8)**, caller de `persistCuratedJob` passando os 8 `PersistOptions` + `ctx.counters`
- [ ] Atualizar `scripts/generate-prompt-version.ts` (§7.9) com `STRUCTURE_VERSION='v1'` hardcoded

*Scripts:*
- [ ] Criar `scripts/backfill-job-description-hash.ts` (§7.6) com condição idempotente OR, `prompt_structure_version='legacy'`, exponential backoff, **captura de `DEPLOY_CUTOFF_TIMESTAMP` no início + filtro `.lt('curated_at', cutoff)` (B7)**, **exceção `human_validated=true` → `PROMPT_STRUCTURE_VERSION` (B5)**, **contadores `totalSkippedPostDeploy` e `totalHumanValidated`**
- [ ] Criar `scripts/monitor-precheck-regression.ts` (§8.4) — agendamento via cron depois do passo 18

*Env vars e deploy:*
- [ ] Adicionar env vars `PIPELINE_V3_ROLLOUT_PERCENT=0` e `PIPELINE_V3_PRECHECK_ONLY=false`
- [ ] Deploy em produção (rollout em 0% = nenhuma vaga usa Camadas)

**Passo 14 — Backfill (v5.2):**
- [ ] Executar `npx tsx scripts/backfill-job-description-hash.ts`
- [ ] **v5.2 — Validar logs expandidos:** confirmar que output mostra `totalSkippedPostDeploy`, `totalHumanValidated`, `totalSkippedShort`, `totalFailed` separadamente
- [ ] **v5.2 — Validar B5 via SQL:** `SELECT COUNT(*) FROM job_postings WHERE human_validated=true AND prompt_structure_version='legacy'` retorna 0
- [ ] **v5.2 — Validar B7 via SQL:** criar 1 vaga teste com `curated_at=NOW()+1min`, rodar backfill, confirmar que `prompt_structure_version` dessa vaga não foi alterado
- [ ] Validar via SQL: vagas curadas pré-cutoff sem `human_validated` têm `prompt_structure_version='legacy'`
- [ ] Validar via SQL: vagas com descrição ≥80 palavras têm `description_hash` populado; < 80 palavras têm NULL
- [ ] Validar sem falhas (`totalFailed` == 0)
- [ ] Reexecutar o script 1× para validar idempotência (deve retornar "0 processadas")

**Passo 14.1 — Cleanup de órfãs pós-backfill (v5.4, GS-R6-2):**

**Contexto do risco:** entre o deploy do Passo 13 (código novo em produção com `ROLLOUT_PERCENT=0`) e a execução do Passo 14 (backfill), vagas podem ser curadas pelo pipeline legado (fluxo B via CRON de ingestão). Como essas vagas entram no `batch-processor.ts` mas caem no branch `jobsLegacy` (rollout=0), são persistidas via caminho antigo **sem** `description_hash`, `prompt_structure_version` ou `curation_source` populados. O backfill posterior, com filtro de cutoff temporal (`B7: .lt('curated_at', DEPLOY_CUTOFF_TIMESTAMP)`), pula essas vagas — deixando-as **órfãs do schema novo permanentemente**.

**Sintoma:** vagas com `curated_at >= DEPLOY_CUTOFF_TIMESTAMP` AND `curated_at < <início do backfill>` que têm `prompt_structure_version IS NULL`.

**Query diagnóstica:**

```sql
-- Identificar vagas órfãs (curadas pelo legado no gap deploy → backfill)
SELECT
    id,
    curated_at,
    canonical_role_label,
    curation_status,
    description_hash IS NULL AS hash_missing,
    prompt_structure_version IS NULL AS version_missing,
    curation_source IS NULL AS source_missing
FROM job_postings
WHERE curated_at >= '<DEPLOY_CUTOFF_TIMESTAMP>'::timestamptz
  AND curated_at < '<timestamp_de_inicio_do_backfill>'::timestamptz
  AND curation_status = 'curated'
  AND (description_hash IS NULL OR prompt_structure_version IS NULL);
```

**Ação de cleanup:**

- [ ] Rodar query diagnóstica acima. Registrar contagem de órfãs.
- [ ] Se contagem > 0: executar segundo passe do backfill sem filtro de cutoff, restrito ao subconjunto órfão:

```bash
# Modo cleanup — reprocessa vagas com (hash NULL OR version NULL)
# curadas entre DEPLOY_CUTOFF e NOW, sem respeitar o cutoff.
CLEANUP_MODE=true \
CLEANUP_MIN_CURATED_AT='<DEPLOY_CUTOFF_TIMESTAMP>' \
CLEANUP_MAX_CURATED_AT='<timestamp_de_inicio_do_backfill>' \
npx tsx scripts/backfill-job-description-hash.ts
```

O script em modo cleanup:
- Ignora `DEPLOY_CUTOFF_TIMESTAMP`
- Filtra por `curated_at BETWEEN CLEANUP_MIN_CURATED_AT AND CLEANUP_MAX_CURATED_AT`
- Só toca vagas com `description_hash IS NULL OR prompt_structure_version IS NULL`
- Marca `prompt_structure_version='legacy'` (não são human_validated — curadas pelo legado)

- [ ] Re-rodar query diagnóstica. Deve retornar 0 linhas.
- [ ] Se ainda houver órfãs, investigar origem manualmente (possível bug de race não previsto).

**Prevenção futura:** após a v5.4 estar em produção, próximas mudanças de schema que adicionem colunas obrigatórias ao schema novo devem seguir ordem: (1) migration cria colunas nullable, (2) deploy código popula, (3) backfill, (4) migration altera colunas para NOT NULL. Essa ordem elimina a janela de vagas órfãs.

**Passo 15 — Rollout gradual (5 estágios v5.3):**
- [ ] **v5.3 — Pré-requisito antes do estágio 10%:** rodar `npx vitest run tests/buildUserPrompt-layer1-integration.test.ts` e confirmar que os 3 casos passam (Validação B, §8.5.2, N5-B). Esse teste valida que o atributo XML `canonical_already_resolved` é de fato incluído no prompt quando a Camada 1 pré-resolve — sem isso, hit na Camada 1 não chega ao Sonnet e a sobrescrita server-side fica invisível para o modelo
- [ ] **v5.3 — Pré-requisito antes do estágio 10%:** rodar query de pool inicial (§8.5.1):
    ```sql
    SELECT COUNT(*) AS pool_inicial
    FROM job_postings
    WHERE human_validated = true
      AND prompt_structure_version != 'legacy'
      AND curation_status = 'curated'
      AND description_hash IS NOT NULL;
    ```
    Esperado: >= 80 linhas. Se < 80, backfill não executou a exceção B5 corretamente — investigar antes de subir rollout
- [ ] Subir `PIPELINE_V3_ROLLOUT_PERCENT=10`; aguardar 48h
- [ ] **v5.2 — Observacional apenas no estágio 10%:** Camada 0 não tem gatilho de rollback (cold start esperado — pool inicial de ~100 âncoras)
- [ ] **v5.3 — Hit rate Camada 0 esperado no estágio 10%:** 0.5-2%. Se for 0% por 48h consecutivas, rodar query diagnóstica de §8.5.1 antes de decidir
- [ ] Rodar **Validação A (§8.5.2)** pós-48h: confirmar `curation_source='layer_1_domain_synonyms'` > 0 (Camada 1 funcionando)
- [ ] Verificar métricas §8.5.1 para estágio 10% — se Camada 1 dentro das faixas e Camada 0 observacional dentro do intervalo esperado, prosseguir
- [ ] Subir para 25%; aguardar 48h; **Camada 0 agora tem gatilho ativo** (faixa ≥ 3%); validar
- [ ] Subir para 50%; aguardar 48h; validar
- [ ] Subir para 75%; aguardar 48h; validar
- [ ] Subir para 100%; aguardar 7 dias; validar estabilidade
- [ ] Remover env var de rollout (setar permanente 100% no código ou eliminar check) após 7 dias estáveis
- [ ] Agendar cron semanal de `scripts/monitor-precheck-regression.ts` (`0 2 * * 0` UTC)
- [ ] Commit incremental por etapa: `feat(pipeline): v5.3 layer 0/1/2 — PR2 step N`

### 11.4 Rollback e remediação

- [ ] Se qualquer gatilho de rollback da §8.5.1 disparar: setar `PIPELINE_V3_ROLLOUT_PERCENT=0` imediatamente
- [ ] Executar diagnóstico da §8.6.1 para identificar vagas afetadas
- [ ] Executar remediação (§8.6.2) conforme gravidade — opção A (cirúrgica) ou B (janela inteira)
- [ ] Registrar eventos de auditoria (§8.6.4) para todas as vagas remediadas
- [ ] Abrir incidente e investigar causa-raiz antes de novo rollout

---

## Anexo A — Lista de canônicos do patch (a validar antes do PR1)

Executar query da §2.4 e classificar cada canônico em (a) promover, (b) whitelist, ou (c) sem auto-decisão. Reportar ao Onsly antes de prosseguir com o PR1.

**Canônicos criados via seed §2.3:**

| Canônico | Ação | Status esperado após seed |
|---|---|---|
| CTO | seed | active |
| CEO | seed | active |
| COO | seed | active |
| CMO | seed | active |
| CFO | seed | active |

Os 5 entram automaticamente em `allowedForPreResolution` (status=active).

**Canônicos do patch — classificação apenas após execução da query §2.4.** Exemplos conhecidos da v3 (podem ter mudado desde então). v5.6 (Claude Code #3) — tabela atualizada para refletir operações de taxonomia já cravadas no Anexo B (rename de Sales Operations, merge Pré-Venda → SDR, rename Desenvolvimento de Negócios → BDR):

| Canônico esperado | Status típico | Decisão padrão sugerida |
|---|---|---|
| Analista de Customer Service | pending | (b) whitelist |
| Analista de Customer Experience | pending | (b) whitelist |
| Analista de Operações de Vendas | pending | (b) whitelist — v5.6: rename de "Analista de Sales Operations" |
| Analista de Marketing Digital | pending | (b) whitelist |
| Analista Tributário | pending | (b) whitelist |
| Analista de Compliance | pending | (b) whitelist |
| Coordenador de Projetos | pending | (b) whitelist |
| Gerente de Engenharia | pending | (b) whitelist |
| Gerente de TI | pending | (b) whitelist |
| Gerente de Operações | pending | (b) whitelist |
| Gerente de Customer Success | pending | (b) whitelist |
| Gerente de Finanças | não retornado | criar via seed (§2.3 bis) |
| Gerente de Marketing | a validar | Onsly decide |
| Gerente de Produto | a validar | Onsly decide |
| Diretor de Finanças | pending | (b) whitelist |
| Diretor de Customer Success | pending | (b) whitelist |
| Diretor de Marketing | a validar | Onsly decide |
| Diretor de Produto | a validar | Onsly decide |
| SDR | active | — (automático, absorveu Representante de Pré-Venda via merge) |
| BDR | active | — (automático, rename de Analista de Desenvolvimento de Negócios) |
| Analista de Recursos Humanos | active | — (automático) |
| Analista de Marketing | active | — (automático) |

Executor não assume nada — executa query, apresenta resultado com sugestões padrão, e aguarda resposta final do Onsly.

---

## Anexo B — Decisões §2.4 cravadas (v5.6 final)

A §2.4 pedia classificação (a)/(b)/(c) para cada canônico pending impactado pelo `domain_synonyms.json` v3.0 + decisão sobre canônicos missing + operações de taxonomia correlatas. As decisões foram tomadas e estão cravadas abaixo. Este anexo é o contrato de execução do Passo 12.

**Legenda de opções:**

- **(a) Promover para `active`** — canônico vira `active` e entra em `allowed_for_pre_resolution`. Camada 1 pode auto-decidir.
- **(b) Manter `pending` + incluir em whitelist** — canônico continua `pending` mas recebe INSERT em `allowed_for_pre_resolution`. Camada 1 pode auto-decidir. Reversível via DELETE na whitelist.
- **(c) Manter `pending` sem whitelist** — canônico fora da whitelist. Camada 1 emite miss `canonical_not_allowed` — LLM decide.

### Missing (1 canônico) — criar via seed no Passo 12

| Canônico | Decisão | Justificativa |
|---|---|---|
| Gerente de Finanças | (a) `active` via seed | Alto volume, synonyms já apontam para ele em `domain_synonyms.json` v3.0 |

### Pending — 19 canônicos (excluindo 2 absorvidos por operações de taxonomia abaixo)

**v5.5 (Claude-2 polimento):** executor resolve UUID via query `SELECT id FROM job_canonical_roles WHERE label = '<nome>'` no momento do INSERT em `allowed_for_pre_resolution`. UUIDs não são listados na tabela porque a v5.4 foi cravada sem eles para permitir que o executor use a fonte autoritativa (banco) em vez de cópia estática que pode ficar obsoleta.

| Canônico | Decisão | Justificativa |
|---|---|---|
| Analista de Operações Financeiras | (c) | Pode colidir com "Analista Financeiro" — manter LLM até desambiguação futura |
| Analista Tributário | (a) | Função estabelecida, baixa ambiguidade |
| Diretor de Finanças | (a) | C-suite bem definido |
| Analista de Marketing Digital | (b) | Variação comum — Camada 1 pode auto-decidir, revisar depois |
| Diretor de Marketing | (a) | C-suite |
| Diretor de Produto | (b) | Convive com CPO no seed — avaliar após 30 dias |
| Gerente de Marketing | (a) | Função estabelecida |
| Gerente de Produto | (a) | PM é cargo consagrado no mercado BR |
| Coordenador de Projetos | (a) | Cargo consagrado no mercado BR |
| Gerente de Operações | (a) | Função estabelecida |
| Analista de Customer Experience | (a) | Analista de CX é função consagrada |
| Analista de Customer Service | (a) | Analista de CS é função consagrada |
| Diretor de Customer Success | (b) | Anglicismo; avaliar consolidação |
| Gerente de Customer Success | (a) | CSM é cargo consagrado no mercado BR |
| Analista de Compliance | (b) | Avaliar conflito com "Analista Jurídico" |
| Analista de Compras | (c) | Pode colidir com "Assistente de Compras" / "Comprador" |
| Analista de Recrutamento e Seleção | (a) | Função estabelecida |
| Gerente de Engenharia | (a) | Função estabelecida |
| Gerente de TI | (a) | Cargo consagrado no mercado BR |

**Distribuição final:** 13 (a) + 4 (b) + 2 (c) = 19 decisões.

### Operações de taxonomia (executar via tela admin atual)

As operações abaixo **não usam RPC nova**. A tela admin de merge de canonicals de roles já existe no CalibraCV (veio em sprint anterior). Essas operações devem ser executadas via essa tela ou via endpoint TS equivalente. Uma eventual migração para RPC `merge_canonical_roles` (paridade com `merge_canonical_skills`) é **escopo da sprint de governança da pt. 15**, não da v5.4.

| # | Operação | Detalhes |
|---|---|---|
| 1 | MERGE | `Representante de Pré-Venda` (UUID `d780aacf-be11-4aeb-ad61-7b242a60178a`, 189 vagas) → `SDR` (UUID já `active`, 249 vagas). Resultado: SDR absorve, fica com 438 vagas. "Representante de Pré-Venda" passa para `status='deprecated'` (v5.6 — ChatGPT C3: `archived_at` não existe em `job_canonical_roles`). |
| 2 | RENAME | `Analista de Sales Operations` (UUID `d5ff56cd-84a3-49c9-9db8-062fa08a58f6`, 27 vagas) → `Analista de Operações de Vendas`. UUID preservado, label/slug atualizados. **Já executado pelo Onsly em 24/04** via SQL direto. |
| 3 | RENAME | `Analista de Desenvolvimento de Negócios` (UUID `b2aa7f8e-e56c-4ea5-9126-009af445aade`, 113 vagas) → `BDR`. UUID preservado, label/slug atualizados. Justificado por 80% das vagas desse canônico terem "BDR" no título bruto. |
| 4 | ARCHIVE | `Supervisor de Pré-Vendas` (UUID `96a478e3-e5ca-4161-a624-450f547c7a36`, 0 vagas). Canônico zumbi. Passar para `status='deprecated'` (v5.6: não existe `archived_at` em `job_canonical_roles`; operação é UPDATE direto do status). |
| 5 | PENDING (c) | `Executivo de Pré-Venda` (UUID `ef683757-55bc-4d04-8435-306c42d75a90`, 29 vagas). Semanticamente é Account Executive mascarado, não SDR. Manter pending sem whitelist e revisitar individualmente. |

### Decisões de nomenclatura (fechadas)

- **"Analista de Sales Operations" canônico oficial** → `Analista de Operações de Vendas` (português). Decisão baseada em princípio default + amostra de 15 títulos mostrando que EN e PT estão equilibrados no catálogo (9 EN vs 6 PT) com preferência metodológica ao PT.
- **"SDR" canônico oficial** → `SDR` (inglês, sigla consagrada). Decisão baseada em 80/19 distribuição EN/PT nos títulos brutos (249 vs 58). "Pré-Venda" e variações PT entram como sinônimos em `domain_synonyms.json`.
- **"BDR" canônico oficial** → `BDR` (inglês, sigla consagrada). Decisão análoga (80% de títulos com "BDR" na amostra). "Desenvolvimento de Negócios" e variações PT entram como sinônimos.

### Sinônimos novos em `domain_synonyms.json` v3.0

Incluir no patch do PR1 (se não estiverem ainda):

**Para "SDR":**
- `"sdr"`, `"business development representative"`, `"sales development representative"`, `"representante de pré-venda"`, `"representante de pre-venda"`, `"pré-vendas"`, `"pre-vendas"`, `"analista de pré-vendas"`, `"analista de pre-vendas"`

**Para "BDR":**
- `"bdr"`, `"business development associate"`, `"bda"`, `"analista de desenvolvimento de negócios"`, `"analista de desenvolvimento de negocios"`

**Para "Analista de Operações de Vendas":**
- `"sales operations"`, `"sales ops"`, `"revops"`, `"rev ops"`, `"revenue operations"`, `"operações de vendas"`, `"operacoes de vendas"`

**NÃO incluir como sinônimo** (evitar falsos positivos):
- `"operações comerciais"` / `"operacoes comerciais"` — ambíguo, pode ser back-office de vendas OU pós-venda operacional. LLM decide.
- `"analista de novos negócios"` / `"executivo de novos negócios"` — ambíguo, pode ser BDR OU AE/parcerias estratégicas. LLM decide.

### Ordem de execução no Passo 12

**v5.5 (Claude-2 #2):** a ordem de execução canônica do Passo 12 está consolidada na tabela **§11.3.0** ("Ordem canônica do Passo 12"). Este anexo apontava para uma versão em prosa que fica desatualizada quando a §11.3.0 é modificada. Para evitar duas fontes de verdade, a ordem autoritativa é a tabela em §11.3.0 com 16 sub-passos numerados (12.1 a 12.16).

Claude Code segue **apenas** a tabela §11.3.0 no Passo 12. Se algo falhar, rollback via §5.0 + correção + re-execução a partir do sub-passo que falhou.

---

**Fim da especificação.**
