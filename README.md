# FastShip — Azure Cloud & DevOps Engineering Project

An event-driven invoice processing platform built with **Azure Functions, .NET, Azure Storage, Event Grid, Terraform, Managed Identity, Azure RBAC, GitHub Actions, OIDC, Application Insights, Azure Monitor, and OpenTelemetry**.

FastShip demonstrates how to **provision, secure, deploy, monitor, troubleshoot, and recover** a cloud-native Azure workload using Infrastructure as Code and modern DevOps practices.

---

## Project Overview

FastShip automatically processes invoice files uploaded to a private Azure Blob Storage container.

The application follows an event-driven architecture:

```text
Invoice Upload
      │
      ▼
Azure Blob Storage
      │
      │ BlobCreated Event
      ▼
Azure Event Grid
      │
      ▼
Azure Function
BlobProcessor
      │
      ▼
Invoice Processing
      │
      ├───────────────┐
      ▼               ▼
ProcessedInvoices   Failure / Retry
Table Storage          │
                       ▼
                InvoiceDeadLetters
                   Table Storage
```

The platform also includes:

* Infrastructure as Code with Terraform
* Managed Identity and Azure RBAC
* GitHub Actions CI/CD
* GitHub → Azure OIDC authentication
* Application Insights and Azure Monitor
* OpenTelemetry
* Health checks
* Idempotency protection
* Retry and dead-letter handling
* Terraform remote state
* Disaster-recovery engineering

---

## Architecture

### Application Architecture

```text
                         Invoice
                            │
                            ▼
                  ┌───────────────────┐
                  │  Azure Blob       │
                  │  Storage          │
                  │  invoices         │
                  │  container        │
                  └─────────┬─────────┘
                            │
                     BlobCreated Event
                            │
                            ▼
                  ┌───────────────────┐
                  │   Azure Event     │
                  │      Grid         │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   BlobProcessor   │
                  │  Azure Function   │
                  └─────────┬─────────┘
                            │
                            ▼
                    InvoiceProcessor
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
       ┌─────────────────┐     ┌──────────────────┐
       │ ProcessedInvoices│     │ Failure / Retry │
       │  Table Storage   │     │                  │
       └─────────────────┘     └────────┬─────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │ InvoiceDeadLetters  │
                              │    Table Storage    │
                              └─────────────────────┘
```

### DevOps Architecture

```text
Developer
   │
   ▼
GitHub Repository
   │
   ├──────── Pull Request ────────┐
   │                              ▼
   │                       GitHub Actions CI
   │                       Build + Validation
   │
   └──────── Merge to main ──────►
                              GitHub Actions CD
                                      │
                                      │ OIDC
                                      ▼
                               Microsoft Entra ID
                                      │
                                      ▼
                                 Azure RBAC
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                    Terraform                Function App
                  Infrastructure               Deployment
                         │                         │
                         └────────────┬────────────┘
                                      ▼
                                    Azure
```

---

## Technology Stack

| Area                      | Technology           |
| ------------------------- | -------------------- |
| Cloud Platform            | Microsoft Azure      |
| Application               | .NET / C#            |
| Compute                   | Azure Functions      |
| Hosting                   | Flex Consumption     |
| Runtime                   | .NET Isolated Worker |
| Operating System          | Linux                |
| Object Storage            | Azure Blob Storage   |
| Processing State          | Azure Table Storage  |
| Eventing                  | Azure Event Grid     |
| Identity                  | Managed Identity     |
| Authorization             | Azure RBAC           |
| Observability             | OpenTelemetry        |
| Application Monitoring    | Application Insights |
| Infrastructure Monitoring | Azure Monitor        |
| Dashboard                 | Azure Workbook       |
| Infrastructure as Code    | Terraform            |
| Terraform Providers       | AzureRM + AzAPI      |
| Terraform State           | Azure Blob Storage   |
| Source Control            | Git / GitHub         |
| CI/CD                     | GitHub Actions       |
| Cloud Authentication      | Azure OIDC           |

---

# Key Engineering Practices

## Infrastructure as Code

FastShip infrastructure is provisioned and managed with **Terraform** rather than relying exclusively on manual Azure Portal configuration.

Terraform manages resources including:

```text
Resource Group
│
├── Storage Account
│   ├── invoices container
│   ├── Function package container
│   ├── ProcessedInvoices table
│   └── InvoiceDeadLetters table
│
├── Flex Consumption Service Plan
│
├── Azure Function App
│   └── System-assigned Managed Identity
│
├── RBAC Role Assignments
│
├── Application Insights
│
├── Azure Monitor Action Group
│
├── Processing Failure Alert
│
├── Operations Workbook
│
└── Event Grid Subscription
```

