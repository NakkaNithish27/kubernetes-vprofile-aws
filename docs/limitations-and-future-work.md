# Limitations and Future Work

[← Back to README](../README.md)

## 1. Purpose

This document defines the boundaries of the **Kubernetes vProfile deployment on AWS** project and identifies logical areas for future improvement.

The project demonstrates Kubernetes deployment engineering around an existing multi-tier application. It should not be interpreted as a complete production Kubernetes platform.

---

# 2. Current Project Scope

The project demonstrates:

- Kubernetes cluster operation on AWS using kOps
- Kubernetes Deployments
- Kubernetes Services
- Kubernetes Secrets
- PersistentVolumeClaims
- AWS-backed persistent storage
- Ingress
- Ingress Controller
- DNS-based external access
- Kubernetes service discovery
- Multi-tier application deployment
- Deployment validation
- Service endpoint validation
- Troubleshooting
- Declarative cleanup

The implementation focuses on understanding how these components work together to deploy and expose the existing vProfile workload.

---

# 3. Application Ownership Limitation

The vProfile application used in this project is an **existing application workload**.

This project does not claim ownership of:

- vProfile business logic
- Java application development
- Original application source code
- Application feature development
- Original application architecture

The engineering contribution represented here is the Kubernetes/AWS deployment layer surrounding the workload.

The appropriate portfolio framing is therefore:

```text
Existing Application
        ↓
Containerized Workload
        ↓
Kubernetes Deployment
        ↓
AWS Infrastructure
        ↓
Validation & Operations
```

rather than:

```text
Application Development Project
```

---

# 4. Infrastructure-as-Code Limitation

The Kubernetes cluster infrastructure is managed through **kOps**, rather than Terraform.

Terraform-based infrastructure provisioning is therefore outside the demonstrated scope.

The current implementation does not provide a Terraform configuration for:

- VPC
- EC2
- IAM
- Route 53
- EBS-related infrastructure
- Kubernetes cluster provisioning

### Future Work

A future iteration could reproduce the AWS infrastructure using Terraform.

Potential target architecture:

```text
Terraform
   │
   ├── VPC
   ├── IAM
   ├── EC2
   ├── DNS
   └── supporting AWS resources
           │
           ▼
      Kubernetes
```

This would improve infrastructure reproducibility and strengthen Infrastructure-as-Code skills.

---

# 5. Helm Limitation

The Kubernetes resources are maintained as individual YAML manifests.

The project does not use Helm.

Current model:

```text
kubedefs/
├── deployment.yaml
├── service.yaml
├── pvc.yaml
├── secret.yaml
└── ingress.yaml
```

### Future Work

A future iteration could package the application using a Helm chart.

Potential structure:

```text
vprofile-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── pvc.yaml
    ├── secret.yaml
    └── ingress.yaml
```

Helm would provide reusable configuration and environment-specific values.

---

# 6. CI/CD Limitation

The current project does not implement a complete CI/CD pipeline.

The deployment workflow is primarily manual:

```text
Manifest Change
      ↓
Git Repository
      ↓
Clone / Pull
      ↓
kubectl
      ↓
Kubernetes
```

There is no demonstrated automated pipeline that performs:

```text
Git Push
   ↓
Build
   ↓
Test
   ↓
Container Image
   ↓
Registry
   ↓
Deploy
   ↓
Validation
```

### Future Work

A future iteration could implement:

- GitHub Actions
- automated Docker image builds
- image tagging
- vulnerability scanning
- Kubernetes deployment
- rollout verification
- automated rollback on failure

This would convert the project from a manually operated deployment into a CI/CD-driven delivery workflow.

---

# 7. Observability Limitation

The project validates workloads primarily through Kubernetes and application-level checks.

It does not implement a complete production observability stack.

Not demonstrated:

- Prometheus
- Grafana
- centralized logging
- distributed tracing
- alerting
- long-term metrics storage
- SLO/SLI monitoring

### Future Work

A future production-oriented iteration could introduce:

```text
Applications
    │
    ├── Metrics
    ├── Logs
    └── Traces
         │
         ▼
   Observability Stack
         │
    ┌────┼────┐
    ▼    ▼    ▼
Metrics Logs Traces
    │
    ▼
Dashboards / Alerts
```

This would allow the platform to be monitored rather than only manually validated.

---

# 8. Security Hardening Limitation

The project demonstrates Secret-based configuration, but it does not represent comprehensive Kubernetes security hardening.

The project does not demonstrate:

- Pod Security Standards enforcement
- NetworkPolicies
- RBAC least-privilege design
- image signing
- admission control
- runtime security
- secrets management through an external secrets platform
- comprehensive vulnerability scanning

### Future Work

A security-focused iteration could introduce:

```text
RBAC
NetworkPolicy
Pod Security
Image Scanning
Secret Management
Admission Control
Runtime Security
```

The goal would be to move from:

```text
Working Kubernetes deployment
```

toward:

```text
Hardened Kubernetes deployment
```

---

# 9. Secret Management Limitation

