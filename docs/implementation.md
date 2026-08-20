# Implementation

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/496487f4-6f06-42ab-9afd-e56f65205ba9" />


## 1. Implementation Overview

This project implements the Kubernetes deployment layer for an existing containerized vProfile workload.

The implementation follows the overall sequence:

```text
Prepare AWS / kOps environment
        ↓
Create or recreate Kubernetes cluster
        ↓
Prepare Kubernetes manifests
        ↓
Place manifests in Git repository
        ↓
Clone repository onto kOps instance
        ↓
Install / configure Ingress Controller
        ↓
Create Kubernetes resources
        ↓
Validate workloads and networking
        ↓
Configure external DNS
        ↓
Validate end-to-end application access
        ↓
Troubleshoot where required
        ↓
Clean up Kubernetes and AWS resources
```

The implementation deliberately separates **building container images** from **deploying those images**. The Kubernetes environment consumes images that already exist in Docker Hub; application source code is therefore not required on the kOps instance for this deployment stage.

---

# 2. Implementation Boundary

The implementation work represented by this repository covers:

- AWS Kubernetes cluster operation with kOps
- Kubernetes manifest implementation
- Kubernetes workload deployment
- Service configuration
- Secret configuration
- Persistent storage configuration
- Ingress configuration
- External DNS integration
- Deployment validation
- Troubleshooting
- Resource cleanup

The vProfile application itself is an existing workload used by the project.

The implementation should therefore be understood as:

```text
Existing Application
        │
        ▼
Existing Container Images
        │
        ▼
Kubernetes Deployment Engineering
        │
        ▼
AWS Kubernetes Runtime
```

The Kubernetes project does not rebuild the application source on the kOps instance.

---

# 3. Prerequisites

The kOps-based deployment requires the supporting environment to be available before the cluster can be created.

The deployment requires:

```text
AWS Access Key
Route 53 Hosted Zone
DNS records / cluster DNS name
S3 bucket for kOps state
kOps EC2 instance
kubectl
kOps
```

The kOps instance is used as the management environment from which cluster operations and Kubernetes commands are performed.

---

# 4. kOps Cluster Setup

## 4.1 Access the kOps Instance

The implementation begins from the kOps management instance.

The operational workflow uses SSH to connect to the instance and then performs cluster operations from that environment.

Conceptually:

```text
Local Machine
     │
     │ SSH
     ▼
kOps EC2 Instance
     │
     ├── kOps
     ├── kubectl
     └── AWS access
```

---

## 4.2 Create the Cluster Configuration

The kOps workflow defines the desired Kubernetes cluster configuration.

The source workflow represents this operation as:

```bash
kops create cluster <cluster-name> <options>
```

This defines the desired cluster configuration before the actual AWS resources are provisioned.

---

## 4.3 Apply the Cluster Configuration

The configuration is then applied with:

```bash
kops update cluster <cluster-name> \
  --state=s3://<bucket> \
  --yes \
  --admin
```

The important flags are:

| Flag | Purpose |
|---|---|
| `--state=s3://<bucket>` | Specifies the kOps state store |
| `--yes` | Applies the configuration |
| `--admin` | Generates/configures administrative kubeconfig access |

This operation provisions the AWS resources required for the cluster and generates the Kubernetes configuration used by `kubectl`.

---

## 4.4 Cluster Validation

After cluster creation, the first operational check is:

```bash
kubectl get nodes
```

The expected state is that the cluster nodes become:

```text
Ready
```

If nodes initially report `NotReady`, cluster initialization may still be in progress and additional time may be required.

The implementation therefore follows:

```text
kops update
     ↓
Wait for cluster initialization
     ↓
kubectl get nodes
     ↓
Nodes Ready
```

---

# 5. Kubernetes Repository Preparation

The Kubernetes project requires the Kubernetes definition files to be available on the deployment machine.

The `kubedefs/` directory is the key deployment artifact containing the Kubernetes manifests.

The complete manifest set includes:

```text
kubedefs/
├── secret.yaml
├── PVC definition
├── database Deployment
├── database Service
├── Memcached Deployment
├── Memcached Service
├── RabbitMQ Deployment
├── RabbitMQ Service
├── Tomcat/application Deployment
├── Tomcat/application Service
└── ingress.yaml
```

The Kubernetes definition set contains the resources required for the application deployment.

---

# 6. Build vs Deploy Separation

A key implementation boundary is the distinction between the previous containerization work and this Kubernetes deployment.

## Previous build stage

```text
Application Source
       ↓
Docker Build
       ↓
Container Image
       ↓
Docker Hub
```