This allows infrastructure configuration to be version-controlled, reviewed, and reproduced from code.

---

## Secure Authentication with Managed Identity

The Azure Function uses a **system-assigned Managed Identity** instead of storing an Azure Storage account key in application configuration.

Required Azure RBAC roles include:

```text
Storage Blob Data Owner
Storage Queue Data Contributor
Storage Table Data Contributor
```

Identity-based configuration is used for Azure Functions runtime storage:

```text
AzureWebJobsStorage__accountName
AzureWebJobsStorage__credential
```

This removes the need to maintain a long-lived storage account key for the Function App.

---

## GitHub Actions → Azure OIDC

GitHub Actions authenticates to Azure using **OpenID Connect (OIDC)**.

```text
GitHub Actions
      │
      │ OIDC Token
      ▼
Microsoft Entra ID
      │
      ▼
Azure RBAC
      │
      ▼
Azure Resources
```

This avoids storing a long-lived Azure client secret for deployment authentication.

The deployment identity is assigned the required Azure permissions rather than relying on unnecessary subscription-wide credentials.

---

# CI/CD Pipeline

## Continuous Integration

Pull requests trigger validation before changes are merged.

```text
Pull Request
      │
      ▼
Checkout Repository
      │
      ▼
Restore Dependencies
      │
      ▼
Build .NET Application
      │
      ▼
Terraform Setup
      │
      ▼
Terraform Validation
      │
      ▼
Terraform Plan
```

This provides an automated quality gate before infrastructure and application changes progress further.

---

## Continuous Deployment

Changes merged into `main` trigger the deployment workflow.

```text
Push / Merge to main
        │
        ▼
Checkout Repository
        │
        ▼
Restore Dependencies
        │
        ▼
Build Application
        │
        ▼
Publish Application
        │
        ▼
Azure OIDC Login
        │
        ▼
Verify Azure Access
        │
        ▼
Terraform Init
        │
        ▼
Terraform Plan
        │
        ▼
Terraform Apply
        │
        ▼
Deploy Azure Function
        │
        ▼
Health Check
```

The deployment pipeline validates the deployed application through:

```text
/api/HealthCheck
```

A successful package deployment alone is therefore not treated as proof that the application is healthy.

---

# Application Design

## Event-Driven Processing

The application does not continuously poll Azure Storage for new invoices.

Instead:

```text
Blob Upload
    ↓
Event Grid Event
    ↓
Azure Function
    ↓
Invoice Processing
    ↓
Processing State
```

Azure Event Grid provides the event-driven connection between Blob Storage and the Azure Function.

---

## Idempotency

Event-driven systems may receive duplicate events.

FastShip maintains processing state in:

```text
ProcessedInvoices
```

This allows the application to identify previously processed work and reduce the risk of duplicate invoice processing.

---

## Retry and Failure Handling

Transient failures are expected in distributed cloud applications.

FastShip includes retry-aware processing and a separate failure path for work that cannot be completed successfully.

```text
Invoice Processing
       │
       ├── Success ──► ProcessedInvoices
       │
       └── Failure
             │
             ▼
           Retry
             │
             ├── Success
             │
             └── Persistent Failure
                    │
                    ▼
             Dead-Letter Record
```

---

## Dead-Letter Recovery

Failed processing is handled through:

```text
PoisonBlobProcessor
```

Failed operations are persisted to:

```text
InvoiceDeadLetters
```

This provides a durable record for investigation and recovery instead of silently losing failed processing operations.

---

# Observability

FastShip uses structured application telemetry and Azure-native monitoring.

```text
OpenTelemetry
      │
      ▼
Application Insights
      │
      ▼
Azure Monitor
```

The application records processing information such as:

```text
ProcessingStatus
```

This allows operational queries to distinguish successful and failed processing events.

---

# Azure Monitor & Alerting

Azure Monitor is used to detect invoice-processing failures.

Example KQL:

```kusto
traces
| extend ProcessingStatus = tostring(customDimensions["ProcessingStatus"])
| where ProcessingStatus == "Failed"
```

Monitoring includes:

* Application Insights
* Azure Monitor
* Log Analytics integration
* Scheduled query alert
* Action Group
* Operations Workbook

---

# Operations Dashboard

An Azure Workbook provides an operational view of the development environment.

The dashboard includes:

* Total requests
* Failed requests
* Exceptions
* Request activity
* Invoice processing status
* Dead-letter activity

The monitoring dashboard is managed through Terraform rather than relying exclusively on manual Azure Portal configuration.

---

# Health Checks

The application provides an HTTP health endpoint:

