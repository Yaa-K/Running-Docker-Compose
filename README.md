// ─────────────────────────────────────────────────────────────
// Core dependencies
// express  → web framework to handle HTTP routes
// os       → gives us system info like hostname
// fs       → filesystem module so we can write log files
// path     → helps build safe file paths across OS types
// ─────────────────────────────────────────────────────────────
const express = require("express");
const os      = require("os");
const fs      = require("fs");
const path    = require("path");

const app  = express();
const PORT = 3000;

// ─────────────────────────────────────────────────────────────
// LOG FILE SETUP
// __dirname  → the folder where this app.js file lives (/app)
// logFile    → full path: /app/app.log
//
// This is the file that will FAIL to create if the container
// runs as root and you switch USER without --chown on COPY.
// appuser won't have write permission on /app (owned by root).
// ─────────────────────────────────────────────────────────────
const logFile = path.join(__dirname, "app.log");

// ─────────────────────────────────────────────────────────────
// LOGGER FUNCTION
// Called on every request. Appends one line to app.log.
// appendFileSync → opens file, adds line, closes file.
//                  If file doesn't exist yet, it creates it.
// toISOString()  → produces a standard timestamp e.g.
//                  2026-04-06T13:00:00.000Z
// ─────────────────────────────────────────────────────────────
function logRequest(method, url, statusCode) {
  const timestamp = new Date().toISOString();
  const line      = `[${timestamp}] ${method} ${url} → ${statusCode}\n`;

  try {
    fs.appendFileSync(logFile, line);
  } catch (err) {
    // If writing fails (e.g. permission denied), print to console
    // so we can see the error in docker logs without crashing the app
    console.error("Could not write to log file:", err.message);
  }
}

