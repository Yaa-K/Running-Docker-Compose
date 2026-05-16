# 🐋 DockerQuiz — Learn Docker by Playing

> A hands-on, Kahoot-style quiz app built with **Python + Flask**, **MongoDB**, and **Mongo Express** — all wired together as a real **3-container Docker application**.
>
> Built by [Samuel Nartey](https://github.com/samuel-nartey) as a beginner-friendly, zero-prerequisites gateway into Docker and container concepts.

---

##  What Is This Project?

DockerQuiz is **not just a quiz** — it is a fully containerized application designed to teach you Docker by letting you *use* Docker, not just read about it.

Most Docker tutorials show you commands in isolation. This project shows you how real applications are built — multiple containers, a shared network, a database, and a visual admin tool — all running with a single command.

**Your learning journey:**

```
① Play the quiz         →   Build your Docker conceptual foundation (20 questions)
② Run with Compose      →   See 3 containers start, connect, and talk to each other
③ Explore Mongo Express →   Watch your own data appear live in the database UI
④ Read the code         →   Understand WHY it is built the way it is
⑤ Modify & experiment   →   Break things, fix things, make it your own
⑥ Build your own        →   Containerize your own project with confidence
```

---

##  The 3-Container Architecture

When you run `docker compose up`, Docker starts **three containers** that communicate over a private internal network called `quiz-network`:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         quiz-network  (bridge)                           │
│                                                                          │
│   ┌──────────────────┐      ┌─────────────────┐    ┌─────────────────┐  │
│   │    quiz-app      │─────▶│      mongo      │◀───│  mongo-express  │  │
│   │  Flask  :5000    │      │  MongoDB :27017 │    │  Web UI  :8081  │  │
│   │  (our Dockerfile)│      │ (official image)│    │ (official image)│  │
│   └──────────────────┘      └─────────────────┘    └─────────────────┘  │
│          │                          │                       │            │
│   Your quiz interface        Stores profiles,        Browse your data   │
│   at localhost:5000          scores & sessions        at localhost:8081  │
└──────────────────────────────────────────────────────────────────────────┘
```

### Why 3 containers and not just 1?

In real-world production applications, you almost never run everything in a single container. Separating concerns gives you:

- **Independent scaling** — scale only the part that needs more resources
- **Independent updates** — update your app without touching the database
- **Reusability** — swap MongoDB for PostgreSQL without rewriting your app
- **Isolation** — a crash in one container does not bring down the others

This project mirrors exactly how real applications are deployed.

###  Key Concept: Container DNS

The single most important thing to understand about this project is how `quiz-app` talks to MongoDB. It does **not** use `localhost` — it uses the **service name** defined in `docker-compose.yml` as the hostname:

```python
# app.py
MONGO_URI = "mongodb://mongo:27017/"
#                       ↑
#           This is the service name — not an IP address, not localhost
```

Docker automatically resolves `mongo` to the correct container IP because both containers share `quiz-network`. This is **Docker's built-in DNS** — containers discover and talk to each other by service name. This is the pattern used in virtually every real containerized application.

---

## 🗂️ Project Structure

```
running docker compose/
│
├── app.py                  # Flask app — all routes, quiz logic, MongoDB writes
├── Dockerfile              # Instructions to build the quiz-app container image
├── docker-compose.yml      # Defines all 3 services, the network, and the volume
├── requirements.txt        # Python dependencies: Flask + PyMongo
├── .dockerignore           # Files excluded from the Docker build context
├── README.md               # You are here
│
└── templates/              # HTML pages rendered by Flask (Jinja2)
    ├── base.html           # Shared navigation, fonts, and CSS variables
    ├── index.html          # Profile creation landing page
    ├── question.html       # Quiz question with 4 answer choices
    ├── feedback.html       # Correct/wrong result + explanation after each answer
    └── results.html        # Final score, grade, and Mongo Express guide
```

---

##  Getting Started

### Prerequisites

**Nothing** — except Docker Desktop. No Python, no MongoDB, no Node.js. Docker manages everything inside containers.

- **Mac**: https://docs.docker.com/desktop/install/mac-install/
- **Windows**: https://docs.docker.com/desktop/install/windows-install/
- **Linux**: https://docs.docker.com/desktop/install/linux-install/

After installing, open Docker Desktop and confirm the whale icon is visible in your taskbar or menu bar. That means Docker is running.

---

### Step 1 — Clone the repository

```bash
git clone https://github.com/samuel-nartey/devops-labs.git
cd devops-labs/Docker\ \&\ Containers/Running\ Your\ First\ Container/running\ docker\ compose
```

---

### Step 2 — Start all 3 containers

```bash
docker compose up --build
```

This single command will:

1. Build the `quiz-app` image from the `Dockerfile` in this folder
2. Pull the official `mongo:7.0` image from Docker Hub
3. Pull the official `mongo-express:1.0.2` image from Docker Hub
4. Create the `quiz-network` bridge network
5. Create the `mongo-data` named volume for data persistence
6. Start all 3 containers and connect them

> **First run note:** Docker needs to download the MongoDB and Mongo Express images. This takes 1-3 minutes depending on your internet speed. Every run after the first will be nearly instant.

---

### Step 3 — Open the apps

Once you see `quiz-app | * Running on http://0.0.0.0:5000` in your terminal:

| Service | URL | What it does |
|---------|-----|--------------|
| 🐋 Quiz App | http://localhost:5000 | Create your profile and play the quiz |
| 🍃 Mongo Express | http://localhost:8081 | Browse your MongoDB data visually |

---

### Step 4 — Play the quiz, then explore your data

1. Go to **http://localhost:5000**
2. Create your profile — enter your name, pick a role and an avatar
3. Answer all 20 Docker questions
4. On the results page, follow the Mongo Express guide
5. Go to **http://localhost:8081** → click `dockerquiz` → open `profiles`, `results`, and `quiz_states`

You will see your own data — written by the Flask container, stored in the MongoDB container, and browsed through the Mongo Express container. Three containers, one network, one command.

---

### Step 5 — Stop everything

```bash
# Stop containers but keep your data
docker compose down

# Stop containers AND delete all saved quiz data
docker compose down -v
```

---

##  Understanding the docker-compose.yml

Open `docker-compose.yml` and read it alongside these annotations — this file is the heart of the project:

```yaml
services:

  quiz-app:                          # Service 1 — our Flask application
    build: .                         # Build image from the Dockerfile in this folder
    container_name: dockerquiz-app
    ports:
      - "5000:5000"                  # HOST:CONTAINER — your port 5000 to container port 5000
    environment:
      - MONGO_URI=mongodb://mongo:27017/  # "mongo" resolves via Docker DNS to the mongo container
    depends_on:
      - mongo                        # Wait for mongo to start before starting quiz-app
    networks:
      - quiz-network                 # Join the shared internal network

  mongo:                             # Service 2 — the database
    image: mongo:7.0                 # Pull this exact image from Docker Hub (no Dockerfile needed)
    container_name: dockerquiz-mongo
    volumes:
      - mongo-data:/data/db          # Persist data to a named volume (survives restarts)
    networks:
      - quiz-network

  mongo-express:                     # Service 3 — visual database browser
    image: mongo-express:1.0.2
    container_name: dockerquiz-mongo-express
    ports:
      - "8081:8081"                  # Access the UI at localhost:8081
    environment:
      ME_CONFIG_MONGODB_SERVER: mongo     # Tells Mongo Express where MongoDB is (by service name)
    depends_on:
      - mongo
    networks:
      - quiz-network

volumes:
  mongo-data:                        # Named volume — your data lives here between restarts

networks:
  quiz-network:
    driver: bridge                   # All 3 containers share this private network
```

---

##  What Gets Saved to MongoDB

Every action in the quiz writes a document to MongoDB. Open Mongo Express at http://localhost:8081 and watch the collections update as you play.

### `profiles` collection
Created the moment you submit the profile form:
```json
{
  "_id": "ObjectId(...)",
  "name": "Samuel Nartey",
  "avatar": "🐋",
  "role": "DevOps Engineer",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### `quiz_states` collection
Updated live after every question — your in-progress state:
```json
{
  "sid": "a3f9c2b1d4e87f09...",
  "profile_name": "Samuel Nartey",
  "score": 60,
  "current": 7,
  "wrong": [2, 5]
}
```

### `results` collection
Created when you complete all 20 questions:
```json
{
  "_id": "ObjectId(...)",
  "name": "Samuel Nartey",
  "avatar": "🐋",
  "score": 170,
  "total": 200,
  "percentage": 85,
  "grade": "Container Expert",
  "num_correct": 17,
  "num_wrong": 3,
  "wrong_question_ids": [4, 9, 14],
  "completed_at": "2024-01-15T10:45:00Z"
}
```

---

##  Useful Commands to Run While the Containers Are Up

Open a second terminal and try these while the quiz is running:

```bash
# See all 3 running containers and their status
docker ps

# Stream live logs from the quiz app
docker logs dockerquiz-app -f

# Stream live logs from MongoDB
docker logs dockerquiz-mongo -f

# Open a shell inside the running quiz-app container
docker exec -it dockerquiz-app bash

# Open a MongoDB shell and query your data directly
docker exec -it dockerquiz-mongo mongosh

# Inside mongosh — explore your collections:
use dockerquiz
db.profiles.find().pretty()
db.results.find().pretty()
db.quiz_states.find().pretty()

# See all Docker networks on your machine
docker network ls

# Inspect quiz-network — see which containers are connected
docker network inspect docker-quiz-v2_quiz-network

# See all volumes
docker volume ls
```

---

##  How the Session System Works (No Cookies)

A common problem with Flask running inside Docker is that browser cookies are unreliable across container network boundaries. This project solves that with a clean architectural decision — the session ID lives in the **URL**, not in a cookie:

```
http://localhost:5000/question/a3f9c2b1d4e87f09...
                               ↑
                    Unique session ID embedded in the URL
```

Your quiz state (score, current question, name, avatar) is stored in MongoDB under that ID and looked up on every request. This means:

- No cookie issues — works reliably inside Docker
- Your progress survives a browser refresh
- Multiple people can play simultaneously without interference
- You can watch your live state update in Mongo Express as you answer each question

---

##  Grade System

| Score | Grade |
|-------|-------|
| 90 – 100% | 🏆 Docker Captain |
| 75 – 89% | 🥇 Container Expert |
| 60 – 74% | 🥈 Image Builder |
| 40 – 59% | 🥉 Dockerfile Rookie |
| 0 – 39% | 🐋 Whale Watcher |

---

##  Experiment and Break Things

Once you have played the quiz, the real learning starts. Make changes and observe what happens:

```bash
# After any code change, rebuild
docker compose up --build

# Force a completely fresh build ignoring all cached layers
docker compose up --build --no-cache

# Nuke everything and start from absolute zero
docker compose down -v
docker system prune -a
docker compose up --build
```

**Ideas to try:**

- Add a new question to the `QUESTIONS` list in `app.py`
- Change the port mapping in `docker-compose.yml` from `"5000:5000"` to `"8000:5000"` and access the app at port 8000
- Add a new field to the profile form in `index.html` and see it appear in MongoDB
- Edit the CSS variables at the top of `base.html` to change the entire colour scheme
- Comment out `depends_on` in `docker-compose.yml` and observe what happens on startup

---

##  FAQ

**Q: Do I need to install Python or MongoDB?**
No. Docker installs and manages everything inside containers. Your machine only needs Docker Desktop.

**Q: The first run takes a long time — is something wrong?**
No. Docker is downloading the MongoDB and Mongo Express images from Docker Hub for the first time. This only happens once. After that, startup takes a few seconds.

**Q: I see "MongoDB not ready" messages in the logs — is it broken?**
No. The quiz app starts faster than MongoDB and retries the connection automatically up to 5 times with 2-second gaps. This is expected behaviour and resolves on its own.

**Q: How do I reset all my quiz data and start fresh?**
```bash
docker compose down -v
docker compose up
```
The `-v` flag removes the `mongo-data` volume, which wipes the database completely.

**Q: Can multiple people play at the same time?**
Yes. Each player gets a unique session ID embedded in their URL, so multiple players never interfere with each other.

**Q: Where do I find my data in Mongo Express?**
Go to http://localhost:8081 → click **dockerquiz** → you will see the `profiles`, `results`, and `quiz_states` collections.

**Q: I changed the code but the app looks the same after rebuilding — why?**
Docker may have used cached image layers. Run `docker compose up --build --no-cache` to force a complete rebuild from scratch.

---

##  Go Further — Hands-On Docker Labs

This quiz gives you the theory. The labs repo gives you the practice.

** [github.com/samuel-nartey/devops-labs](https://github.com/samuel-nartey/devops-labs)**

The DevOps Labs repo contains practical exercises where you will:
- Write Dockerfiles from scratch for different application types
- Use `docker init` to containerize existing projects automatically
- Build multi-stage images to dramatically reduce final image size
- Set up Docker Compose stacks with real-world service combinations
- Work hands-on with volumes, networking, and environment variables

---

##  Author

**Samuel Nartey**
GitHub: [@samuel-nartey](https://github.com/samuel-nartey)
DevOps Labs: [github.com/samuel-nartey/devops-labs](https://github.com/samuel-nartey/devops-labs)

---

## License

MIT — free to use, fork, and share.
If this helped you learn Docker, give it a ⭐ and pass it on to someone who is just getting started.
