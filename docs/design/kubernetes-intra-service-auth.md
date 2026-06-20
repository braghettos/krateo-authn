# Design: Kubernetes intra-service authentication (SA token → service JWT)

> Status: **Draft for discussion** · Date: 2026-06-20
>
> Goal: let any in-cluster Krateo **service** authenticate to authn with its **own
> Kubernetes ServiceAccount token** and receive an authn-issued **JWT bound to a scoped
> `clientconfig`** — the same identity shape human users get — so it can call
> `UserConfig`-gated services (snowplow `/call`, etc.) **unchanged**. This generalises what
> authn already does for its *own* identity.

---

## 1. Why

Krateo back-end services increasingly need to call other Krateo services that are gated by
authn's per-identity model. The motivating case: **core-provider's composition-dynamic
controller (CDC)** must resolve a `RESTAction` through **snowplow** to populate a
Composition's status (see core-provider `docs/design/composition-status-projection.md`).
But snowplow's `/call` (its `UserConfig` middleware) requires an `Authorization: Bearer`
**JWT signed with the shared signing key**, then loads a per-user `<user>-clientconfig` —
i.e. the JWT *is* how snowplow scopes Kubernetes RBAC per identity. A controller is **not a
user** and has no such JWT.

The two obvious workarounds are both bad:

- **Give the caller the signing key** so it self-mints a JWT — leaks a platform-wide secret
  into every consumer.
- **A bespoke no-auth/own-SA endpoint per callee** (e.g. a snowplow service-mode route) —
  per-service special-casing, and it bypasses the per-identity RBAC scoping that is the
  whole point.

**authn already solved this for itself.** On startup it self-mints a service JWT and
provisions its own scoped identity to call snowplow's RESTActions:

- `jwtutil.CreateToken({ Username: "authn", Groups: ["authn"], SigningKey, Duration: 1y })`
  — `main.go:158`.
- `signup.Do({ Username: "authn", UserGroups: ["authn"], … })` → provisions
  `authn-clientconfig` — `main.go:230` ("Create authn clientconfig to call snowplow's
  RESTActions").

It can do this only because **it holds the signing key** and runs the signup machinery.
**Generalising that to other services — without handing them the key — is this proposal.**

---

## 2. What

A new **login strategy**, `serviceaccount` (Kubernetes SA token as the credential), slotting
into authn's existing strategy pattern (`internal/routes/auth/{basic,oauth,ldap}` →
`routes.Route` with `Pattern()/Method()/Handler()`, given the `KubeconfigGenerator` + signing
key). No new framework — it is "log in with your Kubernetes SA token".

```
POST /serviceaccount/login
Authorization: Bearer <caller's projected SA token, audience: "authn">
→ 200 { accessToken: <authn JWT>, ... }   # same success shape as the other strategies
```

### Flow

1. **Caller** mounts a **projected SA token** (`TokenRequest`, `audience: authn`, short TTL)
   and presents it as the Bearer credential.
2. **authn validates it via the Kubernetes `TokenReview` API** (`authentication.k8s.io`),
   *with the expected audience*. On success it gets the authenticated
   `username = system:serviceaccount:<ns>:<sa>` and the SA's groups.
3. **Authorization / identity mapping (the security-critical step, §4):** authn looks up a
   **mapping policy** — is this SA allowed to exchange, and to which service identity
   (username + groups) does it map? Unmapped SAs are rejected.
4. **Issue:** authn mints the JWT with the **existing** `jwtutil.CreateToken` (mapped
   username/groups, signing key, short duration) and **ensures the scoped `clientconfig`**
   with the **existing** `signup.Do` / `KubeconfigGenerator`. RBAC for that identity is the
   clientconfig's — least privilege, centralized here.
5. **Return** the JWT (same response shape as `basic`/`oauth`). The caller presents it to
   snowplow `/call` (or any `UserConfig`-gated service) **unchanged**.

**New code is small and bounded:** the `serviceaccount` strategy handler + the `TokenReview`
call + the mapping policy. JWT minting (`jwtutil.CreateToken`), clientconfig provisioning
(`signup.Do`), and the kubeconfig generator already exist and are reused verbatim.

---

