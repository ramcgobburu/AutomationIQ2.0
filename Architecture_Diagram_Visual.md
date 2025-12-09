# AutomateIQ 2.0: Visual Architecture Diagram

## Recommended Tools for Creating Architecture Diagrams

### 1. **Draw.io / diagrams.net** (⭐ Recommended - Free)
- **Website**: https://app.diagrams.net/ or https://www.diagrams.net/
- **Why**: Free, web-based, has Azure icon library, exports to PNG/SVG/PDF
- **Best for**: Professional diagrams with Azure icons
- **How to use**: 
  1. Go to app.diagrams.net
  2. Choose "Create New Diagram" → "Azure" template
  3. Drag and drop Azure icons
  4. Export as PNG/PDF for presentations

### 2. **Lucidchart** (Professional - Paid)
- **Website**: https://www.lucidchart.com/
- **Why**: Professional, has Azure architecture templates, collaboration features
- **Best for**: Team collaboration and professional presentations
- **Cost**: Free trial, then ~$10-20/month

### 3. **Miro** (Collaborative Whiteboard)
- **Website**: https://miro.com/
- **Why**: Great for brainstorming, has Azure templates, real-time collaboration
- **Best for**: Collaborative design sessions
- **Cost**: Free tier available, paid plans for teams

### 4. **Microsoft Visio** (Enterprise)
- **Why**: Official Microsoft tool, excellent Azure integration
- **Best for**: Enterprise environments already using Microsoft tools
- **Cost**: Part of Microsoft 365 subscription

### 5. **Mermaid** (Code-based - Free)
- **Why**: Write diagrams as code, renders in GitHub/Markdown
- **Best for**: Version-controlled diagrams, documentation
- **See below for Mermaid diagram**

---

## Mermaid Architecture Diagram

You can render this directly in GitHub, GitLab, or any Mermaid-compatible viewer:

