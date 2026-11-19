# ADR-001: Database Selection (PostgreSQL 17)

- **Status:** Accepted
- **Date:** 2026-11-19
- **Related:** [project-foundation requirements](../../specs/project-foundation/requirements.md), [ADR-002](./ADR-002-notification-processing.md)

## Context

The Notification API persists notification records and manages their statuses. This requires reliable transactional storage, constraints and indexing. The database must:

- run exclusively inside Docker (no PostgreSQL on the host, per `CLAUDE.md`);
- keep its data in a Docker volume;
- be shared by multiple stateless application instances in future stages;
- be configured only through environment variables;
- be well supported by Laravel.

## Decision

Use **PostgreSQL, pinned to major version 17**.

- Docker image tag: `postgres:17`. Never `latest` and never an unpinned tag.
- Runs as the `db` service in Docker Compose.
- Data is persisted in a Docker named volume.
- The port is not published to the host. Access is through the Docker network (`db:5432`) or `docker compose exec db psql`.
- Laravel connects with the `pgsql` driver; host, port, database and credentials come from environment variables.
- Tests use a second database in the same PostgreSQL instance.
- Minor and patch updates follow the `17` tag; this means builds are not byte-reproducible and is accepted for the development stage. Pin a minor version in a deployment spec if reproducibility is needed. A major version change requires a new ADR.
- The test database is created by an init script (`docker/postgres/init/`), which only runs on an empty volume.

## Alternatives Considered

| Option | Why not chosen |
|---|---|
| MySQL / MariaDB | Viable, but PostgreSQL was chosen in the project stack and offers stricter constraints and richer types. |
| SQLite | No parity with the target environment, file-based local state, not shareable between application instances. |
| NoSQL (e.g. MongoDB) | Notification data is relational with clear status transitions; this adds a new technology without need. |

## Consequences

- **Positive:** a single mature database, strong transactional guarantees, native Laravel support, safe for horizontal scaling of the application.
- **Negative:** PostgreSQL must be started (and healthy) before the application; `pdo_pgsql` must be installed in the PHP image.
- **Upgrades:** moving to another major version requires an explicit decision, a dump/restore or `pg_upgrade`, and a new ADR.
- **Data safety:** `docker compose down -v` deletes the volume and all data; this must be documented in the README.
