# Chapter 7 — Kubernetes (deep) + ArgoCD (GitOps)

> **What this chapter is.** A from-first-principles book on the two pieces of
> machinery that actually run this product in the cloud: **Kubernetes** — the
> orchestrator that keeps your containers alive, networked, healthy, and scaled — and
> **ArgoCD** — the robot that continuously forces the running cluster to equal what git
> says. By the end you will understand the Kubernetes object model, its control loop,
> scheduling, networking, storage, config/secrets, RBAC, health, and autoscaling; you
> will be able to deploy *any* app to *any* Kubernetes cluster from a blank machine; and
> you will know exactly how *this* repo wires all of it together.

> **Status in this repo.** Live and in production. A **GKE Autopilot** cluster
> (`autopilot-cluster-1`, region `us-west1`, project `gibp-ledger`) runs the `api`, the
> two React webapps (`company-webapp`, `super-webapp`), and the `kafka-worker`, plus
> supporting operators (Strimzi/Kafka, Formance, External Secrets). Every workload is
> declared as YAML under `infra/k8s/` and reconciled by ArgoCD, which watches the
> **`test`** branch. Nobody deploys by typing `kubectl apply` at production — they push
> git, and ArgoCD does the rest.

> **How to read this.** Sections 1–4 build the mental model and the deep theory (read
> straight through the first time). Section 5 is a hands-on **zero-to-running** lab on
> your own laptop. Section 6 goes deep on **ArgoCD/GitOps**. Section 7 maps every concept
> onto *this* repo's real files. Sections 8–10 are production concerns, debugging
> playbooks, and honest gotchas — keep those within reach when you have a real task.
> Section 11 is a glossary + links.
>
> **This chapter is about Kubernetes and ArgoCD as technologies.** For the end-to-end
> **CI/CD pipeline** — how a `git push` becomes four Docker images and an updated image
> tag that ArgoCD then picks up — read the companion doc, and do not expect it repeated
> here:
>
> → **[`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md)** (the whole
> `push → build → deploy` story).
>
> For the exact one-time install/connect commands for ArgoCD, see
> [`../../gibp-docs/cicd/06-argocd-setup.md`](../../gibp-docs/cicd/06-argocd-setup.md).
> For how secrets get from GCP Secret Manager into pods, see **Chapter 8 — GCP**
> (`08-gcp.md`).

---

## Table of contents

1. [The problem: one server isn't enough, and doing it all by hand is impossible](#1)
2. [Mental model: a tireless operations manager running one loop forever](#2)
3. [Core concepts & vocabulary (the big table)](#3)
4. [How it actually works (deep)](#4)
5. [Setup from scratch: zero to running on Kubernetes](#5)
6. [GitOps with ArgoCD (deep)](#6)
7. [How THIS repo uses Kubernetes + ArgoCD](#7)
8. [Production concerns](#8)
9. [Operations & debugging playbooks](#9)
10. [Gotchas & hard-won lessons](#10)
11. [Glossary + further reading](#11)

---

<a name="1"></a>
## 1. The problem: one server isn't enough, and doing it all by hand is impossible

You know Docker. You can `docker build` an image and `docker run` a container. On your
laptop that's the whole story. Let's discover, by hitting walls, why an entire *system*
(Kubernetes) has to exist on top of that. Each wall below forces the next tool.

**Wall 1 — the laptop closes.** A container only runs while something runs it. Users
need the app at 3am. So you rent an always-on computer — a **server**. Fine.

**Wall 2 — the container crashes at 3am.** Now someone has to notice and restart it.
You could write a shell loop that restarts it… but that loop has to survive reboots,
and log why, and back off if it's crash-looping. You're now writing a supervisor.

**Wall 3 — the server itself dies.** Disk fails, kernel panics, the cloud reclaims the
machine. Your one server *is* your product; when it's gone, you're down, and your
restart loop died with it. You need **more than one server**, and something *above* the
servers that can move your container to a healthy one.

**Wall 4 — 10,000 users show up.** One container isn't enough. You need *N* identical
copies and something to **share traffic** across them. When traffic drops, you want to
shrink back down so you're not paying for idle capacity. That's **scaling** and **load
balancing**, continuously.

**Wall 5 — you ship a new version and it's broken.** You need to replace the running
copies *gradually* (never zero copies serving), watch the new ones become healthy, and
if they don't, **stop and roll back** — automatically, in seconds, not by SSHing into
five boxes.

**Wall 6 — the copies need to *find* each other, and the outside world needs to find
them.** Copies come and go; their IP addresses change every time one restarts. Your
frontend can't hard-code an IP that vanishes in an hour. You need **stable names** and a
**public front door** with HTTPS.

**Wall 7 — every copy needs a database password, and it can't be in the image.** You
need to deliver **config and secrets** to running copies without baking them into the
build or typing them by hand on each server.

**Wall 8 — health is subtle.** "The process is running" is not "the app is ready to
serve." A pod might be up but still connecting to the database, or alive but wedged. You
need **health checks** with *different meanings* (is it booting? is it ready for
traffic? is it hung and needs a kick?).

Now count what you'd be hand-building: a supervisor, a scheduler that places containers
on healthy machines, a load balancer, a rolling-update engine with rollback, a private
DNS, a secret courier, and a health checker — *and you'd run all of that across a fleet
of servers, forever, correctly, at 6pm on a Friday.* **That is impossible to do by hand
reliably.**

> **The one idea that fixes all eight walls at once: declare desired state; let a
> controller continuously reconcile reality to match it.** You stop writing *steps*
> ("restart this, move that, add a copy") and start writing a *target* ("I always want
> 2 healthy copies of image X, reachable at name Y, with these secrets"). A system reads
> that target and — forever — observes what's actually running, computes the difference,
> and takes the actions to close the gap. That system is **Kubernetes**.

Everything in this chapter is a consequence of that single sentence.

---

<a name="2"></a>
## 2. Mental model: a tireless operations manager running one loop forever

Picture a **factory operations manager** who never sleeps, never forgets, and does
exactly one thing over and over:

```
              THE CONTROL LOOP (runs forever, for every object)
    ┌──────────────────────────────────────────────────────────────┐
    │                                                                │
    │   1. OBSERVE   read the actual state of the world              │
    │      (what pods are running? are they healthy? on which node?) │
    │                          │                                     │
    │                          ▼                                     │
    │   2. DIFF      compare actual  vs  DESIRED (what you declared)  │
    │      ("I want 2 pods; I see 1 healthy + 1 crashed")            │
    │                          │                                     │
    │                          ▼                                     │
    │   3. ACT       take the smallest action to close the gap       │
    │      (start a new pod; kill an extra; move one off a dead node) │
    │                          │                                     │
    └──────────────────────────┘  ... then loop again, immediately   │
                                                                     ─┘
```

You write the **target** (a text file — a *manifest*). The manager makes it true, and
*keeps* it true. A copy dies → the diff shows "1 vs 2" → it starts one. A server dies →
the pods on it are gone → the diff shows the gap → it reschedules them elsewhere. You
change `2` to `5` → next loop, it starts three more. You never manage individual
containers; you manage the *desired state*, and the loop does the rest.

Two analogies, mapped straight back to the real terms:

| Analogy | Real Kubernetes term |
|---|---|
| A **thermostat**: you set a target temperature; it constantly nudges the room to match and complains if it can't. | The **control loop** / **reconciliation**. Target = your manifest; room = the cluster. |
| A **restaurant standing order**: "always keep 2 cooks making this dish." A cook faints → hire one. Rush → the plan says how to add cooks. | A **Deployment** with `replicas: 2`. Cook = **Pod**. |
| The **single order window** customers use instead of chasing individual cooks. | A **Service** — a stable address that load-balances to the pods. |
| The **front door with the street address and the lock**. | The **Ingress** — public URL + HTTPS + path routing. |

> **Keep this loop in your head for the whole chapter.** Every Kubernetes feature —
> rolling updates, self-healing, autoscaling, ArgoCD itself — is *just another
> controller running this same observe → diff → act loop* over a different kind of
> object. Once you see that, Kubernetes stops being a pile of nouns and becomes one
> pattern repeated.

---

<a name="3"></a>
## 3. Core concepts & vocabulary (the big table)

Kubernetes has a lot of nouns. Here they all are in one place, each with the *one
problem it solves*. Skim it now; you'll return to it constantly. Sections 4–7 make each
one real.

| Term | What it is | The problem it solves |
|---|---|---|
| **Cluster** | A pool of machines managed as one computer. | One server isn't reliable or big enough. |
| **Node** | One machine (VM or physical) in the cluster that runs pods. | You need real hardware to execute on. |
| **Control plane** | The cluster's brain: `api-server`, `scheduler`, `controller-manager`, `etcd`. | Someone has to store the desired state and drive the loop. |
| **`kube-apiserver`** | The single front door to the cluster; everything talks to it (REST API). | One consistent, authenticated place to read/write state. |
| **`etcd`** | A consistent key-value store; **the source of truth** for all cluster state. | Desired + actual state must be stored durably and consistently. |
| **`kube-scheduler`** | Decides *which node* a new pod runs on. | Placing pods onto machines that can fit them. |
| **`controller-manager`** | Runs the built-in control loops (Deployment, ReplicaSet, Node, …). | Continuously reconciling desired vs actual. |
| **`kubelet`** | An agent on every node that actually starts/stops containers and reports health. | Turning "desired pod on this node" into a running container. |
| **`kube-proxy`** | Programs each node's networking so Service IPs route to pods. | Making the stable Service address actually work. |
| **Pod** | The smallest deployable unit: one (usually) or a few containers sharing an IP + storage. | The living instance of your app. |
| **ReplicaSet** | Keeps *N* identical pods running. | Redundancy + capacity, self-healed. |
| **Deployment** | Manages ReplicaSets to give you rolling updates + rollback on top of "keep N pods." | Ship new versions with zero downtime and an undo button. |
| **Service** | A stable virtual IP + DNS name that load-balances to matching pods. | Pods' IPs change; callers need one fixed address. |
| **ClusterIP** | The default Service type — reachable only *inside* the cluster. | Internal service-to-service traffic. |
| **NodePort** | Exposes a Service on a fixed port of every node. | Crude external access without a cloud LB. |
| **LoadBalancer** | Asks the cloud for a real external load balancer + public IP. | External access on managed clouds. |
| **Ingress** | An L7 (HTTP) router: one public entry, routes by host/path, terminates TLS. | One front door + HTTPS + path routing for many Services. |
| **ConfigMap** | Non-secret config as key/value, injected as env or files. | Config out of the image; change without rebuilding. |
| **Secret** | Like a ConfigMap but for sensitive values (**base64, not encrypted by default**). | Delivering passwords/keys to pods. |
| **Namespace** | A named partition of the cluster (`gibp`, `argocd`, …). | Grouping, isolation, scoped permissions, no name clashes. |
| **Label / Selector** | Arbitrary `key: value` tags on objects / a query over them (`app: api`). | Loose coupling — Services find pods by label, not by name. |
| **Job** | Runs a pod to completion (e.g. a migration). | One-off/batch work. |
| **CronJob** | A Job on a schedule. | Recurring batch work. |
| **StatefulSet** | Like a Deployment but for stateful pods: stable identity + stable storage per pod. | Databases, brokers — things where pod #0 ≠ pod #1. |
| **DaemonSet** | One pod on *every* node. | Node-level agents (logging, metrics). |
| **PersistentVolume (PV)** | A piece of real storage in the cluster. | Data that must outlive a pod. |
| **PersistentVolumeClaim (PVC)** | A pod's *request* for storage of a size/class. | Pods ask for storage without knowing the backend. |
| **StorageClass** | A template that dynamically provisions PVs (e.g. a GCP disk). | Auto-create the right disk on demand. |
| **Resource requests / limits** | Guaranteed CPU/RAM (`requests`) and hard cap (`limits`) per container. | Scheduling + fair sharing + protecting nodes. |
| **Probe** | A health check: **startup**, **readiness**, **liveness**. | Knowing if a pod is booting, ready for traffic, or hung. |
| **HPA** (HorizontalPodAutoscaler) | Adds/removes pods based on a metric (e.g. CPU). | Scale copies with load automatically. |
| **PDB** (PodDisruptionBudget) | "Never voluntarily drop below N available pods." | Zero-downtime during node drains/upgrades. |
| **ServiceAccount** | An identity a pod runs as (for talking to the API/cloud). | Least-privilege identity for workloads. |
| **Role / ClusterRole** | A set of allowed API verbs on resources (namespaced / cluster-wide). | Defining *what* an identity may do. |
| **RoleBinding / ClusterRoleBinding** | Grants a Role to a user/group/ServiceAccount. | Attaching permissions to an identity. |
| **CRD** (CustomResourceDefinition) | Teaches the API server a *new kind of object*. | Extending Kubernetes with domain nouns (e.g. `Kafka`, `Application`). |
| **Operator** | A controller that reconciles a CRD (runs the loop for a custom noun). | Automating a complex app (Kafka, Formance) the k8s way. |
| **Helm** | A templating + packaging tool; installs charts as *releases*. | Reusing/parameterizing big bundles of manifests. |
| **Manifest** | A YAML file describing a desired object. | How you *declare* everything above. |

---

<a name="4"></a>
## 4. How it actually works (deep)

This is the theory section. If you internalize just this section, everything else is
detail.

### 4.1 The API server + etcd are the source of truth; controllers reconcile toward it

There is exactly one authoritative place where cluster state lives: **`etcd`** (a
consistent key-value store), fronted by the **`kube-apiserver`**. *Every* actor — you
with `kubectl`, the scheduler, every controller, ArgoCD, the kubelets — reads and writes
state **only** through the API server. Nothing edits `etcd` directly. This is what makes
the whole thing sane: one door, one truth, everything authenticated and validated on the
way in.

Each object in `etcd` has two halves:

- **`spec`** — the *desired* state (what you asked for). You own this.
- **`status`** — the *actual* state (what's really happening). Controllers own this.

```
                       THE CONTROL PLANE
   ┌───────────────────────────────────────────────────────────┐
   │                                                           │
   │   kubectl / ArgoCD ──write spec──►  kube-apiserver         │
   │                                          │  ▲              │
   │                                    read/write               │
   │                                          ▼  │              │
   │                                        etcd  (the truth)    │
   │                                          ▲                 │
   │        ┌─────────────┬───────────────────┤                 │
   │        │             │                   │                 │
   │  controller-mgr   scheduler          (many controllers)     │
   │  (reconcile        (assign pods                             │
   │   Deployments,      to nodes)                               │
   │   ReplicaSets…)                                             │
   └─────────────────────────────────────────────────────────────┘
                │  desired pod bound to node N
                ▼
        ┌─────────────── NODE N ───────────────┐
        │  kubelet  ──starts──►  container(s)    │
        │  kube-proxy  ──routes──► Service IPs    │
        └────────────────────────────────────────┘
```

A **controller** is just a program looping: *watch* objects via the API server, compare
`spec` to `status`, and act to close the gap, writing results back to `status`. There
are dozens of them. The magic is that they're **level-triggered, not
edge-triggered**: they don't react to a one-time "event," they continuously drive toward
the desired *level*. Miss an event, crash, restart — doesn't matter; next loop it
re-observes reality and keeps reconciling. That's why Kubernetes is so robust, and it's
the exact same principle ArgoCD uses one layer up (Section 6).

### 4.2 The Deployment → ReplicaSet → Pod cascade (and how rolling updates + rollback work)

You almost never create a Pod directly. You declare a **Deployment**, and a cascade of
controllers turns it into running pods:

```
  Deployment (you write this)            "keep 2 pods of image :A, and manage versions"
      │  owns
      ▼
  ReplicaSet  (rev 1, image :A)          "keep exactly 2 pods matching app=api"
      │  owns
      ▼
  Pod  Pod                               the actual running containers
```

- The **Deployment controller** creates a **ReplicaSet** for the current pod template.
- The **ReplicaSet controller** ensures exactly `replicas` pods matching its selector
  exist — this is the self-healing layer. Kill a pod, it makes another.
- The **scheduler** assigns each new pod to a node; the node's **kubelet** starts it.

**Rolling update.** When you change the pod template (e.g. a new `image:` tag —
*exactly what happens on every deploy in this repo*), the Deployment does **not** delete
everything and start over. It creates a **new ReplicaSet** (rev 2, image :B) and shifts
pods from old to new *gradually*, governed by two knobs:

- `maxUnavailable` — how many pods may be down during the move (default 25%).
- `maxSurge` — how many *extra* pods may exist temporarily (default 25%).

```
  rev1 (:A)  ●●            rev1 ●              rev1              (done)
  rev2 (:B)         →      rev2 ●        →     rev2 ●●     →     rev2 ●●
             start a :B, wait until it's READY, then retire a :A, repeat
```

Crucially, a new pod only counts as "up" once its **readiness probe** passes (4.6). So
if :B is broken and never becomes ready, the rollout **stalls** with old pods still
serving — no outage. That's the safety net.

**Rollback.** Because the old ReplicaSet (rev 1) is *kept*, not deleted, undo is
trivial: `kubectl rollout undo deployment/api` scales rev 1 back up and rev 2 down. In
this repo you rarely do that — you `git revert` the manifest and let ArgoCD roll it back
(Section 6), which is better because git stays the truth. Either way, rollback works
because **old versions are retained** and every image is pinned to an immutable
`:<git-sha>` tag (never `:latest` — see Section 10).

### 4.3 The scheduler, requests/limits, and how GKE Autopilot provisions nodes

When a pod needs a node, the **scheduler** runs a two-phase decision: **filter** (which
nodes *can* fit this pod?) then **score** (of those, which is *best*?). The single most
important input is the pod's **resource requests**:

- **`requests`** — the amount of CPU/RAM the pod is *guaranteed*. The scheduler will only
  place a pod on a node with at least this much *unreserved*. Requests are how
  Kubernetes does bin-packing.
- **`limits`** — the hard *ceiling*. Exceed the memory limit → the kernel **OOM-kills**
  the container. Exceed the CPU limit → the container is **throttled** (slowed), not
  killed.

CPU is measured in *millicores*: `1000m` = 1 full core, `200m` = 0.2 of a core. Memory
is in bytes with `Mi`/`Gi` suffixes (`256Mi`, `512Mi`).

This repo's `api` container declares (verbatim from `deployment.yaml`):

```yaml
resources:
  requests: { cpu: '200m', memory: '256Mi' }   # guaranteed + used for scheduling
  limits:   { cpu: '500m', memory: '512Mi' }   # ceiling; >512Mi RAM ⇒ OOMKilled
```

**GKE Autopilot changes the mental model.** On a normal ("Standard") cluster you run
a fixed set of nodes and pods pack into them. On **Autopilot**, *you don't manage nodes
at all* — Google provisions node capacity **on demand from your pods' requests**, and
**bills you per pod request, not per node**. Consequences that matter here:

- Your `requests` are literally your bill. Right-sizing them is real money (Section 8).
- If you set no requests, Autopilot injects defaults — so always set them explicitly.
- The cluster autoscaler still runs, and its default profile (`OPTIMIZE_UTILIZATION`)
  aggressively **consolidates** pods onto fewer nodes to save money, evicting and
  rescheduling pods to do so. That behavior caused a real incident in this repo — see
  Section 7 and `pod-node-churn.md`.

### 4.4 Networking: every pod gets an IP, Services give stable names, Ingress is the L7 door

Kubernetes networking rests on one rule: **every pod gets its own routable IP, and any
pod can reach any other pod's IP without NAT.** No port-mapping gymnastics like plain
Docker. But pod IPs are *ephemeral* — a pod restarts, it gets a new IP. So you never
talk to a pod IP directly.

**Service = a stable front for a moving set of pods.** A Service has a fixed virtual IP
(the *ClusterIP*) and a DNS name. It uses a **label selector** to find its backing pods
and load-balances across whichever pods currently match:

```yaml
# infra/k8s/apps/api/service.yaml
kind: Service
spec:
  selector: { app: api }        # any pod labelled app=api is a backend
  ports:
    - port: 80                  # the Service listens on 80
      targetPort: 3000          # forwards to the pod's containerPort 3000
  type: ClusterIP               # internal-only
```

`kube-proxy` on every node programs the kernel (iptables/IPVS) so that traffic to the
Service IP is transparently distributed to a healthy backing pod. **Readiness** gates
membership: a pod that fails its readiness probe is *removed from the Service's rotation*
(4.6), so traffic never hits a not-ready pod.

**DNS.** Kubernetes runs an in-cluster DNS. Every Service is reachable by name:

```
<service>.<namespace>.svc.cluster.local
```

So inside the cluster the api is `http://api.gibp.svc.cluster.local`. Same-namespace
callers can shorten it to just `http://api`. This repo uses the fully-qualified form for
cross-namespace calls, e.g. the api reaches Formance at
`http://gateway.gibp-stack.svc.cluster.local:8080` and the database via
`cloudsql-proxy.gibp.svc.cluster.local:5432`. **Memorize this DNS pattern** — a huge
share of "service can't reach service" bugs are a wrong namespace or a typo'd name.

**Service types, from least to most exposed:**

| Type | Reachable from | Use |
|---|---|---|
| `ClusterIP` | inside the cluster only | service-to-service (the default; all this repo's Services) |
| `NodePort` | a fixed port on every node's IP | dev/on-prem without a cloud LB |
| `LoadBalancer` | a real public IP from the cloud | one external service |

**Ingress = one L7 door for many Services.** You don't want a separate cloud load
balancer per app. An **Ingress** is an HTTP(S) router: a single public entry that
terminates TLS and routes by **host** and **URL path** to different Services. On GKE the
Ingress is backed by a Google HTTP(S) Load Balancer.

```
                     Internet
                        │  https://gibp.wisflux.com
                        ▼
              ┌──────────────────────┐   TLS terminated here (gibp-cf-origin-tls)
              │   Ingress (L7)        │   static IP: gibp-ingress-ip
              └──────────────────────┘
                 /api │  /admin │  /   (route by path)
            ┌────────┘    │       └────────┐
            ▼             ▼                ▼
      Service api   Service super    Service company
       :80→:3000     -webapp :80      -webapp :80
            │             │                │
         Pods api     Pods super       Pods company
```

That's exactly this repo's `public-ingress` (Section 7).

### 4.5 Config & Secrets (and why "Kubernetes Secret" is not actually secret)

Two objects deliver configuration to a pod:

- **ConfigMap** — non-sensitive key/values.
- **Secret** — sensitive key/values.

Both can be consumed three ways:

| Method | What it does | YAML shape |
|---|---|---|
| Inline `env` | Set individual env vars literally. | `env: [{ name: NODE_ENV, value: 'production' }]` |
| `envFrom` | Dump **all** keys of a ConfigMap/Secret in as env vars. | `envFrom: [{ secretRef: { name: api-secrets } }]` |
| Volume mount | Mount keys as files in the container filesystem. | `volumeMounts` + `volumes.secret` |

This repo uses inline `env` for non-secret config and `envFrom` for the whole secret
bundle (Section 7).

**The critical truth about Kubernetes Secrets: they are base64-encoded, not
encrypted.** Anyone who can `kubectl get secret -o yaml` (or read `etcd`) can decode
them with one command:

```bash
kubectl get secret api-secrets -n gibp -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Base64 is *encoding*, not *encryption* — it's reversible with zero key. So a raw Secret
committed to git is a plaintext password in git. **This is the entire reason external
secret managers exist.** The pattern this repo uses:

1. Real secret *values* live in **GCP Secret Manager** (encrypted, IAM-controlled),
   never in git.
2. The **External Secrets Operator (ESO)** — a controller — reads them and
   *materializes* a normal Kubernetes Secret in the cluster.
3. The pod consumes that Secret via `envFrom`, none the wiser.

So git contains only a *mapping* (which vault key → which env var), never a value. The
deep dive on this — ClusterSecretStore, ExternalSecret, the refresh loop — is in
**Chapter 8 (GCP)**; here just remember *why*: because a k8s Secret alone is not a
secret.

### 4.6 Health probes: three checks, three distinct meanings

"Is the process running?" is far too crude. Kubernetes offers three probes, each
answering a *different* question. Confusing them is one of the most common production
mistakes (Section 10).

| Probe | Question it answers | What happens on failure | Runs |
|---|---|---|---|
| **startupProbe** | "Has it finished **booting**?" | Keeps other probes *disabled* until it passes; if it never passes, the container is killed and restarted. | once, at start |
| **readinessProbe** | "Should it receive **traffic** right now?" | Pod is **removed from Service rotation** (no traffic), but **not restarted**. | continuously |
| **livenessProbe** | "Is it **alive**, or hung and unrecoverable?" | Container is **killed and restarted**. | continuously |

The distinction that trips everyone up: **readiness controls traffic; liveness controls
restarts.** A pod that's temporarily overloaded should fail *readiness* (stop sending it
traffic, let it recover) — not *liveness* (killing it makes things worse). A pod that's
genuinely wedged should fail *liveness* (restart it). The startupProbe exists so a
slow-booting app isn't killed by an impatient liveness probe before it's even up.

This repo's `api` uses all three (verbatim):

```yaml
startupProbe:   { httpGet: { path: /api/health,       port: 3000 }, failureThreshold: 30, periodSeconds: 5 }
readinessProbe: { httpGet: { path: /api/health/ready, port: 3000 }, periodSeconds: 10 }
livenessProbe:  { httpGet: { path: /api/health,       port: 3000 }, periodSeconds: 30 }
```

Note the *deliberate* design (it's commented in the file): readiness checks
`/api/health/ready`, which verifies **Postgres** (the universal hard dependency), while
liveness checks the lighter `/api/health`. And a broader `/api/health/deps` (Formance +
BigQuery too) is *intentionally not* wired as a probe — so a Formance or BigQuery blip
doesn't pull the pod out of rotation. The lesson: **choose what each probe checks
carefully** — an over-eager readiness check turns a downstream hiccup into your own
outage.

`startupProbe.failureThreshold: 30 × periodSeconds: 5` = up to **150 seconds** to boot
(migrations run at startup here) before Kubernetes gives up. Generous on purpose.

### 4.7 Autoscaling (HPA) & disruption budgets (PDB)

**Horizontal Pod Autoscaler (HPA).** A controller that watches a metric (classically
CPU utilization) and adjusts a Deployment's `replicas` to keep the metric near a target.
"CPU above 70%? add pods. Below? remove them." It scales *pods* (horizontal), distinct
from the *cluster* autoscaler which scales *nodes*. (This repo runs a fixed
`replicas: 2` and does not currently define an HPA — it's small and steady — but you'd
add one exactly as in Section 5.7.)

**PodDisruptionBudget (PDB).** Disruptions come in two flavors:

- **Involuntary** — a node crashes. Nothing can prevent it.
- **Voluntary** — you/K8s drain a node for an upgrade, or the autoscaler consolidates.

A PDB constrains *voluntary* disruptions: `minAvailable: 1` means "when draining, never
take this app below 1 available pod." Combined with `replicas: 2`, node maintenance
becomes **zero-downtime**: the drain waits for a replacement pod to be Ready on another
node before it removes the last old one. This repo's `api` PDB:

```yaml
kind: PodDisruptionBudget
spec:
  minAvailable: 1
  selector: { matchLabels: { app: api } }
```

> **The catch (Section 10):** a *too-strict* PDB can also *block* a drain forever. If
> `minAvailable` equals your replica count, no pod can ever be evicted and node upgrades
> stall. Leave headroom.

### 4.8 RBAC: least privilege for humans and workloads

**RBAC** (Role-Based Access Control) governs *who can do what* to the API server. Four
objects, in two pairs:

- **What is allowed:** a **Role** (namespaced) or **ClusterRole** (cluster-wide) — a list
  of *verbs* (`get`, `list`, `create`, `delete`) on *resources* (`pods`, `secrets`).
- **Who gets it:** a **RoleBinding** / **ClusterRoleBinding** — attaches a Role to a
  *subject*: a human user, a group, or a **ServiceAccount**.

A **ServiceAccount** is the identity a *pod* runs as. By default pods get a minimal one;
you grant more only when a workload must talk to the API (or, via cloud IAM binding, to
cloud services). The rule is **least privilege**: grant the narrowest set of verbs on
the fewest resources in one namespace. This repo's clearest example is the Cloud SQL
proxy pod, which runs as a dedicated ServiceAccount (`cloudsql-proxy-ksa`) wired via
**Workload Identity** to a GCP identity that may reach exactly one database — nothing
more (Section 7, and Chapter 8).

### 4.9 CRDs & operators: teaching Kubernetes new nouns

Out of the box, Kubernetes understands `Pod`, `Service`, `Deployment`, etc. But how do
you run something complicated like a Kafka cluster, which needs brokers, topics,
rebalancing, and version upgrades done *just so*? You could hand-write dozens of
StatefulSets and pray. Better: **teach Kubernetes the noun `Kafka`** and let an expert
controller manage it.

- A **CustomResourceDefinition (CRD)** registers a new *kind* with the API server. After
  a CRD is installed, `kind: Kafka` (or `kind: Application`, or `kind: ExternalSecret`)
  is a first-class object you can `kubectl apply`, `get`, and `describe` like any other.
- An **operator** is a controller that watches that CRD and runs the observe → diff →
  act loop for it — encoding the operational expertise ("to add a broker, do X, then Y,
  then rebalance") in software.

This is the *same control-loop pattern* as everything else, applied to a domain object.
This repo relies on three operators, each installed into the cluster and each teaching it
new nouns:

| Operator | New nouns (CRDs) it manages | Used for |
|---|---|---|
| **Strimzi** | `Kafka`, `KafkaTopic`, `KafkaConnect`, … | Running Kafka properly on k8s |
| **Formance** | `Stack`, `Ledger`, `Gateway`, … | Running the double-entry ledger engine |
| **External Secrets** | `ClusterSecretStore`, `ExternalSecret` | Delivering GCP secrets into pods |

And — this is the beautiful part — **ArgoCD is itself an operator**: its CRD is
`Application`, and its controller reconciles "the cluster should match this git path."
(Section 6.)

### 4.10 Helm: templated manifests + releases

Writing raw YAML for a big third-party system (Kafka's operator, ArgoCD itself,
Prometheus) means copying hundreds of lines and hand-editing values. **Helm** is the
package manager that fixes this:

- A **chart** is a bundle of *templated* manifests + a `values.yaml` of defaults.
- You install a chart with your overrides; Helm renders the templates and applies the
  result as a named **release** (which it tracks, so you can `helm upgrade` / `helm
  rollback`).

Think "npm for Kubernetes manifests." In this repo you don't hand-install Helm charts at
production — instead, **ArgoCD installs charts for you**: an ArgoCD `Application` can
point its `source` at a Helm chart repo and pass `helm.values`, and ArgoCD renders +
syncs it. That's exactly how the Strimzi operator is installed (Section 6.6 and 7).

---

<a name="5"></a>
## 5. Setup from scratch: zero to running on Kubernetes

Time to make it real. You'll install the tools, spin up a **real Kubernetes cluster on
your laptop**, deploy an app, expose it, scale it, roll it forward and back, add config,
secrets, probes, and resources, and finish with a Helm install. Everything here is
standard Kubernetes — it works identically on your laptop and on GKE.

### 5.1 Install the tools

- **`kubectl`** — the CLI that talks to any cluster's API server.
- **A local cluster.** Two easy options; pick one:
  - **kind** ("Kubernetes IN Docker") — runs the cluster as Docker containers. Great,
    fast, multi-node.
  - **minikube** — runs a single-node cluster in a VM or Docker.

```bash
# macOS (Homebrew)
brew install kubectl kind          # or: brew install minikube

# Linux — kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Linux — kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

Create the cluster and confirm `kubectl` can see it:

```bash
kind create cluster --name lab          # or: minikube start
kubectl cluster-info                     # should print the control-plane URL
kubectl get nodes                        # one (or more) Ready node
```

> **What just happened.** `kind` created a Kubernetes control plane + node(s) as Docker
> containers and wrote a *kubeconfig* (default `~/.kube/config`) so `kubectl` knows which
> cluster to talk to and how to authenticate. The **current context** is which cluster
> `kubectl` points at right now: `kubectl config current-context`.

### 5.2 `kubectl` fundamentals — the eight verbs you'll use forever

```bash
kubectl apply -f app.yaml         # create/update objects from a manifest (declarative)
kubectl get pods                  # list objects (add -A for all namespaces, -o wide for detail)
kubectl describe pod <name>       # full detail + the EVENTS log (your #1 debugging tool)
kubectl logs -f <pod>             # stream a pod's stdout/stderr (-f = follow)
kubectl exec -it <pod> -- sh      # get a shell inside a running container
kubectl port-forward <pod|svc> 8080:80   # tunnel a cluster port to your laptop
kubectl rollout status deploy/<name>     # watch a rollout; rollout undo to roll back
kubectl delete -f app.yaml        # remove what a manifest created
```

`apply` is **declarative**: you hand it the desired state and Kubernetes reconciles.
Prefer it over the imperative `kubectl create/edit/patch` — it's the same philosophy
ArgoCD automates.

### 5.3 Deploy a demo app (Deployment + Service + Ingress) and verify

Save this as `demo.yaml`. It runs the stock nginx image, fronts it with a Service, and
(optionally) an Ingress.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app: web }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }          # <-- must match the pod labels above
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f demo.yaml
kubectl get pods -l app=web           # two web-... pods, Running
kubectl get deploy,svc                # the Deployment (2/2 READY) and the Service

