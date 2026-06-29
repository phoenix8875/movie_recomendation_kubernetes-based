# Movie Watchlist — Helm Chart Explained

A walk-through of how this 3-tier app (Nginx frontend → FastAPI backend → Postgres db)
was turned into a Helm chart, and how every file connects to every other.

---

## 1. What Helm Is (in brief)

Helm is a **templating + packaging layer on top of Kubernetes**. It does not replace
your Kubernetes knowledge — it wraps it. Under the hood it still produces the same
Deployments, Services, Secrets, and StatefulSets you would write by hand.

The core idea:

```
  YOUR YAML with blanks        +        ONE answers file        =     PLAIN K8S YAML
  (templates/*.yaml)                    (values.yaml)                 (what gets applied)
  image: {{ .Values...}}                backend.image.tag: "2.0"      image: raj8875/...:2.0
```

Helm **renders** (templates + values → plain YAML), then **applies** the result to the
cluster and remembers it as a versioned *release* you can upgrade or roll back.

Three terms to keep straight:

| Term | Meaning |
|:---|:---|
| **Chart** | The package — the folder of templates + values. The "recipe." |
| **Release** | One installed instance of a chart on a cluster. The "cooked meal." |
| **Revision** | A numbered snapshot of a release. Each `install`/`upgrade` makes a new one. |

Why it beats `kubectl apply -f`:
- One command (`helm install`) replaces the four ordered `kubectl apply` commands.
- One file (`values.yaml`) holds every knob — no hunting through 11 files to change a tag.
- Every deploy is reversible: `helm rollback` undoes a bad change instantly.
- The same chart spins up staging/prod variants via `--set` or a second values file.

---

## 2. Project Structure

```
movie-watchlist/
├── Chart.yaml            # metadata: chart name + two version numbers
├── values.yaml           # THE ANSWERS SHEET — every knob lives here
└── templates/            # your old manifests, with {{ }} blanks
    │
    ├── namespaces.yaml              # creates the 3 namespaces (loops over values)
    │
    ├── db-secret.yaml               # ─┐
    ├── db-statefulset.yaml          #  ├─ DATABASE TIER  → namespace: db-ns-helm
    ├── db-service.yaml              # ─┘
    │
    ├── backend-db-alias.yaml        # ─┐
    ├── backend-secret.yaml          #  │
    ├── backend-deployment.yaml      #  ├─ BACKEND TIER   → namespace: backend-ns-helm
    ├── backend-service.yaml         # ─┘
    │
    ├── frontend-backend-alias.yaml  # ─┐
    ├── frontend-deployment.yaml     #  ├─ FRONTEND TIER  → namespace: frontend-ns-helm
    └── frontend-service.yaml        # ─┘
```

Two files are new compared to the original repo: `Chart.yaml` (metadata) and
`values.yaml` (the answers). Everything in `templates/` is a near-copy of the original
manifests, with hardcoded literals swapped for `{{ .Values.* }}` references.

---

## 3. The Heart of It — `values.yaml`

Every template reads from this one file. Nothing is hardcoded in the templates anymore;
it all flows from here.

```yaml
namespaces:
  db: db-ns-helm
  backend: backend-ns-helm
  frontend: frontend-ns-helm

db:
  image: postgres:16-alpine
  replicas: 1
  storage:
    size: 100Mi
    className: local-path
  credentials:
    user: movieuser
    password: moviepass
    database: moviedb

backend:
  image:
    repository: raj8875/movie-recommender-backend
    tag: "2.0"
  replicas: 1
  port: 8000
  tmdbApiKey: "..."        # the ONLY secret you fill in by hand

frontend:
  image:
    repository: raj8875/movie-recommender-frontend
    tag: "2.0"
  replicas: 1
  containerPort: 80
  nodePort: 30080
```

A template reaches a value by its **dotted path**, e.g. `{{ .Values.db.storage.size }}`
walks `db → storage → size` and pulls `100Mi`.

---

## 4. How Each File Is Wired

### `Chart.yaml`
Pure metadata. `version` tracks the *chart* (the packaging); `appVersion` tracks the
*app* (your image tags). They move independently — fix a template typo and bump only
`version`, while `appVersion` stays at `2.0`.

### `templates/namespaces.yaml`
Loops over the `namespaces` map instead of hardcoding three blocks.

```
  .Values.namespaces                      rendered output
  ──────────────────                      ───────────────
  db: db-ns-helm          ── range ──►    kind: Namespace  name: db-ns-helm
  backend: backend-ns-helm                kind: Namespace  name: backend-ns-helm
  frontend: frontend-ns-helm              kind: Namespace  name: frontend-ns-helm
```

`{{- range $key, $name := .Values.namespaces }}` walks the map; `$name` is each value.

---

### Database tier

**`db-secret.yaml`** — builds the `db-credentials` Secret. Pulls three values:

```
  .Values.db.credentials.user      ──►  POSTGRES_USER: "movieuser"
  .Values.db.credentials.password  ──►  POSTGRES_PASSWORD: "moviepass"
  .Values.db.credentials.database  ──►  POSTGRES_DB: "moviedb"
```

The `| quote` pipeline wraps each value in `""` so YAML never misreads it.

**`db-statefulset.yaml`** — the Postgres workload. Reads:

```
  .Values.namespaces.db        ──►  namespace: db-ns-helm
  .Values.db.replicas          ──►  replicas: 1
  .Values.db.image             ──►  image: postgres:16-alpine
  .Values.db.storage.className ──►  storageClassName: local-path
  .Values.db.storage.size      ──►  storage: 100Mi
  (envFrom) ──────────────────────► reads the db-credentials Secret above
```

