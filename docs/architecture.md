# Architecture

Every section here ends with one line you could say out loud in an interview. That
line is the point; the paragraphs above it are the evidence.

## The layers a request crosses

A Deployment and a Service get a container running and reachable *inside* a cluster.
Turning that into something a person can type a URL for takes three more layers, and
they have to come up in order. Each one is reached *through* the one below it.

1. **A public load balancer** accepts traffic from the internet. On managed
   Kubernetes you don't create it directly. You install an ingress controller whose
   Service is `type: LoadBalancer`, and the cloud provider's controller provisions a
   real balancer to match. Its shape is pinned to the free-tier allowance, because a
   `LoadBalancer` Service is one of the few Kubernetes objects that can quietly bill
   real money — and the default shape is not free.
2. **An ingress controller** routes that one public entry point to many internal
   services by hostname and path, and terminates TLS.
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
     |  DNS resolves the hostname to the balancer's public IP
     v
   [ LOAD BALANCER ]  public entry point, pinned to the free-tier shape
     |
     v
   [ INGRESS ]  TLS ENDS HERE -- everything below is plain HTTP inside
     |          the cluster (which is why the cookie bug below existed)
     |
     +-- path /ws, /auth ---> [ SERVICE ] --> [ POD ] the Go server
     |                             ^              |
     +-- path /  ------------> [ SERVICE ] --> [ POD ] the frontend's nginx
                                   |              |
              a pod joins its Service's backend   v
              list ONLY while its readiness    [ POSTGRES ]  operator-run,
              probe passes                                   password minted
                                                             into the cluster
```

Read it downward and it's a request. Read it upward and it's the install order.
Nothing in the middle is wired by name — a Service finds its pods by label match, so a
replaced pod with a brand-new IP rejoins with zero configuration.

> **Say it in one line:** the certificate needs the ingress, and the ingress needs the
> balancer — so you install upward, and requests travel back down the same path.

## Network topology

The cluster lives in its own virtual network, split into three subnets rather than
one. That's the standard production layout: each subnet's routing and security rules
can differ without one loosening another.

```
  Virtual Cloud Network
  |
  +-- public endpoint subnet (/28)   Kubernetes API server
  |
  +-- private worker subnet          nodes -- never directly internet-facing;
  |                                   outbound via a NAT gateway only
  |
  +-- public load-balancer subnet    the one public entry point
```

Workers are private. They reach the internet outbound through a NAT gateway for image
pulls and updates, but nothing on the internet can reach them directly. The only
public ingress is the balancer, in its own subnet.

Security is enforced with resource-attached security groups rather than subnet-wide
lists, so a rule follows the resource it protects. Two of those rules are the classic
silent failure. The balancer forwards to the ingress controller on a node-port range
and health-checks it on another port. If the worker-side rules for those are missing,
the balancer's backends stay **permanently unhealthy while every Kubernetes object
looks completely correct**. Nothing in the cluster's own output points at the missing
rule.

> **Say it in one line:** if a node doesn't need a path *from* the internet, it
> doesn't get one — and the rules that break this are invisible from inside the
> cluster.

## Identity: three rings and three keys

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

The ordering is the argument. The credential exercised most often — the one that runs
on every single push — is the one most likely to leak, so it holds the least power. It
may `get`, `list`, `watch`, `create`, `update`, and `patch` a few resource kinds in one
namespace. It cannot delete anything. It cannot read secrets. It deliberately cannot
touch roles or role bindings, so it can never widen itself.

The powerful identity is the opposite: `cluster-admin`, wired only into a workflow a
human triggers by hand, never onto a commit. And the third — the backup writer — is
the one that must live permanently *inside* the cluster as a long-lived secret, so its
policy grants exactly two verbs against exactly one bucket. It cannot create buckets,
read the database, or see anything else.

Every one of those keys is generated *inside* Terraform. None was ever clicked in a
console and pasted by hand.

> **Say it in one line:** the credential you use most should reach the least, and the
> one that lives in the cluster forever should reach least of all.

## Same-origin hosting

The game's static frontend and its API are served from the **same** hostname. The
ingress routes the WebSocket and auth paths to the Go server and everything else to
the frontend's nginx. Three payoffs, all from one decision:

- the frontend derives its WebSocket URL from `window.location`, so one built image
  runs unchanged on any hostname;
- the login-session cookie is first-party;
- CORS never engages in production at all.

> **Say it in one line:** one origin means the cookie is first-party and CORS is a
> development concern, not a production one.

## How a player proves who they are

There are no passwords in this system. Not hashed ones — none. Two different doors
lead to the same session cookie, and neither involves the user choosing a secret.

```
   DOOR 1 -- one-time link                DOOR 2 -- signed mini-app payload
   -----------------------                --------------------------------
   ask for a link                         open the game inside the chat app
        |                                      |
        v                                      v
   server mints a token with a            the platform signs a payload with
   CRYPTO-grade random source             a secret only it and the server know
        |                                      |
        v                                      v
   one transaction:                       verify the signature in constant time,
     consume the token AND                 reject anything older than the
     create-or-find the account            freshness window (no replay)
        |                                      |
        +------------------+-------------------+
                           v
              ONE helper sets ONE session cookie
              HttpOnly · Secure · SameSite=Lax
                           |
                           v
              the browser attaches it to everything after,
              including the WebSocket upgrade -- so the game
              protocol never had to learn about auth at all
