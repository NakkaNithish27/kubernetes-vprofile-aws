# Architecture

[← Back to README](../README.md)

## 1. Architecture Overview

This project deploys the existing containerized **vProfile multi-tier application** onto a Kubernetes cluster running on AWS.

The application is composed of four primary workloads:

```text
vProfile Application / Tomcat
MySQL
Memcached
RabbitMQ
```

Each workload is represented by a Kubernetes Deployment and exposed internally through a Kubernetes Service.

The overall architecture connects:

```text
External User
      │
      ▼
DNS
      │
      ▼
AWS Load Balancer
      │
      ▼
Ingress Controller
      │
      ▼
Ingress
      │
      ▼
Application Service
      │
      ▼
vProfile Application Pod
      │
      ├──────────────► MySQL Service
      │                    │
      │                    ▼
      │                 MySQL Pod
      │                    │
      │                    ▼
      │              Persistent Storage
      │
      ├──────────────► Memcached Service
      │                    │
      │                    ▼
      │               Memcached Pod
      │
      └──────────────► RabbitMQ Service
                           │
                           ▼
                      RabbitMQ Pod
```

The Kubernetes deployment architecture is represented through manifests stored under `kubedefs/`. The complete definition set contains the Secret, database PVC, four Deployments, four Services, and the Ingress resource.

---

## 2. Workload Architecture

### 2.1 vProfile Application

The application tier runs on **Tomcat** inside a Kubernetes Pod managed by a Deployment.

```text
vProfile Deployment
        │
        ▼
vProfile Pod
        │
        ▼
vproapp Service
```

The Service provides a stable internal endpoint for traffic destined for the application Pods.

The application workload itself is an existing workload used by this project; this repository focuses on the Kubernetes/AWS deployment engineering around it.

### 2.2 MySQL

MySQL provides the database tier.

```text
MySQL Deployment
       │
       ▼
   MySQL Pod
       │
       ├── Secret
       │
       └── PVC
            │
            ▼
       StorageClass
            │
            ▼
         AWS EBS
```

The database requires persistent storage because database data should survive the lifecycle of an individual Pod.

The application reaches MySQL through its Kubernetes Service rather than directly addressing the MySQL Pod.

```text
vProfile Pod
     │
     ▼
vproDB Service
     │
     ▼
MySQL Pod
```

### 2.3 Memcached

Memcached provides the caching tier.

```text
Memcached Deployment
        │
        ▼
   Memcached Pod
        │
        ▲
        │
vprocache Service
```

The application communicates with Memcached through the Kubernetes Service name.

### 2.4 RabbitMQ

RabbitMQ provides the message-queue tier.

```text
RabbitMQ Deployment
        │
        ▼
   RabbitMQ Pod
        │
        ▲
        │
vpromq Service
```

The application uses the Kubernetes Service as the stable endpoint for communication with RabbitMQ.

---

# 3. Kubernetes Object Mapping

| Application / Requirement | Kubernetes Object | Purpose |
|---|---|---|
| Application workload | Deployment + Pod | Run the vProfile application |
| MySQL workload | Deployment + Pod | Run the database |
| Memcached workload | Deployment + Pod | Run the cache |
| RabbitMQ workload | Deployment + Pod | Run the message queue |
| Internal application networking | ClusterIP Services | Stable service endpoints |
| Database credentials | Secret | Inject sensitive configuration |
| Database persistence | PersistentVolumeClaim | Request persistent storage |
| AWS persistent storage | StorageClass | Provision storage backing |
| External routing | Ingress | Route external requests |
| Ingress implementation | Ingress Controller | Process Ingress rules and provide external entry point |

This mapping is the central architectural pattern of the project.

---

# 4. Deployment Architecture

The workloads are managed through Kubernetes Deployments.

Conceptually:

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ▼
Pod
    │
    ▼
Container
```

The Deployment provides lifecycle management for the workload rather than requiring individual Pods to be manually maintained.

The four application components are represented as Kubernetes Deployments:

```text
vproapp
vproDB
vprocache
vpromq
```

Each Deployment is associated with a corresponding Service for internal communication.

---

# 5. Service Architecture

Kubernetes Services provide stable networking endpoints in front of the Pods.

The project uses **ClusterIP Services** for internal application communication.

```text
                 Kubernetes Cluster
┌─────────────────────────────────────────────────────┐
│                                                     │
│   vproapp Service ───────► vProfile Pod             │
│                                                     │
│   vproDB Service ────────► MySQL Pod               │
│                                                     │
│   vprocache Service ─────► Memcached Pod            │
│                                                     │
│   vpromq Service ────────► RabbitMQ Pod             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

A Service uses label selectors to identify the Pods that belong to it.

This creates an important separation:

```text
Application
    ↓
Service
    ↓
Pod
```

The application does not need to track individual Pod IP addresses.

The practical validates this relationship by inspecting Service endpoints. A Service with the correct endpoint indicates that its selector has successfully identified a matching Pod.

---

# 6. Kubernetes Service Discovery

One of the most important architectural concepts in this project is **service-name-based discovery**.