`serviceName: db` ties it to the headless Service; `volumeClaimTemplates` auto-creates
the PVC `db-storage-db-0`.

**`db-service.yaml`** — the headless (`clusterIP: None`) Service. Reads only
`.Values.namespaces.db`. Selects `app: postgres` (the label the StatefulSet sets on its
pod), exposing port 5432.

```
  db-statefulset (label app: postgres)  ◄── selector ──  db-service (app: postgres)
```

---

### Backend tier

**`backend-db-alias.yaml`** — an ExternalName Service named `db`, but living in
`backend-ns-helm`. This is the cross-namespace bridge. It **builds** its target DNS
string from a value instead of hardcoding it:

```
  printf "db.%s.svc.cluster.local"  .Values.namespaces.db
                  │                          │
                  └──────────────────────────┘
                              ▼
            db.db-ns-helm.svc.cluster.local
```

So renaming the db namespace in `values.yaml` automatically updates this alias — it
can't drift out of sync.

**`backend-secret.yaml`** — builds `backend-secrets`. Two keys, one **assembled** from
db values, one filled by you:

```
  DATABASE_URL  = printf "postgresql://%s:%s@db:5432/%s"
                         db.credentials.user
                         db.credentials.password
                         db.credentials.database
                ──► postgresql://movieuser:moviepass@db:5432/moviedb

  TMDB_API_KEY  = .Values.backend.tmdbApiKey   ◄── the one value you set by hand
```

Note the host inside `DATABASE_URL` is just `db` — which resolves via the alias above.

**`backend-deployment.yaml`** — the stateless Node/FastAPI pod. Reads:

```
  .Values.namespaces.backend                              ──► namespace
  .Values.backend.replicas                                ──► replicas
  printf "%s:%s" backend.image.repository .image.tag      ──► image: raj8875/...:2.0
  .Values.backend.port                                    ──► containerPort: 8000
  (envFrom) ─────────────────────────────────────────────────► reads backend-secrets
```

**`backend-service.yaml`** — ClusterIP (internal only). Reads `.Values.namespaces.backend`
and `.Values.backend.port` (used for both `port` and `targetPort`). Selects `app: backend`.

```
  backend-deployment (label app: backend)  ◄── selector ──  backend-service (app: backend)
```

---

### Frontend tier

**`frontend-backend-alias.yaml`** — ExternalName Service named `backend`, in
`frontend-ns-helm`. Same printf trick, one tier over:

```
  printf "backend.%s.svc.cluster.local"  .Values.namespaces.backend
                ──► backend.backend-ns-helm.svc.cluster.local
```

Nginx's config has `proxy_pass http://backend:8000` baked in; this alias makes the bare
name `backend` resolve from inside the frontend namespace.

**`frontend-deployment.yaml`** — the Nginx pod. Reads namespace, replicas, the assembled
image string, and `.Values.frontend.containerPort` (80).

**`frontend-service.yaml`** — NodePort, the only public door. Reads:

```
  .Values.namespaces.frontend     ──► namespace
  .Values.frontend.containerPort  ──► targetPort: 80
  .Values.frontend.nodePort       ──► nodePort: 30080   ◄── the public port
```

---

## 5. The Whole Value Flow, End to End

```
                          values.yaml
                               │
        ┌──────────────────────┼───────────────────────┐
        │                      │                        │
        ▼                      ▼                        ▼
   namespaces.*            db.*                  backend.* / frontend.*
        │                    │                          │
        │            ┌───────┴────────┐         ┌────────┴────────┐
        ▼            ▼                ▼         ▼                 ▼
  namespaces    db-secret       db-statefulset  backend-*        frontend-*
   (3 ns)           │           db-service       (4 files)        (3 files)
                    └──envFrom──────┘                 │                │
                                                      │                │
                                          DATABASE_URL assembled       │
                                          from db.credentials.*        │
                                                      │                │
                          printf builds cross-ns DNS  │                │
                          db.db-ns-helm.svc...  ◄──────┘                │
                          backend.backend-ns-helm.svc...  ◄─────────────┘
```

And the request path at runtime (unchanged from the original app — Helm only changed
how it's deployed, not how it behaves):

```
  Browser ──:30080──► frontend NodePort Service
                          │
                          ▼
                      Nginx pod ──proxy /api/──► "backend:8000"
                                                      │  (resolved by frontend-ns alias)
                                                      ▼
                                            backend ClusterIP Service ──► FastAPI pod
                                                                              │
                                                              connects to "db:5432"
                                                              (resolved by backend-ns alias)
                                                                              ▼
                                                              db headless Service ──► db-0
                                                                                       │
                                                                          writes to PVC on node disk
```

---

## 6. Everyday Commands

```bash
# render to plain YAML, no cluster touched — your debugging tool
helm template movie-watchlist .

# install / change / undo
helm install  movie-watchlist .
helm upgrade  movie-watchlist . --set backend.replicas=2
helm rollback movie-watchlist 1
helm uninstall movie-watchlist

# inspect
helm list                 # current releases + revision number
helm history movie-watchlist   # full audit trail of every revision
```

The two you lean on most while learning: `helm template` (see rendered output) and
`helm install --dry-run --debug` (validate before committing to the cluster).

---

## 7. One Honest Caveat

The two `ExternalName` alias services exist *only* because the app images have hostnames
(`db`, `backend`) baked in, and a bare hostname resolves only within its own namespace.
They are a bridge, not a best practice. A simpler design would run all tiers in one
namespace (no aliases needed) or make the hostnames configurable in the app. The current
setup keeps the original architecture intact — worth knowing the tradeoff if asked.
