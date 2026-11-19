# Project Foundation — Tasks

> Depends on: [requirements.md](./requirements.md) (all questions answered; decisions in section 9).
> Rule: a task is only checked after its verification commands were executed and passed.
> All commands run through Docker; nothing requires PHP, Composer or PostgreSQL on the host.

## T-01 — Define repository layout and ignore rules

- [x] **Done** (verified: directories listed, `.gitignore` rules present, `.env.example` not ignored)

**Objective:** Create the base directory structure defined in `CLAUDE.md` and ignore rules.

**Expected result:** Directories `src/`, `docker/php/`, `docker/nginx/`, `docker/postgres/init/` exist; `src/` contains only a `.gitkeep` until T-07; `.gitignore` ignores `.env`, `src/.env*` and local artifacts (not `.env.example`).

**Verification:**
- `ls -d src docker/php docker/nginx docker/postgres/init` lists all four.
- `.gitignore` contains `.env` and `src/.env*`; `git check-ignore .env.example` returns nothing.

---

## T-02 — Create environment variable template

- [ ] **Done**

**Objective:** Document all configuration through environment variables.

**Expected result:** `.env.example` lists every variable from requirements section 5 (including `DB_TEST_DATABASE`, `HOST_UID=1000`, `HOST_GID=1000`, `LOG_CHANNEL=stderr`, cache/session/queue drivers) with safe placeholder values and comments; no real secrets.

**Verification:**
- `.env.example` contains all variables from requirements section 5.
- `grep -i password .env.example` shows only placeholders.

---

## T-03 — PHP 8.4 container image (with Composer)

- [ ] **Done**

**Objective:** Build a `php:8.4-fpm` (Debian) image with the extensions Laravel + PostgreSQL need and Composer, running as a non-root user created from `HOST_UID`/`HOST_GID` build args.

**Expected result:** `docker/php/Dockerfile` builds successfully; image includes `pdo_pgsql` and Composer.

**Verification:**
- `docker build -f docker/php/Dockerfile --build-arg HOST_UID=$(id -u) --build-arg HOST_GID=$(id -g) -t notification-app:dev .` succeeds.
- The Dockerfile sets `WORKDIR /var/www/html`.
- `docker run --rm notification-app:dev php -v` shows PHP 8.4.x.
- `docker run --rm notification-app:dev php -m | grep pdo_pgsql` returns a match.
- `docker run --rm notification-app:dev composer --version` works.
- `docker run --rm notification-app:dev id -u` equals `$(id -u)` of the host user and is not `0`.

---

## T-04 — Nginx configuration

- [ ] **Done**

**Objective:** Configure Nginx to serve Laravel's `public/` and forward PHP requests to the `app` service.

**Expected result:** `docker/nginx/default.conf` using a pinned `nginx:<major.minor>` image (recorded in requirements AC-08), with `root /var/www/html/public`, `fastcgi_pass app:9000`, and front-controller `try_files`.

**Verification:**
- `docker run --rm -v $PWD/docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro nginx:<pinned-tag> nginx -t` reports syntax OK (the `app` upstream may need to be resolved in Compose; if so, this check moves to T-06).

---

## T-05 — Docker Compose: network, PostgreSQL and volume

- [ ] **Done**

**Objective:** Start PostgreSQL via Compose with persistent storage and a health check.

**Expected result:** `docker-compose.yml` defines the `db` service (image `postgres:17`, never `latest`; no published port), a second database for tests created on first start by `docker/postgres/init/01-create-test-db.sh` (only runs on an empty volume), the named volume `notification-pgdata`, the network `notification-net`, a `pg_isready` health check, and credentials mapped from `.env` (`DB_*` → `POSTGRES_*`).

**Verification:**
- `docker compose config` is valid.
- `docker compose up -d db` then `docker compose ps` shows `db` healthy.
- `docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "select 1"'` returns 1 (variables are expanded inside the container, not on the host).
- `docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d <DB_TEST_DATABASE value> -c "select 1"'` returns 1.
- `docker compose port db 5432` shows nothing published.
- `docker compose exec db postgres --version` reports 17.x.
- `grep -n "postgres:" docker-compose.yml` shows `postgres:17`.
- `docker volume ls` shows `notification-pgdata` (with the compose project prefix).

---

## T-06 — Docker Compose: `app` and `nginx` services

- [ ] **Done**

**Objective:** Add `app` and `nginx` services to the same network, with `./src` mounted and `app` depending on a healthy `db`.

**Expected result:** `docker compose up -d --build` starts all three services (Laravel is not installed yet, so only container startup is checked, not HTTP responses); `app` waits for a healthy `db` and `nginx` starts after `app`; only `nginx` publishes a host port.

**Verification:**
- `docker compose ps` shows `app`, `nginx`, `db` running.
- `docker compose exec app php -v` shows 8.4.x.
- `docker compose port nginx 80` shows the configured host port; `db` and `app` publish none.
- `docker compose exec nginx getent hosts app` resolves the service name.

---

## T-07 — Bootstrap Laravel in `./src`

- [ ] **Done**

**Objective:** Create the Laravel project inside the `app` container with a one-off `docker compose run --rm app composer create-project`, with no host Composer.

