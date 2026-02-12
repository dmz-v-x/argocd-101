# Introduction to Argo CD

Argo CD is one of the most important tools in the GitOps ecosystem.  
It is designed specifically to **bring GitOps principles into Kubernetes in a practical, production-ready way**.

This section explains:
- what Argo CD actually is
- how it works at a conceptual level
- how it differs from common tools like Jenkins, Flux, and Helm-only workflows

---

## What Exactly Is Argo CD?

Argo CD is a **declarative GitOps continuous delivery (CD) tool for Kubernetes**.

It does not replace CI systems.  
It does not build code.  
It does not push changes.

Instead, Argo CD’s only responsibility is to **continuously make Kubernetes match Git**.

---

## Declarative GitOps CD Tool

Argo CD follows a **declarative model**.

You do not tell Argo CD:
- how to deploy
- when to deploy
- which commands to run

You tell Argo CD:
- what the desired state looks like (in Git)

Argo CD figures out *how* to reach that state and keeps enforcing it.

---

## Kubernetes-Native by Design

Argo CD is built **for Kubernetes, not adapted to it**.

- Runs inside Kubernetes
- Uses Kubernetes APIs
- Manages Kubernetes resources as first-class citizens
- Uses Custom Resource Definitions (CRDs)

This makes Argo CD:
- secure
- scalable
- aligned with Kubernetes design philosophy

It feels like a natural extension of the cluster, not an external tool.

---

## Continuous Reconciliation (Not Just Deployment)

Traditional CD tools deploy **once per pipeline run**.

Argo CD works differently.

It:
- continuously watches Git
- continuously watches the cluster
- continuously reconciles differences

This means Argo CD:
- detects drift
- fixes drift
- self-heals systems
- enforces desired state forever

Deployment is not an event.  
Deployment is a **continuous process**.

---

## Core Idea of Argo CD

The central idea behind Argo CD is simple:

**Argo CD constantly compares Git vs Cluster.**

- Git represents the desired state
- Kubernetes represents the live state
- Any difference is treated as a problem

Argo CD does not ask:
“Should I deploy now?”

It asks:
“Does the cluster match Git right now?”

If the answer is no, Argo CD acts.

---

## Argo CD vs Other Tools

Argo CD often gets compared with existing tools, which leads to confusion.  
Most of the time, these tools solve **different problems**.

---

## Argo CD vs Jenkins

### Jenkins

- CI/CD automation tool
- Script-driven
- Pipeline-centric
- Typically push-based
- Can deploy to many platforms, not just Kubernetes

Jenkins answers:
“How do I automate steps?”

---

### Argo CD

- Continuous delivery tool
- Declarative and Git-driven
- Kubernetes-only
- Pull-based
- Focused on state reconciliation

Argo CD answers:
“How do I ensure Kubernetes always matches Git?”

---

### Key Difference

- Jenkins **executes pipelines**
- Argo CD **enforces state**

In GitOps:
- Jenkins builds and updates Git
- Argo CD deploys by reconciling Git

They are complementary, not competitors.

---

## Argo CD vs Flux

Flux is another GitOps tool, so the comparison is more subtle.

### Common Ground

Both:
- implement GitOps
- are pull-based
- run inside Kubernetes
- continuously reconcile state
- use Git as source of truth

---

### Argo CD Strengths

- Rich UI and visualization
- Strong multi-cluster support
- Powerful Application and ApplicationSet abstractions
- Easy onboarding and debugging

---

### Flux Strengths

- More modular and lightweight
- Deep Kubernetes-native integrations
- CLI-focused workflows
- Strong automation primitives

---

### Mental Model Difference

- Argo CD emphasizes **visibility and control**
- Flux emphasizes **automation and composability**

Both are valid GitOps implementations with different philosophies.

---

## Argo CD vs Helm-Only Workflows

Helm is often misunderstood as a deployment system.

### What Helm Is

- Package manager for Kubernetes
- Templating engine
- Release installer/upgrader

Helm does **not**:
- continuously reconcile state
- detect drift
- enforce desired state

Helm answers:
“How do I install this application?”

---

### What Argo CD Adds on Top of Helm

Argo CD can use Helm as a **configuration source**, but it adds:

- Git as source of truth
- Continuous reconciliation
- Drift detection
- Rollbacks via Git
- Visibility into live vs desired state

Helm installs.  
Argo CD **enforces**.

---

## When Helm Alone Is Not Enough

Helm-only workflows struggle with:
- configuration drift
- manual changes
- auditability
- multi-cluster consistency
- long-term enforcement

Argo CD solves these problems by wrapping Helm inside GitOps.

---

## Final Mental Model

- Jenkins builds artifacts
- Helm packages applications
- Git defines desired state
- Argo CD ensures Kubernetes matches Git

Argo CD is not just a deployment tool.  
It is a **continuous correctness engine for Kubernetes**.