// ─────────────────────────────────────────────────────────────
// HOME ROUTE  →  GET /
// Serves the main HTML page.
// Template literals (backticks) let us embed JS variables
// directly into the HTML string using ${...}
// ─────────────────────────────────────────────────────────────
app.get("/", (req, res) => {

  // Gather runtime info to display on the page
  const hostname  = os.hostname();           // container ID or name
  const uid       = process.getuid();        // user running the process (0 = root, bad!)
  const platform  = os.platform();           // linux
  const nodeVer   = process.version;         // e.g. v20.11.0
  const uptime    = Math.floor(process.uptime()); // seconds since app started
  const memory    = (process.memoryUsage().rss / 1024 / 1024).toFixed(1); // MB

  // Read the current log file contents to display on the page.
  // If the file doesn't exist yet (first request), show a placeholder.
  let logContents = "No log entries yet.";
  try {
    logContents = fs.readFileSync(logFile, "utf8") || "No log entries yet.";
  } catch {
    logContents = "Log file not created yet — this is the first request!";
  }

  // Log this request BEFORE sending the response
  // so it appears in the log shown on the page
  logRequest(req.method, req.url, 200);

  // Re-read the log after writing so the page shows the latest entry
  try {
    logContents = fs.readFileSync(logFile, "utf8");
  } catch {
    logContents = "Could not read log file.";
  }

  // Send back the full HTML page as a string
  res.status(200).send(`<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Container Security Lab</title>

  <!-- Google Fonts: Syne (headings) + JetBrains Mono (code/log) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;700;800&family=JetBrains+Mono:wght@400;600&display=swap" rel="stylesheet" />

  <style>
    /* ── RESET & BASE ─────────────────────────────────── */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    /* CSS custom properties — change these to retheme everything */
    :root {
      --bg:        #0a0e17;        /* deep navy background          */
      --surface:   #111827;        /* card / panel background       */
      --border:    #1f2d45;        /* subtle borders                */
      --accent:    #00d4ff;        /* cyan highlight                */
      --accent2:   #ff6b35;        /* orange for warnings/uid=0     */
      --green:     #00ff88;        /* green for healthy status      */
      --text:      #e2e8f0;        /* primary text                  */
      --muted:     #64748b;        /* secondary / label text        */
      --font-head: 'Syne', sans-serif;
      --font-mono: 'JetBrains Mono', monospace;
    }

    body {
      background: var(--bg);
      color: var(--text);
      font-family: var(--font-head);
      min-height: 100vh;
      /* subtle grid pattern in background */
      background-image:
        linear-gradient(rgba(0,212,255,0.03) 1px, transparent 1px),
        linear-gradient(90deg, rgba(0,212,255,0.03) 1px, transparent 1px);
      background-size: 40px 40px;
    }

    /* ── LAYOUT ───────────────────────────────────────── */
    .wrapper {
      max-width: 900px;
      margin: 0 auto;
      padding: 48px 24px 80px;
    }

    /* ── HEADER ───────────────────────────────────────── */
    header {
      border-bottom: 1px solid var(--border);
      padding-bottom: 32px;
      margin-bottom: 40px;
    }

    .tag {
      display: inline-block;
      font-family: var(--font-mono);
      font-size: 11px;
      letter-spacing: 0.15em;
      text-transform: uppercase;
      color: var(--accent);
      border: 1px solid var(--accent);
      padding: 4px 10px;
      border-radius: 2px;
      margin-bottom: 16px;
      /* subtle glow on the tag */
      box-shadow: 0 0 12px rgba(0,212,255,0.2);
    }

    h1 {
      font-size: clamp(1.6rem, 4vw, 2.4rem);
      font-weight: 800;
      line-height: 1.2;
      color: #fff;
    }

    h1 span { color: var(--accent); }

    .subtitle {
      margin-top: 12px;
      color: var(--muted);
      font-size: 0.95rem;
      font-weight: 400;
    }

    .subtitle a {
      color: var(--accent);
      text-decoration: none;
      border-bottom: 1px solid transparent;
      transition: border-color 0.2s;
    }
    .subtitle a:hover { border-color: var(--accent); }

    /* ── GRID of STAT CARDS ───────────────────────────── */
    .cards {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
      gap: 16px;
      margin-bottom: 32px;
    }

    .card {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 6px;
      padding: 20px;
      position: relative;
      overflow: hidden;
      /* top coloured bar */
    }

    /* thin accent bar at top of each card */
    .card::before {
      content: '';
      position: absolute;
      top: 0; left: 0; right: 0;
      height: 2px;
      background: var(--accent);
    }

    /* warning card — orange accent for UID=0 (root) */
    .card.warn::before { background: var(--accent2); }
    .card.warn .card-value { color: var(--accent2); }

    /* good card — green for healthy */
    .card.good::before { background: var(--green); }
    .card.good .card-value { color: var(--green); }

    .card-label {
      font-family: var(--font-mono);
      font-size: 10px;
      letter-spacing: 0.12em;
      text-transform: uppercase;
      color: var(--muted);
      margin-bottom: 8px;
    }

    .card-value {
      font-size: 1.25rem;
      font-weight: 700;
      color: var(--accent);
      word-break: break-all;
    }

    /* ── SECURITY NOTE ────────────────────────────────── */
    .security-note {
      background: rgba(255, 107, 53, 0.08);
      border: 1px solid rgba(255, 107, 53, 0.3);
      border-radius: 6px;
      padding: 16px 20px;
      margin-bottom: 32px;
      font-family: var(--font-mono);
      font-size: 0.82rem;
      color: var(--accent2);
      line-height: 1.6;
    }

    .security-note strong { display: block; margin-bottom: 4px; font-size: 0.9rem; }

    /* ── LOG PANEL ────────────────────────────────────── */
    .log-panel {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 6px;
      overflow: hidden;
      margin-bottom: 32px;
    }

    .log-header {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 14px 20px;
      border-bottom: 1px solid var(--border);
      background: rgba(0,212,255,0.04);
    }

    /* three dots like a terminal titlebar */
    .dot { width: 10px; height: 10px; border-radius: 50%; }
    .dot.r { background: #ff5f57; }
    .dot.y { background: #ffbd2e; }
    .dot.g { background: #28c840; }

    .log-title {
      font-family: var(--font-mono);
      font-size: 12px;
      color: var(--muted);
      margin-left: 4px;
    }

    /* the log text area */
    .log-body {
      padding: 20px;
      font-family: var(--font-mono);
      font-size: 0.78rem;
      line-height: 1.8;
      color: #94a3b8;
      white-space: pre-wrap;       /* preserve line breaks */
      word-break: break-all;
      max-height: 260px;
      overflow-y: auto;            /* scroll if many entries */
      /* custom thin scrollbar */
      scrollbar-width: thin;
      scrollbar-color: var(--border) transparent;
    }

    /* highlight the timestamp in each log line */
    .log-body { color: #64748b; }

    /* ── FOOTER ───────────────────────────────────────── */
    footer {
      border-top: 1px solid var(--border);
      padding-top: 24px;
      font-size: 0.82rem;
      color: var(--muted);
      font-family: var(--font-mono);
    }

    footer a { color: var(--accent); text-decoration: none; }
    footer a:hover { text-decoration: underline; }
  </style>
</head>
<body>
<div class="wrapper">

  <!-- ── HEADER ───────────────────────────────────────── -->
  <header>
    <div class="tag">Container Security Lab</div>
    <h1>Hello from <span>Bismark</span></h1>
    <p class="subtitle">
      AWS Community Builder &amp; DevSecOps Facilitator &mdash;
      <a href="https://github.com/samuel-nartey/devops-labs" target="_blank">
        50+ Docker Projects on GitHub
      </a>
    </p>
  </header>

  <!-- ── STAT CARDS ────────────────────────────────────── -->
  <!--
    Each card shows one piece of runtime info.
    The "warn" class turns orange — used when UID is 0 (root),
    which is a security risk you are practising to fix.
  -->
  <div class="cards">

    <div class="card">
      <div class="card-label">Hostname</div>
      <!-- os.hostname() returns the container ID in Docker -->
      <div class="card-value">${hostname}</div>
    </div>

    <div class="card ${uid === 0 ? "warn" : "good"}">
      <div class="card-label">Running as UID</div>
      <!--
        UID 0 = root (dangerous — card turns orange).
        Any other UID = non-root user (good — card turns green).
        This is the core lesson of the containerization module.
      -->
      <div class="card-value">${uid === 0 ? "0 (root)" : uid}</div>
    </div>

    <div class="card">
      <div class="card-label">Platform</div>
      <div class="card-value">${platform}</div>
    </div>

    <div class="card">
      <div class="card-label">Node Version</div>
      <div class="card-value">${nodeVer}</div>
    </div>

    <div class="card">
      <div class="card-label">Uptime</div>
      <div class="card-value">${uptime}s</div>
    </div>

    <div class="card">
      <div class="card-label">Memory (RSS)</div>
      <div class="card-value">${memory} MB</div>
    </div>

  </div>

  <!-- ── SECURITY NOTE ─────────────────────────────────── -->
  <!--
    Shown only when the process is running as root (UID 0).
    This is a visual reminder of what you are practising to fix.
    In production you would remove this block or keep it for
    educational display.
  -->
  ${uid === 0 ? `
  <div class="security-note">
    <strong>Security Warning</strong>
    This container is running as root (UID 0). If an attacker gains access,
    they can potentially escape to the host. Add a non-root user in your
    Dockerfile and use the USER directive to fix this.
  </div>
  ` : `
  <div class="security-note" style="border-color:rgba(0,255,136,0.3);background:rgba(0,255,136,0.06);color:var(--green)">
    <strong>Good — Running as non-root (UID ${uid})</strong>
    The container is running as a non-root user. This reduces the blast
    radius if an attacker gains access to this container.
  </div>
  `}

  <!-- ── REQUEST LOG PANEL ──────────────────────────────── -->
  <!--
    This panel displays the contents of /app/app.log.
    Every GET / request appends one line to that file.
    If the container runs as root without --chown, appuser
    cannot write to this file and the app crashes on requests.
    That is the practical failure scenario you are studying.
  -->
  <div class="log-panel">
    <div class="log-header">
      <!-- terminal-style dots, purely decorative -->
      <div class="dot r"></div>
      <div class="dot y"></div>
      <div class="dot g"></div>
      <span class="log-title">/app/app.log &mdash; live request log</span>
    </div>
    <!--
      pre-wrap keeps the line breaks from the log file.
      Each line was written by logRequest() above.
    -->
    <div class="log-body">${logContents}</div>
  </div>

  <!-- ── FOOTER ────────────────────────────────────────── -->
  <footer>
    Part of the <strong>Container Security Module</strong> &mdash;
    <a href="https://github.com/samuel-nartey/devops-labs" target="_blank">
      github.com/samuel-nartey/devops-labs
    </a>
  </footer>

</div>
</body>
</html>`);
});

