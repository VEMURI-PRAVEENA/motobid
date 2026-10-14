# MotoBid — real-time auctions for used motorcycles

MotoBid is a full-stack auction platform. Registered riders bid on many used motorcycles at the
same time and see every bid, extension and close as it happens. Administrators list bikes,
schedule and publish auctions, cancel them when needed, and monitor the platform.

| | |
|---|---|
| **Architecture & design** | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| **Deployment** | [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) |
| **Assumptions & trade-offs** | [docs/ASSUMPTIONS.md](docs/ASSUMPTIONS.md) |
| **API reference (OpenAPI)** | `/api/docs` on a running instance |

## Highlights
- **Correct bidding under concurrency.** Each bid runs in one transaction that takes a
  per-auction row lock (`SELECT … FOR UPDATE`). Bids carry idempotency keys, a last-minutes bid
  extends the auction (anti-sniping), and the server rejects bids after close even if the worker
  is behind. A test fires 20 bids at the same time and checks that exactly one is accepted.
- **Real-time** over Socket.IO with a Redis adapter, so it works across many API replicas.
  Bidders get "you've been outbid" and "you won" notifications. Countdowns correct for the
  difference between the server clock and the browser clock.
- **Auction lifecycle**: DRAFT → SCHEDULED → LIVE → ENDED (SOLD/UNSOLD, based on a hidden
  reserve) or CANCELLED. A separate worker runs the transitions with `FOR UPDATE SKIP LOCKED`, so
  several workers can run at once.
- **Security**: argon2id password hashes; 15-minute JWTs plus httpOnly refresh cookies that
  rotate on every use and detect reuse; role-based access for admin routes; Zod validation on
  every input; Redis-backed rate limits; helmet/CSP; audit log; containers run as non-root.
- **Observability**: pino JSON logs with request IDs, Prometheus metrics (HTTP, bids,
  rejections by reason, transitions, WebSockets, event loop), a ready-made Grafana dashboard,
  and `/healthz` + `/readyz`.
- **Tests**: unit tests (domain rules), integration tests against real Postgres + Redis,
  component tests (React Testing Library), and Playwright end-to-end tests. CI runs on GitHub
  Actions.

## Demo credentials (seeded)
| Role | Email | Password |
|---|---|---|
| Admin | `admin@motobid.dev` | `Admin@12345` |
| Bidder | `alice@motobid.dev` | `Bidder@12345` |
| Bidder | `bob@motobid.dev` | `Bidder@12345` |
| Bidder | `carol@motobid.dev` | `Bidder@12345` |

The seed creates 12 auctions: 6 live (ending from ~8 minutes to 3 days after seeding),
3 upcoming, 2 ended (one sold, one with the reserve not met) and 1 draft.

## Quick start (Docker — recommended)
Requirements: Docker 24+ with Compose v2.

```sh
cp .env.example .env                       # optional for local use; set JWT_SECRET for anything shared
docker compose up -d --build               # postgres, redis, api, worker, web, prometheus, grafana
docker compose run --rm seed npm run db:seed -w @motobid/api -- --reset   # load demo data
```

| URL | What |
|---|---|
| http://localhost:8080 | MotoBid web app (the main entry point; nginx proxies `/api` and `/socket.io`) |
| http://localhost:8080/api/docs | OpenAPI / Swagger UI |
| http://localhost:3001 | Grafana — "MotoBid Operations" dashboard (anonymous viewer) |
| http://localhost:9090 | Prometheus |

To see live bidding without opening several browsers, start the bot bidders:
```sh
npm install && npm run demo:bots -w @motobid/api      # API_BASE_URL defaults to http://localhost:3000/api/v1
```
Re-run the seed command with `--reset` at any time to get fresh auctions with end times
relative to now.

## Local development (hot reload)
Requirements: Node.js 22 LTS, Docker.

```sh
npm install
docker compose up -d postgres redis
cp .env.example .env
npm run db:deploy -w @motobid/api          # apply migrations
npm run db:seed -w @motobid/api -- --reset # demo data
npm run dev                                # API on :3000 (worker in-process) + Vite on :5173
```

## Quality gates
```sh
docker compose -f docker-compose.test.yml up -d          # isolated test Postgres (5433) + Redis (6380)
export NODE_ENV=test JWT_SECRET=test-secret-at-least-32-characters-long \
  DATABASE_URL='postgresql://motobid:motobid@localhost:5433/motobid_test?schema=public' \
  REDIS_URL='redis://localhost:6380'
npm run db:deploy -w @motobid/api
npm run lint        # eslint + prettier --check
npm run typecheck
npm test            # unit + integration + component tests
npm run test:e2e    # Playwright, against the compose stack on :8080
```

## Repository layout
```
apps/api          Fastify API + lifecycle worker (Prisma, Socket.IO, prom-client)
  src/domain      pure business rules (bid rules, state machine, bidder state) — unit tested
  src/modules     auth, auctions, bids, me, admin  (routes → service → dto)
  src/lib         config, errors, logger, metrics, realtime, redis, prisma
  prisma          schema, migrations, seed
apps/web          React SPA (Vite, Tailwind, TanStack Query, socket.io-client)
packages/shared   Zod schemas, DTO types, socket event contracts, money helpers
infra             nginx, prometheus, grafana provisioning + dashboard
docs              architecture, deployment, assumptions & trade-offs
```
