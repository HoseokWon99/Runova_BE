# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run start:dev          # Start with hot-reload watch mode
npm run start:debug        # Start with debug + watch mode

# Build
npm run build              # Standard NestJS build
npm run build:swc          # Faster build using SWC compiler

# Lint
npm run lint               # ESLint with auto-fix

# Tests
npm run test               # Run all unit tests
npm run test:watch         # Unit tests in watch mode
npm run test:cov           # Unit tests with coverage report
npm run test:e2e           # End-to-end tests

# Database Migrations
npm run migration:run      # Run pending migrations
```

## Architecture

**Runova Backend** is a NestJS monolith for a running/jogging platform with social features.

### Module Structure

Each feature module lives under `src/` and follows `module → controller(s) → service(s)` pattern:

- **`auth/`** — Google OAuth + JWT authentication. Issues access/refresh tokens; refresh token stored as argon2 hash.
- **`user/`** — User profile management.
- **`running/`** — Running records (with geospatial path storage), dashboards, and art URL generation. Larger modules split controllers and services into subdirectories.
- **`courses/`** — Predefined running routes with search, bookmarking, and path inspection.
- **`community/`** — Posts (types: FREE, PROOF, SHARE, MATE), comments, and likes.
- **`files/`** — Presigned S3 URL generation for client-side uploads.
- **`art/`** — Canvas-based art generation from running path data using `skia-canvas`.
- **`config/`** — TypeORM, Redis (via `iovalkey`), and cache-manager setup.
- **`common/`** — Base entities (`EntityBase`, `ImmutableEntityBase`), shared interceptors, geo utilities.
- **`utils/`** — Custom decorators (`@User()`, `@Cached()`, `@Caching()`).
- **`modules/`** — TypeORM entities only (no business logic here).
- **`migrations/`** — TypeORM CLI migrations (synchronize is disabled).

### Key Architectural Details

**Database**: PostgreSQL via TypeORM. `synchronize: false` — all schema changes must go through migrations. Entities are in `src/modules/*/`. `RunningRecord` stores GPS paths as `LineStringColumn` with PostGIS functions (`ST_Length`, `ST_Transform`) for distance/pace calculations.

**Authentication**: JWT access + refresh token strategy. Guards applied via `@UseGuards(JwtAuthGuard)`. The `@User()` decorator extracts the authenticated user from the request context.

**Caching**: Redis (iovalkey client) via NestJS cache-manager. Use `@Caching(ttlSeconds)` and `@Cached()` decorators on endpoints. Default cache TTL is 1 hour for running records.

**Transactions**: `typeorm-transactional` is initialized in `main.ts`. Use `@Transactional()` for service methods that need atomicity.

**Art Generation**: `skia-canvas` renders running path visualizations. Config in `art-config.json` (1024×1024, white lines, glow effects). The `art/` module handles generation and S3 upload; `running/` handles URL retrieval.

**Error Tracking**: Sentry is initialized first in `instrument.ts` (imported before all else in `main.ts`). `SentryGlobalFilter` is registered as a global exception filter.

**API Docs**: Swagger available at `/api-docs` in development.

### Request Lifecycle

```
Request → JwtAuthGuard (validates JWT) → Controller → Service → TypeORM → PostgreSQL
                                                                ↘ Redis Cache
```

Global `ValidationPipe` (with `class-transformer`) handles DTO validation. Global `LoggingInterceptor` logs all requests/responses.

### Environment

The app uses `@nestjs/config` with environment variables. Docker Compose mounts `.env.production`. SSL certificates for PostgreSQL are mounted as volumes in development.