```text
/api/HealthCheck
```

The health endpoint is used to verify that the deployed Function App is responding successfully.

It is also used as a post-deployment validation step in the CI/CD process.

---

# Troubleshooting Experience

One of the practical troubleshooting issues encountered during development involved Azure Functions Storage authentication.

The Function App had been configured for Managed Identity:

```text
AzureWebJobsStorage__accountName
AzureWebJobsStorage__credential
```

However, an older:

```text
AzureWebJobsStorage
```

connection-string configuration was still present.

This caused the runtime to attempt the wrong authentication method.

Removing the legacy configuration restored Managed Identity authentication.

### Lesson

When migrating an Azure Function from connection-string authentication to Managed Identity, legacy storage connection-string configuration should be removed rather than left alongside the identity-based configuration.

---

# Runtime Storage vs Business Storage

FastShip separates Azure Functions runtime storage from application business-data access.

### Function Runtime Storage

```text
AzureWebJobsStorage__accountName
AzureWebJobsStorage__credential
```

### Business Storage

```text
BusinessTableEndpoint=https://<storage-account>.table.core.windows.net
```

This separation makes the application configuration easier to understand, troubleshoot, and maintain.

---

# Storage Security

The project applies several storage security controls:

* Private storage containers
* Blob public access disabled
* HTTPS-only communication
* TLS 1.2 minimum where configured
* Managed Identity authentication
* Azure RBAC authorization
* No storage account keys committed to source control

Sensitive credentials and configuration values are kept outside the repository.

---

# Terraform Remote State

Terraform state is stored remotely in Azure Blob Storage.

```text
Terraform
    │
    ▼
Azure Storage Account
    │
    ▼
Private tfstate Container
    │
    ▼
fastship-dev.tfstate
```

The Terraform backend is maintained separately from the application resource group.

A recovery script is also included:

```text
scripts/bootstrap-terraform-backend.sh
```

The purpose is to provide a way to recreate the basic remote-state infrastructure during disaster recovery.

> Terraform state can contain sensitive infrastructure information and must not be committed to source control.

---

# Disaster Recovery Engineering

A disaster-recovery exercise was performed to test an important infrastructure question:

> Can the FastShip environment be rebuilt from code without depending on undocumented manual Azure Portal configuration?

The exercise identified several dependencies and recovery gaps, including:

* Function package container
* Monitoring resources
* Terraform backend
* Event Grid / Function host key dependency

These gaps were addressed by improving the Terraform configuration and recovery process.

---

## Two-Phase Event Grid Recovery

A newly recreated Azure Function generates a new `blobs_extension` system key.

Event Grid requires this key for the Function webhook.

This creates a dependency:

```text
Function App
     │
     ▼
Function Host Initializes
     │
     ▼
New blobs_extension Key
     │
     ▼
Event Grid Subscription
```

To handle this dependency, the recovery process uses two phases.

### Phase 1 — Core Infrastructure

Event Grid creation is temporarily disabled:

```text
create_eventgrid_subscription = false
```

Terraform creates the core infrastructure:

```text
Resource Group
Storage
Tables
Function Package Container
Service Plan
Function App
Managed Identity
RBAC
Application Insights
Monitoring
Workbook
```

### Phase 2 — Event Grid

After the Function application is deployed:

```text
Deploy Function Code
        │
        ▼
Function Host Initializes
        │
        ▼
Retrieve New blobs_extension Key
        │
        ▼
Create Event Grid Subscription
```

This removes the Function-key dependency from the initial infrastructure creation phase.

---

# Disaster Recovery Test

During the recovery exercise, the development resource group was deliberately deleted.

Terraform subsequently generated a Phase 1 recovery plan:

```text
Plan: 15 to add, 0 to change, 0 to destroy.
```

The plan was reviewed before being applied.

Terraform successfully recreated the core infrastructure:

```text
Apply complete! Resources: 15 added, 0 changed, 0 destroyed.
```

This demonstrated that the core FastShip Azure infrastructure could be reconstructed from Terraform after deletion.

### Current DR Status

The original FastShip application and CI/CD implementation were completed before the disaster-recovery exercise.

The DR exercise is a separate ongoing engineering exercise covering:

```text
Infrastructure Audit
        ↓
Recovery Gaps Identified
        ↓
Terraform Improvements
        ↓
Backend Recovery Script
        ↓
Two-Phase Event Grid Recovery
        ↓
Core Infrastructure Rebuild
        ↓
Application Recovery
        ↓
End-to-End Recovery Validation
```

---

# Environment Configuration

FastShip separates environment configuration from application code.

The development environment uses:

