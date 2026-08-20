# Validation

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/61420543-40b2-4c00-af0e-871f3e857300" />


## 1. Validation Overview

Validation verifies that the Kubernetes deployment is not only created, but that the individual components are functioning together as an application.

The validation strategy follows the architecture from the bottom upward:

```text
Kubernetes Cluster
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
Persistent Storage
        ↓
Ingress
        ↓
DNS / External Access
        ↓
Application
        ↓
Backend Connectivity
```

The goal is to distinguish between:

```text
Resource exists
      ≠
Resource is running
      ≠
Resource is connected
      ≠
Application works
```

---

# 2. Validation Strategy

Validation is performed progressively.

Instead of immediately testing the application through a browser, each architectural layer is checked independently.

```text
Layer 1 → Cluster
Layer 2 → Workloads
Layer 3 → Services
Layer 4 → Service endpoints
Layer 5 → Storage
Layer 6 → Ingress
Layer 7 → External access
Layer 8 → Application/backend functionality
```

This approach makes troubleshooting more precise because a failure can be associated with a specific layer.

---

# 3. Cluster Validation

## 3.1 Verify Kubernetes Nodes

Run:

```bash
kubectl get nodes
```

Expected result:

```text
NAME                         STATUS   ROLES
...                          Ready    ...
...                          Ready    ...
```

The important condition is:

```text
STATUS = Ready
```

A `NotReady` state during initial cluster creation does not necessarily indicate a permanent failure; cluster initialization may still be in progress.

---

## 3.2 What This Proves

A successful node check establishes that:

- the Kubernetes cluster is reachable
- `kubectl` is configured correctly
- Kubernetes nodes have joined the cluster
- the cluster is operational enough to proceed with workload deployment

It does **not** prove that the application workloads are healthy.

---

# 4. Secret Validation

Check the Secret:

```bash
kubectl get secret
```

The expected result is that the application/database Secret exists.

For example:

```text
NAME
app-secret
```

The Secret is required by workloads that consume its configuration.

---

## 4.1 What This Proves

This confirms:

```text
Secret object
     ↓
Exists in Kubernetes
```

It does not prove that the consuming application is correctly using the values.

### Security boundary

Do not print or commit actual Secret values as validation evidence.

Validation should demonstrate the existence and correct resource configuration without exposing credentials.

---

# 5. Persistent Storage Validation

MySQL requires persistent storage.

Check the PVC:

```bash
kubectl get pvc
```

Expected state:

```text
STATUS = Bound
```

Example:

```text
NAME       STATUS   VOLUME
vprodb     Bound    pvc-...
```

---

## 5.1 Storage Validation Chain

The storage relationship is:

```text
MySQL Pod
    ↓
PersistentVolumeClaim
    ↓
PersistentVolume
    ↓
StorageClass
    ↓
AWS EBS
```

The PVC being `Bound` provides evidence that the storage request has been successfully associated with persistent storage.

---

## 5.2 What This Proves

A `Bound` PVC establishes that:

- the PVC exists
- Kubernetes has satisfied the storage request
- the claim has been associated with a volume

It does not, by itself, prove that the application is successfully writing and reading database data.

---

# 6. Deployment Validation

Check the Deployments:

```bash
kubectl get deployments
```

The expected workloads are:

```text
vproapp
vproDB
vprocache
vpromq
```

The desired and available replica counts should indicate that Kubernetes has successfully created the expected workload Pods.

Conceptually:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

---

## 6.1 What This Proves

Deployment validation establishes that Kubernetes has created the workload-management resources and is attempting to maintain the desired state.

It does not by itself prove application functionality.

---

# 7. Pod Validation

Check the Pods:

```bash
kubectl get pods
```

Expected high-level state:

```text
vproapp       Running
vproDB        Running
vprocache     Running
vpromq        Running
```

The exact Pod names will contain generated identifiers.

---

## 7.1 Pod Status

The most important initial state is:

```text
STATUS = Running
```

However, a Running Pod alone does not prove that the application is functioning end-to-end.

The Pod must also be connected to the appropriate Service and, where applicable, its dependencies.

---

## 7.2 Investigating a Failed Pod

If a Pod is not running:

```bash
kubectl describe pod <pod-name>
```

This exposes Pod events and configuration information useful for identifying the failure.

The troubleshooting sequence is:

```text
Pod not healthy
      ↓
kubectl describe pod
      ↓
Inspect events/configuration
      ↓
Identify failing dependency
      ↓
Correct configuration
      ↓
Recreate/reapply resource
      ↓
Validate again
```

---

# 8. Service Validation

Check the Services:

```bash
kubectl get svc
```

Expected Services include:

```text
vproapp
vproDB
vprocache
vpromq
```

The Services should have the expected internal networking configuration.

Conceptually:

```text
vproapp Service
     ↓
vProfile Pod

vproDB Service
     ↓
MySQL Pod

vprocache Service
     ↓
Memcached Pod

vpromq Service
     ↓
RabbitMQ Pod
```

---

# 9. Service Endpoint Validation

A Service existing does not guarantee that it is connected to a Pod.

This is one of the most important validation checks in the project.

Inspect a Service:

```bash
kubectl describe svc vproapp
```

The output should contain an **Endpoints** field populated with the appropriate Pod endpoint.

Conceptually:

```text
Service
   │
   ├── Selector
   │
   ▼
Pod labels
   │
   ▼
Endpoint
```

The Service endpoint confirms that the Service selector is correctly identifying the application Pod.

---

## 9.1 Missing Endpoint

If a Service has no endpoint, inspect:

```text
Service selector
       ↓
Pod labels
       ↓
Pod status
```

Typical reasoning:

```text
Service exists
      ↓
No endpoint
      ↓
Selector may not match Pod
      ↓
Check labels
```

or:

```text
Service exists
      ↓
No endpoint
      ↓
Pod may not be Running
      ↓
Inspect Pod
```

---

# 10. Backend Service Validation

Each backend Service should be validated independently.

### MySQL

```bash
kubectl describe svc vproDB
```

Expected relationship:

```text
vproDB Service
      ↓
MySQL Pod
```

### Memcached

```bash
kubectl describe svc vprocache
```

Expected relationship:

```text
vprocache Service
      ↓
Memcached Pod
```

### RabbitMQ

```bash
kubectl describe svc vpromq
```

Expected relationship:

```text
vpromq Service
      ↓
RabbitMQ Pod
```

This verifies the internal networking layer before relying on application-level testing.

---

# 11. Ingress Validation

Check the Ingress:

```bash
kubectl get ingress
```

The expected result is that the Ingress resource exists and has the expected host/routing configuration.

The architecture should resolve to:

```text
Ingress
   ↓
vproapp Service
   ↓
vProfile Pod
```

---

## 11.1 Inspect Ingress Details

Use:

```bash
kubectl describe ingress <ingress-name>
```

Inspect:

- host
- paths
- backend Service
- backend port
- events

The important relationship is:

```text
Host / Path
     ↓
vproapp Service
     ↓
Application Pod
```

---

# 12. Ingress Controller Validation

The Ingress resource depends on an Ingress Controller.

The Controller should therefore be checked separately.

Conceptually:

```text
Ingress Resource
       │
       ▼
Ingress Controller
       │
       ▼
AWS Load Balancer
```

The validation process checks the Ingress Controller's namespace and verifies that the external load-balancer infrastructure is healthy.

The important distinction is:

```text
Ingress
    =
routing configuration

Ingress Controller
    =
component that implements the routing
```

---

# 13. AWS Load Balancer Validation

After the Ingress Controller is operational, AWS should contain the corresponding external load-balancing infrastructure.

The validation chain is:

```text
Ingress Controller
        ↓
AWS Load Balancer
        ↓
Healthy target
        ↓
Ingress
        ↓
vproapp Service
```

A load balancer existing is useful evidence, but it does not alone prove that the application is functioning.

The downstream Kubernetes Service and application Pod must also be healthy.

---

# 14. DNS Validation

The application hostname must resolve to the external entry point.

The expected flow is:

```text
Application hostname
        ↓
DNS
        ↓
AWS Load Balancer
        ↓
Ingress Controller
```

DNS validation should therefore confirm that the configured hostname points toward the external application entry point.

---

# 15. External Application Validation

Once the Kubernetes resources, Ingress, load balancer, and DNS are ready, access the application through its configured hostname.

The expected request path is:

```text
Browser
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

Successful external access establishes that the external networking path is functioning.

---

# 16. Application-Level Validation

Application access provides a higher-level validation layer.

A successful application login or equivalent successful application interaction can indicate that the request has passed through multiple layers.

Conceptually:

```text
External Request
      ↓
DNS
      ↓
Load Balancer
      ↓
Ingress
      ↓
