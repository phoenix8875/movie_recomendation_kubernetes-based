# 🎬 Movie Recommender — Kubernetes Deployment (k3s)

This is the Kubernetes phase of the project: the same application that
previously ran via Docker Compose, now deployed on a **k3s cluster** —
split into three namespaces, one per tier, with each piece wired together
using core Kubernetes objects.

This README focuses entirely on the **Kubernetes networking and deployment
story** — namespaces, Services, cross-namespace DNS, and how traffic actually
flows from a browser to the database and back. The app itself is just the
payload moving through this pipeline.

---

## 📺 Demo

<!-- TODO: replace with a real demo gif/screenshot of the app running via the cluster's NodePort -->
![App demo](docs/k8s-demo.gif)

---

## 🧱 The cluster

```
                         k3s CLUSTER (3 nodes)
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │   master-k3s              worker-k3s-1         worker-k3s-2  │
   │   (control-plane)         (worker)             (worker)      │
   │   ┌────────────────┐      ┌──────────────┐    ┌────────────┐ │
   │   │ coredns        │      │ backend pod  │    │ db pod     │ │
   │   │ traefik        │      │              │    │ frontend   │ │
   │   │ metrics-server │      │              │    │   pod      │ │
   │   │ local-path-    │      │              │    │            │ │
   │   │  provisioner   │      │              │    │            │ │
   │   └────────────────┘      └──────────────┘    └────────────┘ │
   └──────────────────────────────────────────────────────────────┘
```

- **One control-plane node** (`master-k3s`) runs the cluster's own internal
  machinery — DNS (`coredns`), the built-in load balancer (`traefik`), and a
  few other system components. Our application pods did **not** land here —
  the scheduler placed them on the worker nodes instead.
- **Two worker nodes** run the actual application pods. Which pod lands on
  which worker is decided automatically by Kubernetes' scheduler — we never
  specified that ourselves.
- **`kubectl get pods -A -o wide`** is how you see this for real — the `NODE`
  column shows exactly where each pod is running.

---

## 📁 Three namespaces, one per tier

```
  db-ns            backend-ns            frontend-ns
  ────────         ─────────────         ─────────────
  Postgres pod     FastAPI pod           nginx pod
  PVC (storage)    Secret (JWT/TMDB key) Service (NodePort, public)
  Secret (creds)   Service (internal)    alias Service -> backend
  Service          alias Service -> db
  (internal)
```

A **namespace** is a way to partition one cluster into separate sections —
same physical nodes, same cluster, just logically grouped. We used three: one
per tier, mirroring how a real company might split database, backend, and
frontend ownership across different teams.

**What namespaces give you:**
- Organization — related resources are grouped and easy to find
  (`kubectl get all -n backend-ns` shows only backend things).
- A boundary for **RBAC** (Role-Based Access Control) — you can grant a
  user/service-account permissions scoped to just one namespace, e.g. "the db
  team can only act inside `db-ns`."
- A boundary for resource quotas — limiting how much CPU/memory a namespace
  can consume in total.

**What namespaces do *not* give you, by default:**
- **Network isolation.** A pod in `frontend-ns` can still send a network
  request directly to a pod's IP address in `db-ns` unless something
  explicitly blocks it. That "something" is a separate object called a
  **NetworkPolicy**, which we deliberately did not add in this project. So
  right now, the separation is organizational/administrative, not a network
  firewall between tiers. Worth being precise about this distinction — it's a
  common point of confusion.

---

## 🚧 The core problem: hardcoded hostnames across namespaces

Both the frontend and backend images have a hostname baked in, unchanged from
the Docker Compose setup:

- **Frontend's nginx config** says: forward API requests to `backend:8000`.
- **Backend's database connection string** says: connect to `db:5432`.

In Docker Compose, those plain names just worked, because Compose puts
everything on one shared network. In Kubernetes, a plain name like `backend`
**only resolves automatically for things inside the same namespace.** Since
each tier now lives in its *own* namespace, those hardcoded names would fail:

```
  frontend pod (in frontend-ns)  --- "backend" ---> ??? NOT FOUND
  backend pod  (in backend-ns)   --- "db" -------->  ??? NOT FOUND
```

We deliberately did **not** rebuild the images or edit any code to fix this —
that would mean rebuilding/pushing new images every time the cluster topology
changes, which doesn't scale. Instead, we fixed it entirely at the Kubernetes
configuration level.

---

## 🔑 The fix: ExternalName alias Services

Kubernetes lets you create a Service whose entire job is to be a **DNS
alias** — "whenever something asks for this name, silently redirect to a
different name instead." This object type is called `ExternalName`.

We created one tiny alias Service inside each namespace that needed to reach
across to another:

