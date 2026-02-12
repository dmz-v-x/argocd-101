# Push-Based vs Pull-Based Deployments

When talking about modern Kubernetes deployments, the **biggest architectural difference** is *who initiates the deployment*.

That single decision changes security, scalability, and operational complexity.

---

## Push-Based Deployment Approach

In a push-based model, the **CI/CD system actively deploys** changes to the Kubernetes cluster.

### How Push-Based Deployment Works

1. Developer pushes code
2. CI/CD pipeline runs:
   - tests
   - builds image
   - pushes image to container registry
3. CI/CD pipeline applies changes to the cluster:
   - `kubectl apply`
   - `helm upgrade`
4. Cluster state changes immediately

### Required Access Model

For this to work:
- CI/CD system needs **read-write access** to the Kubernetes cluster
- Kubernetes credentials must be:
  - generated outside the cluster
  - stored in the CI/CD system

This means cluster credentials **leave the cluster**, which is risky and not recommended.

---

## Access Pattern in Push-Based Deployment

- CI/CD system:
  - read-only access to Git repositories
  - read-write access to container registry
  - read-write access to Kubernetes cluster

- Kubernetes cluster:
  - read-only access to container registry

---

## Advantages of Push-Based Deployment

Push-based systems became popular because they are **simple to start with**.

### 1. Easier Helm Chart Deployments
- Helm upgrades can be triggered directly from pipelines
- No additional controllers required

### 2. Image Version Injection
- Build pipeline can dynamically:
  - update image tags
  - inject versions at deploy time

### 3. Simpler Secret Handling (Initially)
- Secrets can be injected at deployment time
- CI tools often have built-in secret stores

---

## Disadvantages of Push-Based Deployment

As systems scale, the drawbacks become clear.

### 1. Cluster Configuration Lives in CI
- Cluster logic is embedded inside pipelines
- Pipelines become complex and hard to audit

### 2. CI Has Full Cluster Access
- Compromised CI = compromised cluster
- Large security blast radius

### 3. Tight Coupling to CD System
- Deployment logic depends on:
  - CI vendor
  - pipeline scripts
- Hard to migrate or standardize

---

## Pull-Based Deployment Approach

In a pull-based model, **the cluster deploys itself**.

A GitOps operator runs **inside the Kubernetes cluster** and continuously pulls changes.

---

## How Pull-Based Deployment Works

1. Developer pushes changes to Git
2. CI pipeline:
   - builds image
   - pushes image to registry
3. GitOps operator inside cluster:
   - watches Git repository *or* container registry
   - detects new versions
   - synchronizes cluster state

The CI/CD system **never talks to the cluster**.

---

## Access Pattern in Pull-Based Deployment

- CI/CD system:
  - read-write access to container registry
  - no access to Kubernetes cluster

- Kubernetes cluster:
  - read-only access to container registry
  - pull access to Git repositories

This keeps cluster credentials **inside the cluster boundary**.

---

## Advantages of Pull-Based Deployment

Pull-based deployment is designed for **security and scale**.

### 1. Strong Security Boundary
- No external user or system can modify the cluster
- Cluster credentials are never exposed

### 2. Image Scanning and Automation
- Operators can:
  - scan registries for new versions
  - automatically deploy approved images

### 3. Better Secret Management
- Secrets can be stored securely using tools like HashiCorp Vault
- Git stores only references, not raw secrets

### 4. Decoupled Deployment Model
- Deployment logic is independent of CI/CD pipelines
- CI focuses only on building and testing

### 5. Multi-Tenant Support
- GitOps operators support:
  - multiple teams
  - multiple namespaces
  - multiple clusters

---

## Disadvantages of Pull-Based Deployment

Pull-based systems trade simplicity for robustness.

### 1. Helm Secret Management Is Harder
- Passing sensitive values to Helm charts requires:
  - external secret managers
  - additional tooling

### 2. Generic Secret Management Is Complex
- Requires:
  - Vault
  - Sealed Secrets
  - External Secrets Operator
- Higher learning curve

---

## Side-by-Side Comparison

| Aspect | Push-Based | Pull-Based |
|-----|-----------|-----------|
| Deployment trigger | CI/CD pipeline | Cluster operator |
| Cluster access | CI has read-write | No external access |
| Security | Weaker | Stronger |
| Drift correction | No | Yes |
| CI complexity | High | Lower |
| Scalability | Limited | High |

---

## Final Mental Model

Push-based deployment says:
> “CI, go deploy this now.”

Pull-based deployment says:
> “Cluster, make yourself match Git.”

Once systems grow beyond a single team or cluster, **pull-based always wins**.
