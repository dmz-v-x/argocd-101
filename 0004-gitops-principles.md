# GitOps and Its Principles

GitOps is not just a tool or a pipeline style — it is a **set of principles** that redefine how infrastructure and applications are deployed and managed, especially in Kubernetes environments.

This blog explains:
- what GitOps is
- why GitOps uses two repositories
- how a GitOps CI pipeline looks with Jenkins and Argo CD
- the core principles that make GitOps work

---

## 1. What Is GitOps?

GitOps is an **operational model** where Git is the **single source of truth** for the desired state of a system.

Everything the system needs to run — applications, configurations, infrastructure definitions — is declared in Git.  
A GitOps operator continuously ensures that the real system matches what is defined in Git.

Key idea:
Git does not *describe history*, it *defines reality*.

---

## Why GitOps Uses Two Repositories

In GitOps, responsibilities are intentionally split to reduce risk and increase clarity.

### 1. Application Code Repository

This repository contains:
- application source code
- tests
- Dockerfile
- CI configuration

Purpose:
- build and test the application
- produce container images

This repository **never talks to Kubernetes directly**.

---

### 2. Kubernetes Manifest Repository

This repository contains:
- Kubernetes manifests
- Helm charts or Kustomize configs
- environment-specific deployment configuration

Purpose:
- define *how* the application runs
- define *what version* runs in the cluster

This repository represents the **desired state of the cluster**.

---

## Why This Separation Is Important

- Developers can change application code without touching infrastructure
- Infrastructure changes go through review and approval
- CI pipelines do not need cluster credentials
- Rollbacks are simple Git reverts
- Clear audit trail for production changes

This separation is one of the strongest safety guarantees of GitOps.

---

## Jenkins Pipeline in a GitOps Model (Using Argo CD)

In GitOps, Jenkins does **not deploy to Kubernetes**.

Jenkins has two responsibilities:
1. build artifacts
2. update Git

---

### Application CI Pipeline (Jenkins)

This pipeline runs when application code changes.

    stage('Test') {
        run unit tests
    }

    stage('Build Image') {
        docker build image
    }

    stage('Push Image') {
        docker push image to registry
    }

    stage('Update Manifests') {
        clone kubernetes-manifest-repo
        update image tag in manifests
        commit changes
        push to feature branch
    }

    stage('Create Pull Request') {
        open PR to manifest repository
    }

Important:
- Jenkins has **no Kubernetes access**
- Jenkins only pushes to Git and container registry

---

### Infrastructure / Manifest Pipeline

This pipeline handles:
- reviewing pull requests
- validating configuration
- merging changes

Once the PR is merged:
- Git becomes updated
- desired state changes

At this point, **no pipeline triggers deployment**.

---

## How Deployment Actually Happens

Deployment is handled by the **GitOps operator (Argo CD)** running inside the cluster.

- Argo CD watches the manifest repository
- detects new commits
- pulls changes
- applies them to the cluster
- keeps reconciling continuously

The cluster deploys itself.

---

## 2. GitOps Principles

GitOps works because it strictly follows a few non-negotiable principles.

---

## Principle 1: Declarative State

The system is described declaratively.

Instead of saying:
“Run these commands in this order”

You say:
“This is how the system should look”

Examples:
- desired number of replicas
- container image version
- resource limits
- service configuration

The *how* is handled by the system, not humans.

---

## Principle 2: Make Use of Git

Git is the **single source of truth**.

Git provides:
- version history
- auditability
- rollback capability
- peer review via pull requests

If something is not in Git:
- it should not exist in the cluster

If something exists in Git:
- the cluster must eventually reflect it

---

## Principle 3: Agents Pull Changes

GitOps uses **pull-based deployment**.

- GitOps operators (agents) run inside the cluster
- They pull changes from Git
- No external system pushes to Kubernetes

Benefits:
- cluster credentials stay inside the cluster
- reduced attack surface
- stronger security boundaries

The cluster trusts itself, not external pipelines.

---

## Principle 4: Reconciliation

Reconciliation is the heart of GitOps.

It follows a continuous loop:

    Observe → Diff → Act

### Observe
- Read desired state from Git
- Read live state from the cluster

### Diff
- Compare desired vs live state
- Detect drift or missing resources

### Act
- Apply changes to fix the difference
- Restore the system to desired state

This loop runs **continuously**, not once.

---

## Why Reconciliation Matters

- Manual changes are automatically reverted
- System self-heals
- Configuration drift disappears
- Git always wins

The system is not just deployed — it is **enforced**.

---

## Final Summary

- GitOps treats Git as the source of truth
- Application code and infrastructure are separated
- CI pipelines build and update Git, not clusters
- GitOps operators pull and reconcile changes
- Reconciliation ensures continuous correctness

**One-line mental model:**

Git defines reality, and the cluster continuously conforms to it.
