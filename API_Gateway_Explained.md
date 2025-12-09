# API Gateway & Azure API Management Explained
## Tenant Detection, Azure AD B2B, Entra ID, Rate Limiting, and Request Transformation

---

## What is an API Gateway?

An **API Gateway** is a single entry point that sits between clients and your backend services. Think of it as a **receptionist** or **concierge** for your APIs.

### Real-World Analogy

Imagine a large office building:
- **Without API Gateway**: Everyone walks directly to different departments (chaos!)
- **With API Gateway**: Everyone goes through reception first (organized!)

The receptionist:
- Checks IDs (authentication)
- Directs people to right department (routing)
- Limits how many people can enter (rate limiting)
- Translates requests (transformation)

### Why Do You Need an API Gateway?

1. **Single Entry Point**: One URL for all your APIs
2. **Security**: Centralized authentication and authorization
3. **Rate Limiting**: Prevents API abuse
4. **Request Transformation**: Modify requests/responses
5. **Monitoring**: Centralized logging and analytics
6. **Versioning**: Manage multiple API versions

---

## Azure API Management (APIM)

**Azure API Management** is Microsoft's fully managed API Gateway service. It's like having a **professional API concierge** that handles everything for you.

### Key Features

1. **API Gateway**: Routes requests to backend services
2. **Developer Portal**: Self-service API documentation
3. **Analytics**: Usage metrics and performance monitoring
4. **Policies**: Rate limiting, transformation, caching
5. **Security**: Authentication, authorization, IP filtering

### How It Works in Your Architecture

```
Client Request
    ↓
Azure API Management
    ├─ Tenant Detection (identifies which tenant)
    ├─ Azure AD B2B Auth (authenticates user)
    ├─ Rate Limiting (checks quota)
    ├─ Request Transformation (modifies request)
    └─ Routing (sends to correct backend)
    ↓
Your Backend Services
```

---

## 1. Tenant Detection

### What is Tenant Detection?

**Tenant Detection** is the process of identifying which tenant (customer) is making the request. This is critical for multi-tenant applications.

### Why Do You Need It?

In a multi-tenant system, you need to know:
- Which tenant's data to access
- Which tenant's quota/limits to apply
- Which tenant's configuration to use

### How Tenant Detection Works

There are several methods to detect tenants:

#### Method 1: **Subdomain-Based** (Recommended)

```
tenant1.automateiq.com → Tenant: tenant1
tenant2.automateiq.com → Tenant: tenant2
acme-corp.automateiq.com → Tenant: acme-corp
```

**How it works**:
1. User accesses `acme-corp.automateiq.com`
2. API Management extracts subdomain: `acme-corp`
3. Maps subdomain to tenant ID
4. Adds `X-Tenant-ID: acme-corp` header to request

**Example**:
```
Request: GET https://acme-corp.automateiq.com/api/workflows
    ↓
API Management extracts: acme-corp
    ↓
Adds header: X-Tenant-ID: acme-corp
    ↓
Backend receives: GET /api/workflows
                  Headers: X-Tenant-ID: acme-corp
```

#### Method 2: **Header-Based**

```
Request includes: X-Tenant-ID: acme-corp
```

**How it works**:
1. Client includes `X-Tenant-ID` header in request
2. API Management validates tenant exists
3. Forwards header to backend

**Example**:
```
Request: GET https://api.automateiq.com/api/workflows
         Headers: X-Tenant-ID: acme-corp
    ↓
API Management validates: acme-corp exists
    ↓
Backend receives: GET /api/workflows
                  Headers: X-Tenant-ID: acme-corp
```

#### Method 3: **Path-Based**

```
/api/tenants/acme-corp/workflows
/api/tenants/acme-corp/users
```

**How it works**:
1. Tenant ID in URL path
2. API Management extracts from path
3. Rewrites path and adds header

**Example**:
```
Request: GET /api/tenants/acme-corp/workflows
    ↓
API Management extracts: acme-corp
    ↓
Rewrites to: GET /api/workflows
             Headers: X-Tenant-ID: acme-corp
```

### Implementation in Azure API Management

**Policy Example** (Subdomain-based):