Although Kubernetes Secrets are used, Kubernetes Secrets alone should not be treated as a complete enterprise secrets-management solution.

The repository must contain sanitized values and must never contain real credentials.

### Future Work

A production implementation could integrate an external secrets-management solution and establish:

- centralized secret storage
- controlled access
- rotation
- auditability
- environment-specific credentials

The implementation could therefore evolve from:

```text
Kubernetes Secret
```

to:

```text
External Secret Store
        ↓
External Secrets Integration
        ↓
Kubernetes Secret
        ↓
Application
```

---

# 10. High Availability Limitation

The project demonstrates Kubernetes workload deployment but does not establish a comprehensive production high-availability architecture.

The current project should not claim:

- multi-region availability
- disaster recovery
- guaranteed zero downtime
- tested failover
- production SLA compliance

### Future Work

Future work could introduce:

- multiple replicas
- topology-aware scheduling
- PodDisruptionBudgets
- multi-AZ architecture
- controlled failover testing
- backup and restore procedures
- disaster recovery testing

---

# 11. Database Reliability Limitation

MySQL persistence is implemented using a PersistentVolumeClaim backed by AWS storage.

However, persistent storage alone does not constitute a complete database disaster-recovery strategy.

The project does not demonstrate:

- automated database backups
- point-in-time recovery
- cross-region replication
- tested database restoration
- managed database migration

### Future Work

A future iteration could introduce:

```text
Application
    ↓
Highly Available Database
    ↓
Automated Backups
    ↓
Restore Testing
    ↓
Disaster Recovery
```

The purpose would be to demonstrate that application data can be recovered, not merely persisted.

---

# 12. Autoscaling Limitation

The project does not demonstrate a production autoscaling strategy.

It does not establish:

- Horizontal Pod Autoscaler
- Vertical Pod Autoscaler
- cluster autoscaling
- resource-based scaling policies
- load-based capacity planning

### Future Work

A future iteration could implement:

```text
Traffic / CPU / Memory
        ↓
Metrics
        ↓
HPA
        ↓
Replica Count
```

and validate scaling behavior under controlled load.

---

# 13. Resource Management Limitation

The project does not establish a comprehensive resource-governance model for every workload.

Future production configuration should consider:

```text
CPU Requests
CPU Limits
Memory Requests
Memory Limits
```

along with workload-specific sizing.

This would make scheduling behavior and resource consumption more predictable.

---

# 14. Network Security Limitation

The project validates Kubernetes Service networking but does not demonstrate fine-grained network isolation.

For example, the architecture allows the application to communicate with backend services through Kubernetes networking, but the project does not establish a complete policy model describing which Pods are allowed to communicate with which other Pods.

### Future Work

NetworkPolicies could establish rules such as:

```text
vProfile
   │
   ├──► MySQL
   ├──► Memcached
   └──► RabbitMQ

Other workloads
   ✕
Backend services
```

This would reduce unnecessary east-west network access.

---

# 15. Ingress and TLS Limitation

The project demonstrates external routing through Ingress but does not establish a complete production TLS lifecycle.

A production implementation should consider:

- TLS certificates
- certificate renewal
- HTTPS-only access
- HTTP-to-HTTPS redirection
- certificate monitoring

### Future Work

A future implementation could use automated certificate management:

```text
Ingress
   ↓
TLS Certificate
   ↓
HTTPS
```

with automated certificate issuance and renewal.

---

# 16. Testing Limitation

The project validates the deployment primarily through operational checks and application access.

It does not demonstrate a comprehensive automated test suite.

Not demonstrated:

- unit tests
- integration tests
- end-to-end automated tests
- load tests
- chaos testing
- failure-injection testing

### Future Work

A CI/CD-oriented implementation could introduce:

```text
Code
 ↓
Unit Tests
 ↓
Build
 ↓
Integration Tests
 ↓
Container Scan
 ↓
Deployment
 ↓
End-to-End Tests
```

---

# 17. Disaster Recovery Limitation

The project includes cleanup and reconstruction concepts but does not provide a formally tested disaster recovery plan.

There is no demonstrated:

- RTO
- RPO
- regional failover
- cluster restore procedure
- database restore drill
- infrastructure recovery drill

### Future Work

A future project iteration could explicitly define:

```text
Failure Scenario
       ↓
Recovery Procedure
       ↓
Restore
       ↓
Validation
       ↓
Measured Recovery Time
```

This would convert disaster recovery from documentation into a tested operational capability.

---

# 18. Production Traffic Limitation

The project demonstrates external application access but does not establish production-scale traffic handling.

There is no demonstrated load-testing methodology for determining:

- maximum sustainable requests
- application throughput
- database bottlenecks
- cache effectiveness
- message-queue throughput
- scaling thresholds

### Future Work

A performance-focused iteration could introduce controlled load testing and measure:

```text
Requests/sec
Latency
Error rate
CPU
Memory
Database load
Pod scaling
```

---

# 19. Environment Management Limitation

The project is focused on a single demonstrated environment.

