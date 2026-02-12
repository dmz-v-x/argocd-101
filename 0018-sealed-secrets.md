# Encrypting Sensitive Data in GitOps with Sealed Secrets (Argo CD + Kubernetes)

## The Core Problem: Secrets + Git = Risk

Kubernetes Secrets are **not encrypted**.

They are only **base64-encoded**, which means:

	anyone with Git access → can decode secrets instantly

Example:

	data:
	  password: cGFzc3dvcmQ=

This is **not encryption**.

So if you:
- store Secrets in Git
- use GitOps
- use Argo CD

You **must** solve secret encryption properly.

---

## The GitOps Requirement

In GitOps:
- Git is the source of truth
- Everything lives in Git
- Argo CD reads from Git

So we need a way to:
- keep secrets **encrypted in Git**
- but **usable in Kubernetes**

This is exactly what **Sealed Secrets** provides.

---

## What Is Sealed Secrets?

**Sealed Secrets** is an open-source project by Bitnami.

It allows you to:
- encrypt Kubernetes Secrets
- store encrypted secrets safely in Git
- decrypt them **only inside the target cluster**

The encrypted object is called a **SealedSecret**.

---

## High-Level Architecture

Sealed Secrets uses **public-key cryptography**.

There are two main components:

1. Sealed Secrets Controller  
2. kubeseal CLI  

---

## Component 1: Sealed Secrets Controller

Runs:
- **inside your Kubernetes cluster**

Responsibilities:
- holds a **private key**
- decrypts SealedSecret objects
- creates real Kubernetes Secrets
- reconciles continuously
- rotates keys automatically

Only this controller can decrypt secrets.

---

## Component 2: kubeseal CLI

Runs:
- on your **local machine**
- or in CI/CD pipelines

Responsibilities:
- fetches the controller’s **public key**
- encrypts Kubernetes Secrets
- outputs SealedSecret YAML
- never sees the private key

---

## Security Guarantee (Very Important)

- Public key → used for encryption
- Private key → never leaves the cluster
- Git only stores encrypted data
- Even cluster admins cannot decrypt manually

This is **one-way encryption**.

---

## Installing kubeseal CLI

Install kubeseal on your local machine:

	curl -Lo kubeseal https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/kubeseal-linux-amd64
	chmod +x kubeseal
	sudo mv kubeseal /usr/local/bin

Verify:

	kubeseal --version

---

## Installing the Sealed Secrets Controller

Install once per cluster:

	kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.2/controller.yaml

This creates:
- controller deployment
- service
- keys

Verify:

	kubectl get pods -n kube-system | grep sealed

---

## Creating a Normal Kubernetes Secret (Example)

Let’s create a simple secret.

	kubectl create secret generic pacman-secret \
	  --from-literal=user=pacman \
	  --from-literal=pass=pacman

Verify:

	kubectl get secret pacman-secret -o yaml

You’ll see:

	apiVersion: v1
	kind: Secret
	data:
	  user: cGFjbWFu
	  pass: cGFjbWFu

This **must never go to Git**.

---

## Converting Secret → SealedSecret

Now we encrypt it.

	kubectl get secret pacman-secret -o yaml \
	  | kubeseal -o yaml > pacman-sealedsecret.yaml

Output:

	apiVersion: bitnami.com/v1alpha1
	kind: SealedSecret
	metadata:
	  name: pacman-secret
	  namespace: default
	spec:
	  encryptedData:
	    user: AgBhYDZQzOwinetPceZL...
	    pass: AgBJR1AgZ5Gu5NOVs...

This file:
- is encrypted
- cannot be decoded
- is safe for Git

---

## What Happens During Encryption?

Internally:

1. kubeseal fetches the controller’s public key
2. encrypts secret values
3. outputs SealedSecret YAML
4. plaintext never leaves your machine

---

## Pushing SealedSecret to Git

Now you can safely:

- commit pacman-sealedsecret.yaml
- push to Git
- use it with Argo CD

Git contains **only encrypted data**.

---

## Deploying Sealed Secrets Using Argo CD

Create an Argo CD application:

	argocd app create pacman \
	  --repo https://github.com/your-org/your-repo \
	  --path k8s/sealedsecrets \
	  --dest-server https://kubernetes.default.svc \
	  --dest-namespace default \
	  --sync-policy auto

Check status:

	argocd app list

Argo CD will:
- apply the SealedSecret
- controller decrypts it
- Kubernetes Secret is created automatically

---

## Verifying Decryption in Cluster

	kubectl get secret pacman-secret -o yaml

You’ll see a normal Secret:

	data:
	  user: cGFjbWFu
	  pass: cGFjbWFu

Your app uses it normally.

---

## Production Reality: Why This Matters

Storing secrets in plain YAML:
- leaks credentials
- violates security policies
- breaks compliance

Sealed Secrets:
- keeps Git clean
- enforces encryption
- works perfectly with GitOps

---

## kubeseal vs Sealed Secrets Controller

| Component | Runs Where | Purpose |
|---------|-----------|--------|
| kubeseal | Local / CI | Encrypt secrets |
| Controller | Cluster | Decrypt secrets |

You install:
- kubeseal on your machine
- controller once per cluster

---

## How Sealed Secrets Work End-to-End

1. Create normal Secret locally
2. Encrypt with kubeseal
3. Commit encrypted YAML to Git
4. Argo CD deploys it
5. Controller decrypts it
6. Kubernetes Secret appears
7. App consumes secret

At no point does Git see plaintext.

---

## Real Production Example: MongoDB URI

### Create Secret YAML

	mongo-secret.yaml

	apiVersion: v1
	kind: Secret
	metadata:
	  name: mongodb-uri
	  namespace: default
	type: Opaque
	stringData:
	  uri: "mongodb+srv://admin:password@mydb.mongodb.net/test"

---

### Encrypt It

	kubeseal \
	  --controller-namespace kube-system \
	  --controller-name sealed-secrets \
	  --format yaml \
	  < mongo-secret.yaml > mongo-sealed.yaml

---

### Apply to Cluster (or via Argo CD)

	kubectl apply -f mongo-sealed.yaml

Controller decrypts it automatically.

---

### Use Secret in Application

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: mongo-app
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: mongo-app
	  template:
	    metadata:
	      labels:
	        app: mongo-app
	    spec:
	      containers:
	      - name: mongo-container
	        image: your-docker-image
	        env:
	        - name: MONGO_URI
	          valueFrom:
	            secretKeyRef:
	              name: mongodb-uri
	              key: uri

App runs with secret — Git never saw it.

---

## How Decryption Actually Works

| Component | Role |
|--------|------|
| kubeseal | Encrypts using public key |
| Controller | Decrypts using private key |
| App | Uses normal Secret |

No manual decryption ever happens.

---

## Key Guarantees

- Secrets are encrypted at rest in Git
- Secrets are decrypted only in cluster
- Private key never leaves cluster
- Even repo admins cannot decrypt secrets

---

## Final Mental Model

- Git must never contain plaintext secrets
- Kubernetes Secrets are unsafe by default
- Sealed Secrets makes GitOps secure
- kubeseal encrypts
- controller decrypts
- Argo CD deploys

**This is the correct, production-grade way to handle secrets in GitOps.**

Once you understand this flow,  
you can confidently run Argo CD in real production environments.
