# 🔧 Smart DIY Garage — Backend API

> A SaaS REST API for managing personal vehicle maintenance ledgers. Built with Java 21, Spring Boot 3.4+, and deployed on AWS.

[![Java](https://img.shields.io/badge/Java-21-orange?logo=java)](https://adoptium.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4+-brightgreen?logo=springboot)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue?logo=postgresql)](https://www.postgresql.org/)
[![AWS](https://img.shields.io/badge/Deployed%20on-AWS-FF9900?logo=amazonaws)](https://aws.amazon.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

---

## 📌 Overview

Smart DIY Garage is a backend-only REST API that allows users to:

- Register and authenticate via **JWT (stateless)**
- Track their **vehicles** (make, model, year, trim, mileage)
- Maintain a **service log** per vehicle (date, service type, description)
- Enforce strict **data ownership** — users can only access their own resources

---

## 🏗️ Architecture

```
Client (Postman / Future Frontend)
        │
        ▼
   REST Controller        ← Input validation, HTTP mapping
        │
        ▼
     Service Layer        ← Business logic, ownership checks
        │
        ▼
   Repository Layer       ← Spring Data JPA queries
        │
        ▼
   PostgreSQL (AWS RDS)   ← Persistent storage
```

**Package Structure:** Package-by-Feature (not by layer)

```
com.garage
├── user/           # Registration, login, JWT issuance
├── vehicle/        # Vehicle CRUD
├── servicelog/     # Service log CRUD
├── security/       # JWT filter, Spring Security config
└── common/         # Global exception handling, shared DTOs
```

---

## ⚙️ Tech Stack

| Layer        | Technology                        | Reason                              |
|--------------|-----------------------------------|-------------------------------------|
| Language     | Java 21                           | LTS, Virtual Threads, Records       |
| Framework    | Spring Boot 3.4+                  | Industry standard                   |
| Database     | PostgreSQL 16 (AWS RDS)           | Relational, ACID-compliant          |
| ORM          | Spring Data JPA / Hibernate       | Entity mapping, query generation    |
| Security     | Spring Security + JWT (JJWT)      | Stateless auth                      |
| Migrations   | Flyway                            | Version-controlled schema changes   |
| Testing      | JUnit 5, Mockito, Testcontainers  | Unit + integration test coverage    |
| Build Tool   | Maven                             | Dependency management               |
| Deployment   | AWS EC2 + RDS, Docker             | Production-grade cloud deployment   |
| CI/CD        | GitHub Actions                    | Automated test & build pipeline     |

---

## 🚀 Getting Started (Local Development)

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker & Docker Compose (for local PostgreSQL)
- An AWS account (for production deployment)

### 1. Clone the repository

```bash
git clone https://github.com/noah-hw-kim/smart-diy-garage-backend.git
cd smart-diy-garage-backend
```

### 2. Start local PostgreSQL via Docker

```bash
docker-compose up -d
```

### 3. Configure environment variables

Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

| Variable            | Description                          |
|---------------------|--------------------------------------|
| `DB_URL`            | JDBC connection string               |
| `DB_USERNAME`       | Database username                    |
| `DB_PASSWORD`       | Database password                    |
| `JWT_SECRET`        | 256-bit secret for signing tokens    |
| `JWT_EXPIRY_MS`     | Token expiry in milliseconds         |

### 4. Run the application

```bash
./mvnw spring-boot:run
```

API is available at: `http://localhost:8080`

---

## 📡 API Endpoints

### Auth

| Method | Endpoint               | Auth Required | Description          |
|--------|------------------------|---------------|----------------------|
| POST   | `/api/v1/auth/register`| No            | Register a new user  |
| POST   | `/api/v1/auth/login`   | No            | Login, receive JWT   |

### Vehicles

| Method | Endpoint                     | Auth Required | Description                    |
|--------|------------------------------|---------------|--------------------------------|
| GET    | `/api/v1/vehicles`           | Yes (JWT)     | List all vehicles for the user |
| POST   | `/api/v1/vehicles`           | Yes (JWT)     | Add a new vehicle              |
| GET    | `/api/v1/vehicles/{id}`      | Yes (JWT)     | Get a specific vehicle         |
| PUT    | `/api/v1/vehicles/{id}`      | Yes (JWT)     | Update a vehicle               |
| DELETE | `/api/v1/vehicles/{id}`      | Yes (JWT)     | Delete a vehicle               |

### Service Logs

| Method | Endpoint                                        | Auth Required | Description                         |
|--------|-------------------------------------------------|---------------|-------------------------------------|
| GET    | `/api/v1/vehicles/{vehicleId}/logs`             | Yes (JWT)     | List all logs for a vehicle         |
| POST   | `/api/v1/vehicles/{vehicleId}/logs`             | Yes (JWT)     | Add a service log to a vehicle      |
| GET    | `/api/v1/vehicles/{vehicleId}/logs/{logId}`     | Yes (JWT)     | Get a specific log                  |
| PUT    | `/api/v1/vehicles/{vehicleId}/logs/{logId}`     | Yes (JWT)     | Update a log                        |
| DELETE | `/api/v1/vehicles/{vehicleId}/logs/{logId}`     | Yes (JWT)     | Delete a log                        |

---

## 🔒 Security Model

- All protected endpoints require a valid JWT in the `Authorization: Bearer <token>` header
- JWT is stateless — no session stored server-side
- **Ownership enforcement:** The authenticated user's ID is extracted from the JWT and compared against the resource's owner ID at the **service layer**. Accessing another user's resource returns `403 Forbidden`

---

## 🧪 Running Tests

```bash
# Unit tests only
./mvnw test

# Full integration tests (requires Docker for Testcontainers)
./mvnw verify
```

---

## ☁️ Deployment

This application is deployed on AWS. See [`docs/DEPLOYMENT.md`](./docs/DEPLOYMENT.md) for the full guide.

**Infrastructure at a glance:**
- **EC2 (t2.micro)** — Runs the Dockerized Spring Boot application
- **RDS (db.t3.micro, PostgreSQL)** — Managed database in a private subnet
- **GitHub Actions** — Runs tests on every push; builds and ships a Docker image on merge to `main`

---

## 📂 Project Structure

```
smart-diy-garage-backend/
├── src/
│   ├── main/
│   │   ├── java/com/garage/
│   │   │   ├── user/
│   │   │   ├── vehicle/
│   │   │   ├── servicelog/
│   │   │   ├── security/
│   │   │   └── common/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/      ← Flyway SQL scripts
│   └── test/
├── docs/
│   ├── PLANNING.md                ← Development roadmap
│   └── DEPLOYMENT.md              ← AWS deployment guide (TBD)
├── docker-compose.yml
├── Dockerfile
├── .env.example
└── README.md
```

---

## 🗺️ Roadmap

- [x] Project planning & architecture design
- [ ] Phase 1: Core domain (User, Vehicle, ServiceLog CRUD)
- [ ] Phase 2: JWT Authentication & Authorization
- [ ] Phase 3: AWS Deployment (EC2 + RDS + Docker)
- [ ] Phase 4: CI/CD Pipeline (GitHub Actions)
- [ ] Phase 5 (Future): AI-powered maintenance recommendations

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file for details.
