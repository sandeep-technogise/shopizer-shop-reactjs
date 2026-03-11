# Shopizer Shop — Architecture Diagram

## High-Level Architecture

```mermaid
graph TB
    subgraph Browser["Browser"]
        subgraph ReactApp["React Application"]
            Index["index.js\nEntry Point"]
            App["App.js\nRouter + Lang"]

            subgraph Pages["Pages (Lazy Loaded)"]
                Home["Home"]
                Category["Category"]
                ProductDetail["ProductDetail"]
                Search["SearchProduct"]
                Cart["Cart"]
                Checkout["Checkout"]
                Auth["Login / Register"]
                Account["MyAccount"]
                Orders["RecentOrder / OrderDetails"]
                Content["Content (CMS)"]
                Contact["Contact"]
                ForgotReset["ForgotPassword / ResetPassword"]
            end

            subgraph Components["Shared Components"]
                Header["Header + IconGroup"]
                Footer["Footer"]
                ProductCard["ProductCard / Modal"]
                HeroSlider["HeroSlider"]
                Breadcrumb["Breadcrumb"]
                Loader["Loader"]
                Cookie["CookieConsent"]
            end

            subgraph Redux["Redux Store"]
                direction TB
                Store["createStore\n+ thunk + localStorage"]
                subgraph Reducers["Reducers"]
                    CartR["cartData\n(cartItems, cartID, cartCount)"]
                    ProductR["productData\n(products, productID, categoryID)"]
                    MerchantR["merchantData\n(store, storeCode)"]
                    UserR["userData\n(user, countries, states, address)"]
                    ContentR["content\n(contentID)"]
                    LoaderR["loading\n(isLoading)"]
                    LangR["multilanguage\n(currentLanguageCode)"]
                end
                subgraph Actions["Action Creators"]
                    CartA["cartActions"]
                    ProductA["productActions"]
                    StoreA["storeAction"]
                    UserA["userAction"]
                    ContentA["contentAction"]
                    LoaderA["loaderActions"]
                end
            end

            subgraph Util["Utilities"]
                WebService["WebService\n(axios wrapper)"]
                Constants["constant.js\n(API path constants)"]
                Helper["helper.js\n(localStorage)"]
            end

            subgraph Persistence["Persistence"]
                LocalStorage["localStorage\n(redux-localstorage-simple)"]
                CookieStore["Cookies\n(cart ID, 6 months)"]
            end
        end
    end

    subgraph Backend["Shopizer Backend"]
        API["REST API\n{APP_BASE_URL}/api/v1/"]
        subgraph Endpoints["API Groups"]
            StoreAPI["Store\n/store/{merchant}"]
            ProductAPI["Products\n/products/ /product/{id}"]
            CategoryAPI["Categories\n/category/"]
            CartAPI["Cart\n/cart/"]
            CheckoutAPI["Checkout\n/checkout/"]
            AuthAPI["Auth\n/customers/ /auth/customer/"]
            ContentAPI["Content\n/content/"]
            LocationAPI["Location\n/country/ /zones/ /shipping/"]
            SearchAPI["Search\n/search/ /autocomplete/"]
            OrdersAPI["Orders\n/auth/customer/orders/"]
        end
    end

    subgraph External["External Services"]
        Stripe["Stripe\nPayment Gateway"]
        GoogleMaps["Google Maps\nGeocoding API"]
    end

    Index --> Store
    Index --> App
    App --> Pages
    App --> Components
    Pages --> Redux
    Components --> Redux
    Redux --> Actions
    Actions --> WebService
    WebService --> API
    API --> Endpoints
    Store --> Reducers
    Redux --> Persistence
    Checkout --> Stripe
    UserA --> GoogleMaps
```

---

## Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Page
    participant Redux
    participant WebService
    participant API as Shopizer API

    User->>Page: Interaction (e.g. Add to Cart)
    Page->>Redux: dispatch(action)
    Redux->>Redux: setLoader(true)
    Redux->>WebService: POST/GET/PUT/DELETE
    WebService->>API: HTTP Request + JWT Bearer token
    API-->>WebService: JSON Response
    WebService-->>Redux: Response data
    Redux->>Redux: dispatch(reducer update)
    Redux->>Redux: setLoader(false)
    Redux-->>Page: Updated state via connect()
    Page-->>User: Re-render with new state
```

---

## Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant LoginPage
    participant Redux
    participant API

    User->>LoginPage: Submit credentials
    LoginPage->>API: POST /customers/login/
    API-->>LoginPage: JWT token + user data
    LoginPage->>Redux: dispatch(setUser(data))
    LoginPage->>LocalStorage: store token
    Note over WebService: Axios interceptor injects<br/>Authorization: Bearer {token}<br/>on every subsequent request
```

---

## Cart Persistence Flow

```mermaid
flowchart LR
    A[Add to Cart] --> B{Cart ID exists?}
    B -- Yes --> C[PUT /cart/cartId]
    B -- No --> D[POST /cart/]
    C --> E[Response: cart code]
    D --> E
    E --> F[setShopizerCartID]
    F --> G[Cookie: merchant_shopizer_cart\n6-month expiry]
    F --> H[localStorage]
    G --> I[On next app load:\nApp.js reads cookie\ndispatches setShopizerCartID]
```

---

## Component Hierarchy

```mermaid
graph TD
    Provider["Redux Provider"] --> App
    App --> ToastProvider
    App --> BreadcrumbsProvider
    App --> Router
    Router --> Loader
    Router --> Cookie["CookieConsent"]
    Router --> Switch

    Switch --> Layout
    Layout --> Header
    Layout --> PageContent["Page Content"]
    Layout --> Footer

    Header --> HeaderTop["HeaderTop\n(header message)"]
    Header --> NavMenu["NavMenu\n(categories)"]
    Header --> IconGroup["IconGroup\n(cart, account, search)"]

    PageContent --> Home
    PageContent --> Category
    PageContent --> ProductDetail
    PageContent --> Cart
    PageContent --> Checkout

    Category --> ProductGrid["ProductGrid"]
    ProductGrid --> ProductCard["ProductCard"]
    ProductCard --> ProductModal["ProductModal (quick view)"]

    ProductDetail --> ProductDescriptionInfo
    ProductDetail --> ProductDescriptionTab["ProductDescriptionTab\n(reviews)"]
```

---

## Build & Deployment

```mermaid
flowchart LR
    subgraph Dev["Development"]
        src["src/"] --> CRA["react-scripts start\nnpm run dev"]
        CRA --> DevServer["localhost:3000"]
    end

    subgraph Prod["Production Build"]
        src2["src/"] --> Build["react-scripts build"]
        Build --> BuildDir["build/"]
    end

    subgraph Docker["Docker (Multi-stage)"]
        Node["node:13.12.0-alpine\nnpm ci + npm run build"] --> Nginx["nginx:stable-alpine\nServes build/"]
        Nginx --> EnvSh["env.sh injects\nruntime env vars\ninto env-config.js"]
        EnvSh --> Port["Port 80"]
    end

    BuildDir --> Docker
```
