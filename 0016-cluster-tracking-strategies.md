# Cluster Tracking Strategies in Argo CD

## 1. What Is Cluster Tracking in Argo CD?

**Cluster Tracking** is the mechanism Argo CD uses to:
- identify a target Kubernetes cluster
- remember that association
- continuously sync applications to the correct cluster

When you define an Argo CD Application, you specify a **destination**.  
Argo CD then uses a **tracking strategy** to bind that application to a cluster.

Without correct tracking:
- apps may go OutOfSync
- apps may deploy to the wrong cluster
- GitOps guarantees break

---

## 2. Why Cluster Tracking Is Needed

Cluster tracking becomes important when you:

- deploy to **multiple clusters** (dev, staging, prod)
- **recreate clusters** (new node pools, new control plane)
- **rename or rotate clusters**
- want **predictable and secure deployments**

If Argo CD identifies clusters in an unstable way, your apps lose their anchor.

---

## 3. Where Cluster Tracking Appears in Argo CD

Cluster tracking shows up in **two places**:

1. Argo CD configuration (global behavior)
2. Application manifest (destination)

Argo CD combines both to decide:
- *which cluster this app belongs to*
- *how stable that reference is*

---

## 4. Types of Cluster Tracking Strategies

Argo CD supports **three cluster tracking strategies**:

1. `server`
2. `label`
3. `name`

Each has a different trade-off.

---

## 5. Strategy 1 — Server-Based Tracking

### How Server Tracking Works

Argo CD tracks the cluster using the **Kubernetes API server URL**.

Example:
	https://1.2.3.4:6443

If your Application says “deploy to this server URL”, Argo CD uses that exact endpoint.

---

### Application Example (Server Strategy)

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: server-based-app
	spec:
	  destination:
	    server: https://1.2.3.4:6443
	    namespace: my-namespace
	  project: default
	  source:
	    repoURL: https://github.com/your/repo.git
	    targetRevision: main
	    path: app

---

### When to Use Server Strategy

Use `server` when:
- cluster API endpoint is stable
- cluster is not frequently recreated
- simple setups or single cluster environments

---

### Pros and Cons

Pros:
- simple
- explicit
- easy to understand

Cons:
- breaks if cluster is recreated
- new cluster = new API endpoint
- apps may lose their target

---

## 6. Strategy 2 — Label-Based Tracking

### How Label Tracking Works

Instead of tracking a **specific API server**, Argo CD tracks clusters by **labels**.

Clusters are registered with labels, such as:
	env=dev
	env=staging
	env=prod

Applications then target clusters by label.

---

### Adding a Cluster with Labels

	argocd cluster add <context-name> \
	  --label env=staging

This tells Argo CD:
- “This cluster belongs to the staging environment”

---

### Application Example (Label Strategy)

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: label-based-app
	spec:
	  destination:
	    name: env=staging
	    namespace: my-namespace
	  project: default
	  source:
	    repoURL: https://github.com/your/repo.git
	    targetRevision: main
	    path: app

Argo CD will:
- find the cluster with label `env=staging`
- deploy the app there

---

### When to Use Label Strategy

Use `label` when:
- clusters are recreated often
- environments are more important than cluster identity
- you want apps to follow “roles” instead of endpoints

---

### Pros and Cons

Pros:
- flexible
- resilient to cluster recreation
- ideal for dynamic infrastructure

Cons:
- requires disciplined label management
- mislabeling can cause wrong deployments

---

## 7. Strategy 3 — Name-Based Tracking

### How Name Tracking Works

Argo CD tracks clusters by their **internal Argo CD name**.

Each cluster added to Argo CD has a name:
- dev-cluster
- staging-cluster
- prod-cluster

Applications reference that name.

---

### Listing Cluster Names

	argocd cluster list

Output example:
	SERVER                          NAME
	https://10.0.0.1:6443           dev-cluster
	https://10.0.0.2:6443           prod-cluster

---

### Application Example (Name Strategy)

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: name-based-app
	spec:
	  destination:
	    name: prod-cluster
	    namespace: my-namespace
	  project: default
	  source:
	    repoURL: https://github.com/your/repo.git
	    targetRevision: main
	    path: app

---

### When to Use Name Strategy

Use `name` when:
- you want human-readable cluster references
- large teams manage many clusters
- you prefer logical naming over URLs

---

### Pros and Cons

Pros:
- readable
- easier governance
- works well at scale

Cons:
- name must be managed carefully
- renaming clusters requires updates

---

## 8. Configuring the Cluster Tracking Strategy Globally

Cluster tracking behavior is configured in the **argocd-cm ConfigMap**.

---

### argocd-cm Configuration

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: argocd-cm
	  namespace: argocd
	data:
	  application.instanceLabelKey: argocd.argoproj.io/instance
	  application.clusterTrackingMethod: server

Possible values:
- server
- label
- name

After updating this:
- restart Argo CD components
- new behavior takes effect

---

## 9. How Application Destination Changes per Strategy

### Server Strategy

	destination:
	  server: https://1.2.3.4:6443
	  namespace: my-namespace

---

### Label Strategy

	destination:
	  name: env=staging
	  namespace: my-namespace

---

### Name Strategy

	destination:
	  name: prod-cluster
	  namespace: my-namespace

---

## 10. Choosing the Right Strategy

| Use Case                                | Best Strategy |
|----------------------------------------|---------------|
| Static cluster, simple setup            | server        |
| Clusters recreated frequently          | label         |
| Large teams, many clusters              | name          |
| Dynamic cloud infrastructure           | label         |
| Governance and readability              | name          |

---

## 11. Final Mental Model

- Cluster tracking answers **“where should this app live?”**
- Strategy defines **how stable that answer is**
- `server` tracks infrastructure
- `label` tracks intent
- `name` tracks identity

**Pick the strategy that matches how often your clusters change, not how often your apps change.**

Once cluster tracking is chosen correctly, GitOps becomes predictable, safe, and scalable.