```
  INSIDE backend-ns:                    INSIDE frontend-ns:
  ┌─────────────────────────┐           ┌────────────────────────────────┐
  │  Service named "db"     │           │  Service named "backend"       │
  │  type: ExternalName     │           │  type: ExternalName            │
  │  points to:             │           │  points to:                    │
  │  db.db-ns.svc.cluster   │           │  backend.backend-ns.svc.cluster│
  │  .local                 │           │  .local                        │
  └─────────────────────────┘           └────────────────────────────────┘
         │                                         │
         │  backend pod asks for "db"              │  frontend's nginx asks
         │  -> gets redirected to the              │  for "backend"
         v  REAL db Service in db-ns               v  -> gets redirected to the
  ┌─────────────────┐                        REAL backend Service in backend-ns
  │ db Service      │                       ┌──────────────────┐
  │ (in db-ns)      │                       │ backend Service  │
  │ -> Postgres pod │                       │ (in backend-ns)  │
  └─────────────────┘                       │ -> FastAPI pod   │
                                            └──────────────────┘
```

**In plain terms:** the backend pod still thinks it's talking to something
called `db`, exactly like before. It has no idea that name is secretly an
alias being redirected to a completely different namespace. Same story for
the frontend talking to `backend`. **Zero code changes, zero image rebuilds —
the fix lives entirely in two small YAML files.**

This is the single most important trick in this deployment — it's what makes
namespace separation actually *work* without breaking the existing images.

---

## 🔁 Request flow — browser to database and back

The user signs up. Traffic travels DOWN through the three namespaces to the
database, then the response travels back UP the exact same path.

```
   +-------------+
   |   BROWSER   |   http://<ec2-ip>:30080/auth/signup
   +------+------+
          |
          v
====================== frontend-ns ==============================
|                                                               |
|   +------------------------+                                  |
|   | frontend Service       |   type: NodePort (port 30080)    |
|   | (the public door)      |                                  |
|   +-----------+------------+                                  |
|               |                                               |
|               v                                               |
|   +------------------------+                                  |
|   | frontend POD (nginx)   |   rule: /auth/ -> backend:8000   |
|   +-----------+------------+                                  |
|               |  asks DNS for "backend"                       |
|               v                                               |
|   +------------------------+                                  |
|   | "backend" ALIAS        |   type: ExternalName             |
|   | (DNS redirect only)    |   -> backend.backend-ns.svc...   |
|   +-----------+------------+                                  |
|               |                                               |
================|===============================================
                |  (crosses the namespace boundary)
                v
====================== backend-ns ===============================
|               |                                               |
|               v                                               |
|   +------------------------+                                  |
|   | backend Service        |   type: ClusterIP (internal)     |
|   +-----------+------------+                                  |
|               |                                               |
|               v                                               |
|   +------------------------+                                  |
|   | backend POD (FastAPI)  |   conn: ...@db:5432              |
|   +-----------+------------+                                  |
|               |  asks DNS for "db"                            |
|               v                                               |
|   +------------------------+                                  |
|   | "db" ALIAS             |   type: ExternalName             |
|   | (DNS redirect only)    |   -> db.db-ns.svc...             |
|   +-----------+------------+                                  |
|               |                                               |
================|===============================================
                |  (crosses the namespace boundary)
                v
======================== db-ns ==================================
|               |                                               |
|               v                                               |
|   +------------------------+                                  |
|   | db Service             |   type: ClusterIP (internal)     |
|   +-----------+------------+                                  |
|               |                                               |
|               v                                               |
|   +------------------------+                                  |
|   | Postgres POD           |   reads/writes PVC storage       |
|   +------------------------+                                  |
|                                                               |
=================================================================

   Response travels back UP the same path, in reverse:
   Postgres -> db Svc -> backend POD -> backend Svc ->
   nginx -> frontend Svc -> browser
```

**Step by step, what happens at each hop:**

- **Browser -> frontend Service.** The user hits `http://<ec2-ip>:30080`.
  That port is opened on every cluster node by the frontend's **NodePort**
  Service — the single public entry point. Nothing else in the cluster is
  reachable from outside.

- **frontend Service -> frontend pod (nginx).** The Service forwards the
  request to the nginx pod. nginx checks the URL path: anything starting with
  `/auth/`, `/movies`, or `/watchlist` is an API call, so its config forwards
  it to `http://backend:8000`.

- **nginx asks DNS for "backend".** The catch: nginx is in `frontend-ns`, but
  the real backend is in `backend-ns`, so the plain name `backend` wouldn't
  normally resolve. It works only because we placed a **`backend`
  ExternalName alias** inside `frontend-ns`. This alias is not a server — it's
  a pure DNS redirect meaning "`backend` really points to
  `backend.backend-ns.svc.cluster.local`."