```

Three details carry most of the value. The token uses a cryptographic random source,
not the same generator that shuffles the cards — if guessing it lets someone in, it
gets `crypto/rand`; if guessing it only spoils a game, the fast one is fine. Consuming
the token and creating the account happen in **one transaction**, because a crash
between them would mark a link used while creating no account: a login that silently
does nothing, forever. And both doors call the *same* cookie helper, so the security
flags can never drift apart between them.

In production only the second door is open. Turning it on doesn't merely hide the
email form — it **unregisters those routes entirely**, because a route left registered
still accepts a direct POST and would still write an unverified email address into the
database.

> **Say it in one line:** retire the endpoint, not just the UI — a hidden form is
> still an open door.

## Playing with the people you came with

The game can be launched from inside a group conversation, which changes what
matchmaking means: not "queue of strangers" but "the people from *this* chat —
preferably." That last word carries the design.

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

The token isn't secret — the player was in the chat it names — it only has to be
**unforgeable**, which is exactly what a MAC buys. It's domain-separated from the
other signatures that share the same key, because one secret signing three different
things is how cross-protocol replays happen. And with no key configured the verifier
is absent entirely, so everyone lands in the open queue rather than the server
trusting a claim it cannot check.

Then the key **sorts players without trapping them**, in three tiers:

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

> **Say it in one line:** a grouping key should be a preference with fallbacks, not a
> partition — because never seating anyone is worse than seating them with strangers.

## TLS issuance, and why there are two issuers

The certificate manager is configured with **two** issuers against the same public
authority: a staging one and a production one. This isn't caution for its own sake.
Production rate limits are counted per registered domain, per week, with no appeal.
Point a misconfigured ingress at production, or let the challenge fire before DNS
resolves, and the real certificate is locked out for days. The staging issuer has
loose limits and an untrusted root — useless to serve, perfect for proving the path.

The challenge type is HTTP-01 rather than DNS-01, because HTTP-01 needs nothing but a
public port and a hostname that already resolves. DNS-01 would mean handing the
certificate manager API credentials for the DNS provider, and it buys only wildcard
certificates in return — which this platform doesn't need.

> **Say it in one line:** rehearse on staging, because production rate limits are
> per-domain, per-week, and there's no appeal.

## Data and its durability

PostgreSQL runs *inside* the cluster under the CloudNativePG operator, not as a
hand-written StatefulSet. The distinction matters. A StatefulSet plus a volume claim gets a
database *process* running. It gives you none of what makes a database survivable:
consistent backups, point-in-time recovery, credential rotation, safe minor upgrades.
The operator encodes all of that, and it mints the application's database password
directly into the cluster — so that credential never appears in git or in a command
anyone typed.

Backups are continuous write-ahead-log archiving to object storage, not a nightly
snapshot. The difference is the recovery window:

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

**And here is the honest part.** The archiving is configured and an on-demand backup
was verified to have written real objects to the bucket. **No restore has ever been
performed from them.** By this project's own standard that makes recovery a
hypothesis, and rehearsing it — with a defined recovery point and recovery time
objective — is named as the next phase rather than quietly implied to be done.

> **Say it in one line:** a snapshot restores to one instant and the log restores to
> any instant — but an untested backup is still only a hypothesis.

## Scaling, safely and within budget

Elasticity happens at three levels, and conflating them is the classic mistake:

```
  Pod    -- a HorizontalPodAutoscaler raises the game server 1 -> 2 on load
  Node   -- a fixed baseline node runs everything stateful (DB, server, ingress)
  Node   -- a scale-to-zero BURST pool absorbs overflow the baseline can't hold
