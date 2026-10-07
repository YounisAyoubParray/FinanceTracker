# Finance Tracker: Role-Based Access Control Backend

A finance tracking web app with JWT authentication and three-level role-based access control,
built with Node.js, Express and PostgreSQL.

## Features

- Registration and login with bcrypt password hashing
- JWT authentication via an `Authorization: Bearer` header or an HTTP-only cookie
- Role-based access control with three roles: **viewer**, **analyst** and **admin**
- Income and expense records with dashboard totals (income, expense, net)
- Admin tools to list users and delete records or users
- Per-IP request rate limiting
- Central error handling with JSON error responses

## Roles

| Capability                          | Viewer | Analyst | Admin |
| ----------------------------------- | :----: | :-----: | :---: |
| Create records and view own records |   ✓    |    ✓    |   ✓   |
| View all records and overall totals |        |    ✓    |   ✓   |
| List users                          |        |         |   ✓   |
| Delete any record                   |        |         |   ✓   |
| Delete other users                  |        |         |   ✓   |

Every role also sees its own personal totals on the dashboard. Admins cannot delete their own account.

## Tech stack

| Layer          | Technology                                        |
| -------------- | ------------------------------------------------- |
| Server         | Node.js, Express                                  |
| Database       | PostgreSQL (`pg`)                                 |
| Authentication | jsonwebtoken, bcrypt, cookie-parser               |
| Security       | express-rate-limit                                |
| Views          | EJS dashboard, plain HTML and JavaScript frontend |

## Getting started

### Prerequisites

- Node.js 18 or later
- PostgreSQL

### 1. Install

```bash
git clone https://github.com/YounisAyoubParray/FinanceTracker.git
cd FinanceTracker
npm install
```

### 2. Create the database schema

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password TEXT NOT NULL,
  role VARCHAR(20) NOT NULL CHECK (role IN ('viewer', 'analyst', 'admin'))
);

CREATE TABLE records (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  amount NUMERIC(12, 2) NOT NULL CHECK (amount >= 0),
  type VARCHAR(20) NOT NULL CHECK (type IN ('income', 'expense')),
  category VARCHAR(100),
  notes TEXT,
  date TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
PORT=3000
PG_USER=your_pg_user
PG_HOST=localhost
PG_DATABASE=your_database_name
PG_PASSWORD=your_pg_password
PG_PORT=5432
JWT_SECRET=a_long_random_secret
NODE_ENV=development
```

| Variable      | Description                                               |
| ------------- | --------------------------------------------------------- |
| `PORT`        | Port the server listens on                                |
| `PG_*`        | PostgreSQL connection settings                            |
| `JWT_SECRET`  | Secret used to sign and verify tokens                     |
| `NODE_ENV`    | `production` enables secure cookies and hides error detail |

### 4. Run

```bash
npm run dev     # development, restarts on changes (nodemon)
npm start       # production
```

Then open http://localhost:3000.

## API

| Method | Route                | Access        | Description                                      |
| ------ | -------------------- | ------------- | ------------------------------------------------ |
| POST   | `/auth/register`     | Public        | Create an account with an email, password and role |
| POST   | `/auth/login`        | Public        | Log in; returns a JWT and sets the `token` cookie |
| POST   | `/records`           | Authenticated | Create an income or expense record               |
| GET    | `/api/dashboard`     | Authenticated | Latest 50 records and totals, scoped by role (JSON) |
| GET    | `/dashboard`         | Authenticated | Dashboard page rendered with EJS                 |
| GET    | `/admin/users`       | Admin         | List all users                                   |
| DELETE | `/admin/records/:id` | Admin         | Delete a record                                  |
| DELETE | `/admin/users/:id`   | Admin         | Delete a user and all of their records           |

### Example

```bash
# Log in and get a token
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "your_password"}'

# Add a record
curl -X POST http://localhost:3000/records \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"amount": 2500, "type": "income", "category": "Salary"}'

# Fetch dashboard data
curl http://localhost:3000/api/dashboard \
  -H "Authorization: Bearer <token>"
```

## How it works

- **Authentication.** `authenticateJWT` reads the token from the `Authorization` header or
  the `token` cookie, verifies it with `JWT_SECRET`, and attaches `{ id, email, role }` to
  `req.user`. Missing tokens get `401`; invalid or expired tokens get `403`.
- **Authorization.** `authorizeAdmin` lets a request through only when `req.user.role` is
  `admin`. Dashboard queries are scoped by role: viewers see only their own records and
  totals, while analysts and admins see every record and the overall totals.
- **Errors.** Unknown routes return `404`, and a central error handler returns JSON errors,
  hiding internal details when `NODE_ENV=production`.

## Security notes

- Passwords are hashed with bcrypt (10 salt rounds) and never stored in plain text.
- Tokens expire after **1 hour**. The auth cookie is `httpOnly` and `sameSite=lax`, and it is
  `secure` in production.
- Each IP is limited to **100 requests per 15 minutes**.
- All SQL queries are parameterised.

## Project structure

```
server.js            Express app: middleware, authentication, API routes and error handling
public/
  index.html         Landing page
  register.html      Registration page
  login.html         Login page
  main.js            Frontend logic for auth, records, dashboard and admin actions
views/
  dashboard.ejs      Dashboard rendered after login
```
