# 📋 Project Planning — Smart DIY Garage Backend

> Internal development roadmap. Tracks architectural decisions, build phases, and technical strategy.

---

## 🧠 Architectural Decision Log (ADL)

Records the rationale behind key design choices made in this project.

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
| Security | Resource IDs are enumerable and guessable | Opaque, non-enumerable identifiers |
| Distribution | Collisions likely across distributed systems | Globally unique by design |

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

### ADL-005: DTOs using Java Records

- API responses use dedicated DTO types rather than JPA entities. This decouples the API contract from the persistence model, prevents accidental exposure of sensitive fields, and avoids lazy-loading serialization issues.
- Java 21 Records are used for all DTOs: they are immutable, concise, and require no boilerplate.

```java
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

### Phase 0: Project Bootstrap *(Done)*
- [x] Initialize Spring Boot project via Spring Initializr
- [x] Configure `application.yml` (local + prod profiles)
- [x] Set up `docker-compose.yml` for local PostgreSQL
- [x] Configure Flyway for database migrations
- [x] Set up `.env.example` and `.gitignore`
- [x] Create GitHub repository and push initial commit

**Exit Criteria:** Application starts locally, connects to PostgreSQL, Flyway runs V1 migration.

---

### Phase 1: Core Domain — CRUD Features *(Current)*

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

*Last Updated: April 2026 | Author: @noah-hw-kim*