```xml
<policies>
    <inbound>
        <!-- Extract tenant from subdomain -->
        <set-variable name="tenantId" value="@(context.Request.OriginalUrl.Host.Split('.')[0])" />
        
        <!-- Validate tenant exists -->
        <send-request mode="new" response-variable-name="tenantCheck" timeout="5" ignore-error="true">
            <set-url>https://tenant-service.azurewebsites.net/api/tenants/validate</set-url>
            <set-method>POST</set-method>
            <set-body>@{
                return new JObject(
                    new JProperty("tenantId", (string)context.Variables["tenantId"])
                ).ToString();
            }</set-body>
        </send-request>
        
        <!-- Add tenant ID to header -->
        <set-header name="X-Tenant-ID" exists-action="override">
            <value>@((string)context.Variables["tenantId"])</value>
        </set-header>
        
        <!-- Block if tenant invalid -->
        <choose>
            <when condition="@(((IResponse)context.Variables["tenantCheck"]).StatusCode != 200)">
                <return-response>
                    <set-status code="403" reason="Invalid Tenant" />
                    <set-body>{"error": "Invalid tenant"}</set-body>
                </return-response>
            </when>
        </choose>
    </inbound>
</policies>
```

### Tenant Detection Flow

```
1. Request arrives at API Management
    ↓
2. Extract tenant identifier
   - From subdomain: acme-corp.automateiq.com → acme-corp
   - OR from header: X-Tenant-ID: acme-corp
   - OR from path: /api/tenants/acme-corp/...
    ↓
3. Validate tenant exists and is active
    ↓
4. Add X-Tenant-ID header to request
    ↓
5. Forward to backend with tenant context
```

---

## 2. Azure AD B2B & Entra ID

### What is Entra ID? (Formerly Azure AD)

**Microsoft Entra ID** (formerly Azure Active Directory) is Microsoft's cloud-based identity and access management service. It's like a **digital ID card system** for your organization.

**Name Change**: Azure AD → Microsoft Entra ID (2023)
- Same service, new name
- "Entra" means "enter" in Latin
- Better reflects its identity management purpose

### What is Azure AD B2B?

**Azure AD B2B** (Business-to-Business) allows you to invite users from **other organizations** to access your applications. They use their own organization's credentials.

### Real-World Analogy

**Traditional (B2C)**:
- Everyone gets a keycard from your company
- You manage all credentials

**B2B**:
- Partner companies use their own keycards
- They authenticate with their own system
- You trust their authentication

### How B2B Works

```
External User (from Partner Company)
    ↓
Uses their own company's credentials
    ↓
Azure AD B2B validates with their company
    ↓
Grants access to your application
    ↓
User accesses AutomateIQ
```

### Example Scenario

**Company A (AutomateIQ)** invites **Company B (Client)**:

1. **Company A** sends invitation to `john@companyb.com`
2. **John** receives email invitation
3. **John** clicks link, signs in with Company B credentials
4. **Azure AD B2B** validates with Company B's identity provider
5. **John** gets access to AutomateIQ with appropriate permissions

### Benefits for Multi-Tenant Platform

1. **No Password Management**: Clients use their own credentials
2. **SSO Support**: Clients can use their existing SSO
3. **Security**: Leverages client's security policies
4. **Compliance**: Easier compliance (clients control their users)
5. **User Experience**: Users don't need new passwords

### Authentication Flow

```
1. User accesses API
    ↓
2. API Management redirects to Azure AD B2B
    ↓
3. User signs in with their organization credentials
    ↓
4. Azure AD B2B validates credentials
    ↓
5. Azure AD B2B issues token (JWT)
    ↓
6. User sends token to API Management
    ↓
7. API Management validates token
    ↓
8. Extracts user info (email, tenant, roles)
    ↓
9. Adds headers: X-User-ID, X-Tenant-ID, X-Roles
    ↓
10. Forwards request to backend
```

### Token Structure (JWT)

```json
{
  "aud": "api://automateiq",
  "iss": "https://sts.windows.net/tenant-id/",
  "sub": "user-id",
  "email": "john@companyb.com",
  "name": "John Doe",
  "roles": ["WorkflowAdmin", "User"],
  "tenant_id": "companyb-tenant-id",
  "exp": 1234567890
}
```

### Implementation in API Management

**Validate JWT Token Policy**:

```xml
<policies>
    <inbound>
        <!-- Validate JWT token -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized">
            <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud" match="any">
                    <value>api://automateiq</value>
                </claim>
            </required-claims>
        </validate-jwt>
        
        <!-- Extract user info from token -->
        <set-variable name="userId" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("sub", ""))" />
        <set-variable name="userEmail" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("email", ""))" />
        <set-variable name="userRoles" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("roles", ""))" />
        
        <!-- Add user info to headers -->
        <set-header name="X-User-ID" exists-action="override">
            <value>@((string)context.Variables["userId"])</value>
        </set-header>
        <set-header name="X-User-Email" exists-action="override">
            <value>@((string)context.Variables["userEmail"])</value>
        </set-header>
        <set-header name="X-User-Roles" exists-action="override">
            <value>@((string)context.Variables["userRoles"])</value>
        </set-header>
    </inbound>
</policies>
```

