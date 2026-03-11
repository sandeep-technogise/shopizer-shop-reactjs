# Shopizer Shop React.js — Codebase Analysis

## Project Overview

E-commerce frontend built with React, connecting to a Shopizer backend API. Supports product browsing, cart management, checkout, user authentication, and multi-language (English & French).

---

## Architecture

### Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 16.6.0 (Create React App) |
| State Management | Redux + Redux Thunk |
| Routing | React Router DOM v5 |
| Styling | Bootstrap 4.5.0 + SCSS |
| HTTP Client | Axios |
| Payment | Stripe (@stripe/react-stripe-js) |
| Multi-language | redux-multilanguage (EN, FR) |
| UI Components | React Bootstrap, Swiper, SweetAlert |

### Project Structure

```
src/
├── assets/          # CSS, SCSS, fonts
├── components/      # Reusable UI components (header, footer, product, etc.)
├── data/            # Static data (hero sliders, feature icons)
├── helpers/         # Utility functions (scroll-top, product helpers)
├── layouts/         # Layout wrapper components
├── pages/           # Route-based page components
│   ├── home/
│   ├── category/
│   ├── product-details/
│   ├── search-product/
│   ├── content/
│   └── other/       # Cart, Checkout, Login, MyAccount, Orders, etc.
├── redux/
│   ├── actions/     # Redux action creators
│   └── reducers/    # Redux reducers
├── translations/    # i18n JSON files (en, fr)
├── util/            # Constants, WebService, helpers
├── wrappers/        # Component wrappers
├── App.js           # Main app with routing
└── index.js         # Entry point
```

### Redux State Slices

| Slice | Purpose |
|---|---|
| `productData` | Product listings, selected product/category IDs |
| `cartData` | Shopping cart items and cart ID |
| `merchantData` | Store/merchant configuration |
| `userData` | User profile, countries, states, addresses |
| `loading` | Global loader state |
| `content` | Content page data |
| `multilanguage` | Language settings (en/fr) |

---

## Main Dependencies

### Core
- `react` 16.6.0, `react-dom`, `react-router-dom` 5.1.2
- `redux` 4.0.4, `react-redux` 7.1.3, `redux-thunk` 2.3.0
- `axios` 0.21.1
- `bootstrap` 4.5.0, `react-bootstrap` 1.0.1

### E-commerce
- `@stripe/react-stripe-js`, `@stripe/stripe-js` — Payment processing
- `react-paginate` — Product pagination
- `react-star-ratings` — Product reviews
- `react-toast-notifications` — User notifications
- `universal-cookie` — Cart persistence

### UI/UX
- `swiper` 5.4.1, `react-id-swiper` — Carousels
- `react-bootstrap-sweetalert` — Alerts
- `react-modal-video` — Video modals
- `react-lightgallery` — Image galleries
- `animate.css` — Animations

### Utilities
- `google-maps-react`, `react-geocode` — Maps & geolocation
- `js-sha512` — Password hashing
- `moment` — Date formatting
- `react-hook-form` — Form validation

---

## Build & Deployment

### Development

```bash
npm install --legacy-peer-deps   # Install dependencies
npm run dev                       # Start dev server → http://localhost:3000
```

### Production Build

```bash
npm run build
```

### Docker

```bash
# Build
docker build . -t shopizerecomm/shopizer-shop:latest

# Run
docker run \
  -e "APP_MERCHANT=DEFAULT" \
  -e "APP_BASE_URL=http://localhost:8080" \
  -it --rm -p 80:80 shopizerecomm/shopizer-shop-reactjs
# → http://localhost
```

**Docker process**: Multi-stage build (Node 13.12.0 → Nginx). Builds React app, serves via Nginx on port 80, runs `env.sh` to inject runtime environment variables.

### Environment Variables

Configured in `public/env-config.js` or `.env`:

| Variable | Default | Description |
|---|---|---|
| `APP_BASE_URL` | `http://localhost:8080` | Backend API URL |
| `APP_API_VERSION` | `/api/v1/` | API version prefix |
| `APP_MERCHANT` | `DEFAULT` | Store identifier |
| `APP_THEME_COLOR` | `#D1D1D1` | Primary theme color |
| `APP_STRIPE_KEY` | — | Stripe publishable key |
| `APP_PAYMENT_TYPE` | `STRIPE` | Payment gateway (`STRIPE` / `NUVEI`) |
| `APP_MAP_API_KEY` | — | Google Maps API key |
| `APP_PRODUCT_GRID_LIMIT` | `15` | Products per page |
| `APP_NUVEI_TERMINAL_ID` | — | Nuvei terminal ID |
| `APP_NUVEI_SECRET` | — | Nuvei secret |

---

## Frontend Routes