// ─────────────────────────────────────────────────────────────
// HEALTH CHECK ROUTE  →  GET /health
// Used by Docker / Kubernetes to check if the app is alive.
// Returns 200 OK with plain text. No logging needed here —
// health checks are called frequently and would flood the log.
// ─────────────────────────────────────────────────────────────
app.get("/health", (req, res) => {
  res.status(200).send("OK");
});

// ─────────────────────────────────────────────────────────────
// START THE SERVER
// Binds to 0.0.0.0 so it is reachable from outside the container.
// console.log prints to stdout — visible via: docker logs <container>
// ─────────────────────────────────────────────────────────────
app.listen(PORT, "0.0.0.0", () => {
  console.log(`App running on port ${PORT}`);
  console.log(`Logging requests to: ${logFile}`);
  console.log(`Running as UID: ${process.getuid()}`);
});


# devops-labs
Hands-on labs and real-world projects for Docker, Kubernetes, GitOps, CI/CD, AWS, DevSecOps and cloud engineering. Build muscle, not just theory.

----
<div align="center">

<img src="./assets/asset1.png" alt="DevOps Labs Banner" width="100%"/>

<h1>⚙️ DevOps Labs</h1>

<p>
  <strong>Production-grade, hands-on labs for DevOps, DevSecOps and Cloud Engineers.</strong><br/>
  No slides. No passive reading. You build, break, fix and ship — like you would on the job.
