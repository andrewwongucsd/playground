# Engineering highlights

The decisions that shaped the build, the bugs found getting it live, and — at the
end — an explicit list of what still isn't proven. Most of the bugs only surfaced
because there was a real deployment to break.

## Decisions

**Everything on one cloud, not split across vendors.** The Go server, *both* React
frontends, and the database run on the same managed Kubernetes cluster. One platform,
one TLS setup, one deploy story. The alternative — parking the static frontends on a
separate static-site vendor — would have meant two of everything for one small system.

**An in-cluster database instead of a managed service.** Keeping Postgres in the
cluster under an operator kept everything on one platform and inside the free tier. The
cost was owning backups, which then became its own piece of work. That trade was made
knowingly, not stumbled into.

**Least privilege, three ways.** Three separate machine identities instead of one
shared credential: a per-push deploy identity boxed into one namespace by RBAC, a
cluster-admin identity wired only into a manually-triggered workflow, and a backup
identity that can write to exactly one bucket. The reasoning is always the same. The
credential exercised most often should hold the least power, and the credential that
lives permanently inside the cluster should be able to touch the least of all.

**A vulnerability scan as a hard gate.** Images are scanned for critical and high
findings *before* being allowed into the registry, so what gets scanned is exactly what
ships. It earned its place on a change that had nothing to do with security: adding a
tracing library pulled in a transitive gRPC dependency carrying a HIGH advisory, and
the gate stopped the deploy. Adding one library means inheriting the vulnerabilities of
its entire dependency tree — a gate turns that from a shipped incident into a one-line
version bump before anything reaches the registry.

**Harden the public edge, because a socket is a resource an attacker gets to spend.**
Per-IP caps on concurrent connections and a token-bucket rate limit, so one address
can't exhaust the server's goroutines or memory. A hard size cap on a single inbound
frame, applied before the very first read, because the library imposes none by default.
Read and write deadlines so one slow client can't wedge a room broadcast. A ping/pong
keepalive that reaps a connection whose client vanished without ever closing — a
backgrounded tab, a slept laptop, a dropped network.

**Automate the waiting — but make the loop read the error.** The free ARM capacity this
cluster runs on was exhausted for weeks; provisioning simply failed with "out of host
capacity," which clears only when some other tenant releases a machine. The manual fix
is a human re-running the same command for hours. The engineered fix is a retry loop —
but a loop that retries on *any* failure will happily retry a bug forever. So it
inspects the failure text and continues only while the error is genuinely a capacity
error, halting immediately on anything else. Sixty attempts over about four hours all
read "out of capacity"; attempt sixty-one hit a DNS resolution failure, and the loop
correctly **stopped** rather than reporting "still blocked." A detail that decided the
whole approach: a retry only helps if it can land somewhere new, and in a
single-availability-domain region, "try again" and "try the same thing again" are the
same command.

**Test the real boundary.** Anything that crosses a socket, a goroutine, a timer, or a
database gets at least one test against the *real* thing, not only an in-process fake.

```
                 /\        few, slow, highest-fidelity
                /  \
               / E2E\      (deliberate gap: no browser-driven test yet)
              /------\
             /  real  \    real WebSocket sockets, a full round played end to end
            / boundary \   real Postgres in CI (skipped cleanly when absent)
           /------------\
          /  concurrency \ real goroutines and timers, all under the race detector
         /   & timers     \
        /------------------\
       /    pure unit       \ the rules engine and pure client libs -- many, instant
      /  (table-driven)      \
     /------------------------\
```

That rule is policy because it *paid*. Two genuine bugs — a missing initial broadcast
and a duplicate round-over carrying zeroed results — were invisible to in-process fakes
and surfaced only against a real connection. A fake re-implements the boundary, and in
doing so it re-implements your assumptions, so it agrees with the bug. The real socket
doesn't. Optional real dependencies **skip rather than fail**, so "no setup needed
locally" and "actually tested in CI" are both true at once.

