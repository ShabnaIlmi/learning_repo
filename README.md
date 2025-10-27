# Sales Summary API - Sequence Diagram

## Direct Sales Summary Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant Auth as Auth Middleware
    participant Controller as Sales Summary Controller
    participant TokenService as Token Service
    participant RoleDB as employee_restaurant_roles DB
    participant RestaurantDB as restaurants DB
    participant SalesService as Sales Summary Service
    participant OrdersDB as orders_reports DB
    participant ETLService as ETL Log Service

    Client->>API: GET /api/v1/reports/sales-summary?date_from=X&date_to=Y
    Note over Client,API: Authorization: Bearer <JWT_TOKEN>
    
    API->>Auth: Extract & Verify JWT Token
    Auth->>TokenService: verifyTokenAndDecode(token)
    TokenService-->>Auth: decoded { empId, empResRoleId, resId }
    
    Auth->>RoleDB: SELECT * FROM employee_restaurant_roles WHERE id = empResRoleId
    RoleDB-->>Auth: { id, empId, resId, role, status, products }
    
    Auth->>Auth: Validate role status = ACTIVE
    Auth->>Auth: Validate products includes BACK_OFFICE
    
    alt Role is Valid
        Auth->>Auth: Set req.user = { empId, empResRoleId, restaurantId: resId }
        Auth->>Controller: next()
        
        Controller->>Controller: Extract restaurantId from req.user
        Controller->>Controller: Extract query params (date_from, date_to, limit, skip)
        
        Controller->>SalesService: groupAndSumPaymentsById(restaurantId, date_from, date_to)
        SalesService->>RestaurantDB: SELECT timezone FROM restaurants WHERE id = restaurantId
        RestaurantDB-->>SalesService: { timezone: "+08:00" }
        
        SalesService->>SalesService: Convert dates to UTC using restaurant timezone
        SalesService->>OrdersDB: SELECT * FROM orders_reports WHERE resId = restaurantId AND completedAt BETWEEN date_from AND date_to
        OrdersDB-->>SalesService: [ { orderId, grossSales, discounts, refunds, netSales, ... } ]
        
        SalesService->>SalesService: GROUP BY DATE(completedAt)
        SalesService->>SalesService: SUM(grossSales, discounts, refunds, netSales)
        SalesService-->>Controller: { rows: [...], count: [...] }
        
        Controller->>SalesService: getTotalSales(restaurantId, date_from, date_to)
        SalesService->>OrdersDB: SELECT SUM(grossSales), SUM(discounts), SUM(refunds), SUM(netSales)
        OrdersDB-->>SalesService: { totalGrossSales, totalDiscounts, totalRefunds, totalNetSales }
        SalesService-->>Controller: totalSalesDetails
        
        Controller->>ETLService: findLastUpdated(PAYMENTS_REPORT)
        ETLService-->>Controller: { completedAt: "2025-10-21T08:09:05.000Z" }
        
        Controller->>Controller: Build response with results, extras, totals
        Controller-->>API: 200 OK { results: [...], extras: {...} }
        API-->>Client: Sales Summary Data
        
    else Role is Invalid
        Auth-->>API: 401 Unauthorized
        API-->>Client: Authentication Error
    end