Application Service
      ↓
Tomcat
      ↓
Backend Services
      ↓
Database
```

This is significantly stronger evidence than simply showing:

```bash
kubectl get pods
```

because it tests the system as an integrated application.

---

# 17. Database Validation

The application depends on MySQL.

The database validation chain is:

```text
vProfile Application
       ↓
vproDB Service
       ↓
MySQL Pod
       ↓
PVC
       ↓
Persistent Storage
```

The following should therefore all be healthy:

```text
MySQL Pod        → Running
vproDB Service   → Exists
Service Endpoint → Present
PVC              → Bound
```

Application-level success provides additional evidence that the application can communicate with the database.

---

# 18. Memcached Validation

The cache validation chain is:

```text
vProfile Application
       ↓
vprocache Service
       ↓
Memcached Pod
```

The infrastructure-level checks are:

```text
Memcached Pod        → Running
vprocache Service    → Exists
Service Endpoint     → Present
```

Where application behavior depends on the cache, successful application functionality provides additional integrated evidence.

---

# 19. RabbitMQ Validation

The message-queue validation chain is:

```text
vProfile Application
       ↓
vpromq Service
       ↓
RabbitMQ Pod
```

The infrastructure-level checks are:

```text
RabbitMQ Pod        → Running
vpromq Service      → Exists
Service Endpoint    → Present
```

The RabbitMQ Secret configuration should also be present where required by the workload.

---

# 20. End-to-End Validation

The strongest validation is the complete application path.

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
                          │
                          ▼
                 vproapp SERVICE
                          │
                          ▼
                 VPROFILE APP POD
                    /     |                         /      |                         ▼       ▼        ▼
             vproDB  vprocache   vpromq
             Service Service    Service
                │       │          │
                ▼       ▼          ▼
             MySQL   Memcached   RabbitMQ
                │
                ▼
               PVC
                │
                ▼
              AWS EBS
```

A successful application interaction provides evidence that multiple layers are working together.

---

# 21. Validation Matrix

| Layer | Check | Expected State | What It Proves |
|---|---|---|---|
| Cluster | `kubectl get nodes` | Nodes `Ready` | Cluster is operational |
| Secret | `kubectl get secret` | Secret exists | Required Secret resource exists |
| Storage | `kubectl get pvc` | `Bound` | Storage request is satisfied |
| Deployments | `kubectl get deployments` | Desired/available replicas | Workloads are being managed |
| Pods | `kubectl get pods` | `Running` | Containers are running |
| Services | `kubectl get svc` | Expected ClusterIP Services | Internal endpoints exist |
| Endpoints | `kubectl describe svc` | Endpoint populated | Service selects a Pod |
| Ingress | `kubectl get ingress` | Rule exists | External routing is configured |
| Controller | Controller status | Healthy | Ingress implementation is operating |
| Load Balancer | AWS state | Healthy | External entry point exists |
| DNS | Hostname resolution | Resolves correctly | Hostname reaches external entry point |
| Application | Browser/application | Successful access | External request reaches application |
| Backend | Application behavior | Successful dependency use | Application can use backend services |

---

# 22. Evidence Strategy

The repository should contain only high-signal evidence.

Recommended evidence files:

```text
evidence/screenshots/
├── cluster-state.png
├── workload-state.png
├── service-endpoints.png
├── ingress-storage-state.png
└── application-validation.png
```

These should contain **your own execution evidence**, not screenshots copied from the course material.

---

## 22.1 Cluster State Evidence

Suggested evidence:

```bash
kubectl get nodes
```

### Demonstrates

- Kubernetes cluster availability
- Node readiness

---

## 22.2 Workload State Evidence

Suggested evidence:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

### Demonstrates

- workloads created
- Pods running
- Services created

---

## 22.3 Service Endpoint Evidence

Suggested evidence:

```bash
kubectl describe svc vproapp
```

### Demonstrates

- Service selector
- populated endpoint
- Service-to-Pod relationship

This is a particularly valuable technical evidence item because it demonstrates understanding beyond simply showing that a Service exists.

---

## 22.4 Ingress / Storage Evidence

Suggested evidence:

```bash
kubectl get pvc
kubectl get ingress
```

Potentially also include relevant detailed output.

### Demonstrates

- persistent storage is provisioned
- Ingress exists
- external routing configuration exists

---

## 22.5 Application Validation Evidence

A screenshot of successful application access/login can demonstrate the final integrated state.