**No passwords anywhere — two doors, one cookie.** Login is a one-time link or a signed
mini-app payload from a chat platform. Nothing to hash, rate-limit, or leak, because
there is no password to steal. Both paths converge on a single cookie helper, so the
security flags can't drift apart. The token uses a cryptographic random source while
the card shuffle uses the fast one: if guessing it lets someone in, it gets the slow
generator. Consuming the token and creating the account happen in one transaction,
because a crash between them would leave a link marked used with no account behind it.
Consuming a link also *lands* you somewhere: it redirects into the app rather than
rendering a "you're logged in" page: that page told you to return to the tab that
asked for the link, which stopped being true the day links started arriving as a DM.

**Sign the state you hand to a client, and fail closed when you can't check it.**
Two places needed a secret that isn't a session. A matchmaking key is proved on one
connection and used on another, so it round-trips through the client under a MAC —
domain-separated from the other signatures sharing the same key, because one secret
signing three different things is how cross-protocol replays happen. A platform
webhook authenticates on an echoed shared secret compared in constant time, where an
*unset* secret rejects every delivery rather than falling open: the URL isn't a
secret, it's in an ingress rule and in TLS SNI, so the shared secret is the only thing
separating a real delivery from a forged one. Both defaults point the same way —
unverifiable input gets the least-privileged outcome, never the convenient one.

**Alert on symptoms and burn rates, not on thresholds.** A dashboard nobody watches
isn't monitoring, and a threshold alert either pages on every blip or sleeps through a
real outage. Objectives are expressed as error budgets and alerted with the
multi-window multi-burn-rate shape: fast burn pages, slow burn tickets, and each needs
a long *and* a short window to agree. Every alert is a symptom, never a cause, and
every one carries a runbook link — because the question at 3 a.m. is never "what's the
number," it's "what do I do."

**Sign what ships; declare what the image already is.** Keyless signing means the build
workflow's own identity is the signer, so there's no key to store or leak, and the
signature lands in a public transparency log. An SBOM rides along as an attestation on
the same digest. The pod's security context mostly *declares* properties the
distroless non-root image already had — which is the point, because an unenforced
property is a promise, and declaring it is what makes it enforceable and auditable.

**Stage what the cluster can't yet hold.** GitOps and a metric-gated canary are
authored, schema-validated, and *deliberately switched off*: a canary needs a second
pod the single baseline node can't fit. So are default-deny network policies, which
would be inert anyway unless the network plugin enforces them. The node autoscaler is
provisioned at the cloud layer and costs nothing at rest, but its controller is
**staged and not installed** — one deliberate step from ready. Shipping any of it
"live" would have produced a demo, not a control.

> **Say it in one line:** claiming less than you've built is cheaper than
> claiming more, and the difference belongs in writing.

## Bugs a real deployment surfaced

**The images were built for the wrong CPU architecture.** Runners are x86-64; the
cluster's nodes are ARM. Images were being built for the runner and would have failed
at container start with `exec format error`. Nothing caught it for weeks, because the
node pool was capacity-blocked so long the pod never left `Pending` — there was never a
running node to fail the pull on. Fixed with a cross-build that compiles for the target
architecture without emulation. A latent bug can hide behind an unrelated blocker for
as long as that blocker lasts.

**The session cookie was insecure behind the proxy.** The login cookie set its `Secure`
flag from whether the *server* saw a TLS connection. But TLS terminates at the ingress,
so the server always sees plain HTTP over the cluster network — meaning every real
user's session cookie would have shipped without the flag. The fix honours the
forwarded-protocol header, which is safe to trust *here* specifically because the
failure runs in the harmless direction: a spoofed value can only *add* the flag, never
remove it. In production a proxy sits in front of you — read what it forwarded, not
what your process saw.

**A workflow reported success while doing nothing.** After adding a manual-dispatch
trigger, the publish-and-deploy jobs still gated on the event being a push. A manual run
therefore built and scanned, then *silently skipped* publishing and deploying — and a
job made entirely of skipped steps reports **success**. Several green runs deployed
nothing at all; the tell was diagnostics showing no ingress objects and week-old
crash-looping pods. A pipeline that can skip its real work and still show green is worse
than one that fails, because the green actively lies.