## Kubernetes deployment stage

```text
Kubernetes Manifest
       ↓
kubectl apply/create
       ↓
Kubernetes
       ↓
Pull Image
       ↓
Pod
```

The kOps instance therefore needs:

```text
kubedefs/
kubectl
kOps cluster access
container registry access
```

It does not need the application source code merely to deploy the existing images.

This is an important engineering separation:

> **Build artifacts are produced first; deployment specifications consume those artifacts later.**

---

# 7. Git Repository Workflow

The Kubernetes definitions are maintained in a Git repository and cloned onto the kOps instance.

Conceptually:

```text
Local project files
       ↓
GitHub repository
       ↓
git clone
       ↓
kOps instance
       ↓
kubedefs/
       ↓
kubectl
```

The workflow uses:

```bash
git clone <repository-HTTPS-URL>
```

followed by:

```bash
cd <repository>
ls kubedefs/
```

to verify that the Kubernetes definitions are available.

For a production engineering workflow, an SSH-based Git workflow or CI/CD would be a natural improvement.

---

# 8. Manifest Implementation Order

The Kubernetes resources are implemented according to their dependencies.

The overall dependency-aware sequence is:

```text
Secret
   ↓
PersistentVolumeClaim
   ↓
Database Deployment
   ↓
Database Service
   ↓
Memcached Deployment / Service
   ↓
RabbitMQ Deployment / Service
   ↓
Application Deployment
   ↓
Application Service
   ↓
Ingress
```

The exact individual manifest creation commands may use either:

```bash
kubectl create -f <manifest>
```

or:

```bash
kubectl apply -f <manifest>
```

depending on the execution stage.

The manifests are applied one by one following the architecture and dependency order.

---

# 9. Secret Implementation

The project uses a Kubernetes Secret for sensitive credentials.

The Secret contains values associated with:

```text
db-pass
rmq-pass
```

These values are consumed by the relevant workloads rather than being treated as ordinary application configuration.

Conceptually:

```text
Secret
 ├── Database password
 │       ↓
 │    MySQL Pod
 │
 └── RabbitMQ password
         ↓
      RabbitMQ Pod
```

The Secret is therefore created before the workloads that depend on it.

### Public repository requirement

The repository version must contain sanitized values or placeholders.

Real passwords must never be committed.

---

# 10. PersistentVolumeClaim Implementation

The database requires persistent storage.

The project defines a PersistentVolumeClaim for MySQL storage.

The implementation relationship is:

```text
PVC
 │
 ▼
StorageClass
 │
 ▼
AWS EBS
```

The database Deployment then mounts the claimed storage into the MySQL container.

Conceptually:

```text
MySQL Pod
    │
    ▼
Volume Mount
    │
    ▼
PVC
    │
    ▼
AWS EBS
```

The database data path is associated with the persistent volume rather than only the Pod filesystem.

---

# 11. Database Deployment Implementation

The MySQL Deployment defines the database workload.

The implementation includes the relevant:

- container image
- container port
- environment configuration
- Secret references
- volume mount
- Pod labels

The relationship is:

```text
MySQL Deployment
       │
       ▼
MySQL Pod
       │
       ├── Secret
       │
       └── PVC
```

---

# 12. Database Service Implementation

The MySQL Service provides the stable internal endpoint used by the application.

The implementation pattern is:

```text
vproDB Service
       │
       ├── ClusterIP
       ├── port: 3306
       └── selector → MySQL Pod
```

The Service uses a selector matching the database Pod's label.

Conceptually:

```text
vproDB Service
     │
     │ selector
     ▼
app=vprodb
     │
     ▼
MySQL Pod
```

A wrong Service name, selector, or Pod label can break the connection between the application and database.

---

# 13. Memcached Implementation

Memcached is implemented as:

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

The Memcached workload uses the official Memcached image.

The Service provides the stable internal endpoint used by the application.

Memcached is treated as a stateless component in this deployment, so persistent storage is not required.

---

# 14. RabbitMQ Implementation

RabbitMQ is implemented as:

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

The RabbitMQ workload uses the official RabbitMQ image.

The RabbitMQ Service provides the stable internal endpoint used by the application.

The project also uses the Secret for the RabbitMQ password.

---

# 15. Application / Tomcat Implementation

The vProfile application runs inside a Tomcat Pod managed by a Deployment.

Conceptually:

```text
vproapp Deployment
        │
        ▼
Tomcat / vProfile Pod
        │
        ▼
vproapp Service
```

The application Deployment uses the existing container image produced during the previous containerization stage.

