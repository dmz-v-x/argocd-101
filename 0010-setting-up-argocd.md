# Setting Up Argo CD — Installation, Access, and CLI 

## . Installation & Setup Overview

Setting up Argo CD involves three major phases:

1. Installing Argo CD into a Kubernetes cluster  
2. Exposing and accessing Argo CD  
3. Interacting with Argo CD using CLI and UI  

Argo CD is **Kubernetes-native**, so everything starts inside the cluster.

---

##  Installing Argo CD

Argo CD is installed **inside a Kubernetes cluster**, typically in its own namespace.

---

## Namespace Setup

Best practice is to isolate Argo CD in a dedicated namespace.

    kubectl create namespace argocd

This ensures:
- clear separation of concerns
- easier RBAC management
- safer upgrades and maintenance

All Argo CD components will live in this namespace.

---

## Installing Argo CD Using Manifests

This is the **official and most common installation method**, especially for learning and debugging.

    kubectl apply -n argocd \
      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

What this does:
- installs all Argo CD components
- creates deployments, services, CRDs
- sets up RBAC and controllers

This method gives you **full visibility and control** over what gets installed.

---

### When to Use Manifest-Based Installation

- learning Argo CD
- debugging issues
- minimal clusters
- when you want explicit YAML control

---

## Installing Argo CD Using Helm

Helm is preferred in **production and enterprise environments**.

Steps:

1. Add Argo Helm repository

    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update

2. Install Argo CD

    helm install argocd argo/argo-cd \
      --namespace argocd \
      --create-namespace

Benefits of Helm:
- easy upgrades
- configurable values
- environment-specific overrides
- standardized release management

---

### When to Use Helm Installation

- production clusters
- HA setups
- managed Kubernetes environments
- teams already standardized on Helm

---

## Initial Admin Password

After installation, Argo CD creates an **admin user** by default.

The initial password is stored as a Kubernetes secret.

    kubectl -n argocd get secret argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d

Important notes:
- this password is auto-generated
- should be changed immediately
- secret is deleted automatically after first login in newer versions

---

## Accessing Argo CD

Argo CD exposes an API server that must be accessed externally.  
There are **multiple ways**, depending on environment and maturity.

---

## Accessing Argo CD via Port-Forwarding

This is the **simplest and safest** method for local development.

    kubectl port-forward svc/argocd-server \
      -n argocd 8080:443

Then open:

    https://localhost:8080

Characteristics:
- no cluster exposure
- ideal for learning
- temporary access

This is **not recommended for production**.

---

## Accessing Argo CD via NodePort

NodePort exposes Argo CD on a port of every node.

    kubectl patch svc argocd-server -n argocd \
      -p '{"spec": {"type": "NodePort"}}'

Then access:

    https://<node-ip>:<node-port>

Pros:
- simple
- works without cloud load balancer

Cons:
- not secure by default
- manual TLS handling
- not ideal for production

---

## Accessing Argo CD via LoadBalancer

In cloud environments, LoadBalancer is common.

    kubectl patch svc argocd-server -n argocd \
      -p '{"spec": {"type": "LoadBalancer"}}'

Cloud provider provisions:
- external IP
- managed load balancer

Pros:
- easy external access
- managed networking

Cons:
- cost
- limited customization
- still not ideal without ingress-level security

---

## Accessing Argo CD via Ingress (Production Mindset)

Ingress is the **recommended production approach**.

Typical setup:
- Ingress Controller (NGINX, Traefik, etc.)
- TLS termination
- Authentication integration
- Domain-based access

Benefits:
- secure HTTPS access
- SSO integration
- rate limiting
- fine-grained routing control

This aligns with **enterprise-grade GitOps setups**.

---

## Argo CD CLI

The Argo CD CLI is a **first-class interface**, not an afterthought.

It is designed for:
- automation
- scripting
- CI integration
- power users

---

## Installing the Argo CD CLI

Example for Linux:

    curl -sSL -o argocd \
      https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    chmod +x argocd
    sudo mv argocd /usr/local/bin/

---

## Logging In Using CLI

    argocd login <ARGOCD_SERVER>

Example:

    argocd login localhost:8080

You will provide:
- username (admin)
- password

After login, the CLI stores an auth token locally.

---

## Listing Applications

    argocd app list

This command:
- shows all registered applications
- displays sync and health status
- helps with quick diagnostics

---

## Syncing an Application

    argocd app sync <app-name>

What this does:
- forces reconciliation
- applies desired state immediately
- useful for manual intervention

Syncing can also be:
- automated
- scheduled
- policy-driven

---

## CLI vs UI — When to Use What

### Use the UI When:
- visualizing drift
- inspecting diffs
- debugging deployments
- onboarding new team members

### Use the CLI When:
- automating workflows
- scripting operations
- CI/CD integration
- bulk operations

Both talk to the **same API Server**.

---

## Final Best Practices

- Use manifests for learning, Helm for production
- Always isolate Argo CD in its own namespace
- Never expose Argo CD insecurely
- Prefer Ingress + TLS for production
- Use CLI for automation, UI for visibility

---

## Final Mental Model

- Argo CD runs inside Kubernetes
- You install it like any other Kubernetes workload
- Access is just networking configuration
- CLI and UI are just clients to the same API

Once Argo CD is installed correctly, **Git becomes your deployment interface**.
