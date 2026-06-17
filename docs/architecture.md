# authn — architecture

How the service is built, traced to the current tree. authn is a single Go binary (`main.go`):
a stateless HTTP server with one in-process route table, no database, and no controller loop. It
reads configuration CRDs from the apiserver on each request and mints client-cert kubeconfigs
through the Kubernetes CSR API.

> Read this as a map. Every claim cites `file:line` in this repo at this tag. If a README or note
> disagrees with the code, the code wins.

## Entry point — `main.go`

`main()` (`main.go:42`) does, in order:

1. **Flags + env.** Every flag has an env fallback via `internal/env` (`main.go:44-70`): port
   (`AUTHN_PORT`, default `8082`, `main.go:48`), CORS toggle (`AUTHN_CORS`, default on,
   `main.go:47`), generated-cert duration (`AUTHN_KUBECONFIG_CRT_EXPIRES_IN`, default `24h`,
   `main.go:49`), cluster name (`main.go:53`), apiserver URL for the generated kubeconfig
   (`main.go:55`), the snowplow URL for RESTAction calls (`main.go:57-65`), the storage namespace
   (`AUTHN_NAMESPACE`, `main.go:66`), the authn service username (`AUTHN_USERNAME`, default `authn`,
   `main.go:68`), and the JWT signing key (`JWT_SIGN_KEY`, `main.go:70`).
2. **Logger.** zerolog to stdout, `info` unless `--debug` (`main.go:88-101`).
3. **Kube rest config.** In-cluster by default, or from `--kubeconfig` (`main.go:117-126`).
4. **Kubeconfig generator.** `kubeconfig.NewGenerator(cfg, …)` (`main.go:128-133`) — the shared
   component every login route uses to mint a per-user kubeconfig.
5. **Route table.** A `[]routes.Route` is assembled (`main.go:137-189`): `strategies`, `info`,
   `health`, and the four login routes (`basic`, `ldap`, `oauth`, `oidc`).
6. **Service JWT.** A 1-year JWT for the `authn` service identity is minted up front
   (`jwtutil.CreateToken`, `main.go:153-161`) and injected into the context the `oauth`/`oidc`
   routes use to call snowplow (`main.go:170-189`).
7. **Handler + CORS.** `routes.Serve(all, log)` (`main.go:191`); if CORS is on, wrapped with a
   permissive `*`-origin handler that allows the custom `X-Auth-Code` header (`main.go:192-204`).
8. **Self-signup.** `signup.Do(...)` (`main.go:233-241`) creates an authn ClientConfig so authn can
   call snowplow's RESTActions as itself.
9. **Serve + graceful shutdown.** `http.Server` with read/write/idle timeouts (`main.go:206-211`),
   served in a goroutine that flips the `healthy` flag (`main.go:243-248`); SIGINT/SIGTERM trigger a
   30s graceful `Shutdown` (`main.go:213-262`).

`Version` and `Build` are `-ldflags`-injected package vars (`main.go:38-39`), surfaced on `/health`.

## The route model — `internal/routes`

The whole HTTP surface is a tiny hand-rolled router, not a framework.

- **`Route` interface** (`internal/routes/routes.go:10-15`): `Name()`, `Pattern()`, `Method()`,
  `Handler()`. Every endpoint is a struct implementing it.
- **`Serve`** (`internal/routes/routes.go:17-43`): exact-path match on `req.URL.Path == Pattern()`;
  if the path matches but the method doesn't, it collects the allowed methods and returns
  `405` with an `Allow` header; otherwise `404`. There is **no path templating and no prefix
  matching** — paths are literal strings.

Each handler is self-contained: it builds its own dynamic/typed client from the `*rest.Config`,
resolves its CRD, and writes JSON through `internal/helpers/encode`.

## The auth packages — `internal/routes/auth/*`

Four login strategies, one package each, all returning a `routes.Route`:

- **basic** (`internal/routes/auth/basic/login.go`): `GET /basic/login`. Reads HTTP Basic creds
  (`login.go:65`), looks up a `User` CR, compares the password against the referenced Secret
  (`validate`, `login.go:107-132`).
- **ldap** (`internal/routes/auth/ldap/login.go`): `POST /ldap/login`. JSON body
  `{username,password}` (`login.go:127-130`); binds + searches the LDAP server
  (`support.go doLogin`).