The application configuration uses Kubernetes Service names rather than Pod IP addresses.

Conceptually:

```text
application.properties

database host  → vproDB
database port  → 3306

cache host     → vprocache
cache port     → 11211

MQ host        → vpromq
```

The resulting flow is:

```text
Application Pod
      │
      │ DNS query
      ▼
Kubernetes DNS
      │
      ▼
Service
      │
      ▼
ClusterIP
      │
      ▼
Service selector
      │
      ▼
Backend Pod
```

This is important because Pod IP addresses are not treated as permanent application endpoints.

---

# 7. Persistent Storage Architecture

MySQL is the stateful component of the application.

The storage relationship is:

```text
MySQL Pod
    │
    ▼
PersistentVolumeClaim
    │
    ▼
StorageClass
    │
    ▼
AWS EBS Volume
```

### PersistentVolumeClaim

The PVC represents the application's request for persistent storage.

### StorageClass

The StorageClass provides the mechanism for provisioning the backing storage.

### AWS EBS

The resulting persistent storage is backed by an AWS EBS volume.

This creates a separation between:

```text
Pod lifecycle
      ≠
Database data lifecycle
```

If the MySQL Pod is recreated or rescheduled, the database data is associated with the persistent storage rather than existing only inside the container filesystem.

---

# 8. Secret Architecture

Sensitive configuration is represented through a Kubernetes Secret.

Conceptually:

```text
Kubernetes Secret
       │
       │ injected into
       ▼
MySQL Container
       │
       ▼
MYSQL_ROOT_PASSWORD
```

The Secret prevents the password from being directly embedded as ordinary application configuration.

### Repository security boundary

Real credentials must **never** be committed to this repository.

The published `secret.yaml` must therefore contain sanitized values or placeholders.

---

# 9. External Traffic Architecture

External traffic enters the Kubernetes application through the Ingress layer.

The high-level flow is:

```text
User
  │
  ▼
DNS / Hostname
  │
  ▼
AWS Load Balancer
  │
  ▼
Ingress Controller
  │
  ▼
Ingress Resource
  │
  ▼
vproapp Service
  │
  ▼
vProfile Application Pod
```

The Ingress resource contains routing rules that determine which Service receives an incoming request.

The Ingress Controller implements those rules and provides the external entry point.

The project therefore separates:

```text
Ingress Resource
      =
routing configuration

Ingress Controller
      =
component that processes the routing configuration
```

---

# 10. Complete Request Flow

The complete request path can be represented as:

```text
                         USER
                          │
                          ▼
                     DNS / HOST
                          │
                          ▼
                 AWS LOAD BALANCER
                          │
                          ▼
                 INGRESS CONTROLLER
                          │
                          ▼
                       INGRESS
                    host / path rule
                          │
                          ▼
                 vproapp SERVICE
                    ClusterIP
                          │
                  label selector
                          │
                          ▼
                 VPROFILE APP POD
                    Tomcat
                   /   |                      /    |                      ▼     ▼      ▼
             vproDB  vprocache vpromq
             Service Service   Service
                │       │        │
                ▼       ▼        ▼
             MySQL   Memcached RabbitMQ
              Pod       Pod       Pod
               │
               ▼
              PVC
               │
               ▼
          StorageClass
               │
               ▼
            AWS EBS
```

This is the central architecture to understand for the entire project.

---

# 11. Application-to-Backend Flow

Once the request reaches the vProfile application Pod, the application communicates with its backend services.

### Database

```text
vProfile Pod
     │
     ▼
vproDB
     │
     ▼
MySQL Pod
     │
     ▼
Persistent Storage
```

### Cache

```text
vProfile Pod
     │
     ▼
vprocache
     │
     ▼
Memcached Pod
```

### Message Queue

```text
vProfile Pod
     │
     ▼
vpromq
     │
     ▼
RabbitMQ Pod
```

The application therefore does not need direct knowledge of individual backend Pod IP addresses.

---

# 12. Architecture Layers

The project can be understood as several logical layers.

```text
┌─────────────────────────────────────────────┐
│  External Access                             │
│  DNS → Load Balancer → Ingress               │
├─────────────────────────────────────────────┤
│  Application                                │
│  vproapp Service → Tomcat Pod                │
├─────────────────────────────────────────────┤
│  Internal Services                           │
│  vproDB / vprocache / vpromq                 │
├─────────────────────────────────────────────┤
│  Backend Workloads                           │
│  MySQL / Memcached / RabbitMQ                │
├─────────────────────────────────────────────┤
│  Persistence                                 │
│  PVC → StorageClass → AWS EBS                │
├─────────────────────────────────────────────┤
│  Configuration                               │
│  Kubernetes Secret                           │
├─────────────────────────────────────────────┤
│  Cluster Infrastructure                     │
│  kOps → AWS Kubernetes cluster               │
└─────────────────────────────────────────────┘
```

This layered view makes it possible to reason about failures by architectural boundary.

---

# 13. kOps and AWS Infrastructure

The Kubernetes cluster is operated using **kOps** on AWS.

At the infrastructure level:

