# Engineering highlights

The decisions that shaped the build, the bugs found getting it live, and — at the
end — an explicit list of what still isn't proven. Most of the bugs only surfaced
because there was a real deployment to break.

**Never used Kubernetes?** Read the [Kubernetes primer](kubernetes-primer.md) first —
it teaches the vocabulary from zero and carries the glossary for every term used here.
[architecture.md](architecture.md) explains how the system is *built*; this document is
what went *wrong* while building it, and what that cost.

**How to read this.** Each section opens with a plain-English box and closes with one
line you could say out loud. The bug stories are deliberately told with the symptom
first, because that is the order you meet them in.

---

## Contents

1. [Decisions](#decisions) — the trades made on purpose
2. [Bugs a real deployment surfaced](#bugs-a-real-deployment-surfaced) — the ones a green pipeline never shows you
3. [Storage gotchas worth remembering](#storage-gotchas-worth-remembering)
4. [What "done" was allowed to mean](#what-done-was-allowed-to-mean)
5. [What isn't proven yet](#what-isnt-proven-yet) — the honest list

---

## Decisions

> **In plain English:** none of these were obvious at the time. Each is a fork in the
> road where the cheap option and the right option pointed different ways, and the
> reason for going one way is written down so it can be argued with later.

**Everything on one cloud, not split across vendors.** The Go server, *both* React
frontends, and the database run on the same managed Kubernetes cluster. One platform,
one TLS setup, one deploy story. The alternative — parking the static frontends on a
separate static-site vendor — would have meant two of everything for one small system.

**An in-cluster database instead of a managed service.** Keeping Postgres in the
cluster under an operator kept everything on one platform and on shapes the provider
gives away permanently. (Worth stating precisely: the *shapes* are free, the *account*
is Pay-As-You-Go with no spending ceiling of its own — free here is a choice you keep
making, not a wall you can lean on.) The cost was owning backups, which then became its
own piece of work. That trade was made knowingly, not stumbled into.

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
security flags can't drift apart.

The login token comes from a cryptographic random source; the card shuffle uses the
fast generator. The line between them is **not** "auth is serious and games are not" —
it is what a successful guess actually *wins*. Guessing a login token gets you
somebody's account, permanently. Guessing the shuffle gets you an edge in a
practice-scoring game with no wagering, no payout, and no cash-out anywhere in the API.
That boundary is what makes the fast generator acceptable, so the trigger is written
down: **add stakes, a ranked ladder, or any persistent standing, and the shuffle needs
`crypto/rand` too.**

Consuming the token and resolving the account happen in one transaction, so a crash
cannot mark a link used while leaving no account behind. But it stops one step short.
Creating the *session row* happens **after** the commit — so if that insert fails, the
user gets a 500 and the link is already burned. That is the same class of bug the
transaction was written to prevent, arriving one step later. Small window, small blast
radius, and listed here rather than left to read as solved.

Where a consumed link *lands* is the other loose end: it still renders a "you're logged
in" page telling you to return to the tab that asked — which stopped being true the
day links started arriving as a DM. The redirect that fixes it is written but not
merged, so it is listed under what isn't proven rather than claimed here.

**Sign the state you hand to a client, and fail closed when you can't check it.**
Two places needed a secret that isn't a session. A matchmaking key is proved on one
connection and used on another, so it round-trips through the client under a MAC —
domain-separated from the other signatures sharing the same key, because one secret
signing three different things is how cross-protocol replays happen. A platform
webhook authenticates on an echoed shared secret, compared in constant time. An
*unset* secret rejects every delivery rather than falling open — and that default
matters, because the URL isn't a secret. It sits in an ingress rule and in TLS SNI. The
shared secret is the only thing separating a real delivery from a forged one. Both
defaults point the same way: unverifiable input gets the least-privileged outcome,
never the convenient one.

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
the same digest. Scoped honestly: the game's two images are signed, the toolbox image
is not, and no admission policy yet refuses an unsigned one — the signature is
provenance anyone can check, not a gate the cluster enforces. The pod's security
context mostly *declares* properties the distroless non-root image already had — which
is the point, because an unenforced property is a promise, and declaring it is what
makes it enforceable and auditable.

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

> **In plain English:** every one of these passed code review, passed CI, and showed a
> green checkmark. They are here because that is exactly what made them expensive — a
> failure that announces itself gets fixed in an hour, and a failure that reports
> success can sit there for weeks.

**The images were built for the wrong CPU architecture.** Runners are x86-64; the
cluster's nodes are ARM. Images were being built for the runner and would have failed
at container start with `exec format error`.

```
   WHERE IT WAS BUILT              WHERE IT WOULD RUN
   ------------------              ------------------
   CI runner: x86-64        -->    cluster node: ARM
        |                                |
        v                                v
   binary of x86 machine           the CPU cannot decode those
   instructions                    instructions at all
                                          |
                                          v
                                   `exec format error` on start

   WHY NOBODY NOTICED FOR WEEKS:

   the node pool was capacity-blocked, so the pod never left `Pending`
        |
        v
   Pending means never scheduled -> never pulled -> never started
        |
        v
   the error needs a RUNNING node to happen on. There wasn't one.
```

Fixed with a cross-build that compiles for the target architecture without emulation.
**A latent bug can hide behind an unrelated blocker for as long as that blocker
lasts** — and when the blocker clears, you get both problems at once.

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
therefore built and scanned, then *silently skipped* publishing and deploying.

```
   a MANUAL run                        every step's condition:  if event == "push"
   ------------                        ----------------------
   build    -> ran                     (no condition)          ✓
   scan     -> ran                     (no condition)          ✓
   push     -> SKIPPED                 event was "dispatch"    -
   deploy   -> SKIPPED                 event was "dispatch"    -
        |
        v
   a job made entirely of SKIPPED steps reports ......... SUCCESS ✅
        |
        v
   green checkmark. Nothing was published. Nothing was deployed.
```

Several green runs deployed nothing at all; the tell was diagnostics showing no ingress
objects and week-old crash-looping pods. **A pipeline that can skip its real work and
still show green is worse than one that fails, because the green actively lies.**

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
metrics store then sat `Pending` for **23 hours** before anyone looked.

```
   t=0    install chart --wait
             |
             v
          creates the chart's OWN resources ......... all Ready ✓
             |
             v
   t=30s  --wait is satisfied. Command exits GREEN. ✅
             |
             |   ... but the chart installed an OPERATOR, and the operator
             |       creates the real metrics pod a moment LATER ...
             v
   t=40s  operator creates the metrics pod
             |
             v
          scheduler: "you asked for more CPU than any node has free"
             |
             v
   t=40s  Pending ......... and still Pending 23 HOURS later.

   The wait watched the wrong generation of objects. It could not have
   caught this: the thing that failed did not exist yet when it stopped
   watching.
```

**A resource request that can't be satisfied isn't an error — it's a pod waiting
politely, forever.**

That one incident taught three separate things:

- **Requests are reservations, not usage.** The scheduler enforces what you asked for
  even when actual consumption is a fraction of it. On an idle service the gap is
  enormous, and it is the *request* that decides whether anything else fits.
- **A full node can't roll.** A *rolling update* replaces pods one at a time, starting
  the new one **before** retiring the old — briefly needing room for both. That extra
  copy is the "surge" pod, and on a saturated node it has nowhere to go, so the rollout
  hangs rather than failing cleanly. Zero-downtime updates need spare capacity to be
  possible at all.
- **Add before you replace.** The safe node migration brings up the bigger node *before*
  draining the old one. That discipline paid immediately: the replacement node failed to
  register — a timeout, not a capacity problem — and because nothing had been drained,
  it was a no-op rollback instead of an outage.

**Draining a stateful pod has its own trap, and forcing it costs six minutes.**

Three words first, because the story is unreadable without them. **Draining** a node
means politely emptying it before maintenance — asking every pod to leave. **Evicting**
is that ask, applied to one pod. A **PodDisruptionBudget** is a floor you set on
availability: *"never leave me with fewer than this many copies."* Now the trap is
visible:

```
   THE GRACEFUL PATH (what you expect)
   -----------------------------------
   drain node -> evict pod -> pod restarts elsewhere -> volume follows
                                                        seconds

   THE DEADLOCK (what happened)
   ----------------------------
   drain node ---> "may I evict the database pod?"
                         |
                         v
              PodDisruptionBudget: "keep at least 1 copy"
                         |
                         v
              there IS only 1 copy. Evicting it leaves 0.
                         |
                         v
              REFUSED -> the drain waits. Forever. No error, no timeout.

   THE FORCED PATH (the way out, and its price)
   --------------------------------------------
   terminate the node outright
        |   safe here: the data is on a separate volume, not the boot disk
        v
   new pod starts elsewhere and asks for its volume
        |
        v
   the volume is still recorded as ATTACHED to a node that no longer answers
        |
        v
   wait for the controller to force-detach it .......... ~6 MINUTES
        |
        v
   attach -> mount -> start
```

The budget has to be relaxed *before* you start the drain, not during — once the drain
is waiting, it is waiting on a rule you can no longer change in time to help.

**A one-line security hardening took the site down.** Setting a read-only root
filesystem on the game server shipped like any other change. The rollout never went
Ready and the game served `503` while the static frontend kept answering `200`.

```
   WHAT A MONITOR SAW                  WHAT A PLAYER SAW
   ------------------                  -----------------
   GET /            -> 200  ✓          page loads, looks perfect
   GET /healthz     -> 200  ✓                  |
        ^                                      v
        |  both served by the             click "play" -> nothing
        |  STATIC frontend, which          works. The socket 503s.
        |  was never touched
                                       "Is the site up?" has two
   GET /queue       -> 503  ✗          different true answers.
   GET /ws          -> 503  ✗
```

A half-up appearance is genuinely worse than a clean outage, because it reads as
"mostly fine" — to a person *and* to any check that happens to probe the wrong path.
The recovery was to revert the manifest and let the pipeline redeploy.
Two lasting outputs: a rollback runbook written the same day while the commands were
still exact, including the hour lost to a theory the facts disproved; and the note that
a timed-out rollout tells you pods aren't Ready but never *why*, so getting the why
first is worth the minute.

**"Run as non-root" needs a number, not a name.** A second hardening change in the same
batch declared that the pod must run as a non-root user. The image already did — but
the **kubelet** (the agent on each node that actually starts containers) has to
*verify* that claim before starting it, and it can only do so from a numeric user id.
Given a username, it cannot resolve the name to prove anything, so it refuses the
container outright with a config error. The site went down until it
was diagnosed. Pinning the user and group to the image's actual numeric ids made the
claim checkable. The general shape is the same as the read-only filesystem incident:
a declaration that *looks* like documentation is enforced by something with an opinion,
and the enforcement happens at container start, not at review time.

**A silent selector scraped nothing.** Prometheus does not receive metrics; it goes
and fetches them, and the things it fetches from are called **scrape targets**. Which
targets it will even look at is decided by a label selector — and by default the
operator only adopts targets stamped with its own release label.

```
   target in another namespace, no matching label
             |
             v
   the operator never adopts it -> Prometheus never scrapes it
             |
             v
   no error. No warning. No failed pod. Just empty graphs.
```

That is the most expensive kind of failure, because everything *looks* installed —
green install, Ready pods, a dashboard that renders perfectly with nothing in it.
**Check for data, not for a green install.**

**A Ready pod can still be wired wrong.** The trace datasource was pointed at the log
store's port. The install was happy, the pod was Ready, and every query came back empty.
Confirm ports from the resource, not from muscle memory.

**A pillar that was "installed" for eight days had never once stayed up.** Tempo's
memory limit was set to 512Mi. It was `OOMKilled` 2,251 times over eight days — never
surviving startup, not once — while every manifest and every install log said the
traces pillar was done. It runs at a 2Gi limit; the chart's own example sizes this
component at a 4Gi *request*, so 512Mi was never in the right range.

Three things made it invisible, and each is worth naming separately:

```
  1. the install reported success        helm --wait covers the chart's own
                                         resources; a container that dies AFTER
                                         the first probe passes is not its problem

  2. its own logs said nothing wrong     Tempo started cleanly every time -- every
                                         module up, every receiver listening -- and
                                         was then killed BY THE KUBELET. The process
                                         never failed, so it never logged a failure.
                                         Only lastState.terminated.reason names it:
                                         OOMKilled, exit 137.

  3. the fix would not land either       a StatefulSet's rolling update will not
                                         replace a pod that is already unhealthy, so
                                         raising the limit changed nothing: the
                                         controller kept restarting the SAME pod.
                                         Restart count climbed 2251 -> 2254 on a pod
                                         whose age never reset, and helm --wait timed
                                         out having changed nothing observable.
```

The repair was to stop waiting on a deadlock: apply the values, delete the stuck pod so
it comes back on the new template, then gate on `rollout status`. It came up healthy on
the first try, and Tempo's API now answers with `big2-server` as a service that has sent
it traces — which is the check that should have been run on day one.

What it cost to find was one question nobody had asked in eight days — *is the pod
Running?* — which is the same question the Prometheus-`Pending` bug above asked, in a
form that `kubectl get pods` answers instantly. The diagnostics workflow now dumps a
crashlooping container's previous logs and its termination reason, and asks Prometheus
whether each SLI actually has samples, because "the rule applied" and "the rule
measures something" are different claims.

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

**A login link that worked landed on a page that told you it hadn't.** The one-time
link consumed the token, set the session cookie, and rendered a plain "you're logged
in — close this tab and go back" page. That copy was correct while a *waiting* tab
existed to return to; it stopped being true the day the delivery channel changed to a
bot DM, so the tab the link opened was the only tab there ever was. The login worked
every time. The screen it ended on told the user to go somewhere they had never been,
and the natural response — click the link again — hit "invalid, expired, or already
used", which is what actually generated every error report. The fix lands the same
response that sets the cookie in a redirect into the app instead, over a 303 so
nothing can replay the token as anything but the GET that consumed it. **A correct
action nobody can observe is indistinguishable from a broken one** — and users
respond to invisible success by retrying, which is what manufactures the visible
failure.

> **Say it in one line:** the failures worth writing down are the ones where
> everything looked green.

## Storage gotchas worth remembering

> **In plain English:** two settings on a storage bucket that sound like they should
> work together, and don't. Neither is written down anywhere obvious — both were
> discovered by an `apply` failing.

Two object-storage behaviours that were real `apply`-time errors, not anything
documented up front:

- **Immutability locks are mutually exclusive with versioning.** What reads like a
  retention setting was actually a write-once-read-many lock, and it refused to coexist
  with versioning. Expiry turned out to be a *separate* lifecycle-policy resource.
- **A lifecycle policy needs the storage service's own principal granted on the bucket
  first.** The deletions are performed by the storage service itself, not by you. So the
  policy cannot even be *created* until that service has rights on the bucket. It is the
  one permission in the whole project granted to a cloud *service* rather than to a
  person or a machine identity.

> **Say it in one line:** a setting that sounds like a preference may be a lock, and a
> policy you write may be executed by someone who needs their own permission.

## What "done" was allowed to mean

> **In plain English:** "the deploy succeeded" only means a server accepted your
> instructions. It says nothing about whether the thing works. So "done" was defined as
> a short list of things a person actually did, not things a pipeline reported.

A green pipeline proves the API accepted a desired state. It does not prove the thing
works. Four checks were allowed to count:

```
   ✓ every hostname returns 200 to a STANDARD client
       -- no certificate-verification bypass, no `-k`, no exceptions

   ✓ the certificate issuer reads as the real authority
       -- not the ingress controller's built-in placeholder cert

   ✓ a FULL ROUND of the game played end to end over a real secure socket
       -- with the outcome persisted to the database afterwards

   ✓ an on-demand backup verified to have written real objects to the bucket
       -- checked in the bucket, not inferred from an exit code
```

Each one is phrased to be un-fakeable by a green checkmark. The pattern is the same
every time: **check the effect, not the report.**

> **Say it in one line:** a played hand beats a green check.

## What isn't proven yet

> **In plain English:** the list nobody puts in a portfolio. Everything below is built
> but unexercised — and until it has been exercised, it is a belief rather than a
> capability. Writing them down is the point; a gap you have named is a plan.

Stating this is part of the engineering, not an apology for it.

- **No restore has ever been performed.** Continuous archiving is configured and
  verified to write objects, but recovery has never been exercised. By this project's
  own standard that makes it a hypothesis. Rehearsing it — with a defined recovery point
  and recovery time objective — is the next phase.
- **No alert has ever been delivered in anger — and the delivery path is not actually
  switched on.** Routing, severities, and the inhibition rule are wired, and a chat-bot
  receiver is authored in full. But the installer applies it only when *both* its bot
  token and its chat id are configured, and the chat id is unset — so the condition
  fails and alerts fall through to the default receiver that notifies nobody. It is one
  variable away from real delivery, which is a materially different claim from "wired to
  a chat bot," and the honest version is the one worth writing down.
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
- **Terraform state still lives on one machine.** Two of the three machine identities
  have their key pairs *generated* by Terraform, which means their private keys sit in
  the state file — and that file has no remote backend, no locking, and no versioning.
  A remote encrypted backend with state locking is now authored but **not migrated**,
  and the two key pairs have **not been rotated**, which they must be once it is. Losing
  that file wouldn't take the site down; it would do something worse — leave the
  infrastructure running while becoming unmanageable, recoverable only by importing
  every resource back by hand.
- **The session-row gap in the login flow.** Described under Decisions above: token
  consumption and account resolution are atomic, but the session insert that follows is
  not, so a failure there burns a link for good. Small window, small blast radius, and
  an unambiguous fix — move the insert inside the transaction — that hasn't been made.

> **Say it in one line:** an untested backup, an unfired alert, and an unrun load
> test are three hypotheses — and calling them anything else is how outages get
> rehearsed on real users.