- **Alias crosses the boundary -> backend Service.** The redirected name now
  points at the real **backend ClusterIP Service** in `backend-ns`. ClusterIP
  means internal-only — correct, since only nginx (never the browser) should
  reach it.

- **backend Service -> backend pod (FastAPI).** The Service routes to the
  FastAPI pod, which processes the signup: validates input, hashes the
  password. To store it, it needs the database — and its connection string
  uses the host `db`.

- **FastAPI asks DNS for "db".** Same trick, one tier deeper. The backend pod
  is in `backend-ns`, the database in `db-ns`, so plain `db` wouldn't resolve
  — except we placed a **`db` ExternalName alias** inside `backend-ns`,
  redirecting to `db.db-ns.svc.cluster.local`.

- **Alias crosses the boundary -> db Service -> Postgres pod.** The redirect
  lands on the real **db ClusterIP Service** in `db-ns`, which routes to the
  Postgres pod. Postgres writes the new user row to its **PVC-backed
  storage**, so the data survives even if the pod is restarted later.

- **Response travels back up the identical chain, in reverse** — Postgres ->
  db Service -> backend pod -> backend Service -> nginx -> frontend Service ->
  browser. The account is created, and from the browser's point of view it all
  came from one address on port 30080.

**The one-line takeaway:** traffic crosses two namespace boundaries (and
possibly two different physical worker nodes), yet neither nginx nor FastAPI
knows any of that — each just talks to a plain name (`backend`, `db`), and the
two ExternalName aliases quietly handle the cross-namespace redirect with zero
code changes.

---

## ⚙️ What we built, folder by folder

```
k8s/
├── 00-namespaces.yaml          # creates db-ns, backend-ns, frontend-ns
│
├── db/
│   ├── 02-secret.yaml          # Postgres username/password/db name
│   ├── 03-pvc.yaml             # persistent storage request
│   ├── 04-deployment.yaml      # the Postgres pod
│   └── 05-service.yaml         # gives Postgres the stable name "db"
│
├── backend/
│   ├── 01-db-alias-service.yaml   # ExternalName: makes "db" resolve here
│   ├── 02-secret.yaml              # JWT key, TMDB key, DATABASE_URL
│   ├── 03-deployment.yaml          # the FastAPI pod
│   └── 04-service.yaml             # gives backend the stable name "backend"
│
└── frontend/
    ├── 01-backend-alias-service.yaml  # ExternalName: makes "backend" resolve here
    ├── 02-deployment.yaml             # the nginx pod
    └── 03-service.yaml                # NodePort — the one externally reachable piece
```

### `db/` — what each file does

- **Secret** — stores `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` as
  key-value pairs. Kubernetes stores these **base64-encoded**, not encrypted
  — that's an important distinction. Encoding is not encryption; anyone with
  permission to read the Secret object can trivially decode it. Real
  protection comes from controlling *who* can read Secrets (RBAC) and, in
  production clusters, enabling encryption-at-rest.
- **PVC (PersistentVolumeClaim)** — a request for storage that survives pod
  restarts. Pods are disposable; if the Postgres pod is destroyed and
  recreated, a fresh pod with no PVC would start with a completely empty,
  brand-new database. The PVC is what makes data durable across that churn.
  k3s automatically provisions the actual storage behind this request.
- **Deployment** — describes the Postgres pod: which image, which port, how
  it gets its credentials (`envFrom` pulls the whole Secret in as environment
  variables), and where the PVC gets mounted inside the container.
- **Service** — gives the Postgres pod a stable name (`db`) and stable
  virtual IP, so that name keeps working even if the underlying pod is
  replaced.

### `backend/` — what each file does

- **ExternalName alias Service** — the fix described above. Makes `db`
  resolve correctly from inside `backend-ns`.
- **Secret** — JWT signing key, TMDB API key, and the full database
  connection string (which still just says `db` as the host — the alias
  handles the rest).
- **Deployment** — the FastAPI pod. No persistent volume needed here, because
  the backend itself stores no data — everything it needs lives in Postgres.
  That statelessness is also why this tier is the easiest one to scale to
  multiple replicas later.
- **Service** — gives the backend pod the stable name `backend`, which the
  frontend's alias (next folder) points to.

### `frontend/` — what each file does

- **ExternalName alias Service** — makes `backend` resolve correctly from
  inside `frontend-ns`, fixing nginx's hardcoded `proxy_pass`.
- **Deployment** — the nginx pod. No secrets or special config — it just
  serves static files and proxies API calls.
- **Service (NodePort)** — the only Service in the entire project that needs
  to be reachable from *outside* the cluster. Real users hit this one.

---

## 🌐 ClusterIP vs NodePort vs ExternalName — why each was chosen