```text
kOps
  │
  ├── Kubernetes cluster
  ├── EC2 infrastructure
  ├── networking
  ├── security infrastructure
  └── AWS-integrated resources
```

The kOps workflow also uses AWS resources for cluster state and DNS-related infrastructure.

The practical describes creating the cluster through kOps after preparing AWS credentials, an S3 bucket for cluster state, and a Route 53 hosted zone.

---

# 14. Build and Deployment Boundary

The project intentionally separates application image creation from Kubernetes deployment.

The conceptual boundary is:

```text
Application / Container Build
          │
          ▼
      Container Image
          │
          ▼
       Docker Hub
          │
          ▼
   Kubernetes Deployment
          │
          ▼
         Pod
```

The Kubernetes environment does not need to rebuild the application source.

Instead, Kubernetes pulls the required container images when the Deployments are applied.

---

# 15. Namespace and Resource Organization

The primary Kubernetes resources are grouped around the application deployment:

```text
Namespace
   │
   ├── Secret
   ├── PVC
   ├── Deployments
   │     ├── vproapp
   │     ├── vproDB
   │     ├── vprocache
   │     └── vpromq
   │
   ├── Services
   │     ├── vproapp
   │     ├── vproDB
   │     ├── vprocache
   │     └── vpromq
   │
   └── Ingress
```

The exact namespace/resource details should be taken from the final manifests used for the completed environment rather than inferred beyond the source material.

---

# 16. Architectural Decisions

## Use Deployments for workloads

Deployments provide lifecycle management for the application's Pods and allow Kubernetes to maintain the desired workload state.

## Use Services for internal communication

Services provide stable endpoints between application components.

This avoids coupling application configuration directly to individual Pod IP addresses.

## Use PVC for MySQL

The database requires persistence beyond an individual Pod lifecycle.

The PVC provides the Kubernetes abstraction for requesting that persistent storage.

## Use a Secret for credentials

Database credentials are configuration data that should not be treated as ordinary application configuration.

The Secret provides a Kubernetes mechanism for injecting the sensitive value into the workload.

## Use Ingress for external routing

The application requires an external entry point.

Ingress provides host/path routing while the Ingress Controller implements that routing.

## Use kOps for the AWS Kubernetes cluster

kOps provides the cluster lifecycle and infrastructure management workflow used in this project.

---

# 17. Dependency Relationships

The important dependencies can be summarized as:

```text
                         Ingress
                            │
                            ▼
                    vproapp Service
                            │
                            ▼
                     vProfile Pod
                    /       |                          /        |                          ▼         ▼         ▼
             vproDB    vprocache    vpromq
             Service    Service     Service
                │          │           │
                ▼          ▼           ▼
             MySQL     Memcached    RabbitMQ
                │
                ▼
               PVC
                │
                ▼
          StorageClass
                │
                ▼
              EBS
```

Configuration adds another dependency:

```text
Secret
  │
  ▼
MySQL Pod
```

Infrastructure provides the underlying environment:

```text
kOps
  │
  ▼
AWS Kubernetes Cluster
  │
  ├── Compute
  ├── Networking
  └── AWS-integrated resources
```

---

# 18. Failure-Domain Reasoning

The architecture also provides a useful way to isolate failures.

### Application cannot reach database

Inspect:

```text
Application configuration
        ↓
vproDB Service
        ↓
Service endpoints
        ↓
MySQL Pod
        ↓
MySQL configuration
        ↓
Persistent storage
```

### Application is not externally accessible

Inspect:

```text
DNS
 ↓
AWS Load Balancer
 ↓
Ingress Controller
 ↓
Ingress
 ↓
vproapp Service
 ↓
vProfile Pod
```

### Service has no endpoints

Inspect:

```text
Service selector
       ↓
Pod labels
       ↓
Pod status
```

This is why Service endpoint inspection is a useful diagnostic boundary in the project.

---

# 19. Complete System Model

```text
                  ┌──────────────────────┐
                  │       Internet       │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    DNS / Hostname    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ AWS Load Balancer    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Ingress Controller   │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Ingress              │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ vproapp Service      │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ vProfile / Tomcat    │
                  │ Pod                  │
                  └──────────┬───────────┘
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
       vproDB Service   vprocache Service  vpromq Service
             │               │                │
             ▼               ▼                ▼
        MySQL Pod       Memcached Pod     RabbitMQ Pod
             │
             ▼
            PVC
             │
             ▼
        StorageClass
             │
             ▼
          AWS EBS
```

---

# 20. Architecture Boundary

This architecture demonstrates a working Kubernetes deployment pattern around an existing multi-tier workload.

It should **not** be interpreted as a claim that the project implements a complete production Kubernetes platform.

The demonstrated architecture does not establish:

- Terraform-managed infrastructure
- Helm-based packaging
- CI/CD
- production observability
- comprehensive security hardening
- advanced production autoscaling
- production disaster recovery
- zero-downtime deployment guarantees

These capabilities belong to potential future iterations rather than this project's demonstrated architecture.

---

## Related Documentation

- [← Back to README](../README.md)
- [Implementation](implementation.md)
- [Validation](validation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
