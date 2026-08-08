# Kubernetes, from zero to usable

A companion to [architecture.md](architecture.md). That document explains how one
real system is built; this one explains the vocabulary it uses, assuming you have
never touched Kubernetes.

Everything here is illustrated with that system's **real** configuration, trimmed of
comments. Nothing is invented for teaching purposes, so when you finish this page you
can read the architecture document — and the actual repository — without translating.

This guide is intentionally repetitive in a good way: every major section ends with a
single sentence you can memorize, so the whole platform stays in your head after one
read.

## How to use this guide (no background assumed)

Use this exact sequence and do not skip steps:

1. Read the mental-model block first: one idea, control plane, kubectl, nouns, and labels/selectors.
2. Memorize the map: 1 idea -> 4 nouns -> 4 workload types -> 7 use cases.
3. Read workload types and all seven use cases, and say each "Say it in one line" sentence out loud.
4. Do the "First lab in 20 minutes" checklist immediately after the seven use cases.
5. Read the surprise block last: storage, identity, secrets, network defaults, CRDs/operators, and packaging.

If you get stuck, use this fallback loop:

```
confused -> find the nearest "Say it in one line"
         -> map it to one noun (Pod / Service / Deployment)
         -> return to the full section
```

## Memory map (the whole platform in one screen)

```
ONE IDEA
  desired state loop

FOUR NOUNS (nested)
  Cluster -> Node -> Pod -> Container

FOUR WORKLOAD CHOICES
  Deployment | StatefulSet | DaemonSet | Job/CronJob

SEVEN DAILY USE CASES
  run | address | publish | config | health | size | update

SIX SURPRISES
  storage | identity | secrets | network defaults | CRDs/operators | packaging
```

## Common first-week mistakes (and fast fixes)

- Mistake: debugging from pod names and IPs. Fix: debug from labels and Services.
- Mistake: treating `Running` as healthy. Fix: trust readiness, not process state.
- Mistake: setting `replicas` and HPA on the same workload. Fix: one owner per
  number.
- Mistake: adding default-deny policy without DNS allow. Fix: allow DNS first.
- Mistake: assuming Secrets are encrypted. Fix: treat them as sensitive plaintext.

---

## Contents

