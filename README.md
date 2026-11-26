# Azure-DevOps-CI-CD-Implementation-for-Azure-Data-Factory

This repository demonstrates how to validate, package, and deploy Azure Data Factory (ADF) resources using Azure DevOps YAML pipelines and ARM templates. It includes a sample ADF data pipeline and a reusable deployment flow for DEV, QA, and Production environments.

## Overview

The project combines ADF resource definitions with an automated CI/CD pipeline. On changes to the `main` branch, Azure DevOps validates the ADF project, generates an ARM template, publishes it as a pipeline artifact, and deploys it to the configured environments.

The sample data pipeline reads delimited data from an HTTP source and copies it to Azure Data Lake Storage Gen2. Its Lookup and ForEach activities use configuration data to process multiple source and sink entries.

## Architecture

The solution has two connected workflows:

### Data workflow

1. A Lookup activity reads copy parameters from the configured ADLS dataset.
2. A ForEach activity processes each parameter entry.
3. A Copy activity reads delimited data from an HTTP endpoint.
4. The data is written to ADLS Gen2 as delimited text.

### CI/CD workflow

1. Changes to `main` trigger the Azure DevOps pipeline.
2. The build stage validates the ADF configuration and exports an ARM template.
3. The generated template is published as the `ARMTemplate` pipeline artifact.
4. Deployment stages use environment-specific parameter files to deploy the template.
5. The Production stage runs after QA and targets the Azure DevOps `pd` environment, where approval checks can be configured.

## Key Features

- YAML-based Azure DevOps CI/CD pipeline
- ADF validation and ARM template export using Microsoft’s ADF utilities
- Shared deployment template for multiple environments
- Separate ARM parameter files for DEV, QA, and Production
- Pipeline artifact publishing and reuse during deployment
- Trigger stop/start handling around ARM deployments
- Incremental ARM deployment mode
- Parameterized HTTP-to-ADLS copy pipeline with Lookup and ForEach processing
- Azure Key Vault, ADLS Gen2, and HTTP linked service definitions

## Project Components

### Pipelines

- `pipeline/pipeline.json` — Looks up copy parameters, loops over the entries, and copies delimited data from HTTP to ADLS Gen2.
- `pipeline/pipeline1.json` — A simple one-second wait pipeline.

### Datasets

- `dataset/ds_params.json` — Dataset used by the Lookup activity to read copy parameters.
- `dataset/ds_source.json` — Parameterized delimited-text source dataset.
- `dataset/ds_sink.json` — Parameterized delimited-text sink dataset.

### Linked Services

- `linkedService/ls_http.json` — HTTP source connection.
- `linkedService/ls_adls.json` — ADLS Gen2 connection.
- `linkedService/AzureKeyVault1.json` — Azure Key Vault connection.

### Trigger

- `trigger/trigger_daily.json` — Daily schedule trigger definition. It is currently stopped and has no pipeline reference configured.

### CI/CD

- `cicd/cicd_pipeline.yml` — Main pipeline, triggered by changes to `main`.
- `cicd/ci_build.yml` — Installs Node.js and the ADF utilities, validates the project, exports the ARM template, and publishes the artifact.
- `cicd/cd_deploy.yml` — Downloads the ARM artifact, handles triggers, and deploys the ARM template.
- `cicd/ARMParams/` — Environment-specific ARM deployment parameters for DEV, QA, and Production.

## Deployment Flow

The main pipeline contains the following stages:

- **Build ARM Template** — Validates the ADF project and publishes the generated ARM template.
- **Deploy to DEV** — Deploys using the `devgroup` variable group and `dev.json`.
- **Deploy to QA** — Deploys using the `qagroup` variable group and `qa.json`.
- **Deploy to Production** — Runs after successful QA deployment using the `prodgroup` variable group and `pd.json`.

DEV and QA both depend on the build stage, so they may run independently of one another. Production depends on QA.

Before deployment, the deployment template runs the ADF pre-deployment script to stop triggers. After deployment, it runs the post-deployment script to start triggers and remove deployment resources as configured. ARM deployments use incremental mode.

## Tools and Technologies

- Azure Data Factory
- Azure DevOps Pipelines
- Azure Resource Manager (ARM) templates
- Azure Data Lake Storage Gen2
- Azure Key Vault
- Node.js 20
- `@microsoft/azure-data-factory-utilities`

## Repository Structure

```text
.
├── cicd/
│   ├── ARMParams/
│   │   ├── dev.json
│   │   ├── qa.json
│   │   └── pd.json
│   ├── cd_deploy.yml
│   ├── ci_build.yml
│   └── cicd_pipeline.yml
├── dataset/
├── factory/
├── linkedService/
├── pipeline/
├── trigger/
├── package.json
├── publish_config.json
└── README.md
```

## Azure DevOps Configuration

Before running the pipeline, configure the following in Azure DevOps:

1. Create variable groups named `devgroup`, `qagroup`, and `prodgroup`.
2. Provide the required values in each group:
   - `DataFactory`
   - `ResourceGroup`
   - `SubscriptionId`
3. Create an Azure Resource Manager service connection named `adf-azure-service-connection`.
4. Grant the service connection permissions to deploy ARM templates and manage ADF triggers in the target resource groups.
5. Create an Azure DevOps environment named `pd` and configure approval checks if production approval is required.
6. Ensure the environment parameter files contain the correct resource names and endpoints for your Azure resources.

## Local Dependency Installation

Install the Node.js dependency used by the ADF build utilities:

```bash
npm install
```

The pipeline uses the package script to run the Microsoft ADF utilities for validation and ARM template export.

## Configuration and Security Notes

- The ARM parameter files contain environment-specific resource names and endpoints. Update them to match your Azure setup before deployment.
- Keep credentials and secrets out of the repository. Store sensitive values in Azure DevOps secret variables or Azure Key Vault.
- The deployment YAML specifies `canadacentral` as the ARM deployment location; change it if your deployment requires a different location.
- The `pd` environment provides a place to configure production checks. Configure approvers in Azure DevOps to enforce an approval gate.
- The daily trigger definition is stopped and currently has no pipeline association. Configure and enable it in ADF if you want it to run.
