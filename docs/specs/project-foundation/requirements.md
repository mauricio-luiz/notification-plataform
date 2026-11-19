# Project Foundation — Requirements

> Status: APPROVED — all questions in section 8 are answered; decisions are consolidated in section 9.
> Source of truth: `CLAUDE.md`.

## 1. Project Objective

Provide the containerized base infrastructure on which the Notification API (PHP / Laravel / PostgreSQL) will be built. At the end of this specification, a developer with only Docker installed can start the whole environment with one command and reach a running Laravel application behind Nginx, connected to a PostgreSQL database with persistent data.

This specification contains **infrastructure only**. No notification business logic is part of it.

## 2. Technical Requirements

| #     | Requirement                                                            |
| ----- | ---------------------------------------------------------------------- |
| TR-01 | PHP 8.4 runs exclusively inside a Docker container.                    |
| TR-02 | Laravel runs inside the PHP container.                                 |
| TR-03 | Composer is available inside the PHP container and never required on the host (Q1). |
| TR-04 | PostgreSQL 17 (image tag `postgres:17`, never `latest`) runs inside a Docker container. See [ADR-001](../../architecture/decisions/ADR-001-database.md). |
| TR-05 | Nginx acts as the HTTP server and reverse proxy in front of PHP.       |
| TR-06 | Docker Compose orchestrates all services.                              |
| TR-07 | Application source code lives in `./src`.                              |
| TR-08 | PostgreSQL data is persisted using a Docker named volume.              |
| TR-09 | Application configuration is managed through environment variables.    |
| TR-10 | Laravel uses `database` for cache and sessions and `sync` for the queue (Q12). Logs go to `stderr` (`LOG_CHANNEL=stderr`) so no persistent data is written to the container filesystem. |
| TR-11 | Two ADRs exist: [ADR-001](../../architecture/decisions/ADR-001-database.md) (database selection, PostgreSQL 17) and [ADR-002](../../architecture/decisions/ADR-002-notification-processing.md) (synchronous processing first, asynchronous background workers later) (Q11). |

### Architecture Constraints

| #     | Constraint                                                                                                                |
| ----- | ------------------------------------------------------------------------------------------------------------------------- |
| AC-01 | No PHP, Composer or PostgreSQL installation on the host machine.                                                          |
| AC-02 | All application commands (artisan, composer, tests) run inside Docker containers.                                         |
| AC-03 | Services communicate through a Docker network.                                                                            |
| AC-04 | Internal communication uses Docker Compose service names (e.g. `db`, `app`), never IPs or `localhost` between containers. |
| AC-05 | Keep the infrastructure simple; no unnecessary abstractions.                                                              |
| AC-06 | The design must allow multiple application instances in future stages (stateless app, shared state in external services). |
| AC-07 | Redis, queues, workers and Kubernetes are **not** introduced in this specification (no code, image, compose service or env variable; documentation may mention them as future work). |
| AC-08 | Image pinning policy: every image uses an explicit version tag (`php:8.4-fpm`, `postgres:17`, `nginx:<major.minor>` chosen at T-04 and recorded here). Never `latest` and never the floating `stable` tag. |

## 3. Infrastructure Components and Responsibilities

| Service (proposed name) | Responsibility                                                                                             | Notes                                                           |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `nginx`                 | Receives HTTP requests from the host, serves as reverse proxy, forwards PHP requests to `app` via FastCGI. | Only service exposing an HTTP port to the host.                 |
| `app`                   | PHP 8.4-FPM container with Laravel and Composer. Runs the application, artisan and tests.                  | Stateless; no persistent data on its filesystem. `WORKDIR` and mount point: `/var/www/html` (same path in `nginx`). |
| `db`                    | PostgreSQL 17 (`postgres:17`). Stores application data.                                                    | Data in a named volume. Port not published to the host (Q5).    |

Supporting artifacts (to be created in later tasks, **not** in this spec-writing step):