```mermaid
graph TB
    subgraph "Client Layer"
        WebUI[Web UI<br/>React]
        Mobile[Mobile App<br/>React Native]
        API[API Clients<br/>REST/SDK]
    end

    subgraph "Edge Layer"
        FrontDoor[Azure Front Door<br/>CDN + WAF + DDoS Protection]
    end

    subgraph "API Gateway"
        APIM[Azure API Management<br/>Premium Tier<br/>- Tenant Detection<br/>- Azure AD B2B Auth<br/>- Rate Limiting<br/>- Request Transformation]
    end

    subgraph "Application Services"
        WorkflowMgmt[Workflow Management<br/>Container App<br/>- CRUD Workflows<br/>- Versioning<br/>- Validation]
        TenantMgmt[Tenant Management<br/>Container App<br/>- Onboarding<br/>- Configuration<br/>- Billing]
        IntegrationMgmt[Integration Management<br/>Container App<br/>- Connectors<br/>- Credentials<br/>- Testing]
    end

    subgraph "Orchestration Layer"
        Orchestrator[Workflow Orchestrator<br/>Container App<br/>- State Management<br/>- Retry Logic<br/>- Error Handling<br/>- Timeout Management]
        
        Worker1[Execution Worker Pool 1<br/>Container Apps<br/>Auto-scale: 0-50]
        Worker2[Execution Worker Pool 2<br/>Container Apps<br/>Auto-scale: 0-50]
        WorkerN[Execution Worker Pool N<br/>Container Apps<br/>Auto-scale: 0-50]
    end

    subgraph "Messaging Layer"
        ServiceBus[Azure Service Bus<br/>Premium<br/>- Workflow Queues<br/>- Priority Queues<br/>- Dead Letter Queue]
        EventGrid[Azure Event Grid<br/>- Scheduled Triggers<br/>- Webhook Events<br/>- Manual Triggers]
    end

    subgraph "Integration Layer"
        Functions[Azure Functions<br/>Integration Connectors<br/>- REST API<br/>- Database<br/>- Email<br/>- File System<br/>- Custom]
    end

    subgraph "Data Layer"
        SQLDB[(Azure SQL Database<br/>Elastic Pool<br/>- Tenant Data<br/>- Workflows<br/>- Executions<br/>- Users<br/>- RLS Policies)]
        CosmosDB[(Azure Cosmos DB<br/>High Volume Tenants<br/>- Execution Logs<br/>- Event Streams)]
        BlobStorage[(Azure Blob Storage<br/>- Artifacts<br/>- Reports<br/>- Archives)]
        Redis[(Azure Cache for Redis<br/>- Sessions<br/>- Workflow Cache<br/>- Rate Limits)]
    end

    subgraph "Observability Layer"
        AppInsights[Application Insights<br/>- Distributed Tracing<br/>- Performance Monitoring<br/>- Exception Tracking]
        LogAnalytics[Log Analytics<br/>- Centralized Logs<br/>- KQL Queries]
        Monitor[Azure Monitor<br/>- Metrics<br/>- Alerts<br/>- Dashboards]
    end

    subgraph "Security Layer"
        AzureAD[Azure AD B2B<br/>- Authentication<br/>- Authorization<br/>- SSO]
        KeyVault[Azure Key Vault<br/>- Secrets<br/>- Certificates<br/>- Keys]
    end

    WebUI --> FrontDoor
    Mobile --> FrontDoor
    API --> FrontDoor
    
    FrontDoor --> APIM
    APIM --> WorkflowMgmt
    APIM --> TenantMgmt
    APIM --> IntegrationMgmt
    
    WorkflowMgmt --> Orchestrator
    TenantMgmt --> SQLDB
    IntegrationMgmt --> SQLDB
    
    Orchestrator --> ServiceBus
    EventGrid --> Orchestrator
    ServiceBus --> Worker1
    ServiceBus --> Worker2
    ServiceBus --> WorkerN
    
    Worker1 --> Functions
    Worker2 --> Functions
    WorkerN --> Functions
    
    Functions --> SQLDB
    Functions --> CosmosDB
    Functions --> BlobStorage
    
    Orchestrator --> SQLDB
    Orchestrator --> Redis
    Worker1 --> SQLDB
    Worker2 --> SQLDB
    WorkerN --> SQLDB
    
    WorkflowMgmt --> AppInsights
    TenantMgmt --> AppInsights
    IntegrationMgmt --> AppInsights
    Orchestrator --> AppInsights
    Worker1 --> AppInsights
    Worker2 --> AppInsights
    WorkerN --> AppInsights
    
    AppInsights --> LogAnalytics
    LogAnalytics --> Monitor
    
    APIM --> AzureAD
    WorkflowMgmt --> KeyVault
    TenantMgmt --> KeyVault
    IntegrationMgmt --> KeyVault
    Functions --> KeyVault

    style WebUI fill:#e1f5ff
    style Mobile fill:#e1f5ff
    style API fill:#e1f5ff
    style FrontDoor fill:#0078d4,color:#fff
    style APIM fill:#0078d4,color:#fff
    style WorkflowMgmt fill:#00bcf2
    style TenantMgmt fill:#00bcf2
    style IntegrationMgmt fill:#00bcf2
    style Orchestrator fill:#00bcf2
    style Worker1 fill:#00bcf2
    style Worker2 fill:#00bcf2
    style WorkerN fill:#00bcf2
    style ServiceBus fill:#0078d4,color:#fff
    style EventGrid fill:#0078d4,color:#fff
    style Functions fill:#0078d4,color:#fff
    style SQLDB fill:#ff6900,color:#fff
    style CosmosDB fill:#ff6900,color:#fff
    style BlobStorage fill:#ff6900,color:#fff
    style Redis fill:#ff6900,color:#fff
    style AppInsights fill:#68217a,color:#fff
    style LogAnalytics fill:#68217a,color:#fff
    style Monitor fill:#68217a,color:#fff
    style AzureAD fill:#71afe5
    style KeyVault fill:#71afe5
```

---

## Workflow Execution Flow Diagram

```mermaid
sequenceDiagram
    participant Client
    participant APIM as API Management
    participant Orchestrator
    participant ServiceBus as Service Bus Queue
    participant Worker as Execution Worker
    participant Function as Azure Function<br/>Connector
    participant External as External System
    participant SQLDB as Azure SQL DB
    participant Redis as Redis Cache

    Client->>APIM: Trigger Workflow
    APIM->>APIM: Authenticate & Identify Tenant
    APIM->>Orchestrator: Create Workflow Execution
    
    Orchestrator->>Redis: Check Cache for Workflow Definition
    alt Cache Hit
        Redis-->>Orchestrator: Return Workflow Definition
    else Cache Miss
        Orchestrator->>SQLDB: Load Workflow Definition
        SQLDB-->>Orchestrator: Return Workflow Definition
        Orchestrator->>Redis: Cache Workflow Definition
    end
    
    Orchestrator->>SQLDB: Create Execution Record
    Orchestrator->>ServiceBus: Enqueue Workflow Step
    
    ServiceBus->>Worker: Dequeue Message
    Worker->>Function: Execute Step (REST/DB/Email)
    Function->>External: Call External API/System
    External-->>Function: Return Result
    Function-->>Worker: Return Result
    
    Worker->>SQLDB: Update Execution State
    Worker->>ServiceBus: Enqueue Next Step (if any)
    
    alt More Steps
        ServiceBus->>Worker: Next Step
        Worker->>Function: Execute Next Step
    else Workflow Complete
        Orchestrator->>SQLDB: Mark Execution Complete
        Orchestrator->>Client: Notify Completion (Webhook/Status API)
    end
```

