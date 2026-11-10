# School Admission System

A Dockerized first-come, first-served school admission portal built with **.NET 8**, **PostgreSQL**, **React 18 + TypeScript**, and **Docker Compose**.

## Run with Docker

Requirements: Docker Desktop / Docker Engine with the Compose plugin. No local .NET, Node.js, or PostgreSQL installation is required.

```bash
cp .env.example .env
# Optional: edit .env and set a strong local database password
docker compose up --build
```

- Frontend: http://localhost:5173
- Backend API: http://localhost:8080
- Health: http://localhost:8080/health
- Swagger: http://localhost:8080/swagger (available in development; use the API routes below if disabled in production)

Stop containers while retaining database data:

```bash
docker compose down
```

Remove containers **and permanently delete local database data**:

```bash
docker compose down -v
```

## User guide

### Student
1. Open the frontend and choose **Student portal**.
2. Enter the student's 17-digit numeric birth registration number. No account login is required by the assignment.
3. Select an available class/shift and choose **Apply for this class**.
4. The backend reserves one seat for the configured hold duration (default 10 minutes).
5. Confirm admission before the countdown ends. Confirmation simulates successful admission-fee payment; no real payment gateway is connected.
6. Application history is restored after refresh or browser restart because it is retrieved from PostgreSQL using the student ID.

### School admin
1. Choose **School admin**.
2. Enter a class/shift name and total seats, then create the class.
3. Review held, confirmed, and available seat counts. Each class/shift has independent capacity.

> The brief does not define administrator authentication. Class creation is intentionally open in this assessment demo; do not expose it publicly in production without authorization.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `POSTGRES_DB` | `school_admission` | PostgreSQL database |
| `POSTGRES_USER` | `postgres` | PostgreSQL user |
| `POSTGRES_PASSWORD` | `postgres` | Local database password; change it |
| `HOLD_DURATION_SECONDS` | `600` | Seat hold duration |
| `VITE_API_BASE_URL` | `http://localhost:8080` | Browser-accessible API URL (frontend build-time value) |

Copy `.env.example` to `.env`. Rebuild the frontend image after changing `VITE_API_BASE_URL`.

## API contract

IDs are returned as `c_<guid>` for classes and `h_<guid>` for holds. Student-specific endpoints require `X-Student-Id` with a 17-digit numeric value.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/classes` | Create a class/shift; body: `{ "name": "Class 6 - Morning Shift", "totalSeats": 120 }` |
| `GET` | `/classes` | List classes with held, confirmed, and available seats |
| `GET` | `/classes/{classId}` | Get one class and seat counts |
| `POST` | `/classes/{classId}/holds` | Reserve a seat; requires `X-Student-Id`; returns `201`, or `409` if unavailable |
| `GET` | `/holds/{holdId}` | Get hold status |
| `POST` | `/holds/{holdId}/confirm` | Confirm simulated fee payment; requires `X-Student-Id` |
| `DELETE` | `/holds/{holdId}` | Cancel a pending hold; requires `X-Student-Id`; returns `204` |
| `GET` | `/me/holds` | List applications for the student; requires `X-Student-Id` |

Hold statuses: `HELD`, `CONFIRMED`, `EXPIRED`, `CANCELLED`.

## Concurrency and persistence

The reservation flow starts a serializable PostgreSQL transaction and locks the target class row with `SELECT ... FOR UPDATE`. Requests competing for the same class are therefore serialized before the service counts occupied seats and creates a hold. Capacity counts include only non-expired `HELD` holds and `CONFIRMED` admissions. Expired holds are marked `EXPIRED` on API activity, while capacity calculations exclude them by timestamp even before the status update. PostgreSQL uses a named Docker volume so records survive `docker compose down` and `docker compose up`.

## Tests

The functional API suite checks class creation and capacity, duplicate active-application rejection, idempotent confirmation, and cancellation. The concurrency smoke test launches simultaneous reservation requests against a class and asserts that no more than the configured capacity succeeds. Start the stack first, then run:

```bash
python tests/api_tests.py
python tests/concurrency_test.py --base-url http://localhost:8080 --seats 5 --requests 30
```

The script creates a temporary class, sends parallel requests with distinct student IDs, and verifies the successful hold count does not exceed capacity. Python 3 is required only to run this optional test; it is not required to run the app.

## Repository structure

```text
backend/SchoolAdmission.Api/  ASP.NET Core API and EF Core data layer
frontend/                    React + TypeScript UI
 tests/                      Automated concurrency smoke test
 docker-compose.yml           Full-stack orchestration
 ASSUMPTIONS.md               Decisions for unspecified requirements
 AI_USAGE.md                  AI assistance disclosure
 README.md                    Setup and API documentation
```
