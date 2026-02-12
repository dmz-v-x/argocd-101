# What is GitOps?

GitOps is a modern way of deploying and managing applications where **Git is the single source of truth for everything** — application code, infrastructure configuration, and deployment state.

Instead of humans or CI pipelines directly deploying to servers or Kubernetes clusters, **Git becomes the control plane**.  
If something exists in Git, it *should* exist in the system.  
If something changes in Git, the system *automatically converges* to that change.

At its core, GitOps is about **trusting Git more than people, scripts, or manual commands**.

---

## Traditional Deployment Model

Before GitOps, deployments usually looked like this:

1. Developer writes code
2. CI pipeline builds the application
3. CI pipeline deploys directly to the server or cluster
4. Infrastructure and configs are often:
   - manually edited
   - partially stored in Git
   - partially applied using scripts
5. Over time, production drifts away from what’s written in Git

### Problems with Traditional Deployments

- No clear source of truth
- Manual `kubectl apply` or SSH into servers
- Configuration drift
- Rollbacks are painful
- Hard to audit:  
  “Who changed what, when, and why?”

In short: **the real state lives in the cluster, not in Git**.

---

## GitOps Deployment Model

GitOps flips the model completely.

1. Developer updates configuration in Git
2. Git now represents the **desired state**
3. A GitOps controller continuously watches Git
4. The controller compares:
   - Desired state (Git)
   - Live state (cluster)
5. If there is a difference, the controller fixes it automatically

### Key Shift

In GitOps:
- Humans **never deploy directly**
- CI **does not touch production**
- The system continuously self-heals

The cluster is no longer the source of truth — **Git is**.

---

## Why Git Becomes the Single Source of Truth

Git is perfect for this role because it provides:

### 1. Version History
Every change is recorded:
- who made it
- when it was made
- why it was made (commit message)

### 2. Auditability
You can answer questions like:
- What changed in production last night?
- Why did this deployment happen?
- What was the previous working version?

### 3. Rollbacks Are Simple
Rollback = revert a Git commit  
No scripts. No manual fixes. No guessing.

### 4. Declarative Model
You declare *what you want*, not *how to do it*.

Git says:
> “This is how the system should look.”

GitOps tools say:
> “I’ll make reality match Git.”

---

## Why CI ≠ CD in GitOps

This is one of the most important mental shifts.

### Traditional Thinking
- CI builds
- CI deploys

### GitOps Thinking
- CI builds
- CD **pulls**

In GitOps:
- **CI ends at Git**
- **CD starts from Git**

CI is responsible for:
- running tests
- building images
- pushing artifacts
- updating configuration in Git

CI is **not responsible for deploying**.

---

## Role of Continuous Delivery in GitOps

Continuous Delivery is handled by a **GitOps controller**, not the CI pipeline.

The controller:
- watches Git continuously
- compares desired vs live state
- applies changes automatically
- keeps reconciling forever

This separation gives:
- stronger security
- fewer permissions for CI
- more predictable deployments

CI prepares changes.  
CD enforces changes.

---

## Push-Based Deployment

### How Push-Based Works

1. CI pipeline finishes building
2. CI pushes changes directly to:
   - servers
   - Kubernetes API
3. Deployment happens immediately

### Problems with Push-Based Model

- CI needs cluster credentials
- Harder to control access
- No continuous reconciliation
- Drift can occur silently
- One-time deployment, not ongoing enforcement

Push-based systems **act once and forget**.

---

## Pull-Based Deployment (GitOps Model)

### How Pull-Based Works

1. Git is updated
2. GitOps controller pulls from Git
3. Controller applies changes
4. Controller keeps checking forever

The controller lives **inside or next to the cluster**, so:
- no external credentials are needed
- security boundaries are stronger
- the system self-heals automatically

### Key Advantage

If someone manually changes the cluster:
- Git doesn’t change
- Controller detects drift
- Controller fixes it

Reality is forced to match Git.

---

## Final Mental Model

Think of GitOps like this:

- Git = desired state
- Cluster = live state
- GitOps controller = referee

If Git and cluster disagree, **Git always wins**.