# Verify it serves, without any Ingress, by tunneling the Service to your laptop:
kubectl port-forward svc/web 8080:80
# in another terminal:
curl -s localhost:8080 | head -n1     # <title>Welcome to nginx!</title>  ✅
```

**Verify self-healing** — the whole point of Kubernetes:

```bash
kubectl delete pod -l app=web --field-selector 'status.phase=Running' | head -n1
kubectl get pods -l app=web -w        # watch: a replacement pod appears within seconds
```

You didn't restart anything. The ReplicaSet controller saw "1 vs 2" and fixed it.
That's Section 2's loop, live.

**Add an Ingress** (needs an ingress controller; on kind, install ingress-nginx first —
see its docs). The manifest mirrors this repo's real one:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```

### 5.4 Scale it

```bash
kubectl scale deploy/web --replicas=5
kubectl get pods -l app=web           # now 5 pods; the Service load-balances all 5
```

Or edit `replicas: 5` in the manifest and `kubectl apply` again — same result, but now
it's written down (the declarative way, which is what GitOps enforces).

### 5.5 Do a rolling update, then a rollback

```bash
# Roll forward to a new image version:
kubectl set image deploy/web web=nginx:1.29
kubectl rollout status deploy/web     # watches new pods become Ready, old ones retire

kubectl rollout history deploy/web    # see the revisions (rev 1: 1.27, rev 2: 1.29)

# Roll back to the previous revision (self-heal keeps the old ReplicaSet around):
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
```