It should not expose:

- passwords
- tokens
- private infrastructure information
- unnecessary personal information

---

# 23. What Each Validation Level Does Not Prove

Validation should not overclaim.

### `kubectl get pods`

Proves:

> Pods are reporting a running state.

Does **not** prove:

> The complete application works.

---

### `kubectl get svc`

Proves:

> Service objects exist.

Does **not** prove:

> Services have healthy endpoints.

---

### Service endpoint

Proves:

> The Service selector has identified a Pod endpoint.

Does **not** prove:

> The application dependency is functioning correctly.

---

### PVC = Bound

Proves:

> Kubernetes has associated the claim with persistent storage.

Does **not** prove:

> The application is successfully reading/writing the expected data.

---

### Ingress exists

Proves:

> Kubernetes contains an Ingress routing configuration.

Does **not** prove:

> External users can successfully reach the application.

---

### Browser access

Provides stronger evidence that:

> External routing and application access are functioning.

It still does not independently prove every backend component's internal state.

---

# 24. Troubleshooting Through Validation

Validation and troubleshooting are closely connected.

The basic troubleshooting model is:

```text
Failure
  ↓
Identify layer
  ↓
Inspect resource
  ↓
Check dependency
  ↓
Correct configuration
  ↓
Recreate/reapply
  ↓
Validate again
```

For example:

```text
Application unavailable
        ↓
Check Ingress
        ↓
Check vproapp Service
        ↓
Check Service endpoint
        ↓
Check vProfile Pod
        ↓
Check application configuration
        ↓
Check backend Services
```

This avoids changing unrelated resources without evidence.

---

# 25. Validation as an Architecture Test

The validation process is also a practical test of whether the architecture was implemented correctly.

For example:

```text
Service endpoint exists
        ↓
Selector ↔ Pod labels are correct
```

```text
PVC is Bound
        ↓
Storage request ↔ backing volume relationship works
```

```text
Ingress routes to Service
        ↓
Ingress configuration ↔ Service relationship works
```

```text
Application successfully uses MySQL
        ↓
Application ↔ Service ↔ Database relationship works
```

The validation therefore reinforces the architectural model rather than being a separate activity.

---

# 26. Final Validation Checklist

Before considering the deployment successfully validated:

### Cluster

- [ ] Kubernetes cluster is reachable
- [ ] Nodes are `Ready`

### Configuration

- [ ] Required Secret exists
- [ ] Real credentials are not exposed in evidence

### Storage

- [ ] MySQL PVC exists
- [ ] PVC is `Bound`
- [ ] Storage is associated with the database workload

### Workloads

- [ ] vProfile Deployment exists
- [ ] MySQL Deployment exists
- [ ] Memcached Deployment exists
- [ ] RabbitMQ Deployment exists
- [ ] Expected Pods are `Running`

### Services

- [ ] vproapp Service exists
- [ ] vproDB Service exists
- [ ] vprocache Service exists
- [ ] vpromq Service exists
- [ ] Service selectors match workload labels
- [ ] Expected Service endpoints are populated

### Ingress

- [ ] Ingress Controller is operational
- [ ] Ingress resource exists
- [ ] Ingress points to the application Service
- [ ] AWS load balancer is available

### DNS

- [ ] Application hostname resolves correctly
- [ ] Hostname reaches the external entry point

### Application

- [ ] Application is externally accessible
- [ ] Application interaction/login succeeds
- [ ] Application can communicate with MySQL
- [ ] Application can communicate with Memcached
- [ ] Application can communicate with RabbitMQ

### Evidence

- [ ] Cluster evidence captured
- [ ] Workload evidence captured
- [ ] Service endpoint evidence captured
- [ ] Ingress/storage evidence captured
- [ ] Application validation evidence captured
- [ ] No course screenshots presented as personal execution evidence

---

# 27. Validation Boundary

The validation performed by this project establishes that the demonstrated Kubernetes deployment can be:

```text
Created
   ↓
Scheduled
   ↓
Connected
   ↓
Exposed
   ↓
Accessed
   ↓
Validated
```

It does **not** establish comprehensive production readiness.

The validation does not demonstrate:

- long-term production reliability
- disaster recovery testing
- comprehensive security testing
- production observability
- load testing
- autoscaling under sustained production load
- zero-downtime deployment guarantees
- production SLA compliance

Those capabilities belong to future iterations.

---

## Related Documentation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Implementation](implementation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