- `docker/php/Dockerfile`
- `docker/nginx/default.conf`
- `docker/postgres/init/01-create-test-db.sh` (creates `DB_TEST_DATABASE` on first start of an empty volume)
- `docker-compose.yml`
- `.env.example`
- `README.md` (how to start the environment)
- `src/` (Laravel project)

## 4. Container Communication

```
Host :<NGINX_HTTP_PORT> ──► nginx ──(FastCGI, app:9000)──► app ──(TCP, db:5432)──► db
                                                                              │
                                                                       named volume
```

- All services join a single user-defined Docker network named `notification-net`; the PostgreSQL volume is named `notification-pgdata`.
- Start order: `app` waits for a healthy `db` (`depends_on: condition: service_healthy`); `nginx` starts after `app` (`depends_on`). Only `db` has a health check (`pg_isready`).
- `nginx` reaches PHP through the hostname `app`.
- `app` reaches PostgreSQL through the hostname `db` (`DB_HOST=db`).
- Only `nginx` publishes a port to the host by default.
- `app` and `nginx` must share the application code (bind mount of `./src` in development).

## 5. Environment Configuration

- All configuration comes from environment variables; nothing is hardcoded in Dockerfiles or Compose files.
- `.env.example` is versioned and documents every variable; the real `.env` is git-ignored.
- There is a **single** `.env`, at the repository root. Compose reads it, and `app` receives it through `env_file`/`environment`. Real container variables take precedence over any Laravel `.env` file.
- `src/.env` (created by `composer create-project`) is deleted after bootstrap and `src/.env*` is git-ignored. `APP_KEY` is generated with `php artisan key:generate --show` and copied into the root `.env`.
- Compose maps `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` to `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` for the `db` service.
- Minimum variables:

| Variable                                     | Purpose                                                |
| -------------------------------------------- | ------------------------------------------------------ |
| `APP_ENV`, `APP_DEBUG`, `APP_KEY`, `APP_URL` | Laravel application settings                           |
| `DB_CONNECTION=pgsql`                        | Laravel database driver                                |
| `DB_HOST=db`                                 | PostgreSQL service name                                |
| `DB_PORT=5432`                               | PostgreSQL port                                        |
| `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`  | PostgreSQL credentials (also used by the `db` service) |
| `NGINX_HTTP_PORT` | Host port published by Nginx (default `8080`) |
| `DB_TEST_DATABASE` | Separate database for PHPUnit, same PostgreSQL instance (Q9) |
| `CACHE_STORE=database` | Laravel cache driver (Q12) |
| `SESSION_DRIVER=database` | Laravel session driver (Q12) |
| `QUEUE_CONNECTION=sync` | Laravel queue driver (Q12) |
| `HOST_UID`, `HOST_GID` | Build args so `app` runs as a non-root user matching the host user (Q8). `.env.example` default: `1000`; fill in with `id -u` / `id -g`. |
| `LOG_CHANNEL=stderr` | Laravel log channel (no log files in the container) |

- Secrets must never be committed.

## 6. Acceptance Criteria

