# Beyond Prevention: Post-Compromise Attack Detection in Java Microservice Architectures Using Gateway-Injected Honeytokens

A **honeytoken-based post-compromise detection system** for Java microservice ecosystems built with
**Spring Cloud Gateway** and **Redis**, implementing the approach described in the paper:

> *"Beyond Prevention: Post-Compromise Attack Detection in Java Microservice Architectures Using Gateway-Injected Honeytokens"*

Traditional perimeter defenses (firewalls, WAFs, authentication) focus on **preventing** initial
compromise, but once an attacker breaches the perimeter — via stolen credentials, a leaked API key,
or a compromised service account — these controls offer no further visibility. This framework
addresses that gap by **injecting honeytokens directly at the API Gateway layer**, so that any
*post-compromise* activity (credential replay, endpoint enumeration, lateral movement, token reuse)
is detected the moment an attacker interacts with a decoy artifact, independent of whether they
already hold valid-looking credentials.

---

## Architecture

```
                         ┌─────────────────────────────────────┐
                         │          API Gateway  :8080          │
                         │  ┌──────────────────────────────┐   │
  Client ──HTTP──►       │  │  HoneytokenGlobalFilter       │   │
                         │  │  ┌────────────────────────┐  │   │
                         │  │  │ 1. Endpoint check       │  │   │
                         │  │  │ 2. X-API-Key check      │  │   │
                         │  │  │ 3. JWT claim check      │  │   │
                         │  │  └────────────┬───────────┘  │   │
                         │  │               │               │   │
                         │  │         ┌─────▼──────┐        │   │
                         │  │         │   Redis    │        │   │
                         │  │         └────────────┘        │   │
                         │  └──────────────────────────────┘   │
                         │  Routes ──► user/order/payment svc  │
                         └─────────────────────────────────────┘
                                   AlertService ──► Logs + Metrics
```

### Services

| Service              | Port | Description                                         |
|----------------------|------|-----------------------------------------------------|
| `api-gateway`        | 8080 | Spring Cloud Gateway with HoneytokenGlobalFilter    |
| `user-service`       | 8081 | Legitimate user CRUD REST API                       |
| `order-service`      | 8082 | Legitimate order REST API                           |
| `payment-service`    | 8083 | Legitimate payment processing REST API              |
| `honeytoken-service` | 8084 | Honeytoken admin: registration, alerts, metrics     |
| Redis                | 6379 | Low-latency honeytoken lookup store                 |

---

## Honeytoken Types (Gateway-Injected Decoys)

| Type                | Example                       | Post-Compromise Detection Target      |
|---------------------|--------------------------------|----------------------------------------|
| `API_KEY`           | `HONEY-KEY-ALPHA-0001`        | Stolen credential replay / exfiltrated key reuse |
| `INTERNAL_ENDPOINT` | `/internal/admin`             | Post-breach reconnaissance / endpoint enumeration |
| `JWT_CLAIM`         | `honey-role-admin-9x` (claim `hny`) | Compromised JWT replay / privilege-escalation attempt |
| `DB_RECORD`         | (tracked by honeytoken-svc)   | Lateral movement / database exfiltration (future)     |

All decoys are injected and verified at the **API Gateway**, the single ingress point for every
microservice, so detection coverage does not depend on instrumenting each downstream service
individually. Any access to these tokens triggers:
1. A structured `HONEYTOKEN_ALERT` JSON log entry
2. Micrometer counter increment (`honeytoken.trigger.total`)
3. `403 Forbidden` response — the attacker is blocked silently

---

## Quick Start

### Prerequisites
- Java 21+
- Maven 3.8+
- Docker (for Redis)

### 1. Start Redis
```bash
docker run -d -p 6379:6379 --name honeytoken-redis redis:7-alpine
```

### 2. Build the project
```bash
mvn clean package -DskipTests
```

### 3. Run all services (IntelliJ Compound Run Config or separate terminals)
```bash
# Terminal 1
cd api-gateway && mvn spring-boot:run

# Terminal 2
cd user-service && mvn spring-boot:run

# Terminal 3
cd order-service && mvn spring-boot:run

# Terminal 4
cd payment-service && mvn spring-boot:run

# Terminal 5
cd honeytoken-service && mvn spring-boot:run
```