```

The load-bearing insight: **an autoscaler for pods cannot create capacity.** If it
scales the server to two pods and the second doesn't fit on the one baseline node,
that pod just sits `Pending`. Only *node* autoscaling adds real capacity — so both
layers exist, and the node layer is bounded hard.

**Bounded, because the guardrail assumption broke.** Node autoscaling was originally
scoped out on the theory that a fixed node pool was a *physical* ceiling. A live check
of the account killed that: far more quota than a pure free tier, and no physical
guardrail at all. Unbounded autoscaling would have provisioned billable nodes with
nothing to stop it. The fix wasn't "never autoscale." It was **autoscale with a hard,
documented ceiling** — a second pool that is tainted, scales to zero, and is capped at
a small `max` that *is* the money limit. The autoscaler physically cannot exceed that
node count no matter what a runaway workload requests.

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
selector would pin it to billable nodes permanently.

**Two honest caveats, both written into the manifests.** The pod autoscaler's CPU
target is a **committed placeholder**, to be right-sized from a k6 WebSocket load test
— each virtual user plays a full hand — that has been written but not yet run. And CPU
is a weak signal for this workload anyway: WebSocket connections are long-lived and
mostly idle, so CPU barely moves with connection count. The right
signal is active rooms or live connections per pod, and the upgrade path to it is
staged in the file.

**Elasticity that's safe for a stateful game.** Rooms are in-memory and pod-local, so
retiring a pod abruptly would drop every hand on it — and Kubernetes sends `SIGTERM`
before killing a pod on *every* scale-down, node drain, and rollout.

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

One more subtlety worth knowing: a PodDisruptionBudget guards *voluntary* disruptions
like a node drain — it does **not** gate autoscaler scale-down, which deletes pods
directly. Protecting in-flight hands from scale-down is a different mechanism, which
is exactly why the drain above exists. And the budget is written as "at most one
unavailable" rather than the more obvious "at least one available," because at a
one-replica floor the obvious version deadlocks every drain forever.

> **Say it in one line:** a pod autoscaler reshuffles capacity and only nodes create
> it — so the `max` on the node pool is the real cost control.

## Seeing inside it: metrics, logs, and traces

A system you can't see inside is one you operate by guessing. All three pillars run
in-cluster, and each answers a different question.

The stack is Prometheus and Grafana for metrics, Loki with a Grafana Alloy agent for
logs, and OpenTelemetry into Tempo for traces — each installed through its own official
chart, then trimmed to fit one small node.

**Metrics** are the RED signals — rate, errors, duration — plus game-domain gauges no
generic exporter could know to collect: live connections, hands in progress,
human-versus-bot seats, and join attempts split by outcome. Those domain metrics
answer "is anyone actually *playing*?", which uptime never does.

**Logs** are structured JSON at the source, shipped by a collector agent, so a field
is a field rather than a regex against a sentence. Getting there cost exactly one line
of code: Go's `slog.SetDefault` also redirects the standard `log` package through the
JSON handler, so 35 existing call sites became structured records with no edits.

**Traces** are OpenTelemetry spans that follow a request through the server and into
its database calls. The payoff is correlation — from a slow trace, jump straight to
the log lines that span emitted. Honest scope: this is one Go binary talking to
Postgres, so most traces are shallow. The deliverable is the *shape* — SDK, exporter,
store, correlated in one pane — which is what scales later.

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
expiry from outside that blast radius, and files an issue when they fail. The two
layers are complementary, not redundant: the inside view explains a problem, the
outside view is the only one that can *notice* a total outage. Its honest limit is
written into the file — a hosted scheduler's floor is coarse and best-effort, so this
is a safety net, not a sub-minute pager.

> **Say it in one line:** monitoring that shares a failure domain with the thing it
> monitors will go quiet exactly when you need it loudest.

## SLOs, and alerts that page for a reason

Three objectives sit on those signals: availability, request success, and latency. The
targets are deliberately modest — this is a single-replica service, and an objective
you cannot miss teaches nothing.

Alerting uses the **multi-window, multi-burn-rate** shape rather than a bare
threshold.

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

Delivery is wired to a chat bot rather than left on the default receiver that notifies
nobody. It has never fired in anger, so it is listed as unproven rather than claimed
as working. An untested notification path deserves exactly the same skepticism as an
untested backup.

> **Say it in one line:** don't alert on errors, alert on how fast they're eating the
> budget — and make a long and a short window agree first.

## Delivery: pipeline-pushed today, GitOps authored next

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
the cluster to match, with Argo Rollouts running the canary on top. The repository
becomes the single source of truth, with drift detection and an auditable history —
and CI stops needing cluster credentials at all, because pushing a commit is the whole
job.

The canary on top gates on the *same* recording rules the alerts use. The metric that
pages you is the metric that decides whether a rollout continues. Two details make
that honest rather than decorative: a low-traffic canary often samples *no* requests,
so "no data" has to count as a pass or every quiet deploy aborts; and these indicators
are service-wide rather than canary-scoped, which catches a gross regression but could
let a subtle one hide behind healthy stable pods. Both are documented in the file
rather than discovered later.

It is **authored, schema-validated, and deliberately not switched on**: a canary needs
a second pod the one baseline node can't hold — the same capacity gate the
observability and autoscaling work both ran into.

> **Say it in one line:** GitOps is a source-of-truth move, and a canary is only as
> honest as its query — so handle no-data, and know what your indicator can't see.

## Hardening: the supply chain, the pod, the network

Three layers, each with a different threat in mind.

**The supply chain.** Trivy scans every image before the push. Then **keyless cosign**
signs it — the build workflow's own OIDC identity is the signer, so there is no
long-lived key to store, rotate, or leak, and the signature is recorded in Sigstore's
public transparency log. A Syft-generated SBOM is attached as an attestation to the
same digest, which turns "are we affected by this new CVE?" from a rebuild-and-grep
into a query. A second, report-only config scan flags build-file smells without
blocking a deploy the day a scanner ships a new rule.

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

That gate earned its place on a change that had nothing to do with security. Adding a
tracing library pulled in a transitive gRPC dependency carrying a HIGH advisory, and
the scan turned "shipped a known-vulnerable image" into "one-line version bump before
it ever deployed."

**The pod.** The images were already hardened at build time — a distroless, non-root,
shell-less server image and an unprivileged nginx for the frontends — so the pod's
security context mostly *declares* what is already true. Declaring it is the point: an
unenforced property is a promise, and the declaration is what lets the platform
enforce it and a scanner verify it. Admission-time standards are enforced at
**baseline** rather than the stricter level, because the database operator shares the
namespace and legitimately needs more; the stricter level runs alongside as
warn-and-audit, so the gap stays visible instead of being assumed away.

**The network.** Default-deny policies are written but **staged, not applied**, with
both traps documented. They must explicitly allow DNS — the rule everyone forgets, and
everything breaks confusingly without it. And they are inert unless the cluster's
network plugin actually enforces them, which makes an unenforced policy worse than
none, because it reads as protection.

> **Say it in one line:** sign the digest, declare what the image already is, and
> never apply a default-deny you can't test.

## Operating a cluster you can't reach directly

For most of this build, direct API access to the cluster was blocked by an upstream
network path that couldn't be fixed locally — while CI runners reached the same
endpoint fine. Rather than fight it, cluster operations were built around what *did*
work: a manually-dispatched workflow for privileged installs, a separate read-only
diagnostics workflow that dumps certificate, ingress, pod, and event state from a
runner that can reach the API, and the provider's browser shell for the one-time
permission bootstrap.

The transferable lesson is broader than Kubernetes. When a network problem blocks a
privileged one-time action, look for a *different path to the same identity* before
you debug the road you're standing on.

> **Say it in one line:** an inconvenience became a forcing function — everything runs
> through pipelines, which is how a team would have done it anyway.
