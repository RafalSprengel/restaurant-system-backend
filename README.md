
<img width="2155" height="1363" alt="Zrzut ekranu 2026-01-06 125154" src="https://github.com/user-attachments/assets/3aea5291-1413-4f53-9152-81f60eef12d7" />

<img width="2155" height="1363" alt="Restaurant system screenshot" src="https://github.com/user-attachments/assets/3aea5291-1413-4f53-9152-81f60eef12d7" />

# 🍽️ Restaurant & Online Ordering System – Backend API

A scalable RESTful API built with **Node.js** and **Express.js**, powering a full restaurant ordering and management platform. It handles authentication, business logic, payments, reservations, messaging, and data persistence using **MongoDB**.

The API is designed to serve multiple client applications, including a customer-facing frontend and an administrative dashboard (both are delivered by the [frontend repository](https://github.com/RafalSprengel/restaurant-pos-frontend)).

## 🚀 Live Environment

- **Connected frontend:** [restaurant.rafalsprengel.com](https://restaurant.rafalsprengel.com/)
- **API base path:** all routes are versioned under `/v1` (in production they are exposed behind a reverse proxy as `/api/v1/...`)

---

## 🧩 System Overview

The backend is structured as a **modular monolith** with a clear separation of concerns:

- **Authentication & Authorization Layer** – JWT cookies, refresh-token rotation, OAuth, role checks
- **Business Logic Layer** – Orders, Products, Reservations, Messaging, Statistics, Settings
- **Data Access Layer** – MongoDB via Mongoose
- **Integration Layer** – Stripe (Checkout + webhooks), Google / Facebook OAuth, SMTP (Nodemailer)

```
Client (React SPA)
      │  HTTPS + httpOnly cookies
      ▼
Express app ── CORS ── cookie-parser ── visit counter
      │
      ├── /v1/stripe/webhook   (raw body, signature verified)
      └── /v1/*                (JSON body)
              │
        routes ─► middleware (authentMiddleware ─► authorize(roles)) ─► controllers ─► Mongoose models ─► MongoDB
```

---

## 🔑 Core Functionalities

### 🔐 Authentication & Authorization

- **JWT authentication stored in `httpOnly`, `Secure`, `SameSite=None` cookies** – tokens are never exposed to client-side JavaScript.
- **Short-lived access token (1 hour) + long-lived refresh token (15 days).** The refresh token is stored server-side in MongoDB (with a TTL index), is **rotated on every refresh**, and is deleted on logout. The refresh cookie is scoped to `/v1/auth/refresh-token` only.
- **Two separate account types:**
  - **Customers** – self-registration with email + password, or sign-in with **Google / Facebook OAuth** (via Passport.js; a Google sign-in is linked to an existing account with the same email).
  - **Staff** – created only by an administrator (`POST /v1/auth/register/mgmt`), with a dedicated login endpoint.
- **Role-based access control for staff:** `guest` (read-only), `member`, `moderator`, `admin`. Roles are verified against the database on every protected request by the `authorize([...roles])` middleware, so a role change takes effect immediately.
- Password hashing with **bcryptjs**.

| Role | Typical permissions |
|------|---------------------|
| `guest` | Read-only access to dashboard data (orders, customers, messages, reservations, staff, settings) |
| `member` | Guest + update orders and customers, reply to / delete messages, delete reservations |
| `moderator` | Member + manage products, categories, tables and system settings |
| `admin` | Full access, including deleting orders/customers and managing staff accounts |

### 🛒 Orders System

- Create orders through the Stripe checkout flow – for **registered customers and guests** (the customer is optionally identified from the auth cookie).
- Three order types: **`delivery`**, **`pickup`**, **`dine-in`**; delivery address is required only for `delivery`; optional requested time and customer note.
- **Prices and totals are calculated server-side** from the product collection – the client only sends product IDs and quantities.
- Each order stores a **snapshot** of purchased items (name, price, ingredients, dietary flags), so history stays correct when the menu changes.
- Human-friendly auto-incremented `orderNumber`.
- Payment tracking (`isPaid`, `paidAt`, `paymentIntent`, `paymentFailureReason`).
- Customers can view their own order history and hide orders from it (soft delete via `isDeletedByCustomer`).
- Staff can search, sort, paginate, update and delete orders.

### 🍔 Product & Category Management

- Full CRUD for menu **products** and **categories**.
- Product attributes: price, description, ingredients, category, `isVegetarian`, `isGlutenFree`, `isAvailable`, `isFeatured`.
- Server-side **search, sorting, pagination and filtering by category** for dynamic frontend rendering.
- **Image upload pipeline:** Multer (JPEG/PNG/GIF, up to 10 MB) → Sharp resizes the image to 600 px width and generates a 300 px thumbnail; files are served from `/uploads`. Images are removed when a product is deleted.

### 🪑 Table Reservation System

- Tables with number, capacity and location (managed by staff).
- **Availability endpoint** returning bookable time slots for a given table and date.
- **Conflict prevention:** overlapping reservations for the same table are rejected (`409`).
- Time-slot validation – reservation window (10:00–23:00), 2-hour reservations, bookings up to 21 days ahead, no bookings in the past.
- Staff can list (search / sort / paginate / filter by table), view and delete reservations.

### 💬 Messaging System

- Public **contact form** endpoint – messages are always saved to the database.
- Staff inbox with **received / sent** folders, read/unread tracking, unread counter, search, pagination and bulk delete.
- Replies are stored as linked `sent` messages (`originalTopic`).
- **Optional email notifications via SMTP (Nodemailer):** when SMTP settings are configured, the restaurant is notified and the sender receives a copy. Email failures never lose the message.

### 📊 Statistics

A single `/v1/stats` endpoint feeds the admin dashboard: total orders and revenue (paid orders only), last-7-days orders and revenue, **unique daily visitors** (cookie-based counter), top-selling products and the latest transactions.

### ⚙️ System Settings

Stored in MongoDB and editable by staff: reservation rules (opening hours, start offset, reservation duration, max days in advance) and SMTP configuration. The SMTP password is never returned by the API.

---

## 💳 Payments

- **Stripe Checkout** (hosted payment page) – supports **card, BLIK and Przelewy24**, currency **PLN**.
- An unpaid order is created first, then linked to the Stripe session through metadata.
- **Webhook endpoint** (`POST /v1/stripe/webhook`) with **signature verification** on the raw request body; handles `payment_intent.succeeded`, `payment_intent.payment_failed` and `payment_intent.canceled` to update the order.
- `GET /v1/stripe/session-status` lets the frontend confirm the result after the redirect.

---

## 🛠 Infrastructure & Security

- Centralised logging with **Winston** (console + `logs/error.log`); `console.error` output is forwarded to the logger.
- **CORS** allow-list with credentials support.
- Environment-based configuration with **dotenv**.
- Middleware layers: authentication, role authorization, optional customer identification, file upload / image processing, visit counting.
- MongoDB connection handling – the process exits on connection loss so the process manager (PM2) can restart it.
- Mongoose schemas use `strict: 'throw'` to reject unknown fields.
- **CI/CD:** GitHub Actions workflow deploys on every push to `main` (rsync over SSH to a VPS, `.env` generated from repository secrets, PM2 restart).

---

## 🗄 Data Layer

MongoDB with the Mongoose ODM. Schema-based models:

| Model | Purpose |
|-------|---------|
| `Customer` | Customer accounts (local / Google / Facebook), saved delivery address, auto-increment `customerNumber` |
| `Staff` | Staff accounts with role (`guest`, `member`, `moderator`, `admin`) |
| `Product` | Menu items with dietary flags, image + thumbnail |
| `Category` | Menu categories with display order and image |
| `Order` | Orders with item snapshots, purchaser details, order type, payment state |
| `Table` | Restaurant tables (number, capacity, location) |
| `Reservation` | Table reservations with time slot and customer details |
| `Message` | Contact-form messages and staff replies |
| `Settings` | Reservation rules and SMTP configuration |
| `RefreshToken` | Server-side refresh tokens with TTL expiry |
| `Visit` | Daily visitor counter |

A sample `mongo_dump/` (BSON/metadata) is included in the repository.

---

## 🌐 API Reference

All routes are prefixed with `/v1`. 🔓 = public, 🔐 = requires login, role in brackets = minimum staff role.

### Auth – `/v1/auth`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/register/customer` | 🔓 | Register a customer |
| POST | `/login/customer` | 🔓 | Customer login (sets cookies) |
| POST | `/register/mgmt` | 🔐 admin | Create a staff account |
| POST | `/login/mgmt` | 🔓 | Staff login (sets cookies) |
| GET | `/session` | 🔐 | Current user from the access token |
| POST | `/refresh-token` | 🔓 (refresh cookie) | Rotate access + refresh tokens |
| POST | `/logout` | 🔐 | Clear cookies and revoke refresh token |
| GET | `/google`, `/google/callback` | 🔓 | Google OAuth |
| GET | `/facebook`, `/facebook/callback` | 🔓 | Facebook OAuth |

### Products & Categories
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/products` | 🔓 | List products (`page`, `limit`, `search`, `sortBy`, `sortOrder`, `category`) |
| GET | `/products/:id` | 🔓 | Single product |
| POST / PUT / DELETE | `/products`, `/products/:id` | 🔐 moderator | Create / update / delete (multipart `image`) |
| GET | `/product-categories`, `/product-categories/:id` | 🔓 | List / single category |
| POST / PUT / DELETE | `/product-categories`, `/product-categories/:id` | 🔐 moderator | Manage categories |

### Orders – `/v1/orders`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/customer` | 🔐 | Logged-in customer's orders |
| PUT | `/customer/:id` | 🔐 | Hide an order from the customer's history |
| GET | `/order-types` | 🔓 | Available order types |
| GET | `/`, `/:id` | 🔐 guest | List (search/sort/paginate) / single order |
| PUT | `/:id` | 🔐 member | Update an order |
| DELETE | `/:id` | 🔐 admin | Delete an order |

### Stripe – `/v1/stripe`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/create-checkout-session` | 🔓 (customer optional) | Create order + Stripe Checkout session |
| GET | `/session-status` | 🔓 | Payment status of a session |
| POST | `/webhook` | Stripe | Signed webhook |

### Customers – `/v1/customers`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET / PUT / DELETE | `/customer` | 🔐 | Own profile: read, update, delete account |
| PUT | `/customer/update-password` | 🔐 | Change own password |
| GET | `/`, `/:id` | 🔐 guest | List / single customer |
| PUT | `/:id` | 🔐 member | Update a customer |
| DELETE | `/:id` | 🔐 admin | Delete a customer |

### Staff – `/v1/staff`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/`, `/roles`, `/:id` | 🔐 guest | List staff / roles / single member |
| PUT, DELETE | `/:id` | 🔐 admin | Update / delete a staff member |

### Tables & Reservations – `/v1/tables`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/type-of-tables` | 🔓 | List tables |
| GET | `/availability?date=&tableNumber=` | 🔓 | Available time slots |
| POST | `/reservations` | 🔓 | Create a reservation |
| GET | `/reservations`, `/reservations/:id` | 🔐 guest | List / single reservation |
| DELETE | `/reservations/:id` | 🔐 member | Delete a reservation |
| POST | `/` | 🔐 moderator | Add a table |

### Messages – `/v1/messages`
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/` | 🔓 | Submit contact-form message |
| GET | `/`, `/received`, `/sent`, `/unread-count`, `/:id` | 🔐 guest | Inbox, folders, counter, single message (marks as read) |
| POST | `/reply` | 🔐 member | Reply to a message |
| POST | `/delete-many`, DELETE `/:id` | 🔐 member | Delete messages |

### Stats & Settings
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/v1/stats` | 🔐 guest | Dashboard statistics |
| GET | `/v1/settings` | 🔐 guest | Read settings |
| PUT | `/v1/settings` | 🔐 moderator | Update settings |

---

## 📁 Architecture

```
├── config/          # Passport strategies (local, Google, Facebook)
├── controllers/     # Business logic handlers (auth, orders, products, stripe, tables, ...)
├── db/
│   ├── models/      # Mongoose schemas
│   └── mongoose.js  # Database connection
├── middleware/      # Authentication, authorization, customer identification, uploads, visit counter
├── routes/
│   └── v1/          # API endpoints grouped by domain
├── utils/           # Winston logger
├── mongo_dump/      # Sample database dump
├── index.js         # Application entry point
└── package.json
```

## 🚀 Purpose

This backend is designed as the core engine of a restaurant management ecosystem, providing secure, scalable, and modular API services for multiple frontend clients.

## 🧰 Tech Stack

| Category | Technology |
|----------|------------|
| Runtime | Node.js |
| Framework | Express.js 4 |
| Database | MongoDB + Mongoose 8 (`mongoose-sequence`, `mongoose-timestamp`) |
| Authentication | JWT (`jsonwebtoken`) in httpOnly cookies, Passport.js (Google & Facebook OAuth), bcryptjs |
| Payments | Stripe API (Checkout + webhooks) |
| Email | Nodemailer (SMTP configured in the database) |
| File handling | Multer, Sharp |
| Dates | Day.js |
| Logging | Winston |
| Deployment | GitHub Actions, rsync over SSH, PM2 |

---

## ⚙️ Getting Started

### Prerequisites

- Node.js 18+
- A MongoDB instance (local or Atlas)
- Stripe account (secret key + webhook secret) – needed for payments
- Optional: Google and Facebook OAuth apps

### Installation

```bash
git clone https://github.com/RafalSprengel/restaurant-pos-backend
cd restaurant-pos-backend
npm install
```

### Environment variables

Create a `.env` file in the project root:

```env
DATABASE_URL=mongodb://localhost:27017/restaurant

# JWT
JWT_SECRET=change-me
JWT_SECRET_REFRESH=change-me-too
# JWT_REFRESH_TOKEN_MAX_AGE=1296000   # optional, seconds (TTL of stored refresh tokens)

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# OAuth (optional)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
FACEBOOK_CLIENT_ID=
FACEBOOK_CLIENT_SECRET=
```

SMTP credentials are **not** environment variables – they are configured in the admin panel (Settings) and stored in the database.

### Run

```bash
npm start        # production
npm run devStart # development with nodemon
```

The server listens on **port 3001**.

### Local development notes

- Auth cookies are set with `Secure` + `SameSite=None`, so the API and frontend should be served over **HTTPS** (browsers treat `localhost` as secure in most cases).
- Allowed CORS origins are defined in `index.js` – add your own frontend origin there.
- Google / Facebook OAuth callback URLs are configured in `controllers/authController.js` and must match the URLs registered in the provider consoles.
- For webhooks locally, use the Stripe CLI: `stripe listen --forward-to localhost:3001/v1/stripe/webhook`.
- Optionally restore sample data: `mongorestore --db restaurant mongo_dump/restaurant`.

---

## 🔄 Status

This repository serves as the centralized backend service. It is architected to be consumed by the React-based frontend, ensuring a clear separation of concerns.