---

## 3. Rate Limiting

### What is Rate Limiting?

**Rate Limiting** restricts how many requests a user or tenant can make within a time period. It prevents API abuse and ensures fair usage.

### Why Do You Need It?

1. **Prevent Abuse**: Stop malicious users from overwhelming your API
2. **Fair Usage**: Ensure all tenants get fair access
3. **Cost Control**: Prevent unexpected costs from high usage
4. **Stability**: Protect backend services from overload

### Real-World Analogy

Think of rate limiting like a **speed limit**:
- **No limit**: Everyone drives as fast as they want (chaos!)
- **With limit**: Everyone follows speed limit (orderly)

Or like a **restaurant reservation system**:
- Only X customers per hour
- Prevents overcrowding
- Ensures good service for everyone

### Types of Rate Limiting

#### 1. **Per-User Rate Limiting**

```
User A: 100 requests/minute
User B: 100 requests/minute
User C: 100 requests/minute
```

#### 2. **Per-Tenant Rate Limiting**

```
Tenant A (Basic): 1,000 requests/minute
Tenant B (Premium): 10,000 requests/minute
Tenant C (Enterprise): 100,000 requests/minute
```

#### 3. **Per-API Rate Limiting**

```
/workflows API: 100 requests/minute
/users API: 50 requests/minute
/integrations API: 200 requests/minute
```

#### 4. **Global Rate Limiting**

```
All requests combined: 1,000,000 requests/minute
```

### How Rate Limiting Works

```
Request arrives
    ↓
Check rate limit counter
    ↓
Is limit exceeded?
    ├─ Yes → Return 429 Too Many Requests
    └─ No → Increment counter, allow request
```

### Example Scenarios

#### Scenario 1: User Exceeds Limit

```
10:00:00 - Request 1 ✅ (Count: 1/100)
10:00:05 - Request 2 ✅ (Count: 2/100)
10:00:10 - Request 3 ✅ (Count: 3/100)
...
10:00:55 - Request 100 ✅ (Count: 100/100)
10:01:00 - Request 101 ❌ (429 Too Many Requests)
10:01:05 - Request 102 ❌ (429 Too Many Requests)
10:02:00 - Request 103 ✅ (Count resets, 1/100)
```

#### Scenario 2: Different Limits per Tenant

```
Basic Tenant (acme-corp):
- Limit: 1,000 requests/hour
- Current: 950 requests
- Next request: ✅ Allowed

Premium Tenant (enterprise-inc):
- Limit: 10,000 requests/hour
- Current: 9,500 requests
- Next request: ✅ Allowed
```

### Implementation in API Management

**Rate Limit Policy** (Per Tenant):

```xml
<policies>
    <inbound>
        <!-- Rate limit by tenant -->
        <rate-limit-by-key 
            calls="1000" 
            renewal-period="3600" 
            counter-key="@(context.Request.Headers.GetValueOrDefault("X-Tenant-ID", ""))"
            increment-condition="@(context.Response.StatusCode == 200)"
            />
    </inbound>
    <outbound>
        <!-- Add rate limit headers -->
        <set-header name="X-RateLimit-Limit" exists-action="override">
            <value>1000</value>
        </set-header>
        <set-header name="X-RateLimit-Remaining" exists-action="override">
            <value>@(context.Variables.GetValueOrDefault<int>("rate-limit-remaining", 0))</value>
        </set-header>
        <set-header name="X-RateLimit-Reset" exists-action="override">
            <value>@(DateTime.UtcNow.AddSeconds(3600).ToString("R"))</value>
        </set-header>
    </outbound>
    <on-error>
        <!-- Handle rate limit exceeded -->
        <choose>
            <when condition="@(context.LastError.Message.Contains("rate limit"))">
                <return-response>
                    <set-status code="429" reason="Too Many Requests" />
                    <set-body>@{
                        return new JObject(
                            new JProperty("error", "Rate limit exceeded"),
                            new JProperty("message", "You have exceeded your rate limit. Please try again later."),
                            new JProperty("retryAfter", 3600)
                        ).ToString();
                    }</set-body>
                </return-response>
            </when>
        </choose>
    </on-error>
</policies>
```

### Rate Limit Headers

When a request is made, API Management includes headers:

```
X-RateLimit-Limit: 1000        (Total allowed)
X-RateLimit-Remaining: 750     (Remaining requests)
X-RateLimit-Reset: 3600        (Seconds until reset)
```

### Rate Limit Tiers (Example)

