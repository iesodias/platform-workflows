# platform-workflows

Esteira CI/CD central (Git Flow) consumida pelos repos gerados a partir de `template-ts`, `template-java` e
`template-infra`. Os repos têm só **stubs finos**; toda a lógica vive aqui, versionada pela tag **`@v2`**
e parametrizada pelo `.advpipes.yml` de cada repo.

```yaml
# exemplo de stub (template-*/.github/workflows/2-develop.yml)
jobs:
  develop:
    uses: iesodias/platform-workflows/.github/workflows/develop.yml@v2
    secrets: inherit
```

## Fluxo (cada etapa é disparada por um PR / ação humana)

| Stub no repo | Gatilho | Orquestrador | Faz |
|---|---|---|---|
| `1-pull-request-to-develop.yml` | push em `feature/*` | `feature.yml` | valida prefixo `feature/` · build/test · terrascan ∥ helm-lint · abre PR → `develop` |
| `2-develop.yml` | PR mergeado em `develop` | `develop.yml` | build once (versão = última tag + 1) · deploy DEV · cria `release/<ver>` e abre PR `develop` → `release/<ver>` |
| `3-pre-release.yml` | PR mergeado em `release/*` | `pre-release.yml` | promove o mesmo pacote · deploy HOM · tag `-rc` · abre PR `release/<ver>` → `main` |
| `4-production.yml` | PR → `main` **editado** com `- [x] Aprovar deploy em PROD` | `production.yml` | promove · deploy PROD (canary) |
| `5-release.yml` | PR mergeado em `main` | `release.yml` | exige o deploy em PROD · tag final `v<ver>` + GitHub Release · remove `release/<ver>` |
| `6-hotfix.yml` | push em `hotfix/*` | `hotfix.yml` | pipeline de emergência · PRs → `develop` e → `main` |
| `8-rollback.yml` | issue com label `promote`/`rollback` | `rollback.yml` | promote / reject / rollback |

A esteira **abre PRs, mas nunca faz merge**. Mergear em `main` sem o deploy em PROD faz o workflow 5 falhar
e **não** gera a tag final.

## Reusables de 2º nível

`source_control` (lê/valida o `.advpipes.yml`) · `ci` (build/test por `project.language`: `ts`, `java-maven`,
`infra`) · `security` · `terrascan` · `publish` (registry via GitHub Releases) · `container` · `deploy` ·
`create_pr` · `auto_tag`. `lint.yml` roda o actionlint neste próprio repo.

## Versionamento

- Tags imutáveis `v2.X.Y` a cada mudança; a tag major `v2` é movida para a última `v2.X.Y`.
- Mudança incompatível nos stubs/`.advpipes.yml` → nova major (`v3`).
