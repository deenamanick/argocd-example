# Install ArgoCD

## 1) Make service routing work (kube-proxy + sysctls)

> This fixes your `10.96.0.1:443` timeout.

```bash
# On the control-plane (master)
sudo kubeadm init phase addon kube-proxy || true

# On ALL nodes (master + workers)
sudo modprobe br_netfilter || true
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl --system

# Verify from a pod (default ns)
kubectl run netcheck --rm -it --image=busybox --restart=Never -- sh -c \
'wget -qO- --timeout=5 https://10.96.0.1:443 && echo OK || echo FAILED'
# Expect to see some TLS bytes then "OK". If "FAILED", fix kube-proxy/logs before continuing:
#   kubectl -n kube-system get ds kube-proxy && kubectl -n kube-system logs ds/kube-proxy --tail=200
```

---

## 2) Install **non-HA** Argo CD (official manifest)

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Watch until all pods are Running/Ready
kubectl -n argocd get pods -w
```

---

## 3) (If Redis init keeps crashing) create the password Secret

```bash
# Create the secret with the expected key 'auth'
kubectl -n argocd create secret generic argocd-redis \
  --from-literal=auth="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 32)" \
  --dry-run=client -o yaml | kubectl apply -f -

# Bounce Argo CD to pick it up
kubectl -n argocd rollout restart deploy/argocd-redis \
  deploy/argocd-repo-server deploy/argocd-server
kubectl -n argocd rollout restart statefulset/argocd-application-controller
```

---

## 4) Login to the UI

```bash
# Port-forward
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open **[https://localhost:8080](https://localhost:8080)**, login as **admin** (paste the password), then change it.

---

## 5) Deploy a sample app (one-shot)

```bash
# Creates an Application that points to the guestbook example
kubectl -n argocd apply -f - <<'YAML'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
YAML
```

Check it rolled out:

```bash
kubectl -n guestbook get pods
```

## Access Argocd with Nodeport

```
# switch the argocd-server service to NodePort and pin the https nodePort
kubectl -n argocd patch svc argocd-server -p '{
  "spec": {
    "type": "NodePort",
    "ports": [
      { "name": "https", "port": 443, "targetPort": 8080, "nodePort": 30443 }
    ]
  }
}'

```
## Now in browser:
```
https://<any-node-IP>:30443
```

Nice! Since you’re already logged in, here’s the **quickest end-to-end test** to prove Argo CD works—no repo setup needed.

## 1) Deploy a sample app (Guestbook)

**Option A: via `kubectl` (one command)**

```bash
kubectl -n argocd apply -f - <<'YAML'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
YAML
```

**Option B: via UI**
New App → Name: `guestbook` → Project: `default`
Repo URL: `https://github.com/argoproj/argocd-example-apps`
Revision: `HEAD` → Path: `guestbook`
Destination: `https://kubernetes.default.svc` → Namespace: `guestbook`
Enable **Auto-Sync**, **Prune**, **Self-Heal** → Create → **Sync**.

Check it’s up:

```bash
kubectl -n guestbook get pods
```

---

## 2) Open the app UI to confirm it runs

Port-forward (simplest):

```bash
kubectl -n guestbook port-forward svc/guestbook-ui 8081:80
# visit http://localhost:8081
```

Or expose as NodePort (if you prefer, like you did for Argo CD):

```bash
kubectl -n guestbook patch svc guestbook-ui -p '{
  "spec": { "type": "NodePort", "ports": [ { "port": 80, "targetPort": 80, "nodePort": 30081, "name": "http" } ] }
}'
# visit http://<any-node-IP>:30081
```

---

## 3) Prove GitOps features (two quick checks)

### a) Self-heal (auto-revert drift)

```bash
# break it intentionally
kubectl -n guestbook scale deploy/guestbook-ui --replicas=0

# watch Argo CD put it back to 1 (because selfHeal=true)
kubectl -n guestbook get deploy/guestbook-ui -w
```

### b) Recreate managed resources

```bash
# delete a managed resource
kubectl -n guestbook delete svc guestbook-ui

# in Argo CD UI, hit "Sync" (or wait for auto-sync) → service comes back
kubectl -n guestbook get svc
```

---

You don’t have the **argocd** CLI on that machine. Two quick ways forward:

## Option 1 — Install the Argo CD CLI (Linux)

```bash
# check your arch
uname -m
# if x86_64:
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
# if aarch64/arm64, use:
# curl -sSL -o argocd-linux-amd64 \
#   https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
argocd version --client
```

Login (use your NodePort or port-forward URL; add `--insecure` for self-signed TLS):

```bash
argocd login <ARGOCD-HOST>:30443 --username admin --password '<your-pass>' --insecure
```

Now the commands you tried will work:

```bash
argocd app list
argocd app get guestbook
argocd app history guestbook
# force a resync
argocd app sync guestbook

```

## Option 2 — Verify without the CLI

```bash
# see what’s running
kubectl -n quiz-app get deploy,svc,pods -o wide

# describe the Argo CD Application CR
kubectl -n argocd get application quiz-app -o yaml | sed -n '1,120p'

# hit your app (NodePort from the Service)
curl -I http://<any-node-IP>:31080
```