- [ ] AC-1: `docker compose up -d --build` starts `nginx`, `app` and `db` with no errors (after Laravel is bootstrapped; before T-07 only container startup is checked).
- [ ] AC-2: `docker compose ps` shows all services running (and `db` healthy).
- [ ] AC-2b: `docker compose exec db postgres --version` reports PostgreSQL 17.x, and the Compose file uses `postgres:17`.
- [ ] AC-3: `docker compose exec app php -v` reports PHP 8.4.x.
- [ ] AC-4: `docker compose exec app composer --version` works.
- [ ] AC-5: `docker compose exec app php artisan --version` reports the Laravel version.
- [ ] AC-6: `curl http://localhost:<NGINX_HTTP_PORT>` returns an HTTP 200 from Laravel through Nginx.
- [ ] AC-7: `docker compose exec app php artisan migrate` succeeds against PostgreSQL (`db` host).
- [ ] AC-8: After `docker compose down` (without `-v`) and `up` again, previously migrated data is still present.
- [ ] AC-9: Laravel source code exists under `./src`; no application code outside it.
- [ ] AC-10: `php`, `composer` and `psql` are not required on the host (verified by running every step above using only Docker).
- [ ] AC-11: Changing a variable in `.env` changes the behavior without editing any Docker/Compose file.
- [ ] AC-12: No Redis, queue worker or Kubernetes artifacts exist in `docker/`, `docker-compose.yml`, `.env.example` or `src/` (mentions in `docs/` and `CLAUDE.md` are allowed).
- [ ] AC-13: `docs/architecture/decisions/ADR-001-database.md`, `ADR-002-notification-processing.md` and `docs/architecture/README.md` exist and agree with this specification on: PostgreSQL major 17 (`postgres:17`), port not published, named volume, synchronous first / asynchronous later, and no queue technology chosen.
- [ ] AC-14: Files created by container commands in `./src` are owned by the host user.
- [ ] AC-15: PHPUnit runs against the separate test database (`DB_TEST_DATABASE`, set in `phpunit.xml`); a test that writes to the database leaves the row counts of the development database unchanged.

## 7. Out of Scope

- Any notification feature (endpoints, models, services, statuses, simulated email).
- Redis, message queues, background workers, schedulers.
- Kubernetes, Swarm or any other orchestrator.
- Load balancing across multiple `app` instances (only _allowing_ it later).
- HTTPS/TLS, production hardening, CI/CD pipelines.
- Monitoring, centralized logging, backups.
- Authentication/authorization.
- Test suite content beyond verifying that PHPUnit runs inside the container.

## 8. Questions and Answers

All questions below were answered by the project owner.

### Q1 — "Compose" inside the container
- **Question:** Did you mean **Composer** (the PHP package manager) inside the PHP container? `CLAUDE.md` also says "Compose must run inside Docker", which, read literally, means Docker-in-Docker.
- **Why it matters:** Docker-in-Docker is a major architectural decision and conflicts with simplicity.
- **Suggested default:** Composer inside `app`. Docker Compose runs on the host's Docker engine, so Docker is the only host requirement.
- **Answer:** You're right! I made a spelling mistake. I meant Composer, the PHP dependency manager. Composer commands must run inside Docker, not on the host machine.

### Q2 — Laravel version
- **Question:** Which Laravel version?
- **Why it matters:** It must be compatible with PHP 8.4.
- **Suggested default:** Latest stable Laravel that supports PHP 8.4.
- **Answer:** Latest stable Laravel that supports PHP 8.4. The resulting major version is recorded in section 9 after T-07, so the setup is reproducible.

### Q3 — Nginx entry point and host port
- **Question:** Is Nginx the only entry point, and which host port does it use?
- **Why it matters:** Avoids port conflicts on the host.
- **Suggested default:** `8080`, configurable via `NGINX_HTTP_PORT`.
- **Answer:** default 8080 configurable via `NGINX_HTTP_PORT`.

### Q4 — PHP-FPM base image
- **Question:** Official `php:8.4-fpm` (Debian) or `-alpine`?
- **Why it matters:** Image size versus compatibility with extensions such as `pdo_pgsql`.
- **Suggested default:** `php:8.4-fpm` (Debian), for fewer surprises.
- **Answer:** PHP:8.4-fpm Debian.

### Q5 — PostgreSQL port exposure
- **Question:** Should PostgreSQL's port be exposed to the host (for GUI clients)?
- **Why it matters:** Security and port conflicts.
- **Suggested default:** Not exposed. Access via `docker compose exec db psql`.
- **Answer:** Access only via docker compose exec.

### Q6 — PostgreSQL version
- **Question:** Which PostgreSQL version?
- **Why it matters:** It pins the choice recorded in ADR-001-database.
- **Suggested default:** Latest stable major (e.g. 17), pinned to a major tag.
- **Answer:** Latest stable major 17. Pinned to major 17 and recorded in ADR-001.

