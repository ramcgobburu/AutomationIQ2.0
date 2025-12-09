# Application Services Explained
## Workflow Management, Tenant Management, and Integration Management

---

## Overview

The **Application Services** layer consists of three core microservices that handle the main business logic of AutomateIQ 2.0. Each service is deployed as an **Azure Container App** and is responsible for a specific domain.

```
Application Services Layer
    ├─ Workflow Management Service
    ├─ Tenant Management Service
    └─ Integration Management Service
```

---

## 1. Workflow Management Service

### What is It?

The **Workflow Management Service** is responsible for managing the lifecycle of workflows - from creation to execution. Think of it as the **library system** for workflows.

### Real-World Analogy

Imagine a **recipe management system**:
- **CRUD Workflows** = Create, edit, delete recipes
- **Versioning** = Keep track of recipe changes (v1, v2, v3)
- **Validation** = Ensure recipes are complete and correct

### Core Responsibilities

#### 1. CRUD Workflows (Create, Read, Update, Delete)

**What it does**: Handles all operations for managing workflow definitions.

##### **Create Workflow**

**API Endpoint**: `POST /api/workflows`

**Request**:
```json
{
  "name": "Employee Onboarding",
  "description": "Automated onboarding process for new employees",
  "steps": [
    {
      "id": "step1",
      "type": "create-user",
      "config": {
        "firstName": "{{employee.firstName}}",
        "lastName": "{{employee.lastName}}",
        "email": "{{employee.email}}"
      }
    },
    {
      "id": "step2",
      "type": "send-email",
      "config": {
        "to": "{{employee.email}}",
        "subject": "Welcome!",
        "template": "welcome-email"
      }
    }
  ],
  "triggers": [
    {
      "type": "scheduled",
      "cron": "0 9 * * *"
    }
  ]
}
```

**What happens**:
1. Validates workflow structure
2. Stores workflow definition in database
3. Creates workflow metadata (ID, version, tenant, created date)
4. Returns workflow ID

**Response**:
```json
{
  "id": "wf-12345",
  "name": "Employee Onboarding",
  "version": 1,
  "tenantId": "acme-corp",
  "status": "active",
  "createdAt": "2024-01-15T10:00:00Z",
  "createdBy": "user-123"
}
```

##### **Read Workflow**

**API Endpoint**: `GET /api/workflows/{workflowId}`

**What happens**:
1. Retrieves workflow from database
2. Applies tenant isolation (only returns workflows for the requesting tenant)
3. Returns workflow definition

**Response**:
```json
{
  "id": "wf-12345",
  "name": "Employee Onboarding",
  "version": 1,
  "definition": { ... },
  "status": "active",
  "lastModified": "2024-01-15T10:00:00Z"
}
```

##### **Update Workflow**

**API Endpoint**: `PUT /api/workflows/{workflowId}`

**What happens**:
1. Validates new workflow definition
2. Creates new version (versioning)
3. Updates workflow in database
4. Invalidates cache (if workflow was cached)

**Important**: Updates create new versions, old versions are preserved for rollback.

##### **Delete Workflow**

**API Endpoint**: `DELETE /api/workflows/{workflowId}`

