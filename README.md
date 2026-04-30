# Auth — Node.js Authentication Service

A minimal Node.js + Express + MongoDB authentication service. Currently exposes a single `POST /api/auth/register` endpoint that creates a user and returns a JWT.

## Tech Stack

| Layer            | Tool                                      |
| ---------------- | ----------------------------------------- |
| Runtime          | Node.js (ES Modules — `"type": "module"`) |
| Web framework    | Express 5                                 |
| Database         | MongoDB via Mongoose                      |
| Auth tokens      | `jsonwebtoken` (JWT)                      |
| Password hashing | Node `crypto` (SHA-256)                   |
| Logging          | `morgan` (dev format)                     |
| Config           | `dotenv`                                  |
| Dev runner       | `nodemon`                                 |

## Project Structure

```
Auth/
├── server.js                         # Entry: connects DB, starts HTTP server
├── package.json
├── .env                              # MONGO_URI, JWT_SECRET, PORT (gitignored)
└── src/
    ├── app.js                        # Express app: middleware + route mounting
    ├── config/
    │   ├── config.js                 # Loads + validates env vars
    │   └── database.js               # Mongoose connection
    ├── controllers/
    │   └── auth.controller.js        # register handler
    ├── models/
    │   └── user.model.js             # User schema (username, email, password)
    └── routes/
        └── auth.routes.js            # /api/auth router
```

## Request Flow

1. [server.js](server.js) — calls `connectDB()` then `app.listen(PORT)`.
2. [src/app.js](src/app.js) — mounts `express.json()`, `morgan("dev")`, and the `/api/auth` router.
3. [src/routes/auth.routes.js](src/routes/auth.routes.js) — wires `POST /register` → controller.
4. [src/controllers/auth.controller.js](src/controllers/auth.controller.js) — checks duplicates, hashes password, creates user, signs JWT, responds.

## Environment Variables

Defined in `.env` (gitignored). Validated at startup in [src/config/config.js](src/config/config.js) — the process throws if any required key is missing.

| Key          | Purpose                        |
| ------------ | ------------------------------ |
| `MONGO_URI`  | MongoDB connection string      |
| `JWT_SECRET` | Secret used to sign JWTs       |
| `PORT`       | HTTP port (defaults to `3000`) |

## Setup & Run

```bash
# 1. Install dependencies
npm install

# 2. Create .env at project root with MONGO_URI, JWT_SECRET, PORT

# 3. Start dev server (nodemon)
npm run dev
```

Server logs `Connected to MongoDB` and `Server is running on port <PORT>` on success.

## API

### `GET /`

Health check — returns `Hello World!`.

### `POST /api/auth/register`

**Request body**

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "password": "plaintext-password"
}
```

**Responses**

| Status | When                                 | Body                                                  |
| ------ | ------------------------------------ | ----------------------------------------------------- |
| `201`  | User created                         | `{ message, user: { username, email }, token }`       |
| `400`  | Missing field(s)                     | `{ message: "All fields are required" }`              |
| `409`  | `username` or `email` already exists | `{ message: "Username or email already registered" }` |

The returned `token` is a JWT signed with `JWT_SECRET`, payload `{ id: <userId> }`, expiry `15min`.

## Data Model — `users`

| Field      | Type   | Constraints                             |
| ---------- | ------ | --------------------------------------- |
| `username` | String | required, unique                        |
| `email`    | String | required, unique                        |
| `password` | String | required (stored as SHA-256 hex digest) |

## Notes & Known Gaps

These are observations from reading the current code — useful context, not required changes.

- **Password hashing uses SHA-256, not bcrypt/argon2.** SHA-256 is fast and unsalted here, which is unsuitable for password storage. Consider `bcrypt` or `argon2`.
- **Field-presence check runs _after_ the duplicate lookup** in [src/controllers/auth.controller.js:9-21](src/controllers/auth.controller.js#L9-L21). The `400 "All fields are required"` branch is unreachable when a field is missing because the Mongo query runs first; reorder so validation precedes the DB call.
- **No `/login` endpoint yet** — only register is implemented.
- **No JWT verification middleware** — there are no protected routes.
- **`unique` is being passed an array** in [src/models/user.model.js](src/models/user.model.js) (e.g. `unique: [true, "..."]`). Mongoose's `unique` is an index hint, not a validator, and it does not take a custom message — the second element is silently ignored.
- **Mongoose model name is `"users"` (plural)** — Mongoose will pluralize this further to `userss`. Use `"User"` to get the conventional `users` collection.
- **`.env` is committed to the working tree** with real credentials visible in git status. The `.gitignore` excludes it going forward, but rotate `MONGO_URI` and `JWT_SECRET` if this repo was ever pushed.
