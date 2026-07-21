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
| `build.yml` (reescrito) | los 9 repos | El deploy tiene **gate**: `test → qa-aprobación (pausa humana) → build(dev)`. NO hay ambiente 'test' de Azure aparte — el seguimiento de QA se hace con Grafana/Loki/Prometheus sobre dev. |
| `sonar-project.properties` | los 9 repos | Config del scanner: fuentes, tests, exclusiones, ruta del lcov |
| `docker-compose.sonar.yml` | deploy/sonarqube (raíz) | SonarQube Community local (http://localhost:9000) |

También se ajustó el script `coverage` de los 9 `package.json` para generar `lcov.info`
(el formato que Sonar consume) además del HTML.

## El flujo completo tras el merge a main

```
push/merge a main
   └─ test  (GATE: tests + cobertura + Sonar — si falla, AQUÍ muere)
       └─ qa-aprobacion  (environment 'qa' de GitHub: ESPERA aprobación humana si tiene reviewers;
       │                  si no está configurado, pasa de largo → no rompe el pipeline)
           └─ build  (imagen → ACR de dev → actualiza la Container App de dev)
```
> No hay un RG `battlecaostest-rg` ni un job `deploy-qa`: se quitaron porque no hay ambiente
> `test` de Azure (decisión: el seguimiento de QA se hace con la observabilidad sobre dev).

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
4. **Secrets de Azure en CADA repo** (OIDC federado) — el paso `azure/login` necesita
   `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` en **todos** los repos (o como
   *Organization secrets* de BattleCaos-Ship, una vez para todos). Además, cada repo necesita su
   propia **credencial federada** en el App Registration de Azure AD, con subject
   `repo:BattleCaos-Ship/battlecaos-<servicio>:ref:refs/heads/main`. Si faltan, el build falla con
   "Login failed... Not all values are present. Ensure 'client-id' and 'tenant-id' are supplied".
   > **Nota:** este fue el error que dio `voice-channel` — le faltaban esos secrets/credencial.
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

## Seguridad Sonar → rating B/A (cambios del 2026-07-19)

SonarCloud (análisis automático sobre main) daba C en casi todo y E en frontend/infra. Arreglos:

| Regla | Qué era | Arreglo aplicado (local, por commitear) |
|---|---|---|
| S6437 BLOCKER (frontend) | `__tmp-cm.mjs` con el JWT_SECRET hardcodeado, público en GitHub | Archivo borrado. **HAY QUE ROTAR el JWT_SECRET** (sigue en el historial git) |
| S7635 (9 repos) | `secrets: inherit` pasa TODOS los secrets al reusable | Lista explícita de secrets en ci.yml/build.yml; los reusables de infra los declaran en `workflow_call.secrets` |
| S7637 (9 repos + infra) | `uses: ...@main` / `@v3` (refs móviles) | Terceros anclados a SHA (sonarqube-scan-action, azure/login). Los `@main` de infra se anclan con `tools/pin-infra-sha.ps1` DESPUÉS de pushear infra |
| S7636 (infra) | Secret interpolado dentro de `run:` | GOOGLE_CLIENT_ID movido a `env:` |
| S2245 | `Math.random()` en game (backoff/autoFleet/handleShot), bot (strategy), room (generarCodigo), frontend (guia.html) | `node:crypto randomInt` / `crypto.getRandomValues` |
| S6471 (todos) | Contenedores node corriendo como root | `USER node` en los 9 Dockerfiles (frontend/nginx queda, es MINOR) |
| S5145 (auth) | Log de datos del usuario sin sanear | `logger.js` reemplaza caracteres de control por espacio |
| S6378 (infra) | ACR/storage sin identity | Bloques `identity { SystemAssigned }` |

**ORDEN de commit/push**: (1) battlecaos-infra primero (merge a main); (2) correr
`tools\pin-infra-sha.ps1` (ancla los `@main` al SHA nuevo de infra en los 10 repos);
(3) commit/push del resto. Si se pushean los repos ANTES que infra con secrets explícitos,
el workflow falla ("secret not defined in the referenced workflow").

**Quedan en infra 5 vulnerabilidades NO arreglables sin romper/pagar** — marcarlas como
*Accepted* en SonarCloud (proyecto infra → Issues → seleccionar → Accept, con comentario):
- S6329 BLOCKER (ACR `public_network_access`): deshabilitarlo exige SKU Premium (~10× costo); mitigado con OIDC + tokens AAD. 
- S6379 (ACR `admin_enabled`): las Container Apps usan user/pass del registro; migrar a managed identity está fuera del alcance del curso.
- S6382 ×3 (`client_certificate_mode`): son ingress TCP INTERNOS (kafka/redis) — el mTLS de cliente no aplica a transporte TCP interno.

**Resultados en GitHub**: badges de SonarCloud agregados al README de los 11 repos
(quality gate, security, reliability, maintainability — se actualizan solos). El check de
SonarCloud aparece en los PRs automáticamente (la GitHub App ya está instalada); para
verlo, trabajar por PR a main. Opcional: exigirlo en la protección de main.


Estas carpetas viven en la RAÍZ del proyecto, que **no es un repositorio git** — hoy no se
versionan en ningún lado: `deploy/` (monitoreo, DR, sonar), `tools/` (k6, chaos, canary),
`docs/event-contracts.md`, `ATRIBUTOS-CALIDAD.md`, `docker-compose.yml`.
Sugerencia: muévelas (o cópialas) a `BattleCaos-Ship-Backend` (este repo paraguas) para que
queden versionadas. Decisión tuya — no se movió nada automáticamente.