You just did — by hand — what this repo's pipeline + ArgoCD do automatically on every
deploy (a new `image:` tag → rolling update) and every `git revert` (→ rollback).

### 5.6 Add a ConfigMap and a Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: web-config }
data:
  GREETING: "hello from k8s"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata: { name: web-secret }
type: Opaque
stringData:                      # stringData: you write plaintext; k8s base64s it for you
  API_TOKEN: "s3cr3t-not-really-secret-in-git"
```

Wire them into the Deployment's container:

```yaml
          envFrom:
            - configMapRef: { name: web-config }   # all ConfigMap keys → env vars
            - secretRef:    { name: web-secret }   # all Secret keys → env vars
```

```bash
kubectl apply -f demo.yaml
kubectl exec deploy/web -- printenv GREETING API_TOKEN   # both present in the container
```

> **Feel the gotcha for real:** `kubectl get secret web-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d`
> prints the value back in plaintext. That's why you never commit a real Secret to git —
> and why this repo uses External Secrets Operator instead (4.5, Chapter 8).

### 5.7 Add probes, resources, and an HPA

Extend the container spec with the production essentials:

```yaml
          resources:
            requests: { cpu: '50m',  memory: '64Mi' }
            limits:   { cpu: '200m', memory: '128Mi' }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 30
