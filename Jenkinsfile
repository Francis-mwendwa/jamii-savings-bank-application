import groovy.transform.Field

@Field final String CAR_NAME = 'family-bank-assessment_1.0.0'

/** Deploys one stage using a single path for dev and prod. */
def deployEnvironment(String targetStage) {
    def suffix = targetStage.toUpperCase()
    def settings = [
        APIM_URL: env["APIM_${suffix}_URL"],
        MI_MANAGEMENT_URL: env["MI_${suffix}_MANAGEMENT_URL"],
        MI_SSH_HOST: env["MI_${suffix}_SSH_HOST"],
        MI_SSH_USER: env["MI_${suffix}_SSH_USER"],
        MI_SSH_PORT: env["MI_${suffix}_SSH_PORT"] ?: '22',
        MI_CARBONAPPS_DIR: env["MI_${suffix}_CARBONAPPS_DIR"],
        MI_DATABASE_DIR: env["MI_${suffix}_DATABASE_DIR"],
        MI_BACKEND_BASE_URL: env["MI_${suffix}_BACKEND_BASE_URL"],
        APIM_INSECURE_TLS: (env["APIM_${suffix}_INSECURE_TLS"] ?: 'false').toLowerCase(),
        MI_INSECURE_TLS: (env["MI_${suffix}_INSECURE_TLS"] ?: 'false').toLowerCase()
    ]
    def missing = settings.findAll { key, value -> !value?.trim() }.keySet()
    if (missing) {
        error("Missing Jenkins environment values for ${targetStage}: ${missing.join(', ')}")
    }
    if (!(settings.MI_SSH_HOST ==~ /^[A-Za-z0-9.-]+$/) || !(settings.MI_SSH_USER ==~ /^[A-Za-z0-9._-]+$/)) {
        error("${targetStage} SSH host/user must contain only letters, digits, dot, underscore, or hyphen")
    }
    if (!(settings.MI_SSH_PORT ==~ /^[0-9]{1,5}$/) || settings.MI_SSH_PORT.toInteger() < 1 || settings.MI_SSH_PORT.toInteger() > 65535) {
        error("${targetStage} SSH port must be between 1 and 65535")
    }
    ['MI_CARBONAPPS_DIR', 'MI_DATABASE_DIR'].each { key ->
        if (!(settings[key] ==~ /^\/[A-Za-z0-9._\/-]+$/)) {
            error("${key} for ${targetStage} must be an absolute POSIX path without spaces")
        }
    }
    if (!(settings.MI_BACKEND_BASE_URL ==~ /^https?:\/\/[A-Za-z0-9.-]+(:[0-9]{1,5})?$/)) {
        error("MI_${suffix}_BACKEND_BASE_URL must be an HTTP(S) origin, such as http://mi-dev:8290, without a trailing slash")
    }
    if (!(settings.APIM_INSECURE_TLS in ['true', 'false']) || !(settings.MI_INSECURE_TLS in ['true', 'false'])) {
        error("TLS bypass settings for ${targetStage} must be true or false")
    }

    def apiCtlEnvironment = "${targetStage}-${env.BUILD_NUMBER}"
    withCredentials([
        usernamePassword(credentialsId: "apim-${targetStage}-credentials", usernameVariable: 'APIM_USERNAME', passwordVariable: 'APIM_PASSWORD'),
        usernamePassword(credentialsId: "mi-${targetStage}-credentials", usernameVariable: 'MI_USERNAME', passwordVariable: 'MI_PASSWORD')
    ]) {
        sshagent(credentials: ["mi-${targetStage}-ssh"]) {
            withEnv(settings.collect { key, value -> "${key}=${value}" } + ["APICTL_ENVIRONMENT=${apiCtlEnvironment}", "CAR_NAME=${CAR_NAME}"]) {
                sh(label: "Deploy MI and APIs to ${targetStage}", script: '''
set +x
set -Eeuo pipefail

APICTL_TLS_ARGS=()
MICTL_TLS_ARGS=()
if [[ "$APIM_INSECURE_TLS" == "true" ]]; then APICTL_TLS_ARGS+=(--insecure); fi
if [[ "$MI_INSECURE_TLS" == "true" ]]; then MICTL_TLS_ARGS+=(--insecure); fi

cleanup_apictl_environment() {
  apictl logout "$APICTL_ENVIRONMENT" >/dev/null 2>&1 || true
  apictl mi logout "$APICTL_ENVIRONMENT" >/dev/null 2>&1 || true
  apictl remove env "$APICTL_ENVIRONMENT" >/dev/null 2>&1 || true
}
trap cleanup_apictl_environment EXIT

REMOTE="${MI_SSH_USER}@${MI_SSH_HOST}"
CAR_FILE="target/${CAR_NAME}.car"
CAR_UPLOAD="${MI_CARBONAPPS_DIR}/.${CAR_NAME}.car.${BUILD_NUMBER}.upload"
SCHEMA_UPLOAD="${MI_DATABASE_DIR}/.accounts-schema.sql.${BUILD_NUMBER}.upload"
SEED_UPLOAD="${MI_DATABASE_DIR}/.accounts-seed.sql.${BUILD_NUMBER}.upload"

for required_file in "$CAR_FILE" db/schema.sql db/seed.sql; do
  if [[ ! -s "$required_file" ]]; then
    echo "Required deployment file is missing or empty: $required_file" >&2
    exit 1
  fi
done

apictl add env "$APICTL_ENVIRONMENT" --apim "$APIM_URL" --mi "$MI_MANAGEMENT_URL"
printf '%s\n' "$APIM_PASSWORD" | apictl "${APICTL_TLS_ARGS[@]}" login "$APICTL_ENVIRONMENT" -u "$APIM_USERNAME" --password-stdin
printf '%s\n' "$MI_PASSWORD" | apictl "${MICTL_TLS_ARGS[@]}" mi login "$APICTL_ENVIRONMENT" -u "$MI_USERNAME" --password-stdin

ssh -p "$MI_SSH_PORT" -o BatchMode=yes -o StrictHostKeyChecking=yes "$REMOTE" "mkdir -p '$MI_CARBONAPPS_DIR' '$MI_DATABASE_DIR'"
scp -P "$MI_SSH_PORT" -o BatchMode=yes -o StrictHostKeyChecking=yes db/schema.sql "${REMOTE}:${SCHEMA_UPLOAD}"
scp -P "$MI_SSH_PORT" -o BatchMode=yes -o StrictHostKeyChecking=yes db/seed.sql "${REMOTE}:${SEED_UPLOAD}"
scp -P "$MI_SSH_PORT" -o BatchMode=yes -o StrictHostKeyChecking=yes "$CAR_FILE" "${REMOTE}:${CAR_UPLOAD}"

# Promote the database scripts first and the CAR last. MI only sees a complete CAR.
ssh -p "$MI_SSH_PORT" -o BatchMode=yes -o StrictHostKeyChecking=yes "$REMOTE" "set -eu; mv -f -- '$SCHEMA_UPLOAD' '$MI_DATABASE_DIR/accounts-schema.sql'; mv -f -- '$SEED_UPLOAD' '$MI_DATABASE_DIR/accounts-seed.sql'; mv -f -- '$CAR_UPLOAD' '$MI_CARBONAPPS_DIR/${CAR_NAME}.car'"

# Wait for MI to report the new composite application before publishing API definitions.
MI_APPS=''
for attempt in $(seq 1 24); do
  if MI_APPS="$(apictl "${MICTL_TLS_ARGS[@]}" mi get composite-apps -e "$APICTL_ENVIRONMENT" 2>&1)" && grep -Fq 'family-bank-assessment' <<< "$MI_APPS"; then
    printf '%s\n' "$MI_APPS"
    break
  fi
  if [[ "$attempt" -eq 24 ]]; then
    printf '%s\n' "$MI_APPS" >&2
    echo 'MI did not report family-bank-assessment after the CAR was copied; API imports were not attempted.' >&2
    exit 1
  fi
  sleep 5
done

mkdir -p target/apictl/runtime-params
make_params() {
  local template="$1"
  local backend_url="$2"
  local output="$3"
  sed -e "s|__APICTL_ENVIRONMENT__|$APICTL_ENVIRONMENT|g" \
      -e "s|__BACKEND_URL__|$backend_url|g" \
      "$template" > "$output"
}
make_params deployment/apim/params/account-balance.yaml "$MI_BACKEND_BASE_URL" target/apictl/runtime-params/account-balance.yaml
make_params deployment/apim/params/loan-eligibility.yaml "$MI_BACKEND_BASE_URL/loans" target/apictl/runtime-params/loan-eligibility.yaml
make_params deployment/apim/params/customer.yaml "$MI_BACKEND_BASE_URL/customers" target/apictl/runtime-params/customer.yaml

import_api() {
  local archive="$1"
  local api_name="$2"
  local params="$3"
  echo "Importing/updating ${api_name}"
  if ! apictl "${APICTL_TLS_ARGS[@]}" import api \
      --file "target/apictl/bundles/${archive}" \
      --environment "$APICTL_ENVIRONMENT" \
      --update \
      --params "$params"; then
    echo "FAILED importing ${api_name}; deployment stopped. Any earlier successful imports in this stage may already be active." >&2
    return 1
  fi
}

import_api AccountBalanceAPi_1.0.0.zip AccountBalanceAPi target/apictl/runtime-params/account-balance.yaml || exit 1
import_api LoanEligibilityApi_1.0.0.zip LoanEligibilityApi target/apictl/runtime-params/loan-eligibility.yaml || exit 1
import_api soapToRestTransformationApi_1.0.0.zip soapToRestTransformationApi target/apictl/runtime-params/customer.yaml || exit 1
''')
            }
        }
    }
}