| Tenant Tier | Requests/Minute | Requests/Hour | Requests/Day |
|------------|----------------|---------------|--------------|
| **Free** | 10 | 100 | 1,000 |
| **Basic** | 100 | 1,000 | 10,000 |
| **Premium** | 1,000 | 10,000 | 100,000 |
| **Enterprise** | 10,000 | 100,000 | 1,000,000 |

---

## 4. Request Transformation

### What is Request Transformation?

**Request Transformation** modifies incoming requests before sending them to the backend. It can change URLs, headers, body, query parameters, etc.

### Why Do You Need It?

1. **API Versioning**: Route to different backend versions
2. **Backend Compatibility**: Adapt client requests to backend format
3. **Security**: Remove sensitive headers
4. **Routing**: Route based on request content
5. **Data Format**: Convert between JSON, XML, etc.

### Real-World Analogy

Think of request transformation like a **translator**:
- Client speaks English (API v1 format)
- Backend speaks Spanish (API v2 format)
- Translator converts between them

### Types of Transformations

#### 1. **URL Rewriting**

```
Client Request: GET /api/v1/workflows
    ↓
Transform to: GET /api/v2/workflows
    ↓
Backend receives: GET /api/v2/workflows
```

#### 2. **Header Transformation**

```
Client Request:
  Headers: X-API-Key: abc123
    ↓
Transform:
  Remove: X-API-Key
  Add: Authorization: Bearer token-from-key
    ↓
Backend receives:
  Headers: Authorization: Bearer token-from-key
```

#### 3. **Body Transformation**

```
Client Request:
  Body: {"workflowName": "My Workflow"}
    ↓
Transform:
  Body: {"name": "My Workflow", "version": 1}
    ↓
Backend receives:
  Body: {"name": "My Workflow", "version": 1}
```

#### 4. **Query Parameter Transformation**

```
Client Request: GET /api/workflows?status=active
    ↓
Transform: GET /api/workflows?filter=status:active
    ↓
Backend receives: GET /api/workflows?filter=status:active
```

### Example Transformations

#### Example 1: API Versioning

**Client sends**:
```
GET /api/v1/workflows/123
Headers: X-Tenant-ID: acme-corp
```

**Transform to**:
```
GET /api/workflows/123
Headers: 
  X-Tenant-ID: acme-corp
  X-API-Version: v1
```

**Policy**:
```xml
<set-backend-service base-url="https://backend-v1.azurewebsites.net" />
<rewrite-uri template="/api/workflows/{id}" />
<set-header name="X-API-Version" exists-action="override">
    <value>v1</value>
</set-header>
```

#### Example 2: Add Tenant Context

**Client sends**:
```
POST /api/workflows
Body: {"name": "My Workflow"}
```

**Transform to**:
```
POST /api/workflows
Headers: X-Tenant-ID: acme-corp
Body: {
  "name": "My Workflow",
  "tenantId": "acme-corp",
  "createdBy": "user-123"
}
```

**Policy**:
```xml
<set-body>@{
    var body = context.Request.Body.As<JObject>();
    body["tenantId"] = context.Request.Headers.GetValueOrDefault("X-Tenant-ID", "");
    body["createdBy"] = context.Request.Headers.GetValueOrDefault("X-User-ID", "");
    return body.ToString();
}</set-body>
```

#### Example 3: Convert Format

**Client sends** (JSON):
```json
{
  "workflowName": "My Workflow",
  "steps": ["step1", "step2"]
}
```

**Transform to** (XML for backend):
```xml
<workflow>
  <name>My Workflow</name>
  <steps>
    <step>step1</step>
    <step>step2</step>
  </steps>
</workflow>
```

**Policy**:
```xml
<set-body>@{
    var json = context.Request.Body.As<JObject>();
    var xml = new XElement("workflow",
        new XElement("name", json["workflowName"]),
        new XElement("steps",
            json["steps"].Select(s => new XElement("step", s))
        )
    );
    return xml.ToString();
}</set-body>
```

### Complete Transformation Example

**Incoming Request**:
```
POST https://acme-corp.automateiq.com/api/v1/workflows
Headers:
  Authorization: Bearer token123
  Content-Type: application/json
Body:
{
  "workflowName": "Employee Onboarding",
  "steps": ["create-account", "send-email"]
}
```

