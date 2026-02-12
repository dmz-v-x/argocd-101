# Argo CD Sync Phases and Sync Waves 

## High-Level Idea: How Argo CD Decides Order

When Argo CD syncs an application, it decides execution order using:

1. **Sync Phases** (coarse-grained ordering)
2. **Sync Waves** (fine-grained ordering inside a phase)

Argo CD evaluates **both together** to decide *what runs first, next, and last*.

---

## Sync Phases in Argo CD

Sync phases define **when** a resource is applied during a sync.

Argo CD supports the following phases:

1. **PreSync**
2. **Sync**
3. **PostSync**
4. **SyncFail**

By default:
- **all resources are in the Sync phase**

You explicitly move a resource to another phase using annotations.

---

## What Each Sync Phase Means

### PreSync Phase

Purpose:
- Runs **before** the main deployment

Typical use cases:
- validation jobs
- cleanup tasks
- backups
- pre-flight checks

If PreSync fails:
- the entire sync stops

---

### Sync Phase (Default)

Purpose:
- Deploy core application resources

Typical resources:
- Namespace
- ConfigMap
- Service
- Deployment
- StatefulSet

This is where **most resources live**.

---

### PostSync Phase

Purpose:
- Runs **after** successful Sync phase

Typical use cases:
- database migrations
- cache warmups
- notifications
- cleanup jobs

PostSync **runs only if Sync succeeds**.

---

### SyncFail Phase

Purpose:
- Runs **only if sync fails**

Typical use cases:
- alerting
- rollback helpers
- debugging hooks

This phase is optional and advanced.

---

## Assigning a Resource to a Sync Phase

You assign a resource to a phase using annotations.

Example:

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-phase: PreSync

If this annotation is missing:
- resource stays in **Sync** phase

---

## Why Sync Waves Exist

Sync phases alone are **not enough**.

Inside a phase:
- Argo CD may have many resources
- Some must still run before others

That’s where **Sync Waves** come in.

---

## Sync Waves Explained

Sync waves define **execution order inside a phase**.

Rules:
- Waves are integers: -10, -1, 0, 1, 10
- Lower wave runs **before** higher wave
- Default wave = **0**

Execution order inside a phase:

	wave -1 → wave 0 → wave 1 → wave 2

---

## Assigning a Sync Wave

Example:

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-wave: "1"

Notes:
- Value must be a **string**
- Can be negative or positive

---

## Combining Sync Phases + Sync Waves

Argo CD always evaluates:

1. Phase first
2. Wave second

Example:

	metadata:
	  annotations:
	    argocd.argoproj.io/sync-phase: Sync
	    argocd.argoproj.io/sync-wave: "1"

Meaning:
- Runs during Sync phase
- After wave 0 resources
- Before wave 2 resources

---

## Complete Real-World Example

Let’s build a **real deployment flow**.

### Requirements

1. Namespace must exist first
2. ConfigMap must be created before Deployment
3. Deployment should start after ConfigMap
4. Database migration Job must run after app is deployed

---

## Resource 1: Namespace (Sync Phase, Wave -1)

Namespace must be created **before anything else**.

	apiVersion: v1
	kind: Namespace
	metadata:
	  name: my-namespace
	  annotations:
	    argocd.argoproj.io/sync-phase: Sync
	    argocd.argoproj.io/sync-wave: "-1"

Why:
- Same phase as app
- Earlier wave
- Guaranteed to exist before other resources

---

## Resource 2: ConfigMap (Sync Phase, Wave 0)

ConfigMap is needed by the application.

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: app-config
	  namespace: my-namespace
	  annotations:
	    argocd.argoproj.io/sync-phase: Sync
	    argocd.argoproj.io/sync-wave: "0"
	data:
	  APP_ENV: production

Why:
- Default wave
- Applied after namespace
- Available before Deployment

---

## Resource 3: Deployment (Sync Phase, Wave 1)

Deployment depends on ConfigMap.

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: my-app
	  namespace: my-namespace
	  annotations:
	    argocd.argoproj.io/sync-phase: Sync
	    argocd.argoproj.io/sync-wave: "1"
	spec:
	  replicas: 2
	  selector:
	    matchLabels:
	      app: my-app
	  template:
	    metadata:
	      labels:
	        app: my-app
	    spec:
	      containers:
	      - name: app
	        image: nginx
	        envFrom:
	        - configMapRef:
	            name: app-config

Why:
- Runs after ConfigMap
- App starts with correct configuration

---

## Resource 4: Job (PostSync Phase, Wave 0)

Database migration should run **only after app is deployed**.

	apiVersion: batch/v1
	kind: Job
	metadata:
	  name: db-migrate
	  namespace: my-namespace
	  annotations:
	    argocd.argoproj.io/sync-phase: PostSync
	    argocd.argoproj.io/sync-wave: "0"
	spec:
	  template:
	    spec:
	      containers:
	      - name: migrate
	        image: busybox
	        command: ["sh", "-c", "echo Running migrations"]
	      restartPolicy: Never

Why:
- Runs after Sync phase completes
- Does not block core deployment
- Ideal for migrations

---

## Execution Order (Exactly What Argo CD Does)

During sync, Argo CD executes in this order:

1. Sync phase, wave -1  
   → Namespace

2. Sync phase, wave 0  
   → ConfigMap

3. Sync phase, wave 1  
   → Deployment

4. PostSync phase, wave 0  
   → Migration Job

This order is **guaranteed**.

---

## Why This Matters in Production

Without sync phases and waves:
- race conditions happen
- apps crash due to missing configs
- migrations run too early
- deployments become flaky

With sync phases + waves:
- deployments are deterministic
- failures are predictable
- GitOps becomes safe at scale

---

## Final Mental Model

Think of Argo CD sync like this:

- **Phase decides WHEN**
- **Wave decides ORDER**
- **Annotations control behavior**
- **Git defines the plan**
- **Argo CD executes the plan**

Once you understand sync phases and waves,  
you unlock **safe, complex, production-grade GitOps deployments**.
