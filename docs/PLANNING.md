# 📋 Project Planning — Smart DIY Garage Backend

> Internal development roadmap. Tracks architectural decisions, build phases, and learning milestones.

---

## 🧠 Architectural Decision Log (ADL)

These are the *why* behind every major decision. Understanding these will help you answer interview questions.

### ADL-001: Package-by-Feature over Package-by-Layer

| | Package-by-Layer | Package-by-Feature ✅ |
|---|---|---|
| Structure | Grouped by technical role (all controllers together) | Grouped by domain (all vehicle code together) |
| Cohesion | Low — feature split across 4+ packages | High — one folder per feature |
| Scalability | Hard to extract to microservices | Easy to extract |
| Onboarding | Confusing for new devs | Intuitive |

**Decision:** Package-by-Feature. Each domain (`user`, `vehicle`, `servicelog`) is a self-contained vertical slice.

---

### ADL-002: UUID Primary Keys over Auto-Increment Long

| | Auto-Increment Long | UUID ✅ |
|---|---|---|
| Security | `vehicle/1` is guessable | `vehicle/a3f8...` is opaque |
| Distribution | Hard to merge across systems | Globally unique |
| Resume signal | Basic | Shows security awareness |

**Decision:** UUID v4 for all entity PKs. Required for any SaaS with user-owned resources.

---

### ADL-003: Stateless JWT over Server Sessions

- **Sessions:** Server stores state. Doesn't scale horizontally. DB bottleneck.
- **JWT:** Token is self-contained and cryptographically signed. Server validates signature only. No DB lookup per request. Scales to any number of servers.

**Decision:** JWT via `JJWT` library. Token contains `userId`, `email`, `role`, expiry.

---

### ADL-004: LAZY Loading as the Default

- `EAGER` loading silently fires extra SQL queries every time you load a parent entity.
- `LAZY` loading only fetches related data when explicitly accessed.
- We use **Repository-level `JOIN FETCH`** queries only when we genuinely need both parent + children in the same response.

**Decision:** All `@OneToMany` mappings are `fetch = FetchType.LAZY`.

---

### ADL-005: DTOs (Data Transfer Objects) using Java Records

- **Never expose your JPA Entity directly in an API response.** Entities contain password hashes, internal IDs, and lazy-loaded proxies that cause serialization errors.
- **DTOs** are clean, immutable contracts between your API and callers.
- Java 21 **Records** are perfect for DTOs: immutable, concise, auto-generates `equals`, `hashCode`, `toString`.

```java
// Clean. Immutable. No boilerplate.
public record VehicleResponse(UUID id, String make, String model, int year, int mileage) {}
```

---

## 🗂️ Domain Model

### Entity-Relationship Summary

```
User (1) ───< Vehicle (1) ───< ServiceLog
```

| Entity     | Key Fields                                      | FK Relationship          |
|------------|-------------------------------------------------|--------------------------|
| User       | id (UUID), email (unique), password_hash, role  | —                        |
| Vehicle    | id (UUID), make, model, year, trim, mileage     | user_id → users.id       |
| ServiceLog | id (UUID), service_date, service_type, cost     | vehicle_id → vehicles.id |

### Ownership Chain

```
JWT Token → userId → Vehicle.userId must match → ServiceLog.vehicleId must belong to that Vehicle
```

This chain is enforced at the **Service Layer**, not the Controller or Repository.

---

## 🗓️ Build Phases

### Phase 0: Project Bootstrap *(Current)*
- [ ] Initialize Spring Boot project via Spring Initializr
- [ ] Configure `application.yml` (local + prod profiles)
- [ ] Set up `docker-compose.yml` for local PostgreSQL
- [ ] Configure Flyway for database migrations
- [ ] Set up `.env.example` and `.gitignore`
- [ ] Create GitHub repository and push initial commit

**Exit Criteria:** Application starts locally, connects to PostgreSQL, Flyway runs V1 migration.

---

### Phase 1: Core Domain — CRUD Features

**Goal:** Build all three domain entities with full CRUD. No security yet.

#### 1A — User Entity & Registration
- [ ] Create `users` Flyway migration (V1)
- [ ] Create `User` JPA entity (UUID pk, ENUM role)
- [ ] Create `UserRepository` (Spring Data JPA)
- [ ] Create `RegisterRequest` and `UserResponse` Records
- [ ] Create `UserService` with `registerUser()` — hash password with BCrypt
- [ ] Create `UserController` with `POST /api/v1/auth/register`
- [ ] Write unit tests for `UserService`

#### 1B — Vehicle Entity & CRUD
- [ ] Create `vehicles` Flyway migration (V2)
- [ ] Create `Vehicle` JPA entity with `@ManyToOne User`
- [ ] Create `VehicleRepository` with `findAllByUserId(UUID userId)`
- [ ] Create DTOs: `CreateVehicleRequest`, `VehicleResponse`
- [ ] Create `VehicleService` (CRUD + ownership check stub)
- [ ] Create `VehicleController` (full CRUD endpoints)
- [ ] Write unit tests for `VehicleService`

#### 1C — ServiceLog Entity & CRUD
- [ ] Create `service_logs` Flyway migration (V3)
- [ ] Create `ServiceLog` JPA entity with `@ManyToOne Vehicle`
- [ ] Create `ServiceLogRepository`
- [ ] Create DTOs: `CreateServiceLogRequest`, `ServiceLogResponse`
- [ ] Create `ServiceLogService` and `ServiceLogController`
- [ ] Write unit tests for `ServiceLogService`

