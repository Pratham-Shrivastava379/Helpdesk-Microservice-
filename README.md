# HelpDesk

A full-stack support ticket system built with React, Spring Boot, and PostgreSQL.

HelpDesk separates authentication and user management from ticket management using two independent Spring Boot services, each with its own PostgreSQL database.

[Demo Video](https://youtu.be/6yJbNEUu_gw)

Live Demo is no longer available due to the expiry of free hosting credits.
![HelpDesk Dashboard](docs/screenshot/dashboard.png)

## What it does

HelpDesk supports three roles:

- **Users** create and manage their own support tickets.
- **Agents** work on tickets assigned to them.
- **Admins** manage users, assign tickets, and view system-wide statistics.

Tickets follow a fixed lifecycle:

`OPEN → IN_PROGRESS → RESOLVED → CLOSED`

Each ticket also has an SLA based on its priority, with the current SLA state
calculated by the backend.

## Architecture

HelpDesk is split into two Spring Boot services with separate PostgreSQL
databases.

![HelpDesk Architecture](docs/screenshot/Architecture.png)

### Why two services?

- **user-service** owns authentication, users, roles, and JWT issuance.
- **ticket-service** owns tickets, comments, notifications, SLA tracking, and assignment.
- Each service owns its own database; neither service reads the other's database directly.
- `ticket-service` makes a synchronous call to `user-service` when an agent needs to be verified for ticket assignment.
- Both services use the same JWT secret so `ticket-service` can validate tokens issued by `user-service`.

This keeps user data and ticket data under separate service boundaries while
keeping the communication between services explicit.

## Tech stack

| Layer | Technologies |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, React Router, Axios |
| Backend | Java 21, Spring Boot 3.3.4, Spring Security, Spring Data JPA |
| Database | PostgreSQL 16 |
| Authentication | JWT (HS256), JJWT |
| Deployment | Docker, Docker Compose, Nginx |
| API documentation | OpenAPI / Swagger UI |
| Testing | JUnit 5, Mockito |
## Key features

### Authentication and authorization

- JWT-based authentication with Spring Security.
- Three roles: `USER`, `AGENT`, and `ADMIN`.
- Role-based access is enforced in the backend, not only in the frontend.
- `user-service` is the only service that issues JWTs.

### Ticket management

- Create, edit, and view support tickets.
- Priority and category tracking.
- Strict ticket lifecycle:

  `OPEN → IN_PROGRESS → RESOLVED → CLOSED`

- Closed tickets are immutable.
- Tickets can be assigned to support agents.
- Comments are attached to individual tickets.

### SLA tracking

Tickets have an SLA based on their priority.

The backend calculates the current SLA state when ticket data is requested:

- `ON_TRACK`
- `AT_RISK`
- `BREACHED`

The dashboard exposes the resulting statistics for the current user's role.

### Role-based workflows

| Role | Main capabilities |
|---|---|
| USER | Create tickets, view own tickets, comment, resolve own tickets |
| AGENT | View assigned tickets, update status, comment |
| ADMIN | View all tickets, assign agents, manage users, view system statistics |

## Screenshots

### Dashboard

The dashboard changes based on the signed-in user's role and shows ticket
statistics and SLA information.

![Admin dashboard](docs/screenshot/dashboard.png)

### Ticket details

Tickets have a defined lifecycle, SLA tracking, assignment, descriptions, and
comments.

![Ticket details](docs/screenshot/ticketdetail.png)

### User management

Administrators can manage users and change their roles between `USER`, `AGENT`,
and `ADMIN`.

![User management](docs/screenshot/userspage.png)
## Quick start

### Option A — Docker Compose

Clone the repository and start the complete stack:

```bash
cp .env.example .env
```
Set `JWT_SECRET` in `.env`, then run:

```bash
docker compose up --build
```

Once the containers are running:

Frontend: http://localhost:5173
User service: http://localhost:8081
Ticket service: http://localhost:8082

Docker Compose starts the frontend, both Spring Boot services, and their PostgreSQL databases.


### Option B — Run each piece manually (best for active development)

Useful if you want hot reload on the frontend or want to iterate on one
backend service without rebuilding a container each time.

1. **[`backend/user-service/README_RUN.md`](./backend/user-service/README_RUN.md)** — start this first (Postgres + the service itself)
2. **[`backend/ticket-service/README_RUN.md`](./backend/ticket-service/README_RUN.md)** — needs user-service already running
3. **[`frontend/README_RUN.md`](./frontend/README_RUN.md)** — needs both backends already running

## Ports (local dev vs. Docker Compose — unambiguous reference)

| Service       | Docker Compose internal (container-to-container) | Host port (both `docker compose` and standalone) |
|---------------|----------------------------------------------------|----------------------------------------------------|
| user-service  | `user-service:8081`                                | `localhost:8081`                                    |
| ticket-service| `ticket-service:8082`                              | `localhost:8082`                                    |
| frontend      | —                                                    | `localhost:5173`                                    |
| user_db       | `user-db:5432`                                     | `localhost:5432`                                    |
| ticket_db     | `ticket-db:5432`                                   | `localhost:5433` *(not 5432 — avoids colliding with user_db)* |

Both backend services' `DATABASE_URL` fallback defaults (used only when
`DATABASE_URL` isn't set — i.e. running via `mvnw spring-boot:run` outside
Docker) already match the host-port column above. Docker Compose itself
never relies on those fallbacks — it always sets `DATABASE_URL` explicitly
to the internal-hostname column.

