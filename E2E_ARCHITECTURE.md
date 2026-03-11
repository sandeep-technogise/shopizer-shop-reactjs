# Shopizer — End-to-End Architecture Diagram

## System Overview

Three repositories make up the full Shopizer platform:

| Repo | Tech | Role |
|------|------|------|
| `shopizer` | Java / Spring Boot | Backend REST API |
| `shopizer-admin` | Angular SPA | Merchant admin panel |
| `shopizer-shop-reactjs` | React SPA | Customer-facing storefront |

---

## End-to-End Architecture

```mermaid
graph TB
    %% ─── CLIENTS ───────────────────────────────────────────────────────────
    subgraph Clients["Clients"]
        subgraph ShopUI["shopizer-shop-reactjs  (React + Redux)"]
            direction TB
            SHOP_PAGES["Pages\nHome · Category · Product\nCart · Checkout · Account"]
            SHOP_REDUX["Redux Store\ncartData · productData\nmerchantData · userData"]
            SHOP_WS["WebService (axios)\nJWT Bearer interceptor"]
            SHOP_PERSIST["Persistence\nlocalStorage + Cookies\n(cart ID, 6-month)"]
            SHOP_EXT["External\nStripe · Google Maps"]
        end

        subgraph AdminUI["shopizer-admin  (Angular)"]
            direction TB
            ADMIN_ROUTER["Angular Router\n(hash-based, AuthGuard)"]
            ADMIN_MODULES["Lazy Modules\nCatalogue · Orders · Customers\nShipping · Payment · Tax\nContent · Store · Users"]
            ADMIN_CRUD["CrudService\n(HTTP wrapper)"]
            ADMIN_INTERCEPTOR["AuthInterceptor\nJWT injection"]
            ADMIN_TOKEN["TokenService\n(localStorage)"]
        end
    end

    %% ─── BACKEND ────────────────────────────────────────────────────────────
    subgraph Backend["shopizer  (Spring Boot)"]
        subgraph Presentation["sm-shop — Presentation Layer"]
            FILTERS["Filters\nCorsFilter · XssFilter"]
            JWT_SEC["Spring Security\nJWT AuthFilter"]
            API_V1["REST API v1\n/api/v1/..."]
            API_V2["REST API v2\n/api/v2/..."]
        end

        subgraph Contract["sm-shop-model — API Contract Layer"]
            FACADES["Facades\nProductFacade · OrderFacade\nCustomerFacade · CategoryFacade\nShoppingCartFacade · StoreFacade\nShippingFacade · TaxFacade\nContentFacade · UserFacade"]
            DTOS["DTOs\nReadable* · Persistable*"]
        end

        subgraph Business["sm-core — Business Layer"]
            SERVICES["Business Services\nProductService · OrderService\nCustomerService · CategoryService\nShoppingCartService · MerchantService\nShippingService · TaxService"]
            REPOS["Repositories\n(Spring Data JPA)"]
        end

        subgraph Domain["sm-core-model — Domain Layer"]
            ENTITIES["JPA Entities\nProduct · Order · Customer\nCategory · MerchantStore\nShoppingCart · User · Tax"]
        end

        subgraph Integrations["sm-core-modules — Integration Layer"]
            PAY_MOD["Payment\nPayPal · Stripe · Braintree"]
            SHIP_MOD["Shipping\nCanada Post · Custom"]
            STORE_MOD["Storage\nLocal · AWS S3 · GCS"]
            EMAIL_MOD["Email\nSMTP · AWS SES"]
            SEARCH_MOD["Search\nElasticsearch"]
        end
    end

    %% ─── INFRASTRUCTURE ─────────────────────────────────────────────────────
    subgraph Infra["Infrastructure"]
        DB[("Database\nH2 · MySQL · PostgreSQL")]
        CACHE[("Cache\nInfinispan · Ehcache")]
        ES[("Elasticsearch")]
        S3[("File Storage\nAWS S3 · GCS · Local")]
        EMAIL_SRV["Email\nSMTP · AWS SES"]
        STRIPE_GW["Stripe / PayPal\n/ Braintree"]
    end

    %% ─── FLOWS ──────────────────────────────────────────────────────────────

    %% Shop → Backend
    SHOP_PAGES --> SHOP_REDUX
    SHOP_REDUX --> SHOP_WS
    SHOP_WS -->|"HTTP/JSON :8080/api/v1"| FILTERS
    SHOP_REDUX --> SHOP_PERSIST
    SHOP_PAGES --> SHOP_EXT

    %% Admin → Backend
    ADMIN_ROUTER --> ADMIN_MODULES
    ADMIN_MODULES --> ADMIN_CRUD
    ADMIN_CRUD --> ADMIN_INTERCEPTOR
    ADMIN_INTERCEPTOR --> ADMIN_TOKEN
    ADMIN_CRUD -->|"HTTP/JSON :8080/api/v1"| FILTERS

    %% Backend internal
    FILTERS --> JWT_SEC
    JWT_SEC --> API_V1
    JWT_SEC --> API_V2
    API_V1 --> FACADES
    API_V2 --> FACADES
    FACADES --> DTOS
    FACADES --> SERVICES
    SERVICES --> REPOS
    REPOS --> ENTITIES
    ENTITIES --> DB
    SERVICES --> CACHE
    SERVICES --> PAY_MOD
    SERVICES --> SHIP_MOD
    SERVICES --> STORE_MOD
    SERVICES --> EMAIL_MOD
    SERVICES --> SEARCH_MOD

    %% Infra
    SEARCH_MOD --> ES
    STORE_MOD --> S3
    EMAIL_MOD --> EMAIL_SRV
    PAY_MOD --> STRIPE_GW
```