### 4. Or use Docker Compose
```bash
mvn clean package -DskipTests
docker-compose up --build
```

---

## Attack Simulation

Use the IntelliJ HTTP Client file: `src/test/resources/attack-simulation.http`

Or with `curl`:

```bash
# Legitimate request (passes through)
curl http://localhost:8080/api/users

# ATTACK: Honeytoken API key (triggers alert + 403)
curl -H "X-API-Key: HONEY-KEY-ALPHA-0001" http://localhost:8080/api/users

# ATTACK: Decoy endpoint enumeration (triggers alert + 403)
curl http://localhost:8080/internal/admin
curl http://localhost:8080/internal/config
curl http://localhost:8080/actuator/env
```

---

## Evaluation Metrics

```bash
# Detection metrics (TP, FP, FN, detection rate, FPR)
curl http://localhost:8080/admin/metrics

# Prometheus metrics (includes honeytoken.trigger.total)
curl http://localhost:8080/actuator/prometheus | grep honeytoken

# Alert log
curl http://localhost:8084/honeytokens/alerts

# All registered honeytokens
curl http://localhost:8084/honeytokens
```

---

## Honeytoken Management

```bash
# Register a custom API key honeytoken
curl -X POST "http://localhost:8084/honeytokens?type=API_KEY&token=MY-DECOY-KEY&description=Custom+decoy"

# Register a custom endpoint honeytoken
curl -X POST "http://localhost:8084/honeytokens?type=INTERNAL_ENDPOINT&token=/internal/vault&description=Vault+decoy"

# Register a custom JWT claim honeytoken
curl -X POST "http://localhost:8084/honeytokens?type=JWT_CLAIM&token=admin-honey-value&description=Decoy+JWT+role"

# List all
curl http://localhost:8084/honeytokens

# Delete
curl -X DELETE "http://localhost:8084/honeytokens/API_KEY/MY-DECOY-KEY"
```

---

## Default Seeded Honeytokens

The gateway auto-seeds these on startup:

| Type                | Value                        |
|---------------------|------------------------------|
| API_KEY             | `HONEY-KEY-ALPHA-0001`       |
| API_KEY             | `HONEY-KEY-BETA-9999`        |
| INTERNAL_ENDPOINT   | `/internal/admin`            |
| INTERNAL_ENDPOINT   | `/internal/config`           |
| INTERNAL_ENDPOINT   | `/internal/debug`            |
| INTERNAL_ENDPOINT   | `/actuator/env`              |
| INTERNAL_ENDPOINT   | `/api/v1/internal/users`     |
| JWT_CLAIM           | `honey-role-admin-9x`        |
| JWT_CLAIM           | `honey-svc-secret-007`       |

---

## Project Structure

```
post-compromise-honeytoken-detection/
├── pom.xml                          ← Parent POM (multi-module)
├── docker-compose.yml
├── api-gateway/                     ← Spring Cloud Gateway + Honeytoken filter
│   └── src/main/java/com/honeytoken/gateway/
│       ├── ApiGatewayApplication.java
│       ├── config/HoneytokenSeeder.java        ← Seeds honeytokens on startup
│       ├── controller/
│       │   ├── HoneytokenAdminController.java  ← Runtime token management
│       │   └── MetricsController.java          ← Evaluation metrics endpoint
│       ├── filter/HoneytokenGlobalFilter.java  ← Core detection filter
│       ├── metrics/DetectionMetricsEvaluator.java
│       ├── model/{Honeytoken,AlertEvent}.java
│       └── service/
│           ├── AlertService.java
│           ├── HoneytokenLookupService.java    ← Redis lookup (reactive)
│           └── JwtParserService.java
├── user-service/     ← Legitimate user CRUD (port 8081)
├── order-service/    ← Legitimate order API  (port 8082)
├── payment-service/  ← Legitimate payment API (port 8083)
├── honeytoken-service/ ← Admin + alert log + metrics (port 8084)
└── src/test/resources/attack-simulation.http ← IntelliJ HTTP Client test file
```

