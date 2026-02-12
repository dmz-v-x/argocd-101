# Argo CD CLI Commands — A Complete Step-by-Step Workflow Guide

## Phase 1 — Connecting to Argo CD (Authentication & Context)

Before anything else, your CLI must talk to the Argo CD API server.

---

## 1. argocd login

	argocd login <argocd-server>

Example:

	argocd login localhost:32073 --insecure --grpc-web

What this does:
- Authenticates your CLI to Argo CD
- Stores a token locally
- Sets the active Argo CD context

Expected output:
- Prompts for username/password
- Login successful message

This is **mandatory** before most commands.

---

## 2. argocd logout

	argocd logout

What this does:
- Removes stored credentials
- Ends the current CLI session

Use when:
- switching users
- rotating credentials
- security cleanup

---

## 3. argocd context

	argocd context

What this does:
- Shows the current Argo CD context
- Displays which server you’re connected to

Useful when:
- working with multiple Argo CD instances

---

## 4. argocd version

	argocd version

What this does:
- Shows CLI version
- Shows Argo CD server version (if logged in)

Output includes:
- client version
- server version
- build info

Important for:
- debugging version mismatches
- upgrade verification

---

## Phase 2 — User & Account Management

These commands **require you to be logged in**.

---

## 5. argocd account list

	argocd account list

What this does:
- Lists all Argo CD accounts
- Shows enabled/disabled status
- Shows capabilities (login, apiKey)

Used by:
- admins
- security audits

---

## 6. argocd account get-user-info

	argocd account get-user-info

What this does:
- Shows details about the currently logged-in user
- Username
- Issuer
- Groups

Useful for:
- RBAC debugging
- SSO verification

---

## 7. argocd account update-password

	argocd account update-password

What this does:
- Allows the logged-in user to change their password
- Prompts for old and new password

Used for:
- password rotation
- security compliance

---

## 8. argocd relogin

	argocd relogin

What this does:
- Forces re-authentication
- Refreshes expired tokens

Useful when:
- token expires
- permissions change

---

## Phase 3 — Cluster Management (Multi-Cluster GitOps)

Before deploying apps, Argo CD must know **where** to deploy them.

---

## 9. argocd cluster add

	argocd cluster add <kube-context>

Example:

	argocd cluster add dev-cluster

What this does:
- Registers a Kubernetes cluster with Argo CD
- Stores credentials securely
- Enables GitOps deployment to that cluster

Expected behavior:
- Prompts for confirmation
- Creates ServiceAccount + RBAC in target cluster

---

## 10. argocd cluster list

	argocd cluster list

What this does:
- Lists all registered clusters
- Shows server URL
- Shows cluster name and connection status

Used to:
- verify cluster registration
- check connectivity

---

## 11. argocd cluster remove

	argocd cluster remove <cluster-name>

What this does:
- Removes cluster from Argo CD
- Stops all GitOps operations to that cluster

Use with caution:
- apps targeting this cluster will fail

---

## Phase 4 — Repository Management (Git as Source of Truth)

Argo CD must access Git repositories.

---

## 12. argocd repo add

	argocd repo add <repo-url>

Example:

	argocd repo add https://github.com/my-org/my-repo.git

What this does:
- Registers a Git repository
- Optionally stores credentials
- Makes repo available for apps

---

## 13. argocd repo list

	argocd repo list

What this does:
- Lists all configured repositories
- Shows connection status
- Shows repo type (git/helm)

Used to:
- verify access
- debug auth issues

---

## 14. argocd repo rm

	argocd repo rm <repo-url>

What this does:
- Removes repository from Argo CD
- Apps using it will stop syncing

---

## 15. argocd repo update

	argocd repo update <repo-url>

What this does:
- Updates credentials or settings
- Useful after token rotation

---

## Phase 5 — Application Lifecycle (Core GitOps Flow)

This is the **heart of Argo CD**.

---

## 16. argocd app create

	argocd app create <app-name>

Example:

	argocd app create my-app \
	  --repo https://github.com/my-org/my-repo.git \
	  --path apps/my-app \
	  --dest-server https://kubernetes.default.svc \
	  --dest-namespace default

What this does:
- Creates an Argo CD Application
- Links Git → Cluster → Namespace

---

## 17. argocd app list

	argocd app list

What this does:
- Lists all applications
- Shows sync status
- Shows health status

First command you run after app creation.

---

## 18. argocd app get

	argocd app get <app-name>

What this does:
- Shows detailed application info
- Source, destination
- Sync status
- Health
- Conditions

---

## 19. argocd app logs

	argocd app logs <app-name>

What this does:
- Shows controller logs related to the app
- Useful for debugging sync failures

---

## 20. argocd app sync

	argocd app sync <app-name>

What this does:
- Forces deployment
- Applies desired state from Git
- Can prune resources

Core GitOps command.

---

## 21. argocd app wait

	argocd app wait <app-name>

What this does:
- Blocks until app is:
  - synced
  - healthy
- Useful in CI/CD pipelines

---

## 22. argocd app status

	argocd app status <app-name>

What this does:
- Shows concise status
- Good for scripting and checks

---

## 23. argocd app history

	argocd app history <app-name>

What this does:
- Shows previous deployments
- Git revisions
- Sync timestamps

---

## 24. argocd app rollback

	argocd app rollback <app-name> <revision>

What this does:
- Rolls back app to previous Git revision
- GitOps-native rollback

---

## 25. argocd app set

	argocd app set <app-name>

What this does:
- Modifies app settings
- Change repo, path, sync policy, etc.

---

## 26. argocd app delete

	argocd app delete <app-name>

What this does:
- Deletes the Argo CD Application
- Optionally deletes managed resources

---

## 27. argocd app resources

	argocd app resources <app-name>

What this does:
- Lists Kubernetes resources managed by app
- Useful for audits and debugging

---

## 28. argocd app diff

	argocd app diff <app-name>

What this does:
- Shows differences:
  - Git vs live cluster
- Critical for safe reviews

---

## 29. argocd app patch-resource

	argocd app patch-resource

What this does:
- Patches a specific resource
- Advanced use cases
- Usually avoided in strict GitOps

---

## 30. argocd app validate

	argocd app validate <app-name>

What this does:
- Validates manifests
- Catches errors before sync

---

## Phase 6 — Project Management (Governance Layer)

---

## 31. argocd proj create

	argocd proj create <project-name>

What this does:
- Creates a new Argo CD project
- Enables governance and restrictions

---

## 32. argocd proj list

	argocd proj list

What this does:
- Lists all projects
- Including default project

---

## 33. argocd proj get

	argocd proj get <project-name>

What this does:
- Shows project configuration
- Source repos
- Destinations
- Roles

---

## 34. argocd proj set

	argocd proj set <project-name>

What this does:
- Modifies project settings
- Repos, destinations, roles

---

## 35. argocd proj delete

	argocd proj delete <project-name>

What this does:
- Deletes the project
- Apps must be removed first

---

## Phase 7 — Admin & Help

---

## 36. argocd admin dashboard

	argocd admin dashboard

What this does:
- Launches Argo CD UI locally
- Admin-only access

---

## 37. argocd help

	argocd help

What this does:
- Shows all commands
- Use with subcommands

Example:

	argocd app --help

---

## Final Mental Model

The CLI flow always looks like this:

1. Login  
2. Register clusters  
3. Add repositories  
4. Create applications  
5. Sync & monitor  
6. Rollback if needed  
7. Govern using projects  

Once you master this flow, **you can operate Argo CD entirely from the CLI** — confidently and safely in production.

This is **real GitOps operations**, end to end.