## Demo accounts

Use these seeded accounts to explore the different role-based workflows.

| Email | Role |
|---|---|
| `admin@helpdesk.dev` | ADMIN |
| `agent1@helpdesk.dev` | AGENT |
| `agent2@helpdesk.dev` | AGENT |
| `user1@helpdesk.dev` | USER |
| `user2@helpdesk.dev` | USER |
| `user3@helpdesk.dev` | USER |

**Password for all demo accounts:** `Password123!`

These accounts are seeded by `user-service` on first boot. `ticket-service` then creates demo tickets using their user IDs.
## What each role can do

| Capability      | USER                            | AGENT            | ADMIN       |
| --------------- | ------------------------------- | ---------------- | ----------- |
| Create tickets  | Own                             | –                | –           |
| View tickets    | Own tickets                     | Assigned tickets | All tickets |
| Comment         | Own tickets                     | Assigned tickets | All tickets |
| Update status   | Resolve own tickets, then close | Assigned tickets | Any ticket  |
| Assign agents   | –                               | –                | Yes         |
| Manage users    | –                               | –                | Yes         |
| View statistics | Own scope                       | Own scope        | All         |

The frontend hides actions that are not available for the current role or ticket state, but authorization is enforced by the backend service layer.

## API documentation

Both backend services expose their REST APIs through Swagger UI:

* `user-service`: `http://localhost:8081/swagger-ui.html`
* `ticket-service`: `http://localhost:8082/swagger-ui.html`

## Tests

Both backend services include JUnit 5 / Mockito unit tests covering service-layer logic, JWT handling, and SLA calculation.

Integration/Testcontainers tests are not currently included.

Run the test suites with:

```bash
cd backend/user-service && ./mvnw test
cd backend/ticket-service && ./mvnw test
```

## Project structure

```text
helpdesk/
├── docker-compose.yml
├── .env.example
├── DEPLOYMENT.md
├── PROJECT_STATUS.md
├── backend/
│   ├── user-service/       # Spring Boot service for authentication, users, and roles
│   └── ticket-service/     # Spring Boot service for tickets, comments, notifications, and SLA
├── frontend/               # React + TypeScript frontend
└── docs/
    └── screenshot/         # Project screenshots and architecture diagram
```

The backend consists of two independent Spring Boot services, while the frontend is a separate React application. Each backend service has its own PostgreSQL database.

## Known constraints

* SLA state is computed when ticket data is requested; there is no background worker.
* Ticket status transitions are strict and forward-only:
  `OPEN → IN_PROGRESS → RESOLVED → CLOSED`.
* `CLOSED` tickets are immutable.
* `user-service` is the only JWT issuer. JWT claims (`sub`, `role`, `email`) must remain compatible between both services.

See [`PROJECT_STATUS.md`](./PROJECT_STATUS.md) for the complete list and current remaining tasks.