---

## Data Flow Diagram

```mermaid
flowchart LR
    subgraph "Data Sources"
        A[Workflow Executions]
        B[User Actions]
        C[System Events]
    end
    
    subgraph "Collection"
        D[Application Insights]
        E[Log Analytics]
    end
    
    subgraph "Storage"
        F[(Hot Storage<br/>90 days)]
        G[(Warm Storage<br/>1 year)]
        H[(Archive Storage<br/>7 years)]
    end
    
    subgraph "Analytics"
        I[Azure Dashboards]
        J[KQL Queries]
        K[Alerts]
    end
    
    A --> D
    B --> D
    C --> D
    A --> E
    B --> E
    C --> E
    
    D --> F
    E --> F
    
    F -->|After 90 days| G
    G -->|After 1 year| H
    
    F --> I
    F --> J
    F --> K
    
    G --> J
    H --> J
```

---

## How to Use These Diagrams

### Option 1: Draw.io (Recommended for Presentations)
1. Go to https://app.diagrams.net/
2. File → New → Choose "Azure" template
3. Use Azure icon library (More Shapes → Azure)
4. Create your diagram
5. Export as PNG (high resolution) or PDF

### Option 2: Mermaid (For Documentation)
- The Mermaid diagrams above will render automatically in:
  - GitHub/GitLab markdown files
  - VS Code with Mermaid extension
  - Online at https://mermaid.live/
- To export as image: Use https://mermaid.live/ → Export as PNG/SVG

### Option 3: Lucidchart (Professional)
1. Sign up at https://www.lucidchart.com/
2. Create new diagram → Choose "Azure Architecture" template
3. Drag Azure icons from the library
4. Export as PNG/PDF

### Option 4: Use Mermaid Live Editor
1. Go to https://mermaid.live/
2. Copy the Mermaid code from above
3. Paste and render
4. Export as PNG/SVG

---

## Quick Start: Draw.io Azure Architecture

**Step-by-step for Draw.io:**

1. **Open Draw.io**: https://app.diagrams.net/
2. **Choose Template**: File → New → Azure → Azure Architecture
3. **Add Components**:
   - Search for "Azure Front Door" in shapes
   - Search for "API Management"
   - Search for "Container Apps"
   - Search for "SQL Database"
   - Search for "Service Bus"
   - Search for "Functions"
   - Search for "Redis"
   - Search for "Blob Storage"
   - Search for "Application Insights"
   - Search for "Key Vault"
   - Search for "Active Directory"

4. **Arrange Layers**:
   - Top: Clients (Web UI, Mobile, API)
   - Second: Front Door
   - Third: API Management
   - Fourth: Application Services (3 boxes)
   - Fifth: Orchestrator
   - Sixth: Workers (3 boxes)
   - Seventh: Messaging (Service Bus, Event Grid)
   - Eighth: Data Layer (SQL, Cosmos, Blob, Redis)
   - Bottom: Observability & Security

5. **Connect Components**: Use arrows to show data flow
6. **Export**: File → Export as → PNG (300 DPI for presentations)

---

## Tips for Professional Diagrams

1. **Use Consistent Colors**:
   - Azure Blue (#0078d4) for Azure services
   - Light Blue (#e1f5ff) for client applications
   - Orange (#ff6900) for data storage
   - Purple (#68217a) for monitoring

2. **Group Related Components**: Use containers/boxes to group layers

3. **Label Connections**: Add labels to arrows showing data flow direction

4. **Keep It Simple**: Don't overcrowd - use multiple diagrams if needed

5. **Add Legends**: Include a legend explaining colors/symbols

6. **High Resolution**: Export at 300 DPI for presentations

---

## Example: Draw.io File Structure

If you create in Draw.io, save as `.drawio` file and you can:
- Version control it (it's XML)
- Edit anytime
- Export to multiple formats
- Share with team

**Recommended file**: `AutomateIQ_Architecture.drawio`

