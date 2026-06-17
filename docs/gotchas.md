# authn — gotchas

Real runtime pitfalls, each grounded in the code/config at this tag.

## Routing is exact-match — no trailing slash, no prefix
`routes.Serve` compares `req.URL.Path == route.Pattern()` exactly
(`internal/routes/routes.go:21`). `/basic/login/` (trailing slash) or any sub-path returns `404`;
hitting a registered path with the wrong verb returns `405` with an `Allow` header
(`routes.go:36-39`). There is no router middleware to normalize paths.

## `/list` is defined but not served
`internal/routes/list.go` implements a `/list` route, but it is **not** appended to the route table
in `main.go`. Don't document or rely on `/list` as a live endpoint at this tag — only the seven
routes wired in `main.go:137-189` are served.

## CSR RBAC is mandatory — and broad
Every login mints a client cert via the Kubernetes CSR API and **self-approves** it
(`config/gen.go:36-64`). Without the ClusterRole in `manifests/rbac.csr.yaml` (create/get/list/
watch/approve/delete/update on `certificatesigningrequests` + `approve` on signer
`kubernetes.io/kube-apiserver-client`) every login fails at cert generation. The shipped binding
targets namespace `demo-system` (`manifests/rbac.csr.yaml`) — the chart must bind authn's real
ServiceAccount/namespace, or approval silently 403s.

## Existing CSR is deleted, not reused
If a CSR with the computed name already exists, the generator **deletes and recreates** it
(`config/gen.go:41-55`) rather than reusing the prior cert. Repeated logins for the same user churn
CSR objects; concurrent logins for the same user can race on that delete/recreate.

## LDAP TLS skips verification
With `spec.tls: true`, the LDAP `StartTLS` call uses `InsecureSkipVerify: true`
(`internal/routes/auth/ldap/support.go:70-74`). TLS here gives transport encryption but **no
certificate validation** — it does not protect against a MITM with a forged cert.

## OIDC ID token is decoded, not cryptographically verified
`decodeJWT` base64-decodes the ID token payload and JSON-parses the claims
(`oidc/support.go:202-223`); it does **not** verify the signature, issuer, audience, or expiry.
Trust rests on the TLS channel to the token endpoint and the preceding code exchange, not on token
verification. Treat any field copied from the ID token accordingly.

## OIDC groups never come from UserInfo
Missing `name`/`email`/`picture`/`preferred_username` claims trigger a UserInfo call, but `groups`
are read **only** from the ID token (`oidc/support.go:141-148`). An IdP that returns groups only on
UserInfo will yield a user with no groups (and therefore no RBAC), unless a `restActionRef` supplies
them.

## RESTAction enrichment can only set five keys
The `status` map snowplow returns may override exactly `name`, `email`, `preferredUsername`,
`groups`, `avatarURL` (`oidc/support.go:225-271`). `checkKeys` rejects **any** other key via its
`default` branch (`oidc/support.go:309-311`), which flips the flow to `LegacyResolve`. A RESTAction
that returns extra top-level status fields will never take the fast path.

## LegacyResolve mutates the cluster mid-login
`LegacyResolve` **creates** a per-user copy of the RESTAction and each referenced endpoint Secret,
calls snowplow, then **deletes** them (`restaction/resolver.go:67-185`). A login that crashes or is
cancelled between create and delete can leave orphaned `<name>-<email>` RESTActions/Secrets behind.
It also requires authn to have write/delete RBAC on `restactions` and Secrets, beyond the read-only
access the other paths need.

## Basic-auth password comparison is plaintext, non-constant-time
`validate` compares `password != string(pwd)` directly (`basic/login.go:122`) against the raw Secret
value — the password is stored in plaintext in the referenced Secret and compared without a
constant-time check.

## snowplow URL resolution has a quirk
The snowplow URL is built eagerly from `SNOWPLOW_SERVICE_HOST`/`PORT` **before** flags are parsed,
so when those env vars are unset it produces the literal `http://:8081`; only then does it fall back
to `URL_SNOWPLOW` (`main.go:57-65`). If `SNOWPLOW_SERVICE_HOST` is empty but `SNOWPLOW_SERVICE_PORT`
is set to something other than `8081`, the fallback check (`== "http://:8081"`) misses and authn
calls a hostless URL. Set `URL_SNOWPLOW` explicitly to be safe.

## No JWT without a signing key
If `JWT_SIGN_KEY` is empty, `encode.Success` omits `accessToken` entirely (`encode/success.go:32`)
and the service-identity token used to call snowplow is signed with an empty key
(`main.go:153-161`). Clients expecting a bearer token, and RESTAction enrichment, both depend on
`JWT_SIGN_KEY` being set.

## Health gates on a flag flipped after listen
`/health` returns `503` until the goroutine sets `healthy=1` (`main.go:244`), and flips it back to
`0` on shutdown (`main.go:256`). It reflects process lifecycle only — it does **not** check
apiserver/snowplow/LDAP reachability, so a "healthy" authn can still fail every login if its
dependencies are down.