```

## Key Components

### 1. Authentication Flow
- JWT token extracted from Authorization header
- Token decoded to get `empResRoleId`
- Database queried to validate role status and permissions
- `restaurantId` extracted from role record

### 2. Sales Data Query
- Restaurant timezone fetched for date conversion
- Dates converted from local time to UTC
- Orders filtered by `restaurantId` and date range
- Data grouped by date and aggregated (SUM)

### 3. Response Structure
```json
{
  "results": [
    {
      "Date": "18-12-2023",
      "grossSales": "42185600",
      "discounts": "0",
      "refunds": "0",
      "netSales": "42185600"
    }
  ],
  "extras": {
    "date_from": "2023-12-18T00:00:00.000Z",
    "date_to": "2023-12-18T23:59:59.000Z",
    "lastUpdated": "2025-10-21T08:09:05.000Z",
    "total": 1,
    "totalSalesDetails": [
      {
        "totalGrossSales": "42185600",
        "totalDiscounts": "0",
        "totalRefunds": "0",
        "totalNetSales": "42185600"
      }
    ]
  }
}
```

## Database Tables

### employee_restaurant_roles
- `id` (empResRoleId) - Primary key
- `empId` - Foreign key to employees
- `resId` - Foreign key to restaurants
- `role` - User role (manager, cashier, etc.)
- `status` - ACTIVE/INACTIVE
- `products` - Comma-separated product access

### orders_reports
- `orderId` - Order identifier
- `resId` - Restaurant ID
- `completedAt` - Order completion timestamp
- `grossSales` - Total sales before discounts
- `discounts` - Total discounts applied
- `refunds` - Total refunds issued
- `netSales` - Final sales amount

### restaurants
- `id` - Restaurant ID
- `name` - Restaurant name
- `timezone` - Restaurant timezone offset
- `paymentProvider` - Payment provider (ADYEN, etc.)

```mermaid
sequenceDiagram
    participant User
    participant WebUI as Backoffice-Web-Latest
    participant API as Backoffice-API
    participant Intent as Intent Classification Service
    participant AI as AI Service

    User->>WebUI: "What is the longest river in the world?"
    WebUI->>API: POST /api/v1/ai-chat/stream
    Note over API: Extract JWT token<br/>Get restaurantId
    
    API->>Intent: classifyIntent(message)
    Intent->>Intent: Analyze: "What is the longest river..."
    Intent->>Intent: No match for SALES_SUMMARY
    Intent-->>API: { intent: "UNKNOWN", confidence: 0.1 }
    
    alt Intent is NOT SALES_SUMMARY
        API->>API: Reject query
        Note over API: Generate error response
        
        API->>AI: streamChatCompletion(errorContext)
        AI-->>API: Stream: "I can only help with sales-related queries"
        API-->>WebUI: Stream error message
        WebUI-->>User: Display: "I can only help with sales-related queries..."
    end
```

```mermaid
sequenceDiagram
    participant API as Backoffice-API
    participant Location as Location Validation Service
    participant AI as AI Service

    API->>Location: validateLocation(message, restaurantId)
    Location->>Location: Check if query mentions<br/>other restaurant
    
    alt Mentions Other Restaurant
        Location-->>API: { isValid: false }
        API->>AI: Generate error response
        AI-->>API: "Access denied"
    else No Other Restaurant Mentioned
        Location-->>API: { isValid: true }
        API->>API: Continue processing
    end
```
```mermaid
sequenceDiagram
    participant API as Backoffice-API
    participant Guardrails as Guardrails Service
    participant DB as Database
    participant AI as AI Service

    API->>Guardrails: validateQuery(message, restaurantId, userId)
    
    Guardrails->>Guardrails: Check 1: Restaurant Name Validation
    Guardrails->>Guardrails: Check 2: Restaurant ID Validation
    Guardrails->>Guardrails: Check 3: Location/Address Validation
    Guardrails->>DB: Verify user has access to restaurantId
    DB-->>Guardrails: Access permissions
    
    alt Any Violation Detected
        Guardrails->>Guardrails: Log security event
        Guardrails-->>API: { allowed: false, violation: "UNAUTHORIZED_RESTAURANT" }
        API->>AI: Generate security error
        AI-->>API: "Access denied"
    else All Checks Pass
        Guardrails-->>API: { allowed: true }
        API->>API: Continue processing
    end
