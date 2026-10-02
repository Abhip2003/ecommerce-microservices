
## 📋 Overview

**ecommerce-microservices** is a cloud-native e-commerce platform built
using a microservice architecture. The backend is organized into **8
Spring Boot 3 microservices**, with an API Gateway, a
**Next.js 16** frontend, and supporting infrastructure deployable on
**Kubernetes (k3d)**.

The platform separates major e-commerce responsibilities into
independently deployable services for products, orders, payments,
inventory, shipping, ratings, search, and notifications.

  ------------------ ----------------------------------------------------
  ⚡ **Backend**     8 microservices · Java 21 · Spring Boot 3.3.5
  🚪 **Gateway**     Apache APISIX 3.9 · Rate limiting · JWT validation
  🗄️ **Databases**   PostgreSQL 16 · Elasticsearch 8
  📨 **Messaging**   Apache Kafka 3.9 (KRaft mode)
  🔐 **Auth**        OAuth2 / OIDC · JWT
  🐳 **Deploy**      Docker Compose · k3d / Kubernetes
  ------------------ ----------------------------------------------------

------------------------------------------------------------------------

## 🧩 Microservices

The project contains the following backend services:

  -----------------------------------------------------------------------------
  Service                                            Port Responsibility
  -------------------------- ---------------------------- ---------------------
  📦 **inventory-service**                           8082 Product stock and
                                                          inventory management

  🔔                                                 8090 Notifications and
  **notification-service**                                email communication

  🛒 **order-service**                               8084 Cart and order
                                                          management

  💳 **payment-service**                             8085 Payment processing
                                                          and payment status

  🏷️ **product-service**                             8086 Products and
                                                          categories

  ⭐ **rating-service**                              8089 Product ratings and
                                                          reviews

  🔎 **search-service**                              8094 Product search using
                                                          Elasticsearch

  🚚 **shipping-service**                            8087 Shipment and shipping
                                                          management
  -----------------------------------------------------------------------------

------------------------------------------------------------------------

## 🛠️ Technology Stack

### Backend

  ----------------------------------- -----------------------------------
  **Runtime**                         Java 21

  **Framework**                       Spring Boot 3.3.5, Spring Security
                                      6, Spring Data JPA

  **Database**                        PostgreSQL 16, Liquibase migrations

  **Search**                          Elasticsearch 8, Spring Data
                                      Elasticsearch

  **Messaging**                       Apache Kafka 3.9 (KRaft), Spring
                                      Kafka

  **Security**                        Keycloak 26, OAuth2 / OIDC, JWT,
                                      Spring Security Resource Server

  **Gateway**                         Apache APISIX 3.9

  **Storage**                         RustFS (S3-compatible object
                                      storage)

  **Observability**                   Micrometer, Prometheus, Spring Boot
                                      Actuator

  **API Docs**                        Springdoc OpenAPI 3, Swagger UI

  **Resilience**                      Resilience4j

  **Build**                           Maven, Jib
  ----------------------------------- -----------------------------------

### Frontend

  --------------- ------------------------------
  **Framework**   Next.js 16.2, React 19.2
  **State**       Zustand 5, TanStack Query 5
  **Styling**     Tailwind CSS 4, Lucide Icons
  **HTTP**        Axios
  --------------- ------------------------------

### Infrastructure

  ---------------------- ---------------------------
  **Containerization**   Docker
  **Local Kubernetes**   k3d / K3s
  **Ingress**            NGINX Ingress Controller
  **API Gateway**        Apache APISIX
  **Deployment**         Kubernetes
  **GitOps**             ArgoCD
  **Registry**           GitHub Container Registry
  **CI**                 GitHub Actions
  **Code Quality**       SonarCloud
  ---------------------- ---------------------------

------------------------------------------------------------------------

## 🏗️ Architecture

### System Architecture

![System Architecture](architecture-kafka.png)


``` text
                         Browser
                            |
                            v
                    Next.js Frontend
                            |
                       REST + JWT
                            |
                            v
                  +-------------------+
                  |   Apache APISIX   |
                  |    API Gateway    |
                  +-------------------+
                            |
        +-------------------+-------------------+
        |         |         |         |         |
        v         v         v         v         v
    Product     Order    Payment  Inventory  Shipping
    Service    Service   Service   Service    Service
        |         |         |         |         |
        +---------+---------+---------+---------+
                            |
                       PostgreSQL
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
  Rating Service      Notification Service   Search Service
                            |                   |
                            v                   v
                         Kafka            Elasticsearch
                            |
                            v
                           SMTP

                 Keycloak
                    |
                    v
                  JWT Auth

                 RustFS
                    |
                    v
              Media / Object Storage
```