```

Add horizontal autoscaling (needs the metrics-server add-on for CPU metrics):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

```bash
kubectl get hpa web        # shows current vs target CPU and the chosen replica count
```

### 5.8 Intro to Helm — install a chart

```bash
# macOS: brew install helm   |   Linux: https://helm.sh/docs/intro/install/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a chart as a named release, overriding a value:
helm install my-nginx bitnami/nginx --set service.type=ClusterIP

helm list                          # your releases
kubectl get deploy,svc -l app.kubernetes.io/instance=my-nginx
helm upgrade my-nginx bitnami/nginx --set replicaCount=3   # change values → re-render
helm uninstall my-nginx            # clean removal of everything the chart created
```

That's the whole Helm model: **chart + your values → rendered manifests → a tracked
release.** In this repo you won't run `helm install` at production — ArgoCD renders
charts for you (Section 6.6) — but understanding Helm is what lets you read those ArgoCD
Applications.

### 5.9 Managed clusters: GKE Autopilot and `get-credentials`

A laptop cluster is for learning. Production is a **managed** cluster — here **GKE
Autopilot**, where Google runs the control plane *and* the nodes, and you just declare
pods (4.3). To point your `kubectl` at this repo's real cluster (you need GCP access):

```bash
gcloud container clusters get-credentials autopilot-cluster-1 \
  --region=us-west1 --project=gibp-ledger

