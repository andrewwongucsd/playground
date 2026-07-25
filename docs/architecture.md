# Architecture

## The layers a request crosses

A Deployment and a Service get a container running and reachable *inside* a
cluster. Turning that into something a person can type a URL for takes three
more layers, and they have to come up in order — each depends on the one below
it.

1. **A public load balancer** accepts traffic from the internet. On a managed
   Kubernetes service you don't create it directly; you install an ingress
   controller whose Service is `type: LoadBalancer`, and the cloud provider's
   controller provisions a real balancer to match. Its shape is pinned
   explicitly to the free-tier allowance — a `LoadBalancer` Service is one of
   the few Kubernetes objects that can quietly bill real money, and the
   default shape is not free.

2. **An ingress controller** routes that one public entry point to many
   internal services by hostname and path, and terminates TLS.

3. **Certificates and DNS.** A certificate manager watches ingress objects and
   runs the ACME challenge automatically — request a certificate, serve the
   HTTP-01 challenge *through the ingress*, swap in the real certificate once
   the authority validates. DNS points the hostnames at the load balancer's
   public IP.

## Network topology

The cluster lives in its own virtual network, split into three subnets rather
than one — the standard production layout, so each subnet's routing and
security rules can differ without one loosening another:

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

Workers are private: they reach the internet outbound through a NAT gateway
for image pulls and updates, but nothing on the internet can reach them
directly. The only public ingress is the load balancer, in its own subnet.

Security is enforced with resource-attached security groups rather than
subnet-wide lists, so a rule follows the resource it protects. Two of these
rules are the classic silent failure: the load balancer forwards to the
ingress controller on a node-port range and health-checks it on another port,
and if the worker-side rules for those are missing, the balancer's backends
stay **permanently unhealthy while every Kubernetes object looks completely
correct** — there is nothing in the cluster's own output pointing at the
missing rule.

## Same-origin hosting

The game's static frontend and its API are served from the **same** hostname,
with the ingress routing the WebSocket and auth paths to the Go server and
everything else to the frontend's nginx. This is a deliberate choice with
three payoffs:

- the frontend derives its WebSocket URL from `window.location`, so one built
  image runs unchanged on any hostname;
- the login-session cookie is a first-party cookie;
- CORS never engages in production at all.

## TLS issuance, and why there are two issuers

The certificate manager is configured with **two** issuers against the same
public certificate authority: a staging one and a production one. This is not
caution for its own sake — the authority's production rate limits are counted
per registered domain, per week, with no appeal. Pointing a misconfigured
ingress at production, or having DNS not resolve yet when the challenge fires,
can lock the real certificate out for days. The staging issuer (loose limits,
untrusted root) is where the whole HTTP-01 path is proven before a single
production issuance is spent.

HTTP-01 rather than DNS-01 as the challenge type: HTTP-01 needs nothing but a
public port and a hostname that already resolves. DNS-01 would require handing
the certificate manager API credentials for the DNS provider, and buys only
wildcard certificates in return — which this platform doesn't need.

## Data and its durability

PostgreSQL runs *inside* the cluster, managed by an operator rather than as a
hand-written StatefulSet. The distinction matters: a StatefulSet plus a
volume claim gets a database *process* running, but none of what makes a
database survivable — consistent backups, point-in-time recovery, credential
rotation, safe minor-version upgrades. The operator encodes all of that.

Backups go to S3-compatible object storage via continuous write-ahead-log
archiving, not just a nightly snapshot. The difference is the recovery
window: a snapshot lets you restore to the instant it was taken; continuous
WAL archiving lets you restore to *any moment* between base backups. That is
the difference between losing a day and losing a few minutes.

## Scaling, safely and within budget

Elasticity happens at three levels, and conflating them is the classic mistake:

```
  Pod    -- a HorizontalPodAutoscaler raises the game server 1 -> 2 on load
  Node   -- a fixed baseline node runs everything stateful (DB, server, ingress)
  Node   -- a scale-to-zero BURST pool absorbs overflow the baseline can't hold
```

The load-bearing insight: **an autoscaler for pods cannot create capacity.** If it
scales the game server to two pods but the second doesn't fit on the one baseline
node, that pod just sits `Pending`. Only *node* autoscaling adds real capacity — so
both layers exist, and the node layer is bounded hard.

**Bounded, because the guardrail assumption broke.** Node autoscaling was originally
scoped out on the theory that a fixed node pool was a *physical* ceiling. A live check
showed the account had far more quota than a pure free-tier one and no physical
guardrail at all — autoscaling would have provisioned billable nodes with nothing to
stop it. The fix wasn't "never autoscale," it was **autoscale with a hard, documented
ceiling**: a second node pool that is *tainted* (only opt-in workloads land on it),
**scales to zero** (so idle costs nothing), and is capped at a small `max` that is the
money limit — a worst case in the low tens of dollars a month, and `$0` at rest. The
autoscaler physically cannot exceed that node count no matter what a runaway workload
requests.

**The two autoscalers hand off through the scheduler, not directly.** The pod
autoscaler adds a replica; the scheduler prefers the always-present baseline node and
only fails to place it if the baseline is full; a replica that doesn't fit goes
`Pending`; because the game server *tolerates* the burst pool's taint — but carries no
node-selector pinning it there — the scheduler sees it could fit a burst node, and the
node autoscaler adds one. The toleration *is* the handoff; the absence of a
node-selector is what keeps the server on the free baseline until it genuinely
overflows.

**Elasticity that's safe for a stateful game.** Game rooms are in-memory and
pod-local, so retiring a pod abruptly would drop every hand on it — and Kubernetes
sends `SIGTERM` before killing a pod on *every* scale-down, node drain, and rollout.
So the server drains: on `SIGTERM` it stops accepting new connections, forms no new
rooms, and gives the hands already under way a bounded window (kept under the pod's
termination grace period) to finish before it exits. A live hand is cut only if it
outlasts that window — and even then it's a client reconnect, not lost data, because
the wallet and history live in the database. None of the autoscaling targets are
guesses, either: a WebSocket load test where each virtual user plays a full hand is
what right-sizes them.

## Operating a cluster you can't reach directly

For most of this build, direct `kubectl` access to the cluster's API endpoint
was blocked by an upstream network path that couldn't be fixed locally — while
CI runners reached the same endpoint fine. Rather than fight it, cluster
operations were built around what *did* work: a manually-dispatched workflow
for privileged installs, a separate read-only diagnostics workflow that dumps
certificate, ingress, pod, and event state from a runner that can reach the
API, and the provider's browser shell for the one-time RBAC bootstrap. An
inconvenience became a forcing function toward operating the cluster the way a
team actually would — through pipelines, not a laptop.
