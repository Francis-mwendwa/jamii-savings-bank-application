# Jenkins deployment pipeline

The repository has one pipeline in the root `Jenkinsfile`. It builds the Micro Integrator composite application, runs the Maven test lifecycle, creates API archives from the three checked-in OpenAPI specifications, deploys the CAR and database scripts to MI, and imports/updates the three APIM APIs with `apictl`.

## Pipeline stages

1. `Build, package and test MI` runs `mvn -B clean install`. This project binds the WSO2 Synapse unit-test plugin to Maven's `test` phase. There are currently no Synapse test case files, so Maven reports that no tests are present; the test phase still runs and any future tests will gate the build.
2. `Build API projects and archives` runs `apictl init` and `apictl bundle` for all three APIs. The generated projects are normalized to the APIM names and contexts listed below before bundling. Archives are fingerprinted in Jenkins.
3. `Deploy to dev` is automatic. It copies the SQL scripts first and atomically promotes the CAR into the MI deployment directory over SSH, waits for MI to report the composite application, then imports all three API archives with `--update`.
4. `Promote to prod` is skipped by default. Set `PROMOTE_TO_PROD=true` to show an approval gate and deploy the same build to prod.

Each `apictl import api` call is checked. The first failure exits the deployment step and marks the Jenkins build failed; subsequent API imports do not run. API Manager imports are individual operations, so if a later import fails, earlier successful imports may already be live. Jenkins logs name the failed API and do not suppress this partial state.

## API mapping

| OpenAPI file | APIM name | APIM context |
| --- | --- | --- |
| `AccountBalanceAPi.yaml` | `AccountBalanceAPi` | `/accountbalanceapi` |
| `LoanEligibilityApi.yaml` | `LoanEligibilityApi` | `/loaneligibilityapi` |
| `soapToRestTransformationApi.yaml` | `soapToRestTransformationApi` | `/soaptoresttransformationapi` |

## Jenkins agent

Use a Linux agent labelled `wso2-mi` with:

- Java 21 and Maven 3.9 or newer
- WSO2 `apictl` 4.2.6 or a version compatible with the target API Manager
- OpenSSH client (`ssh` and `scp`)
- Jenkins SSH Agent plugin

The agent must be able to reach the configured API Manager, MI management endpoint, and MI SSH host. Add the MI SSH host key to the agent's `known_hosts`; the pipeline requires strict host key checking. The workspace needs outbound access to the Maven repositories on the first build.

## Target configuration

Set these non-secret Jenkins environment variables at folder or agent level for dev. Define the equivalent `PROD` values only when production promotion is enabled.

| Variable | Example | Purpose |
| --- | --- | --- |
| `APIM_DEV_URL` | `https://apim-dev.example:9443` | API Manager endpoint for `apictl` |
| `MI_DEV_MANAGEMENT_URL` | `https://mi-dev.example:9164` | MI management endpoint for `apictl` |
| `MI_DEV_SSH_HOST` | `mi-dev.example` | SSH hostname reachable from the Jenkins agent |
| `MI_DEV_SSH_USER` | `wso2carbon` | SSH account that can write the MI directories |
| `MI_DEV_SSH_PORT` | `22` | Optional; defaults to 22 |
| `MI_DEV_CARBONAPPS_DIR` | `/home/wso2carbon/wso2mi-4.6.0/repository/deployment/server/carbonapps` | MI carbonapps directory |
| `MI_DEV_DATABASE_DIR` | `/home/wso2carbon/wso2mi-4.6.0/repository/database` | MI database directory for H2 scripts |
| `MI_DEV_BACKEND_BASE_URL` | `http://mi-dev.example:8290` | MI HTTP origin, without a path or trailing slash |
| `APIM_DEV_INSECURE_TLS` | `false` | Set `true` only for a development APIM with a self-signed certificate |
| `MI_DEV_INSECURE_TLS` | `false` | Set `true` only for a development MI management endpoint with a self-signed certificate |

For prod use the same names with `PROD` in place of `DEV`. The target paths must be absolute POSIX paths without spaces. Jenkins credentials are referenced by ID:

- `apim-dev-credentials` / `apim-prod-credentials`: username/password allowed to import and update APIs in APIM
- `mi-dev-credentials` / `mi-prod-credentials`: username/password allowed to use the MI management API
- `mi-dev-ssh` / `mi-prod-ssh`: SSH private key for the corresponding MI host

The API backend endpoints passed to APIM are `MI_*_BACKEND_BASE_URL` for AccountBalanceAPi, and that origin plus `/loans` and `/customers` for LoanEligibilityApi and soapToRestTransformationApi. Keep the base URL aligned with the MI listener and routing in that environment.

The account data service uses H2 `RUNSCRIPT` files from the MI home directory. The pipeline copies `db/schema.sql` and `db/seed.sql` into `MI_*_DATABASE_DIR` before promoting the CAR, so the datasource can initialize on a clean target.

## Local/CI targets

The pipeline does not start product containers itself. Point the dev variables at a disposable MI/APIM stack (for example, a CI `docker compose` stack with ports reachable from the Jenkins agent). The MI runtime must run the project-compatible WSO2 MI version and have writable target directories. `MI_*_BACKEND_BASE_URL` should resolve from APIM's gateway, not just from the Jenkins agent.