It does not implement a complete multi-environment promotion strategy such as:

```text
Development
     ↓
Staging
     ↓
Production
```

with separate configuration and controlled promotion.

### Future Work

Environment-specific configuration could be introduced through:

- Helm values
- Kustomize overlays
- separate namespaces
- separate clusters
- GitOps environments

---

# 20. GitOps Limitation

The project uses Git as the repository for Kubernetes definitions, but it does not implement a full GitOps reconciliation system.

Current model:

```text
Git
 ↓
Human
 ↓
kubectl
 ↓
Kubernetes
```

A GitOps model would instead look like:

```text
Git
 ↓
GitOps Controller
 ↓
Kubernetes
```

### Future Work

A future iteration could evaluate tools such as:

- Argo CD
- Flux

The goal would be continuous reconciliation between the declared Git state and the Kubernetes cluster state.

---

# 21. Cost Optimization Limitation

The project demonstrates AWS deployment but does not establish a formal cost-optimization strategy.

It does not provide:

- workload cost allocation
- resource-rightsizing analysis
- automated shutdown policies
- reserved capacity analysis
- production cost forecasting

### Future Work

A future AWS iteration could evaluate:

```text
Resource Usage
      ↓
Rightsizing
      ↓
Cost Measurement
      ↓
Optimization
```

---

# 22. Current Project Strengths

Despite the limitations above, the project demonstrates several important Kubernetes engineering capabilities.

### Kubernetes fundamentals

The project demonstrates understanding of:

- Deployments
- Pods
- Services
- Service discovery
- Secrets
- PersistentVolumeClaims
- Ingress

### AWS integration

The project demonstrates Kubernetes operation on AWS and integration with:

- EC2
- EBS
- Route 53
- Load balancing
- S3-backed kOps state

### Operational thinking

The project does not stop at resource creation.

It includes:

```text
Deploy
 ↓
Observe
 ↓
Validate
 ↓
Troubleshoot
 ↓
Correct
 ↓
Validate Again
 ↓
Clean Up
```

This operational lifecycle is an important part of the project's value.

---

# 23. Future Iteration Roadmap

A logical progression for extending this project is:

```text
Current Project
Kubernetes + AWS + kOps
        │
        ▼
Iteration 1
Terraform
        │
        ▼
Iteration 2
Helm
        │
        ▼
Iteration 3
CI/CD
        │
        ▼
Iteration 4
Observability
        │
        ▼
Iteration 5
Security Hardening
        │
        ▼
Iteration 6
Autoscaling / HA
        │
        ▼
Iteration 7
GitOps
        │
        ▼
Production-Grade Platform
```

Each iteration should introduce a clearly measurable engineering capability rather than adding tools without a specific operational purpose.

---

# 24. Recommended Priority

If this project is extended for career portfolio purposes, the highest-value sequence is:

### Priority 1 — Infrastructure as Code

Add Terraform to make the AWS environment reproducible.

```text
Terraform
    ↓
AWS
    ↓
Kubernetes
```

### Priority 2 — CI/CD

Automate:

```text
Git Push
   ↓
Build
   ↓
Test
   ↓
Image
   ↓
Deploy
   ↓
Validate
```

### Priority 3 — Observability

Add:

```text
Metrics
Logs
Alerts
Dashboards
```

### Priority 4 — Security

Add:

```text
RBAC
NetworkPolicy
Image scanning
Secret management
Pod security
```

### Priority 5 — Scaling and Reliability

Add:

```text
HPA
Multi-AZ
Backups
Disaster Recovery
Failure Testing
```

### Priority 6 — GitOps

Finally, replace manual deployment synchronization with:

```text
Git
 ↓
GitOps Controller
 ↓
Kubernetes
```

---

# 25. Portfolio Boundary

The project should be presented accurately in a professional portfolio.

A strong description is:

> **Deployed and validated a containerized multi-tier vProfile application on an AWS Kubernetes cluster using kOps, implementing Kubernetes Deployments, Services, Secrets, persistent storage, Ingress, DNS-based external access, service discovery, troubleshooting, and infrastructure cleanup.**

Avoid claiming:

- Kubernetes platform engineering at production scale
- production-grade security
- complete CI/CD
- production observability
- disaster recovery
- zero-downtime deployments
- Terraform-based infrastructure

unless those capabilities are actually implemented and validated in a later iteration.

---

# 26. Final Boundary

The project currently establishes the following capability:

```text
Containerized Application
        ↓
Kubernetes Architecture
        ↓
AWS Kubernetes Cluster
        ↓
Workload Deployment
        ↓
Service Networking
        ↓
Persistent Storage
        ↓
Ingress / External Access
        ↓
Validation
        ↓
Troubleshooting
        ↓
Cleanup
```

The next stage is not to add complexity for its own sake.

The strongest progression is to turn the demonstrated deployment into a **reproducible, automated, observable, secure, and resilient platform** through incremental iterations.

---

## Related Documentation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Implementation](implementation.md)
- [Validation](validation.md)