kubectl config current-context      # now points at the GKE cluster
kubectl get pods -n gibp            # the real workloads
```

`get-credentials` writes a GKE context into your kubeconfig — it does **not** give you a
password; auth is delegated to your `gcloud` login. From here every command in this
chapter works against production (read freely; **do not** hand-edit — Section 6/10).

> **Guardrail:** when you have multiple contexts (kind + GKE), *always* check
> `kubectl config current-context` before running a mutating command. Deleting a demo
> pod on `kind-lab` is harmless; deleting one on the GKE cluster is production.

---

<a name="6"></a>
## 6. GitOps with ArgoCD (deep)

You can now deploy to Kubernetes by hand. So why isn't that the end of the story?

### 6.1 The problem: `kubectl apply` from CI rots the cluster

Imagine the pipeline just ran `kubectl apply` from a GitHub Action, or engineers ran it
from their laptops. Three problems compound:

1. **Drift.** Someone runs a quick `kubectl edit` at 2am to fix an incident and never
   writes it down. Now the cluster silently differs from every file in git. Six months
   later nobody can explain why production looks the way it does, and the manifests in
   git are fiction.
2. **No audit trail.** Who changed the replica count? When? Why? `kubectl` doesn't
   review, doesn't attribute, doesn't remember.
3. **No easy rollback.** "Undo the last change" means remembering the previous state and
   re-applying it correctly under pressure. There is no button.

The root cause: **the cluster is its own source of truth, and it's mutable by anyone.**

### 6.2 The GitOps principle

Flip it: **git is the single source of truth for the desired state of the cluster, and a
controller inside the cluster continuously reconciles the cluster to match git.**

- Want to change production? **Change a file and push.** (Reviewed via PR, attributed,
  timestamped.)
- Want to undo? **`git revert` and push.** The controller rolls the cluster back.
- Did someone hand-edit the cluster? **The controller reverts them** back to git.
- What's *actually* running? **Whatever commit git is at.** Always. Provably.

Every property you wanted — auditability, reproducibility, rollback, no drift — falls out
of "git is truth + a reconciler." This is just Section 2's control loop, lifted one level
up: instead of "desired = the Deployment spec," it's "desired = the YAML files in this
git path." The reconciler is **ArgoCD**.

```
        GITOPS = the same observe→diff→act loop, over git

   git repo (desired) ──┐
   infra/k8s/apps/api    │        ┌───────────────────────────────┐
                         └──────► │  ArgoCD application-controller │
   live cluster (actual) ───────► │  OBSERVE both, DIFF, then...   │
                                  │  SYNC (apply) / PRUNE / HEAL   │
                                  └───────────────────────────────┘
                                            │ makes cluster == git
                                            ▼
                                  Deployment, Service, PDB, ...
```

### 6.3 ArgoCD architecture

ArgoCD runs *inside* the cluster (in this repo, the `argocd` namespace) as a handful of
components:

| Component | Job |
|---|---|
| **application-controller** | The heart. Runs the reconciliation loop: compares desired (git) vs live (cluster), computes **sync status** and **health**, and (if automated) applies changes, prunes, and self-heals. |
| **repo-server** | Clones the git repo(s), renders manifests (runs Helm/Kustomize if used), and hands the controller the desired manifests. Caches for speed. |
| **redis** | A cache for rendered manifests and computed state (not a source of truth — safe to lose). |
| **api-server / UI** | The web UI + API + CLI backend where you *see* every app's status and can sync/rollback/inspect by hand. |
| **applicationset-controller** (optional) | Generates many `Application`s from a template (fan-out across apps/clusters/environments). |

### 6.4 The `Application` CRD — how you wire an app into GitOps

ArgoCD teaches the cluster a noun: **`Application`** (recall 4.9 — ArgoCD is an
operator). One `Application` says "watch *this* git path on *this* branch and make *this*
namespace match it." Here is this repo's real `api` Application, annotated
(`infra/k8s/argocd/api-app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gibp-api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # on delete, ArgoCD cleans up the app's resources
spec:
  project: default
  source:
    repoURL: https://github.com/gibillpay/gibp-ledger
    targetRevision: test                # WHICH branch/tag/commit is "desired"
    path: infra/k8s/apps/api            # WHICH folder of manifests
  destination:
    server: https://kubernetes.default.svc   # WHICH cluster (this one — where ArgoCD runs)
    namespace: gibp                     # WHICH namespace to apply into
  syncPolicy:
    automated:
      prune: true                       # delete cluster resources removed from git
      selfHeal: true                    # revert manual cluster edits back to git
    syncOptions:
      - CreateNamespace=true            # create the namespace if it doesn't exist
```

Read it as a sentence: *"Watch `infra/k8s/apps/api` on the `test` branch. Make the
`gibp` namespace look exactly like whatever YAML is there. If a file is deleted, delete
the resource (`prune`). If someone hand-edits the cluster, undo them (`selfHeal`)."*

The three `source` fields are the whole contract:

| Field | Means | Rollback lever |
|---|---|---|
| `repoURL` | which repo holds desired state | — |
| `targetRevision` | which branch/tag/**commit** = desired | pin to an old commit to freeze/rollback |
| `path` | which subfolder of manifests | — |

### 6.5 Sync status, health, and OutOfSync — the two axes you read constantly

ArgoCD reports each Application on **two independent axes**. Learn to read both:

- **Sync status** — does the *cluster match git*?
  - `Synced` — live == desired. Good.
  - `OutOfSync` — git changed (or someone edited the cluster) and it hasn't been applied.
- **Health** — are the resources *actually working*? (computed per resource type)
  - `Healthy` — e.g. Deployment has all replicas ready.
  - `Progressing` — a rollout is mid-flight.
  - `Degraded` — something's wrong (pods crash-looping, probe failing).
  - `Missing` / `Unknown` — resource absent / can't tell.

They're orthogonal, and the combination tells you *what* is wrong:

| Sync | Health | Meaning |
|---|---|---|
| Synced | Healthy | All good — cluster == git and everything works. |
| Synced | Degraded | Git was applied, but the app itself is broken (bad image, crash loop, missing secret). Fix the *app*, not the sync. |
| OutOfSync | Healthy | Git changed but not yet applied (auto-sync will catch it; or manifest is invalid). |
| OutOfSync | Degraded | Both: a broken change that also didn't apply cleanly. |

You see this in the ArgoCD **UI** (each app is a card + a live resource tree) or via CLI:

```bash
kubectl get applications -n argocd            # SYNC STATUS + HEALTH for every app
argocd app get gibp-api                        # detailed diff, if you have the argocd CLI
```

### 6.6 Sync waves, hooks, and Helm sources

**Sync waves** solve ordering. Sometimes resources must be applied in a sequence — a CRD
before the custom resource that uses it, a namespace before things in it, a migration
Job before the app. Annotate resources with a wave number and ArgoCD applies lower waves
first, waiting for each wave to become healthy:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # applied before wave 0 (the default)
```

**Sync hooks** run a resource (usually a Job) at a phase of the sync — `PreSync`,
`Sync`, `PostSync` — e.g. a database migration as a `PreSync` hook, or a smoke test as
`PostSync`:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

> Note: in *this* repo, DB migrations don't use an ArgoCD hook — they run inside the
> api container's **entrypoint** at startup (see the deployment-lifecycle guide, §4.1d).
> Both are valid patterns; know that hooks exist for when you need sync-time ordering.

**Helm as a source.** An `Application`'s `source` can point at a **Helm chart** instead
of a folder of raw YAML, and pass values inline. This repo installs the **Strimzi
operator** exactly this way (`infra/k8s/argocd/strimzi-operator-app.yaml`):

```yaml
spec:
  source:
    repoURL: https://strimzi.io/charts/          # a Helm chart repo, not a git repo
    chart: strimzi-kafka-operator
    targetRevision: 0.51.0                        # the chart version
    helm:
      values: |
        watchNamespaces: [ kafka-system ]
        watchAnyNamespace: false
  destination:
    server: https://kubernetes.default.svc
    namespace: kafka-system
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true ]
```

So ArgoCD renders the chart (via repo-server) and reconciles the result — GitOps for
third-party software, no manual `helm install`.