------------------------------------------------------------------------

## 🔄 Service Communication

### Synchronous Communication

Services communicate synchronously using REST APIs.

``` text
Order Service
      |
      | REST / HTTP
      v
Product Service
```

Spring `RestClient` is used for service-to-service HTTP communication.

### Asynchronous Communication

Apache Kafka is used for asynchronous event communication.

``` text
Payment Service
      |
      | Payment Successful Event
      v
    Kafka
      |
      v
Notification Service
      |
      +----> Database
      |
      +----> Email / SMTP
```

This allows notification processing to happen independently from the
payment request.

------------------------------------------------------------------------

## 🔐 Authentication & Authorization

Authentication is handled using **Keycloak** with OAuth2 / OpenID
Connect.

``` text
User
 |
 v
Keycloak
 |
 | JWT
 v
Frontend
 |
 | Authorization: Bearer <JWT>
 v
APISIX
 |
 v
Spring Boot Services
```

Protected services validate JWT tokens using Spring Security OAuth2
Resource Server.

The architecture provides authentication at the gateway and service
levels.

------------------------------------------------------------------------

## 🗄️ Database Architecture

The backend follows a database-per-service approach using PostgreSQL.

``` text
PostgreSQL
│
├── inventory database
├── order database
├── payment database
├── product database
├── rating database
├── shipping database
└── notification database
```

Services access their own data through Spring Data JPA and Hibernate.

Cross-service data is accessed through APIs rather than direct database
access.

### Search

Elasticsearch is used by `search-service` for product search and
indexing.

``` text
Product Service
      |
      v
Search Service
      |
      v
Elasticsearch
```

------------------------------------------------------------------------

## 📨 Kafka Messaging & Saga Pattern

Apache Kafka coordinates the distributed transaction between the **Order**, **Payment**, and
**Inventory** services using a **Saga-based event-driven workflow**.

```text
                    Apache Kafka
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
  Order Service    Payment Service   Inventory Service
        |                |                |
        +----------------+----------------+
                         |
                    Saga Events
                         |
                         v
              Compensation Events
```

The order workflow is coordinated through Kafka events instead of a distributed database
transaction.

### Order Saga Flow

```text
Order Service
      |
      | Order Created
      v
    Kafka
      |
      v
Inventory Service
      |
      | Stock Reserved
      v
    Kafka
      |
      v
Payment Service
      |
      | Payment Successful
      v
    Kafka
      |
      v
Order Service
      |
      | Order Confirmed
      v
   Completed
```

If a step fails, the Saga uses **compensating events** to undo previously completed actions.

For example:

```text
Order Created
      ↓
Stock Reserved
      ↓
Payment Failed
      ↓
    Kafka
      ↓
Release Stock
      ↓
Order Cancelled
```

This approach avoids a distributed two-phase commit and allows each service to maintain its
own database while achieving eventual consistency across the order, payment, and inventory
workflow.

## 🗂️ Object Storage

**RustFS** provides S3-compatible object storage for files and media.

``` text
Application
     |
     v
Media / Storage Layer
     |
     v
RustFS
```

The project uses RustFS as the object storage layer without depending on
a public cloud storage provider.

------------------------------------------------------------------------

## 🐳 Docker

Docker is used to package the services and infrastructure into
reproducible containers.

The local environment can run the complete stack using Docker Compose.

``` bash
docker compose up
```

The stack includes the backend services, frontend, PostgreSQL, Kafka,
Elasticsearch, Redis, RustFS, Keycloak, and APISIX.

------------------------------------------------------------------------

## ☸️ Kubernetes Deployment

The application can also be deployed to Kubernetes using **k3d**.

``` text
                 Kubernetes Cluster
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
      APISIX          Backend         Frontend
                      Services
                        |
              +---------+---------+
              |         |         |
              v         v         v
          PostgreSQL  Kafka  Elasticsearch
```

Kubernetes Services provide internal service discovery, while NGINX
Ingress exposes the application externally.

------------------------------------------------------------------------

## 🔄 CI/CD

The project uses GitHub Actions and ArgoCD.

``` text
Developer
    |
    v
 GitHub
    |
    v
GitHub Actions
    |
    +----> Test
    |
    +----> Build
    |
    +----> Docker Image
    |
    v
GitHub Container Registry
    |
    v
  ArgoCD
    |
    v
Kubernetes
```

ArgoCD follows a GitOps approach and synchronizes Kubernetes resources
from the repository.

------------------------------------------------------------------------

## 📊 Observability

Services expose Spring Boot Actuator endpoints for health and metrics.

### Health

``` text
/actuator/health
```