### Q7 — Source mounting strategy
- **Question:** Bind mount `./src` (development) or copy it into the image?
- **Why it matters:** Affects statelessness and future deployment.
- **Suggested default:** Bind mount for development. Copying into the image is deferred to a deployment spec.
- **Answer:** Bind mount `./src`. Copying into the image is deferred to a deployment spec.

### Q8 — File permissions
- **Question:** How do we avoid root-owned files in `./src` when container commands create them?
- **Why it matters:** Developer experience on Linux.
- **Suggested default:** Run `app` as a non-root user, with host UID/GID passed as build args.
- **Answer:** Run `app` as a non-root user, with host UID/GID passed as build args.

### Q9 — Test database
- **Question:** Should PHPUnit use a separate database?
- **Why it matters:** Tests must not wipe development data.
- **Suggested default:** A second database in the same PostgreSQL instance. SQLite is not used, to keep parity with production.
- **Answer:** A second database in the same PostgreSQL instance.

### Q10 — Creating Laravel without host Composer
- **Question:** How is Laravel created the first time?
- **Why it matters:** Bootstrapping chicken-and-egg problem.
- **Suggested default:** One-off `docker compose run --rm app composer create-project` after the image is built.
- **Answer:** One-off `docker compose run --rm app composer create-project` after the image is built.

### Q11 — ADRs referenced in CLAUDE.md
- **Question:** Should `docs/architecture/decisions/ADR-001` and `ADR-002` be written now?
- **Why it matters:** Decisions on database and processing may be expected as ADRs.
- **Suggested default:** Out of scope here. The database choice is already fixed by `CLAUDE.md`.
- **Answer:** Yes, it can be written now: ADR-001 for the database selection and ADR-002 for the processing model (synchronous first, then asynchronous). Done: see `docs/architecture/decisions/`.

### Q12 — Laravel drivers without Redis
- **Question:** Does "no Redis" also apply to the `CACHE_STORE`, `SESSION_DRIVER` and `QUEUE_CONNECTION` defaults?
- **Why it matters:** File-based sessions and cache violate the "no local filesystem state" principle when scaling.
- **Suggested default:** `database` (or `array`) for cache and session, and `sync` for the queue.
- **Answer:** Initial configuration:
CACHE_STORE	    database
SESSION_DRIVER    database
QUEUE_CONNECTION  sync

## 9. Consolidated Decisions

- Composer (not Docker Compose) runs inside the `app` container; Docker Compose runs on the host's Docker engine.
- Latest stable Laravel supporting PHP 8.4; the installed version is recorded here after T-07: _(pending)_.
- Single root `.env`; `src/.env` is removed after bootstrap; `LOG_CHANNEL=stderr`.
- Test database created by `docker/postgres/init/01-create-test-db.sh`; it only runs on an empty volume, so an existing volume needs a one-time reset (`docker compose down -v`) or a manual `CREATE DATABASE`.
- Images are pinned to explicit tags (see AC-08).
- Nginx is the only entry point; host port `8080` via `NGINX_HTTP_PORT`.
- Image base: `php:8.4-fpm` (Debian).
- PostgreSQL pinned to major version 17 (`postgres:17`), never exposed to the host; access only via `docker compose exec`. A major version change requires a new ADR.
- `./src` is bind-mounted; copying into the image is deferred to a deployment spec.
- `app` runs as a non-root user with host UID/GID passed as build args.
- Tests use a second database in the same PostgreSQL instance.
- Laravel is created with a one-off `docker compose run --rm app composer create-project`.
- ADR-001 (database selection) and ADR-002 (synchronous first, asynchronous workers later) are written and accepted; the queue technology is deliberately not decided.
- Initial Laravel drivers: `CACHE_STORE=database`, `SESSION_DRIVER=database`, `QUEUE_CONNECTION=sync`.