The Kubernetes project therefore focuses on deploying and configuring the workload rather than rebuilding its source code.

---

# 16. Application Service Implementation

The application Service provides the internal endpoint for traffic destined for the Tomcat application.

```text
vproapp Service
      │
      ├── ClusterIP
      ├── application port
      └── selector → vProfile Pod
```

This Service is the target used by the Ingress resource.

The resulting chain is:

```text
Ingress
   ↓
vproapp Service
   ↓
vProfile Pod
```

---

# 17. Service Discovery Configuration

The application's backend configuration uses Kubernetes Service names.

The implementation pattern is:

```text
vproDB
vprocache
vpromq
```

rather than direct Pod IP addresses.

For example:

```text
Application
    │
    │ db.host = vproDB
    │ db.port = 3306
    ▼
Kubernetes DNS
    │
    ▼
vproDB Service
    │
    ▼
MySQL Pod
```

The application configuration therefore depends on the Kubernetes Service abstraction rather than individual Pod addresses.

---

# 18. Ingress Controller Implementation

The Ingress Controller is deployed separately from the application's Kubernetes manifests.

Its role is to:

- process Kubernetes Ingress resources
- provide the external entry point
- create/manage the AWS load-balancing integration
- route external requests toward the appropriate Service

Conceptually:

```text
Ingress Controller
        │
        ▼
AWS Load Balancer
        │
        ▼
Ingress
        │
        ▼
vproapp Service
```

The project architecture distinguishes the Ingress resource from the Ingress Controller itself. The Controller provides the implementation that processes the routing configuration.

---

# 19. Ingress Resource Implementation

The Ingress resource defines the external routing rule.

Conceptually:

```text
Host / Path
     │
     ▼
Ingress Rule
     │
     ▼
vproapp Service
     │
     ▼
Tomcat Pod
```

The Ingress acts as the Kubernetes-native routing configuration while the Ingress Controller provides the runtime implementation.

---

# 20. External DNS Configuration

After the external load-balancing path is established, DNS is configured so that the application hostname resolves to the external entry point.

The resulting flow is:

```text
Application hostname
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
vproapp Service
```

---

# 21. Deployment Validation During Implementation

Validation is performed progressively after resources are created.

A basic implementation verification sequence is:

```bash
kubectl get nodes
kubectl get secret
kubectl get pvc
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get ingress
```

The expected validation states are:

```text
Secret        → exists
PVC           → Bound
Deployments   → desired/available replicas
Pods          → Running
Services      → ClusterIP with correct ports
Ingress       → correct host → service mapping
```

The Ingress Controller is additionally checked in its namespace, and the AWS load balancer is checked for healthy targets.

Detailed validation belongs in:

**[Validation](validation.md)**

---

# 22. Service Endpoint Validation

Service configuration is not considered complete merely because the Service object exists.

The Service must have endpoints corresponding to its selected Pods.

The diagnostic relationship is:

```text
Service
  │
  ├── selector
  │
  ▼
Pod labels
  │
  ▼
Endpoint
```

If the endpoint is missing, investigate:

```text
Service selector
       ↓
Pod labels
       ↓
Pod status
```

Service endpoint inspection is therefore an important troubleshooting boundary.

---

# 23. Troubleshooting Workflow

The implementation uses a layered troubleshooting approach.

Rather than changing multiple resources at once, identify the failing layer.

## Cluster problem

Check:

```bash
kubectl get nodes
```

Then inspect cluster state and wait for initialization if required.

---

## Deployment problem

Check:

```bash
kubectl get deployment
kubectl describe deployment <deployment>
```

The goal is to determine whether the Deployment has created the expected ReplicaSet and Pods.

---

## Pod problem

Check:

```bash
kubectl get pods
kubectl describe pod <pod>
```

The Pod status and events provide the next diagnostic boundary.

---

## Service problem

Check:

```bash
kubectl get svc
kubectl describe svc <service>
```

Then inspect the Service endpoints.

---

## Ingress problem

Check:

```bash
kubectl get ingress
```

Then verify:

```text
Ingress rule
    ↓
vproapp Service
    ↓
Application Pod
```

The external load-balancer state must also be checked when external access is involved.

---

# 24. Manifest Reapplication

One practical troubleshooting pattern is to remove a failed resource and recreate it when appropriate.

The workflow is:

```text
Manifest
   ↓
Apply
   ↓
Failure
   ↓
Inspect
   ↓
Correct
   ↓
Delete affected resource
   ↓
Recreate / Apply
   ↓
Validate
```

