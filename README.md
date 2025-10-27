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

```
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
```
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