```
```mermaid
sequenceDiagram
    participant User
    participant WebUI as Backoffice-Web-Latest
    participant API as Backoffice-API<br/>(AI Chat Controller)
    participant Auth as JWT Middleware
    participant Intent as Intent Classification
    participant Guardrails as Guardrails Service<br/>(Planned)
    participant Location as Location Validation
    participant Extract as Multi-Period Extraction
    participant MCP as MCP Service
    participant SalesAPI as Sales Summary API
    participant DB as Database
    participant AI as AI Service
    participant Logger as Security Logger

    Note over User: User logged in as<br/>Restaurant: "Burger King" (ID: 5678)
    
    User->>WebUI: Types: "Show me sales for Pizza Palace"
    WebUI->>WebUI: Capture user input
    
    WebUI->>API: POST /api/v1/ai-chat/stream<br/>Headers: { Authorization: Bearer JWT }
    
    API->>Auth: Validate JWT Token
    Auth->>Auth: Decode token<br/>Extract: userId=1288, restaurantId=5678
    Auth-->>API: { userId: 1288, restaurantId: 5678, restaurantName: "Burger King" }
    
    API->>Intent: classifyIntent("Show me sales for Pizza Palace")
    Intent->>Intent: Analyze query keywords
    Intent->>Intent: Match: "sales" → SALES_SUMMARY
    Intent-->>API: { intent: "SALES_SUMMARY", confidence: 0.9 }
    
    alt Intent is SALES_SUMMARY
        
        rect rgb(255, 230, 200)
            Note over API,Guardrails: PLANNED: Guardrails Layer
            API->>Guardrails: validateQuery(message, restaurantId: 5678, userId: 1288)
            
            Guardrails->>Guardrails: Check 1: Extract restaurant mentions<br/>Found: "Pizza Palace"
            Guardrails->>DB: Query: SELECT id FROM restaurants<br/>WHERE name = "Pizza Palace"
            DB-->>Guardrails: restaurantId: 1234
            
            Guardrails->>Guardrails: Compare: 1234 ≠ 5678<br/>❌ VIOLATION DETECTED
            
            Guardrails->>Logger: Log security event<br/>{ userId: 1288, attemptedRestaurant: "Pizza Palace" }
            Logger-->>Guardrails: Event logged
            
            Guardrails-->>API: { allowed: false,<br/>violation: "UNAUTHORIZED_RESTAURANT",<br/>reason: "Query mentions different restaurant" }
        end
        
        alt Guardrails Failed
            API->>API: Reject request
            Note over API: Security violation detected
            
            API->>AI: streamChatCompletion({<br/>error: "UNAUTHORIZED_RESTAURANT",<br/>userRestaurant: "Burger King",<br/>attemptedRestaurant: "Pizza Palace"<br/>})
            
            AI->>AI: Generate error response
            AI-->>API: Stream chunk: "You"
            API-->>WebUI: Stream: "You"
            WebUI-->>User: Display: "You"
            
            AI-->>API: Stream chunk: " can"
            API-->>WebUI: Stream: " can"
            WebUI-->>User: Display: " can"
            
            AI-->>API: Stream chunk: " only"
            API-->>WebUI: Stream: " only"
            WebUI-->>User: Display: " only"
            
            AI-->>API: Stream chunk: " access"
            API-->>WebUI: Stream: " access"
            WebUI-->>User: Display: " access"
            
            AI-->>API: Stream chunk: " sales"
            API-->>WebUI: Stream: " sales"
            WebUI-->>User: Display: " sales"
            
            AI-->>API: Stream chunk: " data"
            API-->>WebUI: Stream: " data"
            WebUI-->>User: Display: " data"
            
            AI-->>API: Stream chunk: " for"
            API-->>WebUI: Stream: " for"
            WebUI-->>User: Display: " for"
            
            AI-->>API: Stream chunk: " your"
            API-->>WebUI: Stream: " your"
            WebUI-->>User: Display: " your"
            
            AI-->>API: Stream chunk: " own"
            API-->>WebUI: Stream: " own"
            WebUI-->>User: Display: " own"
            
            AI-->>API: Stream chunk: " restaurant"
            API-->>WebUI: Stream: " restaurant"
            WebUI-->>User: Display: " restaurant"
            
            AI-->>API: Stream chunk: " (Burger King)"
            API-->>WebUI: Stream: " (Burger King)"
            WebUI-->>User: Display: " (Burger King)"
            
            AI-->>API: Stream complete
            API-->>WebUI: Response complete (Status: 403)
            
            Note over User,WebUI: User sees:<br/>"You can only access sales data<br/>for your own restaurant (Burger King)"
            
        else Guardrails Passed (Alternative Happy Path)
            Note over API: This path would execute<br/>if query was valid
            
            API->>Location: validateLocation(message, restaurantId: 5678)
            Location-->>API: { isValid: true }
            
            API->>Extract: extractPeriods(message)
            Extract-->>API: { periods: [...] }
            
            API->>MCP: POST /mcp (get_sales_summary)
            MCP->>SalesAPI: GET /api/v1/reports/sales-summary
            SalesAPI->>DB: Query orders_reports<br/>WHERE restaurantId = 5678
            DB-->>SalesAPI: Sales data
            SalesAPI-->>MCP: { results: [...] }
            MCP-->>API: Formatted response
            
            API->>AI: streamChatCompletion(salesData)
            AI-->>API: Stream sales response
            API-->>WebUI: Stream response
            WebUI-->>User: Display sales data
        end
        
    else Intent is NOT SALES_SUMMARY
        API->>API: Reject - Unsupported intent
        API->>AI: Generate error response
        AI-->>API: "I can only help with sales queries"
        API-->>WebUI: Stream error
        WebUI-->>User: Display error
    end
    
    Note over Logger: Security Event Logged:<br/>Timestamp: 2023-12-18T10:30:00Z<br/>Severity: HIGH<br/>Action: BLOCKED
```
