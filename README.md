# 🎬 Movie Recommender — Containerized 3-Tier App on AWS

A three-tier application (frontend, backend, database) packaged with Docker,
wired together with Docker Compose, and deployed on AWS EC2.

The app itself recommends movies — but this README focuses on **how the pieces
talk to each other**: container networking, the nginx reverse proxy, traffic
flow, and why the same images run anywhere with no reconfiguration. The movie
logic is just the payload moving through that pipeline.

---

## 📺 Demo

<!-- TODO: replace with a real demo gif -->
![App demo](docs/demo.gif)

---

## 🗺️ Architecture at a glance

Three containers on one private network. **Only the frontend is reachable from
the internet.** Everything else is internal.

```
                          [ USER'S BROWSER ]
                                  |
                                  |  all traffic on port 80
                                  v
                  ======= AWS Security Group =======
                  ||      only port 80 is open      ||
                  ==================================
                                  |
   ┌──────────────────────────────┼───────────────────────────────┐
   │   PRIVATE DOCKER NETWORK      │  (containers talk by name)     │
   │                               v                                │
   │        ┌────────────────────────────────────────────┐         │
   │        │  frontend  (nginx)         host port 80 ◄────┼──── public
   │        │  • serves the web page                       │         │
   │        │  • reverse-proxies /api traffic inward       │         │
   │        └───────────────────────┬────────────────────┘         │
   │                                │  http://backend:8000           │
   │                                v                                │
   │        ┌────────────────────────────────────────────┐         │
   │        │  backend  (FastAPI)        NO host port      │         │
   │        │  • application logic + API                   │         │
   │        └───────────────────────┬────────────────────┘         │
   │                                │  postgresql://db:5432          │
   │                                v                                │
   │        ┌────────────────────────────────────────────┐         │
   │        │  db  (PostgreSQL)          NO host port      │         │
   │        │  • persistent data in a docker volume        │         │
   │        └────────────────────────────────────────────┘         │
   └────────────────────────────────────────────────────────────────┘
```

**Reading this diagram:**
- The browser can reach **only** the frontend, and only on port 80.
- The backend and database have **no host port** — nothing outside the private
  network can connect to them directly.
- Inside the network, containers reach each other by **name** (`backend`, `db`),
  not by IP address.

---

## 🔁 Network flow: how a request travels

Every API request follows the same path. The browser thinks it's talking to one
server; really it's hitting nginx, which quietly relays it inward.

```
  BROWSER                nginx (frontend)            backend            db
    │                         │                         │               │
    │  GET /                  │                         │               │
    │────────────────────────>│                         │               │
    │  <── index.html, app.js │                         │               │
    │                         │                         │               │
    │  POST /auth/login       │                         │               │
    │────────────────────────>│                         │               │
    │                         │  proxy to backend:8000  │               │
    │                         │────────────────────────>│               │
    │                         │                         │  SQL query    │
    │                         │                         │──────────────>│
    │                         │                         │  <── rows     │
    │                         │  <── JSON + JWT token   │               │
    │  <── JSON + JWT token   │                         │               │
    │                         │                         │               │
```

**What's happening, step by step:**
- The browser loads the page from nginx over **port 80** (public).
- When the page makes an API call, it goes to the **same port 80**, because the
  frontend uses relative URLs (more on this below).
- nginx looks at the path. `/` → serve a file. `/auth`, `/movies`, `/watchlist`
  → forward to the backend.
- The forwarded request travels the **private network** to `backend:8000`. This
  hop never touches the public internet.
- The backend queries the database at `db:5432` — also entirely private.
- Responses travel back out the same chain to the browser.

---

## 🧭 What is a reverse proxy? (and how this project uses it)

A **reverse proxy** is a server that sits in front of other servers and forwards
requests to them on the client's behalf. The client only ever talks to the
proxy; it never knows (or needs to know) what's behind it.

In this project, **nginx is the reverse proxy.** It does two jobs at once:

- **Static file server** — for normal page requests (`/`), it returns the HTML
  and JavaScript directly.
- **API gateway** — for API requests (`/auth`, `/movies`, `/watchlist`), it
  forwards them to the backend container over the private network and relays the
  response back.

The rule that makes this happen lives in `nginx.conf`:

```nginx
location /auth/ {
    proxy_pass http://backend:8000;
}
```

In plain terms: *"any request starting with `/auth/`, hand off to the container
named `backend` on port 8000, and return whatever it says."*

**Why a reverse proxy is worth using here:**
- **One public door.** The browser only talks to nginx. The backend and database
  stay private — fewer exposed services means a smaller attack surface.
- **No port juggling.** Without it, the browser would need to hit the backend on
  its own public port (e.g. `:8000`), meaning extra firewall rules and the
  backend's address baked into the frontend. The proxy removes all of that.
- **Same origin = no CORS headaches.** Because the page and its API calls both
  come from port 80, the browser sees them as the same origin — no
  cross-origin configuration needed.
- **A single place to add cross-cutting features later** — TLS/HTTPS,
  rate limiting, caching, logging — all slot into nginx without touching the app.

---

## 🌍 Why it runs anywhere with zero reconfiguration

A classic problem: the frontend needs to know the backend's address. Hardcode an
IP and it breaks the moment that IP changes (which EC2 IPs do on restart).

This project avoids that completely using **relative URLs**.

```
  The frontend code calls:        fetch("/auth/login")
                                          │
                                          │  no host, no port, no IP
                                          v
  Browser fills in automatically:  http://<whatever-loaded-the-page>/auth/login
```

- Open the app at `localhost` → the call goes to `localhost`.
- Open it at an EC2 IP → the call goes to that EC2 IP.
- Open it at a domain name later → the call goes to that domain.