pipeline {
    agent { label 'wso2-mi' }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'PROMOTE_TO_PROD', defaultValue: false, description: 'After the automatic dev deployment, request approval and deploy the same build to prod.')
    }

    stages {
        stage('Build, package and test MI') {
            steps {
                sh 'mvn -B clean install'
                archiveArtifacts artifacts: 'target/*.car', fingerprint: true
            }
        }

        stage('Build API projects and archives') {
            steps {
                sh '''
set -euo pipefail
mkdir -p target/apictl/projects target/apictl/bundles

init_and_bundle() {
  local project_name="$1"
  local oas_title="$2"
  local api_name="$3"
  local api_context="$4"
  local spec="$5"
  local project="target/apictl/projects/${project_name}-1.0.0"

  apictl init "$project" --oas "$spec" --initial-state=CREATED
  # Keep API Manager's identity stable when an OAS title differs in casing/name.
  sed -i -e "s/^  name: ${oas_title}$/  name: ${api_name}/" -e "s|^  context: /${oas_title}$|  context: ${api_context}|" "$project/api.yaml"
  sed -i -e "s/^name: ${oas_title}$/name: ${api_name}/" "$project/api_meta.yaml"
  grep -Fxq "  name: ${api_name}" "$project/api.yaml"
  grep -Fxq "  context: ${api_context}" "$project/api.yaml"
  grep -Fxq "name: ${api_name}" "$project/api_meta.yaml"

  apictl bundle --source "$project" --destination target/apictl/bundles
}

init_and_bundle AccountBalanceAPi AccountBalanceAPi AccountBalanceAPi /accountbalanceapi src/main/wso2mi/resources/api-definitions/AccountBalanceAPi.yaml
init_and_bundle LoanEligibilityApi LoanEligibilityAPI LoanEligibilityApi /loaneligibilityapi src/main/wso2mi/resources/api-definitions/LoanEligibilityApi.yaml
init_and_bundle soapToRestTransformationApi CustomerAPI soapToRestTransformationApi /soaptoresttransformationapi src/main/wso2mi/resources/api-definitions/soapToRestTransformationApi.yaml

test -s target/apictl/bundles/AccountBalanceAPi_1.0.0.zip
test -s target/apictl/bundles/LoanEligibilityApi_1.0.0.zip
test -s target/apictl/bundles/soapToRestTransformationApi_1.0.0.zip
'''
                archiveArtifacts artifacts: 'target/apictl/bundles/*.zip', fingerprint: true
            }
        }

        stage('Deploy to dev') {
            steps {
                script { deployEnvironment('dev') }
            }
        }

        stage('Promote to prod') {
            when {
                beforeInput true
                expression { params.PROMOTE_TO_PROD }
            }
            input {
                message 'Approve deployment of this build to production?'
                ok 'Deploy to prod'
            }
            steps {
                script { deployEnvironment('prod') }
            }
        }
    }
}