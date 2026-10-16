# Connecting to the Bootcamp Kubernetes Cluster

_Last updated: 2026-10-16_

## What you need

You connect to the class Kubernetes (k3s) cluster with `kubectl` and one file: your personal kubeconfig, named `studentNN.yaml` (for example `student07.yaml`). That file already contains the cluster address, its certificate, your namespace and your login, so there is nothing else to configure.

**Treat your kubeconfig like a password.** The token inside it is your login. Never commit it to Git, paste it into chat, or share it with another student. If you think it has leaked, tell your instructor so it can be replaced.

## Step 1 — Install kubectl

`kubectl` is not part of the Lab 0 install, so add it now.

**Windows (PowerShell):**

```powershell
winget install -e --id Kubernetes.kubectl
```

Close and reopen PowerShell (or VS Code) so the new command is on your `PATH`.

**macOS (Terminal):**

```bash
brew install kubectl
```

Check it installed:

```bash
kubectl version --client
```

## Step 2 — Save your kubeconfig file

Download your own `studentNN.yaml` from the link your instructor shared. Then copy it to the place `kubectl` looks by default, `.kube/config` in your home folder. Replace `NN` with your number in the commands below.

If you already have a `.kube/config` (Docker Desktop or Rancher Desktop may have created one), rename it to `config.bak` first.

**Windows (PowerShell),** from the folder where you downloaded the file:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.kube" | Out-Null
Copy-Item .\studentNN.yaml "$HOME\.kube\config"
```

**macOS (Terminal):**

```bash
mkdir -p ~/.kube
cp ~/Downloads/studentNN.yaml ~/.kube/config
chmod 600 ~/.kube/config
```

Do not edit the file. Keep it out of your `java-bootcamp` repo folder so it can never be committed.

## Step 3 — Verify the connection

Run these three commands. If each prints what is shown, you are connected.

| Command | Expected result |
| --- | --- |
| `kubectl config current-context` | `studentNN` |
| `kubectl get pods` | `No resources found in studentNN namespace.` |
| `kubectl auth can-i create deployments` | `yes` |

"No resources found" is a success: you reached the cluster and your namespace is simply empty until you deploy something.

## Commands that are supposed to fail

Your account manages only your own namespace, `studentNN`. Cluster-wide commands return **Forbidden**, and that is correct:

- `kubectl get nodes`
- `kubectl get namespaces`
- `kubectl create namespace ...`

**Lab 42 note:** Step 2 of the lab asks you to create a `lab42-crm` namespace. Skip that command. Your namespace is already `studentNN` and every `kubectl` command uses it automatically, so you do not need `-n` or a `namespace:` field in your manifests.

## Troubleshooting

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| `kubectl` is not recognized | Not installed, or the terminal was open during install | Redo Step 1, then open a new terminal |
| `connection refused` to `localhost:8080` | `kubectl` cannot find your kubeconfig | Check the file is at `.kube/config` in your home folder (Step 2) |
| Connection timed out / unable to connect to the server | Your network blocks port 6443 | Try another network (for example a phone hotspot) and tell your instructor |
| `x509: certificate signed by unknown authority` | The file was edited or damaged | Download it again and copy it without opening it in an editor |
| `Unauthorized` | Wrong or altered file | Make sure it is your own `studentNN.yaml`, freshly downloaded |
| Context shows `docker-desktop` or `rancher-desktop` | An old config was not replaced | Rename the old config and redo Step 2 |
| `Forbidden` inside your own namespace | Permissions problem | Tell your instructor, with the exact command and error |

## Quick reference

| Item | Value |
| --- | --- |
| Kubernetes API | `https://32.189.172.33:6443` (already in your kubeconfig) |
| Your namespace | `studentNN` |
| Your kubeconfig | `studentNN.yaml` → saved as `.kube/config` |
| Ingress hostname for your app | `crm-api-studentNN.32.189.172.33.nip.io` |
| PostgreSQL (for your app's Secret) | `jdbc:postgresql://32.189.172.33:5432/bootcamp`, your own DB username and password |

Use your own number in the Ingress hostname so it does not clash with other students. Keep your database password in a Kubernetes Secret, never in a manifest you commit.