The frontend never contains a hardcoded address. nginx receives every call on
port 80 and routes it inward. **Result: the exact same Docker image runs on a
laptop, any EC2 instance, or behind a domain — no edits, no rebuild, no config.**
The only requirement on the server is that port 80 is open.

---

## 🔌 How containers find each other (Docker networking)

When Docker Compose starts the app, it creates a **private virtual network** and
attaches all three containers to it. On that network, each container is reachable
by its **service name** — Docker runs an internal DNS that resolves names to the
right container automatically.

```
  service name      used in                       resolves to
  -------------     ---------------------------   --------------------
  db                backend's DATABASE_URL        the database container
  backend           nginx's proxy_pass rule       the backend container
  frontend          mapped to host port 80        the public entry point
```

- That's why the backend connects to `postgresql://...@db:5432` — `db` is a
  hostname Docker provides, not an IP anyone configured.
- That's why nginx forwards to `http://backend:8000` — same idea.
- **No IP addresses appear anywhere** in the config. Containers can be recreated,
  restarted, or rescheduled and the names still resolve.

**Ports — public vs. internal:**
- A `ports:` mapping like `"80:80"` opens a door from the **host machine** to a
  container. Only the frontend has one.
- The backend listens on `8000` and the database on `5432`, but **only inside the
  private network** — no `ports:` mapping means no door from the outside world.

---

## 🏗️ Build & deploy flow

Images are built once, pushed to Docker Hub, then pulled onto the server. The
server never needs the source code.

```
   YOUR MACHINE                      DOCKER HUB                  EC2 SERVER
  ──────────────                    ───────────                ────────────
   docker build  ───────push───────>  backend:2.0  ───pull────>  run
   docker build  ───────push───────> frontend:2.0  ───pull────>  run
                                     postgres:16   ───pull────>  run
                                     (official image)

   needs: source code               stores: images             needs: only
          + Dockerfiles                                          compose file
                                                                 + .env
```

- **Build** turns your code + Dockerfile into an image.
- **Push** uploads that image to Docker Hub (a registry).
- **Pull** downloads it on any machine — the server just runs the finished image.
- The database image (`postgres`) isn't built by us; it's an official image
  pulled straight from Docker Hub.

---

## 🔒 Security model

- **Port 80 is the only thing open.** Backend and database are unreachable from
  the public internet — they exist only on the private Docker network.
- **Secrets live in `.env`**, never committed to the repository. Database
  password, the token-signing key, and the API key all come from there.
- **Passwords are hashed** (bcrypt), never stored in plain text.
- **Sessions use signed JWT tokens** rather than server-side session state.

---

## 🚀 Run it locally

```bash
cp .env.example .env        # then fill in real values
docker compose up --build
```
Open **http://localhost:8081**

```bash
docker compose down         # stop, keep data
docker compose down -v      # stop, wipe the database volume
```

Generate a secure token key for `.env`:
```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

---

## ☁️ Deploy on AWS EC2

**1. Build & push (from your machine):**
```bash
docker build -t <user>/movie-recommender-backend:2.0 ./backend
docker build -t <user>/movie-recommender-frontend:2.0 ./frontend
docker push <user>/movie-recommender-backend:2.0
docker push <user>/movie-recommender-frontend:2.0
```

**2. EC2 Security Group — open only:**

| Port | Purpose |
|------|---------|
| 22 | SSH (restrict to your IP) |
| 80 | The app (open to all) |

**3. On the server** — copy up just `docker-compose.ec2.yml` and `.env`, then:
```bash
docker compose -f docker-compose.ec2.yml --env-file .env up -d
```
Open **http://&lt;ec2-public-ip&gt;** — no port number, it's on port 80.

---

## 📁 Project structure

```
movie-recommender/
├── frontend/                   # nginx + static UI (the public entry point)
│   ├── nginx.conf              # serves files + reverse-proxies the API  ← key file
│   ├── app.js                  # uses relative URLs (no hardcoded host)
│   ├── index.html
│   └── Dockerfile
├── backend/                    # FastAPI service (internal only)
│   ├── app/
│   │   ├── main.py
│   │   ├── routers/            # /auth, /movies, /watchlist
│   │   └── ...
│   ├── requirements.txt
│   └── Dockerfile
├── docker-compose.yml          # local: builds images from source
├── docker-compose.ec2.yml      # server: pulls prebuilt images
├── .env.example
└── README.md
```

---

## 📡 API endpoints (what flows through the proxy)

| Endpoint | Method | Auth | Reached via |
|----------|--------|:---:|-------------|
| `/auth/signup` | POST | – | nginx → backend |
| `/auth/login` | POST | – | nginx → backend |
| `/movies` | GET | – | nginx → backend |
| `/movies/recommend` | POST | – | nginx → backend |
| `/watchlist` | GET / POST / DELETE | ✓ | nginx → backend |

Every one of these is a relative path the browser sends to port 80; nginx
forwards it to `backend:8000` internally.

---

## 🛠️ Tech stack

- **Reverse proxy / web server:** nginx
- **Backend:** Python, FastAPI
- **Database:** PostgreSQL
- **Containerization:** Docker, Docker Compose
- **Cloud:** AWS EC2
- **App logic (the payload):** scikit-learn content-based recommender

---

## 🧩 The app, briefly

So the data flow has meaning: users sign up, pick movies or genres they like, and
get back similar films (matched with a TF-IDF + cosine-similarity model over a
~210-film dataset) which they can save to a personal watchlist. That's the
content moving through the networking pipeline described above — the focus of
this project is the pipeline, not the recommender.
