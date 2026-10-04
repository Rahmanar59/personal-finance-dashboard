# Ledgerly — Personal Finance Dashboard

Ledgerly is a multi-user personal finance application built with the project’s **Next.js App Router, PostgreSQL, and Drizzle ORM** stack. It provides a responsive dashboard, account registration/login, persistent transaction and monthly-budget CRUD, and database-backed analytics.

> The starter project is a Next.js full-stack application, so the REST backend is implemented as native Next.js route handlers rather than a separate Express process. The API follows the requested REST paths, runs on Node.js, and shares the same PostgreSQL/Drizzle connection as the application. Browser requests are same-origin; no permissive CORS configuration is needed.

## Features

- **Secure accounts:** normalized/validated email, bcrypt password hashes (cost 12), signed seven-day JWT session in an HTTP-only, SameSite cookie, session restoration, logout, and generic invalid-credential errors.
- **User isolation:** every financial read/write requires a verified session. Transaction and budget mutations include both the record ID and authenticated user ID in the database predicate.
- **Transactions:** create, view, edit, delete, search, type/category/date filters, and sort. Custom category names are accepted as well as the included categories.
- **Budgets:** category and month budgets with actual expense totals, remaining amount, progress, approaching-limit warnings, and over-budget states.
- **Dashboard and analytics:** lifetime balance/income/expenses/savings rate, current-month spend, recent records, six-month cash-flow/savings charts, category breakdown, and budget-versus-actual visualizations. All amounts come from PostgreSQL.
- **Validation and feedback:** server-side Zod validation, browser form validation, loading/error/empty states, success feedback, and useful conflict errors.
- **Responsive UI:** desktop sidebar, compact mobile navigation, accessible forms, and responsive tables/charts.

## Requirements

- Node.js 20 or newer
- PostgreSQL 14 or newer
- npm

## Local installation

1. Install dependencies:

   ```bash
   npm install
   ```

2. Create a PostgreSQL database. For example, with a local PostgreSQL installation:

   ```bash
   createdb app_db
   ```

   Alternatively, create a database named `app_db` from `psql` or your PostgreSQL provider.

3. Copy `.env.example` to `.env` and set the connection string and a unique signing secret:

   ```bash
   cp .env.example .env
   ```

   Example values for a local PostgreSQL server:

   ```dotenv
   DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/app_db
   JWT_SECRET=<a unique random value of at least 32 characters>
   PORT=3000
   ```

   Generate a suitable signing key with `openssl rand -base64 48`. Do not commit `.env` or reuse a development secret in production. The JWT secret is required for registration and login.

4. Create/update tables and indexes from the Drizzle schema:

   ```bash
   npx drizzle-kit push
   ```

   `src/db/schema.ts` is the source of truth. `database/schema.sql` is also provided as a readable PostgreSQL schema reference; use **either** the SQL file or `drizzle-kit push` on an empty database, not both if you want to avoid already-existing object notices.

5. Run the app:

   ```bash
   npm run dev
   ```

   Open <http://localhost:3000>. Create an account, then add transactions and category budgets. The app uses the PostgreSQL database as its source of truth; closing the browser does not discard data.

For a production build:

```bash
npx next typegen
npm exec tsc -- --noEmit
npm run build
npm run start
```

Set `DATABASE_URL`, a strong `JWT_SECRET`, and (optionally) `PORT` in the deployment environment before starting the application.

## Architecture

```text
src/app/[section]/page.tsx             Protected workspace route shell
src/app/login, register                Account entry pages
src/components/auth-screen.tsx         Registration/login UI
src/components/finance-app.tsx         Responsive dashboard and CRUD UI
src/app/api/auth/*                     register, login, logout, current session
src/app/api/transactions/*             transaction REST collection and item API
src/app/api/budgets/*                  budget REST collection and item API
src/app/api/dashboard/*                database summary and analytics
src/lib/auth.ts                        bcrypt, JWT signing, cookie/session lookup
src/lib/validators.ts                  shared strict Zod request schemas
src/db/schema.ts                       PostgreSQL tables, checks, foreign keys, indexes
```

There is no separate Express server because this repository is hosted as a Next.js App Router application. Each handler runs on the Node.js runtime and responds with JSON using conventional HTTP methods/status codes. The API can be consumed by same-origin clients:

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/register` | Validate and create an account, hash the password, issue a session |
| `POST` | `/api/auth/login` | Verify credentials and issue a session |
| `POST` | `/api/auth/logout` | Expire the session cookie |
| `GET` | `/api/auth/me` | Return the current public user or `null` |
| `GET`, `POST` | `/api/transactions` | List/filter or create the signed-in user's transactions |
| `PUT`, `DELETE` | `/api/transactions/:id` | Update/delete an owned transaction |
| `GET`, `POST` | `/api/budgets` | Read a month's budgets with actual spend or create a budget |
| `PUT`, `DELETE` | `/api/budgets/:id` | Update/delete an owned budget |
| `GET` | `/api/dashboard/summary` | Database-aggregated dashboard totals, recents, and budget progress |
| `GET` | `/api/dashboard/analytics` | Six-month trends and expense/budget breakdowns |
| `GET` | `/api/health` | PostgreSQL-backed health check |

### Database model

- `users`: unique normalized email, name, bcrypt hash, timestamps.
- `transactions`: owner foreign key with `ON DELETE CASCADE`, `income`/`expense` enum, positive `numeric(12,2)` amount, category, description, payment method, date, timestamps, and user/date/type/category indexes.
- `budgets`: owner foreign key, category, positive `numeric(12,2)` amount, `YYYY-MM` month, unique `(user_id, category, month)`, and user/month index.

### Security notes

- Password hashes are never returned by the API. Password input is capped at 72 characters for bcrypt compatibility; registration requires at least eight characters plus upper/lowercase letters and a number.
- The cookie is `HttpOnly`, `SameSite=Lax`, path-scoped, and `Secure` in production. Set `JWT_SECRET` to a random secret of at least 32 characters.
- Inputs are validated in the route handlers, and database operations use Drizzle parameter binding. User ownership is part of every financial SQL predicate; a different account receives the same not-found response for another user's record ID.
- Do not expose the development database or `.env` to the public internet. For a public deployment, additionally apply your provider's TLS/network restrictions, backups, rate limiting, and monitoring.
