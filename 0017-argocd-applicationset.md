# Argo CD ApplicationSet 

## What Is an ApplicationSet?

An **ApplicationSet** is a **factory for Argo CD Applications**.

Instead of manually writing many `Application` YAMLs, you define:

- **A template** → how one application should look
- **A generator** → the data that changes (env, cluster, repo, PR, etc.)

Argo CD **combines both** and **automatically creates multiple Applications** for you.

---

## Simple Mental Model

Think of ApplicationSet like this:

	Template  +  Data  =  Many Applications

Where:
- Template = Application blueprint
- Data = environments, clusters, repos, PRs, etc.

---

## Why ApplicationSet Exists

Without ApplicationSet:

- You write 3 Application YAMLs for dev/staging/prod
- Or 10 YAMLs for 10 clusters
- Or 50 YAMLs for 50 microservices

This leads to:
- duplication
- drift
- human error
- unmaintainable GitOps repos

ApplicationSet solves this **once and for all**.

---

## What Does ApplicationSet Actually Do?

ApplicationSet:
- Watches a **generator**
- Detects changes automatically
- Creates / updates / deletes Applications
- Keeps Applications in sync with data

If data changes → Applications change automatically.

---

## ApplicationSet Generators (Very Important)

Generators are **data sources**.

These decide **how many applications** get created and **with what values**.

### Main Generator Types

| Generator        | Purpose |
|------------------|--------|
| List             | Hardcoded values (envs, regions) |
| Git              | Scan folders or files in Git |
| Cluster          | One app per cluster |
| Matrix           | Combine multiple generators |
| SCM Provider     | Read GitHub/GitLab repos |
| Pull Request     | One app per PR (preview envs) |

In this blog, we’ll focus on **List** and **Cluster** (most important).

---

## When Should You Use ApplicationSet?

Use ApplicationSet when:

- You deploy the **same app to multiple environments**
- You deploy the **same app to multiple clusters**
- You want **dynamic app generation**
- You want **zero duplication**
- You want **GitOps at scale**

If you have only one app + one cluster → you don’t need it yet.

---

# PART 1 — Same App to Multiple Environments (List Generator)

---

## Goal

Deploy the same app to:

- dev
- staging
- prod

Each environment has its own folder in Git.

---

## Git Repository Structure

	my-git-repo/
	└── apps/
	    ├── dev/
	    │   └── my-app/
	    ├── staging/
	    │   └── my-app/
	    └── prod/
	        └── my-app/

Each folder contains Kubernetes manifests.

---

## ApplicationSet Using List Generator

	apiVersion: argoproj.io/v1alpha1
	kind: ApplicationSet
	metadata:
	  name: demo-appset
	spec:
	  generators:
	    - list:
	        elements:
	          - env: dev
	          - env: staging
	          - env: prod
	  template:
	    metadata:
	      name: '{{env}}-my-app'
	    spec:
	      project: default
	      source:
	        repoURL: https://github.com/my-org/my-app
	        targetRevision: main
	        path: apps/{{env}}/my-app
	      destination:
	        server: https://kubernetes.default.svc
	        namespace: my-app
	      syncPolicy:
	        automated: {}

---

## What Happens Internally?

List generator outputs:

	env=dev
	env=staging
	env=prod

Template is rendered **3 times**.

Argo CD automatically creates:

- dev-my-app
- staging-my-app
- prod-my-app

Each application:
- uses a different Git path
- deploys independently
- syncs automatically

---

## Why This Is Powerful

- Add a new environment → add one line
- No YAML duplication
- No manual Application creation
- Git drives everything

---

# PART 2 — Same App to Multiple Kubernetes Clusters (Cluster Generator)

This is the **most common real-world use case**.

---

## Goal

Deploy the same app to:

- dev-cluster
- staging-cluster
- prod-cluster

Each cluster is separate.

---

## Prerequisites

- Argo CD installed in a management cluster
- kubectl access to all target clusters
- Git repo with app manifests

---

## Step 1: Register All Clusters with Argo CD

Login to Argo CD CLI:

	argocd login <ARGOCD-SERVER>

List kube contexts:

	kubectl config get-contexts

Add each cluster to Argo CD:

	argocd cluster add dev-cluster
	argocd cluster add staging-cluster
	argocd cluster add prod-cluster

Now Argo CD **knows about all clusters**.

---

## Step 2: Label Clusters (Important)

We don’t want to deploy to *every* cluster automatically.

Add a label while registering (or later):

	argocd cluster add dev-cluster --label deploy-group=true
	argocd cluster add staging-cluster --label deploy-group=true
	argocd cluster add prod-cluster --label deploy-group=true

Only labeled clusters will receive the app.

---

## Step 3: Git Repo Structure

	my-git-repo/
	└── apps/
	    └── my-app/
	        ├── deployment.yaml
	        └── service.yaml

Same app for all clusters.

---

## Step 4: ApplicationSet Using Cluster Generator

	apiVersion: argoproj.io/v1alpha1
	kind: ApplicationSet
	metadata:
	  name: my-app-multicluster
	spec:
	  generators:
	    - clusters:
	        selector:
	          matchLabels:
	            deploy-group: "true"
	  template:
	    metadata:
	      name: '{{name}}-my-app'
	    spec:
	      project: default
	      source:
	        repoURL: https://github.com/my-org/my-git-repo
	        targetRevision: main
	        path: apps/my-app
	      destination:
	        server: '{{server}}'
	        namespace: my-app
	      syncPolicy:
	        automated: {}

---

## Understanding the Template Variables

The cluster generator provides built-in values:

- {{name}} → cluster name
- {{server}} → cluster API server URL

So for each cluster:

	dev-cluster → server=https://x.x.x.x
	staging-cluster → server=https://y.y.y.y
	prod-cluster → server=https://z.z.z.z

---

## Step 5: Apply the ApplicationSet

Apply it to the **Argo CD management cluster**:

	kubectl apply -f my-app-multicluster.yaml

---

## What You’ll See in Argo CD UI

Argo CD automatically creates:

- dev-cluster-my-app
- staging-cluster-my-app
- prod-cluster-my-app

Each Application:
- deploys to its own cluster
- syncs independently
- self-heals automatically

---

## What Happens If a New Cluster Is Added?

If you later add:

	argocd cluster add qa-cluster --label deploy-group=true

Argo CD will:
- detect the new cluster
- automatically create qa-cluster-my-app
- deploy the app

**No YAML changes required.**

---

## Why ApplicationSet Is a Game-Changer

Without ApplicationSet:
- app creation is manual
- scaling is painful
- GitOps breaks at scale

With ApplicationSet:
- apps are generated dynamically
- Git is the single source of truth
- clusters and environments scale infinitely

---

## Final Mental Model

- **Application** = deploy one app
- **ApplicationSet** = generate many applications
- **Generator** = data source
- **Template** = blueprint
- **Git controls everything**

Once you understand ApplicationSet,  
**Argo CD stops being a deployment tool and becomes a deployment platform.**

This is real GitOps at scale.