**After Transformation**:
```
POST https://backend.azurewebsites.net/api/workflows
Headers:
  X-Tenant-ID: acme-corp
  X-User-ID: user-123
  X-API-Version: v1
  Authorization: Bearer backend-token
Body:
{
  "name": "Employee Onboarding",
  "tenantId": "acme-corp",
  "version": 1,
  "steps": [
    {"id": "create-account", "order": 1},
    {"id": "send-email", "order": 2}
  ],
  "createdBy": "user-123",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

**Policy**:
```xml
<policies>
    <inbound>
        <!-- Extract tenant from subdomain -->
        <set-variable name="tenantId" value="@(context.Request.OriginalUrl.Host.Split('.')[0])" />
        
        <!-- Authenticate and get user info -->
        <validate-jwt header-name="Authorization" />
        <set-variable name="userId" value="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("sub", ""))" />
        
        <!-- Rewrite URL -->
        <rewrite-uri template="/api/workflows" />
        
        <!-- Transform headers -->
        <set-header name="X-Tenant-ID" exists-action="override">
            <value>@((string)context.Variables["tenantId"])</value>
        </set-header>
        <set-header name="X-User-ID" exists-action="override">
            <value>@((string)context.Variables["userId"])</value>
        </set-header>
        
        <!-- Transform body -->
        <set-body>@{
            var requestBody = context.Request.Body.As<JObject>();
            var transformedBody = new JObject(
                new JProperty("name", requestBody["workflowName"]),
                new JProperty("tenantId", (string)context.Variables["tenantId"]),
                new JProperty("version", 1),
                new JProperty("steps", new JArray(
                    requestBody["steps"].Select((step, index) => new JObject(
                        new JProperty("id", step.ToString()),
                        new JProperty("order", index + 1)
                    ))
                )),
                new JProperty("createdBy", (string)context.Variables["userId"]),
                new JProperty("createdAt", DateTime.UtcNow.ToString("o"))
            );
            return transformedBody.ToString();
        }</set-body>
        
        <!-- Set backend URL -->
        <set-backend-service base-url="https://backend.azurewebsites.net" />
    </inbound>
</policies>
```

---

## Complete Flow in Your Architecture

### End-to-End Request Flow

```
1. Client Request
   POST https://acme-corp.automateiq.com/api/v1/workflows
   Headers: Authorization: Bearer token123
   Body: {"workflowName": "My Workflow"}
    ↓
2. Azure Front Door
   - CDN check
   - DDoS protection
   - WAF check
    ↓
3. Azure API Management
    ↓
   a. Tenant Detection
      Extract: acme-corp from subdomain
      Validate: tenant exists and active
      Add header: X-Tenant-ID: acme-corp
    ↓
   b. Azure AD B2B Authentication
      Validate JWT token
      Extract: user-id, email, roles
      Add headers: X-User-ID, X-User-Email, X-User-Roles
    ↓
   c. Rate Limiting
      Check: acme-corp has 750/1000 requests remaining
      Increment counter
      Add headers: X-RateLimit-Remaining: 749
    ↓
   d. Request Transformation
      Rewrite URL: /api/v1/workflows → /api/workflows
      Transform body: Add tenantId, createdBy, etc.
      Add headers: X-API-Version: v1
    ↓
4. Backend Service Receives
   POST /api/workflows
   Headers:
     X-Tenant-ID: acme-corp
     X-User-ID: user-123
     X-User-Email: john@acme-corp.com
     X-User-Roles: WorkflowAdmin,User
     X-API-Version: v1
   Body:
     {
       "name": "My Workflow",
       "tenantId": "acme-corp",
       "createdBy": "user-123",
       ...
     }
    ↓
5. Backend Processes Request
   - Uses X-Tenant-ID for data isolation
   - Uses X-User-ID for audit logging
   - Validates user roles
   - Creates workflow
    ↓
6. Response Transformation (if needed)
    ↓
7. Response Sent Back to Client
```

---

## Summary Table

| Feature | What It Does | Why You Need It |
|---------|-------------|----------------|
| **API Gateway** | Single entry point for all APIs | Centralized management, security, monitoring |
| **Azure API Management** | Microsoft's managed API Gateway | Fully managed, enterprise features |
| **Tenant Detection** | Identifies which tenant made request | Multi-tenant data isolation |
| **Azure AD B2B** | External user authentication | Clients use their own credentials |
| **Entra ID** | Identity and access management | User authentication and authorization |
| **Rate Limiting** | Limits requests per time period | Prevents abuse, ensures fair usage |
| **Request Transformation** | Modifies requests before backend | API versioning, format conversion |

---

## Key Takeaways

1. **API Gateway** = Single entry point for all your APIs
2. **Tenant Detection** = Identifies which customer is making the request
3. **Azure AD B2B** = Allows external users to use their own credentials
4. **Rate Limiting** = Prevents API abuse and ensures fair usage
5. **Request Transformation** = Adapts client requests to backend format

**Together, these features create a secure, scalable, multi-tenant API platform!** 🚀

