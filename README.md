# MERN Commerce: Nova Store

A full-stack e-commerce app built with **MongoDB, Express, React and Node.js**. Customers browse, search and filter products, manage a cart, check out with Stripe and track orders. Admins manage products (with image uploads), fulfil orders and see sales stats.

## Features

**Shop**
- Product catalogue with search, category filters, sorting and pagination (all kept in the URL, so they're shareable)
- Product pages with live stock status
- Cart saved in localStorage, synced across tabs, with a free-shipping progress bar
- Checkout with a saved shipping address, then Stripe Checkout or simulated payment
- Order history and order detail pages, with "Pay now" for unpaid orders

**Admin** (`/admin`)
- Dashboard: revenue, orders, items to ship, low-stock alerts, recent orders
- Product create, edit and delete, with image upload (Multer)
- Order list with filters; mark orders as delivered

**Security**
- JWT in an `httpOnly`, `SameSite=Lax` cookie (not readable by JavaScript, blocks CSRF)
- bcrypt password hashing; hashes are never returned by the API
- Every input validated with **zod**, which also blocks NoSQL operator injection
- **Prices are recalculated on the server**; the client can't change what it pays
- Stock is checked when an order is placed and decremented once, atomically, on payment
- Rate-limited login and register, Helmet security headers, upload type and size limits
- Users can only see their own orders; admin routes are role-protected

## Quick start

```bash
npm install
npm run dev
```

- Shop: http://localhost:5174
- API: http://localhost:5100

**No MongoDB install needed.** In development, if `MONGODB_URI` isn't set, the server starts a local MongoDB automatically (downloaded on first run), stores data in `server/.mongo-data`, and loads sample products.

| Demo account | Email | Password |
| --- | --- | --- |
| Admin | admin@example.com | admin12345 |
| Customer | jane@example.com | password123 |

To reset the sample data, stop `npm run dev` first, then run `npm run seed`.

## Payments

Without Stripe keys, payments are **simulated**: orders are marked paid instantly.

To use real Stripe test mode, create `server/.env` from `server/.env.example` and add `STRIPE_SECRET_KEY=sk_test_...`. Pay with test card `4242 4242 4242 4242`, any future expiry date and any CVC.

Orders are confirmed when the customer returns from Stripe. For production, also add a webhook pointing at `/api/webhooks/stripe` (event `checkout.session.completed`) and set `STRIPE_WEBHOOK_SECRET`. That way payments are recorded even if the customer closes the tab. For local testing, run `stripe listen --forward-to localhost:5100/api/webhooks/stripe`.

## Project structure

```
shared/pricing.js     Shipping, tax and money formatting, used by both client and server
server/src/
  app.js              Express app: middleware, routes, static files
  models/             User, Product, Order (Mongoose)
  routes/             auth, products, orders, upload, admin
  middleware/         auth (JWT), validate (zod), errors
  services/payments.js  Stripe Checkout, webhook, idempotent markPaid
client/src/
  context/            AuthContext, CartContext (useReducer + localStorage)
  hooks/useApi.js     Data fetching with abort + reload
  pages/              Shop, cart, checkout, orders, auth, admin/*
```

## API

| Method | Route | Access |
| --- | --- | --- |
| POST | `/api/auth/register` · `/login` · `/logout` | public |
| GET | `/api/auth/me` | user |
| GET | `/api/products?search=&category=&sort=&page=&limit=` | public |
| GET | `/api/products/categories` · `/api/products/:idOrSlug` | public |
| POST / PUT / DELETE | `/api/products` · `/api/products/:id` | admin |
| POST | `/api/orders` | user |
| GET | `/api/orders/mine` · `/api/orders/:id` | owner or admin |
| POST | `/api/orders/:id/pay` · `/api/orders/:id/verify` | owner |
| GET / PUT | `/api/orders` · `/api/orders/:id/deliver` | admin |
| POST | `/api/upload` | admin |
| GET | `/api/admin/stats` | admin |

## Deploy (Render, single service)

1. Create a free MongoDB Atlas cluster and copy its connection string.
2. On Render, create a new **Web Service** from this repo:
   - Build command: `npm install && npm run build`
   - Start command: `npm start`
   - Environment: `NODE_ENV=production`, `MONGODB_URI`, `JWT_SECRET` (a long random string), `CLIENT_URL` (your Render URL), and the Stripe keys if you use them
3. Load the sample data once by running `npm run seed` locally with `MONGODB_URI` pointing at Atlas.

Note: uploaded images are stored on the server's disk, which is wiped on Render's free tier when it redeploys. For production, move uploads to Cloudinary or S3.

## Ideas to extend

- Product reviews and ratings
- Wishlist
- Order confirmation emails (Resend or Nodemailer)
- Cloudinary image storage
- Tests with Vitest, Supertest and Playwright, plus CI with GitHub Actions
