# Pipelines CI/CD, SonarQube y ambiente QA — diseño y puesta en marcha

Estado: los archivos ya están creados **localmente en cada repo** (sin commit ni push — los
commits los haces tú). Este documento explica qué hay, cómo funciona y los pasos que solo
tú puedes hacer en GitHub (secrets, environment QA, protección de main).

## La regla central: `main` = producción

- **Las ramas `feature/*` NUNCA despliegan.** En ellas (y en los PRs) corre solo el CI de
  calidad (`ci.yml`: tests + cobertura + Sonar).
- **Solo un push/merge a `main` dispara el pipeline de despliegue** (`build.yml`). Esto ya era
  así en los workflows existentes (`on: push: branches: [main]`) y se mantiene — hoy están
  todos los repos en ramas `feature/Juank-Adelanto-*`, y por eso nada se ha desplegado por CI.
- Recomendado (GitHub → Settings → Branches): proteger `main` con *require pull request* +
  *require status checks* (el check `ci` de este pipeline) → nadie mergea con tests rojos.

## Qué se creó (por repo)

| Archivo | Dónde | Qué hace |
|---|---|---|
| `ci-test.yml` | battlecaos-infra/.github/workflows | **Reusable**: npm ci → tests → cobertura (lcov+html, artifact 7 días) → análisis Sonar (solo si el secret existe) |
| `ci.yml` | los 9 repos de servicio | Invoca el reusable en PRs a main/develop y pushes a ramas feature |
| `build.yml` (reescrito) | los 9 repos | El deploy ahora tiene **gate**: `test → build(dev) → qa-aprobación (pausa humana) → deploy-qa(test)` |
| `sonar-project.properties` | los 9 repos | Config del scanner: fuentes, tests, exclusiones, ruta del lcov |
| `docker-compose.sonar.yml` | deploy/sonarqube (raíz) | SonarQube Community local (http://localhost:9000) |

También se ajustó el script `coverage` de los 9 `package.json` para generar `lcov.info`
(el formato que Sonar consume) además del HTML.

## El flujo completo tras el merge a main

```
push/merge a main
   └─ test  (GATE: 272 tests + cobertura + Sonar — si falla, AQUÍ muere)
       └─ build  (imagen → ACR de dev → actualiza la Container App de dev)
           └─ qa-aprobacion  (environment 'qa' de GitHub: ESPERA aprobación humana)
               └─ deploy-qa  (imagen → ambiente test de Azure = battlecaostest-*)
```

## Pasos que debes hacer TÚ en GitHub (una sola vez)

1. **Commit + push** de estos archivos en cada repo (rama feature → PR → merge a main).
2. **Environment `qa`** — en CADA repo de servicio: Settings → Environments → New environment
   → nombre `qa` → marca **Required reviewers** y agrégate. Eso convierte el job
   `qa-aprobacion` en una pausa de aprobación manual real.
3. **Secrets de Sonar** (opcional pero recomendado) — en cada repo (o como *Organization
   secrets* de BattleCaos-Ship, una sola vez):
   - `SONAR_TOKEN`: token generado en SonarCloud (o en tu SonarQube self-hosted accesible).
   - `SONAR_HOST_URL`: `https://sonarcloud.io` (o la URL de tu servidor).
   - Para SonarCloud: crea la organización, importa los repos, y descomenta
     `sonar.organization` en cada `sonar-project.properties`.
   - Sin estos secrets **nada se rompe**: el paso de Sonar simplemente se salta.
4. **Ambiente test de Azure (para QA)** — el job `deploy-qa` apunta al resource group
   `battlecaostest-rg`. Existe la config (`battlecaos-infra/terraform/environments/test.*`)
   pero la infra de test **no está aplicada** (cuesta crédito). Hasta que corras
   `terraform init -backend-config=environments/test.backend.hcl && terraform apply
   -var-file=environments/test.tfvars`, el job fallará con un error claro ("No encontré un
   ACR en battlecaostest-rg"). El pipeline queda listo-pero-dormido; no afecta a dev.
5. **Protección de main** (recomendado): Settings → Branches → Add rule para `main` con
   *Require a pull request* y *Require status checks* → selecciona el check del CI.

## SonarQube local (sin depender de GitHub)

```powershell
cd deploy/sonarqube
docker compose -f docker-compose.sonar.yml up -d     # http://localhost:9000 (admin/admin)
# análisis de un servicio:
cd ..\..\battlecaos-game
npm run coverage
docker run --rm -v "${PWD}:/usr/src" -e SONAR_HOST_URL=http://host.docker.internal:9000 -e SONAR_TOKEN=<token> sonarsource/sonar-scanner-cli
```
En la UI ves: bugs, code smells, duplicación, cobertura por archivo y la evolución histórica
por análisis (el "seguimiento" pedido).

## ⚠️ Nota importante sobre archivos fuera de los repos

Estas carpetas viven en la RAÍZ del proyecto, que **no es un repositorio git** — hoy no se
versionan en ningún lado: `deploy/` (monitoreo, DR, sonar), `tools/` (k6, chaos, canary),
`docs/event-contracts.md`, `ATRIBUTOS-CALIDAD.md`, `docker-compose.yml`.
Sugerencia: muévelas (o cópialas) a `BattleCaos-Ship-Backend` (este repo paraguas) para que
queden versionadas. Decisión tuya — no se movió nada automáticamente.
