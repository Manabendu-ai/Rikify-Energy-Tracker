<div align="center">

# Rikify

[![Java](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0-green.svg)](https://spring.io/projects/spring-boot)
[![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2025.1.0-blue.svg)](https://spring.io/projects/spring-cloud)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/license-Educational-lightgrey.svg)]()

**A microservices reference implementation for monitoring and reasoning about household electricity usage.**

</div>

---

The system accepts energy readings from devices, processes them asynchronously, stores time-series metrics, raises alerts when usage spikes, and exposes a unified API through an API Gateway with built-in resilience, security, and observability.

> **Author & Maintainer:** Manabendu Karfa — architect and primary contributor of this microservices reference implementation.

---

<div align="center">

## Project Overview

</div>

**Home Energy Tracker** models how a real product might collect power (watts) and timestamps from smart plugs or meters, aggregate that data for dashboards and billing-style views, and notify residents when consumption crosses defined thresholds.

**Problem it solves:** Raw device events are high-volume and require reliable ingestion, decoupled processing, and specialized storage (relational metadata versus time-series measurements). This project demonstrates that separation of concerns: HTTP APIs for users and devices, Kafka for event streaming, InfluxDB for usage series, and MySQL for durable domain data.

**Typical use cases:**

- Track per-device energy usage over time
- Alert when instantaneous or aggregated power exceeds a defined limit
- Gate all public HTTP traffic through a single entry point (API Gateway) with JWT validation
- Observe latency, errors, and circuit-breaker state through Prometheus and Grafana

---

<div align="center">

## Architecture Overview

</div>

The system follows a microservices architecture built primarily with Spring Boot 4 and Java 21. Services are independently deployable modules; integration relies on synchronous HTTP (client → gateway → service) and asynchronous messaging (Kafka) where loose coupling and scale are required.

**Patterns and capabilities:**

| Area | Approach |
|------|----------|
| **API Gateway** | Spring Cloud Gateway (Server MVC); single public HTTP façade, route aggregation, OpenAPI aggregation |
| **Service Communication** | REST between gateway and backends; Kafka for ingestion → usage → alerts |
| **Resilience** | Circuit breakers (Resilience4j) on gateway routes with fallbacks |
| **Security** | OAuth2 Resource Server on the gateway; Keycloak for identity (dev profile in Docker Compose) |
| **Observability** | Spring Boot Actuator, Micrometer, Prometheus scrape targets, Grafana dashboards |
| **Configuration** | Per-service `application.properties` (no separate Spring Cloud Config Server in this repository) |

**High-level interaction:** Clients call the API Gateway. Domain services (user, device, ingestion, insight) sit behind it. Ingestion publishes to Kafka; usage consumes messages, writes to InfluxDB, and may publish alerts; alert consumes alert events and sends email notifications (via Mailpit in local development). Insight can provide AI-generated summaries (Spring AI), routed through the gateway when enabled.

---

<div align="center">

## Services Breakdown

</div>

| Service | Port | Responsibility | Key Technologies | Interactions |
|---------|------|-----------------|-------------------|--------------|
| **api-gateway** | 8080 | Public entry point: routing, circuit breaking, JWT validation, aggregated API documentation | Spring Boot 4, Spring Cloud Gateway (WebMVC), Resilience4j, OAuth2 Resource Server, springdoc | Proxies to user, device, ingestion, and insight services; calls Keycloak JWKS |
| **user-service** | 8081 | User accounts and related persistence | Spring Boot 4, JPA, MySQL, Flyway, Actuator/Prometheus | MySQL; invoked via gateway |
| **device-service** | 8082 | Device registry and metadata | Spring Boot 4, JPA, MySQL, Actuator/Prometheus | MySQL; invoked via gateway |
| **ingestion-service** | 8083 | Accepts energy readings over HTTP and publishes to the streaming pipeline | Spring Boot 4, Kafka producer, Actuator/Prometheus | Produces to Kafka (`energy-usage`); invoked via gateway or directly for testing |
| **usage-service** | 8084 | Consumes usage events, manages time-series storage, and applies aggregation/threshold logic | Spring Boot 4, Kafka consumer/producer, InfluxDB Java client, Actuator/Prometheus | Kafka to InfluxDB; produces alert events for downstream consumers |
| **alert-service** | 8085 | Consumes alert events and notifies users (e.g. via email) | Spring Boot 4, Kafka, JPA, Mail, MySQL, Actuator/Prometheus | Kafka consumer; SMTP (Mailpit locally); MySQL where applicable |
| **insight-service** | 8086 | Generates usage insights (e.g. LLM-backed explanations via Ollama) | Spring Boot 3.5, Spring AI, Ollama starter, Actuator/Prometheus | Invoked via gateway; optional external Ollama runtime |

> **Note:** Most services target Spring Boot 4; `insight-service` uses Spring Boot 3.5 with Spring AI for model integration. There is no Spring Cloud Config Server or Kubernetes manifests in this repository — Docker Compose is the primary local orchestration path.

---

<div align="center">

## System Flow and Diagrams

</div>

### Background and Requirements

Electricity basics, assumptions, and what the system must support (sources, units, constraints).

![Background and system requirements](diagrams/background-and-requirements.png)
*Figure: Background and requirements for the Home Energy Tracker domain.*

---

### Circuit Breaker in the API Gateway

When downstream services fail or slow down, the gateway stops overwhelming them: the circuit breaker opens, short-circuits calls, and can return a controlled fallback — improving stability across the whole system.

![Circuit breaker pattern in the API Gateway](diagrams/circuit-breaker-in-api-gateway.png)
*Figure: Resilience and circuit breaker behavior at the edge (API Gateway).*

---

### Gateway in the Public Network

The API Gateway sits in a public or DMZ-style network segment while core services run in a more private zone. Clients never communicate with individual microservices directly; they use a single controlled entry point.

![Network separation with API Gateway](diagrams/diagram-showing-gateway-in-public-network.png)
*Figure: Public versus private network separation with the gateway as the controlled entry point.*

---

### Full Microservices Flow

End-to-end path: ingestion, messaging, usage processing, storage, alerting, and supporting services.

![Full microservices flow with components](diagrams/full-microservices-flow-diagram-with-components.png)
*Figure: Full system walkthrough across components and data paths.*

---

### Observability with Prometheus and Grafana

Services expose Prometheus-compatible metrics via Actuator. Prometheus scrapes and stores these series; Grafana visualizes SLO-friendly dashboards covering latency, errors, JVM health, and circuit breaker status.

![Observability with Prometheus and Grafana](diagrams/observability-with-prometheus-and-grafana.png)
*Figure: Monitoring and observability stack (metrics flow and tooling).*

---

<div align="center">

## Tech Stack

</div>

- **Language:** Java 21
- **Framework:** Spring Boot 4 (domain services and gateway); Spring Boot 3.5 with Spring AI (`insight-service`)
- **Spring Cloud:** 2025.1.0 — Gateway (Server WebMVC), Circuit Breaker (Resilience4j)
- **Messaging:** Apache Kafka (KRaft)
- **Databases:** MySQL 8 (relational data), InfluxDB 2 (time-series usage)
- **Identity (local development):** Keycloak
- **Email (local development):** Mailpit
- **Observability:** Micrometer, Prometheus, Grafana
- **API Documentation:** springdoc-openapi (gateway aggregates service OpenAPI URLs)
- **Containerization:** Docker and Docker Compose
- **Build:** Maven (each service includes `mvnw`)

Kubernetes is not part of this repository; deploying to Kubernetes would be a natural extension (Helm charts, ConfigMaps, service mesh, etc.).

---

<div align="center">

## Getting Started

</div>

### Prerequisites

- JDK 21
- Docker and Docker Compose
- Maven (optional if using `./mvnw` within each service)

### Clone the Repository

```bash
git clone git@github.com:Manabendu-ai/home-energy-tracker.git
cd home-energy-tracker
```

### Start Infrastructure

From the repository root:

```bash
docker compose -v up -d
```

This brings up MySQL, Kafka, Kafka UI, InfluxDB, Mailpit, Keycloak (with its database), Prometheus, and Grafana.

Stop everything with:

```bash
docker compose down
```

If databases fail to initialize, remove the associated volumes or re-run `docker/mysql/init.sql`.

### Build Services

Each microservice is its own Maven project:

```bash
cd user-service && ./mvnw -q package && cd ..
# Repeat for: device-service, ingestion-service, usage-service, alert-service, insight-service, api-gateway
```

Or run a service directly with:

```bash
./mvnw spring-boot:run
```

### Run Applications

1. Ensure Docker Compose is running (Kafka, MySQL, InfluxDB, etc.).
2. Start services on the host using their default ports (see the services table above), or containerize them independently.
3. For Kafka access from the host, the bootstrap address is typically `localhost:9094` (external listener in Compose).

Prometheus in this repository is configured to scrape `host.docker.internal` for Actuator endpoints, so metrics work correctly when Spring Boot applications run on the host while Prometheus runs in Docker.

### Quick Pipeline Test

Post a sample reading to the ingestion service (directly or via the gateway, if routed):

```bash
curl -X POST http://localhost:8083/api/ingestion \
  -H 'Content-Type: application/json' \
  -d '{"deviceId":"dev-1","timestamp":"2025-01-01T12:00:00Z","watts":1200}'
```

Then verify results through usage-service logs, InfluxDB, Kafka UI (`http://localhost:8070`), and Mailpit (`http://localhost:8025`) once threshold and alert logic executes.

### Access Points (Local Defaults)

| Component | URL |
|-----------|-----|
| **API Gateway** | http://localhost:8080 |
| **Grafana** | http://localhost:3000 (admin / admin) |
| **Prometheus** | http://localhost:9090 |
| **Kafka UI** | http://localhost:8070 |
| **Mailpit** | http://localhost:8025 |
| **Keycloak** | http://localhost:8091 |
| **InfluxDB UI** | http://localhost:8072 |

Service-specific OpenAPI documentation is linked from the gateway's Swagger UI configuration (`/swagger-ui.html`).

---

<div align="center">

## Observability

</div>

- Each Spring Boot application exposes `/actuator/prometheus` (enabled via dependencies and management configuration).
- Prometheus (`docker/prometheus/prometheus.yml`) defines scrape jobs for the gateway and all services on the host.
- Grafana loads provisioning from `docker/grafana/provisioning` and uses Prometheus as its data source.
- Circuit breaker state can be surfaced through Actuator health endpoints where enabled (see the gateway's `application.properties`).

Use Grafana for dashboards and Prometheus for ad-hoc queries and alerting rules as the deployment is extended.

---

<div align="center">

## Future Improvements

</div>

- **End-to-end tests** — Contract or black-box tests spanning gateway, services, Kafka, and the database
- **CI/CD** — Build matrix per service, image publishing, Compose or Kubernetes smoke tests
- **Frontend dashboard** — Single-page application for devices, live usage charts, and alert history
- **Authorization hardening** — Fine-grained scopes, service-to-service tokens, policy engine
- **Kubernetes** — Helm charts, external secrets, horizontal pod autoscaling, and Kafka/Influx operators
- **Centralized configuration** — Spring Cloud Config or external secret stores for non-development environments

---

<div align="center">

## Author

**Manabendu Karfa**
Architect and primary contributor of the Home Energy Tracker microservices reference implementation.

</div>

---

<div align="center">

*Home Energy Tracker — a portfolio-grade Spring microservices example for learning production-style patterns without oversimplifying the moving parts.*

</div>
