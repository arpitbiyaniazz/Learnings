# Chapter 1 — Docker, Container Images, and Multi-Stage Builds

**What this chapter is.** This is the containerization foundation for the GIBP ledger platform — and a stand-alone book on Docker you can carry to any project. It starts from the raw problem ("it works on my machine"), builds up the mental model of *images* and *containers*, teaches every core concept with the exact commands and flags you'll actually type, and then walks the real Dockerfiles in this repository line by line, explaining *why* each decision was made. By the end you should be able to (a) containerize a project from an empty folder, and (b) read, debug, and safely change every `Dockerfile` in this monorepo.

> **Status in this repo.** Docker is load-bearing here. Every deployable app ships as a container image: the NestJS `api`, the two React front-ends (`company-webapp`, `super-webapp`), and the `kafka-worker`. They all build on a **shared base image** (`gibp-base:latest`, defined in `Dockerfile.base`), get pushed to **Google Artifact Registry** (`us-west1-docker.pkg.dev/gibp-ledger/gibp-apps`), and run on **Google Kubernetes Engine (GKE)**. Local development brings the whole system up with `docker-compose.yml`. Node is pinned to **24.14.0** across the board.

**How to read this.** If you're brand new, read top to bottom — each section removes a wall you'd otherwise hit in the next. If you already know Docker, skim §1–§4, then read **§7 (How THIS repo uses Docker)** and **§10 (Gotchas)** closely — that's where the repo-specific knowledge lives. Every code block is copy-pasteable. Commands you can run on your own machine are marked; commands that only make sense inside CI or the cluster are called out. This chapter does **not** cover the CI/CD pipeline (who runs `docker build`, how images reach the cluster) or `docker-compose` in depth — those have their own chapters. See [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md) for the pipeline, and the compose chapter for multi-container local dev.

---

## Table of contents

