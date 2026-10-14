# Jamii Bank Integration APIs

This project packages three WSO2 Micro Integrator (MI) APIs:

- Account Balance API — Retrieves account balances using an H2 data service.
- Loan Eligibility API — Orchestrates loan eligibility processing.
- Customer API — Provides a REST facade for an existing SOAP customer service.

The Jenkins pipeline builds the MI Composite Application (CAR), bundles all three OpenAPI definitions, and deploys the APIs to WSO2 MI and API Manager.

## Architecture and Trade-offs

The solution separates integration logic from external API management:

- WSO2 Micro Integrator (MI):
  - Handles request routing and validation.
  - Performs data transformations.
  - Orchestrates calls between backend services.
  - Exposes the integration endpoints.

- WSO2 API Manager (APIM):
  - Publishes and governs external API contracts.
  - Provides API lifecycle management and gateway capabilities.
  - Supports API security and traffic governance.

- H2 Database:
  - Uses a file-backed database for account data.
  - Includes checked-in schema and seed SQL scripts.
  - Simplifies initialization in a clean assessment environment.
  - Is intended for local demonstration, not production or highly available storage.

- Deployment pipeline:
  - Builds the MI CAR artifact.
  - Packages the three OpenAPI definitions.
  - Imports API archives individually.
  - Stops and fails when a deployment step encounters an error.

  Deployment consideration: API Manager does not make the three API imports transactional. If an import fails, APIs imported successfully in earlier steps may already be live.

## Assumptions

- A WSO2 MI runtime compatible with this project is available.
- The API Manager version is compatible with the configured `apictl` version and deployment commands.
- The seeded H2 data is intended for local demonstration and assessment purposes.
- A compatible API Manager instance is required to test the complete API publishing and gateway deployment flow.
- The local MI environment and any required backend services are started separately.

## Run Locally

Using docker compose 

docker compose up --build
docker compose logs -f


### 5. Deploy Through API Manager

To test the complete API management flow:

- Start a compatible WSO2 API Manager instance.
- Configure the backend URLs and API Manager deployment targets.
- Configure the required Jenkins environment variables and credentials.
- Review the deployment guide for API archive creation, backend configuration, and deployment settings.
- Run the root `Jenkinsfile` to execute the deployment pipeline.

The pipeline can target a disposable local environment or a CI environment. Provisioning and starting the environment are separate responsibilities.

## With More Time

The following improvements would strengthen the solution:

- Automated testing
  - Add representative Synapse mediation tests.
  - Add API contract validation and regression tests.

- Reproducible environments
  - Provide a Docker Compose environment for MI, API Manager, and supporting services.
  - Automate database initialization and service startup.

- Database improvements
  - Replace H2 with a production-grade database for production deployments.
  - Introduce versioned database migrations.

- Deployment reliability
  - Add deployment health checks.
  - Implement rollback or staged-promotion strategies to handle partial API imports.
  - Verify that APIs are available and healthy before marking a deployment successful.

- Add Real time analytics for API traffic and metrics using ELK stack



## Subscription on APIM 

- First step is to set up the APIs on APIM publisher and then expose them for consumption through APIM developer portal
- Onboarded user can access the APis based on their categories and roles on developer portal , create applications and generate client credentials for Oauth2 authentication to the APis
- I did not include an API product for now beacuse each API is offering a unique set of operations.