**What happens**:
1. Soft delete (marks as deleted, doesn't actually remove)
2. Prevents new executions
3. Keeps execution history for audit
4. Can be restored within 30 days

**Soft Delete Example**:
```json
{
  "id": "wf-12345",
  "status": "deleted",
  "deletedAt": "2024-01-20T10:00:00Z",
  "deletedBy": "user-123"
}
```

#### 2. Versioning

**What it does**: Tracks different versions of workflows, allowing you to see history and rollback if needed.

### Why Versioning is Important

1. **Change Tracking**: See what changed and when
2. **Rollback**: Revert to previous version if new version has issues
3. **Audit**: Compliance requirement (who changed what, when)
4. **Testing**: Test new version before making it active

### How Versioning Works

```
Workflow: "Employee Onboarding"
    ├─ Version 1 (2024-01-15) - Initial version
    ├─ Version 2 (2024-01-20) - Added email step
    ├─ Version 3 (2024-01-25) - Fixed bug in step 2
    └─ Version 4 (2024-01-30) - Current active version
```

### Version Management Operations

##### **List Versions**

**API Endpoint**: `GET /api/workflows/{workflowId}/versions`

**Response**:
```json
{
  "workflowId": "wf-12345",
  "versions": [
    {
      "version": 4,
      "status": "active",
      "createdAt": "2024-01-30T10:00:00Z",
      "createdBy": "user-123",
      "changes": "Added email notification step"
    },
    {
      "version": 3,
      "status": "inactive",
      "createdAt": "2024-01-25T10:00:00Z",
      "createdBy": "user-456",
      "changes": "Fixed bug in step 2"
    },
    {
      "version": 2,
      "status": "inactive",
      "createdAt": "2024-01-20T10:00:00Z",
      "createdBy": "user-123",
      "changes": "Added email step"
    },
    {
      "version": 1,
      "status": "inactive",
      "createdAt": "2024-01-15T10:00:00Z",
      "createdBy": "user-123",
      "changes": "Initial version"
    }
  ]
}
```

##### **Get Specific Version**

**API Endpoint**: `GET /api/workflows/{workflowId}/versions/{version}`

**Response**:
```json
{
  "workflowId": "wf-12345",
  "version": 2,
  "definition": { ... },
  "status": "inactive",
  "createdAt": "2024-01-20T10:00:00Z"
}
```

##### **Rollback to Previous Version**

**API Endpoint**: `POST /api/workflows/{workflowId}/rollback`

**Request**:
```json
{
  "targetVersion": 3
}
```

**What happens**:
1. Retrieves version 3 definition
2. Creates new version (version 5) with version 3's definition
3. Sets version 5 as active
4. Version 4 becomes inactive

#### 3. Validation

**What it does**: Ensures workflows are correctly defined before they can be saved or executed.

### Why Validation is Important

1. **Prevent Errors**: Catch issues before execution
2. **Data Integrity**: Ensure workflow structure is correct
3. **User Experience**: Give immediate feedback on errors
4. **Security**: Prevent malicious workflow definitions

### Types of Validation

##### **1. Structure Validation**

Checks if workflow has required fields:

```json
{
  "name": "Employee Onboarding",  // ✅ Required
  "steps": [ ... ],                // ✅ Required
  "triggers": [ ... ]              // ✅ Required
}
```

**Validation Rules**:
- `name` must be present and non-empty
- `steps` must be an array with at least 1 step
- `triggers` must be an array with at least 1 trigger
- Each step must have `id`, `type`, and `config`

##### **2. Step Validation**

Validates each step in the workflow:

```json
{
  "steps": [
    {
      "id": "step1",                    // ✅ Must be unique
      "type": "create-user",            // ✅ Must be valid step type
      "config": { ... },                // ✅ Must match step type schema
      "onSuccess": "step2",             // ✅ Must reference valid step ID
      "onFailure": "error-handler"      // ✅ Must reference valid step ID
    }
  ]
}
```

**Validation Rules**:
- Step IDs must be unique within workflow
- Step types must be registered (valid connector types)
- Step config must match the schema for that step type
- References (`onSuccess`, `onFailure`) must point to valid step IDs
- No circular references (step1 → step2 → step1)

##### **3. Trigger Validation**

Validates workflow triggers:

```json
{
  "triggers": [
    {
      "type": "scheduled",
      "cron": "0 9 * * *"  // ✅ Must be valid cron expression
    },
    {
      "type": "webhook",
      "path": "/webhooks/onboarding"  // ✅ Must be valid URL path
    }
  ]
}
```

**Validation Rules**:
- Cron expressions must be valid
- Webhook paths must be unique per tenant
- API trigger endpoints must be valid
- Manual triggers don't need validation

##### **4. Tenant Validation**

Ensures workflow belongs to correct tenant:

```json
{
  "tenantId": "acme-corp",  // ✅ Must match requesting tenant
  "createdBy": "user-123"   // ✅ User must belong to tenant
}
```

**Validation Rules**:
- `tenantId` must match the tenant making the request
- `createdBy` user must belong to the tenant
- Workflow can only be accessed by its tenant

### Validation Response

**Valid Workflow**:
```json
{
  "valid": true,
  "workflowId": "wf-12345"
}
```

**Invalid Workflow**:
```json
{
  "valid": false,
  "errors": [
    {
      "field": "steps[0].type",
      "message": "Invalid step type: 'invalid-type'. Valid types are: create-user, send-email, ..."
    },
    {
      "field": "steps[1].onSuccess",
      "message": "Step ID 'step99' does not exist in workflow"
    }
  ]
}
```

### Complete Validation Flow

```
1. User submits workflow
    ↓
2. Structure Validation
   - Required fields present?
   - Data types correct?
    ↓
3. Step Validation
   - Step types valid?
   - Step configs correct?
   - References valid?
    ↓
4. Trigger Validation
   - Triggers valid?
   - Unique paths?
    ↓
5. Tenant Validation
   - Tenant matches?
   - User authorized?
    ↓
6. Save or Return Errors
```

---

## 2. Tenant Management Service

### What is It?

The **Tenant Management Service** handles everything related to managing tenants (customers) on the platform. Think of it as the **customer relationship management (CRM)** system for your SaaS platform.

### Real-World Analogy

Imagine a **hotel management system**:
- **Onboarding** = Checking in new guests
- **Configuration** = Setting room preferences, amenities
- **Billing** = Managing payments, invoices, subscriptions

### Core Responsibilities

#### 1. Onboarding

**What it does**: The process of bringing a new tenant (customer) onto the platform.

### Onboarding Flow

```
1. Sales/Signup Process
    ↓
2. Create Tenant Record
    ↓
3. Configure Tenant Settings
    ↓
4. Set Up Billing
    ↓
5. Provision Resources
    ↓
6. Send Welcome Email
    ↓
7. Tenant Ready!
```

### Onboarding Operations

##### **Create Tenant**

**API Endpoint**: `POST /api/tenants`

**Request**:
```json
{
  "name": "Acme Corporation",
  "domain": "acme-corp",
  "contactEmail": "admin@acme-corp.com",
  "contactName": "John Doe",
  "tier": "premium",
  "region": "asia-pacific"
}
```

**What happens**:
1. **Create Tenant Record**:
   ```json
   {
     "id": "tenant-12345",
     "name": "Acme Corporation",
     "domain": "acme-corp",
     "status": "provisioning",
     "createdAt": "2024-01-15T10:00:00Z"
   }
   ```

2. **Provision Resources**:
   - Create database row-level security policy
   - Allocate API rate limits based on tier
   - Set up storage quotas
   - Configure monitoring dashboards

3. **Set Up Authentication**:
   - Create Azure AD B2B tenant relationship
   - Configure SSO (if enterprise tier)
   - Generate API keys

4. **Send Welcome Email**:
   - Welcome message
   - Getting started guide
   - API credentials
   - Support contact information

**Response**:
```json
{
  "id": "tenant-12345",
  "name": "Acme Corporation",
  "domain": "acme-corp",
  "status": "active",
  "apiKey": "ak_live_abc123...",
  "onboardingComplete": true,
  "nextSteps": [
    "Configure your first workflow",
    "Set up integrations",
    "Invite team members"
  ]
}
```

##### **Onboarding Checklist**

The service tracks onboarding progress:

```json
{
  "tenantId": "tenant-12345",
  "checklist": [
    {
      "step": "create-tenant",
      "status": "completed",
      "completedAt": "2024-01-15T10:00:00Z"
    },
    {
      "step": "provision-resources",
      "status": "completed",
      "completedAt": "2024-01-15T10:05:00Z"
    },
    {
      "step": "setup-authentication",
      "status": "completed",
      "completedAt": "2024-01-15T10:10:00Z"
    },
    {
      "step": "send-welcome-email",
      "status": "completed",
      "completedAt": "2024-01-15T10:15:00Z"
    },
    {
      "step": "first-workflow-created",
      "status": "pending"
    }
  ]
}
```

#### 2. Configuration

**What it does**: Manages tenant-specific settings and preferences.

### Configuration Categories

##### **1. General Settings**

```json
{
  "tenantId": "tenant-12345",
  "settings": {
    "name": "Acme Corporation",
    "timezone": "Asia/Singapore",
    "locale": "en-SG",
    "defaultLanguage": "English"
  }
}
```

##### **2. Feature Flags**

Enable/disable features per tenant:

```json
{
  "tenantId": "tenant-12345",
  "features": {
    "advancedWorkflows": true,
    "customIntegrations": true,
    "apiAccess": true,
    "webhooks": true,
    "scheduledWorkflows": true,
    "mlOptimization": false  // Premium feature
  }
}
```

##### **3. Resource Limits**

Set quotas based on tenant tier:

```json
{
  "tenantId": "tenant-12345",
  "limits": {
    "workflows": {
      "max": 1000,
      "current": 45
    },
    "apiCalls": {
      "perMinute": 1000,
      "perHour": 10000,
      "perDay": 100000
    },
    "storage": {
      "maxGB": 100,
      "currentGB": 12.5
    },
    "users": {
      "max": 50,
      "current": 12
    }
  }
}
```

##### **4. Integration Settings**

Configure default integrations:

```json
{
  "tenantId": "tenant-12345",
  "integrations": {
    "defaultEmailProvider": "sendgrid",
    "defaultDatabase": "postgresql",
    "allowedConnectors": [
      "rest-api",
      "database",
      "email",
      "slack",
      "salesforce"
    ]
  }
}
```

##### **5. Security Settings**

```json
{
  "tenantId": "tenant-12345",
  "security": {
    "requireMFA": true,
    "sessionTimeout": 3600,
    "passwordPolicy": {
      "minLength": 12,
      "requireUppercase": true,
      "requireNumbers": true,
      "requireSpecialChars": true
    },
    "ipWhitelist": [
      "192.168.1.0/24",
      "10.0.0.0/8"
    ]
  }
}
```

### Configuration Operations

##### **Get Configuration**

**API Endpoint**: `GET /api/tenants/{tenantId}/configuration`

**Response**: Returns all configuration settings

##### **Update Configuration**

**API Endpoint**: `PUT /api/tenants/{tenantId}/configuration`

**Request**:
```json
{
  "settings": {
    "timezone": "Asia/Tokyo"
  },
  "features": {
    "mlOptimization": true
  }
}
```

**What happens**:
1. Validates new configuration
2. Updates database
3. Applies changes (e.g., update rate limits)
4. Logs change for audit

##### **Validate Configuration**

**API Endpoint**: `POST /api/tenants/{tenantId}/configuration/validate`

Checks if configuration is valid before applying.

#### 3. Billing

**What it does**: Manages subscription plans, invoices, and payments for tenants.

### Billing Components

##### **1. Subscription Plans**

```json
{
  "plans": [
    {
      "id": "free",
      "name": "Free",
      "price": 0,
      "features": {
        "workflows": 10,
        "apiCalls": 1000,
        "storage": 1
      }
    },
    {
      "id": "basic",
      "name": "Basic",
      "price": 99,
      "billingCycle": "monthly",
      "features": {
        "workflows": 100,
        "apiCalls": 10000,
        "storage": 10
      }
    },
    {
      "id": "premium",
      "name": "Premium",
      "price": 299,
      "billingCycle": "monthly",
      "features": {
        "workflows": 1000,
        "apiCalls": 100000,
        "storage": 100,
        "mlOptimization": true
      }
    },
    {
      "id": "enterprise",
      "name": "Enterprise",
      "price": "custom",
      "features": {
        "workflows": "unlimited",
        "apiCalls": "unlimited",
        "storage": "unlimited",
        "dedicatedSupport": true,
        "sla": "99.99%"
      }
    }
  ]
}
```

##### **2. Tenant Subscription**

```json
{
  "tenantId": "tenant-12345",
  "subscription": {
    "planId": "premium",
    "status": "active",
    "currentPeriodStart": "2024-01-01T00:00:00Z",
    "currentPeriodEnd": "2024-02-01T00:00:00Z",
    "cancelAtPeriodEnd": false,
    "paymentMethod": {
      "type": "credit_card",
      "last4": "4242",
      "brand": "visa"
    }
  }
}
```

##### **3. Usage Tracking**

Tracks actual usage for billing:

```json
{
  "tenantId": "tenant-12345",
  "usage": {
    "period": "2024-01",
    "workflows": {
      "created": 45,
      "executed": 1250,
      "limit": 1000
    },
    "apiCalls": {
      "count": 87500,
      "limit": 100000
    },
    "storage": {
      "gb": 12.5,
      "limit": 100
    }
  }
}
```

##### **4. Invoices**

```json
{
  "invoiceId": "inv-12345",
  "tenantId": "tenant-12345",
  "period": "2024-01",
  "amount": 299.00,
  "currency": "USD",
  "status": "paid",
  "items": [
    {
      "description": "Premium Plan - January 2024",
      "amount": 299.00
    },
    {
      "description": "Overage - API Calls (75,000 extra)",
      "amount": 75.00
    }
  ],
  "total": 374.00,
  "paidAt": "2024-02-01T10:00:00Z"
}
```

### Billing Operations

##### **Subscribe to Plan**

**API Endpoint**: `POST /api/tenants/{tenantId}/subscription`

**Request**:
```json
{
  "planId": "premium",
  "paymentMethodId": "pm_12345"
}
```

**What happens**:
1. Validates plan exists
2. Creates subscription
3. Charges payment method
4. Updates tenant limits
5. Sends confirmation email

##### **Update Subscription**

**API Endpoint**: `PUT /api/tenants/{tenantId}/subscription`

**Upgrade/Downgrade**:
```json
{
  "planId": "enterprise",
  "effectiveDate": "2024-02-01T00:00:00Z"
}
```

##### **Cancel Subscription**

**API Endpoint**: `DELETE /api/tenants/{tenantId}/subscription`

**Request**:
```json
{
  "cancelAtPeriodEnd": true,
  "reason": "Switching to competitor"
}
```

**What happens**:
1. Marks subscription for cancellation
2. Continues service until period end
3. Sends cancellation confirmation
4. Triggers retention workflow (if configured)

##### **Get Usage**

**API Endpoint**: `GET /api/tenants/{tenantId}/usage`

Returns current usage vs. limits.

##### **Get Invoices**

**API Endpoint**: `GET /api/tenants/{tenantId}/invoices`

Returns billing history.

---

## 3. Integration Management Service

### What is It?

The **Integration Management Service** manages connections to external systems (APIs, databases, email services, etc.). Think of it as the **connector library** that enables workflows to interact with external systems.

### Real-World Analogy

Imagine a **universal adapter**:
- **Connectors** = Different plug types (USB, HDMI, etc.)
- **Credentials** = The power/connection details
- **Testing** = Verifying the connection works

### Core Responsibilities

#### 1. Connectors

**What it does**: Manages different types of integration connectors available to tenants.

### Connector Types

##### **1. REST API Connector**

Connects to REST APIs:

```json
{
  "id": "connector-rest-001",
  "type": "rest-api",
  "name": "Salesforce API",
  "description": "Connect to Salesforce REST API",
  "config": {
    "baseUrl": "https://api.salesforce.com",
    "authentication": {
      "type": "oauth2",
      "clientId": "{{clientId}}",
      "clientSecret": "{{clientSecret}}"
    },
    "endpoints": {
      "createLead": "/services/data/v58.0/sobjects/Lead",
      "getAccount": "/services/data/v58.0/sobjects/Account/{id}"
    }
  }
}
```

##### **2. Database Connector**

Connects to databases:

```json
{
  "id": "connector-db-001",
  "type": "database",
  "name": "PostgreSQL Database",
  "description": "Connect to PostgreSQL database",
  "config": {
    "type": "postgresql",
    "host": "{{host}}",
    "port": 5432,
    "database": "{{database}}",
    "ssl": true,
    "connectionPool": {
      "min": 2,
      "max": 10
    }
  }
}
```

##### **3. Email Connector**

Connects to email services:

```json
{
  "id": "connector-email-001",
  "type": "email",
  "name": "SendGrid",
  "description": "Send emails via SendGrid",
  "config": {
    "provider": "sendgrid",
    "apiKey": "{{apiKey}}",
    "fromEmail": "noreply@automateiq.com",
    "fromName": "AutomateIQ"
  }
}
```

##### **4. File System Connector**

Connects to file storage:

```json
{
  "id": "connector-file-001",
  "type": "file-system",
  "name": "Azure Blob Storage",
  "description": "Read/write files in Azure Blob Storage",
  "config": {
    "provider": "azure-blob",
    "accountName": "{{accountName}}",
    "accountKey": "{{accountKey}}",
    "container": "{{container}}"
  }
}
```

##### **5. Custom Connector**

Tenant-specific connectors:

```json
{
  "id": "connector-custom-001",
  "type": "custom",
  "name": "Custom ERP System",
  "description": "Connect to internal ERP system",
  "config": {
    "endpoint": "https://erp.internal.com/api",
    "authentication": {
      "type": "api-key",
      "header": "X-API-Key"
    }
  }
}
```

### Connector Operations

##### **List Available Connectors**

**API Endpoint**: `GET /api/integrations/connectors`

**Response**:
```json
{
  "connectors": [
    {
      "id": "rest-api",
      "name": "REST API",
      "description": "Connect to any REST API",
      "icon": "🌐",
      "category": "api"
    },
    {
      "id": "database",
      "name": "Database",
      "description": "Connect to SQL databases",
      "icon": "🗄️",
      "category": "database"
    },
    {
      "id": "email",
      "name": "Email",
      "description": "Send emails",
      "icon": "📧",
      "category": "communication"
    }
  ]
}
```

##### **Create Connector Instance**

**API Endpoint**: `POST /api/integrations/connectors/{connectorId}/instances`

**Request**:
```json
{
  "name": "My Salesforce Connection",
  "config": {
    "baseUrl": "https://api.salesforce.com",
    "clientId": "abc123",
    "clientSecret": "xyz789"
  }
}
```

**Response**:
```json
{
  "id": "instance-12345",
  "connectorId": "rest-api",
  "name": "My Salesforce Connection",
  "status": "active",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

##### **Update Connector Instance**

**API Endpoint**: `PUT /api/integrations/connectors/instances/{instanceId}`

##### **Delete Connector Instance**

**API Endpoint**: `DELETE /api/integrations/connectors/instances/{instanceId}`

#### 2. Credentials

**What it does**: Securely stores and manages authentication credentials for integrations.

### Why Secure Credential Storage is Critical

1. **Security**: Credentials must be encrypted
2. **Compliance**: SOC 2, GDPR require secure credential storage
3. **Convenience**: Users don't need to enter credentials every time
4. **Rotation**: Support credential rotation

### Credential Storage

##### **Encrypted Storage**

Credentials are stored in **Azure Key Vault**:

```
User enters: password123
    ↓
Encrypted: aGVsbG8gd29ybGQ= (encrypted with tenant key)
    ↓
Stored in: Azure Key Vault
    ↓
When needed: Decrypted on-the-fly for use
```

##### **Credential Types**

**1. API Key**:
```json
{
  "type": "api-key",
  "value": "sk_live_abc123...",
  "header": "X-API-Key"
}
```

**2. Username/Password**:
```json
{
  "type": "basic-auth",
  "username": "user@example.com",
  "password": "encrypted_password"
}
```

**3. OAuth 2.0**:
```json
{
  "type": "oauth2",
  "clientId": "abc123",
  "clientSecret": "encrypted_secret",
  "accessToken": "encrypted_token",
  "refreshToken": "encrypted_refresh_token",
  "expiresAt": "2024-01-16T10:00:00Z"
}
```

**4. Database Connection String**:
```json
{
  "type": "connection-string",
  "value": "encrypted_connection_string"
}
```

### Credential Operations

##### **Store Credentials**

**API Endpoint**: `POST /api/integrations/credentials`

**Request**:
```json
{
  "connectorInstanceId": "instance-12345",
  "type": "oauth2",
  "clientId": "abc123",
  "clientSecret": "xyz789"
}
```

**What happens**:
1. Encrypts credentials using tenant-specific key
2. Stores in Azure Key Vault
3. Returns credential ID (not the actual credentials)

**Response**:
```json
{
  "id": "cred-12345",
  "connectorInstanceId": "instance-12345",
  "type": "oauth2",
  "status": "active",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

##### **Retrieve Credentials** (for use, not display)

**API Endpoint**: `GET /api/integrations/credentials/{credentialId}`

**Response**: Returns encrypted credentials (decrypted on-the-fly for use in workflows)

**Security**: Credentials are never returned in API responses for security reasons.

##### **Rotate Credentials**

**API Endpoint**: `POST /api/integrations/credentials/{credentialId}/rotate`

**Request**:
```json
{
  "newClientSecret": "new_secret_xyz"
}
```

**What happens**:
1. Validates new credentials work
2. Updates encrypted storage
3. Invalidates old credentials
4. Updates all active connections

##### **Delete Credentials**

**API Endpoint**: `DELETE /api/integrations/credentials/{credentialId}`

**What happens**:
1. Removes from Key Vault
2. Invalidates all connections using these credentials
3. Logs deletion for audit

#### 3. Testing

**What it does**: Validates that integrations are configured correctly and can connect to external systems.

### Why Testing is Important

1. **Catch Errors Early**: Find issues before workflows run
2. **User Experience**: Immediate feedback on configuration
3. **Debugging**: Helps troubleshoot connection issues
4. **Confidence**: Users know their integration works

### Testing Operations

##### **Test Connection**

**API Endpoint**: `POST /api/integrations/connectors/instances/{instanceId}/test`

**Request**:
```json
{
  "testType": "connection"
}
```

**What happens**:
1. Retrieves credentials from Key Vault
2. Attempts to connect to external system
3. Performs basic operation (e.g., list records)
4. Returns test results

**Response**:
```json
{
  "success": true,
  "message": "Connection successful",
  "details": {
    "responseTime": "125ms",
    "testedAt": "2024-01-15T10:00:00Z"
  }
}
```

**Or if it fails**:
```json
{
  "success": false,
  "message": "Connection failed: Invalid credentials",
  "error": {
    "code": "AUTH_ERROR",
    "message": "Invalid API key",
    "suggestion": "Please check your API key and try again"
  }
}
```

##### **Test Specific Operation**

**API Endpoint**: `POST /api/integrations/connectors/instances/{instanceId}/test/operation`

**Request**:
```json
{
  "operation": "createLead",
  "testData": {
    "firstName": "Test",
    "lastName": "User",
    "email": "test@example.com"
  }
}
```

**What happens**:
1. Connects to external system
2. Performs the specific operation with test data
3. Validates response
4. Cleans up test data (if possible)

**Response**:
```json
{
  "success": true,
  "message": "Operation successful",
  "result": {
    "leadId": "lead-12345",
    "createdAt": "2024-01-15T10:00:00Z"
  },
  "cleanedUp": true
}
```

##### **Validate Configuration**

**API Endpoint**: `POST /api/integrations/connectors/instances/{instanceId}/validate`

**Request**:
```json
{
  "config": {
    "baseUrl": "https://api.example.com",
    "apiKey": "test-key"
  }
}
```

**Response**:
```json
{
  "valid": true,
  "errors": []
}
```

**Or if invalid**:
```json
{
  "valid": false,
  "errors": [
    {
      "field": "baseUrl",
      "message": "Invalid URL format"
    },
    {
      "field": "apiKey",
      "message": "API key is required"
    }
  ]
}
```

### Testing Flow

```
1. User configures integration
    ↓
2. User clicks "Test Connection"
    ↓
3. Integration Management Service:
   a. Retrieves credentials
   b. Connects to external system
   c. Performs test operation
   d. Validates response
    ↓
4. Returns test results
    ↓
5. If successful: Integration ready to use
   If failed: Show error with suggestions
```

---

## How Services Work Together

### Complete Workflow Creation Flow

```
1. User creates workflow via API
    ↓
2. API Management (tenant detection, auth, rate limiting)
    ↓
3. Workflow Management Service
   - Validates workflow structure
   - Validates step types (checks Integration Management)
   - Saves workflow definition
    ↓
4. Integration Management Service
   - Validates connector instances exist
   - Validates credentials are stored
   - Tests connections (if requested)
    ↓
5. Workflow saved successfully
```

### Tenant Onboarding Flow

```
1. New tenant signs up
    ↓
2. Tenant Management Service
   - Creates tenant record
   - Provisions resources
   - Sets up billing
    ↓
3. Integration Management Service
   - Sets up default connectors
   - Configures default credentials
    ↓
4. Workflow Management Service
   - Creates sample workflows
   - Sets up default templates
    ↓
5. Tenant ready to use platform
```

### Integration Setup Flow

```
1. User wants to connect to Salesforce
    ↓
2. Integration Management Service
   - User selects "Salesforce" connector
   - User enters credentials
   - Service stores credentials securely
   - Service tests connection
    ↓
3. If test successful:
   - Integration marked as active
   - Available for use in workflows
    ↓
4. User creates workflow using Salesforce connector
    ↓
5. Workflow Management Service
   - Validates workflow uses valid connector
   - Saves workflow
```

---

## Summary

| Service | Main Purpose | Key Operations |
|---------|-------------|----------------|
| **Workflow Management** | Manage workflow lifecycle | CRUD workflows, versioning, validation |
| **Tenant Management** | Manage customers | Onboarding, configuration, billing |
| **Integration Management** | Manage external connections | Connectors, credentials, testing |

**Together, these three services form the core business logic layer of AutomateIQ 2.0!** 🚀

