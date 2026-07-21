# The Game Cellar

> A microservice platform for managing a personal game backlog, with content-based recommendations driven by user-declared preferences and rating history.

![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0-brightgreen)
![React](https://img.shields.io/badge/React-19-blue)
![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6)
![Postgres](https://img.shields.io/badge/PostgreSQL-17-336791)
![Keycloak](https://img.shields.io/badge/Keycloak-26-blue)
![License](https://img.shields.io/badge/License-MIT-blue)

![The Game Cellar](https://raw.githubusercontent.com/The-Game-Cellar/.github/main/docs/hero.png)

## About

The Game Cellar is a backlog manager for players whose unplayed pile has outgrown their memory of it. Add a game, mark it `PLAYING`, `BACKLOG`, `COMPLETED`, `DROPPED`, or `WISHLIST`, rate it 1 to 10, and the recommendation service surfaces what to play next based on what you have already liked and what genres, tags, and release eras you have told it about.

The catalog is sourced from the [IGDB API](https://www.igdb.com/). Authentication is handled by a self-hosted Keycloak realm. Every service is independently deployable.

## Architecture

```
Frontend (5173)
      |
API Gateway (8000)  <->  Keycloak (8080)
      |
      +--> Redis (6379)  (rate-limit buckets + recommendation cache / pub-sub)
      |
+-------------+----------------------+----------------------------+
Game Service   Library Service    Recommendation Service
   (8081)         (8082)                (8083)
 PostgreSQL    PostgreSQL              PostgreSQL
  game_db       library_db          recommendation_db
 (port 5432)  (port 5433)             (port 5434)
```

| Service                | Port | Database               |
|------------------------|------|------------------------|
| Frontend               | 5173 |                        |
| API Gateway            | 8000 |                        |
| Keycloak               | 8080 | `keycloak_db` (5432\*) |
| Game Service           | 8081 | `game_db` (5432)       |
| Library Service        | 8082 | `library_db` (5433)    |
| Recommendation Service | 8083 | `recommendation_db` (5434)          |
| Redis                  | 6379 | in-memory (rate-limit + rec cache) |

\* Keycloak's own Postgres instance is internal to the Docker network and not exposed on the host.

All traffic from the frontend goes through the API Gateway. Services never accept a `user_id` from a request body or query string; identity comes from the JWT `sub` claim issued by Keycloak. There are no cross-service foreign keys: `user_id` is a Keycloak UUID stored as `VARCHAR`, and `igdb_game_id` is an IGDB reference. Services stay independently deployable.

## Tech Stack

**Backend**
- Java 17, Spring Boot 4.0
- Spring Cloud Gateway MVC (servlet-based) for routing
- Spring Security with OAuth 2 Resource Server for JWT validation
- PostgreSQL 17 with Flyway-managed schemas
- Redis for rate-limit buckets, recommendation caches, and cross-service pub/sub
- Keycloak 26 for authentication and identity
- IGDB API for catalog data, with a nightly cache worker

**Frontend**
- React 19 (functional components only)
- TypeScript 5.9 with `strict: true`
- Vite 8 build tool
- Tailwind CSS v4
- TanStack Query v5 for server state
- React Router v7
- DTOs generated end-to-end from each service's OpenAPI spec via `openapi-typescript`

**Testing**
- JUnit 5 + Mockito on the backend
- Vitest, Testing Library, and Mock Service Worker on the frontend

## Repositories

| Repository                                                                                       | Purpose                                                                                                                       |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| [`api-gateway`](https://github.com/The-Game-Cellar/api-gateway)                                  | Single entry point for the frontend. Routing, JWT validation, CORS, login / register / refresh / logout endpoints.            |
| [`game-service`](https://github.com/The-Game-Cellar/game-service)                                | IGDB API client and local catalog cache. Search, browse, and game-detail data. Background worker walks IGDB nightly.          |
| [`library-service`](https://github.com/The-Game-Cellar/library-service)                          | User game collection, statuses, ratings, platforms, and declared preferences (genre, tag, release-year). Daily DUSTY sweep.   |
| [`recommendation-service`](https://github.com/The-Game-Cellar/recommendation-service)            | Three-tier content-based recommendations, precomputed per user by background workers into its own Postgres store with a Redis cache. Blends rating evidence with declared preferences in a multi-dim profile. |
| [`frontend`](https://github.com/The-Game-Cellar/frontend)                                        | React app: dashboard, library, recommendations, explore, wildcard, game detail, profile.                                      |
| [`.github`](https://github.com/The-Game-Cellar/.github)                                          | Organization profile, top-level `docker-compose.yml`, `.env.example`, and shared documentation.                               |

## Run Locally

The fastest path is Docker Compose. Every service is dockerized; this `.github` repo holds the top-level compose file and a `.env.example` template.

### Prerequisites

- Docker 24+ and Docker Compose
- A Twitch Developer application for IGDB credentials, free at [dev.twitch.tv/console](https://dev.twitch.tv/console)

### First-time setup

```bash
# 1. Create a parent folder and clone all six repos as siblings.
mkdir the-game-cellar && cd the-game-cellar

git clone https://github.com/The-Game-Cellar/.github
git clone https://github.com/The-Game-Cellar/api-gateway
git clone https://github.com/The-Game-Cellar/game-service
git clone https://github.com/The-Game-Cellar/library-service
git clone https://github.com/The-Game-Cellar/recommendation-service
git clone https://github.com/The-Game-Cellar/frontend

# 2. Copy the env template to the parent folder and fill it in.
cp .github/.env.example .env
# Edit .env: set TWITCH_CLIENT_ID, TWITCH_CLIENT_SECRET, and the Keycloak / database passwords.

# 3. Boot everything.
docker compose -f .github/docker-compose.yml --env-file .env up -d
```

Keycloak takes ~60 seconds to become healthy on first boot. The other services wait for their databases via `depends_on: condition: service_healthy`. Once everything is up:

| URL                                  | What                                |
|--------------------------------------|-------------------------------------|
| <http://localhost:5173>              | Frontend                            |
| <http://localhost:8000>              | API Gateway (no UI; programmatic)   |
| <http://localhost:8080>              | Keycloak admin console              |

### Keycloak realm setup

The realm is not yet packaged for import (tracked separately as a launch-readiness item). After first boot, log in to the Keycloak admin console and create a realm called `game-cellar`, a public client `game-cellar-client`, and a confidential service-account client `gateway-admin` with the `realm-management/manage-users` role.

### Running a single service in your IDE

Each backend service is a standard Spring Boot project. From inside the service repo:

```bash
./mvnw spring-boot:run
```

The frontend runs against an already-running API Gateway:

```bash
cd frontend
npm install
npm run dev
```

## Configuration

Every environment variable used by the project is documented in [`.env.example`](https://github.com/The-Game-Cellar/.github/blob/main/.env.example). Spring services map them in via `${VAR:default}` syntax in their `application.yaml`. The frontend only reads keys prefixed with `VITE_` (baked at build time, not runtime).

Secrets never live in source. `.env` is gitignored. Frontend `VITE_*` values ship in the browser bundle; do not put API keys there.

## Security

- All user identity is extracted from JWT `sub`, never from request bodies.
- Tokens live in HttpOnly cookies. The frontend never holds raw JWTs in JavaScript.
- A 401 response triggers a transparent refresh-token flow, with the original request replayed once the new access token lands.
- Rate limiting via Bucket4j: per-IP on `/auth/login` + `/register`, per-user on `/recommendations/*`, with spoof-resistant client-IP derivation behind trusted proxies.
- A `gitleaks` sweep across all repository histories is part of the pre-public-launch checklist.

## Acknowledgements

- Game catalog data: [IGDB.com](https://www.igdb.com/) (Twitch Developer Services Agreement).
- Authentication: [Keycloak](https://www.keycloak.org/).
- Stack: [Spring Boot](https://spring.io/projects/spring-boot), [React](https://react.dev/), [TanStack Query](https://tanstack.com/query), [Tailwind CSS](https://tailwindcss.com/).

## License

[MIT](https://github.com/The-Game-Cellar/.github/blob/main/LICENSE) © 2026 Alexander Westerberg Svedberg