**An invalid character made a workflow silently not run.** A zero-second failure with no
logs, caused by an unquoted colon in a YAML string that invalidated the whole file. "It
didn't run" and "it ran and failed" look different for a reason — a zero-second failure
almost always means the file, not the code.

**A sixty-second proxy default was quietly ending games.** Players waiting at a table
that hadn't filled would drop with a bare "Connection closed." in the UI and **no
server-side error at all**, because nothing had failed on the server. The ingress closes
a proxied connection after 60 seconds with nothing to read on it — a sane default for
ordinary HTTP and exactly wrong for a long-lived, mostly-idle WebSocket where a waiting
player legitimately sends nothing for minutes. The fix belongs in the edge config, not
the app: the app's behaviour was correct, and the door in front of it had an opinion
about how long a quiet connection may live. A proxy's timeouts are part of your
application's contract.

**The monitoring stack didn't fit — and the installer reported success.** Installing it
on a one-core node "succeeded": the install command waited, returned green, and the
metrics store then sat `Pending` for **23 hours** before anyone looked. The wait only
covers the chart's own resources, and the operator creates the real pod *afterwards*, so
the wait never saw it fail to schedule. A resource request that can't be satisfied isn't
an error — it's a pod waiting politely, forever.

That one incident taught three separate things:

- **Requests are reservations, not usage.** The scheduler enforces what you asked for
  even when actual consumption is a fraction of it. On an idle service the gap is
  enormous, and it is the *request* that decides whether anything else fits.
- **A full node can't roll.** A rolling update wants to start a new pod before retiring
  the old one, and on a saturated node that surge pod has nowhere to go — so the rollout
  hangs rather than failing cleanly.
- **Add before you replace.** The safe node migration brings up the bigger node *before*
  draining the old one. That discipline paid immediately: the replacement node failed to
  register — a timeout, not a capacity problem — and because nothing had been drained,
  it was a no-op rollback instead of an outage.

**Draining a stateful pod has its own trap, and forcing it costs six minutes.** A
single-instance database pod can't be evicted while its disruption budget insists on
keeping one copy, so the drain waits forever. The eviction policy has to be set *before*
the drain, not during. Terminating the node's instance outright is safe by design here —
the data lives on a separate volume, not the boot disk — but recovery wasn't instant:
the record of the old attachment lingers, so the volume can't reattach until the
controller force-detaches from the unreachable node, roughly six minutes later. The
honest cost of a *forced* stateful migration is a six-minute outage, not the few seconds
a graceful pod move takes.

**A one-line security hardening took the site down.** Setting a read-only root
filesystem on the game server shipped like any other change. The rollout never went
Ready and the game served `503` while the static frontend kept answering `200` — a
half-up appearance that is genuinely worse than a clean outage, because it reads as
"mostly fine." The recovery was to revert the manifest and let the pipeline redeploy.
Two lasting outputs: a rollback runbook written the same day while the commands were
still exact, including the hour lost to a theory the facts disproved; and the note that
a timed-out rollout tells you pods aren't Ready but never *why*, so getting the why
first is worth the minute.

**"Run as non-root" needs a number, not a name.** A second hardening change in the same
batch declared that the pod must run as a non-root user. The image already did — but
the kubelet has to *verify* that before starting the container, and it can only do so
from a numeric user id. Given a username, it cannot resolve the name to prove anything,
so it refuses the container outright with a config error. The site went down until it
was diagnosed. Pinning the user and group to the image's actual numeric ids made the
claim checkable. The general shape is the same as the read-only filesystem incident:
a declaration that *looks* like documentation is enforced by something with an opinion,
and the enforcement happens at container start, not at review time.

**A silent selector scraped nothing.** The metrics operator only discovers scrape
targets carrying its own release label by default. Point it at a target in another
namespace without widening that selector and you get no error, no warning, and empty
graphs — the most expensive kind of failure, because everything looks installed. Check
for *data*, not for a green install.

**A Ready pod can still be wired wrong.** The trace datasource was pointed at the log
store's port. The install was happy, the pod was Ready, and every query came back empty.
Confirm ports from the resource, not from muscle memory.