## 3. Caller side (e.g. the CDC)

- Project an audience-bound SA token via the Deployment's
  `serviceAccountToken` projected volume (`audience: authn`, `expirationSeconds: ~600`).
- A tiny client (candidate home: **`plumbing`**, so every service shares it): read the
  projected token, `POST /serviceaccount/login`, **cache the returned JWT** until shortly
  before its expiry, present it on downstream calls. This mirrors the existing
  `internal/chartinspector` HTTP-client pattern in the CDC.
- For composition status, the CDC then calls snowplow `/call` with the JWT — no signing key,
  no endpoint credentials in the CDC.

---

## 4. Security model

The exchange must be **strictly bounded** — it issues platform identities:

- **Audience-bound tokens.** Require `audience: authn` on the projected token and pass it to
  `TokenReview`; reject tokens minted for any other audience. Prevents replay of tokens
  issued for the kube-apiserver or other services.
- **Bound SA tokens only.** Use `TokenRequest`-projected, short-TTL, audience-scoped tokens
  — never legacy long-lived SA Secrets.
- **Explicit mapping allowlist.** Only SAs named in the mapping policy can exchange, and the
  policy fixes the **identity + groups** they map to (hence their `clientconfig` RBAC). No
  implicit "any SA → some identity". This is where per-service least privilege is set, and
  it replaces "resolve under snowplow's broad SA".
- **Short JWT TTL** on issued service tokens (re-exchange on expiry; the caller caches
  between).
- **No privilege amplification:** the mapped `clientconfig` should grant only what that
  service legitimately needs (e.g. the reads its RESTActions perform), provisioned/audited in
  one place (authn) rather than scattered across callees.

authn's own SA gains one permission: `create` on
`tokenreviews.authentication.k8s.io`.

### Mapping policy — shape (open, §6)

A declarative mapping from SA → service identity, e.g. a cluster-scoped CR or a config:

```yaml
# illustrative
serviceAccount: { namespace: krateo-system, name: composition-dynamic-controller }
mapsTo:
  username: svc:composition-dynamic-controller
  groups:   [ "krateo:services", "composition-status" ]   # → drives the clientconfig RBAC
```

---

## 5. Why this is the right layer

- **One mechanism, platform-wide.** Any Krateo service → any `UserConfig`-gated service.
  The composition-status CDC→snowplow path is just the first consumer.
- **No callee changes.** snowplow `/call` and its `UserConfig` are untouched; the service is
  simply a first-class identity with a `clientconfig`.
- **No secret sprawl.** The signing key never leaves authn; callers use k8s-native,
  audience-bound, short-lived SA tokens validated by the apiserver (`TokenReview`).
- **Reuses existing authn machinery** (`jwtutil.CreateToken`, `signup.Do`, kubeconfig
  generator) — already exercised by authn's own identity.

---

## 6. Open questions

- **Mapping policy home & shape** — a dedicated CRD (e.g. `ServiceAccountBinding`) vs. a
  ConfigMap; how it maps SA → username/groups, and how the resulting `clientconfig` RBAC is
  expressed/provisioned.
- **Token audience & naming** — the canonical audience string and service-username scheme
  (`svc:<name>`?), and group conventions.
- **JWT lifetime & caching** — issued-token TTL vs. caller cache window; revocation story.
- **Multi-cluster** — `TokenReview` is cluster-local. A service running in a **remote target
  cluster** authenticating to authn on the management cluster needs either authn to validate
  against the target's apiserver, or an authn presence per cluster. (Mirrors the
  composition-status multi-cluster `apiRef` placement question.)
- **Endpoint path/verb** — `/serviceaccount/login` (POST) vs. a token-exchange-style
  `/token` (RFC 8693 framing).
- **Bootstrap ordering** — authn must be reachable before dependent controllers can resolve
  `apiRef`; degrade gracefully (status fields stale, never a failed reconcile).

---

## 7. Scope

In: the `serviceaccount` login strategy (TokenReview + mapping + issue), authn SA RBAC for
`tokenreviews`, a shared caller client (plumbing). Out (separate): the consumers
(core-provider/CDC wiring lives in its own design), and the multi-cluster validation path
(tracked with the composition-status multi-cluster question).