**Exit Criteria:** All CRUD endpoints work via Postman. Data persists to PostgreSQL.

---

### Phase 2: Security Wall — JWT Authentication & Authorization

**Goal:** Lock every `/vehicles` and `/logs` endpoint behind a valid JWT. Enforce ownership.

- [ ] Add Spring Security dependency
- [ ] Configure `SecurityConfig` — permit `/auth/**`, require auth on everything else
- [ ] Create `JwtService` — generate token, validate token, extract claims
- [ ] Create `JwtAuthFilter` — intercept every request, validate token, set `SecurityContext`
- [ ] Implement `POST /api/v1/auth/login` — verify credentials, return JWT
- [ ] Update `VehicleService` — extract `userId` from SecurityContext, compare to resource owner
- [ ] Update `ServiceLogService` — same ownership pattern
- [ ] Write integration tests for auth flows (valid token, expired token, wrong owner)

**Exit Criteria:** Unauthorized requests get `401`. Cross-user access gets `403`. All endpoints secured.

---

### Phase 3: AWS Deployment

**Goal:** Run production app on AWS EC2 + RDS, accessible via a public URL.

**Infrastructure Plan:**

```
Internet → EC2 (t2.micro, free tier)
              └── Docker container (Spring Boot app)
                        └── RDS (db.t3.micro, PostgreSQL, private subnet)
```

**Steps:**
- [ ] Write `Dockerfile` for the Spring Boot application
- [ ] Test Docker build locally
- [ ] Create AWS RDS instance (PostgreSQL, private subnet, no public access)
- [ ] Create AWS EC2 instance (Amazon Linux 2023, t2.micro)
- [ ] Configure Security Groups:
  - EC2: allow port 8080 from internet, port 5432 outbound to RDS only
  - RDS: allow port 5432 inbound from EC2 security group only
- [ ] SSH into EC2, install Docker, pull and run the container
- [ ] Set environment variables on EC2 (DB_URL pointing to RDS endpoint)
- [ ] Test all endpoints via Postman using the EC2 public IP

**Exit Criteria:** API is live at `http://<ec2-ip>:8080`. Data persists in RDS.

---

### Phase 4: CI/CD Pipeline — GitHub Actions

**Goal:** Every push to `main` automatically runs tests and redeploys to EC2.

- [ ] Create `.github/workflows/ci.yml` — run tests on every push/PR
- [ ] Create `.github/workflows/deploy.yml` — on merge to `main`:
  - Build Docker image
  - Push to Amazon ECR (Elastic Container Registry)
  - SSH into EC2 and pull + restart the container
- [ ] Store AWS credentials and SSH key as GitHub Secrets

**Exit Criteria:** Push to `main` → tests run → EC2 automatically running the new version.

---

## 🧪 Testing Strategy

| Test Type        | Tool          | What It Tests                                      | Where |
|------------------|---------------|----------------------------------------------------|-------|
| Unit Test        | JUnit 5, Mockito | Service layer logic, ownership checks, JWT validation | `src/test/.../service/` |
| Integration Test | Testcontainers   | Full request → DB round trip with real PostgreSQL  | `src/test/.../integration/` |
| Manual Test      | Postman       | Endpoint contracts, auth flows, error responses    | Postman collection |

**Unit test target:** `VehicleService.getVehicleById()` — mock the repository, verify that a `403` is thrown when `userId` doesn't match.

---

## 📚 Learning Checkpoints

Use these questions to self-assess before moving to the next phase.

### Before Phase 1
- [ ] Can you explain Authentication vs. Authorization with a concrete example from this app?
- [ ] What is the N+1 problem and how does `LAZY` loading help?
- [ ] Why do we use DTOs instead of returning JPA entities from controllers?
- [ ] What is Flyway and why is it better than `spring.jpa.hibernate.ddl-auto=create`?

### Before Phase 2
- [ ] What is inside a JWT? What are the three parts?
- [ ] What does the `SecurityContext` hold and how is it populated?
- [ ] What HTTP status code should an expired token return vs. a valid token accessing the wrong resource?

### Before Phase 3
- [ ] What is the difference between a Security Group's inbound and outbound rules?
- [ ] Why should RDS never be in a public subnet?
- [ ] What does a `Dockerfile` do vs. a `docker-compose.yml`?

### Before Phase 4
- [ ] What is the difference between CI and CD?
- [ ] Why should AWS credentials never be hardcoded in `.yml` files?

---

## 🔑 Key Vocabulary (Cheat Sheet)

| Term | What It Means |
|------|---------------|
| JWT | JSON Web Token — a signed, self-contained auth token |
| JPA | Java Persistence API — the standard for ORM in Java |
| ORM | Object-Relational Mapping — maps Java classes to DB tables |
| DTO | Data Transfer Object — an immutable API contract, not an entity |
| Flyway | A DB migration tool — keeps schema changes version-controlled |
| Testcontainers | Spins up a real Docker PostgreSQL for integration tests |
| BCrypt | A password hashing algorithm — never store plaintext passwords |
| SecurityContext | Spring's thread-local holder for the currently authenticated user |
| LAZY | Load related data only on demand, not automatically |
| EAGER | Load ALL related data immediately — the N+1 trap |
| N+1 Problem | 1 query to get N records, then N more queries for each record's relations |

---

*Last Updated: April 2026 | Author: @orrijoa*
