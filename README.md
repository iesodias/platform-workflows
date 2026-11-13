# platform-workflows

Esteira CI/CD central (Git Flow) consumida pelos repos gerados a partir de `template-ts`, `template-java` e
`template-infra`. Os repos têm só **stubs finos**; toda a lógica vive aqui, versionada pela tag **`@v3`**
e parametrizada pelo `.advpipes.yml` de cada repo.

```yaml
# exemplo de stub (template-*/.github/workflows/2-dev.yml)
jobs:
  dev:
    uses: iesodias/platform-workflows/.github/workflows/flow-dev.yml@v3
    secrets: inherit
```

## Fluxo (cada etapa é disparada por um PR / ação humana)

| Stub no repo | Gatilho | Orquestrador | Faz |
|---|---|---|---|
| `1-feature.yml` | push em `feature/*` | `flow-feature.yml` | valida prefixo `feature/` · build/test · iac-scan ∥ helm-lint · abre PR → `develop` |
| `2-dev.yml` | PR mergeado em `develop` | `flow-dev.yml` | build único (versão = última tag + 1) · deploy DEV · cria `release/<ver>` e abre PR `develop` → `release/<ver>` |
| `3-hom.yml` | PR mergeado em `release/*` | `flow-hom.yml` | promove o mesmo pacote · deploy HOM · registra no Release `-rc` · abre PR `release/<ver>` → `main` |
| `4-prod.yml` | PR → `main` **editado** com `- [x] Aprovar deploy em PROD` | `flow-prod.yml` | promove · GMUD · deploy PROD (canary) |
| `5-release.yml` | PR mergeado em `main` | `flow-release.yml` | exige o deploy em PROD · tag final `v<ver>` + GitHub Release · remove `release/<ver>` |
| `6-hotfix.yml` | push em `hotfix/*` | `flow-hotfix.yml` | pipeline de emergência · PRs → `develop` e → `main` |
| `7-deploy-ops.yml` | issue com label `canary` / `rollback` | `flow-deploy-ops.yml` | rollout / reject do canary · rollback |

A esteira **abre PRs, mas nunca faz merge**. Mergear em `main` sem o deploy em PROD faz o workflow 5 falhar
e **não** gera a tag final.

## Convenções de nome

- **Arquivos:** kebab-case. `flow-*.yml` = orquestrador (um por stub, é o que os repos chamam). Sem prefixo =
  bloco reutilizado pelos orquestradores.
- **Blocos:** `config` (lê/valida o `.advpipes.yml` e aplica guards do evento) · `ci` (build/test por
  `project.language`: `ts`, `java-maven`, `infra`) · `sast` · `iac-scan` · `package` (registry via GitHub
  Releases) · `image` · `deploy` · `open-pr` · `tag`. `lint.yml` roda o actionlint neste próprio repo.
- **Vocabulário (um termo, um significado):**

  | Termo | Significado | Onde aparece |
  |---|---|---|
  | **build** | empacota uma única vez, no DEV (ou no hotfix) | `package`/`image` com `mode: build` |
  | **promote** | leva o MESMO pacote ao próximo ambiente, sem rebuild | `package`/`image` com `mode: promote` |
  | **approve** | checkbox `Aprovar deploy em PROD` no PR para `main` | `flow-prod` |
  | **rollout / reject** | decisão do canary, por Issue (label `canary`) | `deploy` com `deploy-type` |
  | **rollback** | volta para a versão final anterior, por Issue (label `rollback`) | `deploy` com `rollback: true` |
  | **release** | só a tag final `v<ver>` | `flow-release`, `tag` com `mode: final` (`mode: rc` anota o `-rc`) |

- **Títulos de PR:** `[DEV] feature/x → develop` · `[HOM] Release <ver> → release/<ver>` ·
  `[PROD] Release <ver> → main` · `[DEV]`/`[PROD] Hotfix hotfix/x → …`.
- **Issue Forms:** campos em português, delimitados por `>>Campo<<`: `>>Ambiente<<`, `>>Acao<<`, `>>Versao<<`, `>>GMUD<<`.

## Versionamento

- Tags imutáveis `v3.X.Y` a cada mudança; a tag major `v3` é movida para a última `v3.X.Y`.
- Mudança incompatível nos stubs/`.advpipes.yml` → nova major.
- Migração da v2: os arquivos foram renomeados (ex.: `develop.yml` → `flow-dev.yml`); repos antigos precisam
  trocar os stubs pelos do template atual e a label `promote` por `canary`.
