# Project Overview
- This project is a notification API for sales system. When an order is completed, the sales application send a notification request to this API.

- The API is responsible for:
    - Receiving notification requests.
    - Validating incoming data.
    - Persisting notification records in the database.
    - Managing notification statuses.
    - Processing notification deliveries through a simulated email service.
    - Return appropriete HTTP responses.

Initial architecture: Notification delivery will be synchronous, meaning the API will handle the entire process within the HTTP request lifecycle.

Future architecture: Notification delivery will be delegated to background workers throught an asyncronous message queue, allowing the application to scale horizontally and process notifications independently of incoming HTTP requests.

## Tech Stack (will be improve)
- **Language / Framework:**  PHP / Laravel
- **Database / ORM:** PostgreSQL
- **Testing:** PHPUNIT
- **Infra:** Docker

# Docker & Enviroment Rules

- PHP 8.4 must run exclusively inside Docker.
- Compose must run inside Docker.
- Laravel Artisan commands must run inside Docker.
- PostgresSQL must run inside Docker.
- All application dependencies must be containerized.
- Never require PHP Composer or PostgreSQL on the host machine.
- Use Docker Compose for local orchestration.
- Application source code must reside in ./src.
- Persist PostgresSQL data using Docker volumes.

# Scalability Principles

- Design the application to support multiple instances.
- Aplication instances must be stateless.
- Never store critical application state exclusively in local memory.
- Never depend on local container filesystem for persisten data.
- Shared state must use external infrastructure.
- Database connections must be use environment configuration.
- Background processing must be independent of HTTP request lifecycle.
- Avoid introducing distributed architecture components before their correspoding specification.

## IA Collaboration Rules

- Explain architectural decisions before implementation.
- Do not introduce new technologies without approval.
- Do not install unnecessary dependencies.
- Do not implement future stages prematurely.
- Prefer simple solutions over unnecesary abstractions.
- Explain trade-offs when multiple solutions are possible.
- Ask before changing existing architecture.
- Never mark a task as complete without executing its acceptance criteria.
- Report commands executed and test results.
- Never read the prompt-flow.md

## Project dir

### docs

docs/
├── specs/
│   └── project-foundation/
│       ├── requirements.md
│       └── tasks.md
│
└── architecture/
    ├── README.md
    └── decisions/
        ├── ADR-001-database.md
        └── ADR-002-notification-processing.md

### all

notification-plataform/
│
├── src/
│   └── Laravel
│
├── docker/
│   ├── php/
│   │   └── Dockerfile
│   ├── nginx/
│   │   └── default.conf
│   └── postgres/
│       └── init/
│           └── 01-create-test-db.sh
│
├── docs/
│   ├── specs/
│   └── architecture/
│       └── decisions/
│
├── .claude/
│   └── ...
│
├── CLAUDE.md
├── docker-compose.yml
├── .env.example
└── README.md

## Architecture & Code Conventions
- Keep the bussiness rules  out of route handles/controllers. Use Service for bussiness rules.
- validate all external inputs at system boundaries using.
- Write unit tests for new service function s and integration test for new endpoints
- Using "Clean Code" concept.

## Active Specifications

- [project-foundation](docs/specs/project-foundation/requirements.md) — containerized base infrastructure (tasks: [tasks.md](docs/specs/project-foundation/tasks.md))
