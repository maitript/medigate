# Patient Management

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![gRPC](https://img.shields.io/badge/RPC-gRPC-blue)](https://grpc.io/)
[![Kafka](https://img.shields.io/badge/Messaging-Kafka-black?logo=apachekafka)](https://kafka.apache.org/)
[![AWS CDK](https://img.shields.io/badge/Infra-AWS%20CDK%20%2B%20LocalStack-FF9900?logo=amazonwebservices&logoColor=white)](https://localstack.cloud/)
[![License](https://img.shields.io/badge/License-see%20repo-lightgrey)](LICENSE)

> Spring Boot microservices for patient CRUD with JWT auth, gRPC billing, Kafka analytics, an API gateway, and AWS CDK deployable to LocalStack.

A multi-service patient platform: clients hit the gateway, auth issues JWTs, the patient service owns CRUD and fans out to billing (gRPC) and analytics (Kafka). Infrastructure is defined as CDK and can be exercised against LocalStack (ECS, RDS, MSK, ALB).

## Architecture

```text
Client
  │
  ▼
API Gateway (:4004)
  ├── /auth/**          → Auth Service (:4005)
  └── /api/patients/**  → Patient Service (:4000)  [JWT required]
                              │
                              ├── gRPC → Billing Service (:9001)
                              └── Kafka → Analytics Service
```

| Service | Responsibility | Ports |
|---|---|---|
| **api-gateway** | Edge routing + JWT validation for patient APIs | `4004` |
| **auth-service** | Login, JWT issue/validate | `4005` |
| **patient-service** | Patient CRUD, publishes events, calls billing | `4000` |
| **billing-service** | Creates billing accounts over gRPC | `4001` (HTTP), `9001` (gRPC) |
| **analytics-service** | Consumes patient Kafka events | — |
| **infrastructure** | AWS CDK stack (VPC, ECS/Fargate, RDS, MSK, ALB) for LocalStack | — |

### Cross-cutting concerns

- **Auth**: JWT issued by `auth-service`; gateway validates tokens on `/api/patients/**`
- **gRPC**: `patient-service` → `billing-service` when a patient is created
- **Kafka**: `patient-service` produces events; `analytics-service` consumes them
- **Persistence**: PostgreSQL for auth and patient data (H2 available for local/dev)

## Tech stack

- Java 21, Spring Boot 3.3
- Spring Cloud Gateway
- Spring Security + JWT
- Spring Data JPA, PostgreSQL / H2
- gRPC + Protobuf
- Apache Kafka
- AWS CDK (Java), LocalStack, ECS Fargate, RDS, MSK
- Maven, Docker
- Rest Assured integration tests

## Project structure

```text
patient-management/
├── api-gateway/          # Spring Cloud Gateway
├── auth-service/         # Authentication & JWT
├── patient-service/      # Patient REST API
├── billing-service/      # gRPC billing
├── analytics-service/    # Kafka consumer
├── infrastructure/       # AWS CDK + LocalStack deploy
├── integration-tests/    # End-to-end HTTP tests
├── api-requests/         # HTTP client samples (auth + patients)
└── grpc-requests/        # gRPC sample requests
```

## Prerequisites

- JDK 21+
- Maven 3.9+
- Docker
- [LocalStack](https://localstack.cloud/) (for cloud-style deployment)
- AWS CLI (configured for LocalStack endpoint)

## Quick start (local services)

```bash
git clone https://github.com/maitript/patient-management.git
cd patient-management
```

Run each service from its module (or via your IDE):

```bash
cd auth-service && ./mvnw spring-boot:run
cd patient-service && ./mvnw spring-boot:run
cd billing-service && ./mvnw spring-boot:run
cd analytics-service && ./mvnw spring-boot:run
cd api-gateway && ./mvnw spring-boot:run
```

For local patient-service development without Postgres, uncomment the H2 settings in  
`patient-service/src/main/resources/application.properties`.

### Demo credentials

| Email | Password | Role |
|---|---|---|
| `testuser@test.com` | `password123` | `ADMIN` |

## API overview

All client traffic should go through the gateway when deployed.

### Auth

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/login` | Returns a JWT |
| `GET` | `/auth/validate` | Validates `Authorization: Bearer <token>` |

**Login body**

```json
{
  "email": "testuser@test.com",
  "password": "password123"
}
```

### Patients (JWT required via gateway)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/patients` | List patients |
| `POST` | `/api/patients` | Create patient |
| `PUT` | `/api/patients/{id}` | Update patient |
| `DELETE` | `/api/patients/{id}` | Delete patient |

Sample HTTP requests live under `api-requests/`. gRPC samples: `grpc-requests/`.

## Docker images

Each runnable service has a multi-stage Dockerfile (Maven build → JRE runtime):

```bash
docker build -t auth-service ./auth-service
docker build -t patient-service ./patient-service
docker build -t billing-service ./billing-service
docker build -t analytics-service ./analytics-service
docker build -t api-gateway ./api-gateway
```

## Deploy with LocalStack + CDK

Infrastructure is defined in `infrastructure/` as an AWS CDK Java app targeting LocalStack (VPC, ECS Fargate services, Postgres RDS, MSK Kafka, ALB).

1. Start LocalStack
2. Synthesize / build the CDK template (`infrastructure/cdk.out/localstack.template.json`)
3. Deploy:

```bash
cd infrastructure
./localstack-deploy.sh
```

The script deletes/redeploys the `patient-management` CloudFormation stack on `http://localhost:4566` and prints the load balancer DNS. Point gateway traffic at that host on port `4004`.

## Integration tests

```bash
cd integration-tests
mvn test
```

Tests use Rest Assured against the running stack (auth + patient flows).

## Notes

- Gateway strips the `/auth` and `/api` prefixes before forwarding.
- Patient routes require a valid JWT; auth routes do not.
- Billing is invoked over gRPC (`9001`), not through the HTTP gateway.
- Root `.gitignore` excludes IDE metadata (`.idea/`, `*.iml`) and Maven `target/` outputs.