**Expected result:** Laravel project present in `./src` (the `.gitkeep` is handled so the directory is accepted by `create-project`, e.g. install into a temporary directory and move the contents); files are owned by the host user (see Q8); `src/.env` is removed and `APP_KEY` from `php artisan key:generate --show` is copied into the root `.env`; the installed Laravel major version is recorded in requirements section 9.

**Verification:**
- `docker compose exec app php artisan --version` prints the Laravel version.
- `ls src/artisan src/composer.json` exist.
- `ls -l src/artisan` shows the host user as owner.

---

## T-08 — Wire Laravel to environment variables and PostgreSQL

- [ ] **Done**

**Objective:** Make Laravel read its configuration from the container environment and connect to `db`.

**Expected result:** Laravel uses `pgsql` with `DB_HOST=db`; `LOG_CHANNEL=stderr`; `phpunit.xml` sets `DB_DATABASE` to `DB_TEST_DATABASE`; `APP_KEY` is generated; `CACHE_STORE=database`, `SESSION_DRIVER=database`, `QUEUE_CONNECTION=sync`; the cache and sessions tables are migrated.

**Verification:**
- `docker compose exec app php artisan config:show database.default` prints `pgsql`.
- `docker compose exec app php artisan migrate` succeeds.
- Changing `DB_DATABASE` in `.env` and recreating containers changes the connected database, with no Docker file edited.

---

## T-09 — HTTP end-to-end through Nginx

- [ ] **Done**

**Objective:** Prove the full request path host → nginx → app works.

**Expected result:** Laravel responds through Nginx on the configured host port.

**Verification:**
- `curl -i http://localhost:<NGINX_HTTP_PORT>/` returns HTTP 200.
- `curl -i http://localhost:<NGINX_HTTP_PORT>/up` (Laravel health route) returns HTTP 200.
- `docker compose logs nginx` shows the requests.

---

## T-10 — Data persistence check

- [ ] **Done**

**Objective:** Prove PostgreSQL data survives container recreation.

**Expected result:** Data created before `docker compose down` exists after `docker compose up`.

**Verification:**
- `docker compose exec app php artisan migrate` (creates the `migrations` table).
- Insert a marker row: `docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "create table persist_check(id int); insert into persist_check values (1);"'`.
- `docker compose down` (without `-v`), then `docker compose up -d --build`.
- The same `psql` with `select * from persist_check;` returns the row, and `\dt` still lists the Laravel tables.

---

## T-11 — Run the test suite inside Docker

- [ ] **Done**

**Objective:** Confirm PHPUnit runs inside the container against the separate test database (`DB_TEST_DATABASE`).

**Expected result:** The tests pass against `DB_TEST_DATABASE`, without touching the development data. Because the default Laravel tests don't use the database, add one temporary test that writes a row (e.g. with `RefreshDatabase`).

**Verification:**
- `docker compose exec app php artisan test` exits 0.
- Row counts of the development database tables (`select count(*)` on `migrations`, `cache`, `sessions`) are identical before and after the run, and the temporary test's row exists only in `DB_TEST_DATABASE` during the run.

---

## T-12 — Write ADR-001 (database) and ADR-002 (notification processing)

- [x] **Done** (verified: see commands below)

**Objective:** Record the architectural decisions required by Q11.

**Expected result:**
- [ADR-001](../../architecture/decisions/ADR-001-database.md): PostgreSQL pinned to major 17 (`postgres:17`) in Docker with a named volume, with context, alternatives and consequences.
- [ADR-002](../../architecture/decisions/ADR-002-notification-processing.md): synchronous processing in the HTTP lifecycle first, then asynchronous background workers via a message queue, with trade-offs and stateless-instance consequences.
- [architecture README](../../architecture/README.md) indexes both ADRs.

**Verification:**
- `ls docs/architecture/decisions docs/architecture/README.md` lists both ADRs and the README.
- `grep -c "^## " docs/architecture/decisions/ADR-00*.md` shows Context, Decision, Alternatives Considered and Consequences in each.
- `grep -rn "postgres:latest" docs/architecture` returns nothing, and every `postgres:` tag mentioned is `postgres:17`.
- ADR-002 does not pick a specific queue technology (that needs its own spec and approval).

---

## T-13 — Document how to run the environment

- [ ] **Done**

**Objective:** Make the setup reproducible by anyone with only Docker installed.

**Expected result:** `README.md` documents: prerequisites (Docker only), first-time setup, start/stop, running artisan/composer/tests via `docker compose exec`, and how to reset the database volume.

**Verification:**
- A fresh clone, following only the README, reaches the state of T-09 without extra steps.

---

## T-14 — Final acceptance review

- [ ] **Done**

**Objective:** Validate every acceptance criterion in requirements section 6 and confirm scope boundaries.

**Expected result:** All AC-1…AC-15 verified with commands and results recorded.

**Verification:**
- Each acceptance criterion executed and its output reported.
- `grep -ril "redis\|kubernetes" docker docker-compose.yml .env.example src` returns nothing (docs are excluded; see AC-12).
- Every step above passes on a host where `php`, `composer` and `psql` are absent (or are not invoked).

---

## Suggested Order

`T-01 → T-02 → T-03 → T-04 → T-05 → T-06 → T-07 → T-08 → T-09 → T-10 → T-11 → T-12 → T-13 → T-14`

T-12 (ADRs) has no technical dependencies; it was completed first.

Each task is independently verifiable once its predecessors are complete.