</p>

![Stars](https://img.shields.io/github/stars/YOUR_USERNAME/devops-labs?style=for-the-badge&logo=github&color=FFD700&labelColor=0D1117)
![Forks](https://img.shields.io/github/forks/YOUR_USERNAME/devops-labs?style=for-the-badge&logo=github&color=4A90D9&labelColor=0D1117)
![Last Commit](https://img.shields.io/github/last-commit/YOUR_USERNAME/devops-labs?style=for-the-badge&color=2ECC71&labelColor=0D1117)
![License](https://img.shields.io/github/license/YOUR_USERNAME/devops-labs?style=for-the-badge&color=E74C3C&labelColor=0D1117)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=for-the-badge&labelColor=0D1117)
![Contributors](https://img.shields.io/github/contributors/YOUR_USERNAME/devops-labs?style=for-the-badge&color=blueviolet&labelColor=0D1117)

</div>

---

## 🧭 What This Is

**DevOps Labs** is a structured, progressive workshop repo built for engineers who want to move from *knowing* DevOps tools to *actually using them under realistic conditions*.

Every lab is modelled on real-world scenarios — the kind you encounter in production environments, technical interviews and cloud certifications. Each one has a clear objective, a defined outcome and a measurable result.

> Built by a practitioner, for practitioners. Start with Git. Graduate to the full production stack.

---

## 👥 Who This Is For

| Role | How you'll use this |
|------|-------------------|
| **DevOps Engineers** | Build and sharpen core container, CI/CD and IaC skills |
| **DevSecOps Practitioners** | Integrate security scanning, policies and secrets management into pipelines |
| **Cloud Engineers** | Deploy and manage workloads across AWS, GCP or Azure |
| **Platform Engineers** | Build internal tooling, GitOps workflows and developer platforms |
| **SREs** | Practice observability, incident response and reliability patterns |
| **Software Engineers** | Learn the operational side — containers, pipelines, cloud deployments |

---

## 🗺️ Learning Roadmap
```
Git & GitHub → Linux → Docker → Kubernetes → CI/CD → Terraform → AWS/Cloud → DevSecOps → Observability
     │              │        │           │          │           │            │              │
  Foundation    Foundation  Core      Orchestration Automation  IaC       Cloud Native  Security
```

Start at the beginning or jump into the track that matches your current level.

---

## 🗂️ Lab Tracks

---

### 🌿 Track 1 — Git & GitHub *(coming soon)*

> **Goal:** Master version control and collaborative workflows the way engineering teams actually use them — branching strategies, PR workflows, hooks and automation.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Git Fundamentals & Mental Model](./git/lab-01/) | Local repo, staging, commits, history | 🟢 Foundation | 30 min |
| 02 | [Branching Strategies](./git/lab-02/) | GitFlow, trunk-based, feature branches | 🟢 Foundation | 45 min |
| 03 | [Merge vs Rebase vs Squash](./git/lab-03/) | Clean history, conflict resolution | 🟡 Intermediate | 45 min |
| 04 | [Pull Request Workflows](./git/lab-04/) | PR templates, reviews, branch protection | 🟡 Intermediate | 60 min |
| 05 | [Git Hooks & Automation](./git/lab-05/) | Pre-commit hooks, linting, secret scanning | 🟡 Intermediate | 45 min |
| 06 | [GitHub Actions — First Pipeline](./git/lab-06/) | Trigger CI on push, run tests, notify | 🔴 Advanced | 60 min |
| 07 | [Monorepo Management](./git/lab-07/) | Path filtering, affected-only pipelines | 🔴 Advanced | 75 min |

---

### 🐧 Track 2 — Linux & Shell *(coming soon)*

> **Goal:** Operate confidently in Linux environments — the foundation every DevOps and cloud role assumes you have.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Linux CLI Survival Kit](./linux/lab-01/) | Navigate, manage files, permissions | 🟢 Foundation | 45 min |
| 02 | [Users, Groups & Permissions](./linux/lab-02/) | sudoers, chmod, chown, ACLs | 🟢 Foundation | 45 min |
| 03 | [Shell Scripting for DevOps](./linux/lab-03/) | Automate backups, deploys, health checks | 🟡 Intermediate | 60 min |
| 04 | [Process & Service Management](./linux/lab-04/) | systemd, journalctl, ps, kill | 🟡 Intermediate | 45 min |
| 05 | [Networking Fundamentals](./linux/lab-05/) | curl, netstat, ss, iptables, DNS | 🟡 Intermediate | 60 min |
| 06 | [Linux Hardening Basics](./linux/lab-06/) | SSH keys, fail2ban, auditd, ufw | 🔴 Advanced | 75 min |

---

### 🐳 Track 3 — Docker & Containers

> **Goal:** Master containers from first principles to production-hardened, security-scanned deployments.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Running Your First Container](./docker/lab-01/) | Run, inspect and manage containers | 🟢 Foundation | 30 min |
| 02 | [Writing Production Dockerfiles](./docker/lab-02/) | Multi-stage builds, layer optimisation | 🟡 Intermediate | 45 min |
| 03 | [Multi-Service App with Compose](./docker/lab-03/) | 3-tier app: app + db + reverse proxy | 🟡 Intermediate | 60 min |
| 04 | [Docker Networking Deep Dive](./docker/lab-04/) | Bridge, host and overlay networks | 🟡 Intermediate | 45 min |
| 05 | [Volumes & Persistent Storage](./docker/lab-05/) | Bind mounts, named volumes, backups | 🟡 Intermediate | 45 min |
| 06 | [Container Security Scanning](./docker/lab-06/) | Scan images with Trivy, fix CVEs | 🔴 Advanced | 60 min |
| 07 | [Rootless Containers & Hardening](./docker/lab-07/) | Least privilege, read-only FS, seccomp | 🔴 Advanced | 75 min |
| 08 | [Private Registry & Image Signing](./docker/lab-08/) | Harbor registry, Cosign, SBOM | 🔴 Advanced | 90 min |

---

### ☸️ Track 4 — Kubernetes *(coming soon)*

> **Goal:** Deploy, manage, scale and secure containerised workloads on Kubernetes — from local clusters to production-grade configurations.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Kubernetes Architecture & Core Objects](./kubernetes/lab-01/) | Pods, Deployments, Services, Namespaces | 🟢 Foundation | 60 min |
| 02 | [ConfigMaps, Secrets & Environment Config](./kubernetes/lab-02/) | Externalise config, manage secrets safely | 🟡 Intermediate | 45 min |
| 03 | [Persistent Volumes & StatefulSets](./kubernetes/lab-03/) | Deploy a stateful database workload | 🟡 Intermediate | 60 min |
| 04 | [Ingress Controllers & TLS Termination](./kubernetes/lab-04/) | NGINX Ingress, cert-manager, HTTPS | 🟡 Intermediate | 75 min |
| 05 | [Resource Limits, HPA & Autoscaling](./kubernetes/lab-05/) | CPU/memory limits, horizontal pod autoscaler | 🟡 Intermediate | 60 min |
| 06 | [RBAC & Service Account Hardening](./kubernetes/lab-06/) | Roles, RoleBindings, least privilege | 🔴 Advanced | 75 min |
| 07 | [Network Policies](./kubernetes/lab-07/) | Zero-trust networking between pods | 🔴 Advanced | 60 min |
| 08 | [Helm — Package & Deploy Applications](./kubernetes/lab-08/) | Write, template and release a Helm chart | 🔴 Advanced | 90 min |
| 09 | [GitOps with ArgoCD](./kubernetes/lab-09/) | Declarative deployments, sync, rollback | 🔴 Advanced | 90 min |
| 10 | [Cluster Hardening & CIS Benchmarks](./kubernetes/lab-10/) | kube-bench, Pod Security Standards, audit logs | 🔵 Expert | 120 min |

---

### 🔄 Track 5 — CI/CD Pipelines  *(coming soon)*

> **Goal:** Build automated, secure, end-to-end pipelines from commit to production deployment.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [GitHub Actions — CI Pipeline](./cicd/lab-01/) | Build, test and lint on every push | 🟢 Foundation | 45 min |
| 02 | [Dockerise & Push in a Pipeline](./cicd/lab-02/) | Build image, tag, push to registry | 🟡 Intermediate | 60 min |
| 03 | [Secrets Management in Pipelines](./cicd/lab-03/) | GitHub Secrets, OIDC, least privilege | 🟡 Intermediate | 60 min |
| 04 | [Matrix Builds & Parallelism](./cicd/lab-04/) | Multi-OS, multi-version parallel jobs | 🟡 Intermediate | 45 min |
| 05 | [Security Scanning in CI](./cicd/lab-05/) | Trivy, Snyk, SAST — fail on critical CVEs | 🔴 Advanced | 75 min |
| 06 | [CD to Kubernetes via ArgoCD](./cicd/lab-06/) | GitOps CD pipeline, image update automation | 🔴 Advanced | 90 min |
| 07 | [Pipeline as Code with Jenkins](./cicd/lab-07/) | Declarative Jenkinsfile, shared libraries | 🔴 Advanced | 90 min |
| 08 | [GitLab CI End-to-End Pipeline](./cicd/lab-08/) | Full GitLab CI/CD with environments | 🔴 Advanced | 90 min |

---

### 🏗️ Track 6 — Infrastructure as Code (Terraform) -- *(coming soon)*

> **Goal:** Provision, manage and version cloud infrastructure declaratively — and safely destroy it when you're done.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Terraform Fundamentals](./terraform/lab-01/) | Providers, resources, state, plan, apply | 🟢 Foundation | 60 min |
| 02 | [Variables, Outputs & Locals](./terraform/lab-02/) | Reusable, parameterised configs | 🟢 Foundation | 45 min |
| 03 | [Remote State & Locking](./terraform/lab-03/) | S3 backend, DynamoDB locking | 🟡 Intermediate | 60 min |
| 04 | [Modules — Write Your Own](./terraform/lab-04/) | Reusable VPC, compute and security modules | 🟡 Intermediate | 90 min |
| 05 | [Provision an AWS EKS Cluster](./terraform/lab-05/) | Full EKS cluster with node groups via Terraform | 🔴 Advanced | 120 min |
| 06 | [Terraform Security & Compliance](./terraform/lab-06/) | tfsec, Checkov, policy-as-code with Sentinel | 🔴 Advanced | 90 min |
| 07 | [Terragrunt — DRY Terraform at Scale](./terraform/lab-07/) | Multi-env, multi-account structure | 🔵 Expert | 120 min |

---

### ☁️ Track 7 — AWS & Cloud Engineering *(coming soon)*

> **Goal:** Deploy and operate real workloads on AWS — compute, networking, storage, security and cost management.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [AWS Core Services Orientation](./aws/lab-01/) | EC2, S3, IAM, VPC — hands-on tour | 🟢 Foundation | 60 min |
| 02 | [VPC from Scratch](./aws/lab-02/) | Custom VPC, subnets, IGW, route tables, NAT | 🟡 Intermediate | 90 min |
| 03 | [IAM — Least Privilege in Practice](./aws/lab-03/) | Roles, policies, SCPs, permission boundaries | 🟡 Intermediate | 75 min |
| 04 | [Deploy a Containerised App on ECS Fargate](./aws/lab-04/) | Task definitions, services, ALB, ECR | 🟡 Intermediate | 90 min |
| 05 | [Serverless with Lambda & API Gateway](./aws/lab-05/) | Event-driven API, S3 trigger, DLQ | 🟡 Intermediate | 90 min |
| 06 | [EKS — Managed Kubernetes on AWS](./aws/lab-06/) | Deploy cluster, IRSA, ALB Ingress, EBS CSI | 🔴 Advanced | 120 min |
| 07 | [AWS Security Hardening](./aws/lab-07/) | GuardDuty, Security Hub, Config, CloudTrail | 🔴 Advanced | 90 min |
| 08 | [Multi-Account Architecture](./aws/lab-08/) | AWS Organizations, Control Tower, SCPs | 🔵 Expert | 120 min |
| 09 | [Cost Optimisation Engineering](./aws/lab-09/) | Cost Explorer, Budgets, rightsizing, Savings Plans | 🔴 Advanced | 75 min |

---

### 🛡️ Track 8 — DevSecOps & Security Engineering  *(coming soon)*

> **Goal:** Shift security left — embed it into code, containers, pipelines and infrastructure from day one.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Threat Modelling for DevOps Pipelines](./devsecops/lab-01/) | STRIDE model applied to a real pipeline | 🟡 Intermediate | 60 min |
| 02 | [SAST — Static Code Analysis in CI](./devsecops/lab-02/) | Semgrep, Bandit, fail pipeline on findings | 🟡 Intermediate | 60 min |
| 03 | [SCA — Dependency & Licence Scanning](./devsecops/lab-03/) | Snyk, OWASP Dependency-Check, SBOM | 🟡 Intermediate | 60 min |
| 04 | [Secrets Detection & Prevention](./devsecops/lab-04/) | Gitleaks, truffleHog, pre-commit hooks | 🟡 Intermediate | 45 min |
| 05 | [Container Image Hardening](./devsecops/lab-05/) | Distroless, Trivy, Dockle, CIS benchmarks | 🔴 Advanced | 90 min |
| 06 | [Policy as Code with OPA & Gatekeeper](./devsecops/lab-06/) | Enforce Kubernetes admission policies | 🔴 Advanced | 90 min |
| 07 | [Runtime Security with Falco](./devsecops/lab-07/) | Detect anomalous container behaviour live | 🔴 Advanced | 90 min |
| 08 | [Secrets Management with HashiCorp Vault](./devsecops/lab-08/) | Dynamic secrets, PKI, Kubernetes auth | 🔴 Advanced | 120 min |
| 09 | [Supply Chain Security & SLSA](./devsecops/lab-09/) | Cosign, SBOM, provenance, SLSA Level 2 | 🔵 Expert | 120 min |
| 10 | [Zero Trust Architecture on Kubernetes](./devsecops/lab-10/) | mTLS with Istio, SPIFFE/SPIRE, network policies | 🔵 Expert | 150 min |