1. [The problem: why containers exist](#1-the-problem-why-containers-exist)
2. [Mental model: the lunchbox that packs the kitchen](#2-mental-model-the-lunchbox-that-packs-the-kitchen)
3. [Core concepts & vocabulary](#3-core-concepts--vocabulary)
4. [How it actually works](#4-how-it-actually-works)
5. [Setup from scratch: zero to a running container](#5-setup-from-scratch-zero-to-a-running-container)
6. [Using it well: patterns that matter](#6-using-it-well-patterns-that-matter)
7. [How THIS repo uses Docker](#7-how-this-repo-uses-docker)
8. [Production concerns](#8-production-concerns)
9. [Operations & debugging playbooks](#9-operations--debugging-playbooks)
10. [Gotchas & hard-won lessons](#10-gotchas--hard-won-lessons)
11. [Glossary & further reading](#11-glossary--further-reading)

---

## 1. The problem: why containers exist

Before you learn *what* Docker is, you have to feel the pain it removes. Every one of these is a real wall developers hit, in roughly the order they hit them.

### 1.1 "It works on my machine"

You write code. It runs perfectly. You hand it to a teammate — or to a production server — and it explodes. Why?

Because your program is never *just* your program. It silently depends on a hundred things around it:

- The **language runtime** and its exact version (Node 24.14.0 vs Node 18 — different behavior, different bugs).
- **System libraries** (`glibc`, `libssl`, the fonts and graphics libraries a headless browser needs).
- **Environment variables**, config files, file paths, locale, timezone.
- Other **tools on the PATH** (a specific `npm`, a specific `nx`, a `sequelize-cli`).

Your machine has a particular blend of all of these, accumulated over months. Your teammate's machine has a *different* blend. Production has yet another. The code is identical; the *environment* is not. So the code behaves differently. This is the single most expensive time-sink in software, and it has a name: **environment drift**.

### 1.2 Dependency hell

Now scale that up. You have two projects on one machine. Project A needs Node 18 and PostgreSQL client v14. Project B needs Node 24 and PostgreSQL client v16. You can only have one "default" Node on your PATH at a time. You install version managers, you fight `PATH`, you break Project A while fixing Project B. Multiply by a team of ten and years of history, and you get **dependency hell**: a combinatorial explosion of "which versions of what, installed where, in which order."

### 1.3 Isolation and the blast radius

Two apps on one server share the same filesystem, the same ports, the same libraries. App A upgrades a shared library; App B, which depended on the old behavior, silently breaks. One app leaks memory and starves the other. A bug in one process can read another's files. There is no wall between them. You want each app to run as if it owns the whole machine — without actually buying a whole machine per app.

### 1.4 Reproducibility

Six months from now a bug appears in production. You need to run *exactly* the build that's deployed — same code, same dependencies, same OS libraries — to reproduce it. If your build is "clone the repo, then `npm install` and hope the registry still serves the same versions, on whatever OS you happen to have," you cannot reproduce anything reliably. You need a build that is **frozen**: byte-for-byte the same every time, forever.

### 1.5 The heavyweight answer, and why it's too heavy

The old fix was the **virtual machine (VM)**: emulate an entire computer — its own kernel, its own full operating system — on top of your real one. VMs *do* solve isolation and reproducibility. But each VM boots a whole OS: gigabytes of disk, a minute to start, hundreds of megabytes of RAM overhead *before your app runs*. Running twenty microservices means twenty full operating systems. Wasteful and slow.

**Containers are the lightweight answer.** A container gives you *most* of the isolation of a VM — its own filesystem, its own process tree, its own network view — but it **shares the host's kernel** instead of booting its own. No emulated hardware, no second OS kernel. The result starts in milliseconds, weighs tens of megabytes, and you can run dozens on one laptop. Docker is the most common toolset for building and running containers.

```
        VIRTUAL MACHINES                        CONTAINERS
   ┌──────────┐ ┌──────────┐            ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  App A   │ │  App B   │            │  App A   │ │  App B   │ │  App C   │
   ├──────────┤ ├──────────┤            ├──────────┤ ├──────────┤ ├──────────┤
   │ Libs/Bins│ │ Libs/Bins│            │ Libs/Bins│ │ Libs/Bins│ │ Libs/Bins│
   ├──────────┤ ├──────────┤            └────┬─────┘ └────┬─────┘ └────┬─────┘
   │ Guest OS │ │ Guest OS │  ← heavy        └──────┬─────┴────────────┘
   ├──────────┤ ├──────────┤                 ┌──────┴───────┐
   │  Hypervisor (emulator) │                │ Docker Engine │  ← thin
   ├────────────────────────┤                ├──────────────┤
   │       Host OS          │                │   Host OS (shared kernel) │
   ├────────────────────────┤                ├──────────────────────────┤
   │       Hardware         │                │         Hardware         │
   └────────────────────────┘                └──────────────────────────┘
```

**The one-line takeaway:** a container is a way to package your app *together with everything it needs to run*, so it behaves identically on your laptop, your teammate's laptop, CI, and production — while staying light enough to run many at once.

---

## 2. Mental model: the lunchbox that packs the kitchen

Here's the analogy to hold onto for the rest of the chapter.

> An **image** is a lunchbox that packs not just the food, but the *entire kitchen that made it* — the stove, the exact ingredients, the recipe, the right knives. It is sealed, labeled, and frozen. Nothing about it changes. You can copy it a thousand times and every copy is identical.
>
> A **container** is what happens when someone *opens* a lunchbox and starts eating — a live, running instance. You can open the same lunchbox a hundred times and get a hundred independent meals. Each running meal can get messy (scribble notes, spill things), but none of that mess flows back into the sealed lunchbox. Throw the meal away and the lunchbox is still pristine, ready to make the next one.

Now map the analogy back to the real terms — memorize this table, it's the whole model:

| Analogy | Real Docker term | What it really is |
|---|---|---|
| The sealed lunchbox (food **+** kitchen) | **Image** | A frozen, read-only bundle of a filesystem + metadata: your app, its runtime, its OS libraries, and the command to start it. |
| Opening the box and eating | **Container** | A *running* (or stopped) instance of an image — a live process with its own isolated filesystem view, layered on top of the read-only image. |
| The recipe that builds the kitchen | **Dockerfile** | A plain-text script of steps that *bakes* an image. |
| The mess you make while eating | The container's **writable layer** | Changes a running container makes on top of the image. Discarded when the container is removed — the image is untouched. |
| The shelf where sealed boxes are stored & shared | **Registry** | A server that stores images so other machines can download ("pull") them. This repo uses Google **Artifact Registry**. |
| The label on the box ("lunch — v3") | **Tag** | A human-friendly name+version pointer to an image, e.g. `gibp-base:latest`. |
| The tamper-proof serial number | **Digest** | A cryptographic hash (`sha256:…`) that identifies the *exact* image bytes. Unlike a tag, it can never point to different contents. |

Two properties fall straight out of this model and explain almost everything Docker does:

1. **An image is immutable.** Once built, it never changes. That's what makes it reproducible — the thing you tested is *exactly* the thing that runs in production.
2. **A container is disposable.** It's cheap to create and destroy. You don't "fix" a running container; you fix the Dockerfile, rebuild the image, and start a fresh container. Containers are cattle, not pets.

---

## 3. Core concepts & vocabulary

You'll meet these words constantly. Read the table once now; you'll return to it.

| Term | One-line definition | Why you care |
|---|---|---|
| **Image** | A read-only, layered filesystem + metadata (start command, env, ports). The "lunchbox." | The unit you build, push, pull, and run. |
| **Layer** | One filesystem diff produced by one build step. Layers stack and are cached & shared. | Layer order decides your build speed and image size (§4). |
| **Container** | A running or stopped instance of an image, with a thin writable layer on top. | The thing that actually serves traffic. |
| **Dockerfile** | The recipe: an ordered list of instructions (`FROM`, `COPY`, `RUN`, …) that builds an image. | Everything you author lives here. |
| **Build context** | The set of files you send to the builder (usually the current directory), filtered by `.dockerignore`. | A bloated context = slow builds and accidental secret leaks. |
| **Registry / repository** | A server storing images (Artifact Registry, Docker Hub, GHCR). A *repository* is one named image line within it. | Where CI pushes and the cluster pulls. |
| **Tag** | A mutable pointer/label on an image: `name:tag`, e.g. `api:latest`, `api:c4b6ff6`. | Human-readable version; but tags can be moved (see `latest` is a lie, §10). |
| **Digest** | Immutable content hash: `name@sha256:…`. | The only way to name an *exact* image forever. |
| **Base image** | The image your `FROM` line starts from (`node:24.14.0-alpine`, `nginx:alpine`). | Determines your OS, size, and available system libraries. |
| **ARG** | A **build-time** variable, available only while building. Set with `--build-arg`. | Frontends pass `VITE_*` values in as ARGs (§6.2). *Not present at runtime.* |
| **ENV** | An **environment variable baked into the image**, present at runtime inside the container. | How the app reads config when running. |
| **ENTRYPOINT** | The command that *always* runs when the container starts; hard to override. | This repo's API uses it to run migrations then start the server. |
| **CMD** | The *default* command/arguments; easily overridden on `docker run`. | Frontend containers use it to start nginx. |
| **Volume / bind mount** | Storage that lives *outside* the container's disposable writable layer and survives restarts. | Databases keep their data in volumes so it isn't lost. |
| **Network** | A virtual network Docker creates so containers can find each other by name. | In compose, `api` reaches the DB at host `db`, not `localhost`. |
| **Multi-stage build** | One Dockerfile with several `FROM` stages, where a lean final stage copies only the artifacts it needs from a fat build stage. | The single most important size/security technique here (§4.6). |

A note on the daemon: `docker` (the command you type, the **client**) talks to a background service called the **Docker daemon** (`dockerd`), which does the actual building and running. On Docker Desktop (Mac/Windows) the daemon runs inside a small hidden Linux VM. This is why "Docker is running on my Mac" still means "a Linux kernel is running your Linux containers" — you just don't see the VM.

---

## 4. How it actually works

This is the section that turns you from "I can copy a Dockerfile" into "I understand why it's slow / big / broken." Read it slowly.

### 4.1 Layers and the union filesystem

An image is not one flat blob. It's a **stack of layers**, and *each instruction in your Dockerfile that changes the filesystem creates one new layer* — a diff on top of the layer below.

```dockerfile
FROM node:24.14.0-alpine     # layer 0: the whole base OS + Node
WORKDIR /app                 # metadata (tiny/no filesystem layer)
COPY package.json ./         # layer 1: adds one file
RUN npm install              # layer 2: adds node_modules/ (big!)
COPY . .                     # layer 3: adds your source
```

Docker stacks these with a **union filesystem** (OverlayFS on Linux): it merges all layers into a single view where upper layers win. Picture sheets of transparent film stacked on a projector — you see the combined image, but each sheet is a separate, reusable object underneath.

Two consequences make layers powerful:

- **Layers are shared.** If ten images all start `FROM node:24.14.0-alpine`, that base layer is stored **once** on disk and once in the registry. Pull the second image and Docker skips layers it already has.
- **Layers are cached.** This is the engine of fast builds — next section.

### 4.2 The build cache and why instruction order is everything

When Docker runs a build, for each instruction it asks: *"Have I built this exact instruction, on top of this exact parent layer, with these exact inputs, before?"* If yes, it reuses the cached layer instantly instead of re-running the step. This is the **build cache**.

The catch: **the moment one instruction's cache misses, every instruction after it must rebuild** — the cache invalidates downward, because each layer depends on the one before.

For `COPY`/`ADD`, the cache key includes the *contents* of the files copied. So this ordering is a disaster:

```dockerfile
# BAD — cache busts on every code change
FROM node:24.14.0-alpine
WORKDIR /app
COPY . .                # any source edit changes this layer...
RUN npm install         # ...so npm install re-runs EVERY build. Slow.
```

Every time you touch a single `.ts` file, the `COPY . .` layer changes, which busts the cache for `RUN npm install`, so you reinstall hundreds of packages for a one-line change.

The fix is to **order instructions from least-frequently-changed to most-frequently-changed**, and copy the dependency manifest *before* the source:

```dockerfile
# GOOD — dependencies cached until package.json actually changes
FROM node:24.14.0-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # changes rarely
RUN npm install                          # cached until manifests change
COPY . .                                 # changes often, but comes AFTER install
```

Now editing source only busts the cheap final `COPY`; `npm install` stays cached. **This exact principle is why `Dockerfile.base` in this repo copies `package.json`/`package-lock.json` and runs `npm install` before it copies anything else** — and why the whole base-image pattern exists (§7.1). Hold this idea; it's the "aha."

### 4.3 Tags vs digests: names that move vs names that don't

- A **tag** is a sticky note. `gibp-base:latest` means "whatever image someone last labeled `latest`." Tags are *mutable* — the same tag can point to different bytes tomorrow.
- A **digest** is a fingerprint: `gibp-base@sha256:9f2a…`. It's derived from the image's exact content, so it *cannot* be moved. Same digest ⇒ byte-identical image, guaranteed, forever.

Why this matters for a financial system: reproducibility and auditability. `:latest` is convenient but ambiguous; a digest (or an immutable tag like a git SHA) is a promise. This repo hedges: CI tags every image **twice** — once with the exact `:<git-sha>` (immutable-by-convention, the auditable one) and once with `:latest` (the convenient pointer). Kubernetes deployments reference the `:<git-sha>` form so every running pod traces back to one commit. More in [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

### 4.4 A container is just a process (namespaces & cgroups, briefly)

Here's the demystifying truth: a container is **not a mini-VM**. It's an ordinary Linux **process** on the host, wearing two pieces of costume the kernel provides:

- **Namespaces** give the process a *restricted view* of the system. A PID namespace makes the container think its process is PID 1 and it can't see the host's other processes. Mount, network, user, and hostname namespaces likewise wall off the filesystem, network interfaces, user IDs, and hostname. It *feels* like its own machine because it can't see anything else.
- **cgroups (control groups)** *limit and meter* what the process can consume — CPU, memory, I/O. This is how you cap a container at "1 CPU, 512 MB" and how Kubernetes enforces resource limits.

Because it's just a host process sharing the host kernel, it starts in milliseconds and has near-zero overhead — the payoff from §1.5. It also means container isolation is real but *not* as absolute as a VM's (they share a kernel), which is why the security practices in §8 matter.

### 4.5 The image start command: ENTRYPOINT vs CMD

When you `docker run` an image, Docker needs to know *what process to start*. Two instructions define that, and their interaction trips up everyone at first:

- **`ENTRYPOINT`** — the executable that *always* runs. Think "this container fundamentally *is* this program."
- **`CMD`** — the *default arguments* (or the default command if there's no `ENTRYPOINT`). Whatever you type after `docker run <image> …` **replaces `CMD`**.

Rules of thumb:

| You want… | Use |
|---|---|
| A container that always runs one program, optionally with default args | `ENTRYPOINT` (the program) + `CMD` (default args) |
| A default command that's easy to override ad-hoc | `CMD` alone |
| To run a wrapper script (e.g. "migrate, then start the app") | `ENTRYPOINT` pointing at the script |

Always use the **exec form** — a JSON array — not the shell form:

```dockerfile
ENTRYPOINT ["./entrypoint.sh"]     # exec form: runs the script directly as PID 1
CMD ["node", "dist/main.js"]       # exec form
# NOT: CMD node dist/main.js       # shell form → wraps in /bin/sh -c, breaks signals
```

The exec form makes your process **PID 1** directly, so it receives `SIGTERM` when Kubernetes stops the pod and can shut down gracefully. The shell form wraps your command in `/bin/sh -c "…"`, and the shell (not your app) becomes PID 1 and often swallows the signal — your app gets killed hard instead of draining cleanly. This repo uses the exec form everywhere; keep it that way.

### 4.6 Multi-stage builds, explained deeply

This is the technique that ties the chapter together, so we'll build up to it from the pain.

**The problem.** To *build* a modern app you need a fat toolchain: the compiler/bundler, dev dependencies, source files, caches. To *run* the built app you need almost none of that — just the runtime and the compiled output. If you build and run in the same image, you ship the entire toolchain to production: a 1.5 GB image full of things an attacker could use and that you have to store, transfer, and scan. Bloated and less secure.

**The fix.** A single Dockerfile can declare **multiple `FROM` stages**. Early stages are your messy "workshop" (build there, make a mess). The final stage is a clean, minimal "display case" that starts fresh and **copies only the finished artifacts** out of the workshop with `COPY --from=<stage>`. Everything in the earlier stages — compilers, dev deps, source, caches — is left behind and never ships.

```dockerfile
# ── Stage 1: the workshop (fat, has the whole toolchain) ──
FROM node:24.14.0-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install                 # ALL deps, including dev/build tooling
COPY . .
RUN npm run build               # produces ./dist

# ── Stage 2: the display case (lean, ships to prod) ──
FROM node:24.14.0-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm install --omit=dev      # runtime deps ONLY — no build tooling
COPY --from=build /app/dist ./dist   # copy JUST the compiled output
USER node
CMD ["node", "dist/main.js"]
```

```
   ┌─────────────── STAGE: build ───────────────┐
   │ node + npm + ALL deps + source + dist       │   (huge, thrown away)
   └───────────────────┬─────────────────────────┘
                       │ COPY --from=build /app/dist
                       ▼
   ┌─────────────── STAGE: runtime ──────────────┐
   │ node + prod deps + dist only                │   ← this is what ships
   └─────────────────────────────────────────────┘
```

The final image contains *only* the runtime stage's layers. The build stage never leaves the builder. That's how this repo turns a heavy Node/Nx/Playwright toolchain into a comparatively lean runtime image — and it's the shape of **every** production Dockerfile in the repo (§7).

---

## 5. Setup from scratch: zero to a running container

This section is a lab. Follow it on a scratch folder and you'll go from nothing to a running, pushed container. You need no prior Docker state.

### 5.1 Install Docker

**Linux (Ubuntu/Debian — the closest match to this repo's servers):** install Docker Engine from Docker's official apt repository (not the older `docker.io` package):

```bash
# Remove any stale packages, then install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Run docker without sudo (log out/in afterward for it to take effect)
sudo usermod -aG docker "$USER"
```

**macOS / Windows:** install **Docker Desktop** from <https://docs.docker.com/desktop/>. On Windows, use the **WSL 2** backend (Docker Desktop will prompt you). Docker Desktop bundles the daemon (inside a lightweight Linux VM), the CLI, Buildx, and Compose.

**Verify the install** (this is your "it works" checkpoint):

```bash
docker --version                 # prints a version, e.g. "Docker version 27.x"
docker run --rm hello-world      # pulls a tiny image and prints a success message
```

If `hello-world` prints "Hello from Docker!", your client, daemon, registry pull, and container run all work. `--rm` auto-deletes the container when it exits so you don't accumulate junk.

### 5.2 `docker run` basics

`docker run` = "create a container from an image and start it." The flags you'll use constantly:

```bash
# Run an interactive shell in an Alpine container; -it = interactive + TTY; --rm = clean up on exit
docker run --rm -it alpine sh

# Run nginx in the background (-d = detached), map host port 8080 → container port 80, name it
docker run -d -p 8080:80 --name web nginx:alpine
#            │    │            │        └ friendly name for later commands
#            │    │            └ PUBLISH: hostPort:containerPort
#            │    └ run in background, print the container id
#            └ (create + start)

# Pass an environment variable at runtime
docker run --rm -e GREETING=hello alpine sh -c 'echo $GREETING'

# Mount a host folder into the container (bind mount): host:container
docker run --rm -v "$PWD":/data alpine ls /data
```

Now visit <http://localhost:8080> — nginx answers. Everyday container management:

```bash
docker ps                 # list RUNNING containers
docker ps -a              # list ALL containers (including stopped)
docker logs web           # print a container's stdout/stderr
docker logs -f web        # ...and follow (stream) new output
docker exec -it web sh    # open a shell INSIDE a running container
docker stop web           # graceful stop (SIGTERM, then SIGKILL after grace)
docker rm web             # delete the stopped container
docker images             # list local images
docker rmi nginx:alpine   # delete a local image
```

### 5.3 Your first Dockerfile

Make a folder with a trivial Node app:

```bash
mkdir docker-lab && cd docker-lab
```

`server.js`:

```javascript
const http = require('http');
const port = process.env.PORT || 3000;
http
  .createServer((_req, res) => res.end(`Hello from a container on port ${port}\n`))
  .listen(port, () => console.log(`listening on ${port}`));
```

`Dockerfile`:

```dockerfile
# 1. Start from a small official Node image, pinned to an exact version
FROM node:24.14.0-alpine

# 2. All following paths are relative to /app inside the image
WORKDIR /app

# 3. Copy the app in (no dependencies in this toy example)
COPY server.js ./

# 4. Document the port the app listens on (informational; does not publish it)
EXPOSE 3000

# 5. Default runtime command
CMD ["node", "server.js"]
```

### 5.4 Build, tag, run, exec, logs

```bash
# Build: -t names+tags the image; the final "." is the BUILD CONTEXT (this folder)
docker build -t my-app:1.0 .

# Run it, mapping host 3000 → container 3000
docker run -d -p 3000:3000 --name my-app my-app:1.0

# Verify it works
curl localhost:3000            # → Hello from a container on port 3000

# Peek inside and inspect
docker logs my-app             # → listening on 3000
docker exec -it my-app sh      # you're now inside the container
#   / # ls /app                → server.js
#   / # exit

# Clean up
docker rm -f my-app
```

You just built an image, ran it, entered it, and read its logs. That loop — **edit Dockerfile → `docker build` → `docker run` → verify → `docker rm`** — is 90% of the job.

### 5.5 A real multi-stage Node build

The toy above ships source directly. A real app *compiles* first. Here's the production shape (this mirrors what the repo does, minus the monorepo machinery):

```dockerfile
# ── build stage ──
FROM node:24.14.0-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                       # reproducible install from the lockfile
COPY . .
RUN npm run build                # e.g. tsc/vite/nest build → ./dist

# ── runtime stage ──
FROM node:24.14.0-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev            # runtime deps only
COPY --from=build /app/dist ./dist
USER node                        # drop root (see §6.4)
CMD ["node", "dist/main.js"]
```

> **`npm ci` vs `npm install`:** `npm ci` installs *exactly* what `package-lock.json` specifies and errors if the lockfile is out of sync — deterministic, ideal for images. `npm install` may update the lockfile. (This repo's Dockerfiles happen to use `npm install`/`npm i`; `npm ci` is the stricter general best practice for reproducible builds.)

### 5.6 `.dockerignore` — keep junk out of the build context

Remember §4.2: `docker build .` sends the **entire folder** (the build context) to the daemon. If that folder contains `node_modules/` (hundreds of MB), `.git/`, build outputs, and `.env` secrets, you get: slow builds, giant contexts, busted caches, and **secrets accidentally copied into layers**.

`.dockerignore` (same syntax as `.gitignore`) excludes paths from the context *before* they're sent:

```gitignore
node_modules
dist
.git
.env
.env.*
!.env.example
*.log
coverage
```

Rule of thumb: anything the build *regenerates* (`node_modules`, `dist`) or must *never see* (`.env`, keys) belongs here. This repo's real `.dockerignore` is dissected in §7.5.

### 5.7 Pushing to a registry

A local image is useless to a server that can't reach your laptop. You **push** it to a **registry** so machines can **pull** it.

**Generic (Docker Hub):**

```bash
docker login                                   # authenticate once
docker tag my-app:1.0 yourname/my-app:1.0      # give it a registry-qualified name
docker push yourname/my-app:1.0                # upload
```

**This repo — Google Artifact Registry.** Images live at `us-west1-docker.pkg.dev/gibp-ledger/gibp-apps`. The host prefix tells Docker which registry; the path is `<project>/<repo>/<image>`.

```bash
# One-time: let Docker authenticate to Artifact Registry using gcloud credentials
gcloud auth configure-docker us-west1-docker.pkg.dev

# Tag with the full registry path. This repo tags twice: an immutable git-sha and latest.
GIT_SHA=$(git rev-parse --short HEAD)
docker tag api:local us-west1-docker.pkg.dev/gibp-ledger/gibp-apps/api:${GIT_SHA}
docker tag api:local us-west1-docker.pkg.dev/gibp-ledger/gibp-apps/api:latest

# Push both tags
docker push us-west1-docker.pkg.dev/gibp-ledger/gibp-apps/api:${GIT_SHA}
docker push us-west1-docker.pkg.dev/gibp-ledger/gibp-apps/api:latest
```

> In practice **you almost never run these by hand** — CI (GitHub Actions) builds and pushes on every merge, authenticating via Workload Identity Federation rather than a static key. The commands above are for understanding and the rare manual push. See [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

### 5.8 Image size & a first look at optimization

Check your image sizes:

```bash
docker images
docker history my-app:1.0     # per-layer size breakdown — find the fat layer
```

Quick wins, in rough order of impact:

1. **Multi-stage build** — leave the toolchain behind (§4.6). Biggest single win.
2. **A small base image** — `node:24.14.0-alpine` (~150 MB) vs `node:24` (~1 GB). Choose deliberately (§6.5).
3. **`--omit=dev`** in the runtime stage — don't ship dev dependencies.
4. **Good `.dockerignore`** — don't copy `node_modules`/`.git` into the context.
5. **Fewer, ordered layers** — combine related `RUN` steps; put volatile steps last.

---

## 6. Using it well: patterns that matter

### 6.1 Multi-stage patterns

You've seen the two-stage build→runtime split. Two more moves show up in this repo:

- **A shared base stage.** When many images share the same expensive setup (installing all monorepo dependencies), extract it into its *own* image and `FROM` it. That's `gibp-base` — see §6.6 and §7.1.
- **Copy only what the runtime needs.** The final stage should `COPY --from=builder` the *narrowest* set of artifacts: the compiled `dist`, the pruned `node_modules`, and specific config files — nothing else. The API's runtime stage copies exactly those, plus its migrations and entrypoint, and nothing from the source tree.

### 6.2 Build-time vs runtime config — the single most important teaching point

This trips up *everyone*, and getting it wrong ships broken frontends or leaks secrets. Internalize this:

> **`ARG` exists only while the image is being built. `ENV` exists while the container is running.** They are two different worlds. A value you pass as `--build-arg` is **gone** by the time the container starts — unless the build *copied it into the artifact* or promoted it into an `ENV`.

Now the crucial split between frontends and backends:

**Frontend (React/Vite) — config is baked at BUILD time.** A browser app is just static files (HTML/JS) served to a user's browser. The browser can't read your server's environment. So Vite **inlines** every `VITE_*` variable into the JavaScript *at build time* — literally find-and-replaces `import.meta.env.VITE_API_URL` with the string value and freezes it into the bundle. That means:

```dockerfile
FROM gibp-base:latest AS builder
ARG VITE_API_URL                       # received from the build command / CI
ENV VITE_API_URL=$VITE_API_URL         # promote ARG → ENV so Vite's build sees it
RUN npx nx build company-webapp --prod # the value is now HARD-CODED into the JS
```

Consequences you must respect:
- Changing `VITE_API_URL` requires a **rebuild** — you cannot flip it at runtime.
- The value is **public** — it ships in JS to every browser. **Never put a secret in a `VITE_*` var.** (An API *URL* is fine; an API *key* is not.)

**Backend (NestJS/Node) — config is read at RUNTIME.** A server process can read its own environment when it starts. So backend config (`DB_HOST`, `DB_PASSWORD`, `FORMANCE_API_KEY`) is **not** baked into the image. The image is config-agnostic; the same image byte-for-byte runs in staging and production, and the environment (injected by Kubernetes / compose) decides its behavior:

```bash
docker run -e DB_HOST=... -e DB_PASSWORD=... us-west1-docker.pkg.dev/.../api:<sha>
```

This is also *why the same backend image is portable and reusable* while a frontend image is environment-specific. Memorize the table:

| | Frontend (Vite) | Backend (Node) |
|---|---|---|
| When is config resolved? | **Build time** (baked into JS) | **Run time** (read from env) |
| Mechanism | `ARG` → `ENV` → inlined by Vite | `process.env.*` read on startup |
| Change a value → | **Rebuild the image** | Restart with new env; same image |
| Is one image reusable across envs? | No (env-specific) | Yes (env-agnostic) |
| Safe to hold a secret? | **No** — ships to browsers | Yes — stays server-side |

### 6.3 Running migrations from an entrypoint

A backend usually must **migrate the database schema before it serves traffic**. The clean pattern: point `ENTRYPOINT` at a small shell script that (1) validates required env vars, (2) runs migrations, (3) `exec`s the app. This repo's API does exactly this — see §7.2. Two rules:

- **Fail fast and loud** on missing config (`exit 1`), so a misconfigured deploy dies immediately instead of half-working.
- Use `exec node …` as the *last* line so Node **replaces** the shell and becomes PID 1 — inheriting signal handling for graceful shutdown (§4.5).

### 6.4 Run as a non-root user

By default a container runs as **root**. If an attacker escapes your app, root inside the container is a far bigger blast radius (and shares the host kernel). Create/So use the unprivileged `node` user that official Node images already provide, and switch to it near the end:

```dockerfile
COPY --chown=node:node ... ...   # make copied files owned by node
USER node                        # everything after runs as non-root
```

Order matters: put `USER node` **after** the steps that need root (installing OS packages, `chmod`), and ensure files the app must read are `--chown`ed to `node`. Every runtime stage in this repo ends with `USER node` (the frontends run nginx, which drops privileges for its workers itself).

### 6.5 Choosing a base image: alpine vs bookworm-slim

Your `FROM` line picks your OS, and it's a real tradeoff:

| Base | Size | libc | Best for | Watch out |
|---|---|---|---|---|
| `alpine` | Tiny (~5 MB OS) | **musl** libc | Pure-JS services, static file serving (nginx) | Some native/prebuilt binaries expect **glibc** and break on musl; missing system libs for heavy tools |
| `bookworm-slim` (Debian 12, slimmed) | Bigger (~75 MB OS) | **glibc** | Anything needing broad system libraries or prebuilt native binaries | Larger image |
| Full `debian`/`node:24` | Large (~350 MB+ OS) | glibc | When you truly need the full toolchain at runtime | Big, more surface area |

The decision rule: **alpine by default for size; switch to `bookworm-slim` (glibc) the moment you need a heavy binary that expects glibc and a full set of system libraries.** This repo lives out both sides of that rule — the frontends and worker use alpine, but the **API's runtime uses `node:24-bookworm-slim`** specifically because it runs **Playwright/Chromium** (to render report PDFs), and headless Chromium needs a pile of glibc-based system libraries that Alpine doesn't ship. See §7.2 — this is the most important base-image decision in the codebase.

### 6.6 The shared base-image pattern

If every app's build starts by installing the same hundreds of dependencies, you pay that cost N times and your CI crawls. The fix: build the expensive shared setup **once** into its own image, tag it, and have every app `FROM` it. That's `gibp-base` (§7.1). The payoff is enormous in a monorepo; the cost is one extra image to build and keep current. It's the base-image pattern from §4.1 applied at the project level.

### 6.7 Healthchecks

A running container isn't necessarily a *healthy* one — Node can be up but wedged. A `HEALTHCHECK` (or, in Kubernetes, a liveness/readiness probe) lets the orchestrator know:

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

In this repo the *application* Dockerfiles mostly delegate health to Kubernetes probes and to `docker-compose` service `healthcheck:` blocks (you can see rich examples in `docker-compose.yml` — Postgres uses `pg_isready`, Formance curls `/_healthcheck`). The concept is the same: define a cheap command that returns non-zero when the service is unusable.

---

## 7. How THIS repo uses Docker

Now we read the real thing. Four production Dockerfiles, one dev Dockerfile, and the `.dockerignore` — with the *why* behind each choice. All apps target **Node 24.14.0** and build via **Nx** (the monorepo build tool).

The shape is consistent across the repo:

```
                    Dockerfile.base  →  gibp-base:latest   (deps installed ONCE, in CI)
                          │
      ┌───────────────────┼────────────────────┬─────────────────────┐
      ▼                   ▼                    ▼                     ▼
 api/Dockerfile.prod  company-webapp/     super-webapp/         kafka-worker/
 builder: FROM base   Dockerfile.prod     Dockerfile.prod       Dockerfile.prod
 runtime: bookworm    builder: FROM base  builder: FROM base    builder: FROM base
   + Playwright       runtime: nginx      runtime: nginx        runtime: alpine
   + migrations       (bakes VITE_*)      (bakes VITE_*)        (node dist/main.js)
```

### 7.1 `Dockerfile.base` — install dependencies once

```dockerfile
ARG NODE_IMAGE=node:24.14.0-alpine
FROM ${NODE_IMAGE}
WORKDIR /app

# 1. Copy package manifests
COPY package.json package-lock.json ./

# 2. Install dependencies
RUN npm install

# 3. Copy shared libraries and NX config
COPY packages/ ./packages/
COPY nx.json tsconfig.base.json ./

# 4. Install NX globally
RUN npm install -g nx@22.5.2
```

**Why it exists.** Every app in the monorepo needs the same big `node_modules`. Installing it inside each app's build would repeat the most expensive step four times. Instead this image installs *all* dependencies **once**, is tagged `gibp-base:latest` in CI, and every app's `Dockerfile.prod` starts `FROM gibp-base:latest`. The expensive layer is built once and shared.

**Why this exact order.** Straight application of the cache lesson (§4.2): manifests first (`COPY package.json package-lock.json`), then `npm install`, *then* the source-ish files (`packages/`, config). Dependencies stay cached until `package.json`/`package-lock.json` actually change. The `ARG NODE_IMAGE` makes the base image pinnable/overridable, but defaults to the repo-wide `node:24.14.0-alpine`.

> **An honest nuance:** the base image installs **`nx@22.5.2` globally**, while the repo's `package.json` currently pins `nx` at `22.5.0` in devDependencies. In practice Nx is invoked with `npx nx …` inside the app builds (which prefers the local version), so the global pin is a convenience; if you're chasing an Nx version mismatch, this is a place to look.

### 7.2 `apps/api/Dockerfile.prod` — builder + bookworm-slim runtime + Playwright + migrations

This is the most involved and most instructive Dockerfile in the repo. Reading it top to bottom:

```dockerfile
# Extend from the unified base image built in CI
FROM gibp-base:latest AS builder
WORKDIR /app

# Only copy the specific API codebase to preserve cache
COPY apps/api ./apps/api
COPY packages ./packages

# Build the NestJS API application bundle (Nx builds package deps first via ^build)
RUN NX_DAEMON=false npx nx build api --prod --skip-sync

# Production Environment container with migrations support (bookworm-slim for Playwright Chromium)
FROM node:24-bookworm-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production

# Install browsers to a shared, user-independent path so the (root) build-time install and the
# (node) runtime launch resolve the same location — otherwise the browser lands in /root/.cache
# while the app, running as node, looks in /home/node/.cache and fails to launch.
ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright

# Copy node_modules first so the Chromium install below uses the project's pinned Playwright version
COPY --from=builder --chown=node:node /app/node_modules ./node_modules

# Headless Chromium for report PDF export — runs the project's own Playwright binary
RUN ./node_modules/.bin/playwright install --with-deps chromium

# Copy compiled application with migrations
COPY --from=builder --chown=node:node /app/apps/api/dist ./dist

# Copy database config / constants / app config / migrations (needed by sequelize-cli)
COPY --chown=node:node apps/api/src/database/config     ./dist/database/config
COPY --chown=node:node apps/api/src/database/constants  ./dist/database/constants
COPY --chown=node:node apps/api/src/config              ./dist/config
COPY --chown=node:node apps/api/src/database/migrations ./dist/database/migrations

# Copy production entrypoint script
COPY --chown=node:node apps/api/docker-entrypoint-prod.sh ./entrypoint.sh
RUN chmod +x ./entrypoint.sh

USER node
ENTRYPOINT ["./entrypoint.sh"]
```

Decision-by-decision:

- **`FROM gibp-base:latest AS builder`** — reuses the pre-installed dependencies (§7.1). The builder only needs to add the API source and compile.
- **`COPY apps/api` + `COPY packages` (not `COPY .`)** — copies the *narrowest* input, so an unrelated change elsewhere in the monorepo doesn't bust this image's cache. (§4.2 again.)
- **`NX_DAEMON=false npx nx build api --prod --skip-sync`** — one-shot build with no Nx daemon (pointless in a throwaway build container) and `--skip-sync` to skip Nx's TS project-reference sync step. Nx compiles the package dependencies first (`^build`) then the API into `apps/api/dist`.
- **`FROM node:24-bookworm-slim AS runtime`** — the pivotal base-image choice. The API renders **PDF reports with Playwright/Chromium**, and headless Chromium needs a set of **glibc-based system libraries** that Alpine's musl world doesn't provide. `bookworm-slim` (slim Debian, glibc) supplies them at a fraction of full-Debian size. This is the concrete case of the §6.5 rule.
- **`ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright`** — subtle but important. Chromium is *installed* as root (build time) but *launched* as the `node` user (run time). Without a fixed path, the install lands in `/root/.cache` while `node` looks in `/home/node/.cache` and Chromium fails to launch. Pinning a shared, user-independent path makes both resolve the same location.
- **`COPY node_modules` before `playwright install`** — so `./node_modules/.bin/playwright` is the **project's pinned** Playwright version. Installing the matching browser from the project's own binary keeps browser and library in lockstep (single source of truth from `package.json`).
- **`playwright install --with-deps chromium`** — downloads the exact Chromium build *and* apt-installs its system dependencies (`--with-deps`). Chromium only, not all three browsers — smaller image.
- **The four `COPY … ./dist/…` lines** — Nx compiles the TypeScript, but `sequelize-cli` needs some files *as-is* at runtime: the DB config, its constants, app config, and the migration files. They're copied into `dist/` where the entrypoint expects them. (Migrations are read from `dist/database/migrations`.)
- **entrypoint copied + `chmod +x`** — makes the script executable inside the image.
- **`USER node`** then **`ENTRYPOINT ["./entrypoint.sh"]`** — drop root (§6.4), and always run the migrate-then-start script (§6.3, §4.5). Note the exec-form JSON array.

### 7.3 `apps/api/docker-entrypoint-prod.sh` — migrate before serving

```sh
#!/bin/sh
set -e
echo "[Entrypoint] Starting GIBP API migration and startup..."

# Verify required environment variables are set (fail fast if the deploy is misconfigured)
if [ -z "$DB_HOST" ] || [ -z "$DB_PORT" ] || [ -z "$DB_NAME" ] || [ -z "$DB_USER" ] || [ -z "$DB_PASSWORD" ]; then
  echo "[Entrypoint] ERROR: Missing required database environment variables"
  exit 1
fi

echo "[Entrypoint] Running database migrations..."
node -r tsconfig-paths/register -r @swc-node/register \
  node_modules/sequelize-cli/lib/sequelize db:migrate \
  --env development \
  --config dist/database/config/sequelize-cli.config.js \
  --migrations-path dist/database/migrations || {
  echo "[Entrypoint] ERROR: Database migrations failed"
  exit 1
}

echo "[Entrypoint] Database migrations completed successfully"
echo "[Entrypoint] Starting GIBP API server..."
exec node dist/main.js
```

What to notice:

- **`set -e`** — abort on the first error, so a failed step can't be silently skipped.
- **Env-var validation** — the container refuses to start without the `DB_*` values. Backend config is read at **runtime** (§6.2), and this is the guardrail that turns a misconfigured deploy into an immediate, obvious failure rather than a half-working service.
- **Config comes from the environment, not an `.env` file** — no `--env-file` flag; the values arrive from Kubernetes/compose at runtime. (`.env` files are excluded from the image on purpose — see §7.5.)
- **Migrations run *before* the app serves** — the schema is guaranteed current before the first request. (`--env development` here selects a *sequelize-cli config profile name*, not the app's environment; the actual connection values still come from the `DB_*` env vars. Don't let the profile name confuse you.)
- **`exec node dist/main.js`** — `exec` replaces the shell so Node becomes PID 1 and receives `SIGTERM` for graceful shutdown (§4.5).

### 7.4 `apps/company-webapp/Dockerfile.prod` — bake `VITE_*` args, serve with nginx

```dockerfile
FROM gibp-base:latest AS builder

# Declare the variables passed in securely from the GitHub Action during compilation
ARG VITE_API_URL
ARG VITE_FORMANCE_PUBLIC_URL

# Load them into the actual environment so Vite detects and embeds them
ENV VITE_API_URL=$VITE_API_URL
ENV VITE_FORMANCE_PUBLIC_URL=$VITE_FORMANCE_PUBLIC_URL

# Copy the frontend code
COPY apps/company-webapp ./apps/company-webapp
COPY packages ./packages

# Build the frontend, which will embed the ENV variables into the static javascript
RUN npx nx build company-webapp --prod

# Lean Nginx Web Server for static hosting
FROM nginx:alpine
COPY --from=builder /app/apps/company-webapp/dist /usr/share/nginx/html
COPY --from=builder /app/apps/company-webapp/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

This is the §6.2 lesson made concrete:

- **`ARG VITE_API_URL` → `ENV VITE_API_URL=$VITE_API_URL` → `nx build`.** The build-time ARG is promoted to an ENV so the Vite build *sees* it, and Vite **inlines** it into the static JavaScript. After the build the value is frozen into the bundle. Change it ⇒ rebuild. These are non-secret URLs (they ship to every browser) — which is exactly why they're safe to bake, and why a secret must **never** go here.
- **Runtime is `nginx:alpine`, not Node.** The output is static files; you don't need Node to serve them. A tiny nginx serves `/usr/share/nginx/html`. The multi-stage split means the entire Node/Nx toolchain is left behind — the shipped image is just nginx + a folder of static assets.
- **`nginx.conf`** does SPA routing — `try_files $uri $uri/ /index.html` so client-side routes resolve to the app shell. `EXPOSE 80` documents the port; `CMD ["nginx", "-g", "daemon off;"]` keeps nginx in the foreground (a container must run a foreground process or it exits).

`apps/super-webapp/Dockerfile.prod` is byte-for-byte the same pattern for the other front-end — same `VITE_*` bake, same nginx runtime.

### 7.5 The dev `apps/api/Dockerfile` — simpler, for compose

CI ships the `.prod` variants; **local `docker-compose` uses the plain `Dockerfile`s**, which are simpler because they don't need Playwright, migrations-in-entrypoint, or the base image:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24.14.0-alpine AS build
WORKDIR /app
COPY package.json package-lock.json nx.json tsconfig.base.json tsconfig.json ./
COPY apps ./apps
COPY packages ./packages
RUN npm i
RUN NX_DAEMON=false npx nx build api --skip-sync

FROM node:24.14.0-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm i --omit=dev --ignore-scripts && npm cache clean --force
COPY --from=build --chown=node:node /app/apps/api/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

Differences from the prod variant, and why:

- **Self-contained** — it does its *own* `npm i` instead of `FROM gibp-base`, so a developer can build the API without first building the base image.
- **Alpine runtime, no Playwright** — local dev doesn't render PDFs, so it skips Chromium and stays on tiny Alpine.
- **`npm i --omit=dev --ignore-scripts && npm cache clean --force`** — runtime deps only, skip lifecycle scripts, wipe the npm cache to shrink the layer.
- **`CMD ["node", "dist/main.js"]`, no entrypoint** — no migration step; simpler for a laptop.

`apps/kafka-worker/Dockerfile.prod` follows the same builder→alpine-runtime shape as the frontends' *builder*, but its runtime is plain `node:24.14.0-alpine` running `node dist/main.js` (no browser, no HTTP server, so no nginx and no Playwright).

### 7.6 The `.dockerignore`

```gitignore
node_modules
.nx
dist
tmp
coverage
playwright-report
test-results
npm-debug.log
*.log
.env
.env.*
!.env.example
.DS_Store
```

Why each entry earns its place:

- **`node_modules`, `.nx`, `dist`, `tmp`, `coverage`, `playwright-report`, `test-results`** — build/test outputs and caches. They're *regenerated* inside the image; sending them only bloats the context and can leak stale artifacts into a layer. (`node_modules` in particular is hundreds of MB and would murder build times — §4.2.)
- **`.env` and `.env.*`** — **secrets**. Excluding them guarantees credentials can't be accidentally `COPY`'d into a layer, where they'd be extractable by anyone who pulls the image (§8, §9). This is a security control, not a nicety.
- **`!.env.example`** — the leading `!` *un-ignores* one file: the committed, secret-free template stays available.
- **`*.log`, `.DS_Store`** — noise.

### 7.7 Where the images go from here

Once built and tagged, images push to **Artifact Registry** (`us-west1-docker.pkg.dev/gibp-ledger/gibp-apps`), each with a `:<git-sha>` and a `:latest` tag, and GKE pulls the `:<git-sha>` form so every pod traces to one commit. *Who* runs the build, how the `gibp-base` image is produced in CI, how tags flow into Kubernetes manifests, and how ArgoCD rolls it out — all of that is the deployment pipeline, covered in **[`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md)** (see its Part 3 "The whole journey" and §4.1 "Docker"). This chapter deliberately stops at the image boundary.

---

## 8. Production concerns

### 8.1 Image size

Smaller images pull faster (quicker deploys and autoscaling), cost less to store, and expose less attack surface. The levers, in order of impact, are the same as §5.8: multi-stage builds, a small base, `--omit=dev`, a tight `.dockerignore`, and fewer/ordered layers. The API is the intentional exception — it's larger *because* it carries Chromium, and that's a justified tradeoff for PDF rendering, not neglect.

### 8.2 Layer caching in CI

Locally the build cache lives on your disk. In CI, each run may start on a **fresh machine with an empty cache**, so builds can be slow unless the cache is preserved. The two mitigations this repo's architecture leans on:

- **The `gibp-base` image** — the most expensive layer (installing all dependencies) is built once and reused by every app build, so per-app builds only compile source.
- **Registry-backed cache / Buildx** — CI can pull prior layers from the registry (`--cache-from`) so unchanged steps are reused across runs. (Pipeline specifics live in the deployment guide.)

The universal rule still applies: **order Dockerfile instructions so the rarely-changing, expensive steps come first** and get cached; volatile source copies come last.

### 8.3 Security

- **Non-root.** Every runtime stage ends `USER node` (§6.4). Never ship a container that runs your app as root.
- **Minimal base.** Alpine / `bookworm-slim` carry far fewer packages than a full OS — fewer CVEs to patch, smaller attack surface. Choose the smallest base that has the libraries you need.
- **No secrets in layers.** *Layers are permanent and extractable.* A secret `COPY`'d in, or a `RUN` that echoes a token, stays in the image history **even if a later layer deletes the file** — anyone with the image can `docker history`/extract it. Keep secrets out via `.dockerignore` (§7.6), inject them at **runtime** as env vars (backends) or Kubernetes secrets, and use **BuildKit secret mounts** (`RUN --mount=type=secret,...`) if a build genuinely needs a credential — those never land in a layer. **Never** bake a secret into a `VITE_*` var (it ships to browsers — §6.2).
- **`.dockerignore` as a control.** It's your first line of defense against leaking `.env` and keys into the context.
- **Scanning.** Scan images for known vulnerabilities before they ship — `docker scout cves <image>`, `trivy image <image>`, or Artifact Registry's built-in scanning. Rebuild on base-image security updates so patches actually reach production.

### 8.4 Reproducibility

- **Pin tags; prefer digests.** `FROM node:24.14.0-alpine` beats `FROM node:alpine` (which drifts). For the strongest guarantee, pin by digest: `FROM node:24.14.0-alpine@sha256:…`. This repo pins exact versions across the board (Node 24.14.0, `nx@22.5.2`) and deploys images by `:<git-sha>`, so a running pod is always traceable to exact bytes.
- **Lockfiles.** `package-lock.json` + `npm ci` makes dependency resolution deterministic (the repo uses `npm install`; `npm ci` is the stricter option).

### 8.5 Multi-arch (brief)

Apple Silicon laptops are `arm64`; most cloud nodes are `amd64` (`x86_64`). An image built for one arch won't run on the other. If you build on an M-series Mac and deploy to `amd64` nodes, either build multi-arch manifests with `docker buildx build --platform linux/amd64,linux/arm64 …` (which publishes one tag that resolves to the right arch per host), or ensure CI builds for the target arch. This repo's CI builds for the cluster's architecture, so you rarely handle this by hand — but it's the reason an image that "runs on my Mac" can be rejected by the cluster.

---

## 9. Operations & debugging playbooks

Concrete, copy-pasteable recipes for the situations you'll actually hit.

### 9.1 The everyday command set

```bash
docker build -t name:tag .            # build from ./Dockerfile using . as context
docker build -f path/to/Dockerfile .  # use a non-default Dockerfile
docker run -d -p 8080:80 name:tag     # run detached, publish a port
docker exec -it <container> sh        # shell into a running container
docker logs -f <container>            # stream logs
docker inspect <container|image>      # full JSON: env, mounts, network, entrypoint
docker history <image>                # per-layer breakdown (find the fat/leaky layer)
docker system df                      # how much disk images/containers/cache use
docker system prune                   # delete stopped containers, unused nets, dangling images
docker system prune -a --volumes      # AGGRESSIVE: also removes unused images + volumes (careful!)
```

### 9.2 "My build is slow"

Almost always a **cache-order** problem (§4.2).

1. Run `docker build` and watch which step stops printing `CACHED`. Everything from there down is rebuilding.
2. Is an expensive step (`npm install`) *after* a volatile `COPY . .`? Move the manifest copy + install **above** the source copy.
3. Is `node_modules`/`.git` being sent in the context? Add them to `.dockerignore` — check with `docker build` output's "transferring context" size.
4. In this repo: is your app's `Dockerfile.prod` correctly starting `FROM gibp-base:latest`? If the base is stale or missing, you're reinstalling deps from scratch.

### 9.3 "My image is too big"

```bash
docker images                 # spot the offender's size
docker history <image>        # which layer is fat?
```

Then: confirm you have a **multi-stage** build (are you shipping the build stage?); confirm the runtime stage uses `--omit=dev`; confirm the base is Alpine/slim not full; confirm `.dockerignore` excludes `node_modules`/caches. If Chromium is the weight (the API), that's expected.

### 9.4 "It works locally but not in the container"

The classic environment-drift ambush — and usually one of these:

- **A missing env var.** Backends read config at runtime (§6.2). `docker exec -it <c> env` and compare to what the app expects. The API's entrypoint *validates* `DB_*` and exits loudly — read its log line.
- **`localhost` means the container, not the host.** Inside a container, `localhost` is that container. To reach the DB, use the service name on the Docker network (`db`), not `localhost`. (Compose sets this up.)
- **A missing system library.** Symptom: a native binary or Chromium fails to launch. This is the alpine-vs-glibc trap (§6.5) — the reason the API runtime is `bookworm-slim`.
- **A file that exists locally but was `.dockerignore`d** out of the image. `docker exec -it <c> ls <path>` to confirm it's actually present.
- **Stale frontend config.** A `VITE_*` value is baked at build (§6.2); if it's wrong, rebuilding with the right `--build-arg` is the only fix — you can't change it at runtime.

Fast triage: `docker run --rm -it <image> sh` (or `docker exec` into the running one) and *look around* — check `env`, `ls dist`, try to start the process manually and read the error.

### 9.5 "A secret leaked into a layer"

1. **Assume it's compromised.** Layers are permanent; a later `RUN rm` does **not** remove the secret from earlier layers, and anyone with the image can extract it. **Rotate the secret immediately.**
2. Find it: `docker history --no-trunc <image>` and inspect layer contents.
3. Fix the cause: add the file to `.dockerignore`; move the value to a **runtime** env var or Kubernetes secret; if a build truly needs it, use a BuildKit secret mount (`RUN --mount=type=secret,id=...`) which never persists into a layer.
4. Rebuild clean and, if it was pushed, delete the compromised image from the registry.

### 9.6 Cleaning up disk

Docker hoards images, stopped containers, and build cache. When your disk fills:

```bash
docker system df                  # see what's using space
docker container prune            # remove stopped containers
docker image prune                # remove dangling (untagged) images
docker builder prune              # clear build cache
docker system prune -a --volumes  # nuke everything unused (double-check first!)
```

---

## 10. Gotchas & hard-won lessons

- **`ARG` is not available at runtime.** A `--build-arg` value evaporates when the build ends. If the running container needs it, either promote it (`ENV FOO=$FOO`, as the frontends do for `VITE_*`) or bake it into the artifact at build. Backends should read runtime env, not ARGs.
- **`COPY` busts the cache by *content*.** Copying source before installing dependencies makes every code change reinstall everything (§4.2). Copy manifests → install → then source. This is the #1 cause of slow builds.
- **`latest` is a lie.** A tag is a movable pointer; `:latest` can mean different bytes tomorrow. For anything you must reproduce or audit, deploy by immutable git-sha (as this repo does) or by digest. Treat `:latest` as "probably recent," never as "exactly this."
- **Build-context bloat.** `docker build .` ships the whole folder to the daemon. Without `.dockerignore`, that includes `node_modules` and `.git` — slow, and a secret-leak risk. Keep `.dockerignore` tight and current.
- **Secrets are forever in layers.** Deleting a file in a later layer doesn't scrub it from earlier ones. Never `COPY` or `echo` a secret during build; inject at runtime.
- **ENTRYPOINT vs CMD confusion.** `ENTRYPOINT` always runs; `CMD` is default args that `docker run …` overrides. If your "override" is being ignored, you probably put the value in `ENTRYPOINT`. Use the **exec (JSON-array) form** so signals reach your app and it shuts down gracefully.
- **Alpine (musl) surprises.** Some prebuilt/native binaries expect glibc and fail cryptically on Alpine. If a tool won't launch on Alpine, `bookworm-slim` (glibc) is usually the fix — exactly why the API's runtime isn't Alpine.
- **A container needs a foreground process.** If your `CMD` runs something that backgrounds itself and exits, the container stops. That's why nginx runs with `daemon off;` and Node runs in the foreground.
- **`localhost` inside a container is the container.** Cross-service calls use service/hostnames on the Docker network, not `localhost`.
- **`chown`/`USER` ordering.** Switch to `USER node` *after* root-only steps, and `--chown=node:node` files the app must read — otherwise the non-root process hits permission errors it can't explain.
- **Playwright browser path.** If Chromium is installed as one user and launched as another, it looks in the wrong cache dir and fails. Pin `PLAYWRIGHT_BROWSERS_PATH` to a shared location (the repo's `/ms-playwright`).

---

## 11. Glossary & further reading

### Glossary

- **Image** — a frozen, read-only, layered bundle of a filesystem + start metadata; the thing you build and ship.
- **Container** — a running (or stopped) instance of an image, with a disposable writable layer on top.
- **Layer** — one filesystem diff from one build step; cached and shared across images.
- **Union filesystem (OverlayFS)** — the mechanism that merges stacked layers into one view.
- **Dockerfile** — the text recipe of instructions that builds an image.
- **Build context** — the files sent to the builder for a build, filtered by `.dockerignore`.
- **`.dockerignore`** — a `.gitignore`-style list of paths to keep *out* of the build context.
- **Registry / repository** — server storing images (this repo: Google Artifact Registry).
- **Tag** — a mutable, human-readable pointer to an image (`name:tag`).
- **Digest** — an immutable content hash (`name@sha256:…`) naming exact image bytes.
- **Base image** — the image your `FROM` starts from.
- **Multi-stage build** — one Dockerfile with multiple `FROM` stages; the lean final stage copies only artifacts from fatter build stages.
- **ARG** — a build-time-only variable (`--build-arg`).
- **ENV** — an environment variable baked into the image, present at runtime.
- **ENTRYPOINT** — the command that always runs on container start.
- **CMD** — the default command/args, overridable at `docker run`.
- **Volume / bind mount** — persistent storage outside the container's disposable layer.
- **Namespaces** — kernel feature giving a container its isolated view of PIDs/mounts/network.
- **cgroups** — kernel feature limiting/metering a container's CPU/memory/I/O.
- **Daemon (`dockerd`)** — the background service that builds and runs containers; the `docker` CLI talks to it.
- **BuildKit / Buildx** — Docker's modern builder (better caching, secret mounts, multi-arch).
- **`gibp-base`** — this repo's shared base image with all monorepo dependencies pre-installed.
- **Artifact Registry** — GCP's image store, at `us-west1-docker.pkg.dev/gibp-ledger/gibp-apps` here.

### Further reading (official)

- Docker overview & get started — <https://docs.docker.com/get-started/>
- Dockerfile reference — <https://docs.docker.com/reference/dockerfile/>
- Dockerfile best practices — <https://docs.docker.com/build/building/best-practices/>
- Multi-stage builds — <https://docs.docker.com/build/building/multi-stage/>
- Build cache — <https://docs.docker.com/build/cache/>
- `.dockerignore` — <https://docs.docker.com/build/concepts/context/#dockerignore-files>
- BuildKit build secrets — <https://docs.docker.com/build/building/secrets/>
- Multi-platform images — <https://docs.docker.com/build/building/multi-platform/>
- Google Artifact Registry (Docker) — <https://cloud.google.com/artifact-registry/docs/docker>
- Playwright in Docker — <https://playwright.dev/docs/docker>

### Where to go next in this handbook

- **`../deployment-lifecycle-guide.md`** — how CI builds `gibp-base` and each app, pushes to Artifact Registry, and rolls out to GKE via ArgoCD. This chapter stops at the image; that one takes it to production.
- **The `docker-compose` chapter** — how `docker-compose.yml` wires the whole system (Postgres, Formance, Kafka, the worker, Debezium) together on your laptop, using the *dev* Dockerfiles you met in §7.5.