---

## Request Lifecycle (Customer Storefront)

```mermaid
sequenceDiagram
    participant User
    participant React as React (Shop)
    participant Redux
    participant Axios as Axios (WebService)
    participant Filter as CORS/XSS Filter
    participant JWT as JWT Auth Filter
    participant API as REST Controller
    participant Facade
    participant Service
    participant DB

    User->>React: Interaction (e.g. Add to Cart)
    React->>Redux: dispatch(action)
    Redux->>Axios: HTTP request
    Axios->>Filter: POST /api/v1/cart + Bearer token
    Filter->>JWT: Sanitized request
    JWT->>API: Authenticated request
    API->>Facade: Facade method call
    Facade->>Service: Business logic
    Service->>DB: JPA query
    DB-->>Service: Entity
    Service-->>Facade: Processed entity
    Facade-->>API: DTO (Readable*)
    API-->>Axios: JSON response
    Axios-->>Redux: Response data
    Redux-->>React: State update
    React-->>User: Re-render
```

---

## Request Lifecycle (Admin Panel)

```mermaid
sequenceDiagram
    participant Admin
    participant Angular
    participant AuthGuard
    participant Interceptor as AuthInterceptor
    participant CrudService
    participant API as Shopizer API

    Admin->>Angular: Navigate to /pages/*
    Angular->>AuthGuard: canActivate()
    AuthGuard->>Angular: Check localStorage token
    alt No token → redirect to /auth/login
        Admin->>Angular: Submit credentials
        Angular->>API: POST /api/v1/private/login
        API-->>Angular: JWT token
        Angular->>Angular: saveToken → redirect /pages
    end
    Angular->>CrudService: HTTP call
    CrudService->>Interceptor: Outgoing request
    Interceptor->>API: Request + Authorization: Bearer <token>
    API-->>CrudService: JSON response
    CrudService-->>Angular: Observable
    Angular-->>Admin: Rendered view
```

---

## Authentication & Security

```mermaid
graph TD
    REQ["Incoming Request\n(Shop or Admin)"] --> CORS["CorsFilter"]
    CORS --> XSS["XssFilter"]
    XSS --> JWT_F["JWT AuthFilter"]

    JWT_F --> PUB{"Public endpoint?"}
    PUB -->|Yes| CTRL["REST Controller"]
    PUB -->|No| AUTH{"Valid JWT?"}

    AUTH -->|No| R401["401 Unauthorized"]
    AUTH -->|Yes| ROLE{"Role check"}

    ROLE -->|"/private/* → ADMIN role"| CTRL
    ROLE -->|"/auth/* → CUSTOMER role"| CTRL
    ROLE -->|Mismatch| R403["403 Forbidden"]

    CTRL --> FACADE["Facade → Service → DB"]
```

---

## Cart Persistence (Storefront)

```mermaid
flowchart LR
    A["Add to Cart"] --> B{"Cart ID\nin cookie?"}
    B -- Yes --> C["PUT /api/v1/cart/{id}"]
    B -- No --> D["POST /api/v1/cart/"]
    C --> E["Response: cart code"]
    D --> E
    E --> F["Redux: setShopizerCartID"]
    F --> G["Cookie\nmerchant_shopizer_cart\n(6-month expiry)"]
    F --> H["localStorage\n(redux-localstorage-simple)"]
    G --> I["App.js on load:\nreads cookie → dispatches\nsetShopizerCartID"]
```

---

## Deployment Topology

```mermaid
graph LR
    subgraph Docker_Shop["Docker: shopizer-shop-reactjs"]
        N1["node:13 — npm build"] --> NG1["nginx:alpine\nPort 80\nenv.sh injects runtime env"]
    end

    subgraph Docker_Admin["Docker: shopizer-admin"]
        N2["node — ng build"] --> NG2["nginx\nPort 80"]
    end

    subgraph Docker_Backend["Docker: shopizer"]
        JVM["JVM — Spring Boot\nPort 8080"]
    end

    subgraph Docker_Infra["Infrastructure Containers"]
        DB2[("MySQL / PostgreSQL\nPort 3306/5432")]
        ES2[("Elasticsearch\nPort 9200")]
    end

    NG1 -->|"API calls :8080"| JVM
    NG2 -->|"API calls :8080"| JVM
    JVM --> DB2
    JVM --> ES2
```

---

## Module Dependency Summary

```mermaid
graph LR
    subgraph Frontend
        SHOP["shopizer-shop-reactjs\n(React + Redux)"]
        ADMIN["shopizer-admin\n(Angular)"]
    end

    subgraph Backend["shopizer (Spring Boot)"]
        SM_SHOP["sm-shop\n(Presentation)"]
        SM_SHOP_MODEL["sm-shop-model\n(API Contracts)"]
        SM_CORE["sm-core\n(Business Logic)"]
        SM_CORE_MODEL["sm-core-model\n(Domain Entities)"]
        SM_CORE_MODULES["sm-core-modules\n(Integrations)"]
    end

    SHOP -->|"REST /api/v1"| SM_SHOP
    ADMIN -->|"REST /api/v1"| SM_SHOP

    SM_SHOP --> SM_SHOP_MODEL
    SM_SHOP --> SM_CORE
    SM_SHOP --> SM_CORE_MODEL
    SM_SHOP_MODEL --> SM_CORE_MODEL
    SM_CORE --> SM_CORE_MODEL
    SM_CORE --> SM_CORE_MODULES
    SM_CORE_MODULES --> SM_CORE_MODEL
```
