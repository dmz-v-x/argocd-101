# Sync Policies and Sync Options in Argo CD

## Mental Model: What Is Sync in Argo CD?

- **Desired state** → Git repository
- **Live state** → Kubernetes cluster
- **Sync** → the action of making live state match Git

Argo CD can:
- sync manually
- sync automatically
- prune deleted resources
- self-heal drift
- selectively sync resources
- control update strategy (patch vs replace)

---

## 1. Manual Sync (No Automation)

By default, **Argo CD does NOT auto-deploy**.

Let’s first create an application **without automated sync**.

---

### Application Without Automated Sync

	no-automated-sync.yaml

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: application-no-automated-sync
	spec:
	  destination:
	    namespace: no-automated-sync
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/directoryOfmanifests
	    repoURL: https://github.com/<username>/<repository>
	    targetRevision: main

---

### Deploy the Application

	kubectl create ns no-automated-sync
	kubectl apply -f no-automated-sync.yaml

---

### What You’ll Observe

- Application appears in Argo CD UI
- Sync status = **OutOfSync**
- Nothing is deployed yet
- You must click **Sync** manually

This is **manual GitOps**.

---

## 2. Automated Sync

Automated sync tells Argo CD:

> “Whenever Git changes, automatically sync the cluster.”

This removes the need for:
- CI pipelines accessing Kubernetes API
- manual clicking in UI

---

### Enable Automated Sync

Add this block:

	syncPolicy:
	  automated: {}

---

### Application With Automated Sync

	app-automated-sync.yaml

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: application-automated-sync
	spec:
	  destination:
	    namespace: automated-sync
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/directoryOfmanifests
	    repoURL: https://github.com/<username>/<repository>
	    targetRevision: main
	  syncPolicy:
	    automated: {}

---

### Apply It

	kubectl create ns automated-sync
	kubectl apply -f app-automated-sync.yaml

---

### Result

- Application auto-syncs
- Resources are deployed immediately
- No manual sync required

This is **true GitOps automation**.

---

## 3. Auto-Pruning (Handling Deleted Resources)

### The Problem

What happens if you delete a resource from Git?

Example:
- You delete a Service from the repo
- Cluster still has that Service
- App becomes **OutOfSync**

Automated sync alone **does not delete resources**.

---

### Manual Way (Not Ideal)

- Click **Sync**
- Enable **Prune**
- Synchronize

This is manual and error-prone.

---

### Enable Auto-Prune

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: prune-application
	spec:
	  destination:
	    namespace: prune-sync
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/directoryOfmanifests
	    repoURL: https://github.com/<username>/<repository>
	    targetRevision: main
	  syncPolicy:
	    automated:
	      prune: true

---

### What Auto-Prune Does

- Deletes resources removed from Git
- Keeps cluster clean
- Prevents orphaned resources

Now Git **fully controls lifecycle**.

---

## 4. Self-Healing (Fixing Manual Changes)

### The Problem

Someone changes the live cluster directly:

	kubectl scale deploy/nginx --replicas=10 -n automated-sync

Git says replicas = 1  
Cluster says replicas = 10

---

### What Argo CD Does

- Detects drift
- App becomes **OutOfSync**
- Git is still the source of truth

If auto-sync is enabled:
- Argo CD **restores replicas back to Git value**

This is **self-healing GitOps**.

---

### Important Rule

> **Never change production through kubectl.**  
> Always change Git.

---

## 5. Sync Options (Fine-Grained Control)

Sync options give **extra behavior controls** beyond policies.

They can be applied:
- at **resource level**
- at **application level**

---

## Prevent Pruning of Specific Resources

Example: Protect a ServiceAccount.

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-options: Prune=false

---

### Example Resource

	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  name: nginx
	  annotations:
	    argocd.argoproj.io/sync-options: Prune=false

Even if auto-prune is enabled:
- This resource will never be deleted

---

## Disable Kubernetes Validation

Some resources fail kubectl validation.

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-options: Validate=false

Used for:
- CRDs
- special controllers
- experimental resources

---

## Selective Sync (Apply Only OutOfSync Resources)

By default:
- Argo CD reapplies everything

Selective sync improves performance.

---

### Enable Selective Sync

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	spec:
	  syncPolicy:
	    syncOptions:
	    - ApplyOutOfSyncOnly=true

---

### Combined Example

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: automated-application
	spec:
	  destination:
	    namespace: automated-sync
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/directoryOfmanifests
	    repoURL: https://github.com/<username>/<repository>
	    targetRevision: main
	  syncPolicy:
	    automated:
	      prune: true
	    syncOptions:
	    - ApplyOutOfSyncOnly=true

---

## Replace Sync Option (Patch vs Recreate)

### The Problem

Some resources cannot be updated:
- Jobs
- immutable fields
- certain CRDs

Normal patching fails.

---

### Replace Sync Option

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-options: Replace=true

---

### Example Job

	apiVersion: batch/v1
	kind: Job
	metadata:
	  name: my-job
	  annotations:
	    argocd.argoproj.io/sync-options: Replace=true
	spec:
	  template:
	    spec:
	      containers:
	      - name: example
	        image: busybox
	        command: ["echo", "Hello"]
	      restartPolicy: Never

Argo CD will:
- delete old job
- create a new one

---

## Fail on Shared Resources

### The Problem

Two applications managing the same resource causes:
- race conditions
- unexpected overwrites

---

### Enable Protection

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-options: FailOnSharedResource=true

---

### Behavior

- Sync fails if resource is owned by another app
- Prevents silent conflicts
- Essential for multi-team environments

---

## CreateNamespace Automatically

By default:
- Namespace must exist

Enable auto-creation:

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	spec:
	  syncPolicy:
	    syncOptions:
	    - CreateNamespace=true

---

### Example

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: demo-app
	spec:
	  destination:
	    server: https://kubernetes.default.svc
	    namespace: my-namespace
	  project: default
	  source:
	    repoURL: https://github.com/your/repo.git
	    targetRevision: HEAD
	    path: my-app
	  syncPolicy:
	    syncOptions:
	    - CreateNamespace=true

---

## PruneLast (Safe Deletion Order)

### Default Behavior

1. Prune old resources
2. Apply new resources

This can break dependencies.

---

### Enable PruneLast

	syncOptions:
	- PruneLast=true

---

### Example

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	spec:
	  syncPolicy:
	    syncOptions:
	    - PruneLast=true

Now:
- New resources applied first
- Old resources deleted last

Safer for:
- config migrations
- renames
- dependency chains

---

## Final Mental Model

- **Automated Sync** → auto-deploy
- **Auto-Prune** → auto-delete
- **Self-Heal** → undo manual changes
- **Sync Options** → fine control
- **Git always wins**

Argo CD sync policies turn Kubernetes into a **self-correcting system**.

This is GitOps done right.
