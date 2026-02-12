# How Argo CD Works Internally

Argo CD often looks simple from the outside:  
you connect a Git repository, and your Kubernetes cluster magically stays in sync.

Internally, however, Argo CD follows a **very deliberate, continuous control loop** designed around GitOps principles.

This blog explains **how Argo CD works step by step**, focusing on:
- how it watches Git
- how it detects drift
- how it applies manifests
- why the entire system is pull-based

---

## High-Level Mental Model

At its core, Argo CD runs a **continuous reconciliation loop**:

- Git represents the **desired state**
- Kubernetes represents the **actual state**
- Argo CD continuously compares the two
- If they differ, Argo CD fixes the cluster

This loop never stops.

---

## Core Internal Flow (Bird’s-Eye View)

1. Argo CD monitors Git repositories
2. Argo CD reads the desired state
3. Argo CD observes the live cluster state
4. Argo CD calculates differences (diff)
5. Argo CD applies corrective actions
6. Argo CD repeats forever

This is **continuous delivery**, not one-time deployment.

---

## How Argo CD Watches Git

Argo CD does not blindly poll Git for no reason.  
It uses **multiple mechanisms** to stay aware of changes.

### Git Repository Monitoring

Argo CD watches Git repositories in two ways:

- **Polling**
  - Periodically fetches the repository
  - Checks for new commits
- **Webhooks (optional)**
  - Git provider notifies Argo CD immediately
  - Faster reaction time

Regardless of method:
- Git is always treated as the source of truth
- No change in Git = no change in the cluster

---

### What Happens When Git Changes?

When a commit is detected:

1. Repository Server fetches the latest state
2. Manifests are rendered (YAML, Helm, Kustomize, etc.)
3. Desired state snapshot is created
4. Application Controller is notified

At this point, Argo CD **does not deploy immediately**.  
It first compares states.

---

## How Argo CD Detects Drift

Drift detection is the heart of Argo CD.

### What Is Drift?

Drift occurs when:
- someone runs `kubectl apply`
- someone edits a live resource
- a resource is deleted manually
- runtime mutation changes state

Drift means:
> Live cluster state ≠ Git desired state

---

### Live State Observation

The Application Controller continuously:
- queries the Kubernetes API
- fetches live resources
- reads current configuration and status

This gives Argo CD a **real-time view** of the cluster.

---

### Desired vs Live State Comparison

For each Application, Argo CD:

1. Reads desired manifests from Git
2. Reads live resources from the cluster
3. Normalizes both representations
4. Computes a diff

If differences exist:
- Application is marked **OutOfSync**
- Differences are visible in UI and CLI

No guessing. No assumptions. Just comparison.

---

## How Argo CD Applies Manifests

Argo CD does not run shell scripts or pipelines.

It uses the **Kubernetes API directly**.

### Apply Strategy

When synchronization is needed:

- Argo CD creates or updates resources
- Uses declarative `apply` semantics
- Respects ownership and managed fields
- Applies resources in dependency order

This ensures:
- safe updates
- minimal disruption
- predictable behavior

---

### Sync Phases

Argo CD applies resources in phases:

- PreSync hooks (optional)
- Sync phase (main deployment)
- PostSync hooks (optional)

These hooks enable:
- database migrations
- blue-green deployments
- canary strategies

All without leaving GitOps.

---

### Continuous Enforcement

After applying manifests:
- Argo CD does **not stop**
- It keeps watching Git and the cluster
- Any future drift triggers correction

Deployment is not a one-time action.  
It is **continuous enforcement**.

---

## Why Argo CD Is Pull-Based

The pull-based model is a **design choice**, not an implementation detail.

### What Pull-Based Means

- Argo CD runs inside the Kubernetes cluster
- It pulls desired state from Git
- No external system pushes changes

The cluster manages itself.

---

### Why Push-Based Is Risky

In push-based systems:
- CI pipelines need cluster credentials
- Credentials live outside the cluster
- Compromise of CI = compromise of cluster
- Hard to scale to multiple clusters

Push-based deployment creates a large security blast radius.

---

### Benefits of Pull-Based Design

Pull-based Argo CD provides:

- Strong security boundaries
- No external access to cluster APIs
- Continuous reconciliation
- Natural multi-cluster scalability
- Self-healing behavior

Each cluster independently:
- watches Git
- applies changes
- enforces state

No central coordinator required.

---

## End-to-End Internal Flow (Step-by-Step)

1. Developer commits changes to Git
2. Git repository updates desired state
3. Argo CD Repository Server fetches updates
4. Application Controller reads desired state
5. Controller reads live cluster state
6. Controller computes diff
7. If OutOfSync:
   - apply manifests
8. Controller continues monitoring

This loop runs **forever**.

---

## What Makes Argo CD Different

Traditional CD tools:
- deploy once
- exit
- forget

Argo CD:
- deploys
- watches
- enforces
- corrects
- repeats

This is why Argo CD is not “just a deployment tool”.

---

## Final Mental Model

Think of Argo CD as a **Kubernetes control system**:

- Git defines how the system should look
- Kubernetes exposes how it actually looks
- Argo CD continuously closes the gap

**Git is the desired state.  
The cluster is reality.  
Argo CD keeps reality honest.**
