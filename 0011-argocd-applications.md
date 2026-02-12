# Argo CD Applications

## What Is an Argo CD Application?

An **Application in Argo CD** is:
- a **group of Kubernetes manifests**
- defined as a **Custom Resource Definition (CRD)**
- managed declaratively

In simple words:

    An Argo CD Application tells Argo CD what to deploy, from where, and where to deploy it.

---

## Step 1: Set the Default Namespace to Argo CD

Instead of passing `-n argocd` every time, we set the namespace once.

	kubectl config set-context --current --namespace=argocd

Now verify:

	kubectl get pods

You should see all Argo CD pods because `argocd` is now the default namespace.

---

## Step 2: Get the Argo CD Admin Password

Argo CD creates an initial admin password as a Kubernetes secret.

	kubectl get secret argocd-initial-admin-secret -o yaml

Extract and decode the password:

	echo <base64-password> | base64 --decode

Save this password — you’ll need it for UI and CLI login.

---

## Step 3: Login to Argo CD UI

Access Argo CD in your browser using:

	https://localhost:32073

Login with:
- Username: admin
- Password: decoded password

You should now see the Argo CD dashboard.

---

## Step 4: Login to Argo CD Using CLI

### Get Node IP Address

	kubectl get nodes -o wide

Note the node IP.

### Login Using CLI

	argocd login <node-ip>:32073 --insecure --grpc-web

Credentials:
- Username: admin
- Password: decoded password

Now your CLI is authenticated.

---

## Step 5: Prepare a Helm-Based Application Repository

An Argo CD Application needs a **group of manifests**.  
We will use a **Helm chart** for this purpose.

### Create Project Structure

	mkdir argocd-application
	cd argocd-application
	mkdir helm
	helm create nginx

This creates a Helm chart structure:

	nginx/
	├── Chart.yaml
	├── values.yaml
	├── templates/

Push this repository to GitHub.

---

## Step 6: Writing an Argo CD Application From Scratch

An Application is a Kubernetes resource.

Key points:
- kind: Application
- apiVersion: argoproj.io/v1alpha1

### Application YAML

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: application-from-scratch
	spec:
	  destination:
	    namespace: default
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/helm/nginx
	    repoURL: https://github.com/devopshobbies/argocd-tutorial.git
	    targetRevision: main

### Apply the Application

	kubectl apply -f application.yaml

Verify:

	kubectl get applications

You will see:
- Sync status: OutOfSync
- Health: Missing

---

## Step 7: Sync the Application

In the Argo CD UI:
- Click **Sync**
- Click **Synchronize**

Now check:

	kubectl get all -n default

You will see NGINX resources deployed.

---

## Step 8: Understanding OutOfSync Behavior

Change `replicaCount` in `values.yaml` from 1 to 2 and push to Git.

Argo CD:
- checks for changes every **3 minutes**
- does NOT update instantly

To force detection:
- click **Refresh** in UI

Status changes to **OutOfSync**.

Click **Sync** → application updates → replicas become 2.

---

## How Argo CD Detects Helm Automatically

You never told Argo CD “this is Helm”.

Argo CD:
- goes to the specified path
- finds `Chart.yaml`
- automatically chooses **Helm** as the builder

If no `Chart.yaml` exists:
- Argo CD treats it as **plain directory manifests**

---

## Step 9: Changing Helm Release Name

By default:
- Helm release name = Application name

To override:

	source:
	  helm:
	    releaseName: application-from-helm

Apply again:

	kubectl apply -f application.yaml

Now:
- old resources must be **pruned**
- click **Sync**
- check **Prune**
- then **Synchronize**

---

## Step 10: Overriding Helm Values Using Parameters

Instead of editing `values.yaml`, override via Application YAML.

	source:
	  helm:
	    releaseName: application-from-helm
	    parameters:
	    - name: replicaCount
	      value: "3"

Apply:

	kubectl apply -f application.yaml

Sync → replicas change to 3.

---

## Step 11: Overriding Multiple Values Using Value Files

Create `custom-values.yaml` in Git.

Then update Application YAML:

	source:
	  helm:
	    releaseName: application-from-helm
	    valueFiles:
	    - custom-values.yaml

Apply and sync.

This is the **recommended approach** for complex overrides.

---

## Step 12: Using Remote Helm Charts Directly

You can deploy Helm charts **without cloning them into Git**.

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: sealed-secrets
	  namespace: argocd
	spec:
	  project: default
	  source:
	    chart: sealed-secrets
	    repoURL: https://bitnami-labs.github.io/sealed-secrets
	    targetRevision: 1.16.1
	    helm:
	      releaseName: sealed-secrets
	  destination:
	    server: https://kubernetes.default.svc
	    namespace: default

Apply and sync — the chart is deployed directly.

---

## Step 13: Directory-Based Applications (No Helm)

Now let’s use **plain Kubernetes manifests**.

### Create Directory Structure

	mkdir directoryofmanifests

Move manifests inside:

	deployment.yaml
	service.yaml
	serviceaccount.yaml
	test-connection.yaml

Push changes to Git.

---

## Create Directory-Based Application

	apiVersion: argoproj.io/v1alpha1
	kind: Application
	metadata:
	  name: application-directory
	spec:
	  destination:
	    namespace: directory
	    server: https://kubernetes.default.svc
	  project: default
	  source:
	    path: argocd-application/directoryofmanifests
	    repoURL: https://github.com/devopshobbies/argocd-tutorial.git
	    targetRevision: main

Create namespace:

	kubectl create ns directory

Apply application.

---

## Step 14: Excluding Files From Directory

Exclude a file:

	source:
	  directory:
	    exclude: 'serviceaccount.yaml'

Apply → **Prune + Sync** → file removed.

---

## Step 15: Including Only Specific Files

Include only selected files:

	source:
	  directory:
	    include: '{serviceaccount.yaml,service.yaml}'

Everything else is deleted during prune.

---

## Step 16: Recursive Directory Deployment

Enable nested directories:

	source:
	  directory:
	    recurse: true

Argo CD will deploy all manifests recursively.

---

## Final Mental Model

- Application = desired state definition
- Git = truth
- Argo CD = continuous enforcer
- Sync = reconcile Git to cluster
- Prune = remove unwanted resources
- Helm or Directory = build strategy auto-detected

Once you understand Applications deeply, **everything else in Argo CD becomes obvious**.

This is the heart of GitOps.
