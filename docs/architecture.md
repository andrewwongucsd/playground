# Architecture

Every section here ends with one line you could say out loud in an interview. That
line is the point; the paragraphs above it are the evidence.

**How to read this.** The doc runs on two tracks. Each section opens with a short
**In plain English** box that assumes you have never touched Kubernetes, and closes
with a **Say it in one line** takeaway. The paragraphs between them are the real
engineering, written for someone who has. Read the boxes first if you want the shape
of the system in ten minutes; read the middles for how it was actually decided.

**Never used Kubernetes?** Read the companion
**[Kubernetes primer](kubernetes-primer.md)** first. It teaches the vocabulary from
zero — the control plane, the four workload types, storage, RBAC, operators — using
this system's real configuration as its examples, and it carries the glossary for
every term used here.

---

## Contents

**Orientation**
1. [Start here: what this thing actually is](#start-here-what-this-thing-actually-is)
2. [You need eight Kubernetes words to read this](#you-need-eight-kubernetes-words-to-read-this)
3. [The second product, and why it shares the cluster](#the-second-product-and-why-it-shares-the-cluster)
4. [Running it with nothing configured](#running-it-with-nothing-configured)

**Getting a request in**
5. [The layers a request crosses](#the-layers-a-request-crosses)
6. [Network topology](#network-topology)
7. [Identity: three rings and three keys](#identity-three-rings-and-three-keys)
8. [Same-origin hosting](#same-origin-hosting)

**Surviving the open internet**
9. [Hardening the public edge](#hardening-the-public-edge)

**Who the player is**
10. [How a player proves who they are](#how-a-player-proves-who-they-are)
11. [Playing with the people you came with](#playing-with-the-people-you-came-with)

**The game itself**
12. [The protocol, and why a modified client can't cheat](#the-protocol-and-why-a-modified-client-cant-cheat)
13. [The browser client](#the-browser-client)

**Keeping it up**
14. [TLS issuance, and why there are two issuers](#tls-issuance-and-why-there-are-two-issuers)
15. [Data durability: how far back you can rewind](#data-durability-how-far-back-you-can-rewind)
16. [Scaling, safely and within budget](#scaling-safely-and-within-budget)

**Knowing what it's doing**
17. [Seeing inside it: metrics, logs, and traces](#seeing-inside-it-metrics-logs-and-traces)
18. [SLOs, and alerts that page for a reason](#slos-and-alerts-that-page-for-a-reason)
19. [Testing across the boundaries that actually break](#testing-across-the-boundaries-that-actually-break)

**Getting changes out**
20. [Delivery: pipeline-pushed today, GitOps authored next](#delivery-pipeline-pushed-today-gitops-authored-next)
21. [Hardening: the supply chain, the pod, the network](#hardening-the-supply-chain-the-pod-the-network)
22. [Operating a cluster you can't reach directly](#operating-a-cluster-you-cant-reach-directly)

**Reference**
- [Kubernetes primer](kubernetes-primer.md) — the vocabulary, from zero, plus the glossary
- [Every one-liner in one place](#every-one-liner-in-one-place)

---

## Start here: what this thing actually is

> **In plain English:** a few small websites and one small database, running on one
> rented computer — wrapped in all the machinery a real company would put around
> them. The point was never the size of the app. It was operating it properly.

**Two products** are what this document is about:

- **A multiplayer card game.** A Go server holds the rules and the live tables. A
  React page draws the table in your browser. The two talk over a **WebSocket** — a
  connection that stays open, so the server can push a card to you the instant
  someone plays it, instead of your browser asking "anything new?" over and over.
- **A browser-only utilities toolbox.** Ten small tools that run entirely in your
  browser. There is no backend at all; it is static files served by nginx.

**A third workload also lives on the cluster**, and it is named here so the capacity
arithmetic later is honest: a small distributed-scheduler demo, built as a standalone
learning artifact in its own repository and deployed onto this platform because the
platform already existed. Its internals are out of scope for this document; its
*footprint on the baseline node* is not.

All three sit behind one public entrance:

```
                            the internet
                                 |
                                 v
              +--------------------------------------+
              |  ONE public load balancer            |  the only way in
              +--------------------------------------+
                                 |
                                 v
              +--------------------------------------+
              |  INGRESS -- reads the hostname and   |  TLS ends here
              |  the path, then picks a destination  |
              +--------------------------------------+
             |                  |                    |
   apex / www|      the game's  |          the demo's|
             v         hostname v            hostname v
   +-----------------+ +-----------------------+ +------------------+
   | utilities       | | /ws /auth /telegram   | | scheduler demo   |
   | toolbox         | | /queue -> Go server   | | (own repo, out   |
   | React, no       | | everything else       | |  of scope here)  |
   | backend         | |   -> the React page   | +------------------+
   +-----------------+ +-----------------------+
                                  |
                                  v
                            [ PostgreSQL ]
                            the only thing that talks
                            to the database is the
                            game's Go server
```

Every hostname above resolves to that one balancer's IP, and each carries its own
certificate issued by the same certificate manager — which is what makes "one public
entry point" a literal statement rather than a simplification.

**One machine, on purpose.** The baseline is a single ARM node on a free shape, and it
is carrying all three of those workloads plus the database, the ingress controller, and
the observability stack. Every design decision below is shaped by that: there is rarely
room for a second copy of anything, which is why several capabilities in this document
are *authored but switched off* rather than claimed as live. The third workload does not
change that conclusion — it is one more reason for it.

**Free shape, paid account — and the difference matters.** The node and the load
balancer both use shapes the cloud provider gives away permanently. But the *account*
is Pay-As-You-Go: its quota is far larger than a pure free tier's, and it has no
spending ceiling of its own. So "free" here is a shape you keep choosing, not a wall
you can lean on. That distinction is not pedantry — it is the exact reason the
[node pool later gets a hard cap](#scaling-safely-and-within-budget), and it is a
correction this project had to make mid-build.

> **Say it in one line:** it is a small system operated like a large one — and the
> single free node is the constraint that made every trade-off honest.

## You need eight Kubernetes words to read this

The [primer](kubernetes-primer.md) teaches all of this properly. If you'd rather press
on from here, these eight words carry most of the pages below:

| Word | The one-line version |
| --- | --- |
| **Cluster** | The whole managed system: a control plane, plus the computers it runs your code on. |
| **Node** | One actual computer. This project has one that always runs, plus one that appears only under load. |
| **Pod** | The smallest thing that runs — your container, with its own IP. **Disposable by design.** |
| **Deployment** | A standing wish: *"keep N copies of this running; replace any that die."* |
| **Service** | A stable address that finds pods by **label**, not by IP — so a replaced pod rejoins with zero configuration. |
| **Ingress** | The one public entrance. Routes by hostname and path, and terminates TLS. |
| **Readiness probe** | A health question. A pod receives traffic **only** while it passes. |
| **`Pending`** | A pod accepted but not placed on any node. *Waiting*, not failing — nothing is broken, there is just no room. |

The idea underneath all of them: you never tell Kubernetes to *do* something. You
describe the state you want, and a control loop spends forever making reality match
the description. Pods are cattle, everything finds everything else by label, and a
`Pending` pod is the most misread status in the system.

> **Say it in one line:** you declare what should be true and a control loop keeps it
> true — so pods are disposable and every address is found by label.

## The second product, and why it shares the cluster

> **In plain English:** there is a second, much simpler site here — a box of small
> developer tools that run entirely in your browser. It has no server, no database,
> and no login. It exists in this document because *where it runs* was a real
> decision.

The utilities toolbox is ten tools — a JSON/YAML editor, Base64 and JWT decoding, a
regex tester, hashes and UUIDs, timestamps and cron, QR codes, a Markdown editor, a
thumbnail grabber, an image converter, a meme/GIF generator. Every one runs
client-side. Nothing is uploaded, which is the product's whole promise, and it means
the entire "backend" is nginx serving static files.

The meme generator is the sharpest test of that promise, because it is the one tool
doing real work: sample frames from an uploaded video by seeking a `<video>` element
onto a `<canvas>`, burn in captions and meme text, encode the result as a GIF —
entirely with in-browser APIs, no `WebCodecs` dependency, so it runs the same in
Safari as anywhere else. **No upload, no backend, same as the other nine** — the
newest tool is the one that had the most reason to break that rule and didn't.

```
   THE GAME                         THE TOOLBOX
   --------                         -----------
   Go server + WebSocket            nothing but static files
   Postgres                         no database
   login, sessions, matchmaking     no accounts at all
        |                                |
        +--------------+-----------------+
                       |
              the SAME cluster, the same ingress,
              the same TLS setup, the same pipeline
```

**The decision was to not scatter it.** A static site is exactly the sort of thing
that gets parked on a separate static-hosting vendor, and that would have been less
work in the short run. It would also have meant two deploy stories, two TLS setups,
two places to check when something breaks, and two vendors' outages to care about —
for one small system.

Its deployment is deliberately boring: one replica, an unprivileged nginx image, and
one detail that catches people out — the container listens on **8080, not 80**,
because a non-root process cannot bind a port below 1024. That single line is what
lets the whole pod run as a non-root user.

### The same argument, applied a third time

The scheduler demo landed here for the same reason, and it is worth being precise about
what that does and doesn't say. It is a **standalone learning artifact** — its own
repository, its own CI, its own README — and it is deliberately *not* part of this
platform's story. It runs here because a working cluster with an ingress, a certificate
manager, and a deploy pipeline already existed, so the marginal cost of one more
hostname was close to zero.

```
   what it cost to add a third workload

   a new vendor            ->  a new TLS setup, a new deploy story, a new
                               place to look when something breaks

   one more host on THIS   ->  a DNS record, an ingress rule, a certificate
   cluster                    the existing manager issues automatically
                               ... and a slice of the one baseline node
```

That last line is the honest half. The marginal *operational* cost was near zero; the
marginal *capacity* cost was not, because the baseline node is the scarce resource this
whole document keeps running into. A platform that makes adding things cheap will get
things added — which is a good property right up until the node is full.

> **Say it in one line:** one platform, one TLS story, one pipeline — a second vendor
> is a second thing to operate, and it buys nothing for a static site.

## Running it with nothing configured

> **In plain English:** clone the repository, run one command, and the game works —
> no database, no accounts, no cloud, no Kubernetes. That isn't an accident or a
> half-finished feature. It's a design rule, and it costs something to maintain.

Every external dependency in the server is **optional**, and its absence degrades one
feature rather than breaking start-up:

```
   DATABASE_URL unset        -> in-memory wallet, no match history,
                                /readyz always green
   TELEGRAM_BOT_TOKEN unset  -> the chat login endpoints 404;
                                guest play is unchanged
   webhook secret unset      -> the webhook route isn't registered at all
   OTEL endpoint unset       -> tracing is a no-op
   no table-key verifier     -> everyone lands in the open queue

   `go run .` with zero setup: a playable game.
```

Three environments, from the same manifests:

| Environment | Namespace | What differs |
| --- | --- | --- |
| **Local** | none — no Kubernetes at all | Everything optional is off. One process. |
| **Dev** | `big2-dev` | The `dev` image tag; no autoscaler, no disruption budget — the local cluster is single-node and has no metrics server, so an autoscaler there would only ever report `<unknown>`. |
| **Prod** | `big2` | The real digest, plus the HPA, the PDB, and the burst toleration. |

**Why this is worth the discipline.** A project where "run it locally" means
provisioning a database is a project where nobody runs it locally — and the first
thing a new reader does is give up. Making every dependency optional is what keeps the
distance between reading the code and running it at one command.

**And the cost is real**, which is the part usually left out. Every optional
dependency is a branch: a nil check, a fallback path, and a second behaviour to reason
about. The `optional: true` on each secret reference, the nil `SessionResolver`, the
absent table-key verifier — each is a small tax paid on every read of that code. The
trade is only worth it when the degraded mode is *genuinely useful*, as a playable
game is. Optional dependencies that degrade into something nobody wants are pure cost.

> **Say it in one line:** a missing dependency should cost you one feature, not
> the whole start-up — that is what keeps "clone and run" a single command.

## The layers a request crosses

> **In plain English:** getting a program running inside the cluster is the easy
> half. Getting a stranger's browser to reach it needs three more layers — a front
> door, a receptionist, and an ID card — and they only work if you build them from
> the bottom up.

A **Deployment** (keep this program running) and a **Service** (give it a stable
address) get a container running and reachable *inside* the cluster. Nobody outside
can touch it yet. Turning that into something a person can type a URL for takes three
more layers, and they have to come up in order. Each one is reached *through* the one
below it.

1. **A public load balancer** accepts traffic from the internet. On managed
   Kubernetes you don't create it directly. You install an ingress controller whose
   Service is `type: LoadBalancer`, and the cloud provider's controller provisions a
   real balancer to match. Its shape is pinned to a free allowance, because a
   `LoadBalancer` Service is one of the few Kubernetes objects that can quietly bill
   real money. The default shape is not free. And flexible shapes bill on their
   *maximum*, so both the minimum and the maximum are pinned.
2. **An ingress controller** routes that one public entry point to many internal
   services by hostname and path. It also terminates TLS — the padlock ends here.
3. **Certificates and DNS.** A certificate manager watches ingress objects and runs
   the ACME challenge automatically: request a certificate, serve the challenge
   *through the ingress*, swap in the real certificate once the authority validates.
   DNS points the hostnames at the balancer.

Install them in the wrong order and you get failures that look like unrelated bugs.
The certificate is served through the ingress, and the ingress is reached through the
balancer. So you build bottom-up — and once built, a request runs the same path
downward:

```
   browser
     |   DNS resolves the hostname to the balancer's public IP
     v
  [ LOAD BALANCER ]   the one public entry point, pinned to a free shape
     |
     v
  [ INGRESS ]   TLS ENDS HERE. Everything past this point is plain
     |          HTTP travelling inside the cluster.
     |
     +-- game host: /ws /auth /telegram /queue
     |                       ---------> [ SERVICE ] --> [ POD ] Go game server
     |
     +-- game host, everything else --> [ SERVICE ] --> [ POD ] game page + nginx
     |
     +-- apex / www ------------------> [ SERVICE ] --> [ POD ] toolbox + nginx


   Only the Go server ever reaches the database:

     [ POD ] Go game server -----> [ SERVICE ] -----> [ POSTGRES ]
                                                      operator-run, password
                                                      minted into the cluster,
                                                      never in git

   A pod joins its Service's backend list ONLY while its readiness probe passes.
```

Read it downward and it's a request. Read it upward and it's the install order.
Nothing in the middle is wired by IP address — a Service finds its pods by label
match, so a replaced pod with a brand-new IP rejoins with zero configuration.

**"TLS ends here" is not trivia — it caused a real bug.** The login cookie set its
`Secure` flag based on whether *the server* saw an encrypted connection. But TLS
terminates at the ingress, so the Go server always sees plain HTTP over the cluster
network. Every real user's session cookie would have shipped without the flag that
stops it travelling in the clear. The fix reads the forwarded-protocol header the
proxy sets, which is safe *here specifically* because the failure runs in the
harmless direction: a spoofed value can only *add* the flag, never remove it. The
full write-up is in
[engineering-highlights.md](engineering-highlights.md#bugs-a-real-deployment-surfaced).

**The proxy has opinions about time, too.** The ingress closes a proxied connection
that has been idle for 60 seconds. That is a sane default for ordinary HTTP requests
and exactly wrong for a WebSocket, where a player waiting at an unfilled table
legitimately sends nothing for minutes. Players were dropping with a bare
"Connection closed." and **no server-side error at all**, because nothing on the
server had failed. The fix belonged in the edge configuration, not the app.

### The same ordering problem, one level down

Install order isn't only a platform concern. The application's own manifests have a
chicken-and-egg problem, which is why a **bootstrap** directory exists separately from
everything CI applies on each push:

```
   bootstrap/  (applied ONCE, by a human-triggered privileged workflow)
     namespace.yaml        the namespace -- AND its Pod Security labels
     rbac-role.yaml        what the deploy identity may do
     rbac-rolebinding.yaml who that role is granted to
        |
        |  only after all three exist can this work:
        v
   base/ + overlays/prod/  (applied on EVERY push, by the deploy identity)
     deployment, service, ingress, hpa, pdb ...
```

Three orderings are forced, and each fails differently if you get it wrong:

- **The namespace must exist before anything in it.** Obvious, and the only one that
  fails loudly.
- **Its Pod Security labels must be there before the first pod arrives** — those
  labels are checked at admission, so a namespace created without them would happily
  accept a privileged pod, and adding the labels later doesn't retroactively evict
  anything.
- **The deploy identity's Role and RoleBinding must exist before it can deploy
  anything at all** — and the deploy identity deliberately cannot create them, because
  [it cannot touch roles or bindings](#identity-three-rings-and-three-keys). That's not
  an inconvenience to route around; it's the property that stops the routine
  credential from widening itself, and it *requires* a separate privileged step.

So "bootstrap" here isn't a synonym for setup. It's the set of things the everyday
credential is deliberately not allowed to do, which is exactly why they run once,
by hand, under a different identity.

> **Say it in one line:** the certificate needs the ingress, and the ingress needs the
> balancer — so you install upward, and requests travel back down the same path.

## Network topology

> **In plain English:** the machines that run your code should have no doorway from
> the internet at all. They can call out; nothing can call in. Only the front door is
> exposed, and it stands in its own room.

The cluster lives in its own virtual network, split into three subnets rather than
one. That's the standard production layout: each subnet's routing and security rules
can differ without one loosening another.

```
  Virtual Cloud Network
  |
  +-- public endpoint subnet (/28)   the Kubernetes API server
  |                                   (where kubectl and CI connect)
  |
  +-- private worker subnet          the nodes -- never directly internet-facing;
  |                                   outbound only, through a NAT gateway
  |
  +-- public load-balancer subnet    the one public entry point
```

Workers are private. They reach the internet outbound through a NAT gateway for image
pulls and updates, but nothing on the internet can reach them directly. A NAT gateway
is a one-way valve: calls out are allowed, and the return traffic for those calls is
allowed, but a stranger cannot originate a connection inward. The only public ingress
is the balancer, in its own subnet.

Security is enforced with resource-attached security groups rather than subnet-wide
lists, so a rule follows the resource it protects. Two of those rules are the classic
silent failure. The balancer forwards to the ingress controller on a node-port range
and health-checks it on another port. If the worker-side rules for those are missing,
the balancer's backends stay **permanently unhealthy while every Kubernetes object
looks completely correct**:

```
   [ LOAD BALANCER ]                          [ WORKER NODE ]
         |                                          |
         |--- forward traffic -->  X  --------------|  no inbound rule for
         |                         ^                |  the node-port range
         |--- health check ----->  X  --------------|  no inbound rule for
                                   ^                   the health-check port
                                   |
                    the packets die HERE, at the cloud network layer

   What the cluster reports:   every pod Ready, every Service bound, ingress
                               object created, certificate issued.  All green.
   What the user gets:         nothing. The balancer has no healthy backend.

   Nothing inside Kubernetes can see this. The evidence lives one layer below it.
```

> **Say it in one line:** if a node doesn't need a path *from* the internet, it
> doesn't get one — and the rules that break this are invisible from inside the
> cluster.

## Identity: three rings and three keys

> **In plain English:** don't hand out one master key. Give the key you use a hundred
> times a day the power to open exactly one small room, and keep the key that opens
> everything in a drawer you almost never touch. If a key gets copied, the damage is
> whatever that key could reach — so decide that in advance.

Blast radius asks one question: if this is misused, how much can it touch? The answer
is kept small in two dimensions at once — *where* things live, and *who* can act.

```
   Nested isolation -- each ring contains the next:

   cloud account (tenancy)
    +-- one compartment            <- the first resource ever created, on purpose
         +-- the cluster
              +-- one namespace    <- RBAC boxes the deploy identity in HERE

   Three identities, three scopes:

     routine deploy   ->  a handful of verbs, ONE namespace     (runs on EVERY push)
     platform admin   ->  cluster-admin, whole cluster          (manual only, rare)
     backup writer    ->  read + write objects in ONE bucket    (lives IN the cluster)

   A leak of the deploy identity stays trapped in the innermost box.
```

A **compartment** is the cloud provider's own folder, one level above the cluster.
Creating it first — before anything else exists — means no resource is ever born in
the account's root, where policies are hardest to write narrowly. **RBAC** is
Kubernetes' permission system: it grants named verbs (`get`, `create`, `patch`) on
named resource kinds, inside a named namespace.

The ordering is the argument. The credential exercised most often — the one that runs
on every single push — is the one most likely to leak, so it holds the least power. It
may `get`, `list`, `watch`, `create`, `update`, and `patch` a few resource kinds in one
namespace. It cannot delete anything. It cannot read secrets. It deliberately cannot
touch roles or role bindings, so **it can never widen itself** — the property that
turns a small breach into a contained one rather than a first step.

The powerful identity is the opposite: `cluster-admin`, wired only into a workflow a
human triggers by hand, never onto a commit. And the third — the backup writer — is
the one that must live permanently *inside* the cluster as a long-lived secret, so its
policy grants exactly two verbs against exactly one bucket. It cannot create buckets,
read the database, or see anything else.

Every one of those keys is generated *inside* Terraform. None was ever clicked in a
console and pasted by hand.

### How a secret actually reaches a running pod

"Generated in Terraform" is only the first hop. Here is the whole path, because the
interesting risk is in the middle of it, not at either end:

```
   [ TERRAFORM ]  generates the key pair (tls provider)
        |         the PRIVATE key is now in terraform state
        v
   [ sync script ] reads `terraform output`, pipes straight to `gh`
        |          never writes a key to disk, never echoes one to the terminal
        v
   [ GITHUB ACTIONS SECRETS ]  encrypted at rest, masked in logs
        |
        v
   [ PRIVILEGED WORKFLOW ]  `kubectl create secret generic ...`
        |                   human-triggered, not on a push
        v
   [ KUBERNETES SECRET ]
        |
        v
   [ POD ]  reads it via secretKeyRef as an environment variable

   The database password skips ALL of this: the operator mints it directly
   into the cluster, so it exists at exactly one hop and nowhere else.
```

Two properties worth naming. The GitHub secrets are a **derived artifact**, not a
source of truth — they can be regenerated from Terraform at any time, which is what
makes rotation a re-run rather than an archaeology project. And they have to be
re-synced whenever the repository moves or is forked, because secrets don't follow a
repo.

### The honest gap: where that state file lives

The path above has an unglamorous weak point, and it is the most concrete thing on
this page still to fix.

**Terraform state still lives on one machine.** And because two of the three
identities have their key pairs generated *by* Terraform, that state file contains
**private keys in plaintext**.

```
   WHAT THE STATE FILE HOLDS          WHAT PROTECTS IT TODAY
   -------------------------          ----------------------
   the CI deploy identity's           file permissions on one
   private key                        laptop, and nothing else

   the platform-admin identity's      no encryption at rest
   private key                        no locking (a concurrent apply
                                        can corrupt it)
   every resource id in the           no versioning (delete it and the
   account                              cloud resources are orphaned --
                                        still running, still billing,
                                        no longer managed)
```

Losing that file does not take the site down. It does something arguably worse: the
infrastructure keeps running while becoming unmanageable, and the only recovery is
importing every resource back by hand.

The fix is well-trodden, and it is now **authored but not switched on** — the same
status as the canary and the network policies:

```
   DONE                              NOT DONE
   ----                              --------
   remote backend configured,        the state has not been MIGRATED --
   as a partial config so the        it still lives locally until an
   tenancy namespace stays           operator runs the init that copies
   out of git                        it across
        |                                 |
   S3-native state LOCKING            the two key pairs have NOT been
   enabled, so concurrent             rotated -- and they must be, because
   applies can't corrupt it           the old ones spent time in an
        |                             unencrypted local file
   versioning on the bucket,               |
   which is the whole point           until then, the risk above is
                                      unchanged
```

The migration is deliberately a human step, because it is the one operation where
getting it wrong loses the mapping between the configuration and the running
infrastructure. The sequence — migrate, confirm `plan` reports **no changes**, only
then delete the local file, and only then rotate the keys and re-sync them — is
written into the configuration next to the backend block rather than trusted to
memory.

It is listed here for the same reason the untested restore is: a gap you have written
down is a plan, and a gap you haven't is a surprise. Writing the configuration is not
the same as having done it, and this document does not get to claim the second because
it did the first.

> **Say it in one line:** the credential you use most should reach the least, and the
> one that lives in the cluster forever should reach least of all.

## Same-origin hosting

> **In plain English:** the game's page and the game's API answer on the *same*
> web address. Browsers treat "same address" as "same thing," which quietly deletes
> an entire category of configuration you would otherwise have to get right.

The game's static frontend and its API are served from the **same** hostname. The
ingress routes the four API prefixes — `/ws`, `/auth`, `/telegram`, `/queue` — to the
Go server, and everything else to the frontend's nginx. Three payoffs, all from one
decision:

- the frontend derives its WebSocket URL from `window.location`, so one built image
  runs unchanged on any hostname — no rebuild per environment;
- the login-session cookie is **first-party**, so no browser privacy setting treats it
  as a tracker and drops it;
- **CORS** — the browser rule that blocks a page on one origin from calling another —
  never engages in production at all, because there is no second origin to call.

```
   TWO ORIGINS (not this)                ONE ORIGIN (this)
   ----------------------                -----------------
   game.example.com  -> page             game.example.com -> page AND api
   api.example.com   -> api                          |
        |                                            v
        v                                   the cookie is first-party,
   preflight requests, CORS headers,        the socket URL is derived,
   a third-party cookie, and a              and CORS is a localhost-only
   different URL per environment            concern
```

> **Say it in one line:** one origin means the cookie is first-party and CORS is a
> development concern, not a production one.

## Hardening the public edge

> **In plain English:** the moment a socket is open to the internet, strangers can
> hold it open, flood it, or walk away without hanging up. None of that is a bug in
> your code — it is your code being used exactly as designed, by someone who wishes
> you harm. Every limit here exists because the default is "unlimited."

A WebSocket is a resource an attacker gets to spend. Unlike an HTTP request, which
ends, a socket **persists** — it holds a goroutine, a read buffer, a write buffer, and
a slot in every room broadcast, for as long as it stays open. The default posture of
every WebSocket library is generous, and generous is the wrong default facing the open
internet.

Five limits, each closing a different way to hurt the server:

```
   a stranger connects
        |
        v
   1. PER-IP CONCURRENT CAP (8)
      one address may hold 8 sockets at once. Beyond that: refused.
      Stops one machine from exhausting goroutines and memory.
        |
        v
   2. TOKEN-BUCKET RATE LIMIT (0.5/sec, burst 10)
      how FAST new sockets may be opened, separately from how many.
      A fresh IP starts with a full bucket, so real page reloads
      never notice; a reconnect storm drains it and gets throttled.
        |
        v
   3. FRAME SIZE CAP (4 KB, set BEFORE the first read)
      the library imposes none by default -- so a single frame could
      otherwise be gigabytes, allocated before you ever inspect it.
        |
        v
   4. JOIN DEADLINE (10s)
      connect and then say nothing, and you are dropped. Otherwise an
      idle half-open socket costs the same as a playing one.
        |
        v
   5. READ / WRITE DEADLINES + PING/PONG KEEPALIVE
      60s to hear anything at all; a ping every 54s; 10s to complete
      a single write.
```

**Two of those five are subtler than they look.**

The frame cap is applied *before the first read*, not after. Checking a size after
reading is checking it after you have already allocated the memory — the attack
succeeded and you noticed afterwards. The limit has to sit upstream of the allocation
to mean anything.

And the write deadline exists for a reason that isn't obvious: **one slow client can
wedge everyone else.** Room state is broadcast to every seat, so a player whose
network has stalled — but whose socket is still technically open — would block that
broadcast indefinitely without a bounded write.

```
   WITHOUT a write deadline            WITH one
   ------------------------            --------
   broadcast to seat 1  ok             broadcast to seat 1  ok
   broadcast to seat 2  ok             broadcast to seat 2  ok
   broadcast to seat 3  ...hangs       broadcast to seat 3  fails after 10s
   broadcast to seat 4  never runs     broadcast to seat 4  ok
        |                                   |
        v                                   v
   the whole TABLE is frozen by        the stalled client is dropped;
   one stalled player                  the other three keep playing
```

### The client that vanishes without saying goodbye

The keepalive solves a problem TCP alone does not. A backgrounded tab, a slept laptop,
or a dropped network leaves a socket that is *open* as far as the server is concerned,
forever, because nothing ever arrives to tell it otherwise.

```
   server ---- ping ----> client        every 54 seconds
          <--- pong -----                the library answers automatically

   heard a pong?  -> extend the deadline another 60s
   heard nothing? -> the read deadline expires, the socket is reaped

   The ping interval is deliberately 90% of the timeout. If it were the
   other way round, the deadline would expire before the ping that was
   supposed to refresh it -- disconnecting every healthy player on a timer.
```

That 90% relationship is the kind of thing you get wrong once. Set the ping slower
than the timeout and you have not built a keepalive — you have built a scheduled
disconnect for everyone.

### A third-party call inside the join path

One feature deliberately reaches outside the system while a player is waiting:
a nickname-less player can be named after their approximate location, which means
calling a geolocation service *during the join*.

That is exactly the shape of dependency that turns one vendor's bad afternoon into
your outage. Three properties keep it bounded:

```
   join arrives with no nickname
        |
        v
   private or loopback IP?  -- yes --> skip the call entirely
        |  no                          (don't spend the timeout to be told "no")
        v
   cached for this IP?      -- yes --> use it (6-hour TTL, so reconnects
        |  no                           never re-query)
        v
   call the service, HARD TIMEOUT 1.2s
        |
        +-- answered in time --> use the place name
        |
        +-- slow, failed, or unhelpful --> return ""
                                            |
                                            v
                              folds into the EXISTING random-name default.
                              Nothing downstream can distinguish
                              "no answer" from "never asked."
```

The last property is the one worth naming. There is a single "I don't know" return
value, and it merges into a default that already existed — so the failure path is not
a special case anyone has to remember to handle. A best-effort feature that degrades
into an ordinary one is safe to put in a request path. One that degrades into an error
is not.

> **Say it in one line:** a socket is a resource an attacker gets to spend — so cap
> how many, how fast, how big, and how long.

## How a player proves who they are

> **In plain English:** there are no passwords here. Not encrypted ones — none at
> all. You either click a one-time link, or you open the game inside a chat app that
> vouches for you. Both roads end at the same cookie in your browser, and from then
> on the browser proves who you are without anyone typing anything.

Two different doors lead to the same session cookie, and neither involves the user
choosing a secret.

```
   DOOR 1 -- one-time link                DOOR 2 -- signed mini-app payload
   -----------------------                --------------------------------
   an allow-listed operator DMs           open the game inside the chat app
   the bot and gets a link back                |
        |                                      |
        v                                      v
   server mints a token with a            the platform signs a payload with
   CRYPTO-grade random source             a secret only it and the server know
        |                                      |
        v                                      v
   ONE transaction:                       verify the signature in constant time,
     consume the token AND                 reject anything older than the
     resolve-or-create the account         freshness window: 24h for a Mini
        |                                  App launch, 10 min for the
        |                                  browser login widget
        |                                      |
        +------------------+-------------------+
                           v
              create the session row  <-- NOTE: this is OUTSIDE the
                           |               transaction above. See the
                           |               gap described below.
                           v
              ONE helper sets ONE session cookie
              HttpOnly · Secure · SameSite=Lax
                           |
                           v
              the browser attaches it to everything after,
              including the WebSocket upgrade -- so the game
              protocol never had to learn about auth at all
```

Those three cookie flags each block a specific attack. `HttpOnly` means page
JavaScript cannot read the cookie, so a script injection cannot steal the session.
`Secure` means it is never sent over an unencrypted connection. `SameSite=Lax` means
another site cannot cause your browser to send it along with a forged request.

Three details carry most of the value.

**The token uses a cryptographic random source.** Not the fast generator — the slow,
unpredictable one. The rule is: if guessing a value lets someone *in*, it comes from
`crypto/rand`. A fast pseudo-random generator produces numbers that look random but
follow from an internal state, and an attacker who observes enough output can compute
the rest of the sequence. For a login token that is total compromise.

**Consuming the token and resolving the account happen in one transaction.** The
`UPDATE ... SET consumed_at = now() WHERE consumed_at IS NULL ... RETURNING` is what
makes the link single-use: only one caller can ever match that row. Resolving the
account it belongs to happens inside the same transaction, so a crash cannot mark a
link used while leaving no account behind, and two simultaneous clicks cannot create
two accounts.

**But the transaction stops one step short, and that gap is real.** Creating the
*session row* — the thing the cookie actually points at — happens after the commit.
So the failure this design set out to eliminate still exists, one step later:

```
   token consumed, account resolved, COMMIT succeeds
        |
        v
   create the session row  ---- fails (DB blip, timeout, pod killed) ----+
        |                                                               |
        v                                                               v
   normal: cookie set, player is in                        500 to the user, and the
                                                           link is ALREADY BURNED
                                                                        |
                                                                        v
                                                     clicking it again returns
                                                     "invalid, expired, or already
                                                     used" -- forever. A new link
                                                     must be issued by an admin.
```

The window is small and the blast radius is one user needing a fresh link, not data
loss. But it is the same class of bug the transaction above was written to prevent,
so it is named here rather than left to read as fully solved. The fix is to move the
session insert inside the transaction, or to make token consumption conditional on
the session insert succeeding.

**Both doors call the same cookie helper**, so the security flags above can never
drift apart between them. A second implementation is a second place to forget
`Secure`.

### Where the shuffle's randomness actually sits

The card shuffle does *not* use `crypto/rand` — it uses the fast generator. Each hand
builds a fresh generator seeded from the runtime's global source, roughly:

```go
engine.Shuffle(deck, rand.New(rand.NewSource(rand.Int63())))
```

That is a deliberate choice, and it deserves a sharper defence than "guessing it only
spoils a game," because a predictable shuffle in a multiplayer card game is a cheating
vector, not a cosmetic flaw. Reading it precisely, there are two separate questions:

- **Where does the seed come from?** On modern Go the global source is itself seeded
  from the operating system, so the per-hand seed is not something an attacker can
  predict from the clock or from a process restart. This part is fine.
- **How strong is the generator built from it?** Not strong. It is a fast
  non-cryptographic generator carrying a 63-bit seed, and its internal state can be
  reconstructed from enough observed output. A player sees their own 13 cards and
  every card played all hand — that is a lot of observed output. Recovering the state
  would reveal the other three hands.

So the exposure is real but bounded. The honest framing is what a successful guess
actually *wins*:

```
   what an attacker gets by predicting...

   the LOGIN TOKEN            the SHUFFLE
   ----------------           -----------
   another person's           an advantage in a practice-scoring
   account and session        game with no wagering, no payout,
   -- unbounded, and          and no cash-out anywhere in the API
   permanent                  -- bounded, and worth nothing off the table
        |                          |
        v                          v
   crypto/rand, always        fast generator, with the boundary
                              WRITTEN DOWN: add stakes, a ranked
                              ladder, or any persistent standing,
                              and the shuffle needs crypto/rand too
```

So the rule is not "auth is serious and games are not." The rule is that the
randomness requirement follows the value at risk, and the value at risk here is
capped by a product decision — practice scoring only — that is itself enforced in the
API. If that decision ever changes, this one changes with it. Naming the trigger is
what makes it a decision rather than an oversight.

### A flow that used to end on a page that lied

Consuming a link used to render a plain "you're logged in" page telling you to close
the tab and go back to the one that asked — which made sense only while a waiting tab
existed. Once links started arriving as a bot DM, that page began instructing people
to return somewhere they had never been: the tab it opened *was* the only tab there
was.

The fix landed the same response that sets the cookie in a redirect into the app
instead — a 303, so nothing can be tempted to replay the one-time token as anything
but the GET that consumed it. The `Set-Cookie` still applies to a redirect response,
and the browser's follow-up is a top-level navigation, which `SameSite=Lax` permits,
so the frontend's own "am I logged in?" check sees the session on first load. It is
named here rather than folded silently into the feature list, because a flow whose
last step used to lie to the user is exactly the kind of bug a portfolio is tempted to
quietly fix in prose without saying so.

### Retiring a door properly

The email door was retired along the way, and *how* is the point. It wasn't put behind
a feature flag — the endpoint was **deleted**. A flag would have left "is this route
live?" answerable only per-deployment, when the honest answer was "never again."
Hiding a form stops honest users, but a direct POST would still have written an
unverified email address into the database.

Its sibling — the endpoint that turns a token into a session — went the other way. It
is no longer gated on the chat-platform token at all; it is registered whenever the
service has a database, which in production is always, because it is where both
surviving doors finish.

```
   BEFORE                              AFTER
   ------                              -----
   POST /register  \                   POST /register   -> DELETED, not flagged
   GET  /session   /  both behind      GET  /session    -> registered whenever a
                      the same flag                        database is configured
                                                           (both doors end here)

   Two routes that once lived and died together now have opposite lifetimes,
   because only one still has a job.
```

**One honest footnote on "deleted."** The *route* is genuinely gone — nothing
registers it, so no request can reach it. But the consumption path it used to feed
still contains its branch: the token-verifying transaction can still resolve a token
that carries an email address by get-or-creating a user from it. No endpoint can mint
such a token any more, so the branch is unreachable in practice. It is dead code, not
a live door — but "the door is gone" and "the corridor behind it was demolished" are
different claims, and only the first one is true here.

> **Say it in one line:** retire the endpoint, not just the UI — and when a flow
> changes shape, check that the page it ends on still tells the truth.

### The same door, gating a very different room

Door 1 above already starts from "an allow-listed operator DMs the bot" — which is
exactly the precondition an admin dashboard needs. So the operator console that lists
players, inspects a wallet, and revokes a session or a token doesn't get its own login.
It reuses the one-time-link door verbatim, on its own subdomain, with one addition: a
second, independent check on every single request, not just at link-mint time.

```
   same link, same cookie, same session row  (Door 1, unchanged)
                    |
                    v
   every admin request re-resolves the caller, THEN asks one more question:
   is this account's Telegram id on the admin allow-list?
                    |                                  |
                 yes                                   no  (or not logged in at all)
                    |                                  |
                    v                                  v
              the dashboard                      404 -- identical response
                                                  either way, so a probe can't
                                                  tell "wrong door" from
                                                  "no door here"
```

Two details worth naming. First, the allow-list check runs on *every* request, not
once at sign-in — a session that was valid and non-admin an hour ago is checked again
right now, so there's no window where a stale cookie outlives a change to the
allow-list. Second, an unauthenticated request and an authenticated-but-not-admin one
get the *same* `404`, deliberately: a `403` would confirm to anyone probing the host
that an admin route exists at all, which is one bit of information it costs nothing to
withhold.

The dashboard also sits on its own subdomain with its **own** certificate, not folded
into the game's — the same "don't let a not-yet-live host block renewal of an
already-live one" reasoning as the [TLS issuance](#tls-issuance-and-why-there-are-two-issuers)
section above, applied to a second host instead of a second issuer.

> **Say it in one line:** the safest new door is the one you don't build — reuse the
> login you already trust, and add the one check that's actually new.

## Playing with the people you came with

> **In plain English:** if you launch the game from a group chat, you should end up
> at a table with those people. But "should" is doing real work there — if the system
> insists on it, four friends where only three showed up wait forever. So the chat is
> a *preference* that decays into fallbacks, never a hard wall.

The game can be launched from inside a group conversation, which changes what
matchmaking means: not "queue of strangers" but "the people from *this* chat —
preferably." That last word carries the design.

### Getting the chat's identity from one connection to another

The chat identifier arrives inside the same signed payload as the user, so it needs
no separate trust. But it is *proved* on the HTTP login request and *used* on the
WebSocket — a different connection, possibly served by a different replica, with
nothing shared between them. Rather than add a session store for one string, the key
takes a round trip through the client:

```
   HTTP login                            WebSocket join
   ----------                            --------------
   signed launch payload arrives
   signature verified
   table key derived  -------+
                             |    a DIFFERENT connection, possibly a
   key returned MAC'd        |    different replica -- nothing is
   (signed, not encrypted)   |    remembered between the two
                             v
                    client sends the token back
                             |
                             v
                    verify the MAC -> recover the key -> queue there
                    (unverifiable or absent -> the open queue)
```

A **MAC** is a short tag computed from the data plus a secret key. Anyone holding the
key can check the tag; nobody without it can produce a valid one. "Signed, not
encrypted" means the value is readable — it is protected against *tampering*, not
against *reading*.

**What the MAC buys, and what it deliberately does not.** The token doesn't need to be
secret: the player was already in the chat it names, so its contents reveal nothing
they didn't have. It has to be **unforgeable**, which is exactly what a MAC provides —
a player cannot mint a token for a chat they were never in.

But unforgeable is not the same as **non-transferable**, and that gap is real: a
player can hand their valid token to someone who was never in that chat, and the
server will accept it.

```
   FORGERY -- blocked                    TRANSFER -- currently possible
   ------------------                    -----------------------------
   attacker invents a token for          player copies their own valid
   a chat they were never in             token and gives it to an outsider
        |                                     |
        v                                     v
   MAC check fails -> open queue         MAC check passes -> they join
                                              |
                                              v
                                    accepted, because the worst case is an
                                    uninvited player at a casual, unstaked
                                    table -- the same exposure as someone
                                    forwarding the chat's invite link

   The close: bind the MAC to the user id it was ISSUED to, and reject a
   token presented by anyone else. Cheap, and named here rather than
   discovered by an interviewer.
```

That is the honest state: a bounded, understood gap with a known fix, not a property
the design silently assumes it has.

Two more defensive details. The key is **domain-separated** from the other signatures
that share the same secret — each use prepends a distinct label before signing, so a
tag minted for one purpose can never be replayed as another. One secret signing three
different things without separation is how cross-protocol replays happen. And with no
key configured the verifier is absent entirely, so everyone lands in the open queue
rather than the server trusting a claim it cannot check. **Unverifiable input gets the
least-privileged outcome, never the convenient one.**

### The second thing that round trip fixed — and the bug it was built for

The table key is one of the three signed round trips sharing that key. The second is
sharper, because what rides on it is not seating but *money*: a player's own chip
balance.

`/ws` normally learns who is playing from the session cookie — but inside a Mini App
on Telegram Desktop or Telegram Web, this server is loaded in an **iframe** on a
`telegram.org` page. That makes it a third party, and a `SameSite=Lax` cookie is
neither stored on login nor sent on the socket upgrade from inside one:

```
   INSIDE A MINI APP (Desktop / Web)         WHAT THE PLAYER SEES
   --------------------------------          --------------------
   telegram.org page
     +-- iframe: this server         --      "Logged in" -- the client reads its
          |                                  nickname straight out of the login
          | POST /auth/telegram               response body, not the cookie
          | response tries to                       |
          | Set-Cookie ...                           v
          |         X   THIRD-PARTY,           looks completely fine
          |             SameSite=Lax
          |             cookie DROPPED
          v
   WebSocket /ws upgrade
     -- no cookie arrives --
          |
          v
   seated as a GUEST, on a fresh random session id
          |
          v
   wallet opens at StartingBalance -- EVERY SINGLE LAUNCH
```

That is the exact shape of the "Telegram players' points never accumulate" bug: not
the wallet, not the scoring — an identity that silently failed to travel one hop
further than the login response. Nothing errored. The login even *looked* right, which
is what let it hide.

The fix is the same shape as the table key: sign the identity server-side at login,
hand it to the client, have the client echo it back on the socket upgrade the cookie
never reaches. A client can't forge one, so it grants exactly the authority the cookie
would have and no more — and because it doesn't ride in a cookie at all, it is immune
to whatever a browser's cookie policy decides that day.

**One property is unique to this token and worth naming.** Every other credential
here expires or can be killed by hand: the session cookie in 30 days or on logout, a
magic link on click or on `/revoke_tokens`. This one, unbounded, would stay valid until
the bot token itself rotated — while it names an account on `/ws` and decides whose
wallet a round settles into. So it carries its own issue time inside the signed
payload, and a verifier-side max age (24h, matched to the Mini App's own initData
freshness window) is
non-negotiable in production. Generous on purpose: a launch mints one fresh token, but
a tab can sit open far longer than that, and too short a window means a long-lived tab
quietly drops back to a guest wallet mid-session — the identical silent failure this
mechanism exists to prevent, just reintroduced by an overzealous expiry instead of a
missing one.

> **The pattern, stated once:** when a credential can't ride the transport it needs to
> (a cookie a proxy strips, a cookie a third-party frame drops), sign it and hand the
> round trip to the client instead — and anything with no natural expiry needs one
> written into the payload, not assumed from context.

### Sorting players without trapping them

```
   four from one chat        -> start immediately
        |
        | nobody filled from their own chat within ~5s
        v
   combine tables, longest-waiting first, each group kept contiguous
        |
        | still short-handed at ~10s
        v
   OFFER to fill the empty seats with bots -- one human accepting starts it
```

Merging comes first because playing with other humans, even strangers, beats playing
with bots. And the third tier is an *offer*, which reversed the original behaviour:
bots used to seat themselves on a timeout, which meant a group waiting on a fourth
friend could have the game start without them. A timeout that acts on your behalf is
a decision you never got to make.

The server sends the client what it needs to render — how many are seated, out of how
many, and whether the offer is open — and then re-checks both rules when the request
actually arrives. A modified client that reveals the button early achieves nothing.
**Render from server state; authorize on server state.**

> **Say it in one line:** a grouping key should be a preference with fallbacks, not a
> partition — because never seating anyone is worse than seating them with strangers.

### Talking to the table, and a feature check that lies

Seated players can talk: a chat panel alongside the table, relayed by the server (not
echoed by a client, so nobody can spoof another seat's line) and capped at 200
characters both ends. Small on purpose — table talk, not a messaging app.

The more interesting piece sits one layer earlier, in the composer: a mic button that
lets a player dictate a line instead of typing it. `SpeechRecognition` is a real
browser API, and the obvious way to offer the button is feature-detecting the
constructor. That check **lies** on one platform that matters a great deal here: inside
Telegram's iOS Mini App, `webkitSpeechRecognition` exists, reports itself supported,
and then simply never produces a result — a documented WebKit bug in the exact engine
this game's largest slice of mobile players sits inside.

```
   THE CHECK THAT LIES                    WHAT ACTUALLY HAPPENS ON iOS
   --------------------                   -----------------------------
   "window.webkitSpeechRecognition        recognition.start()
    exists?" -- YES                            |
        |                                      v
        v                                 no onstart, no onaudiostart,
   show the mic button                    often no event at ALL
        |                                      |
        v                                      v
   player taps it, talks --            a mic button that is permanently
   nothing happens, ever                dead, with no error to explain why
```

A capability check answers "does the API exist," not "does it work" — and on this one
platform those are different questions with different answers. So availability here is
decided by a **probe**, not a check: attempt recognition for real, and if the engine
doesn't audibly come alive within 1.5 seconds — far past the few tens of milliseconds
a working implementation takes — record that verdict and stop offering the button on
that device. The verdict is written to `localStorage` rather than re-probed every
render, because it is a property of the *browser engine*, not of one attempt: asking
again on the next screen would just mean a dead button that times out freshly each
time instead of never appearing at all.

Nothing about this is required to play. The composer is a text input first and
dictation is an accelerator on top of it — a player whose device fails the probe types,
exactly as if the feature had never shipped. **A feature check that only asks "does the
API exist" can pass on a browser where the feature doesn't actually work — probe the
real behaviour, not its advertised name**, the same rule the [table-key
MAC](#getting-the-chats-identity-from-one-connection-to-another) applies to a claim a
client makes about itself: verify it, don't take its word.

One more command belongs in this story for what it *doesn't* do: `/voice`, typed to the
bot in a group chat, replies with directions to that group's own native Telegram voice
chat. It doesn't start one — the Bot API has no method to, a voice call is a
first-party Telegram feature this server can only point at, never operate. Naming that
boundary is the honest version of "voice chat support": real directions to a real
feature, not a facade over one this server doesn't run.

## The protocol, and why a modified client can't cheat

> **In plain English:** in a card game, the browser is the enemy. Anyone can open
> developer tools and change what their copy of the page does. So the rule is: never
> send a player information they aren't allowed to have, and never trust anything they
> send back. The strongest version of "the client can't cheat" is *the client never
> had the data.*

The whole game runs over the one WebSocket opened at `/ws`. It is a small protocol —
four messages up, four messages down — and it carries the session cookie the browser
already attached at the upgrade, so the protocol itself never had to learn about auth.

```
   CLIENT -> SERVER                    SERVER -> CLIENT
   ----------------                    ----------------
   join      take a seat               joined    your session, your nickname
   play      these cards               waiting   N of 4 seated, offer open?
   pass      I can't or won't          state     the whole table, as YOU see it
   fillBots  start with bots           error     that move was rejected
```

Four verbs is the entire game. Everything else — whose turn it is, what beats what,
whether a round ended — is derived on the server and pushed down inside `state`.

### The one message that carries the security property

`state` is sent to every seat after every action, and it is **not the same message for
everyone**. Each player's copy contains their own cards and nobody else's:

```
   what seat 2 receives                what seat 2 does NOT receive
   --------------------                ----------------------------
   hand:       [ their 13 cards ]      any other player's cards
   handCounts: [ 13, 13, 9, 13 ]           |
   turn:       2                            +-- not encrypted,
   previous:   the combo to beat            +-- not obfuscated,
   passed:     [f, f, f, t]                 +-- NEVER SENT.
   legalMoves: [ ...computed server-side ]
```

That is the difference between hiding information and not sending it. A client that
hides opponents' cards can be modified to stop hiding them. A client that was never
sent them has nothing to reveal — the data does not exist in that browser, so no
amount of tampering produces it.

```
   THE WEAK PATTERN                    THIS PATTERN
   ----------------                    ------------
   server sends the full table         server sends each seat only its
   client renders only your hand       own hand plus counts
        |                                   |
        v                                   v
   open devtools, read the array,      there is no array to read.
   see all four hands                  The cheat requires the SERVER
                                       to be compromised, not the client.
```

**`legalMoves` is the same idea from the other direction.** The server computes which
combinations a seat may legally play and sends that list down — so the UI can grey out
illegal moves without shipping the rules engine to the browser. But sending it is a
*convenience*, not a permission: every incoming `play` is re-validated against the
authoritative game state anyway. A client that invents a move it was never offered
gets an `error` and nothing else.

The rule generalises to the matchmaking button from the previous section, and it is
the same sentence in both places: **render from server state, authorize on server
state.** The client is told enough to draw a screen and trusted for nothing.

### Nobody gets to stall the table

A turn-based game has an obvious denial-of-service problem that has nothing to do with
attackers: someone closes their laptop mid-hand, and three other people wait forever.

```
   a seat's turn begins
        |
        v
   wait up to 30 SECONDS for a legal move or a pass
        |
        +-- it arrives ------------------> play it, next seat
        |
        +-- silence, or the socket died --> a BOT plays for that seat
                                            |
                                            v
                                   the hand continues. The table is
                                   never blocked on one person.
```

The room treats "slow human," "disconnected human," and "bot" **identically** — all
three are just a seat that did or didn't answer in time. That uniformity is why there
is no special disconnection-handling code path to get wrong: a vanished player is
simply a seat that times out, forever, and the existing fallback covers it.

The bot filling in is deliberately simple — a heuristic over the same legal-move
generator the server already uses for validation, with no lookahead. Leading, it plays
the lowest combination of the largest shape it holds, so it sheds five-card hands and
triples rather than dribbling out singles. Responding, it plays the lowest legal combo
that beats the table, or passes. It is not trying to be a good opponent; it is trying
to be an *unobtrusive* one, and a bot that thinks for two seconds before acting reads
as a person taking a turn rather than a machine snapping one off.

> **Say it in one line:** the strongest anti-cheat is data the client never
> received — so send each seat only its own hand, and re-validate every move anyway.

## The browser client

> **In plain English:** the part that runs in your browser is a real piece of
> engineering too, not just a skin. It has to work on any hostname without being
> rebuilt, keep a live socket healthy, speak two languages, and read the game aloud —
> all without a backend of its own.

Four decisions in the frontend earn their place in an architecture document.

**One image, any hostname.** The bundle never has a server address compiled into it.
It derives one from the page it was served by:

```
   window.location = https://game.example.com/
                            |
                            v
   scheme https -> wss                 hostname taken as-is
                            |
                            v
        wss://game.example.com/ws      and the auth base URL is derived
                                       from that same string, so the socket
                                       and the login calls can never disagree
```

That single derivation is what makes the same built artifact run in dev, in staging,
and in production without a rebuild — and it is why the
[same-origin decision](#same-origin-hosting) pays off twice: once for cookies, once
for build simplicity. Development is the only case that needs an explicit fallback,
because Vite serves the page on one port and the Go server listens on another.

**The wire types are mirrored, not shared.** The client keeps its own TypeScript copy
of the message shapes. Go and TypeScript can't share a type definition without a code
generator, so this is duplication by necessity — the honest trade is that the two
must be changed together, and a protocol test on the Go side is what keeps the drift
visible.

**The dealer speaks, with no assets and no network.** Each play is announced aloud
through the browser's own speech synthesis — no audio files to host, no request to a
voice service, nothing added to the image. It is entirely best-effort by design: where
speech synthesis is unavailable, or no voice matches the active language, it simply
stays silent rather than erroring. It also shares the one mute toggle with the sound
effects, so "sound off" means off, including mid-sentence.

**Bilingual, in the register the game is actually played in.** The second locale is
colloquial rather than formal — the words people use at a real table — selected
automatically from the device's own signals with a manual override. That is a product
decision more than a technical one, but it is the kind that has to be designed in from
the start: retrofitting a second language into a UI built for one is a rewrite.

> **Say it in one line:** derive the server URL from the page's own origin and one
> built image runs anywhere — and a best-effort feature that degrades to silence never
> needs an error path.

## TLS issuance, and why there are two issuers

> **In plain English:** the padlock in the address bar comes from a free authority
> that hands out certificates automatically. It also rate-limits you hard, and the
> limit resets on a *weekly* clock with no appeal. So you rehearse against a practice
> version of the same service first — one that issues junk certificates freely.

The certificate manager is configured with **two** issuers against the same public
authority: a staging one and a production one. This isn't caution for its own sake.
Production rate limits are counted per registered domain, per week, with no appeal.
Point a misconfigured ingress at production, or let the challenge fire before DNS
resolves, and the real certificate is locked out for days.

```
   change an ingress, a hostname, or the DNS
              |
              v
   [ STAGING issuer ]   loose limits, untrusted root certificate
              |         -> useless to actually serve, perfect for proving
              |            the whole path works end to end
              |
              | it issued cleanly
              v
   [ PRODUCTION issuer ]  counted per REGISTERED DOMAIN, per WEEK, no appeal
              |
              v
        a browser-trusted certificate

   Skip the rehearsal and a typo costs you days of no padlock -- and the
   clock is weekly, so there is nothing to do but wait.
```

### Why HTTP-01, and the constraint behind it

The challenge type is HTTP-01 rather than DNS-01. HTTP-01 proves control by serving a
file at a known path over port 80, so it needs nothing but a public port and a
hostname that already resolves.

There are two reasons for that choice, and honesty requires separating them, because
only one is a preference:

```
   THE PREFERENCE                      THE HARD CONSTRAINT
   --------------                      -------------------
   DNS-01 means handing the            the domain's nameservers are run by
   certificate manager API             a registrar with NO supported
   credentials for the DNS             DNS-01 solver in the certificate
   provider -- a powerful,             manager at all
   long-lived secret -- and it              |
   buys only wildcards, which               v
   this platform doesn't need          DNS-01 was never actually on the
                                       menu. HTTP-01 wasn't chosen over
                                       it so much as it was the only
                                       option that existed.
```

Both belong in the record. A document that presents a forced move as a considered
trade-off is subtly lying about how much was decided — and the constraint is the more
useful fact, because it is the thing that would change if the domain ever moved to a
different provider.

### DNS itself, and the one thing that isn't code

Certificates and DNS came up together as the third layer, and DNS deserves its own
paragraph because of what it reveals.

The records are ordinary and few: an A record per hostname, all pointing at the same
load-balancer IP — apex and `www` to the toolbox, the game's hostname to the game, and
one each for the dashboards and the GitOps UI. The ingress does the actual routing, so
DNS only has to get a browser to the front door.

**And these records are the one piece of infrastructure that is not code.** Every
network, every node pool, every identity, every policy is a Terraform resource. The
DNS records are typed into a registrar's control panel by hand.

```
   IN TERRAFORM                        NOT IN TERRAFORM
   ------------                        ----------------
   network, subnets, gateways          the DNS records
   cluster, node pools                      |
   every machine identity                   v
   the backup bucket + policies        typed into a registrar's web
   the burst pool and its cap          console, by a human, with no
        |                              plan, no review, and no record
        v                              of who changed what when
   reviewable, reproducible,
   destroyable, auditable
```

That asymmetry matters more than it looks. A mistyped A record is a total outage that
no `terraform plan` would have caught, no code review would have flagged, and no
version history can explain afterwards. It is also the reason the
[external probe](#the-monitor-that-cant-be-taken-down-by-what-it-monitors) checks
hostname resolution from outside — for the one layer with no change control, detection
is the only control available.

The fix is not exotic: the registrar has an API, and a DNS provider with a Terraform
provider would put these records under the same review as everything else. It is
listed here because "100% infrastructure as code" is a claim worth being precise
about, and the precise version is *everything except DNS*.

> **Say it in one line:** rehearse on staging, because production rate limits are
> per-domain, per-week, and there's no appeal.

## Data durability: how far back you can rewind

> **In plain English:** a nightly backup means that when something goes wrong at
> 3 p.m., you lose the whole day. Continuously recording every change instead means
> you can rewind to any second you like — including the second before someone ran the
> wrong `DELETE`.

PostgreSQL runs *inside* the cluster under the CloudNativePG **operator**, not as a
hand-written StatefulSet. An operator is a program running in the cluster that knows
how to run one specific piece of software properly — it watches, repairs, upgrades,
and backs up on your behalf.

The distinction matters. A StatefulSet plus a volume claim gets a database *process*
running. It gives you none of what makes a database survivable: consistent backups,
point-in-time recovery, credential rotation, safe minor upgrades. The operator
encodes all of that, and it mints the application's database password directly into
the cluster — so that credential never appears in git or in a command anyone typed.

The Postgres major version is **pinned**, not floating, for a reason worth stating
plainly: an operator that silently moved the major version would be a data-migration
event disguised as a pod restart.

### There is exactly one instance, and it cannot fail over

Before any of the backup machinery below, the honest headline:

```
   instances: 1
```

One Postgres pod. No replica, no standby, no automatic failover. If that pod's node
dies, the database is **down** until it is rescheduled and its volume reattaches —
and the volume reattachment alone costs about six minutes when the old node is
unreachable, because the previous attachment has to time out first.

This is an accepted limit rather than an oversight, and the reasoning is specific:
there is exactly one worker node, so three replicas would all land on **that same
node**. That buys the appearance of redundancy with none of the substance — three
copies that die together, plus three times the resource cost, plus the operational
weight of managing replication.

```
   WHAT 3 REPLICAS WOULD BUY          WHAT ACTUALLY PROTECTS THE DATA
   HERE, TODAY                        HERE, TODAY
   -----------------------            -------------------------------
   node A  [ db-1 ]                   continuous WAL archiving to
           [ db-2 ]                   object storage, OFF the node
           [ db-3 ]                        |
              |                            v
              v                       the node can burn down and the
   the node dies -> all three die.    data is still recoverable to
   Zero availability gained,          within seconds of the failure
   triple the cost.
```

So this system trades **availability** for **durability**, knowingly. A node failure
means downtime; it does not mean data loss. Raising `instances` to 3 is the correct
move the moment the node pool grows past one node — and not before, because until
then it would be theatre.

That is worth being blunt about, because "we run Postgres under an operator with
point-in-time recovery" sounds like high availability and is not. Durability and
availability are different properties, and this system has bought exactly one of them.

Backups are continuous **write-ahead-log** archiving to object storage, not a nightly
snapshot. The write-ahead log is how Postgres already works internally: every change
is written to an append-only log *before* it touches the data files. Shipping that log
off-machine as it is written means the backup is a complete record of every change,
not a photograph taken once a night.

```
   time ----------------------------------------------------------->

   3:00am                                        2:55pm        now
   base backup                                   bad DELETE     |
      [B]======== continuous WAL (every change) =====[X]========|
                                                      ^
      restore = [B], then replay the log forward, STOP one second
                before the mistake

   snapshot only :  newest safe point is 3:00am   -> lose ~12 hours
   base + WAL    :  newest safe point is 2:54:59  -> lose seconds
```

The vocabulary an interviewer will use for those two numbers: **RPO** (recovery point
objective) is how much data you accept losing — the gap in that diagram. **RTO**
(recovery time objective) is how long you accept being down while you restore them.
Snapshots give a bad RPO; WAL archiving gives a good one. Neither says anything about
RTO until you have actually timed a restore.

**And here is the honest part.** The archiving is configured and an on-demand backup
was verified to have written real objects to the bucket. **No restore has ever been
performed from them.** By this project's own standard that makes recovery a
hypothesis, and rehearsing it — with a defined RPO and RTO — is named as the next
phase rather than quietly implied to be done.

### What is actually in there

Four things, and the shape of the list is itself a design decision:

| Table | Holds | Notes |
| --- | --- | --- |
| **users** | one row per account | Keyed by chat-platform id. No password column, because there are no passwords. |
| **login tokens / sessions** | short-lived credentials | Tokens expire in 15 minutes, sessions in 30 days — and an hourly sweep **deletes** expired rows rather than just ignoring them. |
| **ledger** | practice-chip balances | Every player opens at 1000 chips. |
| **match results** | one row per finished round | What makes a hand persist beyond the pod that dealt it. |

Two of those deserve a sentence each.

**Expiry and deletion are different promises.** Every read path already filters on
`expires_at`, so a stale row is unusable the moment it expires. It would still *exist*
forever. Retention is a promise the privacy policy makes on the site's behalf, so
something has to actually do the deleting — an hourly sweep, unbounded `DELETE`,
cheap at this scale.

**The ledger is where "practice scoring only" stops being a claim.** Balances are real
and persistent, so the game has stakes in the ordinary sense — but there is no
deposit, no payout, and no cash-out path anywhere in the API. That boundary is what
[bounds the shuffle-randomness trade-off](#where-the-shuffles-randomness-actually-sits)
made earlier: a predictable shuffle wins you chips that can never leave the table.

### A migration that fails does *not* stop the server

The last decision here is counterintuitive, and it was paid for with an outage.

The obvious behaviour on a failed schema migration is to refuse to start. That is what
this server used to do — and it caused a live outage, because the failure mode is
vicious: a migration interrupted partway leaves the schema marked "dirty," every
subsequent restart hits the same error, and the crash loop is *permanent*. The server
never comes back on its own.

```
   FAIL CLOSED (what it used to do)     FAIL OPEN (what it does now)
   ---------------------------------    ----------------------------
   migration fails                      migration fails
        |                                    |
        v                                    v
   log.Fatal -> process exits           log it LOUDLY, keep going on
        |                               whatever schema exists
        v                                    |
   restart -> same error -> exit             v
        |                               guest play works. Chat login
        v                               works (it only needs the
   CrashLoopBackOff, forever.           long-stable users table).
   The whole site is down over a             |
   schema change.                            v
                                        fix the dirty migration out of
                                        band, at human speed
```

The reasoning: the core flows do not depend on the newest migration, so **staying up
beats staying consistent-or-dead**. A degraded site that serves most players beats a
crash loop that serves none.

This is a genuine trade, not a free win. Failing open means the server can run against
a schema it does not fully expect, and a migration that half-applied could produce
confusing errors later. It is the right call *here* because the blast radius is
bounded and the alternative was a total outage — not because failing open is generally
safer. Reach for it when the failure is recoverable at human speed and the alternative
is unrecoverable without a human anyway.

> **Say it in one line:** a snapshot restores to one instant and the log restores to
> any instant — but an untested backup is still only a hypothesis.

## Scaling, safely and within budget

> **In plain English:** "add more copies of the app" and "add more computers" are two
> completely different buttons. Pressing the first one when you needed the second
> gets you a copy of the app sitting in a corner doing nothing. And the second button
> spends money, so it needs a hard limit that a runaway program cannot argue with.

Elasticity happens at three levels, and conflating them is the classic mistake:

```
  Pod    -- a HorizontalPodAutoscaler raises the game server 1 -> 2 on load
  Node   -- a fixed baseline node runs everything stateful (DB, server, ingress)
  Node   -- a scale-to-zero BURST pool absorbs overflow the baseline can't hold
```

The load-bearing insight: **an autoscaler for pods cannot create capacity.** If it
scales the server to two pods and the second doesn't fit on the one baseline node,
that pod just sits `Pending` — not failing, not working, and not generating an error
anyone will see. Only *node* autoscaling adds real capacity. So both layers exist,
and the node layer is bounded hard.

**Bounded, because the guardrail assumption broke.** Node autoscaling was originally
scoped out on the theory that a fixed node pool was a *physical* ceiling — the account
simply could not create more machines, so nothing could run away. A live check of the
account killed that theory: far more quota than a pure free tier, and no physical
guardrail at all. Unbounded autoscaling would have provisioned billable nodes with
nothing to stop it.

The fix wasn't "never autoscale." It was **autoscale with a hard, documented
ceiling** — a second pool that is tainted, scales to zero, and is capped at a small
`max` that *is* the money limit. The autoscaler physically cannot exceed that node
count no matter what a runaway workload requests.

Two Kubernetes terms make the next diagram readable. A **taint** on a node means "no
pod may land here unless it explicitly says it accepts this." A **toleration** on a
pod is that explicit acceptance. Together they let a node pool exist without anything
drifting onto it by accident.

**The two autoscalers hand off through the scheduler, not directly.**

```
   load rises
       |
       v
   [ pod autoscaler ] adds a replica          <-- adds no capacity, only demand
       |
       v
   [ scheduler ] tries the baseline node first (no node-selector = preferred)
       |
       +-- it fits ------------------------> running on the FREE baseline node
       |
       +-- it does NOT fit --> pod Pending
                                   |
                                   v
                        [ node autoscaler ] sees a Pending pod that TOLERATES
                        the burst pool's taint, so it adds a burst node
                                   |
                                   v
                        replica runs on a BURST node ($, hard max = the cap)
                                   |
                          load falls; pool scales back to ZERO ($0 at rest)
```

The toleration *is* the handoff. Without it the overflow pod sits `Pending` forever
and the node autoscaler never fires. And the deliberate *absence* of a node-selector
is what keeps the server on the free baseline until it genuinely overflows — a
selector would pin it to billable nodes permanently, turning a burst pool into a
standing bill.

**Two honest caveats, both written into the manifests.** The pod autoscaler's CPU
target is a **committed placeholder**, to be right-sized from a k6 WebSocket load test
— each virtual user plays a full hand — that has been written but not yet run. And CPU
is a weak signal for this workload anyway: WebSocket connections are long-lived and
mostly idle, so CPU barely moves with connection count. Ten thousand idle sockets and
ten busy ones can look identical to a CPU gauge. The right signal is active rooms or
live connections per pod, and the upgrade path to it is staged in the file.

### What the ceiling costs, in money

"Bounded" is only meaningful with a number attached, so the number lives in the
variable file next to the setting it bounds:

```
   worst case = burst_pool_max x (ocpus x $0.01 + GB x $0.0015) x 730h

   at the defaults -- 2 nodes, 2 OCPU and 12 GB each:

        2 x (2 x $0.01 + 12 x $0.0015) x 730   =   ~$55/month

   ...and that is the figure for every burst node running FLAT OUT, all
   month, never scaling down. Realistically the pool sits at zero and
   costs $0; this is the number that cannot be exceeded, not the
   number expected.
```

Writing the arithmetic down — rather than just "it's capped" — is what makes the cap
reviewable. Anyone can check whether `$55/month` is an acceptable worst case; nobody
can check whether "bounded" is. And the lever is `burst_pool_max`, not the node size:
a bigger node raises the bill linearly, but the *count* is what turns a runaway
workload into a runaway invoice.

### Elasticity that's safe for a stateful game

Rooms are in-memory and pod-local, so retiring a pod abruptly would drop every hand on
it. Kubernetes sends `SIGTERM` — a polite "please finish up" signal — before killing a
pod on *every* scale-down, node drain, and rollout. Handling it is the difference
between elasticity and data loss.

```
   SIGTERM arrives
       |
       v
   stop accepting new connections   (the door closes)
       |
       v
   form no new rooms                (the queue stops feeding)
       |
       v
   wait while hands are in progress ... bounded by a drain window
       |                                 kept UNDER the pod's grace period,
       |                                 so the process always exits on its
       |                                 own terms, never mid-write
       v
   exit -- a hand is cut only if it outlasts the window,
           and even then it's a client reconnect, not lost data
```

The drain window sits *under* the grace period on purpose. The grace period is the
hard deadline after which Kubernetes stops asking and sends `SIGKILL`, which cannot be
caught or handled. Finishing first means the process always chooses its own exit
point.

One more subtlety worth knowing: a **PodDisruptionBudget** guards *voluntary*
disruptions like a node drain — it does **not** gate autoscaler scale-down, which
deletes pods directly. Protecting in-flight hands from scale-down is a different
mechanism, which is exactly why the drain above exists.

And the budget is written as "at most one unavailable" rather than the more obvious
"at least one available," because at a one-replica floor the obvious version deadlocks
every drain forever:

```
   one replica running, and you try to drain its node:

   "at least 1 available"   -> evicting the only pod would leave 0 available
                               -> eviction REFUSED -> the drain waits forever
   "at most 1 unavailable"  -> evicting one pod leaves exactly 1 unavailable
                               -> allowed -> the pod moves, the drain completes
```

> **Say it in one line:** a pod autoscaler reshuffles capacity and only nodes create
> it — so the `max` on the node pool is the real cost control.

## Seeing inside it: metrics, logs, and traces

> **In plain English:** three different ways of watching your own system. Metrics are
> numbers over time — *how many, how fast, how often*. Logs are sentences about
> individual events — *what happened, exactly*. Traces follow one single request
> through everything it touched — *where did the time go*. You need all three because
> each answers a question the others can't.

A system you can't see inside is one you operate by guessing. All three pillars run
in-cluster, and each answers a different question. The stack is Prometheus and Grafana
for metrics, Loki with a Grafana Alloy agent for logs, and OpenTelemetry into Tempo
for traces — each installed through its own official chart, then trimmed to fit one
small node.

**Metrics** are the RED signals — **R**ate, **E**rrors, **D**uration, the three
numbers that describe any request-serving system — plus game-domain gauges no generic
exporter could know to collect: live connections, hands in progress, human-versus-bot
seats, and join attempts split by outcome. Those domain metrics answer "is anyone
actually *playing*?", which uptime never does. A perfectly healthy server with zero
players looks identical to a healthy busy one on every generic dashboard.

**Logs** are structured JSON at the source, shipped by a collector agent, so a field
is a field rather than a regex against a sentence. Getting there cost exactly one line
of code: Go's `slog.SetDefault` also redirects the standard `log` package through the
JSON handler, so 35 existing call sites became structured records with no edits.

**Traces** are OpenTelemetry spans that follow a request through the server and into
its database calls. The payoff is correlation — from a slow trace, jump straight to
the log lines that span emitted, without guessing at timestamps. Honest scope: this is
one Go binary talking to Postgres, so most traces are shallow. The deliverable is the
*shape* — SDK, exporter, store, correlated in one pane — which is what scales later.

```
                        the game server (one pod)
                                   |
       +---------------------------+---------------------------+
       |                           |                           |
   METRICS                       LOGS                       TRACES
   RED signals + domain          structured JSON            spans through the
   gauges, on a SEPARATE         at the source              server and into
   in-cluster-only port          (no regex parsing)         its DB calls
       |                           |                           |
       v                           v                           v
  +------------+             +------------+             +------------+
  | metrics DB |             |  log store |             | trace store|
  +------------+             +------------+             +------------+
       |                           |                           |
       +---------------------------+---------------------------+
                                   |
                                   v
                        +----------------------+
                        |  one dashboard UI    |  dashboards committed as code
                        +----------------------+  a slow TRACE links straight
                                                  to the LOGS it emitted

  The public ingress routes only the game's paths -- it has NO route to the
  metrics port at all. Scraping happens inside the cluster, by design.
```

Dashboards are committed as JSON and loaded from version control, so a rebuilt
dashboard server comes back identical instead of losing someone's afternoon of
clicking. And the whole stack is, honestly, *bigger than the app it watches* — which
is precisely how it exposed a capacity problem the app alone never would.

### The monitor that can't be taken down by what it monitors

> **In plain English:** a smoke alarm wired to the same fuse as the kitchen is not a
> smoke alarm. If everything you use to watch the system lives *inside* the system,
> then the one failure you most need to hear about is the one that silences it.

Every pillar above runs **inside** the cluster. That's a structural blind spot, and
it's worth stating plainly: if the node, the load balancer, the ingress, DNS, or the
certificate fails, the monitoring fails *with* the site and nothing fires. The system
is least able to tell you it's broken exactly when it is most broken.

```
   INSIDE the cluster                  OUTSIDE it
   ------------------                  ----------
   metrics · logs · traces             a scheduled probe on someone
   alerting rules                      else's infrastructure
        |                                   |
        v                                   v
   tells you WHY it broke              tells you THAT it broke
        |                                   |
        |  shares a failure domain          |  survives node loss, balancer
        |  with the thing it watches        |  loss, ingress loss, DNS, and
        v                                   |  an expired certificate
   node/LB/ingress/DNS dies                 v
   -> the monitoring dies too          still reports -- and opens a ticket
   -> NOTHING fires                    with the failing detail attached
```

So a scheduled external probe checks the public endpoints and the certificate's
expiry from outside that blast radius, and files an issue when they fail. It aims at
`/queue` rather than `/`, because `/` is answered by the static frontend's nginx and
would keep returning 200 with the game server dead — the probe has to fail when the
thing you care about fails.

On failure it doesn't page on the first bad run. It takes **two consecutive**
failures — because the runner executing the probe is its own failure domain, and a
GitHub Actions outage (the job's runner never gets acquired) reads identically, from
the alert, to a real site outage, but clears on its own within one cycle:

```
   1st failure                    2nd CONSECUTIVE failure         recovery
   ------------                   ------------------------        --------
   open a QUIET issue             ESCALATE that same issue        comment "recovered",
   ("uptime-watch")               to PUBLIC + fail the run         close whichever
   run stays green                ("uptime-alert")                 issue is open
        |                                |                              |
        v                                v                              v
   holding for a blip          confirmed, not a blip            back to holding
   confirmation next run       -- this is the page

   one label lookup makes the whole thing idempotent: re-running the check
   never opens a second issue for the same incident.
```

That two-step gate is a smaller version of the [SLO burn-rate rule](#slos-and-alerts-that-page-for-a-reason)
just ahead — a long window and a short window both have to agree before anything
pages — just implemented in twenty lines against the GitHub issues API instead of
Prometheus rules. One labelled issue per incident, updated while it lasts and closed
on recovery, either way: alert dedup with no alerting system.

The two layers are complementary, not redundant: the inside view explains a problem,
the outside view is the only one that can *notice* a total outage.

**Its honest limit is bigger than "best-effort" suggests, so here is the measured
number.** The workflow asks for every 15 minutes. Across the last twelve real runs the
gaps were 50–71 minutes, with one of **213**. Hosted cron is best-effort under load, so
the achieved cadence is roughly hourly and a total outage could sit unnoticed for
longer than a working morning. Nothing is misconfigured — that is simply what a free
scheduler on someone else's infrastructure buys. It is written down because "we have
external monitoring" and "we would know within 15 minutes" are different claims, and
only the first is true. A second free-tier prober is the documented next step: one
best-effort scheduler is a single point of *silence*.

> **Say it in one line:** monitoring that shares a failure domain with the thing it
> monitors will go quiet exactly when you need it loudest.

## SLOs, and alerts that page for a reason

> **In plain English:** decide up front how much failure is acceptable in a month —
> that allowance is your *error budget*. Then don't alert when errors happen; alert
> when the budget is draining fast enough to run out early. A blip shouldn't wake
> anyone. A slow leak shouldn't be slept through.

Three objectives sit on those signals: availability, request success, and latency. The
targets are deliberately modest — this is a single-replica service, and an objective
you cannot miss teaches nothing.

An **SLO** (service level objective) is a target like "99.5% of requests succeed over
28 days." The 0.5% left over is the error budget, and it is a real, spendable number.
The three actual targets here are availability at 99.5%, request success at 99.5%,
and latency at 99% served under 500ms — all measured over a rolling 28 days:

```
   99.5% over 28 days  =  40,320 minutes x 0.5%  =  ~202 minutes of budget
                                                     (about 3h 20m)

   "burn rate" = how fast you are spending it, relative to the pace that
   would exactly use up the period's budget over exactly the period.

     1x    -> you finish the 28 days right on target
     6x    -> the whole budget is gone in ~4.7 days   -> TICKET
     14.4x -> the whole budget is gone in ~2 days     -> PAGE

   The alert thresholds are just burn_rate x (1 - SLO):
     14.4 x 0.005 = 0.072   (7.2% of requests failing)
      6   x 0.005 = 0.03    (3% of requests failing)
```

Alerting uses the **multi-window, multi-burn-rate** shape rather than a bare
threshold. Two burn rates, and each one needs a long window and a short window to
agree before it fires:

```
   How fast is the month's error budget draining right now?

   FAST BURN (14.4x)                        SLOW BURN (6x)
   1h window over threshold                 6h window over threshold
            AND                                      AND
   5m window over threshold                 30m window over threshold
            |                                        |
       sustained 2m                            sustained 15m
            |                                        |
            v                                        v
         [ PAGE ]  --------- inhibits --------> [ TICKET ]
      wake someone now                       queue it for work hours

   WHY TWO WINDOWS: the long one says "this is real, not a blip"; the short one
   says "it is STILL happening" -- so a recovered incident stops paging on its
   own, and a slow bleed is never simply waited out.
```

An inhibition rule stops a page from also delivering its own slower ticket: one
incident, one stream. Every alert names a *symptom* — users are getting errors — never
a cause like high CPU, and every one carries a runbook link, because the question at
3 a.m. is never "what's the number" but "what do I do."

A chat-bot receiver is authored in full, but it is applied only when *both* its bot token
and its chat id are configured — and the chat id is unset, so the condition fails and
alerts still fall through to the default receiver that notifies nobody. It is one
variable away from real delivery. That distinction is worth stating precisely rather
than rounding up to "delivery is wired": an untested notification path deserves the same
skepticism as an untested backup, and an *unactivated* one deserves more.

> **Say it in one line:** don't alert on errors, alert on how fast they're eating the
> budget — and make a long and a short window agree first.

## Testing across the boundaries that actually break

> **In plain English:** a fake version of a database agrees with whatever you assumed
> when you wrote it — including your mistakes. A real one doesn't. So anything that
> crosses a boundary gets tested against the real thing at least once, even though
> that's slower and more annoying to set up.

Test strategy belongs in an architecture document because it is a claim about
*boundaries*, and boundaries are architecture. The rule here is one sentence: anything
that crosses a **socket, a goroutine, a timer, or a database** gets at least one test
against the real thing, not only an in-process fake.

```
                 /\        few, slow, highest-fidelity
                /  \
               / E2E\      DELIBERATE GAP: no browser-driven test
              /------\
             /  real  \    real WebSocket sockets, a full round played
            / boundary \   end to end; real Postgres in CI
           /------------\
          /  concurrency \ real goroutines and timers, all under the
         /   & timers     \ race detector
        /------------------\
       /    pure unit       \ the rules engine and pure client libs --
      /  (table-driven)      \ many, instant
     /------------------------\
```

**That rule is policy because it paid.** Two genuine bugs — a missing initial
broadcast, and a duplicate round-over carrying zeroed results — were **invisible to
in-process fakes** and surfaced only against a real connection. The reason is worth
internalising:

> A fake re-implements the boundary, and in doing so it re-implements your
> assumptions — so it agrees with the bug. The real socket doesn't.

Two supporting decisions keep this from being merely aspirational.

**Optional real dependencies skip rather than fail.** No Postgres available? Those
tests skip cleanly instead of going red. That is what lets "no setup needed locally"
and "actually tested in CI" both be true at once — the usual trap is picking one and
quietly abandoning the other.

**The gap at the top is drawn empty on purpose.** The browser client has no automated
tests; it is verified by hand. Drawing the pyramid with a hole in it is more useful
than drawing a pyramid that implies coverage nobody has.

### The load test that hasn't run, and why that isn't laziness

The k6 scenario is written — each virtual user plays a full hand over a real socket —
and it has not been executed, which is why the autoscaler's CPU target is still a
placeholder. There is a specific reason it isn't a free afternoon:

```
   one load generator = ONE source IP
                            |
                            v
   the server rate-limits PER IP (8 concurrent, 0.5/sec)
                            |
                            v
   past a few dozen virtual users you stop measuring the game
   and start measuring your own defences

   A useful run needs several source addresses, or a test deployment
   with the limits lifted -- which is a piece of work, not a command.
```

That is a genuinely instructive collision: the [edge hardening](#hardening-the-public-edge)
that protects the server in production is the same thing that makes load-testing it
hard. Both are correct; they just have to be reconciled deliberately rather than
discovered mid-test.

> **Say it in one line:** a fake agrees with your assumptions and a real socket
> doesn't — so test every boundary against the real thing, and draw the gaps you
> haven't covered.

## Delivery: pipeline-pushed today, GitOps authored next

> **In plain English:** right now the build robot logs into the cluster and pushes
> the new version. That works, but it means the only record of *what is actually
> running* lives in the robot's memory, not in your repository. The next step flips
> it: the robot writes the version into git, and something inside the cluster reads
> git and makes reality match.

Today CI pushes. It pins the freshly built image by immutable digest, applies the
overlay, and blocks on the rollout reporting healthy. That works, but it has one real
gap: the deployed tag is written *in the runner*, so **git never shows what's actually
live**.

```
   TODAY -- pipeline pushes                 STAGED -- git is the source of truth
   ------------------------                 ---------------------------------
   commit                                   commit
     |                                        |
     v                                        v
   CI builds, pins the image by              CI builds, pins the digest, and
   immutable digest                          WRITES that tag back INTO git
     |                                        |
     v                                        v
   CI applies to the cluster                in-cluster controller syncs
   (the runner holds the credential)        git -> cluster, and detects drift
     |                                        |
     v                                        v
   blocks on the rollout reporting          canary shifts traffic gradually,
   Ready, else the build goes red           gated on the SAME metrics that page
     |                                        |
     v                                        v
   git never shows what is LIVE             git IS what is live; history is
   (the tag exists only in the runner)      the audit log, revert = rollback
```

GitOps inverts the flow: CI writes the tag into git, and Argo CD in the cluster syncs
the cluster to match, with Argo Rollouts running the canary on top. A **canary** ships
a new version to a small slice of traffic first and watches its metrics before
committing to the rest. The repository becomes the single source of truth, with drift
detection and an auditable history — and CI stops needing cluster credentials at all,
because pushing a commit is the whole job. That last consequence is a security win as
much as a workflow one: the credential that no longer exists cannot leak.

The canary gates on the *same* recording rules the alerts use. The metric that pages
you is the metric that decides whether a rollout continues. Two details make that
honest rather than decorative:

- a low-traffic canary often samples *no* requests, so "no data" has to count as a
  pass — otherwise every quiet deploy aborts on an empty query;
- these indicators are service-wide rather than canary-scoped, which catches a gross
  regression but could let a subtle one hide behind healthy stable pods.

Both are documented in the file rather than discovered later.

It is **authored, schema-validated, and deliberately not switched on**: a canary needs
a second pod the one baseline node can't hold — the same capacity gate the
observability and autoscaling work both ran into.

**And the blocker is smaller than it looks, which is worth saying rather than
leaving "staged" to sound like "stuck".** The GitOps overlay is the production
overlay plus a Rollout, an analysis template, and one autoscaler patch — and every
one of those is a *progressive-delivery* object, not a GitOps one. The GitOps
controller is only the thing that applies them. So the existing pipeline could
apply that overlay directly and get the canary with **one** controller instead of
five, changing no manifest at all.

That splits a problem that looked singular. The real constraint was never the
GitOps controller's footprint; it is that a canary needs a second application pod
this node cannot schedule. Running progressive delivery on its own doesn't remove
that constraint — it reduces the question from "can this node hold four platform
pods *and* a second app pod?" to "can it hold one more app pod?", which is the
cheaper question and the one worth answering first. Git-as-source-of-truth is a
genuinely separate win, and it is the half that needs the four pods.

> **Say it in one line:** GitOps is a source-of-truth move and a canary is a safety
> move — so when "it's blocked on capacity" comes up, check which *half* is blocked.

## Hardening: the supply chain, the pod, the network

> **In plain English:** three different attackers, three different defences. One
> poisons a library you depend on. One breaks into your running container and tries
> to spread. One is already inside your network and moves sideways. Nothing here
> defends against all three.

### The supply chain

Trivy scans every image before the push. The game's two images are then signed by
**keyless cosign** — the build workflow's own OIDC identity is the signer, so there is
no long-lived key to store, rotate, or leak, and the signature is recorded in
Sigstore's public transparency log.
A Syft-generated **SBOM** — a software bill of materials, the full ingredient list of
every library inside the image — is attached as an attestation to the same digest,
which turns "are we affected by this new CVE?" from a rebuild-and-grep into a query. A
second, report-only config scan flags build-file smells without blocking a deploy the
day a scanner ships a new rule.

Two limits worth stating rather than rounding away. The signing and SBOM steps live in
the game's workflow only — the utilities toolbox image is built, scanned, and
rollout-gated the same way, but ships **unsigned**: two of the three published images
carry a signature and an attestation, not all three. And nothing *verifies* a signature
at deploy time — there is no admission policy that refuses an unsigned image. The
signature proves provenance to anyone who runs `cosign verify`; it does not yet gate
what the cluster will run.

```
   source
     |
     v
   build image (cross-compiled for the target CPU architecture)
     |
     v
   SCAN for critical/high CVEs        <-- HARD GATE: a finding fails the build,
     |                                    and it runs BEFORE the push, so what
     | (only if clean)                    was scanned is exactly what ships
     v
   push to registry ---> digest (an immutable content hash, not a tag)
                             |
              +--------------+---------------+
              v                              v
      keyless SIGNATURE                 SBOM ATTESTATION
      signed by the build workflow's    the image's full dependency
      own identity -- no key to store,  inventory, attached to the
      rotate, or leak; recorded in a    same digest: "are we affected
      public transparency log           by this CVE?" becomes a query

   Both bind to the DIGEST. A tag can be moved to different bytes; a digest
   cannot -- so a signature on a tag would prove nothing about what runs.
```

That tag-versus-digest point deserves one more beat, because it is where most
supply-chain reasoning quietly fails. A tag like `:latest` is a *label* someone can
repoint at different bytes tomorrow. A digest is a hash *of* the bytes: change one
byte and it is a different digest. Signing a tag would certify a name, not a payload.

The gate earned its place on a change that had nothing to do with security. Adding a
tracing library pulled in a transitive gRPC dependency carrying a HIGH advisory, and
the scan turned "shipped a known-vulnerable image" into "one-line version bump before
it ever deployed." Adding one library means inheriting the vulnerabilities of its
entire dependency tree.

### The pod

The images were already hardened at build time — a distroless, non-root, shell-less
server image and an unprivileged nginx for the frontends — so the pod's security
context mostly *declares* what is already true. **Distroless** means the image
contains the compiled binary and nothing else: no shell, no package manager, no
utilities for an intruder to pick up and use.

Declaring it is the point: an unenforced property is a promise, and the declaration is
what lets the platform enforce it and a scanner verify it.

Admission-time standards are enforced at **baseline** rather than the stricter
`restricted` level, because the database operator shares the namespace and
legitimately needs more privilege than `restricted` allows. The stricter level runs
alongside as warn-and-audit, so the gap stays visible in the logs instead of being
assumed away.

### The network

Default-deny policies are written but **staged, not applied**, with both traps
documented:

- They must explicitly allow **DNS**. It is the rule everyone forgets, and everything
  breaks confusingly without it — pods can reach each other by IP but nothing can
  resolve a name, so failures look like random timeouts rather than a policy problem.
- They are inert unless the cluster's network plugin actually enforces them. An
  unenforced policy is worse than none, because it reads as protection while providing
  zero. Shipping it "live" would produce a screenshot, not a control.

> **Say it in one line:** sign the digest, declare what the image already is, and
> never apply a default-deny you can't test.

## Operating a cluster you can't reach directly

> **In plain English:** for most of this build, the cluster's control API simply
> would not answer from the workstation — a network path upstream, nothing that could
> be fixed locally. But the CI runners reached it fine. So instead of fighting the
> road, everything moved to the road that already worked.

Rather than fight it, cluster operations were built around what *did* work:

```
   workstation ----X----> [ Kubernetes API ]   blocked upstream, not fixable here
                            ^   ^   ^
                            |   |   |
       manually-dispatched  |   |   |  read-only diagnostics workflow:
       privileged installs  |   |   |  dumps certificate, ingress, pod,
       (rare, human-gated) -+   |   +- and event state from a runner that
                                |      CAN reach the API
                                |
              the provider's browser shell, used exactly once for the
              permission bootstrap that had to come before everything else
```

Three paths, each matched to a job: a manually-dispatched workflow for privileged
installs, a separate read-only diagnostics workflow for seeing what's wrong, and the
provider's browser shell for the one-time permission bootstrap.

### Every operation is a workflow, and the split is the point

Because nothing could be run from a laptop, *every* operation had to become a
pipeline — which forced a division that most solo projects never bother with:

| Workflow | Trigger | Identity |
| --- | --- | --- |
| **Build and deploy** (per product) | every push | the boxed deploy identity |
| **Platform add-ons** | manual only | `cluster-admin` |
| **Diagnostics** | manual, **read-only** | read scoped |
| **On-demand database backup** | manual | platform |
| **Chat login activation** | manual | platform |
| **Webhook registration** | manual | platform |
| **External uptime probe** | asks for 15 min, gets ~60 | none — it's outside the cluster |
| **Docs CI** | every push | none |

Two of those are worth a sentence.

**Diagnostics is read-only on purpose.** When something is broken at 2 a.m., the
temptation is to reach for the admin credential because it definitely works. A
separate workflow that can *only* dump certificate, ingress, pod, and event state
means the common case — "what is actually wrong?" — never requires reaching for the
credential that could make things worse.

**Docs CI checks that documentation didn't rot.** The learning guide cites real source
files by name *and line number*, which is the most useful thing about it and the most
fragile — editing a source file tells you nothing about the six documents that pointed
at line 47. So a CI job verifies the citations still resolve. Documentation that
silently goes stale is worse than none, because it is confidently wrong, and the only
durable fix is to make staleness fail a build.

The transferable lesson is broader than Kubernetes. When a network problem blocks a
privileged one-time action, look for a *different path to the same identity* before
you debug the road you're standing on.

> **Say it in one line:** an inconvenience became a forcing function — everything runs
> through pipelines, which is how a team would have done it anyway.

---

## Every one-liner in one place

The whole document, compressed. If you remember nothing else, remember these.

**Orientation**

1. It is a small system operated like a large one — and the single free node is the
   constraint that made every trade-off honest.
2. One platform, one TLS story, one pipeline — a second vendor is a second thing to
   operate, and it buys nothing for a static site.
3. A missing dependency should cost you one feature, not the whole start-up — that is
   what keeps "clone and run" a single command.

**Getting a request in**

4. The certificate needs the ingress, and the ingress needs the balancer — so you
   install upward, and requests travel back down the same path.
5. If a node doesn't need a path *from* the internet, it doesn't get one — and the
   rules that break this are invisible from inside the cluster.
6. The credential you use most should reach the least, and the one that lives in the
   cluster forever should reach least of all.
7. One origin means the cookie is first-party and CORS is a development concern, not a
   production one.

**Surviving the open internet**

8. A socket is a resource an attacker gets to spend — so cap how many, how fast, how
   big, and how long.

**Who the player is**

9. Retire the endpoint, not just the UI — and when a flow changes shape, check that
   the page it ends on still tells the truth.
10. A grouping key should be a preference with fallbacks, not a partition — because
    never seating anyone is worse than seating them with strangers.

**The game itself**

11. The strongest anti-cheat is data the client never received — so send each seat only
    its own hand, and re-validate every move anyway.
12. Derive the server URL from the page's own origin and one built image runs anywhere
    — and a best-effort feature that degrades to silence never needs an error path.

**Keeping it up**

13. Rehearse on staging, because production rate limits are per-domain, per-week, and
    there's no appeal.
14. A snapshot restores to one instant and the log restores to any instant — but an
    untested backup is still only a hypothesis.
15. Staying up beats staying consistent-or-dead — when the failure is recoverable at
    human speed and the alternative is a permanent crash loop.
16. A pod autoscaler reshuffles capacity and only nodes create it — so the `max` on the
    node pool is the real cost control.

**Knowing what it's doing**

17. Monitoring that shares a failure domain with the thing it monitors will go quiet
    exactly when you need it loudest.
18. Don't alert on errors, alert on how fast they're eating the budget — and make a
    long and a short window agree first.
19. A fake agrees with your assumptions and a real socket doesn't — so test every
    boundary against the real thing, and draw the gaps you haven't covered.

**Getting changes out**

20. GitOps is a source-of-truth move and a canary is a safety move — so when "it's
    blocked on capacity" comes up, check which *half* is blocked.
21. Sign the digest, declare what the image already is, and never apply a default-deny
    you can't test.
22. An inconvenience became a forcing function — everything runs through pipelines,
    which is how a team would have done it anyway.

---

*Companion documents: [kubernetes-primer.md](kubernetes-primer.md) for the vocabulary
from zero plus the glossary, [README](../README.md) for the overview and the
live/staged/unproven table, and [engineering-highlights.md](engineering-highlights.md)
for the production bugs in depth and the explicit list of what still isn't proven.*
