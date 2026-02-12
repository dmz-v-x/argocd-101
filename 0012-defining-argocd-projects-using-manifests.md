# Defining Argo CD Projects Using Manifests

## What Is an Argo CD Project?

An **Argo CD Project** is a **logical container for applications**.

Think of it as:
- a boundary
- a policy layer
- a security guard

> Applications do not exist freely in Argo CD — they always belong to a project.

A project can contain:
- one application
- many applications

And most importantly:
> A project can **restrict what applications are allowed to do**.

---

## Why Do We Need Projects?

Projects help solve:
- uncontrolled deployments
- accidental use of wrong repositories
- deploying to wrong namespaces or clusters
- excessive permissions for CI/CD systems
- lack of multi-team isolation

Projects enforce **who can deploy what, from where, and to where**.

---

## Three Ways Projects Restrict Applications

Argo CD Projects can restrict applications in **three major dimensions**:

1. Source repositories  
2. Destinations (cluster + namespace)  
3. Kubernetes resources  

We’ll go through each one hands-on.

---

## Switching to Argo CD Namespace

Set Argo CD as the default namespace so we don’t need `-n argocd` everywhere.

	kubectl config set-context --current --namespace=argocd

Verify:

	kubectl get pods

You should now see all Argo CD pods directly.

---

## Exploring the Default Project

List all projects:

	kubectl get appproject

You will see a project named `default`.

View its full configuration:

	kubectl get appproject default -o yaml

You’ll notice:

	apiVersion: argoproj.io/v1alpha1
	kind: AppProject
	metadata:
	  name: default
	  namespace: argocd
	spec:
	  clusterResourceWhitelist:
	  - group: '*'
	    kind: '*'
	  destinations:
	  - namespace: '*'
	    server: '*'
	  sourceRepos:
	  - '*'

### What This Means

- All repositories allowed
- All namespaces allowed
- All clusters allowed
- All resources allowed

> The default project has **no restrictions**.

---

## Creating a Custom Project

Now let’s create our own project.

### Directory Setup

	mkdir customProject
	cd customProject
	vi project.yaml

---

## Project With Source Repository Restriction

	project.yaml

	apiVersion: argoproj.io/v1alpha1
	kind: AppProject
	metadata:
	  name: project-1
	  namespace: argocd
	spec:
	  clusterResourceWhitelist:
	  - group: '*'
	    kind: '*'
	  destinations:
	  - namespace: '*'
	    server: '*'
	  sourceRepos:
	  - https://github.com/devopshobbies/argocd-tutorial.git

This project:
- allows only **one Git repository**
- blocks all other repositories

Apply the project:

	kubectl apply -f project.yaml

Verify:

	kubectl get appproject

---

## Creating an Application Using the Project

	application.yaml

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: app-1
	spec:
	  destination:
	    namespace: default
	    server: https://kubernetes.default.svc
	  project: project-1
	  source:
	    path: argocd-application/helm/nginx
	    repoURL: https://github.com/devopshobbies/argocd-tutorial.git
	    targetRevision: main

Apply:

	kubectl apply -f application.yaml

Now check UI or CLI.

---

## What Happens If the Repo Is Not Allowed?

If you use **any other repo URL**:
- `kubectl apply` will succeed
- but Argo CD will block the application internally

Symptoms:
- Application health = Unknown
- Status condition shows Error
- Error message:
  repository is not permitted by the project

Verify via CLI:

	argocd login localhost:32073 --insecure --grpc-web
	argocd app list
	kubectl describe app app-1

---

## Using Exclamation (!) in sourceRepos

Block a specific repository but allow all others:

	sourceRepos:
	- '!https://github.com/devopshobbies/argocd-tutorial.git'
	- '*'

Meaning:
- this repo is forbidden
- all other repos are allowed

---

## Restricting Applications Using Destinations

Projects can restrict:
- namespaces
- clusters (servers)

---

### Allow Deployment Only in `dev` Namespace

	project.yaml

	destinations:
	- namespace: dev
	  server: '*'

Now applications:
- **must deploy only to `dev` namespace**

If application tries `default` namespace:
- Argo CD throws error:
  destination does not match allowed destinations

---

### Fixing the Application

Create namespace:

	kubectl create ns dev

Update application:

	destination:
	  namespace: dev
	  server: https://kubernetes.default.svc

Application becomes OutOfSync → Sync → Running.

---

### Blocking a Specific Namespace

	destinations:
	- namespace: '!dev'
	  server: '*'

Meaning:
- cannot deploy to dev
- all other namespaces allowed

---

## Restricting Applications Using Resources

Projects can restrict **what Kubernetes resources** are allowed.

There are two scopes:
- cluster-scoped resources
- namespace-scoped resources

---

### Whitelisting Resources

	clusterResourceWhitelist:
	- group: '*'
	  kind: '*'

	namespaceResourceWhitelist:
	- group: ''
	  kind: ServiceAccount

Meaning:
- only ServiceAccount is allowed
- Deployments, Services, etc. are blocked

If application contains multiple resources:
- deployment will fail

---

### Blacklisting Resources

	namespaceResourceBlacklist:
	- group: apps
	  kind: Deployment

Meaning:
- Deployments are forbidden
- everything else is allowed

---

## Project Roles — Controlled Access for Automation

Projects support **roles**, which are essential for:
- CI/CD pipelines
- automation
- least-privilege access

---

## What Is a Project Role?

A role:
- belongs to a project
- defines permissions via policies
- can generate tokens
- limits access to project applications only

---

## Defining a Read-Only Role

	project.yaml

	spec:
	  roles:
	  - name: read-only
	    description: this role can read applications
	    policies:
	    - p, proj:project-7:read-only, applications, get, project-7/*, allow

Meaning:
- can only **get (read)** applications
- cannot sync, create, or delete

Apply project:

	kubectl apply -f project-7.yaml

---

## Generating a Token for the Role

	argocd proj role create-token project-7 read-only

Copy the generated token.

---

## Using Token to List Applications

Using admin:

	argocd app list

Using token:

	argocd app list --auth-token <token>

Only applications from `project-7` are visible.

---

## Trying to Sync With Read-Only Token

	argocd app sync app-9 --auth-token <token>

This fails because:
- role allows only `get`
- sync is not permitted

---

## Creating a Read + Sync Role

	project.yaml

	roles:
	- name: read-sync
	  description: read and sync applications
	  policies:
	  - p, proj:project-7:read-sync, applications, get, project-7/*, allow
	  - p, proj:project-7:read-sync, applications, sync, project-7/*, allow

Apply changes:

	kubectl apply -f project-7.yaml

Generate new token:

	argocd proj role create-token project-7 read-sync

---

## Syncing Application Using Token

	argocd app sync app-9 --auth-token <new-token>

Now sync works successfully.

---

## Important Rule About Tokens

> Anytime you update project role policies,  
> you **must generate a new token**.

Old tokens do not inherit new permissions.

---

## Final Mental Model

- Project = security boundary
- Applications live inside projects
- Projects restrict:
  - Git repositories
  - namespaces and clusters
  - Kubernetes resources
- Roles provide least-privilege access
- Tokens enable safe automation

**Projects are what make Argo CD production-grade.**  
Without projects, GitOps has no guardrails.
