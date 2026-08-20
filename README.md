# Kubernetes vProfile Application Deployment on AWS

A hands-on Kubernetes deployment project that deploys an existing containerized multi-tier **vProfile** workload onto an AWS-hosted Kubernetes cluster using **kOps**, Kubernetes Deployments and Services, persistent storage, Secrets, and Ingress.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/8f71786d-9ef7-4836-8a3e-70bc7b0671dd" />


> **Portfolio focus:** This repository represents the Kubernetes/AWS deployment engineering performed around the application workload. It does **not** claim ownership of the vProfile application's business logic or original application development.

---

## Overview

This project demonstrates the deployment and operation of a multi-tier application on Kubernetes running in AWS.

The workload consists of:

- **Tomcat / vProfile application**
- **MySQL**
- **Memcached**
- **RabbitMQ**

The Kubernetes deployment uses:

- Deployments for workload management
- ClusterIP Services for internal communication
- Kubernetes Secret for sensitive configuration
- PersistentVolumeClaim for MySQL persistence
- StorageClass-backed AWS EBS storage
- Ingress for external application routing
- Kubernetes service discovery for application-to-backend communication
- AWS infrastructure provisioned and managed through kOps

The project was taken through deployment, validation, troubleshooting, and cleanup.

---

## Ownership Boundary

The **vProfile application was used as the existing workload** for this project.

My engineering contribution was the Kubernetes/AWS deployment layer around that workload, including:

- Kubernetes cluster operation with kOps
- Kubernetes manifest implementation
- Deployment configuration
- Service configuration
- Secret configuration
- Persistent storage configuration
- Ingress configuration
- DNS integration
- Application deployment
- Service and backend validation
- Troubleshooting
- Infrastructure cleanup

I did **not** develop the vProfile application's business logic, Java backend, authentication implementation, or original application source.

The repository therefore represents the **DevOps/Kubernetes engineering performed around the application**, rather than application development.

---

## Architecture

```text
                         INTERNET
                            │
                            ▼
                    DNS / Route 53
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
                  vproapp Service
                     (ClusterIP)
                            │
                            ▼
                  vProfile App Pod
                     (Tomcat)
                    /     |      \
                   /      |       \
                  ▼       ▼        ▼
           vproDB      vprocache   vpromq
           Service      Service    Service
              │            │          │
              ▼            ▼          ▼
          MySQL Pod    Memcached   RabbitMQ
              │            Pod        Pod
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

The application uses Kubernetes **Service names** for backend connectivity rather than relying on Pod IP addresses.

This allows the application to communicate with stable Kubernetes service endpoints while Pods can be recreated or rescheduled.

For the complete architecture and request flow:

**[Architecture](docs/architecture.md)**

---

## Engineering Contribution

### 1. Kubernetes Workload Deployment

Implemented Kubernetes Deployments for the application's major components:

```text
vProfile Application
MySQL
Memcached
RabbitMQ
```

Deployments provide the Kubernetes workload-management layer for these components.

---

### 2. Internal Service Networking

Configured Kubernetes ClusterIP Services to provide stable internal endpoints for the workloads.

The application communicates with backend services through Kubernetes service discovery.

Conceptually:

```text
vProfile Pod
     │
     ├──→ vproDB:3306
     ├──→ vprocache:11211
     └──→ vpromq:15672
```

The Services then select the appropriate backend Pods.

---

### 3. Persistent Database Storage

Configured persistent storage for MySQL using:

```text
PersistentVolumeClaim
        ↓
StorageClass
        ↓
AWS EBS
```

This separates database data persistence from the lifecycle of the MySQL Pod.

---

### 4. Secret-Based Configuration

Configured a Kubernetes Secret for sensitive application/database configuration.

Credentials are injected into the workload rather than being intended as plain-text configuration in the application deployment.

> Published manifests must contain sanitized placeholders and must never contain real credentials.

---

### 5. External Application Access

Configured Kubernetes Ingress for external application routing.

The traffic path is:

```text
User
 ↓
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

---

### 6. Cluster and Infrastructure Operations

Used **kOps** to operate the Kubernetes cluster on AWS.

The project also involved AWS infrastructure associated with:

- EC2
- EBS
- Route 53
- S3-backed kOps state
- AWS load balancing

---

### 7. Validation and Troubleshooting

Validated the deployment progressively rather than relying only on browser access.

Validation included:

```text
Cluster
  ↓
Nodes
  ↓
Deployments
  ↓
Pods
  ↓
Services
  ↓
Service Endpoints
  ↓
PVC / Storage
  ↓
Ingress
  ↓
DNS
  ↓
Application
  ↓
Backend connectivity
```

