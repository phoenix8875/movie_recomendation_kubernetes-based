# 🎬 Movie Watchlist — Kubernetes (k3s) 3-Tier App

A three-tier movie app — **Nginx frontend → Node.js API → PostgreSQL** — deployed on a **k3s cluster** on EC2. Each tier runs in its own namespace, and only the frontend is exposed to the internet. The backend and database have no public ports and are reachable only from inside the cluster.

![App Preview](ss/preview_live.gif)

---

## Cluster Topology

```
AWS / EC2
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   ┌──────────────┐                                       │
│   │  EC2 Master  │  k3s server (control plane)           │
│   └──────┬───────┘                                       │
│          │ schedules pods                                │
│   ┌──────┴───────┐        ┌──────────────┐               │
│   │ EC2 Worker 1 │        │ EC2 Worker 2 │  k3s agents   │
│   │  runs pods   │        │  runs pods   │               │
│   └──────────────┘        └──────────────┘               │
│                                                          │
└──────────────────────────────────────────────────────────┘

The master decides WHERE pods run; the workers actually run them.
Your 3 tiers (frontend / backend / db) get scheduled across the workers.
```

---

## Application Architecture

Each tier lives in a separate **namespace** for isolation. Tiers talk to each other across namespaces through **ExternalName alias services** (explained below), so app code keeps using short names like `db` and `backend`.

```
                       [ CLIENT BROWSER ]
                              │
                              │  http://EC2_IP:30080
                              ▼
   ┌──────────── namespace: frontend-ns ─────────────────┐
   │                                                     │
   │   [ frontend Service ]  NodePort  80 ─► 30080       │
   │            │                                        │
   │            ▼                                        │
   │   [ frontend Pod ]  Nginx                           │
   │     • serves index.html / script.js (static)        │
   │     • reverse-proxies /api/ ─► http://backend:8000  │
   │            │                                        │
   │            ▼                                        │
   │   [ backend ]  ExternalName ─► backend.backend-ns   │
   └────────────────────────┬────────────────────────────┘
                            │  (in-cluster only)
                            ▼
   ┌──────────── namespace: backend-ns ───────────────────┐
   │                                                      │
   │   [ backend Service ]  ClusterIP  :8000              │
   │            │                                         │
   │            ▼                                         │
   │   [ backend Pod ]  Node.js API                       │
   │     • DATABASE_URL = ...@db:5432/moviedb             │
   │            │                                         │
   │            ▼                                         │
   │   [ db ]  ExternalName ─► db.db-ns                   │
   └────────────────────────┬─────────────────────────────┘
                            │  (in-cluster only)
                            ▼
   ┌──────────── namespace: db-ns ────────────────────────┐
   │                                                      │
   │   [ db Service ]  Headless (clusterIP: None)  :5432  │
   │            │                                         │
   │            ▼                                         │
   │   [ db-0 Pod ]  PostgreSQL  (managed by StatefulSet) │
   │            │  mounts /var/lib/postgresql/data        │
   │            ▼                                         │
   │   [ PVC: db-storage-db-0 ] ─► PV on node disk        │
   └──────────────────────────────────────────────────────┘

   PUBLIC    : only :30080 (frontend NodePort)
   INTERNAL  : backend :8000 and db :5432 — no public route
```

| Tier | Workload | Service type | Port | Public? |
|:---|:---|:---|:---|:---|
| Frontend (Nginx) | Deployment | NodePort | 80 → 30080 | ✅ Yes |
| Backend (Node.js) | Deployment | ClusterIP | 8000 | ❌ Internal |
| Database (Postgres) | **StatefulSet** | **Headless** | 5432 | ❌ Internal |

---

## How Connectivity Works (cross-namespace DNS)

In Kubernetes, a **bare hostname only resolves inside its own namespace**. The backend in `backend-ns` can't just say "connect to `db`" — the real `db` lives in `db-ns`. This is solved with **ExternalName services**:

- An ExternalName service is a pure **DNS alias** — no pod, no IP, just "this name means that name."
- You place the alias **in the namespace that needs it**, pointing at the **full name of the target**.
- The app keeps using a short hostname; the alias quietly forwards it across the namespace boundary.

