# Rideshare Chat API

A standard-layout Go REST API starter using [chi](https://github.com/go-chi/chi) as the
router and [pgx](https://github.com/jackc/pgx) for PostgreSQL access.

## Folder structure

```
rideshare-chat-api/
├── cmd/
│   └── api/            # main.go — application entrypoint
├── internal/
│   ├── config/          # env/config loading
│   ├── database/         # PostgreSQL connection pool
│   ├── handler/          # HTTP handlers (controllers)
│   │   ├── admin/        # admin endpoints (conversations, bot toggle, bot status)
│   │   ├── customer/     # customer chat endpoints (messages, history)
│   │   ├── driver/       # driver chat endpoints (messages, history)
│   │   ├── ws_handler.go # WebSocket upgrade handler
│   │   └── health_handler.go
│   ├── middleware/        # HTTP middleware (HMAC auth, logging, security headers, body limit)
│   ├── router/           # route registration
│   ├── socket/           # WebSocket hub & client broadcast manager
│   ├── store/            # data access layer (admin, customer, driver, bot history, message history)
│   ├── vendor/           # external chat platform API clients (driver, customer, SSE managers)
│   └── utils/            # helpers (vendor ID prefixing, text sanitization, upload paths)
├── pkg/
│   └── response/          # shared reusable packages (JSON envelope)
├── migrations/            # raw SQL migrations
├── uploads/               # served media uploads directory
├── .env.example
├── go.mod
├── Makefile
└── README.md
```

This follows the community-standard [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
conventions: `cmd/` for entrypoints, `internal/` for private application code
that can't be imported by other projects, and `pkg/` for code that is safe to
share/reuse.

## Prerequisites

- Go 1.22+
- PostgreSQL running locally (or a connection string to a remote instance)

## Setup

1. Copy the example env file and adjust values:

   ```bash
   cp .env.example .env
   ```
   
   Ensure that all environment variables are correctly populated, especially the new variables for Vendor Chat API integration (`VENDOR_CHAT_API_URL`, `VENDOR_SECRET_KEY`) and your public endpoint (`BASE_URL`).

2. Create the database referenced in `DATABASE_URL` (default name: `chat-api`):

   ```bash
   createdb chat-api
   ```

3. Install dependencies:

   ```bash
   make tidy
   ```

4. (Optional) apply the sample migration:

   ```bash
   make migrate-up
   ```

5. Run the API:

   ```bash
   make run
   ```

   The server starts on `http://localhost:8080` by default.

## Architecture Overview

The API supports three roles: **Customer**, **Driver**, and **Admin**. Each role
has its own handler package under `internal/handler/`. All REST chat endpoints are
protected by HMAC request signing (see `internal/middleware/hmac.go`), while WebSocket upgrades happen after authentication. The API employs robust middlewares, including CORS configuration and a 10MB payload size limit.

### WebSocket Hub

A single in-memory hub (`internal/socket/hub.go`) manages WebSocket connections
and broadcasts messages. Clients connect with a `channel` query parameter
(`DRIVER` or `CUSTOMER`) to receive only messages targeting their role.

### Vendor Integration

External chat platform API clients live in `internal/vendor/`:
- `vendor_client.go` — driver-side vendor client (upload media, forward messages)
- `customer_client.go` — customer-side vendor client (forward messages, toggle bot)
- `sse_manager.go` — background SSE listener for driver bot/agent replies
- `customer_sse.go` — background SSE listener for customer bot/agent replies

### Admin & Bot Management

Admins can toggle the AI bot on/off and query its status for any driver or
customer conversation via:
- `POST /api/admin/bot/toggle`
- `GET /api/admin/bot/status`
- `GET /api/admin/bot/history`

Admins can also query active conversations:
- `GET /api/admin/conversations` (all active conversations)
- `GET /api/admin/conversations/customers` (customer conversations)
- `GET /api/admin/conversations/drivers` (driver conversations)

### REST Chat Endpoints

Beyond historical messages (`GET /api/{role}/messages`) and sending new messages (`POST /api/{role}/messages`), both Driver and Customer endpoints support:
- `POST /api/{role}/messages/seen` - Mark messages as read/seen.
- `PATCH /api/{role}/messages/{id}` - Edit a previously sent message.
- `GET /api/{role}/messages/{id}/edits` - View the edit history of a message.

### Static File Serving

Uploaded media files are hosted locally and securely served through the `/uploads/*` endpoint.

## Documentation

Please see the complete [API Documentation & Integration Guide](API_DOCUMENTATION.md) for full details on:
- How to authenticate requests (HMAC Signatures).
- How to connect WebSockets and make REST API calls.
- Complete copy-paste code examples for Next.js, Flutter, and PHP/Laravel.

## Next steps

- Add more resources by creating a handler in `internal/handler/`, wiring it
  in `internal/router/router.go`, and adding any queries in `internal/database/`
  or a new `internal/repository/` package as the app grows.
- Swap the manual SQL migration approach for a tool like
  [golang-migrate](https://github.com/golang-migrate/migrate) once you have
  more than a couple of migrations.
- Add unit tests alongside each package (`*_test.go`).