Troubleshooting focused on identifying failures at the appropriate Kubernetes layer and correcting the relevant resource before continuing.

More detail:

**[Validation](docs/validation.md)**

---

## Deployment Model

The project separates **container image creation** from **Kubernetes deployment**.

The Kubernetes environment consumes the existing container images rather than rebuilding the application source on the cluster-management machine.

The repository therefore focuses on the Kubernetes deployment definitions under:

```text
kubedefs/
```

The directory contains the Kubernetes resources required for the deployment.

---

## Kubernetes Resources

The deployment is represented by:

```text
kubedefs/
│
├── secret.yaml
├── db-pvc.yaml
├── db-deployment.yaml
├── db-service.yaml
├── mc-deployment.yaml
├── mc-service.yaml
├── rmq-deployment.yaml
├── rmq-service.yaml
├── app-deployment.yaml
├── app-service.yaml
└── ingress.yaml
```

These manifests represent the deployment configuration rather than the source code of the vProfile application.

---

## Validation Result

The completed deployment flow was validated across multiple layers:

- Kubernetes cluster availability
- Node availability
- Deployment status
- Pod status
- Service configuration
- Service-to-Pod endpoints
- PersistentVolumeClaim provisioning
- Ingress configuration
- External application access
- Application-to-backend connectivity
- Database persistence
- Cache and message-queue connectivity

The project was subsequently cleaned up to remove the Kubernetes workloads and associated AWS resources.

---

## Project Boundaries

This project demonstrates **Kubernetes deployment engineering**, not every capability associated with a production Kubernetes platform.

The following are **outside the demonstrated scope of this iteration**:

- Terraform-based infrastructure provisioning
- Helm-based application packaging
- CI/CD pipeline implementation
- Production observability stack
- Comprehensive Kubernetes security hardening
- Advanced production autoscaling strategy
- Production-grade disaster recovery
- Zero-downtime deployment guarantees

These are potential areas for future iterations rather than capabilities claimed by this project.

---

## Technologies

### Kubernetes

- Kubernetes
- kubectl
- kOps
- NGINX Ingress Controller

### AWS

- Amazon EC2
- Amazon EBS
- Amazon Route 53
- Amazon S3
- AWS Load Balancing

### Containers

- Docker
- Docker Hub

### Application Workloads

- Apache Tomcat
- MySQL
- Memcached
- RabbitMQ

---

## Project Documentation

### Architecture

Detailed component relationships, service discovery, storage, ingress, and request flow.

**[Read Architecture →](docs/architecture.md)**

### Implementation

Cluster setup, manifest implementation, deployment sequence, configuration, troubleshooting, and cleanup.

**[Read Implementation →](docs/implementation.md)**

### Validation

Validation methodology, checks performed, evidence mapping, and what each validation step proves.

**[Read Validation →](docs/validation.md)**

### Limitations & Future Work

Project boundaries, capabilities not demonstrated, and logical next iterations.

**[Read Limitations & Future Work →](docs/limitations-and-future-work.md)**

---

## Evidence

High-signal execution evidence can be found here when populated with screenshots from the completed environment:

```text
evidence/screenshots/
```

Evidence should demonstrate actual execution and validation rather than reproduce course material.

Examples include:

- Cluster state
- Workload state
- Service endpoints
- Ingress/storage state
- Final application validation

---

## Engineering Takeaway

The central engineering pattern demonstrated by this project is:

```text
Understand the application
        ↓
Map application components to Kubernetes objects
        ↓
Implement manifests
        ↓
Deploy to Kubernetes
        ↓
Connect workloads through Services
        ↓
Provide persistent storage where required
        ↓
Expose the application through Ingress
        ↓
Validate the complete request path
        ↓
Troubleshoot failures
        ↓
Clean up infrastructure
```

The important capability is not memorizing individual `kubectl` commands.

It is understanding **how Kubernetes objects work together to run an application** and being able to reconstruct, deploy, validate, troubleshoot, and explain that architecture.

---

## Repository Structure

```text
kubernetes-vprofile-aws/
│
├── README.md
│
├── kubedefs/
│   ├── secret.yaml
│   ├── db-pvc.yaml
│   ├── db-deployment.yaml
│   ├── db-service.yaml
│   ├── mc-deployment.yaml
│   ├── mc-service.yaml
│   ├── rmq-deployment.yaml
│   ├── rmq-service.yaml
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   └── ingress.yaml
│
├── docs/
│   ├── architecture.md
│   ├── implementation.md
│   ├── validation.md
│   └── limitations-and-future-work.md
│
├── evidence/
│   └── screenshots/
│
└── .gitignore
```

---

**Project focus:** Kubernetes deployment engineering on AWS, with explicit ownership boundaries around the existing application workload.