**Start here (no skip)**
1. [How to use this guide](#how-to-use-this-guide-no-background-assumed)
2. [Memory map](#memory-map-the-whole-platform-in-one-screen)
3. [Common first-week mistakes](#common-first-week-mistakes-and-fast-fixes)

**The mental model**
4. [The one idea underneath everything](#the-one-idea-underneath-everything)
5. [Who actually runs the loops: the control plane](#who-actually-runs-the-loops-the-control-plane)
6. [How you talk to it: `kubectl`](#how-you-talk-to-it-kubectl)
7. [The nouns, nested](#the-nouns-nested)
8. [Labels, selectors, and annotations](#labels-selectors-and-annotations)

**Choosing what to run**
9. [The four kinds of workload](#the-four-kinds-of-workload--pick-by-how-the-thing-behaves)

**The seven things people actually use it for**
10. [Keep my program running](#use-case-1--keep-my-program-running)
11. [Give it a stable address](#use-case-2--give-it-a-stable-address)
12. [Let the internet reach it](#use-case-3--let-the-internet-reach-it)
13. [Give it config and passwords](#use-case-4--give-it-config-and-passwords-without-baking-them-in)
14. [Only send traffic when it's ready](#use-case-5--only-send-traffic-when-its-actually-ready)
15. [Reserve the resources it needs](#use-case-6--reserve-the-resources-it-needs)
16. [Update it without dropping anyone](#use-case-7--update-it-without-dropping-anyone)
17. [First lab in 20 minutes](#first-lab-in-20-minutes-no-skipped-steps)

**The parts that surprise people**
18. [Storage: how a disposable pod keeps data](#storage-how-a-disposable-pod-keeps-data)
19. [What identity does a pod run as?](#what-identity-does-a-pod-run-as)
20. [Secrets are not encrypted](#secrets-are-not-encrypted)
21. [Pulling a private image](#pulling-a-private-image-imagepullsecrets)
22. [The network starts wide open](#the-network-starts-wide-open)
23. [Extending the API: custom resources and operators](#extending-the-api-custom-resources-and-operators)
24. [Packaging: Helm, Kustomize, and overlays](#packaging-helm-kustomize-and-overlays)

**Reference**
- [The pod states you will actually see](#the-pod-states-you-will-actually-see)
- [Glossary](#glossary) — the Kubernetes vocabulary
- [Glossary, part two](#glossary-part-two-the-terms-that-arent-kubernetes) — reliability, cryptography, and the browser

---

## The one idea underneath everything

> **In plain English:** Kubernetes is a manager you give a wish to. You don't say
> "start this program on that machine." You say "one copy of this should always be
> running," and it keeps making that true — restarting, replacing, rescheduling —
> without you watching.

Almost every tool you have used is **imperative**: you give a command and it happens
once. `./myserver &` starts a server. If it crashes at 3 a.m., it is simply not
running any more, because nothing was ever responsible for it.

Kubernetes is **declarative**: you write down the state you want, hand it over, and a
control loop spends forever comparing reality against that description and fixing the
difference.

```
   IMPERATIVE (a shell script)        DECLARATIVE (Kubernetes)
   ---------------------------        ------------------------
   "start the server"                 "one server should be running"
        |                                   |
        v                                   v
   it starts                          a loop checks, forever:
        |                                   |
        v                              +----+------------------+
   it crashes                          | is one running?       |
        |                              +----+-------------+----+
        v                                   | yes         | no
   nothing happens.                         v             v
   It stays down until                   do nothing    start one
   a human notices.                          |             |
                                             +------+------+
                                                    |
                                            (repeat forever)
```

That loop is the entire product. Everything else is just *what* you are allowed to
declare.

> **Say it in one line:** you don't run programs, you describe them — and a loop
> spends forever making reality match the description.

## Who actually runs the loops: the control plane

A fair question the metaphor above dodges: *who* is running these loops, and where?

A cluster splits into two halves. The **control plane** is the brain: it stores what
you asked for and runs the loops. The **nodes** are the muscle: they run your actual
containers.

```
   CONTROL PLANE  (managed by the cloud provider here -- you never see these pods)
   +---------------------------------------------------------------+
   |                                                               |
   |   API SERVER  <-- the ONLY way in. Everything talks to this,  |
   |       |           including every other component. It         |
   |       |           authenticates, authorizes, and validates.   |
   |       v                                                       |
   |   etcd            the database of record. Your entire         |
   |                   cluster is rows in here.                    |
   |                                                               |
   |   SCHEDULER       watches for pods with no node assigned,     |
   |                   picks a node with room, writes it back.     |
   |                                                               |
   |   CONTROLLER      runs the "make reality match" loops --      |
   |   MANAGER         Deployment, ReplicaSet, Job, and friends.   |
   |                                                               |
   +---------------------------------------------------------------+
                                |
             every node polls the API server for its assignments
                                |
   NODES  (your actual computers)
   +---------------------------------------------------------------+
   |   KUBELET     the agent on each node. Asks "what should I be  |
   |       |       running?", then starts and supervises it.       |
   |       v                                                       |
   |   your containers                                             |
   +---------------------------------------------------------------+
```

Three consequences worth carrying forward:

- **For control decisions, components coordinate through the API server.** The
  scheduler doesn't phone the kubelet. It writes "this pod goes on node B" into the
  API server, and the kubelet on node B reads it. Every control-plane component is a
  loop watching the same database. This is why RBAC on the API server is a core part
  of cluster security.
- **The scheduler only *decides*; the kubelet *acts*.** They are separate loops that
  never speak directly. In the architecture document this is exactly why the pod
  autoscaler and node autoscaler can hand off to each other without knowing each
  other exists — they both just read and write the same shared state.
- **"The cluster is down" usually means the control plane is unreachable.** Your
  containers keep running: the kubelet is still supervising them locally. You just
  can't *change* anything, and nothing can be rescheduled.

On managed Kubernetes — as here — the cloud provider runs the control plane for you.
You never see those pods, you never patch them, and you reach them at a single URL.

> **Say it in one line:** every component is a loop watching one database through one
> API server — which is why that API server is both the whole interface and the whole
> attack surface.

## How you talk to it: `kubectl`

`kubectl` is the command-line client. It is a thin wrapper over HTTP calls to the API
server, which matters more than it sounds: anything `kubectl` can do, a script or a CI
job can do with the same credentials.

The handful of commands that cover most days:

```
   kubectl apply -f thing.yaml     send a description; the loop makes it true
   kubectl get pods                what exists right now
   kubectl describe pod NAME       the detail, INCLUDING recent events
   kubectl logs NAME               that container's stdout
   kubectl exec -it NAME -- sh     a shell inside the container
   kubectl delete pod NAME         delete it (a Deployment will replace it)
```

When debugging, use this fixed order so you do not waste time:

```
1) kubectl get       -> what exists
2) kubectl describe  -> why it is in this state (Events is the gold mine)
3) kubectl logs      -> what your process says
4) kubectl exec      -> only if you truly need an interactive check
```

**`describe` is the one beginners under-use.** Its `Events:` section at the bottom is
usually where the actual answer is — "no node had enough CPU," "image pull failed,"
"readiness probe returned 500." A pod stuck in `Pending` tells you nothing from `get`;
`describe` tells you exactly why in one line.

Two notes specific to the system this documents. The images are **distroless** — no
shell inside — so `kubectl exec -it ... -- sh` fails by design. And for most of that
project's build the API server was unreachable from the developer's laptop entirely,
so every one of these commands ran inside CI instead.

> **Say it in one line:** `kubectl` is just HTTP to the API server — and when
> something is stuck, `describe` and read the events.

## The nouns, nested

Four nouns — but **not four of the same kind of relationship**, and that difference is
the thing worth learning:

```
   CLUSTER                  the whole managed system
     |
     |  CONTAINS        <-- structural. A node belongs to a cluster.
     v
   NODE                     one actual computer.
     :
     :  RUNS, FOR NOW   <-- NOT containment. The scheduler PLACED this pod
     :  (an assignment)     here, and the replacement may land elsewhere.
     v                      This is the only link that ever moves.
   POD                      the smallest thing that gets scheduled,
     |                      and the smallest thing with an address.
     |  CONTAINS        <-- structural. Fixed for this pod's whole life.
     v
   CONTAINER                your compiled program plus everything it needs
                            to run, sealed into one image.
```

**The dotted link is the whole story.** Two of those relationships are a folder inside
a folder. The middle one is a *placement* — a decision the scheduler made, and remakes
every time a pod is replaced. Everything strange about Kubernetes follows from that one
link being temporary.

(In this project there is one node that always runs, plus a second that appears only
under load and then vanishes again.)

### Why does Pod exist at all?

If a pod almost always holds exactly one container, it looks like pointless
indirection — why not schedule containers directly? Because a pod buys two specific
things:

- Every container in a pod **shares one IP address and one network namespace**. They
  reach each other on `localhost`.
- They are **always scheduled onto the same node**, and they live and die together.

So a pod is not "a container." It is **the smallest unit Kubernetes will schedule and
give an address to**. When there *is* a second container, it is usually a helper — a
log shipper, a proxy — that has to sit right beside the main one to do its job.

### The one that trips people up

A pod is not a long-lived thing you name and care about, the way you care about a
server called `web-01`. It is closer to a paper cup: identical to every other one,
thrown away without ceremony, replaced instantly — and replaced *by the control loop
from the first section*, which noticed the wish was no longer true and acted.

Anything that must survive the cup has to live somewhere else. That is not a slogan,
it is a checklist:

```
   what a pod appears to hold      where it must ACTUALLY live to survive
   --------------------------      --------------------------------------
   data on disk              -->   a PersistentVolume  (a separate object)
   an identity               -->   a ServiceAccount
   an address                -->   a Service
```

**The address is the sharpest case.** A pod gets its own IP — and that IP dies with it.
The replacement comes back with a different one. So nothing can safely remember a pod's
address, which is precisely the problem the next use cases solve.

Two more nouns surround pods rather than being pods — one groups them, one feeds them:

| Noun | What it actually means |
| --- | --- |
| **Namespace** | A folder inside the cluster that pods live *in*. Permissions can be made to stop at its edge. |
| **Secret / ConfigMap** | Values injected *into* a pod at start-up, so the image holds no environment-specific data. |

> **Say it in one line:** a pod is the smallest thing that gets scheduled and the
> smallest thing with an address — and both of those are temporary.

## Labels, selectors, and annotations

Kubernetes has no concept of "this object points at that object." There are no
foreign keys and no names wired between objects. Instead, objects wear **labels**, and
other objects carry **selectors** that query for them.

```
   A POD wears labels:              A SERVICE carries a selector:

     labels:                          selector:
       app: big2-server                 app: big2-server
       tier: backend                          |
            ^                                 |
            |                                 v
            +---- matched at runtime, continuously -----+

   Nothing is bound. The Service does not know the pod's name or IP.
   It asks "who currently wears app=big2-server?" and gets today's answer.
```

This is why a pod can die and its replacement — different name, different IP — is
picked up with no configuration change anywhere. It is also why a *typo* in a label is
such a nasty bug: nothing errors. The selector simply matches nothing, and you get an
address that routes to a void.

**Annotations look identical and do the opposite job.** Same key-value shape, but
they are never used for selection. They carry configuration *for other tools* to read:

```yaml
metadata:
  labels:                                    # for SELECTING this object
    app: big2-server
  annotations:                               # for TOOLS to read
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
```

Those two annotations are doing real work. The first is how the certificate manager
learns it should issue a certificate for this Ingress. The second reconfigures the
ingress controller's proxy timeout. Neither is part of the Ingress specification —
they are messages addressed to specific controllers, and the prefix (`cert-manager.io/`,
`nginx.ingress.kubernetes.io/`) says who they are for.

> **Say it in one line:** labels are for finding things and annotations are for
> telling tools things — and a mistyped label fails silently, matching nothing.

## The four kinds of workload — pick by how the thing behaves

You almost never create a pod directly. You create a **controller** that creates pods
for you, and there are four to choose from. Choosing wrong is one of the most common
beginner mistakes, and the choice is not about what your program *is* — it is about
how it needs to *behave*.

```
   Does it ever finish on its own?
        |
        +-- YES --> does it need to run on a schedule?
        |             |
        |             +-- YES --> CRONJOB    (runs a Job on a clock)
        |             +-- NO  --> JOB        (run once, to completion)
        |
        +-- NO, it runs forever
              |
              +-- must a copy run on EVERY node?
                    |
                    +-- YES --> DAEMONSET    (one per node, automatically)
                    |
                    +-- NO --> does each copy need a STABLE name
                               and its OWN disk that follows it?
                                 |
                                 +-- YES --> STATEFULSET
                                 +-- NO  --> DEPLOYMENT   <-- the common case
```

All four are in use in the system this documents:

| Controller | It guarantees | Real example there |
| --- | --- | --- |
| **Deployment** | N interchangeable copies; any one can be replaced by any other. Pods get random names. | The game server, both React frontends, the ingress controller. |
| **StatefulSet** | Ordered, *stable* identities — `pod-0`, `pod-1` — each keeping its own volume across restarts. | The metrics and log stores, created by their own charts. |
| **DaemonSet** | Exactly one copy on every node, including nodes that appear later. | The log collector. It tails every pod's output, so it must exist wherever pods do. |
| **CronJob → Job** | Runs to completion on a schedule, then stops. Not restarted forever. | The nightly base backup of the database, at 03:00. |

**The DaemonSet is the one worth pausing on**, because it inverts the usual question.
With a Deployment you ask *"how many copies do I want?"* With a DaemonSet you never
specify a number at all — the answer is always "however many nodes exist." Add a node
and a copy appears there automatically:

```
   DEPLOYMENT: "2 copies, anywhere"      DAEMONSET: "1 copy per node, always"

   node A  [ pod ] [ pod ]               node A  [ collector ]
   node B  (empty)                       node B  [ collector ]
                                             |
   a new node C appears:                 a new node C appears:
   node C  (empty -- nothing moves)      node C  [ collector ]  <-- automatic
```

That is exactly why the log collector is a DaemonSet. Logs are written on the node the
pod runs on, so a collector that only existed on *some* nodes would silently miss the
logs of everything running elsewhere — and missing logs look identical to no logs.

> **Say it in one line:** pick the controller by behaviour, not by what the program
> is — finishes or not, every node or not, stable identity or not.

## Use case 1 — "keep my program running"

Memory card:
- Need: a process should run forever.
- Object: Deployment.
- First failure pattern: `CrashLoopBackOff` when your process exits.

The most common thing anyone uses Kubernetes for, and the reason **Deployment** is the
default answer among the four controllers above: a web server has no finish line, runs
on whichever node has room, and any copy is as good as any other. You describe the
program; the control loop keeps it alive. This is the game server, shortened:

```yaml
kind: Deployment
metadata:
  name: big2-server
spec:
  selector:
    matchLabels:
      app: big2-server        # which pods this Deployment owns
  template:                   # the cookie-cutter for every pod it makes
    metadata:
      labels:
        app: big2-server      # the stamp each pod is born with
    spec:
      containers:
        - name: big2-server
          image: big2-server:latest
          ports:
            - name: http
              containerPort: 8080
```

Read it as a sentence: *"pods carrying the label `app: big2-server` should exist, and
here is how to build one."* Kill the pod by hand and a replacement appears in seconds,
because deleting a pod does not change the wish.

**Notice what is missing: there is no `replicas` field.** A Deployment normally takes
one — `replicas: 3` means three copies — and leaving it out defaults to one. Here it
is omitted *on purpose*, because in production a pod autoscaler owns that number.
Setting it in both places is a documented anti-pattern: every `kubectl apply` would
reset the count and fight the autoscaler, which then scales it back, forever.

> **The rule:** exactly one thing should own any given number. If an autoscaler owns
> the replica count, the manifest must not also state it.

## Use case 2 — "give it a stable address"

Memory card:
- Need: callers should not care when pods are replaced.
- Object: Service.
- First failure pattern: selector typo matches zero pods.

Pods get a fresh IP every time they are replaced, so nothing can safely dial a pod
directly. A **Service** is the stable name in front of them:

```yaml
kind: Service
metadata:
  name: big2-server
spec:
  type: ClusterIP             # reachable inside the cluster only
  selector:
    app: big2-server          # <-- the SAME label the Deployment stamps on
  ports:
    - name: http
      port: 80                # what callers dial
      targetPort: http        # the container port it forwards to
```

```
   other pods dial "big2-server:80"
                 |
                 v
        +------------------+
        |     SERVICE      |   holds no traffic itself -- it is a rule
        +------------------+   for finding whoever currently matches
                 |
        "who has label app=big2-server AND passes readiness?"
                 |
        +--------+---------+
        v                  v
   [ POD 10.4.1.7 ]   [ POD 10.4.2.9 ]
        |
        |  this pod dies; a new one starts at 10.4.3.2
        v
   the Service picks it up automatically -- no config changed anywhere
```

This is why the architecture document says *"nothing in the middle is wired by IP."*
It is not a nicety; it is the only reason disposable pods can work at all.

**A Service does two jobs, and the second is easy to miss.** It is a stable *address*,
and it is a *load balancer*. When several pods match the selector, each new connection
goes to one of them — roughly evenly, and only ever to pods currently passing their
readiness probe. You do not configure this and there is nothing to install; it is what
a Service is.

```
   ONE address, N healthy pods behind it

   caller ---> big2-server:80 ---+---> [ pod A ]  Ready    <- gets traffic
                                 +---> [ pod B ]  Ready    <- gets traffic
                                 +---> [ pod C ]  NOT ready <- skipped entirely

   Scale from 1 to 3 pods and callers change nothing. They were never
   talking to a pod; they were talking to the Service.
```

### How does the name `big2-server` even resolve?

The diagram above has pods dialling `big2-server:80` as if that were an ordinary
hostname. It is — because the cluster runs its **own DNS server**, and every Service
automatically gets a record in it.

```
   pod asks its resolver for "big2-server"
             |
             v
   [ cluster DNS ]  every Service has a record here, created automatically
             |
             v
   returns the Service's stable IP -> the Service load-balances to a healthy pod

   The full name is:   big2-server.big2.svc.cluster.local
                       ^^^^^^^^^^^ ^^^^ ^^^ ^^^^^^^^^^^^^
                       service     ns   |   cluster suffix
                                        +-- "this is a Service"

   Inside the same namespace, the short name is enough. From another
   namespace you need `big2-server.big2`.
```

Two things follow that catch people out. Service discovery is **just DNS** — there is
no registry to configure and no client library to import; anything that can resolve a
hostname can find a Service. And because it is DNS, a network policy that forgets to
allow DNS breaks *everything* in a uniquely confusing way: pods can still reach each
other by raw IP, but no name resolves, so failures look like random timeouts rather
than a firewall problem. That is exactly the trap the architecture document flags in
its default-deny policies.

## Use case 3 — "let the internet reach it"

Memory card:
- Need: one public entry point for many internal services.
- Object: Ingress.
- First failure pattern: TLS and timeout annotations missing for long-lived traffic.

A `ClusterIP` Service is private. To publish it you add an **Ingress** — one public
entrance that routes by hostname and path, and terminates HTTPS:

```yaml
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
spec:
  tls:
    - hosts: [big2.example.com]
      secretName: big2-tls          # cert-manager creates and renews this
  rules:
    - host: big2.example.com
      http:
        paths:
          - path: /ws                # WebSocket  -> Go server
          - path: /auth              # login      -> Go server
          - path: /telegram          # bot hook   -> Go server
          - path: /queue             # counts     -> Go server
          - path: /                  # everything else -> the React page
```

```
internet user
    |
    v
cloud load balancer (public IP)
    |
    v
ingress controller pod
    |
    +--> path /ws, /auth, /queue -> Service A -> ready game-server pods
    |
    +--> path /                   -> Service B -> ready frontend pods
```

Two details in there are worth more than they look. `secretName` is filled in
automatically by the certificate manager — you never paste a certificate. And
`proxy-read-timeout: "3600"` is the fix for a real outage: the default is 60 seconds,
which silently killed WebSockets belonging to players waiting at a table.

## Use case 4 — "give it config and passwords without baking them in"

Memory card:
- Need: same image in every environment, different runtime values.
- Object: ConfigMap and Secret references.
- First failure pattern: secret value committed to git by accident.

The image must be identical in every environment, so anything environment-specific is
injected at start-up. Plain values inline, sensitive values by reference:

```yaml
env:
  - name: PORT
    value: "8080"                    # plain config, fine in git
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:                  # a reference, NOT the value
        name: big2-db-app            # the database operator created this
        key: uri
        optional: true               # missing -> the app runs without a DB
```

```
your image (immutable)
    |
    +--> plain env value from manifest: PORT=8080
    |
    +--> secretKeyRef -> Secret object -> key uri -> DATABASE_URL at startup
```

The database password never appears in this file, in git, or in anyone's terminal —
the operator mints it directly into the cluster and this only names it. `optional:
true` is the small touch that lets the same manifest run locally with no database at
all.

## Use case 5 — "only send traffic when it's actually ready"

Memory card:
- Need: avoid sending users to half-started pods.
- Object: readiness and liveness probes.
- First failure pattern: database check in liveness causes restart storms.

Two health checks that sound similar and do opposite things:

```yaml
readinessProbe:                 # "should you get traffic?"
  httpGet: { path: /readyz, port: http }
  periodSeconds: 5
livenessProbe:                  # "are you alive, or should I kill you?"
  httpGet: { path: /healthz, port: http }
  periodSeconds: 10
```

```
   READINESS fails  ->  pod is REMOVED from the Service's backend list
                        but keeps running. Traffic stops; the pod recovers.

   LIVENESS  fails  ->  pod is KILLED and restarted.
```

Getting these backwards is a classic outage: put a database check in the *liveness*
probe and a brief database blip restart-loops every pod in your fleet. Here `/readyz`
checks the database and `/healthz` does not — so a database outage quietly removes the
pod from rotation instead of killing it.

## Use case 6 — "reserve the resources it needs"

Memory card:
- Need: predictable scheduling and safe CPU/memory ceilings.
- Object: `requests` and `limits`.
- First failure pattern: pod stays `Pending` because requests do not fit any node.

```yaml
resources:
  requests:                     # a RESERVATION -- used for scheduling
    cpu: 100m                   # 100 millicores = 10% of one core
    memory: 128Mi
  limits:                       # a CEILING -- exceed it and you're throttled/killed
    cpu: "1"
    memory: 512Mi
```

**`requests` is the single most misunderstood field in Kubernetes.** It is not what
your program uses; it is what the scheduler *sets aside* for it. A node with 1 core
fits ten pods requesting `100m` each — even if all ten sit completely idle, and
even if one of them is actually burning 200m. The scheduler does arithmetic on the
requests, not on reality.

```
   NODE: 1000m CPU total
   +--------------------------------------------------------------+
   | requested: 900m                              | free: 100m    |
   +--------------------------------------------------------------+
                                                        ^
   A new pod requesting 200m does NOT fit here, no matter how idle
   the node actually is. It goes Pending and waits -- forever, if
   nothing frees up.
```

That is exactly how that project's monitoring stack "installed successfully" and then
sat unschedulable for 23 hours.

One more asymmetry worth knowing: exceeding a **CPU** limit throttles you — your
program just runs slower. Exceeding a **memory** limit kills the container outright,
with `OOMKilled` in its status. CPU is elastic; memory is a wall.

## Use case 7 — "update it without dropping anyone"

Memory card:
- Need: deploy a new version while the old one still serves.
- Object: Deployment rolling update.
- First failure pattern: rollout hangs because there is no spare capacity.

A rolling update replaces pods gradually instead of all at once:

```
   start: [ v1 ]

   1. start a v2 pod, wait for its READINESS probe
      [ v1 ]  [ v2 starting... ]

   2. v2 is Ready -> the Service starts sending it traffic
      [ v1 ]  [ v2 READY ]

   3. send v1 SIGTERM, let it finish, remove it
      [ v2 READY ]

   If v2 NEVER becomes Ready, the rollout stops and v1 keeps serving.
   A broken deploy fails safe -- provided something is watching for it.
```

Underneath, that is **two ReplicaSets** — one per version. The Deployment grows the
new one and shrinks the old one, pod by pod. You never write a ReplicaSet, but they
are what `kubectl get all` shows you during a deploy, and seeing two is normal rather
than alarming.

That last line is why that project's pipeline blocks on the rollout actually reporting
healthy: without that gate, a deploy that never becomes Ready still reports success,
and you are left with a green checkmark and an old pod.

Note step 1 needs room for a v2 pod *while v1 is still running*. On a node with no
spare capacity, a rolling update cannot start — it hangs rather than failing cleanly.

> **Say it in one line:** those seven cover almost everything — run it, address it,
> publish it, configure it, health-check it, size it, and update it safely.

## First lab in 20 minutes (no skipped steps)

Goal: build one tiny service, publish it inside the cluster, then safely update it.

```
step 1  create a namespace
step 2  apply one Deployment (1 pod)
step 3  apply one Service (stable name)
step 4  verify with kubectl get and kubectl describe
step 5  add readiness and liveness probes
step 6  add requests and limits
step 7  scale from 1 to 2 replicas and verify load-balancing
step 8  change image tag and watch a rolling update complete
```

Suggested command sequence:

```
kubectl create namespace demo
kubectl apply -n demo -f deployment.yaml
kubectl apply -n demo -f service.yaml
kubectl get pods,svc -n demo
kubectl describe pod <name> -n demo
kubectl set image deployment/myapp myapp=myapp:v2 -n demo
kubectl rollout status deployment/myapp -n demo
```

If any step fails, return to this fixed debug loop:

```
get -> describe -> events -> logs
```

## Storage: how a disposable pod keeps data

If pods are paper cups, the obvious question is how a database survives at all. The
answer is that storage is a **separate object with a separate lifetime**, and the pod
merely borrows it.

```
   WITHOUT a volume                      WITH a persistent volume
   ----------------                      ------------------------
   [ POD ]                               [ POD ] ---- mounts ----> [ PV ]
     |  writes to the container's          |                        |
     |  own filesystem                     | pod dies               | survives
     v                                     v                        |
   pod dies -> DATA GONE                 [ NEW POD ] -- remounts -> [ PV ]
                                                                    |
                                         the disk is a real cloud volume,
                                         independent of any pod
```

Three objects, and the indirection between them is the point:

| Object | What it is |
| --- | --- |
| **PersistentVolume (PV)** | The actual piece of storage — a real cloud block volume. |
| **PersistentVolumeClaim (PVC)** | A *request* for storage: "I need 50Gi that I can read and write." The pod references this, never the PV. |
| **StorageClass** | The recipe for creating a PV on demand: which cloud disk type, which parameters. |

The pod asks for a claim; the claim gets matched to (or triggers creation of) a
volume. That indirection is what lets the identical manifest run on a laptop and in
the cloud — only the StorageClass differs.

Here is the real request from the database in that project:

```yaml
storage:
  size: 50Gi
  storageClass: oci-bv        # the cloud's block-volume driver
```

Two details worth stealing. The workload needs about 2Gi, but it asks for **50Gi**
because that cloud's block volumes have a 50GB minimum — asking for less doesn't save
money, it just fails to provision. And the volume being separate from the pod is what
made a risky node migration survivable in that project: terminating the machine was
safe precisely because the data was never on the machine's boot disk.

**The catch nobody warns you about.** A cloud volume can usually only be attached to
one node at a time. So a pod using one can't freely move: the volume has to be
detached from the old node before it can attach to the new one. When the old node is
unreachable, that detach waits for a timeout — which is why a *forced* stateful
migration in that project cost about six minutes rather than seconds.

> **Say it in one line:** storage is a separate object with its own lifetime and the
> pod only borrows it — which is why the data outlives the pod that wrote it.

## What identity does a pod run as?

Everything reaching the API server needs an identity, including programs running
*inside* the cluster. That identity is a **ServiceAccount**.

```
   [ POD ]
      |  serviceAccountName: cluster-autoscaler
      v
   [ SERVICEACCOUNT ]  an identity that exists inside the cluster
      |
      |  a RoleBinding / ClusterRoleBinding connects it to...
      v
   [ ROLE ]  a list of allowed verbs on allowed resource kinds
      |          e.g. get, list, watch on pods and nodes
      v
   the API server allows exactly those calls and refuses everything else
```

Four objects, in two pairs, and the split is the useful part:

- **Role** and **ClusterRole** list *permissions* — verbs on resource kinds. A Role is
  scoped to one namespace; a ClusterRole applies cluster-wide.
- **RoleBinding** and **ClusterRoleBinding** *attach* those permissions to an identity.

Permissions and the granting of them are deliberately separate objects. That is what
lets you write one Role and grant it to several identities, and it is why a credential
can be prevented from ever widening itself: deny it access to roles and bindings, and
it cannot grant itself anything, no matter what else it can do.

Every pod gets a ServiceAccount whether you name one or not — the namespace's
`default`, which should have no meaningful permissions. Naming one explicitly, as the
cluster autoscaler does above, is how a workload gets exactly the access it needs and
nothing else.

> **Say it in one line:** a pod's identity is a ServiceAccount, permissions live in
> Roles, and bindings connect the two — kept separate so a credential can be stopped
> from widening itself.

## Secrets are not encrypted

The most important thing to know about Kubernetes Secrets, and the least advertised:

```
   kind: Secret
   data:
     password: c3VwZXJzZWNyZXQ=      <-- this is BASE64, not encryption.
                                         Decoding it is one command with
                                         no key and no permission needed.
```

A Secret is a ConfigMap with a warning label. Base64 is an encoding, not a cipher — it
exists so binary values survive YAML, not to hide anything. Committing a Secret
manifest to git is committing the password in a thin disguise.

What actually protects a Secret is everything *around* it:

- **RBAC** — the deploy identity in that project deliberately cannot read secrets at
  all, so a leak of it exposes none of them.
- **Never writing them down** — the database password there is generated by the
  operator directly into the cluster, so it exists in no file and no terminal history.
- **Encryption at rest**, if the cluster is configured for it, so the values aren't
  plaintext inside etcd.

> **Say it in one line:** a Secret is base64, not encrypted — what protects it is
> RBAC and never having written it down in the first place.

## Pulling a private image: `imagePullSecrets`

Memory card:
- Need: the kubelet has to authenticate to a registry to pull an image that isn't public.
- Object: a Secret of type `kubernetes.io/dockerconfigjson`, referenced by name.
- First failure pattern: `ImagePullBackOff` because the Secret is missing, wrong namespace, or the token expired.

A public image (this platform's game server, for example) needs no credential —
anyone can `docker pull` it. A **private** package on a registry like GHCR
(GitHub Container Registry) is different: the kubelet needs something to
authenticate with, the same way your laptop needs `docker login` before it can
pull one by hand. That "something" is a Secret, referenced by name on the Pod
spec:

```yaml
spec:
  imagePullSecrets:
    - name: ghcr-pull                # a Secret, type kubernetes.io/dockerconfigjson
  containers:
    - name: admin-server
      image: ghcr.io/example-owner/admin-server:sha-abc1234
```

```
   kubectl create secret docker-registry ghcr-pull \
     --docker-server=ghcr.io --docker-username=<anything-non-empty> \
     --docker-password=<a PAT scoped to read:packages ONLY>
        |
        v
   Secret ghcr-pull (namespace-scoped -- one per namespace that needs it)
        |
        v
   Pod spec references it by name -> kubelet presents it to ghcr.io on pull
```

The credential itself is usually a **personal access token (PAT)** — scoped as
narrowly as the registry allows (`read:packages` only, nothing that could push
an image or touch repo contents) — created once and stored as a CI secret, not
typed by hand into the cluster.

**Why this, and not something fancier?** The alternatives trade one problem
for another:

| Approach | Pro | Con |
| --- | --- | --- |
| PAT-backed `imagePullSecret` (above) | Works on any Kubernetes distribution; reuses an identity you already have (GitHub) | Long-lived — nothing rotates it automatically; leak it and it's valid until someone revokes it |
| Cloud-native identity (IRSA on EKS, Workload Identity on GKE) | Short-lived, auto-rotated tokens; no secret to store at all | Ties you to one cloud's registry integration — doesn't travel if you change providers |
| A shared team credential typed into every cluster by hand | Zero setup | No audit trail for who has it, no per-namespace scoping, and rotating it means finding every place it was pasted |

The narrower a pull secret's scope, the smaller the blast radius if it leaks —
`read:packages` only, one Secret per namespace rather than one shared
everywhere, are both worth the small extra setup.

This isn't only a Kubernetes-cluster problem — the same private-registry
credential shows up in a few other places, each with its own tradeoff:

- **CI pulling into another workflow.** If the workflow lives in the *same*
  org as the registry, most CI systems mint a short-lived, run-scoped token
  automatically — no secret to manage. Cross-org needs a PAT again.
- **A developer's laptop** (`docker login`). Convenient for debugging, but the
  credential ends up base64-stored in a local config file — not meaningfully
  more protected than a committed plaintext value, just less obvious.
- **A second cluster** (disaster recovery, a second region, a different
  cloud). A registry-agnostic PAT travels with you across clouds in a way a
  cloud-native identity doesn't — but you re-do the "create the Secret here
  too" step for every cluster unless something automates it for you.

> **Say it in one line:** `imagePullSecrets` is how a Pod authenticates to a
> private registry — scope the credential as narrowly as the registry allows,
> because nothing about a plain PAT expires on its own.

## The network starts wide open

The other default that surprises people. Kubernetes gives every pod an IP on one flat
network, and **by default every pod can reach every other pod** — any namespace, any
workload, any port.

```
   THE DEFAULT (no NetworkPolicy anywhere)

   [ game server ] <---> [ frontend nginx ] <---> [ Postgres ]
          ^                     ^                      ^
          +---------------------+----------------------+
                      everything can dial everything

   Nothing is stopping the static web pod from opening a connection
   straight to the database. It has no reason to. It simply may.
```

That is a deliberate design choice, not an oversight — it makes the platform simple
and means nothing mysteriously fails to connect. But it means a single compromised pod
starts with network reach to your whole cluster.

A **NetworkPolicy** narrows it. The usual shape is *default-deny* — one policy that
blocks everything — followed by narrow policies re-opening exactly the flows that
should exist:

```
   1. default-deny            nothing in or out, for every pod
   2. allow DNS               or nothing can resolve a name (see above)
   3. allow web -> server     the one HTTP path that must work
   4. allow server -> DB      the one database path that must work
```

Two traps, both of which the architecture document hit. **You must explicitly allow
DNS**, or step 1 silently breaks name resolution everywhere. And a NetworkPolicy is
**inert unless the cluster's network plugin enforces it** — write one on a cluster
whose plugin ignores policies and you have a YAML file that reads like protection and
provides none, which is worse than having nothing.

> **Say it in one line:** every pod can reach every pod until you say otherwise — so
> default-deny is the starting move, and an unenforced policy is a comforting lie.

## Extending the API: custom resources and operators

So far every `kind:` has been built in. But most of the interesting things in a real
cluster are **not**. Here are the custom kinds that system uses:

```
   kind: Cluster            -> a PostgreSQL cluster
   kind: ScheduledBackup    -> a recurring database backup
   kind: ClusterIssuer      -> a certificate authority to request from
   kind: PrometheusRule     -> alerting and recording rules
   kind: ServiceMonitor     -> "scrape metrics from these pods"
   kind: Rollout            -> a Deployment that can run a canary
   kind: AnalysisTemplate   -> the metric queries that judge that canary
   kind: Application        -> "keep this git path synced into the cluster"
```

None of those ship with Kubernetes. Each was added by installing something, and the
mechanism is worth understanding because it explains how the whole ecosystem works.

**A CustomResourceDefinition (CRD) teaches the API server a new noun.** Once it is
installed, `kind: Cluster` is as real as `kind: Pod` — you `kubectl apply` it,
`kubectl get` it, and RBAC governs it identically.

But a noun alone does nothing. It is inert data until something watches for it. That
something is an **operator**: a controller running the same "make reality match"
loop as the built-in ones, except its expertise is domain-specific.

```
   YOU WRITE                    THE OPERATOR DOES
   ---------                    -----------------
   kind: Cluster                creates the pods and their volumes
   instances: 1                 generates a password into a Secret
   storage: 50Gi                streams the write-ahead log to a bucket
                                handles minor-version upgrades safely
                                takes and verifies backups

                                (it can also run replication and
                                 failover -- but `instances: 1` above
                                 asks for neither, deliberately. See
                                 architecture.md.)

   CRD      = a new noun the API server understands
   OPERATOR = the controller that gives that noun meaning
```

This is the answer to a question the architecture document raises: why run a database
under an operator instead of a StatefulSet? Because a StatefulSet gets you a database
*process*, and an operator gets you the accumulated operational knowledge of people
who run that database for a living — encoded as a loop that never sleeps.

The pattern generalises. When some software is genuinely hard to operate correctly,
check whether an operator already exists before hand-rolling manifests and inheriting
every operational problem yourself.

> **Say it in one line:** a CRD adds a noun and an operator gives it meaning — which
> is how "I want a Postgres cluster" becomes one you can actually operate.

## Packaging: Helm, Kustomize, and overlays

Raw YAML has an obvious problem: production and development need *almost* the same
manifests. Copy-pasting them means every change happens twice, and eventually doesn't.

Two tools solve this differently, and the system this documents uses **both** —
deliberately, each for what it is good at.

**Helm** is a package manager. A chart is a parameterised bundle of manifests, and you
supply a `values.yaml` to fill in the blanks. It is how you install *other people's*
software:

```yaml
# values for the log collector's chart -- you configure, you don't author
controller:
  type: daemonset
alloy:
  resources:
    requests: { cpu: 20m, memory: 96Mi }
```

**Kustomize** is a patcher. You write real, valid YAML once as a **base**, then keep
small **overlays** that patch it per environment. Nothing is templated; every file is
a manifest you could apply directly:

```
   base/                      the real manifests, complete and valid
     deployment.yaml            "big2-server, 1 replica, image :latest"
     service.yaml
     ingress.yaml
        |
        +--- overlays/dev/     patch: use the :dev image tag
        |
        +--- overlays/prod/    patch: add the HPA, the PDB, the burst
                               toleration, and the real image digest
```

An **overlay** is just that patch directory. When the architecture document says CI
"applies the overlay," this is what it means: take the base, layer the production
differences on top, send the result to the API server.

The division that project uses is the conventional one, not a preference:

| Tool | Used for | Why |
| --- | --- | --- |
| **Helm** | Third-party add-ons — ingress controller, cert-manager, monitoring, log collector | They ship official charts. Re-authoring their manifests would mean maintaining someone else's software. |
| **Kustomize** | First-party manifests — the game server, the frontends | You own the YAML, and a patch is easier to review than a template. Every file stays directly applicable. |

> **Say it in one line:** Helm installs other people's software with values, Kustomize
> patches your own with overlays — and an overlay is just the diff between
> environments.

## The pod states you will actually see

```
   Pending             accepted, but not placed on a node yet.
                       Almost always: no node has room. WAITING, not failing.

   ContainerCreating   placed; pulling the image, mounting volumes.

   Running             the container started. NOTE: not necessarily *working* --
                       that is what the readiness probe is for.

   CrashLoopBackOff    it starts, dies, starts, dies. Kubernetes is now waiting
                       longer between attempts. This means YOUR program exited.

   ImagePullBackOff    the image name is wrong, or the registry rejected the
                       credentials. Nothing to do with your code.

   OOMKilled           the container exceeded its MEMORY limit and was killed.
                       (Exceeding a CPU limit only throttles.)

   Terminating         got SIGTERM, inside its grace period, finishing up.
```

The two that mislead beginners are `Pending` (nothing is broken — there is just no
room) and `Running` (the process started, which is not the same as serving correctly).

**Two verbs used throughout the architecture document:**

- **Schedule** — deciding which node a pod runs on, by comparing `requests` against
  free capacity. No room means `Pending`.
- **Drain** — politely emptying a node before maintenance: evict every pod, let them
  restart elsewhere. This is when `SIGTERM` handling and disruption budgets matter.

> **Say it in one line:** `Pending` means no room, `CrashLoopBackOff` means your code
> exited, `ImagePullBackOff` means your image name is wrong — and `Running` doesn't
> mean working.

---

## Glossary

**Core objects**

| Term | Meaning |
| --- | --- |
| **Cluster** | The whole managed system: a control plane plus the nodes it schedules work onto. |
| **Control plane** | The brain — API server, etcd, scheduler, controller manager. Managed by the cloud provider here. |
| **API server** | The only entrance. Every component, and every `kubectl` call, goes through it. |
| **etcd** | The database of record. Your entire cluster is rows in here. |
| **Scheduler** | Decides which node each pod runs on, then writes that decision back. |
| **Kubelet** | The agent on each node that actually starts and supervises containers. |
| **`kubectl`** | The CLI. A thin wrapper over HTTP calls to the API server. |
| **Node** | One actual computer. |
| **Pod** | The smallest unit Kubernetes runs — usually one container, with its own IP. Disposable by design. |
| **Container** | A compiled program plus its dependencies, sealed into one immutable image. |
| **Image / digest / tag** | The image is the bytes. A *tag* is a movable label; a *digest* is a hash of the bytes and cannot be repointed. |
| **Namespace** | A folder inside the cluster. Permissions can be scoped to stop at its boundary. |

**Workloads**

| Term | Meaning |
| --- | --- |
| **Controller** | The thing that creates and maintains pods for you. You write a controller, not a pod. |
| **Deployment** | Workload 1 of 4. N interchangeable copies that run forever, placed anywhere. The default choice. |
| **ReplicaSet** | The layer a Deployment creates underneath itself to hold one *version's* pods. You never write one; you see them in `kubectl get all`, and a rolling update is really one ReplicaSet growing while another shrinks. |
| **StatefulSet** | Workload 2 of 4. Each pod gets a stable name and its own volume that follows it. |
| **DaemonSet** | Workload 3 of 4. Exactly one copy per node, including nodes added later. You never specify a count. |
| **Job / CronJob** | Workload 4 of 4. Runs to completion and stops; a CronJob runs a Job on a schedule. |
| **Rolling update** | Replacing pods gradually — new pod Ready first, then retire an old one. Needs spare capacity to begin. |

**Networking**

| Term | Meaning |
| --- | --- |
| **Label / selector** | A key-value stamp on an object, and a query that matches it. How everything finds everything else — never by name or IP. |
| **Annotation** | Same shape as a label, never used for selection. Configuration addressed to other tools. |
| **Service** | A stable address that routes to pods found by label — *and* load-balances across the healthy ones. |
| **Cluster DNS** | The in-cluster resolver. Every Service gets a record automatically, so service discovery is just DNS: `<service>.<namespace>.svc.cluster.local`. |
| **`ClusterIP`** | The default Service type: reachable from inside the cluster only. |
| **`NodePort`** | A Service type that opens the same high-numbered port on *every* node and forwards it inward. It is how an external load balancer reaches the cluster — and the worker firewall rule for that port range is a classic silent failure when missing. |
| **`LoadBalancer`** | A Service type that asks the cloud to provision a real load balancer. Built on top of `NodePort`. One of the few objects that can quietly cost money. |
| **Ingress / ingress controller** | Routes one public entrance to many Services by hostname and path, and terminates TLS. |
| **NetworkPolicy** | A firewall between pods. The default is **allow-all**, so a policy only matters once you add a default-deny. Inert unless the cluster's network plugin enforces it. |

**Configuration and identity**

| Term | Meaning |
| --- | --- |
| **ConfigMap** | Non-sensitive values injected into pods at start-up. |
| **Secret** | The same, flagged as sensitive — but **base64-encoded, not encrypted**. |
| **`secretKeyRef`** | A reference that names a secret without containing its value. |
| **ServiceAccount** | The identity a pod runs as when it calls the API server. |
| **Role / ClusterRole** | A list of allowed verbs on allowed resource kinds; namespaced or cluster-wide. |
| **RoleBinding / ClusterRoleBinding** | Attaches a Role to an identity. Kept separate so permissions can be granted without being rewritten. |
| **RBAC** | The whole system above: named verbs, on named kinds, for named identities. |

**Storage**

| Term | Meaning |
| --- | --- |
| **PersistentVolume (PV)** | The actual piece of storage — a real cloud disk. |
| **PersistentVolumeClaim (PVC)** | A request for storage. Pods reference the claim, never the volume. |
| **StorageClass** | The recipe for creating a PV on demand. |

**Health, scheduling, and lifecycle**

| Term | Meaning |
| --- | --- |
| **Readiness probe** | "Should this pod get traffic?" Failing removes it from the Service and leaves it running. |
| **Liveness probe** | "Is this pod alive?" Failing **kills and restarts** it. Confusing the two is a self-inflicted outage. |
| **`requests` vs `limits`** | `requests` is a reservation the scheduler does arithmetic on; `limits` is a ceiling that throttles (CPU) or kills (memory). |
| **`Pending`** | Accepted but not placed on any node. Waiting, not failing. |
| **`CrashLoopBackOff`** | The container keeps starting and exiting. Your program died. |
| **`ImagePullBackOff`** | Wrong image name or registry credentials. Not your code. |
| **`OOMKilled`** | Exceeded the memory limit and was killed. |
| **`SIGTERM` / grace period** | The "please finish up" signal, and the deadline after which Kubernetes sends the uncatchable `SIGKILL`. |
| **Taint / toleration** | A taint repels pods from a node; a matching toleration lets a pod land there anyway. |
| **`nodeSelector`** | The opposite of a toleration — it *pins* a pod to specific nodes. |
| **PodDisruptionBudget** | A floor on availability during *voluntary* disruptions like a drain. Does not gate autoscaler scale-down. |
| **Drain** | Emptying a node of its pods before maintenance. |
| **HorizontalPodAutoscaler** | Adds or removes *pod* replicas based on a metric. Creates demand, never capacity. |
| **Cluster autoscaler** | Adds or removes *nodes*. The only thing that creates real capacity — and the only one that spends money. |

**Extending and packaging**

| Term | Meaning |
| --- | --- |
| **CRD** | A CustomResourceDefinition — teaches the API server a new `kind:`. |
| **Custom resource** | An object of that new kind. Inert until an operator watches it. |
| **Operator** | A controller encoding how to run one specific piece of software properly. |
| **Helm / chart / values** | A package manager for other people's manifests, parameterised by a values file. |
| **Kustomize / base / overlay** | A patcher for your own manifests: one real base, small per-environment patches on top. |
| **Pod Security Standards** | Cluster-enforced tiers (`privileged`, `baseline`, `restricted`) checked when a pod is admitted. |
| **Admission** | The moment a pod is *created*, before it runs — where those tiers are checked and a non-compliant pod is refused. |

---

## Glossary, part two: the terms that aren't Kubernetes

Everything above is Kubernetes vocabulary. The architecture document also leans on
terms from reliability, cryptography, and the browser — the ones below. They are here
rather than there so there is exactly one place to look something up.

**Reliability and operations**

| Term | Meaning |
| --- | --- |
| **SLO** (service level objective) | A target like "99.5% of requests succeed over 28 days." |
| **SLI** (service level indicator) | The measurement the SLO is judged against — the actual ratio of good requests to all requests. |
| **Error budget** | The failure the SLO permits. At 99.5% over 28 days that is ~202 minutes. A real, spendable number. |
| **Burn rate** | How fast the budget is draining, relative to the pace that would exactly exhaust it over the period. 1× finishes on target; 14.4× empties it in ~2 days. |
| **Multi-window, multi-burn-rate** | The alerting shape: a fast burn and a slow burn, each needing a long *and* a short window to agree before firing. The long window says "this is real"; the short says "it is still happening." |
| **Recording rule** | A query computed once and stored, so alerts compare cheap stored values instead of recomputing. |
| **Inhibition rule** | Stops a page from also delivering its own slower ticket. One incident, one stream. |
| **Runbook** | The written answer to "it is 3 a.m. and this alert fired — now what?" |
| **RED signals** | **R**ate, **E**rrors, **D**uration — the three numbers that describe any request-serving system. |
| **Blast radius** | If this credential or component is misused, how much can it reach? |
| **Least privilege** | Every identity gets the smallest set of powers that still lets it do its job. |

**Data durability**

| Term | Meaning |
| --- | --- |
| **WAL** (write-ahead log) | Postgres writes every change to an append-only log *before* touching the data files. Shipping that log off-machine is the backup. |
| **PITR** (point-in-time recovery) | Restoring a base backup and replaying the log forward to any chosen second. |
| **RPO** (recovery point objective) | How much data you accept losing. |
| **RTO** (recovery time objective) | How long you accept being down while recovering. |
| **Durability vs availability** | Durability is "the data survives"; availability is "you can reach it now." Different properties — a system can buy one without the other. |

**Security and cryptography**

| Term | Meaning |
| --- | --- |
| **MAC** (message authentication code) | A short tag computed from data plus a secret key. Proves the data wasn't tampered with; does **not** hide it. |
| **Unforgeable vs non-transferable** | A MAC stops someone *minting* a token they were never given. It does not stop them *handing theirs to someone else*. |
| **Domain separation** | Prefixing a distinct label before signing, so a tag minted for one purpose can never be replayed as another. |
| **Constant-time comparison** | Comparing secrets in a way whose duration doesn't leak how much of the guess was right. |
| **`crypto/rand` vs the fast generator** | Unpredictable versus merely random-looking. Anything guessable that grants access needs the first. |
| **CVE** | A publicly catalogued vulnerability, with an identifier. |
| **SBOM** | Software bill of materials — the full ingredient list of an image, so "are we affected by this CVE?" becomes a query. |
| **Keyless signing / OIDC identity** | The build workflow's own short-lived identity signs, so there is no long-lived key to store, rotate, or leak. |
| **Transparency log** | A public, append-only record of what was signed and when. |
| **Attestation** | A signed statement *about* an artifact (such as its SBOM), bound to the same digest. |
| **Distroless** | An image containing the compiled binary and nothing else — no shell, no package manager, nothing for an intruder to pick up. |

**The browser and the edge**

| Term | Meaning |
| --- | --- |
| **WebSocket** | A connection that stays open in both directions, so the server can push without being asked. |
| **`ws` vs `wss`** | The same protocol, one wrapped in TLS — exactly `http` : `https`. An `https` page is *forbidden* to open a plain `ws://` socket. |
| **TLS termination** | The point where encryption ends and plaintext begins. Here, the ingress — which is why the server can't tell whether the *user* was on HTTPS. |
| **CORS** | The browser rule that restricts a page on one origin from calling another. Irrelevant when there is only one origin. |
| **First-party cookie** | A cookie set by the same origin as the page, so browser privacy defaults keep it. |
| **`HttpOnly` / `Secure` / `SameSite`** | Cookie flags that block script access, unencrypted transmission, and cross-site sending, respectively. |
| **NAT gateway** | A one-way valve: outbound connections allowed, inbound origination blocked. |
| **Security group** | Firewall rules attached to a resource rather than to a whole subnet. |

**Certificates**

| Term | Meaning |
| --- | --- |
| **ACME** | The protocol for automatically requesting and renewing certificates. |
| **HTTP-01 / DNS-01** | Two ways to prove you control a domain: serve a file at a known path, or write a DNS record. |
| **Staging vs production issuer** | The same authority, one with loose limits and an untrusted root — useless to serve, perfect for rehearsing. |

**Delivery**

| Term | Meaning |
| --- | --- |
| **GitOps** | Git is the source of truth; a controller in the cluster continuously makes reality match it. |
| **Canary** | Shipping to a small slice of traffic first and watching its metrics before committing to the rest. |
| **Drift detection** | Noticing that the cluster no longer matches what git says it should be. |
| **Digest vs tag** | A tag is a movable label; a digest is a hash of the bytes. Signing a tag would certify a name, not a payload. |

**Cloud account structure**

| Term | Meaning |
| --- | --- |
| **Tenancy** | The whole cloud account. |
| **Compartment** | The provider's own folder one level above the cluster, so no resource is born in the account root. |

---

*Back to [architecture.md](architecture.md) for how one real system puts all of this
together.*