This reinforces the declarative nature of the Kubernetes configuration.

---

# 25. Declarative Deployment Model

The same manifests define both the desired deployment state and the resources that can later be removed.

Creation:

```bash
kubectl apply -f .
```

Destruction:

```bash
kubectl delete -f .
```

This provides a mirror relationship between applying and deleting declarative resources.

It also makes the Kubernetes deployment reproducible from the definition files.

---

# 26. Complete Implementation Sequence

The complete project can be reconstructed using this sequence:

```text
1. Prepare AWS prerequisites
        ↓
2. Access kOps EC2 instance
        ↓
3. Create/update Kubernetes cluster with kOps
        ↓
4. Verify nodes
        ↓
5. Prepare Git repository
        ↓
6. Upload Kubernetes definitions
        ↓
7. Clone repository onto kOps instance
        ↓
8. Verify kubedefs/
        ↓
9. Install/configure Ingress Controller
        ↓
10. Create Secret
        ↓
11. Create PVC
        ↓
12. Deploy MySQL
        ↓
13. Create MySQL Service
        ↓
14. Deploy Memcached
        ↓
15. Create Memcached Service
        ↓
16. Deploy RabbitMQ
        ↓
17. Create RabbitMQ Service
        ↓
18. Deploy vProfile/Tomcat
        ↓
19. Create application Service
        ↓
20. Create Ingress
        ↓
21. Configure/verify DNS
        ↓
22. Validate resources
        ↓
23. Validate Service endpoints
        ↓
24. Validate external application access
        ↓
25. Validate backend functionality
        ↓
26. Troubleshoot any failures
        ↓
27. Clean up resources
```

---

# 27. Cleanup Implementation

Cleanup is performed in **reverse dependency order**.

## Step 1 — Delete Ingress Controller

```bash
kubectl delete -f <ingress-controller-url>
```

The purpose is to remove the controller and its AWS load balancer before the cluster itself is destroyed.

Deleting the cluster first can leave externally managed resources behind.

---

## Step 2 — Delete Application Resources

From the Kubernetes definitions directory:

```bash
cd kubedefs
kubectl delete -f .
```

This removes the resources represented by the manifests.

The cascade includes:

```text
Deployments
   ↓
ReplicaSets
   ↓
Pods

PVC
   ↓
StorageClass reclaim policy
   ↓
AWS EBS volume
```

When the StorageClass reclaim policy is `Delete`, deleting the PVC can result in deletion of the dynamically provisioned EBS volume.

---

## Step 3 — Delete the kOps Cluster

```bash
kops delete cluster --name=<cluster-name> --yes
```

This removes the Kubernetes cluster and its associated AWS infrastructure managed by kOps.

---

## Step 4 — Delete the Route 53 Hosted Zone

After the cluster has been removed, the Route 53 hosted zone can be deleted if it is no longer required.

This prevents leaving an unnecessary DNS resource behind.

---

## Step 5 — Stop or Delete the kOps Instance

If the environment will be reused soon:

```text
Stop the kOps EC2 instance.
```

If the project is complete for the foreseeable future:

```text
Terminate the kOps EC2 instance.
```

Keeping the instance can be useful when it will be reused because it contains the previously installed operational prerequisites.

---

# 28. Implementation Lifecycle

The complete implementation lifecycle can therefore be compressed into:

```text
                    BUILD
                      │
                      ▼
              Container Images
                      │
                      ▼
                 Docker Hub
                      │
                      │
                 DEPLOY
                      │
                      ▼
              Kubernetes Manifests
                      │
                      ▼
                   kOps
                      │
                      ▼
              AWS Kubernetes
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   Services        Storage        Ingress
       │              │              │
       ▼              ▼              ▼
    Pods             EBS       Load Balancer
       │
       ▼
   Application
       │
       ▼
   Validation
       │
       ▼
    Cleanup
```

---

# 29. Implementation Boundaries

This implementation demonstrates:

- Kubernetes manifest-based deployment
- kOps cluster operation
- Kubernetes workload management
- internal Service networking
- Kubernetes service discovery
- persistent storage integration
- Secret-based configuration
- Ingress-based external access
- DNS integration
- deployment validation
- troubleshooting
- declarative cleanup

It does **not** demonstrate:

- Terraform-based infrastructure provisioning
- Helm packaging
- CI/CD implementation
- production observability
- comprehensive Kubernetes security hardening
- advanced production autoscaling
- production disaster recovery
- guaranteed zero-downtime deployment

These should be treated as future engineering work rather than implemented capabilities.

---

## Related Documentation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Validation](validation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
