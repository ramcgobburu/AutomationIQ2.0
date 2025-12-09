# AutomateIQ 2.0: High-Level Architecture Diagram

## Architecture Overview

This document provides a detailed visual representation of the AutomateIQ 2.0 multi-tenant automation platform architecture.

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENT LAYER                                     │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                    │
│  │   Web UI     │      │  Mobile App  │      │  API Clients │                    │
│  │  (React)     │      │  (React      │      │  (REST/SDK)  │                    │
│  │              │      │   Native)    │      │              │                    │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘                    │
└─────────┼──────────────────────┼──────────────────────┼────────────────────────────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   Azure Front Door      │
                    │  - Global CDN           │
                    │  - DDoS Protection      │
                    │  - WAF (OWASP Rules)    │
                    │  - SSL Termination      │
                    │  - Geo-Load Balancing   │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Azure API Management   │
                    │  (Premium Tier)         │
                    │  ┌──────────────────┐  │
                    │  │ Tenant Detection │  │
                    │  │ (Header/Subdomain)│ │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure AD B2B     │  │
                    │  │ Authentication   │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Rate Limiting    │  │
                    │  │ (Per Tenant)     │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Request/Response │  │
                    │  │ Transformation   │  │
                    │  └──────────────────┘  │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
┌─────────▼──────────┐  ┌────────▼──────────┐  ┌──────▼──────────┐
│  Workflow          │  │  Tenant           │  │  Integration    │
│  Management        │  │  Management       │  │  Management     │
│  Service           │  │  Service          │  │  Service        │
│  (Container App)   │  │  (Container App)  │  │  (Container App)│
│                    │  │                   │  │                 │
│  - CRUD Workflows  │  │  - Onboarding     │  │  - Connectors   │
│  - Versioning      │  │  - Configuration  │  │  - Credentials   │
│  - Validation      │  │  - Billing        │  │  - Testing       │
└─────────┬──────────┘  └────────┬──────────┘  └──────┬──────────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Workflow Orchestrator  │
                    │  (Container App)        │
                    │  ┌──────────────────┐  │
                    │  │ State Management │  │
                    │  │ - Workflow State │  │
                    │  │ - Step Tracking  │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Execution Logic  │  │
                    │  │ - Retry Logic    │  │
                    │  │ - Error Handling │  │
                    │  │ - Timeout Mgmt   │  │
                    │  └──────────────────┘  │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
┌─────────▼──────────┐  ┌────────▼──────────┐  ┌──────▼──────────┐
│  Execution Worker  │  │  Execution Worker │  │  Execution      │
│  Pool 1            │  │  Pool 2           │  │  Worker Pool N  │
│  (Container Apps)  │  │  (Container Apps) │  │  (Container Apps)│
│  Auto-scale: 0-50  │  │  Auto-scale: 0-50 │  │  Auto-scale:    │
│                    │  │                   │  │  0-50           │
│  - Execute Steps   │  │  - Execute Steps  │  │  - Execute Steps│
│  - Call Integrations│ │  - Call Integrations│ │  - Call Integrations│
│  - Return Results  │  │  - Return Results │  │  - Return Results│
└─────────┬──────────┘  └────────┬──────────┘  └──────┬──────────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   MESSAGING LAYER       │
                    │  ┌──────────────────┐  │
                    │  │ Azure Service Bus│  │
                    │  │ - Workflow Queue │  │
                    │  │ - Priority Queues│  │
                    │  │ - DLQ            │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Event Grid │  │
                    │  │ - Scheduled      │  │
                    │  │ - Webhook        │  │
                    │  │ - Manual Trigger │  │
                    │  └──────────────────┘  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    INTEGRATION LAYER    │
                    │  ┌──────────────────┐  │
                    │  │ Azure Functions  │  │
                    │  │ - REST Connector │  │
                    │  │ - DB Connector   │  │
                    │  │ - Email Connector│  │
                    │  │ - File Connector │  │
                    │  │ - Custom         │  │
                    │  └──────────────────┘  │
                    └─────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      DATA LAYER         │
                    │  ┌──────────────────┐  │
                    │  │ Azure SQL DB     │  │
                    │  │ (Elastic Pool)   │  │
                    │  │ - Tenants        │  │
                    │  │ - Workflows      │  │
                    │  │ - Executions     │  │
                    │  │ - Users          │  │
                    │  │ - RLS Policies  │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Cosmos DB  │  │
                    │  │ (High Volume)    │  │
                    │  │ - Execution Logs │  │
                    │  │ - Event Streams  │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Blob Storage│  │
                    │  │ - Artifacts       │  │
                    │  │ - Reports         │  │
                    │  │ - Archives        │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Redis Cache│  │
                    │  │ - Sessions        │  │
                    │  │ - Workflow Cache │  │
                    │  │ - Rate Limits    │  │
                    │  └──────────────────┘  │
                    └─────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  OBSERVABILITY LAYER   │
                    │  ┌──────────────────┐  │
                    │  │ Application      │  │
                    │  │ Insights (APM)   │  │
                    │  │ - Distributed    │  │
                    │  │   Tracing        │  │
                    │  │ - Performance    │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Log Analytics    │  │
                    │  │ - Centralized    │  │
                    │  │   Logs           │  │
                    │  │ - KQL Queries    │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Monitor   │  │
                    │  │ - Metrics       │  │
                    │  │ - Alerts        │  │
                    │  │ - Dashboards    │  │
                    │  └──────────────────┘  │
                    └─────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   SECURITY LAYER        │
                    │  ┌──────────────────┐  │
                    │  │ Azure AD B2B     │  │
                    │  │ - Authentication │  │
                    │  │ - Authorization  │  │
                    │  │ - SSO            │  │
                    │  └──────────────────┘  │
                    │  ┌──────────────────┐  │
                    │  │ Azure Key Vault  │  │
                    │  │ - Secrets        │  │
                    │  │ - Certificates   │  │
                    │  │ - Keys           │  │
                    │  └──────────────────┘  │
                    └─────────────────────────┘