---

### 📊 Track 9 — Observability & Monitoring  *(coming soon)*

> **Goal:** Build visibility into systems before things break — and diagnose fast when they do.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Prometheus & Grafana Stack](./observability/lab-01/) | Scrape metrics, build dashboards | 🟢 Foundation | 60 min |
| 02 | [Application Instrumentation](./observability/lab-02/) | Custom metrics with Prometheus client libs | 🟡 Intermediate | 60 min |
| 03 | [Centralised Logging — ELK Stack](./observability/lab-03/) | Ship, parse and query logs with Elasticsearch | 🟡 Intermediate | 90 min |
| 04 | [Distributed Tracing with Jaeger](./observability/lab-04/) | Trace requests across microservices | 🔴 Advanced | 90 min |
| 05 | [Alerting & On-Call Runbooks](./observability/lab-05/) | AlertManager, PagerDuty, runbook templates | 🔴 Advanced | 75 min |
| 06 | [SLOs, SLIs & Error Budgets](./observability/lab-06/) | Define, measure and report reliability | 🔴 Advanced | 90 min |

---

### ⚙️ Track 10 — Configuration Management *(coming soon)*

> **Goal:** Manage infrastructure configuration at scale — consistently, repeatably and auditably.

| # | Lab | What You'll Build | Difficulty | Est. Time |
|---|-----|-------------------|-----------|-----------|
| 01 | [Ansible Fundamentals](./ansible/lab-01/) | Inventory, playbooks, ad-hoc commands | 🟢 Foundation | 60 min |
| 02 | [Roles & Reusable Playbooks](./ansible/lab-02/) | Modular, idempotent configuration | 🟡 Intermediate | 75 min |
| 03 | [Ansible Vault — Secrets at Rest](./ansible/lab-03/) | Encrypt secrets in playbooks and vars | 🟡 Intermediate | 45 min |
| 04 | [Configure a K8s Cluster with Ansible](./ansible/lab-04/) | Provision and configure nodes end-to-end | 🔴 Advanced | 120 min |

