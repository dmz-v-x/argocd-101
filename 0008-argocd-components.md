# Core Components of Argo CD

Argo CD looks simple on the surface, but internally it is a **well-structured, decoupled system** designed to continuously reconcile Git with Kubernetes.

To truly understand Argo CD (and to debug it confidently in production), you must understand:
- what each core component does
- who talks to whom
- why Argo CD is designed this way

---

## High-Level View of Argo CD

Argo CD runs **inside a Kubernetes cluster** and is composed of multiple cooperating components rather than a single binary.

At a high level, Argo CD consists of:

- API Server  
- Repository Server  
- Application Controller  
- Redis  
- UI and CLI  

Each component has a **clear, limited responsibility**, which is key to scalability and reliability.

---

## 1. Argo CD API Server

The **API Server** is the main entry point into Argo CD.

### What the API Server Does

- Exposes **gRPC and REST APIs**
- Handles **authentication and authorization**
- Acts as the gateway for:
  - Web UI
  - CLI
  - CI/CD systems
  - Automation tools

Anything that wants to *interact* with Argo CD talks to the API Server first.

---

### Who Talks to the API Server?

- Humans via:
  - Web UI
  - Argo CD CLI
- Automation systems via:
  - REST API
  - gRPC API

The API Server does **not** deploy applications itself.  
It only **coordinates and validates requests**.

---

### Why This Separation Matters

- Centralized auth and RBAC
- Clean security boundary
- Easy integration with SSO and external systems
- No direct cluster manipulation by users

Think of the API Server as the **control plane interface** of Argo CD.

---

## 2. Repository Server

The **Repository Server** is responsible for interacting with Git.

### What the Repository Server Does

- Clones Git repositories
- Maintains a **local cache** of repositories
- Generates Kubernetes manifests from:
  - plain YAML
  - Helm charts
  - Kustomize
  - Jsonnet
- Always keeps repositories up to date

It does **not** apply anything to the cluster.

---

### Why a Separate Repository Server?

Without this component:
- every reconciliation would re-clone Git
- performance would suffer
- Git servers would be overloaded

By caching repositories:
- Argo CD becomes fast
- Git becomes scalable
- Reconciliation becomes efficient

---

### Role of Redis Here

The Repository Server uses **Redis** to:
- cache repository data
- speed up manifest generation
- reduce repeated Git operations

Redis acts as a **shared performance layer**, not a source of truth.

---

## 3. Application Controller

The **Application Controller** is the heart of Argo CD.

This is the component that actually **enforces GitOps**.

---

### What the Application Controller Does

For every Argo CD Application, it:

1. Reads the **desired state** from Git (via Repository Server)
2. Reads the **live state** from the Kubernetes cluster
3. Compares both states
4. Detects differences (OutOfSync)
5. Takes corrective action
6. Repeats continuously

This loop never stops.

---

### Reconciliation Loop

The controller follows a strict loop:

- Observe  
- Diff  
- Act  

If Git and cluster do not match:
- Argo CD fixes the cluster
- notifies via UI / CLI
- records application status

This is what enables:
- self-healing
- drift correction
- safe automation

---

### Why This Component Is Critical

- Detects configuration drift
- Enforces desired state
- Enables rollback via Git
- Provides application health status

Without the Application Controller, Argo CD would just be a dashboard.

---

## 4. Redis

Redis is a **supporting component**, but an important one.

### What Redis Is Used For

- Caching repository data
- Caching application state
- Reducing repeated expensive operations

Redis **does not store desired state** and **does not store cluster state**.

Git remains the source of truth.

---

### Why Redis Is Needed

- Improves performance
- Reduces Git load
- Enables fast UI updates
- Helps Argo CD scale to many applications and clusters

Redis is about **speed**, not correctness.

---

## 5. Argo CD UI

The **Web UI** is the visibility layer of Argo CD.

### What the UI Provides

- Real-time sync status
- Visual diff between Git and cluster
- Health status of resources
- Application history
- Manual sync and rollback controls

The UI never talks directly to Kubernetes.

It talks **only to the API Server**.

---

### Why the UI Matters

Without the UI:
- GitOps becomes opaque
- debugging becomes hard
- teams lose confidence

The UI turns GitOps from a black box into a **transparent system**.

---

## 6. Argo CD CLI

The **CLI** is designed for automation and power users.

### What the CLI Is Used For

- Logging into Argo CD
- Managing applications
- Triggering syncs
- Querying application status
- Scripting and CI integration

Just like the UI:
- CLI → API Server → Controller

No direct cluster access.

---

## How All Components Work Together

A simplified interaction flow looks like this:

1. User or CI interacts with UI or CLI  
2. UI / CLI calls the API Server  
3. API Server validates and authorizes request  
4. Application Controller performs reconciliation  
5. Repository Server supplies Git manifests  
6. Redis accelerates caching and state lookups  
7. Kubernetes cluster is updated if needed  

Each component has **one job**.

---

## Why This Architecture Works Well

- Strong separation of concerns
- Secure by default
- Scales horizontally
- Cloud-native design
- Continuous enforcement instead of one-time execution

This design is the reason Argo CD works reliably in:
- single-cluster setups
- large multi-cluster environments
- regulated production systems

---

## Final Mental Model

- API Server: gatekeeper and coordinator  
- Repository Server: Git and manifest engine  
- Application Controller: reconciliation brain  
- Redis: performance accelerator  
- UI / CLI: visibility and control  

**Git defines the desired state.  
The Application Controller enforces it.  
Everything else exists to support that loop.**

Once this architecture clicks, Argo CD stops feeling complex — it feels inevitable.