```

---

## Data Flow Diagrams

### 1. Workflow Creation Flow

```
Client Request
    │
    ▼
Azure Front Door (CDN/WAF)
    │
    ▼
Azure API Management
    │
    ├─► Authenticate (Azure AD B2B)
    ├─► Identify Tenant
    ├─► Rate Limit Check
    │
    ▼
Workflow Management Service
    │
    ├─► Validate Workflow Definition
    ├─► Store in Azure SQL (with tenant_id)
    ├─► Cache in Redis
    │
    ▼
Response to Client
```

### 2. Workflow Execution Flow

```
Trigger Event
    │
    ├─► Scheduled (Cron) ──┐
    ├─► Webhook Event ────┼──► Azure Event Grid
    ├─► Manual Trigger ───┤
    └─► API Call ─────────┘
            │
            ▼
    Azure Event Grid
            │
            ▼
    Workflow Orchestrator
            │
            ├─► Load Workflow Definition (Redis/SQL)
            ├─► Create Execution Record (SQL)
            ├─► Enqueue to Service Bus
            │
            ▼
    Azure Service Bus Queue
            │
            ▼
    Execution Worker (Container App)
            │
            ├─► Dequeue Message
            ├─► Execute Step
            ├─► Call Integration Connector (Azure Function)
            │
            ▼
    Integration Connector
            │
            ├─► REST API Call
            ├─► Database Query
            ├─► Email Send
            └─► File Operation
            │
            ▼
    External System
            │
            ▼
    Results Returned
            │
            ▼
    Execution Worker
            │
            ├─► Update Execution State (SQL)
            ├─► Log Results (Cosmos DB)
            ├─► Notify Orchestrator (Service Bus)
            │
            ▼
    Workflow Orchestrator
            │
            ├─► Check if More Steps
            ├─► If Yes: Enqueue Next Step
            ├─► If No: Mark Complete
            │
            ▼
    Client Notification (Webhook/Status API)
```

### 3. Monitoring & Observability Flow

```
All Services
    │
    ├─► Application Insights (APM)
    │   ├─► Distributed Tracing
    │   ├─► Performance Metrics
    │   └─► Exception Tracking
    │
    ├─► Log Analytics
    │   ├─► Application Logs
    │   ├─► Audit Logs
    │   └─► Security Logs
    │
    └─► Azure Monitor
        ├─► Infrastructure Metrics
        ├─► Custom Metrics
        └─► Health Checks
            │
            ▼
    Azure Dashboards
            │
            ├─► Platform Health Dashboard
            ├─► Tenant-Specific Dashboards
            └─► Cost Analytics Dashboard
            │
            ▼
    Azure Alerts
            │
            ├─► Email Notifications
            ├─► SMS Notifications
            └─► Webhook Notifications