| Service type | Used for | Reachable from |
|---|---|---|
| **ClusterIP** (default) | `db`, `backend` | Only inside the cluster |
| **NodePort** | `frontend` | Outside the cluster, on a specific port |
| **ExternalName** | the two alias Services | Anywhere inside the cluster — it's a DNS redirect, not a real network endpoint |

- **ClusterIP** is the right choice whenever nothing outside the cluster
  should ever connect directly — true for both Postgres and the backend API.
  The browser should never talk to either of these directly; it only ever
  talks to nginx.
- **NodePort** opens a specific port (we used `30080`, inside Kubernetes'
  default NodePort range of 30000–32767) on *every* node in the cluster, and
  forwards traffic from that port into the target Service. It's the simplest
  way to expose something externally on a small/learning cluster. A
  production setup would more likely use a `LoadBalancer` type Service
  (provisioning a real cloud load balancer) or an `Ingress` — both are
  natural next steps beyond this project.
- **ExternalName** isn't really a network endpoint at all — it's pure DNS
  redirection, which is exactly why it was the right tool for solving the
  cross-namespace hostname problem with zero code changes.

---

## 🧩 Key concepts glossary (plain language)

- **Namespace** — a logical folder inside one cluster, used to group and
  separate resources administratively (not a network firewall by itself).
- **Pod** — the smallest deployable unit in Kubernetes; in this project, one
  container per pod.
- **Deployment** — a controller that keeps a specified number of pod copies
  running, replacing them automatically if they crash or are deleted.
- **Service** — a stable name + virtual IP that routes to whichever pod(s)
  currently match its label selector, regardless of how many times those
  pods get replaced underneath it.
- **Secret** — a Kubernetes object for storing sensitive config, encoded
  (not encrypted) in base64.
- **PersistentVolumeClaim (PVC)** — a request for storage that outlives any
  individual pod.
- **Scheduler** — the cluster component that decides which node a new pod
  runs on, based on available resources and any constraints you've set (we
  set none, so it chose freely between the two worker nodes).
- **Taint** — a marker on a node that repels pods unless they explicitly
  "tolerate" it. Standard Kubernetes taints control-plane nodes by default so
  application pods avoid them; k3s does not do this automatically.
- **DaemonSet** — a controller that runs exactly one copy of a pod on *every*
  node, no exceptions (you can see this cluster's own `svclb-traefik` pods
  doing exactly this, one per node, used internally by k3s's load balancer).

---

## 🚀 Deploying this yourself

Apply in this order — namespaces first, then each tier (db before backend,
since backend's alias depends on db already existing as a concept; backend
before frontend, same reasoning):

```bash
kubectl apply -f 00-namespaces.yaml

kubectl apply -f db/02-secret.yaml
kubectl apply -f db/03-pvc.yaml
kubectl apply -f db/04-deployment.yaml
kubectl apply -f db/05-service.yaml

kubectl apply -f backend/01-db-alias-service.yaml
kubectl apply -f backend/02-secret.yaml
kubectl apply -f backend/03-deployment.yaml
kubectl apply -f backend/04-service.yaml

kubectl apply -f frontend/01-backend-alias-service.yaml
kubectl apply -f frontend/02-deployment.yaml
kubectl apply -f frontend/03-service.yaml
```

**Verify each tier as you go:**
```bash
kubectl get all -n db-ns
kubectl get all -n backend-ns
kubectl get all -n frontend-ns
```

**Open the NodePort in your EC2 Security Group** (inbound rule, port `30080`,
source `0.0.0.0/0`), then visit:

```
http://<ec2-public-ip>:30080
```

---

## 🔍 Useful diagnostic commands

```bash
# See every pod across the whole cluster, and which node each landed on
kubectl get pods -A -o wide

# Check whether the master node is tainted against regular pods
kubectl describe node master-k3s | grep -A 3 Taints

# Confirm a cross-namespace alias actually resolves, from inside a pod
kubectl exec -n backend-ns deployment/backend-deployment -- getent hosts db

# Watch a pod's logs live
kubectl logs -n backend-ns deployment/backend-deployment -f

# If something's stuck, get the full event history for the pod
kubectl describe pod -n backend-ns <pod-name>
```

---

## 🛠️ Tech stack

- **Orchestration:** Kubernetes (k3s), 3-node cluster (1 control-plane, 2 workers)
- **Networking:** Kubernetes Services (ClusterIP, NodePort, ExternalName),
  cluster-internal DNS
- **Containers:** the same Docker images built in the Docker Compose phase
- **Storage:** PersistentVolumeClaim, backed by k3s's local-path-provisioner
- **Secrets:** Kubernetes Secrets (base64-encoded config)