### 6.7 App-of-apps: bootstrapping many Applications from one

You have one `Application` per workload — but who creates all those `Application`
objects in the first place? The **app-of-apps** pattern: a *single* root `Application`
whose `path` is a folder full of *other* `Application` manifests (e.g.
`infra/k8s/argocd/`). Apply the root once; ArgoCD syncs it, which creates all the child
Applications, which sync their workloads. One `kubectl apply` bootstraps the entire
platform, and adding a new app becomes "commit a new `<app>-app.yaml` — the root picks it
up." This is the natural next step for this repo's `infra/k8s/argocd/` directory (today
each `Application` is applied individually — Section 7).

### 6.8 Rollback with ArgoCD

Three ways, best first:

1. **`git revert` (the GitOps way).** Revert the commit that changed the manifest and
   push. ArgoCD sees git move and syncs the cluster back. Because images are pinned to
   immutable `:<git-sha>` tags, the old image is still in the registry — it just works.
   This is the *only* method that keeps git as truth.
2. **ArgoCD UI → History and Rollback.** Pick a previous synced revision and roll back.
   Convenient, but it puts the app `OutOfSync` with git's HEAD — you must then reconcile
   git, or auto-sync will roll you *forward* again.
3. **`kubectl rollout undo` (emergency only).** Fast, but `selfHeal` will revert it back
   to git within a loop — so you *must* also fix git, or it's pointless.

### 6.9 Installing ArgoCD & connecting a private repo

You install ArgoCD once, into its own namespace (typically via Helm or the upstream
install manifest), get the initial admin password, log in, then register your git repo's
credentials and apply your `Application`(s). Those exact commands (values, repo
connection with a deploy key/token, first login) are already written down for this repo —
don't re-derive them:

→ **[`../../gibp-docs/cicd/06-argocd-setup.md`](../../gibp-docs/cicd/06-argocd-setup.md)**.

The *shape* of it (generic, for your own clusters):

```bash
# Install (Helm)
helm repo add argo https://argoproj.github.io/argo-helm
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd

# First login
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d           # the admin password
kubectl -n argocd port-forward svc/argocd-server 8080:443   # then open https://localhost:8080

# Connect a private repo, then register your app
argocd repo add https://github.com/you/your-repo --username x --password <token>
kubectl apply -f infra/k8s/argocd/api-app.yaml         # ArgoCD starts watching immediately
```

---

<a name="7"></a>
## 7. How THIS repo uses Kubernetes + ArgoCD

Now the concrete map. Everything below is verbatim from the manifests under
`infra/k8s/`. For the *pipeline* that changes the `image:` tags these manifests contain,
read [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md) — this
section is about the *cluster state* those tags land in.

### 7.1 The cluster and its namespaces

| | |
|---|---|
| **Cluster** | `autopilot-cluster-1` (GKE **Autopilot**) |
| **Region** | `us-west1` |
| **Project** | `gibp-ledger` |
| **Cloud SQL instance** | `gibp-ledger:us-west1:gibp-ledger-postgres` |

| Namespace | Holds |
|---|---|
| `gibp` | The apps: `api`, `company-webapp`, `super-webapp`, `kafka-worker`, plus the Cloud SQL proxy and the public Ingress. |
| `argocd` | ArgoCD itself and all the `Application` objects. |
| `gibp-stack` / `formance-system` | The Formance ledger stack (operator + `Stack`/`Ledger`/`Gateway`). |
| `external-secrets` | The External Secrets Operator. |
| `kafka-system` | The Strimzi Kafka operator (per its ArgoCD app's `watchNamespaces`). |

### 7.2 The per-app folder pattern

Every app is a folder `infra/k8s/apps/<app>/` containing a small, consistent set of
manifests. For `api`:

```
infra/k8s/apps/api/
├── deployment.yaml       replicas, image (git-sha), env + envFrom, resources, probes, anti-churn annotation
├── service.yaml          ClusterIP, 80 → 3000
├── pdb.yaml              minAvailable: 1
└── external-secret.yaml  GCP Secret Manager keys → the api-secrets Secret
```

**`api` Deployment** — the load-bearing manifest. Highlights (all verbatim):

- `replicas: 2` — HA (paired with the PDB).
- `image: us-west1-docker.pkg.dev/gibp-ledger/gibp-apps/api:<git-sha>` — **CI rewrites
  this one line** every deploy; ArgoCD then syncs it. Pinned to an immutable SHA, never
  `:latest`.
- Inline `env:` for **non-secret** config: `NODE_ENV=production`, `PORT=3000`,
  `APP_GLOBAL_PREFIX=api`, `FORMANCE_API_URL=http://gateway.gibp-stack.svc.cluster.local:8080`
  (note the cross-namespace cluster DNS), `GCP_PROJECT_ID=gibp-ledger`,
  `BQ_AUDIT_DATASET=gibp_audit_stag`, `GCS_BUCKET_NAME`, the `OTEL_*` toggles, etc.
- `envFrom: secretRef: api-secrets` — dumps **all** keys of the ESO-materialized Secret
  in as env vars (DB creds, JWT secrets, Formance ledger names, the BigQuery SA key,
  `KAFKA_BROKER`, …).
- `resources`: requests `cpu 200m` / `mem 256Mi`, limits `cpu 500m` / `mem 512Mi`.
- Three probes: startup `/api/health` (up to 150s to boot), readiness `/api/health/ready`
  (checks Postgres), liveness `/api/health` — the deliberate design from 4.6.
- The annotation **`cluster-autoscaler.kubernetes.io/safe-to-evict: 'false'`** — the
  anti-churn fix (7.5).

**`api` Service** — `ClusterIP`, `port 80 → targetPort 3000`, selector `app: api`.
Reachable inside the cluster as `api.gibp.svc.cluster.local`.

**`api` PDB** — `minAvailable: 1`, selector `app: api`.

**`company-webapp` Deployment** — the frontend contrast:

- `replicas: 2`, image pinned to git-sha, `containerPort: 80` (nginx serving static
  files).
- **Tiny resources**: requests `cpu 50m` / `mem 64Mi`, limits `cpu 200m` / `mem 128Mi`.
- **A single readiness probe** on `/` (port 80), no startup/liveness — it's just nginx.
- **No env vars at runtime.** All `VITE_*` config is *baked into the JS bundle at build
  time* (via `--build-arg`), so there's nothing to inject at runtime — the frontend's
  deep gotcha (Section 10 and the deployment guide §4.1c). Same anti-churn annotation.

### 7.3 The public entry: one Ingress for three apps

A single `Ingress` (`infra/k8s/ingress/ingress.yaml`, namespace `gibp`, name
`public-ingress`) is the whole public surface:

- **Host** `gibp.wisflux.com`.
- **TLS** via the `gibp-cf-origin-tls` Secret (certificate for that host).
- **A global static IP**, bound by the annotation
  `kubernetes.io/ingress.global-static-ip-name: "gibp-ingress-ip"`.
- **Path routing** (all to `:80` Services):

  | Path | Service |
  |---|---|
  | `/api` | `api` |
  | `/admin` | `super-webapp` |
  | `/` (everything else) | `company-webapp` |

That's the request path from 4.4, made real: Internet → Ingress (TLS + routing) →
Service → Pod.

### 7.4 Secrets via External Secrets Operator

Git never contains a secret *value* — only a *mapping*. Two objects:

- **`ClusterSecretStore` `gcp-secret-manager`**
  (`infra/k8s/cluster/external-secrets/secret-store.yaml`) — tells ESO how to reach the
  vault: provider `gcpsm`, `projectID: gibp-ledger`, authenticating via a `gcp-sa-key`
  Secret (`key.json`) in the `gibp` namespace. Cluster-scoped, so every namespace can
  reference it.
- **Per-app `ExternalSecret`** (e.g. `api-secrets`) — a mapping table: each entry maps a
  **GCP Secret Manager key** (`remoteRef.key: gibp-api-db-password`) to a **key in the
  resulting Kubernetes Secret** (`secretKey: DB_PASSWORD`), refreshed **every 1h**. ESO
  assembles a normal Secret named `api-secrets`, which the Deployment consumes via
  `envFrom`.

> **The failure mode to remember:** if an `ExternalSecret` lists a GCP key that doesn't
> exist yet, the *entire* `api-secrets` sync fails, the Secret is incomplete, and the pod
> won't start (`CreateContainerConfigError`). The manifests explicitly warn about this
> for `KAFKA_BROKER` and `LINKER_WEBHOOK_TOKEN`: **create the GCP secret first, then
> merge the mapping.** Full ESO mechanics are in **Chapter 8 (GCP)**.

### 7.5 Cloud SQL Auth Proxy — its own Deployment + Service

The managed Postgres is **not** exposed to the internet. Pods reach it through a proxy
(`infra/k8s/infrastructure/cloudsql-proxy.yaml`):

- A `Deployment` (`replicas: 1`) running `gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0`,
  args pointing at `gibp-ledger:us-west1:gibp-ledger-postgres`, listening on `:5432`.
- Runs as a **dedicated ServiceAccount** `cloudsql-proxy-ksa` (Workload Identity → a GCP
  identity allowed to reach exactly that DB — least privilege in action, 4.8).
- `securityContext.runAsNonRoot: true`.
- A `ClusterIP` `Service` named `cloudsql-proxy` on `:5432`, so the api simply connects
  to `cloudsql-proxy.gibp.svc.cluster.local:5432` as if it were a local Postgres — the
  proxy handles encryption + auth. No DB password on the wire, no public database.

### 7.6 Supporting operators, installed as ArgoCD apps

Beyond the four workloads, the platform's *operators* are themselves ArgoCD
Applications:

| ArgoCD Application | Source | Installs |
|---|---|---|
| `gibp-strimzi-operator` | Helm chart `strimzi-kafka-operator` `0.51.0` from `strimzi.io/charts/`, into `kafka-system` | The Kafka operator |
| Formance operator | (Formance stack manifests) | The ledger engine's CRDs + controller |
| External Secrets operator | (ESO install) | ESO + the `ExternalSecret`/`ClusterSecretStore` CRDs |

So the *entire* platform — apps and the machinery that runs Kafka, the ledger, and
secrets — is declared in git and reconciled by ArgoCD. One consistent model, top to
bottom.

### 7.7 The ArgoCD wiring

Under `infra/k8s/argocd/` there is one `Application` per workload/operator:
`api-app.yaml`, `company-webapp-app.yaml`, `super-webapp-app.yaml`, `kafka-worker-app.yaml`,
`kafka-app.yaml`, `strimzi-operator-app.yaml`. Every git-sourced one shares the same
shape (from `api-app.yaml`):

- `repoURL: https://github.com/gibillpay/gibp-ledger`
- `targetRevision: test` — **all apps watch the `test` branch.**
- `path: infra/k8s/apps/<app>`
- `destination.namespace: gibp` (into the app namespace)
- `syncPolicy.automated: { prune: true, selfHeal: true }`, `syncOptions: [CreateNamespace=true]`
- a `resources-finalizer.argocd.argoproj.io` finalizer for clean deletion.

**Adding a new app** is therefore: create `infra/k8s/apps/<app>/` (deployment, service,
pdb, external-secret), add `infra/k8s/argocd/<app>-app.yaml`, and apply that Application
once (`kubectl apply -f`) so ArgoCD starts watching. From then on it's pure GitOps. (Full
new-app checklist: deployment-lifecycle guide, Playbook E.)

