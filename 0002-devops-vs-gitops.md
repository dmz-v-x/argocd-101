# DevOps vs GitOps

DevOps and GitOps are often used together, but they are **not the same thing**.  
Understanding the difference clears up a lot of confusion around CI/CD, Kubernetes, and tools like Argo CD.

---

## What is DevOps?

**DevOps is a culture and set of practices**, not a single tool or technology.

DevOps focuses on:
- collaboration between development and operations
- automation
- faster and safer software delivery
- reducing manual work and handoffs

DevOps does **not mandate**:
- a specific tool
- a specific deployment method
- a specific architecture

You can do DevOps with:
- virtual machines
- containers
- Kubernetes
- cloud or on-prem
- any CI/CD tool

> DevOps = *how teams work together*  
> GitOps = *how deployments are done*

---

## What is GitOps?

**GitOps is a deployment model**, not a culture.

GitOps is opinionated and strict:
- Git is the single source of truth
- The system is declared, not scripted
- Deployments are pull-based
- A controller continuously reconciles state

GitOps **assumes containerized workloads**, usually running on Kubernetes.

> GitOps is built on top of DevOps ideas, not a replacement for DevOps.

---

## DevOps vs GitOps at a Glance

| Aspect | DevOps | GitOps |
|------|-------|--------|
| Nature | Culture + practices | Deployment methodology |
| Technology | Any | Container + Kubernetes focused |
| Source of truth | Tools + scripts | Git only |
| Deployment | Push-based | Pull-based |
| Drift handling | Manual | Automatic reconciliation |

---

## Traditional DevOps CI/CD Pipeline

A typical DevOps pipeline looks like this:

1. Developer writes code
2. Developer pushes source code to repository
3. Continuous Integration (CI) starts:
   - run unit tests
   - build application artifact
   - build container image
   - push image to container registry
4. Continuous Deployment (CD):
   - pipeline runs `kubectl apply`
   - changes are pushed directly to the cluster

### Flow Diagram (Conceptual)

developer code  
→ push source code  
→ CI (test → build → image → registry)  
→ CD (`kubectl apply`)  
→ cluster

---

## Problems with Traditional DevOps Deployment

- CI pipeline needs access to the cluster
- Cluster credentials live outside the cluster
- Deployments happen once, not continuously
- Manual changes can cause configuration drift
- Rollbacks require scripts or manual fixes

In this model:
> **The cluster becomes the source of truth**

---

## GitOps Pipeline Model

GitOps changes **only the CD part** of the pipeline.

### Key Idea
CI still exists.  
CI still builds.  
But **CI never deploys**.

---

## GitOps Uses Two Git Repositories

### 1. Application Code Repository
Contains:
- source code
- tests
- Dockerfile
- CI pipeline

### 2. Manifest Configuration Repository
Contains:
- Kubernetes manifests
- Helm charts or Kustomize configs
- environment-specific configuration

---

## GitOps Pipeline Step-by-Step

1. Developer pushes application code
2. CI pipeline runs:
   - unit tests
   - build artifact
   - build container image
   - push image to registry
3. CI clones the **manifest repository**
4. CI updates Kubernetes manifests:
   - new image tag
   - config changes
5. CI pushes changes to a **feature branch**
6. CI opens a **pull request** to the manifest repo
7. Team member reviews the PR:
   - suggests changes (if needed)
   - approves and merges
8. GitOps operator inside the cluster:
   - pulls changes from Git
   - synchronizes cluster state

At this point, deployment happens automatically.

---

## Important Shift in Responsibility

### In DevOps
- CI pipeline pushes changes to the cluster

### In GitOps
- CI pipeline pushes changes to Git
- GitOps operator pulls changes from Git

This single change has massive consequences.

---

## Push vs Pull Deployment

### DevOps (Push-Based)
- External system pushes to cluster
- Credentials live outside the cluster
- One-time deployment
- Drift possible

### GitOps (Pull-Based)
- Cluster pulls desired state
- Credentials stay inside cluster
- Continuous reconciliation
- Drift is automatically fixed

---

## Final Summary

- DevOps is **technology-agnostic**
- GitOps is **Kubernetes and Git-centric**
- DevOps pipelines push to the cluster
- GitOps operators pull from Git
- GitOps adds safety, auditability, and self-healing

**One-line mental model:**

In DevOps, changes are *pushed* to the cluster.  
In GitOps, changes are *pulled* by the cluster.
