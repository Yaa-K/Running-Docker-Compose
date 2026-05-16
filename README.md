# 🐋 DockerQuiz — My Lab 

> **Original project by [Samuel Nartey](https://github.com/samuel-nartey) · [devops-labs](https://github.com/samuel-nartey/devops-labs)**
> This is my personal fork documenting what I ran, what I broke, and what I learned.

---

## What This Is

DockerQuiz is a hands-on Docker learning project — a Flask quiz app wired to MongoDB and Mongo Express, all running as three containers on a shared network. You learn Docker by *using* Docker, not just reading about it.

This README is my lab journal: the commands I ran, the experiments I tried, and the moments things finally clicked.

---

## The 3-Container Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     quiz-network  (bridge)                      │
│                                                                 │
│   ┌──────────────┐      ┌───────────────┐    ┌──────────────┐  │
│   │   quiz-app   │─────▶│     mongo     │◀───│mongo-express │  │
│   │ Flask :5000  │      │ MongoDB :27017│    │ Web UI :8081 │  │
│   └──────────────┘      └───────────────┘    └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

| Container | Image | What it does |
|-----------|-------|-------------|
| `quiz-app` | Built from `Dockerfile` | Flask app serving the quiz at `localhost:5000` |
| `mongo` | `mongo:7.0` | MongoDB storing profiles, scores, and session state |
| `mongo-express` | `mongo-express:1.0.2` | Visual DB browser at `localhost:8081` |

---

## Running It

**Only prerequisite: [Docker Desktop](https://docs.docker.com/desktop/)**. No Python, no MongoDB, nothing else.

```bash
# Clone the original repo
git clone https://github.com/samuel-nartey/devops-labs.git
cd devops-labs/Docker\ \&\ Containers/Running\ Your\ First\ Container/running\ docker\ compose

# Start all 3 containers
docker compose up --build
```

Once the terminal shows `quiz-app | * Running on http://0.0.0.0:5000`:

| URL | Service |
|-----|---------|
| http://localhost:5000 | 🐋 Quiz App |
| http://localhost:8081 | 🍃 Mongo Express |

```bash
# Stop containers, keep data
docker compose down

# Stop containers and wipe the database
docker compose down -v
```

---

## My Run

### Starting the containers

Running `docker compose up --build` for the first time pulled the MongoDB and Mongo Express images from Docker Hub — a few minutes on first run, then nearly instant every time after.

![All 3 containers starting up via docker compose up --build](./screenshots/Step%202%20—%20Start%20all%203%20containers.png)

### Playing the quiz and exploring the data

I created a profile, answered all 20 questions, then opened Mongo Express on the side to watch my data appear live. Every answer updated a document in `quiz_states`. When I finished, a full result document landed in `results`.

![The quiz app running at localhost:5000](./screenshots/Step%203%20—%20Open%20the%20apps-dockerquiz.png)
The quiz app running at localhost:5000

![](./screenshots/Step%204%20—%20Play%20the%20quiz.png)
Played the quiz and got a score


![Mongo Express showing the live MongoDB database](./screenshots/Step%203%20—%20Open%20the%20apps-mongodb.png)
Mongo Express showing the live MongoDB database


![My profile document in the profiles collection](./screenshots/Step%204%20—%20%20explore%20your%20data-profile.png)
My profile document in the profiles collection

![My completed result in the results collection](./screenshots/Step%204%20—%20explore%20your%20data-results.png)
My completed result in the results collection

### Stopping everything

![docker compose down stopping all containers](./screenshots/Step%205%20—%20Stop%20everything.png)

---

## Experiments

### ① Changing the port mapping

I changed the host port in `docker-compose.yml` from `5000` to `8000`:

```yaml
# Before
ports:
  - "5000:5000"

# After
ports:
  - "8000:5000"
```

The container still listens on port 5000 internally — nothing inside Docker changes. But now my machine maps port 8000 to that container port. `localhost:5000` stopped working; `localhost:8000` served the app instead.

![Port mapping changed to 8000:5000 in docker-compose.yml](./screenshots/Experiment%20and%20Break%20Things-Change%20the%20port%20mapping%20in%20docker-compose.yml%20to%208000%205000.png)

![Quiz app loading correctly at localhost:8000](./screenshots/access%20the%20app%20at%20port%208000.png)

**What I learned:** `HOST_PORT:CONTAINER_PORT` — the two sides are completely independent. Changing the host port changes nothing inside the container or the network. It only changes what port I use on my machine.

---

### ② Commenting out `depends_on`

I commented out the `depends_on` block so `quiz-app` would no longer wait for `mongo` to start first:

```yaml
  quiz-app:
    build: .
    # depends_on:
    #   - mongo
```

I expected the app to crash — starting before the database should break everything, right?

![Terminal output after removing depends_on — first attempt](./screenshots/Comment%20out%20depends_on%20in%20docker-compose.yml.png)

![Terminal output — subsequent run](./screenshots/Comment%20out%20depends_on%20in%20docker-compose.yml-2.png)

It worked fine.

**What I learned:** `depends_on` only controls *start order*, not readiness. It tells Docker to start the `mongo` container before `quiz-app` — but it does not wait for MongoDB to actually be accepting connections. In practice, MongoDB started fast enough that it didn't matter. And even if it hadn't, `app.py` has a built-in retry loop that attempts the connection up to 5 times with 2-second gaps.

The real takeaway: **retry logic in your application is what actually handles race conditions**. `depends_on` is a helpful hint, not a guarantee. Real-world apps can't assume their dependencies are ready the moment they start.

---

## Key Concepts This Project Made Real

| Concept | What it means in practice |
|---------|--------------------------|
| **Container DNS** | `mongo` in `app.py` resolves to the MongoDB container because both share `quiz-network`. No IPs, no localhost. |
| **Port mapping** | `HOST:CONTAINER` — the container's internal port and what you expose on your machine are independent. |
| **Named volumes** | `mongo-data:/data/db` — data written inside the container persists in a Docker-managed volume on the host. |
| **`depends_on`** | Controls start order only. Actual readiness handling belongs in the application via retry logic. |
| **Why 3 containers** | Independent scaling, independent updates, fault isolation. This mirrors real production architecture. |

---

## Credit

Original project by **[Samuel Nartey](https://github.com/samuel-nartey)**
Full DevOps labs repo: [github.com/samuel-nartey/devops-labs](https://github.com/samuel-nartey/devops-labs)