```text
APP_ENVIRONMENT=Development
```

The configuration approach is designed to support separate environments such as:

```text
Development
Staging
Production
```

without coupling environment-specific values directly to the application source code.

---

# Repository Structure

```text
blobprocessor/
│
├── BlobProcessor.cs
├── HealthCheck.cs
├── PoisonBlobProcessor.cs
├── Program.cs
├── host.json
├── blobprocessor.csproj
│
├── Models/
│
├── Services/
│
├── infra/
│   └── terraform/
│       ├── main.tf
│       ├── providers.tf
│       ├── variables.tf
│       └── terraform.tfvars
│
├── scripts/
│   └── bootstrap-terraform-backend.sh
│
└── .github/
    └── workflows/
        ├── ci.yml
        └── cd.yml
```

> Sensitive local configuration and Terraform state files are intentionally excluded from source control.

---

# Key Engineering Lessons

### 1. Identity configuration must match runtime behavior

Assigning a Managed Identity is not sufficient by itself. Application configuration must cause the runtime to actually use the identity.

### 2. Remove legacy authentication configuration

Old connection strings can interfere with identity-based authentication.

### 3. Separate runtime and business storage

Azure Functions runtime storage and application business storage should be treated as separate concerns.

### 4. Build observability into the application

Logs, structured telemetry, health checks, dashboards, and alerts should be part of the architecture.

### 5. Design for duplicate events

Event-driven systems should not assume exactly-once delivery. Idempotency is therefore an important application concern.

### 6. Make failed work recoverable

Retries help with transient failures, while durable dead-letter records provide information for investigating failures that cannot be resolved automatically.

### 7. Terraform must represent the real environment

Infrastructure is not fully reproducible if important live resources exist only because somebody created them manually.

### 8. Protect Terraform state

Remote Terraform state is part of the infrastructure-management system and requires its own security and recovery strategy.

### 9. Prefer OIDC over long-lived deployment secrets

GitHub Actions can authenticate to Azure using federated identity rather than storing a reusable Azure client secret.

### 10. Disaster recovery exposes hidden dependencies

The Event Grid / Function `blobs_extension` dependency became apparent through an actual rebuild exercise and demonstrated the value of testing infrastructure recovery rather than assuming it works.

---

# What This Project Demonstrates

This project demonstrates hands-on experience with:

```text
Azure Functions
.NET / C#
Azure Blob Storage
Azure Table Storage
Azure Event Grid
Managed Identity
Azure RBAC
Application Insights
Azure Monitor
Log Analytics
OpenTelemetry
Azure Workbooks
Terraform
Terraform Remote State
Infrastructure as Code
Git
GitHub
GitHub Actions
Azure OIDC
CI/CD
Idempotency
Retry Handling
Dead-Letter Recovery
Monitoring & Alerting
Health Checks
Troubleshooting
Disaster Recovery
```

---

# Project Status

| Area                                | Status      |
| ----------------------------------- | ----------- |
| Azure Functions application         | Complete    |
| Event-driven invoice processing     | Complete    |
| Managed Identity                    | Complete    |
| Azure RBAC                          | Complete    |
| Runtime/business storage separation | Complete    |
| Observability                       | Complete    |
| Monitoring and alerts               | Complete    |
| Operations dashboard                | Complete    |
| Idempotency                         | Complete    |
| Retry handling                      | Complete    |
| Dead-letter recovery                | Complete    |
| Storage hardening                   | Complete    |
| Terraform Infrastructure as Code    | Complete    |
| Terraform remote state              | Complete    |
| GitHub OIDC                         | Complete    |
| CI workflow                         | Complete    |
| CD workflow                         | Complete    |
| End-to-end CI/CD validation         | Complete    |
| Disaster-recovery exercise          | In progress |

---

# Skills Demonstrated

**Azure Cloud:**
Azure Functions, Blob Storage, Table Storage, Event Grid, Application Insights, Azure Monitor, Log Analytics, Azure Workbooks

**DevOps:**
Terraform, Infrastructure as Code, GitHub Actions, CI/CD, OIDC, deployment validation, remote state

**Cloud Security:**
Managed Identity, Azure RBAC, identity-based authentication, secure storage configuration

**Reliability:**
Idempotency, retries, dead-letter handling, health checks, monitoring, alerting, disaster recovery

**Engineering:**
.NET/C#, event-driven architecture, troubleshooting, infrastructure recovery, operational observability

---

# Author

**Mojeed Tijani**

Cloud Engineer (Azure )

### Certifications

* AZ-104 — Microsoft Azure Administrator
* KCNA — Kubernetes and Cloud Native Associate
* FinOps Certified Engineer