### Prometheus Metrics

``` text
/actuator/prometheus
```

### Correlation ID

Requests can be traced across services using:

``` text
X-Correlation-Id
```

APISIX also exposes Prometheus metrics.

------------------------------------------------------------------------

## 🚀 Quick Start

### Prerequisites

-   Docker Desktop on macOS / Windows
-   Docker Engine on Linux
-   8 GB+ RAM allocated to Docker
-   Git

### Kubernetes Setup

Clone the repository:

``` bash
git clone https://github.com/hoangtien2k3/ecommerce-microservices.git
cd ecommerce-microservices
```

Run the setup script:

``` bash
bash start-ecommerce.sh
```

The setup process creates the local Kubernetes cluster and deploys the
required infrastructure and services.

Check pod status:

``` bash
kubectl get pods -n ecommerce -w
```

------------------------------------------------------------------------

## 🌐 URLs

  --------------------------------------------------------------------------------------------
  Service                             URL
  ----------------------------------- --------------------------------------------------------
  🏠 **Frontend**                     `http://ecommerce.local`

  🚪 **API Gateway**                  `http://api.ecommerce.local`

  🔐 **Keycloak Admin**               `http://keycloak.ecommerce.local/admin/master/console`

  📦 **RustFS Console**               `http://rustfs.ecommerce.local/rustfs/console/`
  --------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 📁 Project Structure

``` text
ecommerce-microservices/
│
├── inventory-service/
├── notification-service/
├── order-service/
├── payment-service/
├── product-service/
├── rating-service/
├── search-service/
├── shipping-service/
│
├── common-lib/
│   ├── common-core/
│   ├── common-spring/
│   ├── common-security/
│   ├── common-keycloak/
│   ├── common-kafka/
│   ├── common-logging/
│   └── common-storage/
│
├── frontend/
│
├── deploy/
│   └── apisix/
│
├── docker/
│   ├── keycloak/
│   └── postgres/
│
├── k8s/
│   ├── argocd/
│   ├── backend/
│   ├── frontend/
│   ├── gateway/
│   ├── infra/
│   ├── ingress/
│   ├── configmap.yaml
│   ├── namespace.yaml
│   └── secrets.yaml
│
├── docker-compose.yml
├── k3d-config.yaml
├── k3d-setup.sh
├── Makefile
├── pom.xml
└── start-ecommerce.sh
```

------------------------------------------------------------------------

## 🎯 Key Features

-   8 independent Spring Boot microservices
-   REST-based service-to-service communication
-   Apache APISIX API Gateway
-   Keycloak OAuth2 / OIDC authentication
-   JWT-based authorization
-   PostgreSQL database-per-service architecture
-   Elasticsearch-powered product search
-   Kafka asynchronous messaging and Saga-based transaction coordination
-   RustFS S3-compatible object storage
-   Docker containerization
-   Kubernetes deployment with k3d
-   ArgoCD GitOps deployment
-   GitHub Actions CI pipeline
-   Prometheus-compatible metrics
-   Spring Boot Actuator health checks
-   Correlation ID propagation
-   Resilience4j-based resilience

------------------------------------------------------------------------

## ⚖️ Architecture Trade-offs

### Benefits

-   Independent service deployment
-   Clear separation of business responsibilities
-   Services can scale independently
-   Failure isolation between services
-   Technology-specific infrastructure such as Elasticsearch for search
-   Centralized API gateway and authentication
-   Kubernetes-based deployment

### Challenges

-   Distributed service communication
-   Network failures and latency
-   More complicated debugging
-   Distributed data consistency
-   Multiple databases and migrations
-   Increased deployment and operational complexity

For a small application, a modular monolith can be simpler.
Microservices become more useful when independent deployment, scaling,
team ownership, and service isolation justify the additional complexity.

------------------------------------------------------------------------

## 📌 Architecture Summary

``` text
Frontend
   |
   v
Apache APISIX
   |
   +---- Product Service
   +---- Order Service
   +---- Payment Service
   +---- Inventory Service
   +---- Shipping Service
   +---- Rating Service
   +---- Search Service
   +---- Notification Service
            |
            v
          Kafka
            |
            v
         Email

Infrastructure:
PostgreSQL + Elasticsearch + Kafka + Keycloak
                         |
                         v
                 Kubernetes / k3d
                         |
                       ArgoCD
```

### Core Stack

``` text
Java 21
Spring Boot 3.3.5
Spring Security
Spring Data JPA
PostgreSQL
Apache Kafka
Elasticsearch
Docker
Kubernetes
Next.js
React
```

------------------------------------------------------------------------

## 📄 License

This project is licensed under the MIT License.