- **oauth** (`internal/routes/auth/oauth/login.go`): `GET /oauth/login`. Exchanges the
  `X-Auth-Code` header for a token via `golang.org/x/oauth2` (`login.go:95`), optionally enriches
  the user via a snowplow RESTAction (`login.go:104-130`).
- **oidc** (`internal/routes/auth/oidc/login.go`): `GET /oidc/login`. Exchanges the code at the
  token endpoint, decodes the ID token's claims, falls back to the UserInfo endpoint for missing
  claims (`support.go doLogin`, lines 78-200), optionally enriches via RESTAction.

**strategies** (`internal/routes/auth/strategies/strategies.go`): `GET /strategies` enumerates
which strategies are configured by listing the CRDs — basic appears only if ≥1 `User` exists
(`strategies.go:58-64`); each `LDAPConfig`/`OAuthConfig`/`OIDCConfig` becomes an entry with default
`Graphics` filled in (`support.go:12-19`) when the CR omits them.

## The kubeconfig generator — `internal/helpers/kube/config`

The common back-end of every successful login.

- **`Generate(user)`** (`build.go:96-148`): resolves the cluster CA from a ConfigMap if not set
  (`build.go:97-103`), generates the client cert/key (below), persists an `AuthInfo` to storage
  (`build.go:150-160`), and marshals a `Kind: Config` kubeconfig JSON (`build.go:115-147`).
- **`generateClientCertAndKey`** (`gen.go:24-85`): builds a CSR (`kube.NewCertificateRequest` with
  username as CN and groups as O), creates the Kubernetes `CertificateSigningRequest`
  (`gen.go:36-55`), **auto-approves it** (`gen.go:60`), waits for the signed cert (`gen.go:68`),
  and base64-encodes cert + PKCS#1 key. If a CSR with the same name already exists it is **deleted
  and recreated** (`gen.go:41-55`).
- **Signer/usages** (`internal/helpers/kube/certs.go:171-173`): signer
  `kubernetes.io/kube-apiserver-client`, usage `client auth`, with an explicit
  `ExpirationSeconds`. This is why authn needs broad CSR RBAC (see `manifests/rbac.csr.yaml`).

The generated cert's CN/O become the Kubernetes identity the frontend then uses; RBAC is enforced
later by the apiserver, not by authn.

## The response encoder + JWT — `internal/helpers/encode`

`encode.Success` (`success.go:18-53`) wraps the kubeconfig bytes in `{accessToken,user,groups,data}`
and, when a `JwtSingKey` is configured, mints a JWT via `plumbing/jwtutil` (default 8h if no
duration, `success.go:32-46`). `encode.Attach` (`attach.go`) instead streams the kubeconfig as a
file download when the basic route is called with `?d` (`basic/login.go:94-97`).

## API types & CRDs — `apis/`

Four CRD groups under `authn.krateo.io` (`apis/authn/*`), each with a `v1alpha1` package and
generated `zz_generated.deepcopy.go`:

- `basic.authn.krateo.io` → `User` (`apis/authn/basic/v1alpha1/types.go`)
- `ldap.authn.krateo.io` → `LDAPConfig` (`apis/authn/ldap/v1alpha1/types.go`)
- `oidc.authn.krateo.io` → `OIDCConfig` (`apis/authn/oidc/v1alpha1/types.go`)
- `oauth.authn.krateo.io` → `OAuthConfig` (`apis/authn/oauth/types.go`)

Shared field types (`Graphics`, `ObjectRef`, `SecretKeySelector`) live in `apis/core/core.go`. The
rendered manifests are committed under `crds/` and are what this repo publishes. See
[behavior.md](behavior.md) for the field-level contract.

## Dependencies on other Krateo components

authn is **not a standalone fork** — it composes shared Krateo libraries: `plumbing/jwtutil`
(JWTs), `plumbing/signup` (self-signup), `plumbing/context` (access-token context), and it imports
the **snowplow** RESTAction Go types directly (`snowplow/apis/templates/v1`) to copy/resolve
RESTActions (`internal/helpers/restaction/`). RESTAction resolution is an HTTP call to snowplow's
`/call`, not an in-process operation.