| Route | Component | Description |
|---|---|---|
| `/` | `Home` | Homepage |
| `/category/:id` | `Category` | Product listing by category |
| `/product/:id` | `ProductDetail` | Product detail page |
| `/search/:id` | `SearchProduct` | Search results |
| `/content/:id` | `Content` | CMS content pages |
| `/cart` | `Cart` | Shopping cart |
| `/checkout` | `Checkout` | Checkout flow |
| `/order-confirm` | `OrderConfirm` | Order confirmation |
| `/order-details/:id` | `OrderDetails` | Single order detail |
| `/recent-order` | `RecentOrder` | Order history |
| `/login` | `LoginRegister` | Login form |
| `/register` | `LoginRegister` | Registration form |
| `/my-account` | `MyAccount` | User profile management |
| `/forgot-password` | `ForgotPassword` | Password reset request |
| `/customer/:code/reset/:id` | `ResetPassword` | Password reset |
| `/contact` | `Contact` | Contact form |

---

## API Endpoints

Base URL: `{APP_BASE_URL}/api/v1/`

### Store & Health

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/store/{merchant}` | Get merchant/store details |
| `GET` | `/actuator/health/ping` | Health check |

### Products

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/products/?store={merchant}&lang={lang}&category={id}&start={page}&count={limit}` | List products by category |
| `GET` | `/product/{id}?store={merchant}&lang={lang}` | Get product details |
| `GET` | `/products/group/{code}?store={merchant}&lang={lang}` | Get product group |
| `POST` | `/search/?store={merchant}&lang={lang}` | Full-text product search |
| `GET` | `/autocomplete/?name={query}&store={merchant}&lang={lang}` | Search autocomplete |

### Categories

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/category/?store={merchant}&lang={lang}` | List all categories |
| `GET` | `/category/{id}?store={merchant}&lang={lang}` | Get category details |

### Cart

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/cart/?store={merchant}` | Create cart (body: `{product, quantity, attributes?}`) |
| `PUT` | `/cart/{cartId}?store={merchant}` | Update cart item |
| `GET` | `/cart/{cartId}?lang={lang}` | Get cart details |
| `DELETE` | `/cart/{cartId}/product/{itemId}?store={merchant}` | Remove item from cart |
| `GET` | `/auth/customer/cart?cart={cartId}&lang={lang}` | Get authenticated user's cart |

### Checkout & Orders

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/cart/{cartId}/total/?store={merchant}` | Calculate cart total |
| `GET` | `/cart/{cartId}/shipping?country={code}&postalCode={zip}&store={merchant}` | Get shipping options |
| `POST` | `/checkout/?store={merchant}` | Place order |
| `GET` | `/auth/customer/orders/?store={merchant}&lang={lang}` | List user orders |
| `GET` | `/auth/customer/orders/{orderId}?store={merchant}&lang={lang}` | Get order details |

### Authentication & Users

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/customers/login/?store={merchant}` | Login |
| `POST` | `/customers/register?store={merchant}` | Register new customer |
| `GET` | `/auth/customer/profile` | Get user profile |
| `PATCH` | `/auth/customer/profile` | Update profile |
| `PATCH` | `/auth/customer/address` | Update address |
| `PATCH` | `/auth/customer/password` | Change password |
| `POST` | `/customers/password/request/?store={merchant}` | Request password reset |
| `GET` | `/customers/password/reset/{code}/{id}` | Validate reset token |
| `POST` | `/customers/password/reset/{code}/{id}` | Reset password |

### Location & Shipping

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/country/?store={merchant}&lang={lang}` | List countries |
| `GET` | `/shipping/country?store={merchant}&lang={lang}` | List shipping countries |
| `GET` | `/zones/?code={countryCode}` | Get states/zones for a country |

### Content & CMS

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/content/{code}?store={merchant}&lang={lang}` | Get content page |
| `GET` | `/content/pages/?store={merchant}&lang={lang}` | List content pages |
| `GET` | `/content/boxes/?store={merchant}&lang={lang}&code={boxCode}` | Get content boxes |
| `GET` | `/content/bannerText/?store={merchant}&lang={lang}` | Get banner text |
| `GET` | `/content/headerMessage/?store={merchant}&lang={lang}` | Get header message |
| `GET` | `/content/agreement/?store={merchant}&lang={lang}` | Get terms/agreement |
| `GET` | `/content/promo/?store={merchant}&lang={lang}` | Get promotions |

### Reviews, Newsletter & Contact

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/product/{productId}/reviews?store={merchant}&lang={lang}` | Submit product review |
| `POST` | `/newsletter/?store={merchant}` | Subscribe to newsletter |
| `POST` | `/contact/?store={merchant}` | Submit contact form |

### Manufacturers

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/manufacturers/?store={merchant}&lang={lang}` | List manufacturers |

---

## Key Features

1. **Multi-language** — English & French via redux-multilanguage
2. **Persistent Cart** — Cart ID stored in cookies (6-month expiry)
3. **Product Variants** — Supports product attributes/options in cart
4. **JWT Authentication** — Bearer token injected via Axios interceptor
5. **Checkout Flow** — Shipping calculation + Stripe payment
6. **Search** — Full-text search with autocomplete
7. **Order History** — View past orders and details
8. **Geolocation** — Auto-detect user location for shipping
9. **Cookie Consent** — GDPR-compliant banner
10. **SEO** — React Meta Tags for page metadata
11. **Lazy Loading** — All pages loaded lazily via `React.lazy`
12. **Theme Color** — Configurable via `APP_THEME_COLOR` env var (CSS variable)