---

## 🛠️ Tools & Technologies Covered

<div align="center">

| Category | Tools |
|----------|-------|
| **Version Control** | Git, GitHub, GitLab, pre-commit, Gitleaks |
| **Containers** | Docker, Docker Compose, Podman, Harbor, Cosign |
| **Orchestration** | Kubernetes, Helm, ArgoCD, Kustomize, k3s |
| **CI/CD** | GitHub Actions, GitLab CI, Jenkins, Tekton |
| **Infrastructure as Code** | Terraform, Terragrunt, Pulumi, Ansible |
| **Cloud** | AWS (EKS, ECS, Lambda, VPC, IAM, S3, RDS) |
| **Security** | Trivy, Falco, OPA, Vault, Semgrep, Snyk, Gatekeeper |
| **Observability** | Prometheus, Grafana, ELK Stack, Jaeger, AlertManager |
| **Networking** | NGINX, Istio, Envoy, cert-manager, ExternalDNS |
| **Operating Systems** | Ubuntu, Amazon Linux 2, Alpine |

</div>

---

## 📁 Repository Structure
```
devops-labs/
├── git/
│   ├── lab-01-fundamentals/
│   ├── lab-02-branching/
│   └── ...
├── linux/
│   ├── lab-01-cli/
│   └── ...
├── docker/
│   ├── lab-01-first-container/
│   ├── lab-02-dockerfiles/
│   └── ...
├── kubernetes/
│   ├── lab-01-core-objects/
│   └── ...
├── cicd/
│   ├── lab-01-github-actions/
│   └── ...
├── terraform/
│   ├── lab-01-fundamentals/
│   └── ...
├── aws/
│   ├── lab-01-core-services/
│   └── ...
├── devsecops/
│   ├── lab-01-threat-modelling/
│   └── ...
├── observability/
│   ├── lab-01-prometheus-grafana/
│   └── ...
├── ansible/
│   ├── lab-01-fundamentals/
│   └── ...
└── assets/
    └── banner.png
```