```
ALIAS RESOLUTION
================

  backend pod asks DNS for:  "db"
        │
        ▼
  backend-ns/db  (ExternalName)
        │  aliases to
        ▼
  db.db-ns.svc.cluster.local
        │  resolves to
        ▼
  db-ns/db  (headless Service)
        │  routes to
        ▼
  db-0 pod  :5432
```

- `frontend-ns/backend` → `backend.backend-ns.svc.cluster.local`  (Nginx → API)
- `backend-ns/db` → `db.db-ns.svc.cluster.local`  (API → database)

**Why headless for the DB?** The `db` service uses `clusterIP: None`. A normal service load-balances behind one virtual IP; a headless service returns the pod's address directly. Since there's exactly one stable DB pod (`db-0`), you want to reach *that pod*, not balance across replicas — which is exactly the pattern a StatefulSet expects.

---

## Request Lifecycle (one "Add Movie" click)

```
USER REQUEST FLOW
=================

  [Browser]  click "Add Movie"
      │  POST /api/movies   (relative path → same origin :30080)
      ▼
  [Nginx / frontend pod]
      │  matches "location /api/"
      │  proxy_pass ─► http://backend:8000   (via frontend-ns/backend alias)
      ▼
  [backend Service :8000] ─► [backend pod / Node.js]
      │  reads DATABASE_URL
      │  connects to db:5432   (via backend-ns/db alias → db-0)
      ▼
  [db headless svc] ─► [db-0 / PostgreSQL]
      │  INSERT INTO movies ...
      ▼
  [PVC db-storage-db-0]  data written to node disk
      │
      ▼
  Response travels back: Postgres → backend → Nginx → browser
  [User sees the updated list] ✅
```

**Key isolation facts:**
- 🔓 Only `:30080` is open to the internet (frontend NodePort).
- 🔒 Backend `:8000` and db `:5432` are ClusterIP/headless — invisible to outside port scans.
- 🧱 The only bridges between tiers are the two ExternalName aliases.

---

## Project Structure

```
.
├── namespaces.yaml                   # creates frontend-ns, backend-ns, db-ns
├── db/
│   ├── 02-secret.yaml                # db-credentials: POSTGRES_USER / PASSWORD / DB
│   ├── statefulset.yaml              # Postgres StatefulSet → pod db-0 + its own PVC
│   └── service.yaml                  # headless db Service (clusterIP: None)
├── backend/
│   ├── 01-db-alias-service.yaml      # ExternalName: db → db.db-ns
│   ├── 02-secret.yaml                # DATABASE_URL, TMDB_API_KEY
│   ├── 03-deployment.yaml            # Node.js API (stateless), listens on 8000
│   └── 04-service.yaml               # backend ClusterIP Service :8000
└── frontend/
    ├── 01-backend-alias-service.yaml # ExternalName: backend → backend.backend-ns
    ├── 02-deployment.yaml            # Nginx pod (static files + reverse proxy)
    └── 03-service.yaml               # frontend NodePort Service :30080
```

### What each YAML does

**Root**
- `namespaces.yaml` — declares the three namespaces. Applied first so everything else has a home.

**Database (`db/`)**
- `02-secret.yaml` — `db-credentials` secret with `POSTGRES_USER` (`movieuser`), `POSTGRES_PASSWORD`, `POSTGRES_DB` (`moviedb`). Pulled in by the StatefulSet via `envFrom`, so Postgres initializes itself on first boot.
- `statefulset.yaml` — the Postgres workload. `serviceName: db` ties it to the headless service; `image: postgres:16-alpine`; `envFrom` the secret; and `volumeClaimTemplates` auto-creates the PVC `db-storage-db-0`, mounted at `/var/lib/postgresql/data` with `subPath: postgres-data`.
- `service.yaml` — the `db` service with `clusterIP: None`, selecting `app: postgres` on port `5432`.

**Backend (`backend/`)**
- `01-db-alias-service.yaml` — ExternalName so the API can reach `db` across namespaces.
- `02-secret.yaml` — holds `DATABASE_URL` (connection string using host `db`) and `TMDB_API_KEY`, injected as env vars.
- `03-deployment.yaml` — the Node.js API. A plain **Deployment** because it's stateless (no data of its own). Listens on `8000`.
- `04-service.yaml` — ClusterIP `:8000`. No NodePort, so it's never publicly exposed.