```

---

## Component Details

### 1. Client Layer
- **Web UI**: React-based single-page application
- **Mobile App**: React Native for iOS/Android
- **API Clients**: REST API with SDKs for popular languages

### 2. Edge Layer (Azure Front Door)
- **Global CDN**: Static asset delivery
- **DDoS Protection**: Layer 3/4/7 protection
- **WAF**: OWASP Top 10 protection
- **SSL/TLS**: Certificate management
- **Geo-Load Balancing**: Route to nearest region

### 3. API Gateway (Azure API Management)
- **Tenant Identification**: Via header (X-Tenant-ID) or subdomain
- **Authentication**: Azure AD B2B integration
- **Rate Limiting**: Per-tenant quotas
- **Request Transformation**: API versioning, request/response transformation
- **Developer Portal**: API documentation and testing

### 4. Application Services (Container Apps)
- **Workflow Management**: CRUD operations, versioning, validation
- **Tenant Management**: Onboarding, configuration, billing
- **Integration Management**: Connector management, credential storage
- **Auto-Scaling**: Based on HTTP requests, CPU, memory

### 5. Orchestration Layer
- **Workflow Orchestrator**: Manages workflow state, coordinates step execution
- **Execution Workers**: Stateless workers that execute workflow steps
- **Scaling**: Auto-scale based on Service Bus queue depth

### 6. Messaging Layer
- **Azure Service Bus**: Reliable message delivery, dead-letter queues
- **Azure Event Grid**: Event-driven triggers (scheduled, webhook, manual)

### 7. Data Layer
- **Azure SQL Database**: Transactional data with row-level security
- **Azure Cosmos DB**: High-volume execution logs for large tenants
- **Azure Blob Storage**: Artifacts, reports, archives
- **Azure Redis Cache**: Sessions, workflow cache, rate limiting

### 8. Integration Layer
- **Azure Functions**: Serverless integration connectors
- **Connector Types**: REST, Database, Email, File System, Custom

### 9. Observability Layer
- **Application Insights**: APM with distributed tracing
- **Log Analytics**: Centralized log aggregation
- **Azure Monitor**: Metrics, alerts, dashboards

### 10. Security Layer
- **Azure AD B2B**: Multi-tenant authentication and authorization
- **Azure Key Vault**: Secrets, certificates, keys management

---

## Multi-Tenancy Isolation Strategy

### 1. Data Isolation
- **Row-Level Security (RLS)**: Azure SQL RLS policies filter all queries by tenant_id
- **Tenant Context**: Injected at API Gateway level, propagated via headers/claims
- **Database Schema**: Single schema with tenant_id column in all tables

### 2. Compute Isolation
- **Logical Isolation**: All services are tenant-aware, no physical isolation
- **Resource Quotas**: Per-tenant limits on API calls, workflow executions
- **Priority Queues**: Premium tenants can use dedicated high-priority queues

### 3. Network Isolation
- **Private Endpoints**: Databases and storage accessible only via private network
- **Network Security Groups**: Restrict traffic between services
- **Azure Private Link**: Secure connectivity to Azure services

### 4. Security Isolation
- **Azure AD B2B**: Each tenant has separate Azure AD tenant
- **RBAC**: Role-based access control per tenant
- **Audit Logging**: All actions logged with tenant_id for compliance

---

## High Availability & Disaster Recovery

### Primary Region (US East)
- **Availability Zones**: Services deployed across 3 availability zones
- **Database**: Primary database with read replicas in different zones
- **Load Balancing**: Azure Load Balancer distributes traffic across zones

### Secondary Region (EU West)
- **Warm Standby**: Services scaled to minimum (1-2 instances)
- **Database**: Geo-replicated from primary region
- **Failover**: Automatic failover via Azure SQL Failover Groups (<30s RTO)

### Backup Strategy
- **Database**: Automated backups (full daily, log every 5 minutes), 35-day retention
- **Blob Storage**: Geo-redundant storage (GRS)
- **Key Vault**: Geo-redundant backup

---

## Security Architecture

### Network Security
- **Azure Front Door**: DDoS protection, WAF
- **Private Endpoints**: Databases and storage not exposed to internet
- **Network Security Groups**: Restrict traffic between services
- **Azure Private Link**: Secure connectivity

### Identity & Access
- **Azure AD B2B**: Multi-tenant authentication
- **RBAC**: Role-based access control
- **MFA**: Multi-factor authentication for admin accounts
- **Service Principals**: Managed identities for service-to-service auth

### Data Security
- **Encryption at Rest**: TDE for SQL, SSE for Blob Storage
- **Encryption in Transit**: TLS 1.2+ for all communications
- **Row-Level Security**: Azure SQL RLS policies
- **Data Classification**: Tag sensitive data (PII, PHI)

### Secrets Management
- **Azure Key Vault**: All secrets, certificates, keys
- **Automatic Rotation**: Key rotation policies
- **Access Policies**: Least privilege access
- **Audit Logging**: All Key Vault access logged

---

## Performance Characteristics

### Latency Targets
- **API Response Time**: <2s (p95)
- **Workflow Execution Start**: <5s from trigger
- **Step Execution**: <30s per step (p95)
- **Dashboard Load**: <3s

### Throughput Targets
- **API Requests**: 10,000+ requests/minute (Phase 3)
- **Workflow Executions**: 1,000+ concurrent executions
- **Database Queries**: 50,000+ queries/minute

### Scalability
- **Horizontal Scaling**: Auto-scale from 0 to N instances
- **Database Scaling**: Elastic pools scale from 200 to 4000 eDTUs
- **Storage Scaling**: Unlimited blob storage with tiered access

---

## Cost Optimization

### Resource Sharing
- **Elastic Pools**: Share database resources across tenants (60-70% savings)
- **Container Apps**: Pay only for active instances
- **Reserved Instances**: 40-72% savings for predictable workloads

### Tiered Storage
- **Hot Tier**: Active workflow artifacts (frequent access)
- **Cool Tier**: Recent archives (infrequent access, 30-90 days)
- **Archive Tier**: Long-term retention (rarely accessed, 7 years)

### Auto-Scaling
- **Scale to Zero**: Non-critical services scale to 0 during off-hours
- **Minimum Instances**: Critical services maintain minimum 1-2 instances
- **Spot VMs**: Use spot instances for fault-tolerant workloads (90% savings)

---

This architecture provides a comprehensive, scalable, and secure foundation for AutomateIQ 2.0, enabling the platform to grow from 10 to 1000+ tenants while maintaining strong security, compliance, and cost efficiency.