**A flaky test where both outcomes were correct.** A test asserted that the server tears
down a connection after an oversized frame. But the client's write and the server's
close are two ends of one socket racing each other:

```
  client lane                          server lane
  -----------                          -----------
  write oversized frame ---+
                           |           read exceeds the cap
                           |           close the connection ---+
                           v                                   v
  time ------------------------------------------------------------>

  outcome A (write wins the buffer):  the next read sees the close   ✓ cap held
  outcome B (close wins):             the write fails, reset by peer ✓ cap held
```

Either side may win, and **both prove the cap held**. The test was accepting one of two
correct answers. Two lessons stacked: a race detector over real sockets surfaces timing
truths a mocked test never would, and the commit that reveals a flake is rarely the
commit that caused it.

**A best-effort feature had to be bounded before it could ship.** Naming a nameless
player after their approximate location means calling a third party *inside* the join
path. Three properties made it safe rather than a liability: a hard timeout, so a slow
service can't become a slow join; a cache, so reconnects don't re-query; and a single
"I don't know" return value that folds into the existing random-name default. Nothing
downstream can tell the difference between "no answer" and "no attempt".

> **Say it in one line:** the failures worth writing down are the ones where
> everything looked green.

## Storage gotchas worth remembering

Two object-storage behaviours that were real `apply`-time errors, not anything
documented up front:

- **Immutability locks are mutually exclusive with versioning.** What reads like a
  retention setting was actually a write-once-read-many lock, and it refused to coexist
  with versioning. Expiry turned out to be a *separate* lifecycle-policy resource.
- **A lifecycle policy needs the storage service's own principal granted on the bucket
  first.** The deletions are executed by the service itself, so the policy can't even be
  created until that principal has rights — the one permission grant in the whole project
  made to a cloud *service* rather than to a user or a machine identity.

## What "done" was allowed to mean

A green pipeline proves the API accepted a desired state. It does not prove the thing
works. The real acceptance checks were: every hostname returning `200` to a *standard*
client with no certificate-verification bypass; the certificate issuer reading as the
real authority rather than the ingress controller's placeholder; a full round of the
game played end to end over a real secure WebSocket with the outcome persisted to the
database; and an on-demand backup verified to have actually written objects to the
bucket.

**A played hand beats a green check.**

## What isn't proven yet

Stating this is part of the engineering, not an apology for it.

- **No restore has ever been performed.** Continuous archiving is configured and
  verified to write objects, but recovery has never been exercised. By this project's
  own standard that makes it a hypothesis. Rehearsing it — with a defined recovery point
  and recovery time objective — is the next phase.
- **No alert has ever been delivered in anger.** Routing, severities, and the inhibition
  rule are wired, and delivery points at a real chat bot rather than the default
  receiver that notifies nobody. It has not yet fired on a live incident, so it is
  listed as unverified.
- **The load test hasn't been run.** The scenario is written — each virtual user plays a
  full hand over a real socket — but it has not been executed, so the pod autoscaler's
  CPU target is still a placeholder, and the manifest says so in a comment. Untested
  autoscaling is an assumption, not a control. There's also a real reason it isn't a
  free afternoon: **one load generator is one IP, and the server rate-limits per IP**,
  so past a few dozen virtual users you stop measuring the game and start measuring
  your own defences. A useful run needs several source addresses or a test deployment
  with the limits lifted — which is exactly the kind of detail that turns "just run a
  load test" into a piece of work.
- **CPU is the wrong signal, knowingly.** Long-lived idle WebSocket connections barely
  move CPU, so it is a weak proxy for real load. The better signal is active rooms per
  pod, and the upgrade to it is staged in the file rather than pretended away.
- **The game client has no automated tests.** The pure client-side libraries in the
  other product are unit-tested, and the server's real-socket test verifies the wire
  protocol from the Go side. The browser client is verified by hand. That's a real gap,
  and the top of the test pyramid above is drawn empty on purpose.

> **Say it in one line:** an untested backup, an unfired alert, and an unrun load
> test are three hypotheses — and calling them anything else is how outages get
> rehearsed on real users.