Each lab folder follows this structure:
```
lab-01-first-container/
├── README.md          ← objectives, background, tasks
├── solution/          ← reference solution (attempt first)
├── Dockerfile         ← starter file where applicable
└── docker-compose.yml ← where applicable
```

---

## 🚀 Getting Started
```bash
# 1. Clone the repo
git clone https://github.com/YOUR_USERNAME/devops-labs.git

# 2. Navigate into a track
cd devops-labs/docker/lab-01-first-container

# 3. Read the lab brief
cat README.md
```

### Difficulty guide

| Badge | Level | Assumes |
|-------|-------|---------|
| 🟢 Foundation | No prior experience with this tool | Basic Linux CLI |
| 🟡 Intermediate | Comfortable with the basics | Tool fundamentals done |
| 🔴 Advanced | Production-level challenge | Intermediate labs done |
| 🔵 Expert | Mirrors real engineering interviews and incidents | Full track completed |

---

## 🛠️ Prerequisites by Track

| Track | Minimum Requirements |
|-------|---------------------|
| Git & GitHub | A GitHub account, terminal access |
| Linux | A Linux/macOS terminal or WSL2 on Windows |
| Docker | Docker Desktop or Docker Engine installed |
| Kubernetes | Docker done, kubectl installed, k3s or minikube |
| CI/CD | GitHub account, Docker Hub or ECR account |
| Terraform | AWS account (free tier), Terraform CLI installed |
| AWS | AWS account, AWS CLI configured |
| DevSecOps | Kubernetes track recommended first |
| Observability | Docker or Kubernetes track done |
| Ansible | Linux track done, SSH access to at least one target |

---

## 🤝 Contributing

Contributions are welcome — new labs, corrections, improved solutions and translations.

1. Fork this repository
2. Create a branch: `git checkout -b lab/track-name-lab-title`
3. Follow the [lab template](./CONTRIBUTING.md)
4. Open a Pull Request with a clear description of what the lab teaches

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before submitting.

---

## 📄 License

MIT — free to use, share and build on. Attribution appreciated.

---

<div align="center">

**If this repo helped you — star it ⭐**<br/>
<sub>It takes 2 seconds and helps other engineers find it.</sub>

<br/>

![Visitors](https://visitor-badge.laobi.icu/badge?page_id=YOUR_USERNAME.devops-labs)

</div>