### 7.8 The anti-churn lesson (a real GKE Autopilot incident)

Documented in [`../../infra/docs/pod-node-churn.md`](../../infra/docs/pod-node-churn.md):
app pods were being terminated and rescheduled onto different nodes **every ~7–10
minutes** with no load. Root cause: Autopilot's cluster autoscaler runs the
`OPTIMIZE_UTILIZATION` profile (fixed on Autopilot — you cannot switch it to `BALANCED`),
which aggressively drains underutilized nodes to pack pods onto fewer of them. The
workloads had *no guardrails*: no PDB, no `safe-to-evict` annotation, and `replicas: 1`
(so every move = a mini-outage). Two fixes, applied via git (a live `kubectl edit` would
be reverted by `selfHeal`):

- `cluster-autoscaler.kubernetes.io/safe-to-evict: 'false'` — the autoscaler won't drain
  the pod for scale-down (stops ~95% of moves). **Free on Autopilot**, since billing is
  per pod *request*, not per node.
- `replicas: 2` + PDB `minAvailable: 1` — real HA, so the *unavoidable* moves
  (node auto-upgrade/repair) are zero-downtime.

`kafka-worker` got the annotation but **stays `replicas: 1`** — it's a singleton
consumer; a second instance could double-process or reorder CDC/Kafka messages until
partitioning/idempotency is proven safe. This is the whole chapter's theory (4.3, 4.7)
hitting production reality.

---

<a name="8"></a>
## 8. Production concerns