**Frontend (`frontend/`)**
- `01-backend-alias-service.yaml` — ExternalName so Nginx can proxy to `backend` across namespaces.
- `02-deployment.yaml` — the Nginx pod that serves static files and reverse-proxies `/api/`.
- `03-service.yaml` — NodePort `:30080`, the single public door into the app.

---

## Why a StatefulSet (and a PVC) for the Database

The frontend and backend are **Deployments** because they're stateless — any pod is interchangeable and can be killed/replaced freely. The database is different: it owns data that must survive restarts.

```
DEPLOYMENT vs STATEFULSET (for the DB)
======================================

  Deployment                      StatefulSet
  ----------                      -----------
  random pod name                 stable name: db-0
  db-deployment-7fbd-cf75z        db-0
  shared/awkward storage          OWN PVC per pod (db-storage-db-0)
  rolling update: new pod         replaces pods one at a time,
  starts before old dies          in order — no volume clash
  → both fight over the same
    ReadWriteOnce volume
```

- **A StatefulSet gives the DB a stable identity** (`db-0`) and, via `volumeClaimTemplates`, its **own dedicated PVC** that re-attaches to the same pod across restarts.
- **It avoids a Deployment pitfall:** a rolling update would try to start a new pod before terminating the old one, and both would fight over the same `ReadWriteOnce` disk and hang.
- **"Stateful" ≠ "StatefulSet":** what actually persists data is the **PVC** — storage that lives *outside* the pod on the node's disk. The StatefulSet is just the controller that manages stable identity + per-pod PVCs.

```
WHY A PVC = DATA SURVIVES
=========================

  No PVC:  data inside pod ─► pod dies ─► ❌ data gone
  With PVC: data on node disk
            db-0 dies ─► StatefulSet recreates db-0
            ─► same PVC re-attached ─► ✅ data still there
```

> The PVC here is **100Mi** via the k3s `local-path` StorageClass — small on purpose for a learning project. PVCs are easy to grow but hard to shrink, so size up if you ever expect real data.

---

## Deploy

Apply in dependency order: namespaces → database → backend → frontend.

```bash
kubectl apply -f namespaces.yaml
kubectl apply -f db/
kubectl apply -f backend/
kubectl apply -f frontend/
```

Then open `http://<EC2_PUBLIC_IP>:30080`.

---

## Verify

```bash
kubectl get pods -A                       # db-0 should be 1/1 Running
kubectl get pvc -n db-ns                  # db-storage-db-0 should be Bound
kubectl get svc db -n db-ns               # CLUSTER-IP should read None
kubectl exec -it db-0 -n db-ns -- psql -U movieuser -d moviedb -c "\dt"
```

**Prove persistence (the whole point of the StatefulSet):**

```bash
# add a movie in the browser, then:
kubectl delete pod db-0 -n db-ns          # recreated as db-0 with the same volume
kubectl get pods -n db-ns -w              # wait for 1/1, reload browser — movie is still there ✅
```

---

## AWS Security Group

| Port | Source | Purpose |
|:---|:---|:---|
| 22 | Your IP | SSH |
| 30080 | 0.0.0.0/0 | App (frontend NodePort) |
| 6443 / 10250 / 8472 | Cluster SG only | k3s API / kubelet / Flannel — node-to-node, not public |

Backend (`8000`) and database (`5432`) expose **no public ports**.

---

## Troubleshooting

```bash
kubectl logs db-0 -n db-ns
kubectl logs deploy/backend-deployment -n backend-ns
kubectl exec -it deploy/backend-deployment -n backend-ns -- printenv | grep -i db
```

- **Backend can't reach DB** → check the `db` alias in `backend-ns` points to `db.db-ns.svc.cluster.local`, and that `DATABASE_URL` uses host `db`.
- **`db` shows a real CLUSTER-IP instead of `None`** → it wasn't recreated as headless; `clusterIP` is immutable, so delete and re-apply the service.
- **Empty list after a reset** → expected; the `movies` table auto-recreates on the first backend connection (`CREATE TABLE IF NOT EXISTS`).
