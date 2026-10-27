# 🍕 Pizza Planet

**A full-stack food ordering and restaurant management system for a single restaurant: from menu browsing to delivery tracking.**

Final Year Project · BS Computer Science (2022–2026) · Sarhad University of Science and Information Technology (SUIT), Peshawar, Pakistan

![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-Express-339933?logo=node.js&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-InnoDB-4479A1?logo=mysql&logoColor=white)
![Tests](https://img.shields.io/badge/tests-22%20passing-brightgreen?logo=jest)
![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-lightgrey)

---

## 📖 Overview

Small restaurants often take orders over phone calls, WhatsApp messages, or paper notes. That leads to misunderstood items, manual totals, incomplete delivery details, and customers repeatedly calling to ask *"where is my order?"*

**Pizza Planet** replaces that with one web application that serves two audiences:

- **Customers** browse the menu, pick sizes, manage a cart, check out with cash on delivery, and track their order live.
- **Restaurant staff** use a protected admin dashboard to view incoming orders, filter by status, update progress, and monitor revenue and customer counts.

Unlike large multi-vendor delivery platforms, Pizza Planet gives one restaurant direct control over its menu, customer data, order workflow, and branding.

---

## ✨ Features

### Customer side
- Categorised menu with keyword search and item cards
- Multiple sizes per item (Small / Medium / Large) with individual pricing
- Cart drawer with a compound cart key (item + size), so the same item in different sizes is handled correctly
- Checkout with phone and address validation, special instructions, and **cash on delivery**
- Optional customer accounts (register / login) with saved delivery addresses
- Unique order numbers (`PP-YYYYMMDD-…`) and a dedicated **order tracking page** (polls every 10 seconds)
- Floating active-order tracker, plus device and audio notifications when status changes
- Cart and session persisted in browser local storage

### Admin side
- Role-protected dashboard (`/admin`) using JWT and role-verification middleware
- Order list with status filtering and one-click status updates
- Operational stats: total orders, pending orders, today's orders, revenue, customer count
- Auto-refresh every 30 seconds
- Full order-status history kept as an audit trail

### Business rules (enforced server-side)
| Rule | Behaviour |
|---|---|
| Delivery fee | Rs. 100 when subtotal is below Rs. 2,000; **free at Rs. 2,000 or above** |
| Totals | Recalculated on the server; client-submitted totals are never trusted |
| COD settlement | Order marked `is_paid = TRUE` automatically when it moves to `delivered` |
| Status lifecycle | `pending → confirmed → preparing → ready → out_for_delivery → delivered` (or `cancelled`) |

---

## 🏗️ Architecture

Three-tier client–server design:

```
┌─────────────────────┐      REST/JSON      ┌──────────────────────┐      SQL (mysql2)      ┌────────────────┐
│  React 18 + Vite    │ ◄─────────────────► │  Node.js + Express   │ ◄────────────────────► │  MySQL (InnoDB)│
│  React Router       │                     │  JWT · bcrypt · RBAC │   connection pool (10) │  8 tables, 3NF │
│  Context API state  │                     │  Business rules      │                        │                │
└─────────────────────┘                     └──────────────────────┘                        └────────────────┘
```

### Tech stack

| Layer | Technologies |
|---|---|
| Frontend | React 18, Vite 5, React Router 7, Context API (Cart, Auth, Wishlist, Toast), CSS |
| Backend | Node.js, Express 4, mysql2 (promise pool), bcryptjs, jsonwebtoken, cors, dotenv |
| Database | MySQL / MariaDB (InnoDB) |
| Testing | Jest 29, Supertest 7 |
| Security | bcrypt hashing, JWT (15-minute expiry), role-based middleware, Helmet headers, rate limiting on auth routes, parameterised queries |

---

## 🗄️ Database design

Eight normalised tables (3NF): `users`, `addresses`, `categories`, `menu_items`, `item_sizes`, `orders`, `order_items`, `order_status_history`.

Key design decisions:

- **`DECIMAL(10,2)` for money**, avoiding floating-point rounding errors.
- **`order_items` stores a snapshot** of item name and unit price, so old orders never change when the menu is repriced.
- **`orders.user_id` uses `ON DELETE SET NULL`**, so financial records survive account deletion, while saved addresses use `ON DELETE CASCADE`.
- **`ENUM` columns** for `role` and order `status` enforce valid values at the engine level.
- **Native JSON column** for menu item tags.
- **B-tree indexes** on email, phone, order status, and foreign keys used in joins.

Order creation runs inside a **single ACID transaction** (order → line items → initial status history). Any failure triggers `rollback()`, and the connection is always released in a `finally` block to prevent pool exhaustion.

Full schema: [`backend/database/schema.sql`](backend/database/schema.sql)

---

## 🔌 API overview

| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public (rate limited) | Create a customer account |
| POST | `/api/auth/login` | Public (rate limited) | Log in and receive a JWT |
| GET | `/api/menu/items` | Public | Menu data |
| POST | `/api/orders` | Public / customer | Place an order (transactional) |
| GET | `/api/orders/:orderNumber` | Public | Track an order |
| GET | `/api/admin/dashboard` | Admin | Summary statistics |
| GET | `/api/admin/orders` | Admin | List and filter orders |
| PATCH | `/api/admin/orders/:id/status` | Admin | Update order status |
| GET | `/api/health` | Public | Health check |

> Check the files in `backend/routes/` for exact paths and payloads, as some routes may differ slightly from this summary.

---

## 🚀 Getting started

### Prerequisites
- [Node.js](https://nodejs.org/) 18+
- [MySQL Community Server](https://dev.mysql.com/downloads/mysql/) 5.7+ (JSON column support required)

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/pizza-planet.git
cd pizza-planet
```

### 2. Configure the backend
Create `backend/.env` (use `backend/.env.example` as a template):

```env
PORT=5000
NODE_ENV=development

# Use a long random string, never commit the real value
JWT_SECRET=replace_with_a_long_random_secret

DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_mysql_password
DB_NAME=pizza_planet

WHATSAPP_NUMBER=your_number
```

### 3. Install dependencies and set up the database
```bash
# Backend
cd backend
npm install
npm run setup-db      # creates the database, tables, menu data, and an admin user

# Frontend
cd ..
npm install
```

### 4. Run the app
Open two terminals:

```bash
# Terminal 1: API on http://localhost:5000
cd backend
npm start

# Terminal 2: Web app on http://localhost:5173
npm run dev
```

Open **http://localhost:5173** for the customer site and **http://localhost:5173/admin** for the admin dashboard.

> ⚠️ The setup script seeds a default admin account for local development. **Change its credentials before any real deployment.**

---

## 🧪 Testing

Automated integration tests use **Jest** and **Supertest** against the Express app, with the MySQL pool mocked so tests are fast, deterministic, and can inject database failures.

```bash
cd backend
npm test
```

**Result: 3 suites, 22 tests, all passing (~2.4 s).**

| Suite | What it verifies |
|---|---|
| `auth.test.js` | Registration validation, duplicate accounts, JWT issuance, invalid logins, brute-force rate limiting (429 on the 6th attempt) |
| `orders.test.js` | Empty-cart rejection, delivery-fee boundaries (below / at / above Rs. 2,000), **transaction rollback** on failures during order and order-item inserts |
| `admin.test.js` | 401 without token, 403 for customers, 200 for admins, COD auto-settlement on delivery, Helmet security headers |

Why mock the database? It gives deterministic runs with no external state, sub-second speed, the ability to simulate crashes mid-transaction to prove rollbacks work, and portability to any machine or CI runner.

---

## 📁 Project structure

```
pizza-planet/
├── src/                      # React frontend
│   ├── components/           # Header, Menu, MenuCard, CartDrawer, ActiveOrderPopup, auth modal…
│   ├── contexts/             # Cart, Auth, Wishlist, Toast state
│   ├── pages/                # Checkout, OrderTracking, Admin
│   ├── hooks/                # Scroll reveal, device notifications
│   ├── data/                 # Menu data
│   └── styles/               # CSS
├── public/images/            # Food photography
└── backend/
    ├── server.js             # Express app entry point
    ├── routes/               # auth, menu, orders, admin
    ├── middleware/           # JWT auth + admin role guard
    ├── config/db.js          # MySQL connection pool
    ├── database/schema.sql   # Schema and seed data
    ├── setup_local_db.js     # One-command DB setup
    └── tests/                # Jest + Supertest suites
```

---

## ⚠️ Known limitations

Pizza Planet currently runs in a local development environment. It does **not** include:

- Online payment gateway (cash on delivery only)
- Live GPS rider tracking
- Inventory management
- WebSocket-based real-time updates (status is polled at fixed intervals)
- The customer menu is currently served from local data in `src/data/menu-data.js`; the MySQL menu API exists for future integration

## 🔮 Future work

- Payment gateway integration (card and mobile wallets)
- Live delivery tracking
- Dynamic menu loaded from the database with an admin menu editor
- Inventory management
- WebSockets or server-sent events instead of polling
- Production deployment (hosted API, managed MySQL, CI/CD)

---

## 👥 Team

| Name | Roll No. |
|---|---|
| Samiullah | SUIT-22-01-001-0011 |
| Raza Ur Rehman | SUIT-22-01-001-0026 |
| Saif Ul Islam | SUIT-22-01-001-0028 |


**Supervisor:** Engr. Usman Saleem
**Department:** Computer Science & Information Technology, SUIT, Peshawar

## 📄 License

**Copyright © 2026 Raza Ur Rehman, Saif Ul Islam, Samiullah. All Rights Reserved.**

This repository is published for portfolio and academic evaluation only. The code and assets may not be copied, modified, distributed, or used without written permission from the authors. (LICENSE).

---

<p align="center">Built as a Final Year Project at SUIT, Peshawar 🇵🇰</p>