| Concern | What to do | In this repo |
|---|---|---|
| **High availability** | `replicas ≥ 2` + a PDB + spread pods across nodes/zones (anti-affinity / topology spread). | `api` + both webapps: `replicas: 2` + PDB `minAvailable: 1`. `kafka-worker`: intentionally 1 (singleton). |
| **Resource right-sizing** | Set `requests` from real usage; keep a `limits` ceiling. On Autopilot **requests are your bill** — too high wastes money, too low gets you scheduled onto starvation or OOM-killed. | `api` 200m/256Mi req; webapps tiny (50m/64Mi). Revisit as traffic grows; add an HPA if load becomes spiky. |
| **Rolling-update safety** | Rely on readiness gating so a broken image stalls the rollout instead of taking traffic; keep old ReplicaSets for instant rollback. | Readiness `/api/health/ready` (Postgres); images pinned to `:<git-sha>`. |
| **Secret management** | Never put raw Secrets in git (base64 ≠ encryption). Use an external manager + operator; rotate in the manager. | GCP Secret Manager + ESO; git holds only mappings. → Chapter 8. |
| **RBAC least privilege** | Workloads run as narrow ServiceAccounts; humans get scoped Roles; no blanket cluster-admin. | Cloud SQL proxy's dedicated KSA via Workload Identity; ArgoCD holds the apply privileges, not individuals. |
| **Namespace isolation** | Group by trust/lifecycle; scope RBAC and network policy per namespace. | `gibp` / `argocd` / `gibp-stack` / `external-secrets` / `kafka-system`. |
| **Ingress / TLS** | Single L7 entry, HTTPS terminated, a stable static IP, routing by path. | `public-ingress`: `gibp.wisflux.com`, `gibp-cf-origin-tls`, `gibp-ingress-ip`. |
| **Autoscaling** | HPA for pods, cluster autoscaler for nodes; understand their interaction with PDBs. | Cluster autoscaler is Autopilot's (tamed with `safe-to-evict`); no HPA yet — add per 5.7 when needed. |
| **Observability** | Metrics, logs, and **events** are your eyes. `kubectl get events`, container logs, and (here) OpenTelemetry → Cloud Trace. | `OTEL_*` env in the api Deployment (ships disabled). Logs/events via `kubectl` (Section 9). |
| **Upgrades & node churn** | Managed control-plane/node upgrades are automatic on GKE; make workloads survive node moves. | The `safe-to-evict` + `replicas 2` + PDB pattern (7.8). |
| **Backups of cluster state** | `etcd` (the cluster's truth) must be backed up. | Managed by Google on GKE — you don't run `etcd`. Your *app* data is in Cloud SQL (its own backups). And GitOps means the desired state is *already* backed up: it's in git. |

---

<a name="9"></a>
## 9. Operations & debugging playbooks

### 9.1 The kubectl survival kit

```bash
# Point at the cluster (once)
gcloud container clusters get-credentials autopilot-cluster-1 --region=us-west1 --project=gibp-ledger

# See what's running / the shape of things
kubectl get pods -n gibp                     # pods + STATUS + RESTARTS + AGE
kubectl get deploy,svc,ingress -n gibp       # workloads, addresses, the front door
kubectl get pdb -n gibp                      # allowed disruptions
kubectl get events -n gibp --sort-by=.lastTimestamp   # the timeline — what k8s DID and why

# Zoom into one thing
kubectl describe pod <pod> -n gibp           # events at the bottom = the "why" for most failures
kubectl logs -f <pod> -n gibp                # app logs (migrations run at startup here)
kubectl logs <pod> -n gibp --previous        # logs of the PREVIOUS (crashed) container
kubectl exec -it <pod> -n gibp -- sh         # shell inside the container
kubectl port-forward svc/api 8080:80 -n gibp # hit a ClusterIP service from your laptop

# Rollouts
kubectl rollout status deploy/api -n gibp
kubectl rollout undo   deploy/api -n gibp    # emergency only — selfHeal reverts unless you fix git

# GitOps / secrets state
kubectl get applications -n argocd           # every app's SYNC STATUS + HEALTH
kubectl get externalsecret -n gibp           # did secrets sync? look for SecretSynced=True
```

### 9.2 Diagnosing the common pod failures

`kubectl describe pod` (read the **Events**) + `kubectl logs` solve almost all of these.

| Symptom (`kubectl get pods`) | Most likely cause | Where to look / fix |
|---|---|---|
| **`CrashLoopBackOff`** | App boots then exits (failed migration, missing required env var, bad config, unhandled startup error). | `kubectl logs <pod> --previous`. In this repo, often a failed DB migration or a config the app validates at boot. Fix the code/config, redeploy. |
| **`ImagePullBackOff` / `ErrImagePull`** | Wrong image name/tag, or no permission to pull from the registry. | `describe` → Events show the pull error. Check the `:<git-sha>` exists in Artifact Registry and node has pull perms. |
| **`CreateContainerConfigError`** | A referenced ConfigMap/**Secret key is missing** — here, the `api-secrets` sync failed because a GCP Secret Manager key doesn't exist. | `kubectl get externalsecret -n gibp` (SecretSynced?), `kubectl describe externalsecret api-secrets`. **Create the GCP secret, then it self-resolves.** (The 7.4 gotcha.) |
| **`Pending`** (never schedules) | No node can satisfy the pod's `requests`, or a taint/affinity blocks it. | `describe` → Events say "insufficient cpu/memory" or "0/N nodes available." On Autopilot, capacity is provisioned from requests — usually a *too-large* request or a quota limit. Right-size or check quota. |
| **`OOMKilled`** (in `describe` → Last State) | Container exceeded its memory **limit**. | Raise the memory `limit`/`request`, or fix a leak. `api` limit is 512Mi. |
| **Running, but no traffic** | **Readiness** probe failing → pulled from the Service. | `describe` → readiness events; hit `/api/health/ready` — here it means Postgres is unreachable (check the Cloud SQL proxy pod + the DB creds Secret). |
| **`Terminating`/`ContainerCreating` cycling every few minutes** | Autoscaler evicting for node consolidation. | The 7.8 anti-churn story — `safe-to-evict` + PDB + `replicas 2`. Confirm with `kubectl get events | grep -i scaledown`. |

### 9.3 Diagnosing ArgoCD

| Symptom | Meaning | Action |
|---|---|---|
| App **`OutOfSync`** and stays that way | Git changed but the apply failed, or a manifest is invalid. | `argocd app get <app>` or the UI diff; `kubectl describe application <app> -n argocd` → sync error. Fix the manifest and push. |
| App **`Degraded`** but **`Synced`** | The manifest applied fine, but the *workload* is unhealthy (crash loop, bad image, missing secret). | Ignore the sync — debug the pod (9.2). Fixing sync won't help; fix the app. |
| Change pushed to `test`, nothing happens | ArgoCD hasn't polled yet, or is watching a different branch/path. | Confirm `targetRevision: test` and `path`; hit **Refresh** in the UI or `argocd app get --refresh`. |
| A hand-edit "keeps getting undone" | `selfHeal: true` is doing its job — reverting you to git. | Stop hand-editing; **change the file and push** (Section 10). |

---

<a name="10"></a>
## 10. Gotchas & hard-won lessons

1. **A Kubernetes Secret is base64, not encrypted.** `kubectl get secret ... -o yaml`
   plus `base64 -d` reveals every value. Never commit a raw Secret to git; that's a
   plaintext password in git. Use External Secrets Operator (4.5, 7.4, Chapter 8).

2. **`selfHeal` reverts your manual `kubectl` edits — change git, not the cluster.** Any
   `kubectl edit`/`patch`/`scale` on an ArgoCD-managed resource is undone on the next
   reconcile loop. The *only* durable way to change production is to change a file and
   push. (This is a feature: it's what kills drift.)

3. **`requests`/`limits` mistakes cause eviction or OOM.** Requests too low → the
   scheduler crams the pod onto a starved node (or, under memory pressure, it's evicted
   first). Memory over the **limit** → instant `OOMKilled`. CPU over the limit →
   throttled (slow), not killed. On Autopilot, requests are also literally your bill.

4. **Readiness vs liveness confusion causes self-inflicted outages.** A too-aggressive
   *readiness* probe that checks a flaky downstream will yank your pod out of rotation
   over a blip. A too-aggressive *liveness* probe will *restart* a pod that was merely
   busy, deepening an overload. Readiness = traffic; liveness = restart. This repo's api
   deliberately keeps Formance/BigQuery *out* of the readiness check for exactly this
   reason (4.6).

5. **`:latest` defeats rollout and rollback.** If a Deployment references `:latest`, two
   pods can silently run different code, and "roll back to the previous version" is
   impossible because `latest` doesn't name a version. Always pin an immutable tag — here,
   `:<git-sha>`. Don't "simplify" a manifest to `:latest`.

6. **DNS / service-name assumptions.** Cross-namespace calls need the full
   `<svc>.<namespace>.svc.cluster.local`. `http://gateway:8080` from the `gibp` namespace
   won't reach Formance in `gibp-stack` — it must be
   `gateway.gibp-stack.svc.cluster.local:8080` (which is exactly what the api sets). A
   huge share of "can't connect" bugs is a wrong or missing namespace.

7. **Namespace mismatch.** A Service, its Deployment, and its ExternalSecret must live in
   the *same* namespace the ArgoCD `Application` deploys into (`gibp` here). A manifest
   with the wrong `metadata.namespace` (or none, defaulting to `default`) silently lands
   somewhere nothing else can find it.

8. **A PDB can *block* node drains.** `minAvailable` equal to (or above) the replica
   count means no pod may ever be voluntarily evicted, so node upgrades stall
   indefinitely. `minAvailable: 1` with `replicas: 2` leaves the needed headroom; keep
   that gap.

9. **Frontend config can't be changed on a running pod.** The webapps bake `VITE_*` into
   the JS at *build* time; there are no runtime env vars to edit. Fixing a wrong frontend
   API URL means changing the build value and **rebuilding the image** — not patching the
   Deployment. (Deployment guide §4.1c.)

10. **Branch-name drift in older docs.** Some comments and older docs say `demo`; the live
    pipeline and every ArgoCD `Application` watch **`test`**. When prose and a manifest
    disagree, **the manifest is the truth.**

---

<a name="11"></a>
## 11. Glossary + further reading

### Glossary (one line each)

- **Cluster / Node** — the managed pool of machines / one machine in it.
- **Control plane** — the cluster's brain: `api-server`, `scheduler`, `controller-manager`, `etcd`.
- **`etcd`** — the consistent key-value store; the single source of truth for cluster state.
- **`kubelet` / `kube-proxy`** — the node agent that runs containers / programs Service networking.
- **Pod** — the smallest deployable unit; one (or a few) containers sharing an IP.
- **ReplicaSet** — keeps N identical pods running (the self-healing layer).
- **Deployment** — manages ReplicaSets for rolling updates + rollback on top of "keep N pods."
- **Service** — a stable virtual IP + DNS name load-balancing to pods by label selector.
- **ClusterIP / NodePort / LoadBalancer** — internal-only / node-port / cloud external Service types.
- **Ingress** — L7 router: one public entry, TLS termination, host/path routing.
- **ConfigMap / Secret** — non-secret / "secret" (base64!) key-values, injected as env or files.
- **Namespace** — a named partition of the cluster for grouping + isolation + scoped RBAC.
- **Label / Selector** — tags on objects / a query that loosely couples (Service→pods).
- **Job / CronJob** — run-to-completion pod / on a schedule.
- **StatefulSet / DaemonSet** — stable-identity stateful pods / one pod per node.
- **PV / PVC / StorageClass** — real storage / a pod's claim on it / a template that provisions it.
- **requests / limits** — guaranteed CPU/RAM / hard ceiling (memory-over-limit ⇒ OOMKilled).
- **Probe** — startup (booting?) / readiness (traffic?) / liveness (alive?).
- **HPA / PDB** — autoscale pods on a metric / keep ≥N pods during voluntary disruptions.
- **ServiceAccount / Role / RoleBinding** — a workload identity / allowed verbs / the grant (RBAC).
- **CRD / Operator** — teaches the API server a new kind / a controller that reconciles that kind.
- **Helm / chart / release** — package manager / a templated bundle / an installed, tracked instance.
- **GitOps** — git is the desired state; a controller continuously reconciles the cluster to it.
- **ArgoCD / Application** — the GitOps reconciler / the CRD that says "watch this git path+branch."
- **Sync status / Health** — cluster matches git? / are the resources actually working?
- **prune / selfHeal** — delete cluster resources removed from git / revert manual cluster edits.
- **Sync wave / hook** — apply-order control / run a resource at PreSync/Sync/PostSync.
- **App-of-apps** — one root Application that creates many child Applications.
- **GKE Autopilot** — Google-managed Kubernetes where Google runs the nodes; billed per pod request.
- **External Secrets Operator (ESO)** — copies vault secrets (GCP Secret Manager) into k8s Secrets.

### Further reading

**Kubernetes**
- Concepts (the canonical docs): https://kubernetes.io/docs/concepts/
- kubectl reference & cheat sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- Configure liveness/readiness/startup probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Managing resources (requests/limits): https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

**Local clusters**
- kind: https://kind.sigs.k8s.io/  •  minikube: https://minikube.sigs.k8s.io/docs/

**ArgoCD**
- Documentation home: https://argo-cd.readthedocs.io/
- Application CRD spec: https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/
- Sync options, waves & hooks: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
- App-of-apps pattern: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/

**Helm**
- Docs & chart guide: https://helm.sh/docs/

**GKE**
- Autopilot overview: https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview
- get-credentials: https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials

**In this repo (read the real thing — manifests are the truth)**
- The full CI/CD pipeline: [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md)
- ArgoCD install/connect commands: [`../../gibp-docs/cicd/06-argocd-setup.md`](../../gibp-docs/cicd/06-argocd-setup.md)
- Kubernetes deployment reference: [`../../gibp-docs/cicd/04-kubernetes-deployment.md`](../../gibp-docs/cicd/04-kubernetes-deployment.md)
- The node-churn incident + fix: [`../../infra/docs/pod-node-churn.md`](../../infra/docs/pod-node-churn.md)
- Secrets deep dive: **Chapter 8 — GCP** (`08-gcp.md`)
- The manifests themselves: [`../../infra/k8s/`](../../infra/k8s/)

---

*You've now got both halves: Kubernetes as a technology — the object model, the control
loop, scheduling, networking, storage, config, health, RBAC, autoscaling, operators, and
Helm — and ArgoCD as the GitOps engine that keeps the cluster equal to git. Do the
Section 5 lab once on kind; after that, reading and reasoning about this repo's `infra/k8s/`
manifests will feel like common sense.*
