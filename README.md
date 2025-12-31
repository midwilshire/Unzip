# Event-driven Unzip Pipeline Architecture

## Overview

This architecture implements an event-driven, serverless file-processing pipeline on Azure. When a compressed file (ZIP / 7Z) is uploaded to Azure Blob Storage, it is automatically processed by an Azure Container Apps Job, extracted, and routed to the appropriate storage containers with full observability and alerting.

This design starts as a POC and scales cleanly into production / enterprise without changing the core pattern.

---

## Objectives

- Automatically unzip files uploaded to Blob Storage
- Event-driven execution (no polling)
- Clear success / failure paths
- Built-in observability and alerting
- Secure, secretless authentication (Managed Identity)

---

## Storage Layer

### Azure Blob Storage Account

**Containers:**
- **`landing/`** - Input container. New ZIP / 7Z files are uploaded here.
- **`processed/`** - Destination for extracted files.
- **`archive/`** - Stores original compressed files after successful processing.
- **`failed/`** - Dead-letter container for files that failed processing.

This layout provides a clear file lifecycle and simplifies auditing, troubleshooting, and reprocessing.

---

## Eventing Layer

### Azure Event Grid (System Topic)

- **Source**: Azure Storage Account
- **Event Type**: `Microsoft.Storage.BlobCreated`
- **Subject Filter**: `/blobServices/default/containers/landing/`

Event Grid emits an event whenever a new blob is created in the landing container.

---

## Compute Layer

### Azure Container Apps Job

- **Trigger Type**: Event
- **Trigger Source**: Event Grid
- **Execution Model**: One job execution per blob
- **Retry Limit**: 1
- **Timeout**: 30 minutes
- **Scaling**: Event-driven (blob-based)

The job runs only when a new file arrives, making it cost-efficient and highly scalable.

---

## Runtime Logic (Container)

### Unzipper Container Responsibilities

1. Receive blob metadata (blob name)
2. Download the archive from `landing/`
3. Detect archive type (ZIP / 7Z)
4. Extract contents
5. Upload extracted files to `processed/`
6. Move original archive:
   - **Success** → `archive/`
   - **Failure** → `failed/`
7. Emit structured logs

### Design Principles

- Fail fast
- Idempotent uploads
- Explicit success and failure paths
- Clear, searchable logs

---

## Identity & Security

### Managed Identity (Recommended)

- **Type**: System-assigned Managed Identity on the Container App Job
- **RBAC Role**: Storage Blob Data Contributor
- **Scope**: Storage Account

**Benefits:**
- No secrets or credentials
- Automatic key rotation
- Auditable access
- Enterprise security standard

---

## Observability Layer

### Azure Monitor & Log Analytics

**Collected telemetry:**
- Job execution status (Succeeded / Failed)
- Execution duration
- Blob name
- Error messages

**Log source:**
- ContainerAppConsoleLogs

---

## Alerting Strategy

### Phase 1 (Immediate)

**Metric Alert:**
- Failed job executions > 0

**Notifications:**
- Email
- Microsoft Teams webhook

### Phase 2 (Enhanced Reliability)

- Alert on blobs present in `failed/` container
- Log Analytics alert for error patterns in logs

---

## End-to-End Flow

```
File Uploaded
     ↓
Azure Blob Storage (landing)
     ↓ BlobCreated Event
Azure Event Grid
     ↓
Azure Container Apps Job
     ↓
Unzipper Container
     ↓
┌───────────────┬───────────────┐
│   Success     │    Failure    │
│               │               │
│ processed/    │ failed/       │
│ archive/      │               │
└───────────────┴───────────────┘
     ↓
Azure Monitor & Alerts
```

---

## Why This Architecture Works

- **Event-driven and serverless**
- **No infrastructure to manage**
- **Scales automatically**
- **Cost-efficient** (pay per execution)
- **Secure by default**
- **Auditable and observable**
- **Easy to automate** with Terraform

---

## Future Enhancements (Optional)

- Virus scanning (Defender for Storage)
- Schema or checksum validation
- Concurrency limits
- Reprocessing workflow (`failed/` → `landing/`)
- Customer-managed encryption keys (CMK)
- SLA dashboards

---

## Summary

This architecture provides a clean, production-ready foundation for file-based batch processing on Azure while remaining simple enough for a POC.

It balances:
- **Reliability**
- **Security**
- **Cost**
- **Operational clarity**

and can evolve without re-architecture.
