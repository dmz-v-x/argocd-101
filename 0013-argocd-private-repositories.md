# Argo CD Private Repositories

## Public vs Private Repositories in Argo CD

### Public Repository
- No authentication required
- Argo CD can fetch manifests directly

### Private Repository
- Authentication **is mandatory**
- Argo CD **cannot access GitHub/GitLab without credentials**

So the question becomes:

> How does Argo CD authenticate to private repositories?

---

## Two Authentication Methods Supported by Argo CD

Argo CD supports **two Git authentication mechanisms**:

1. HTTPS authentication  
2. SSH authentication  

Both are handled using **Kubernetes Secrets**.

---

## Why Argo CD Uses Kubernetes Secrets

Argo CD is Kubernetes-native.

So instead of:
- storing credentials inside Argo CD itself
- hardcoding credentials in YAML files

Argo CD:
- reads Git credentials from **Kubernetes Secret objects**
- watches secrets labeled in a special way
- automatically loads repositories from them

This makes Git access:
- secure
- declarative
- auditable

---

## Authentication Method 1 — HTTPS (Token / Username + Password)

This is the **most commonly used method**.

### Step 1: Create a GitHub Personal Access Token (PAT)

In GitHub:
- Settings
- Developer Settings
- Personal Access Tokens
- Generate New Token

Make sure the token has:
- repo access (read-only is enough)

Create a **private repository** and push content to it.

---

## Step 2: Create Kubernetes Secret for HTTPS Authentication

Create a secret manifest file.

	https-secret.yaml

	apiVersion: v1
	kind: Secret
	metadata:
	  name: argocd-private-repo
	  namespace: argocd
	  labels:
	    argocd.argoproj.io/secret-type: repository
	stringData:
	  type: git
	  username: argocd-private-repo
	  password: <github-personal-access-token>

### Important Points

- The label is **mandatory**:
  
	  argocd.argoproj.io/secret-type: repository

- `username` can be:
  - GitHub username
  - or any identifier name
- `password` is the GitHub token

---

## Step 3: Apply the Secret

	kubectl apply -f https-secret.yaml

---

## Step 4: Fix the Missing Repository URL

At this point:
- Secret exists
- But repository URL is missing

So Argo CD **cannot connect yet**.

Go to:
- Argo CD UI
- Settings → Repositories

You’ll see:
- Connection status = Failed

---

## Step 5: Add Repository URL to the Secret

Update the secret:

	https-secret.yaml

	apiVersion: v1
	kind: Secret
	metadata:
	  name: argocd-private-repo
	  namespace: argocd
	  labels:
	    argocd.argoproj.io/secret-type: repository
	stringData:
	  type: git
	  url: https://github.com/<username>/<repo-name>.git
	  username: argocd-private-repo
	  password: <github-personal-access-token>

Apply again:

	kubectl apply -f https-secret.yaml

---

## Step 6: Verify in Argo CD UI

Now go back to:
- Settings → Repositories

You should see:
- Repository added
- Connection status = Successful

At this point:
> Argo CD can securely access your private Git repository using HTTPS.

---

## Authentication Method 2 — SSH-Based Authentication

SSH is commonly used when:
- organizations disable HTTPS tokens
- stricter security policies are enforced

---

## Step 1: Add SSH Public Key to GitHub

On your local machine:
- Generate SSH key pair (if not already present)
- Add **public key** to GitHub account

---

## Step 2: Create SSH Repository Secret

Create a new secret manifest.

	ssh-secret.yaml

	apiVersion: v1
	kind: Secret
	metadata:
	  name: argocd-private-ssh-repo
	  namespace: argocd
	  labels:
	    argocd.argoproj.io/secret-type: repository
	stringData:
	  type: git
	  url: <ssh-url-of-github-repo>
	  sshPrivateKey: |
	    -----BEGIN OPENSSH PRIVATE KEY-----
	    <your-private-key>
	    -----END OPENSSH PRIVATE KEY-----

---

## Step 3: Apply SSH Secret

	kubectl apply -f ssh-secret.yaml

---

## Step 4: Verify SSH Repository

Go to:
- Argo CD UI → Settings → Repositories

You should now see:
- SSH repository listed
- Connection status = Successful

---

## Adding Private Repositories Using CLI

Argo CD also provides a CLI-based way.

To see options:

	argocd repo add -h

You’ll find flags for:
- HTTPS username/password
- SSH private key
- repository URLs

CLI internally:
- creates Kubernetes secrets
- registers them with Argo CD

---

## Problem: Multiple Private Repositories

If you manage:
- multiple repos
- same GitHub org
- same credentials

Creating **one secret per repository** becomes painful.

Argo CD solves this with **credential templates**.

---

## Repository Credential Templates (repo-creds)

Credential templates:
- define credentials once
- apply to multiple repositories
- work using URL prefix matching

---

## Step 1: Change Secret Type Label

Instead of:

	argocd.argoproj.io/secret-type: repository

We use:

	argocd.argoproj.io/secret-type: repo-creds

---

## Step 2: Create Credential Template Secret

	template.yaml

	apiVersion: v1
	kind: Secret
	metadata:
	  name: argocd-template
	  namespace: argocd
	  labels:
	    argocd.argoproj.io/secret-type: repo-creds
	stringData:
	  type: git
	  url: https://github.com/dmz_v_x
	  sshPrivateKey: |
	    -----BEGIN OPENSSH PRIVATE KEY-----
	    <private-key>
	    -----END OPENSSH PRIVATE KEY-----

### Key Difference

- URL is **prefix only**
- No repo name
- No folder path

This means:
> Any repository under `https://github.com/dmz_v_x/*` can use this credential.

---

## Step 3: Apply the Template

	kubectl apply -f template.yaml

---

## Step 4: Verify in Argo CD UI

Go to:
- Settings → Repositories

You will see:
- Credential Template listed
- Not a single repository
- Marked as reusable credentials

---

## Using Credential Templates in Applications

Now when you create an application with:

	repoURL: https://github.com/dmz_v_x/some-private-repo.git

Argo CD will:
- automatically match the URL prefix
- use the credential template
- authenticate successfully

No extra secrets required.

---

## Final Mental Model

- Argo CD never stores credentials internally
- Everything is Kubernetes-native
- Secrets control Git access
- Two auth methods:
  - HTTPS (token)
  - SSH (private key)
- Credential templates eliminate duplication
- Everything is declarative and auditable

**This is how GitOps stays secure at scale.**
