# Engineering highlights

The decisions that shaped the build, and the bugs found getting it live — most
of the latter only surfaced because there was a real deployment to break.

## Decisions

**Everything on one cloud, not split across vendors.** The Go server, *both*
React frontends (as nginx pods behind the same ingress), and the database all
run on the same managed Kubernetes cluster. One platform, one TLS setup, one
deploy story — over the alternative of hosting the static frontends on a
separate static-site vendor.

**In-cluster database over a managed service.** Keeping the database in the
cluster (operator-run, on a block volume) rather than reaching for an external
managed Postgres kept everything on one platform and inside the free tier,
at the cost of owning backups — which then became their own piece of work.

**Least-privilege by default, three ways.** Three separate machine identities
instead of one shared credential: a per-push deploy identity scoped by
Kubernetes RBAC to a handful of resource kinds in two namespaces; a
cluster-admin identity wired only into a manually-triggered workflow, never
the automated one; and a backup identity that can write to exactly one bucket.
The reasoning is always the same — the credential exercised most often (the
per-push deployer) should hold the least power, and the credential that lives
permanently inside the cluster (the backup key) should be able to touch the
least.

**A vulnerability scan as a hard gate.** Images are scanned for critical and
high CVEs *before* being allowed into the registry — scan-before-push, so what
gets scanned is exactly what ships. The first real run of this immediately
earned its place by rejecting a base image with four fixable high-severity
CVEs; the fix was a base-image bump the scan made unmissable.

**Keep the runbook in the repo.** Every infrastructure step — the real
commands, the resources they produced, and *why* that approach is the
industry-standard one rather than the minimal one — was written up as it
happened, not narrated once in a chat and lost. A written record is the
difference between "it worked once" and "anyone can do it again."

## Bugs a real deployment surfaced

**The images were built for the wrong CPU architecture.** CI runners are
x86-64; the cluster's nodes are ARM. Images were being built for the runner's
architecture and would have failed at container start with `exec format
error`. Nothing caught it for weeks because the node pool was capacity-blocked
so long the pod never left `Pending` — there was never a running node to fail
the pull on. Fixed with a cross-build that compiles for the target
architecture without emulation.

**The session cookie was insecure behind the proxy.** The login cookie set its
`Secure` flag based on whether the *server* saw a TLS connection. But TLS
terminates at the ingress, so the server always sees plain HTTP over the
cluster network — meaning every real user's session cookie would have shipped
without the Secure flag. The fix honours the forwarded-protocol header, which
is safe to trust here specifically because the failure runs in the harmless
direction: a spoofed value can only *add* the Secure flag, never remove it.

**A workflow that reported success while doing nothing.** After adding a
manual-dispatch trigger to the deploy pipelines, their publish-and-deploy jobs
still gated on the event being a `push`. A manual run therefore built and
scanned, then *silently skipped* publishing the image and deploying it — and a
job made entirely of skipped steps reports **success**. Several "green" runs
deployed nothing at all; the tell was cluster diagnostics showing no ingress
objects and only week-old crash-looping pods. The lesson is sharp: a pipeline
that can skip its real work and still show a green check is worse than one that
fails, because the green check actively lies. The fix was to gate on "not a
pull request" instead of "is a push."

**An invalid character in a workflow file.** One workflow silently failed to
run at all — a `0`-second failure with no logs — because an unquoted colon in a
YAML string made the whole file invalid. A reminder that "it didn't run" and
"it ran and failed" look different for a reason, and a 0-second failure almost
always means the file, not the code.

## Storage gotchas worth remembering

Two object-storage behaviors that were real `apply`-time errors, not anything
documented up front:

- **Immutability locks are mutually exclusive with versioning.** What reads
  like a retention/expiry setting on the bucket was actually a
  write-once-read-many immutability lock, and it refused to coexist with
  versioning. Expiry turned out to be a *separate* lifecycle-policy resource.
- **A lifecycle policy needs the storage service's own principal granted on
  the bucket first.** Lifecycle deletions are executed by the object-storage
  service itself, so the policy can't even be created until that service
  principal has rights on the bucket — the one permission grant in the whole
  project made to a cloud *service* rather than to a user or a machine
  identity.

## What "done" was allowed to mean

A green pipeline proves the API accepted a desired state. It does not prove the
thing works. The real acceptance checks were: every hostname returning `200`
to a *standard* client (no certificate-verification bypass), the certificate
issuer reading as the real authority rather than the ingress controller's
built-in placeholder, a full round of the game played end-to-end over a real
secure WebSocket through the ingress with the outcome persisted to the
database, and an on-demand backup verified to have actually written objects to
the bucket. A played hand beats a green check.
