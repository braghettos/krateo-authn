# authn — runtime behavior

What the running service exposes and the contracts it depends on. All paths are **literal** (exact
match, see [architecture.md](architecture.md)); the default listen port is `8082`
(`AUTHN_PORT`, `main.go:48`).

## HTTP endpoints

Every route is registered in `main.go:137-189` and implements `routes.Route`.

| Method | Path            | Purpose | Source |
| ------ | --------------- | ------- | ------ |
| GET    | `/strategies`   | List configured login strategies (for the frontend login page) | `internal/routes/auth/strategies/strategies.go` |
| GET    | `/info`         | Fetch a stored `AuthInfo` by `?name=` | `internal/routes/auth/info/info.go` |
| GET    | `/health`       | Liveness/readiness; returns `{name,version}` once healthy, else `503` | `internal/routes/health/health.go` |
| GET    | `/basic/login`  | HTTP Basic login → kubeconfig (+JWT) | `internal/routes/auth/basic/login.go` |
| POST   | `/ldap/login`   | LDAP login (JSON body) → kubeconfig (+JWT) | `internal/routes/auth/ldap/login.go` |
| GET    | `/oauth/login`  | OAuth2 code exchange → kubeconfig (+JWT) | `internal/routes/auth/oauth/login.go` |
| GET    | `/oidc/login`   | OIDC code exchange → kubeconfig (+JWT) | `internal/routes/auth/oidc/login.go` |

Note: `internal/routes/list.go` defines a `/list` route (OAuthConfigs only), but it is **not
registered** in `main.go` and so is not served at this tag.

### `/strategies`

Returns a JSON array of `{kind, name?, graphics?, path, extensions?}` (`strategies.go:191-197`).
- `basic` appears **only if at least one `User` CR exists** (`strategies.go:58-64`).
- `oidc`/`ldap`/`oauth` produce one entry per CR. Missing `graphics` is filled with a default
  (`key` icon, "Login with <Kind>", white/black, `support.go:5-19`).
- `oidc` and `oauth` entries carry `extensions.authCodeURL` / `extensions.redirectURL` so the
  frontend can start the provider redirect (`strategies.go:122-131`, `177-186`).

### The login response contract

All four login routes converge on `encode.Success` (`encode/success.go:18-53`) and return:

```json
{
  "accessToken": "<JWT, omitted if no signing key>",
  "user":   { "displayName": "...", "username": "...", "avatarURL": "..." },
  "groups": ["..."],
  "data":   <base64-free JSON kubeconfig (Kind: Config)>
}
```

- `data` is the per-user kubeconfig minted by the generator (`config/build.go:115-147`).
- `accessToken` is present only when `JWT_SIGN_KEY` is set; default JWT lifetime is 8h if no
  explicit duration (`success.go:32-46`).
- `/basic/login?d` instead returns the kubeconfig as a file download (`Content-Disposition`,
  `basic/login.go:94-97` → `encode/attach.go`).

### Login inputs per strategy

- **basic** — HTTP `Authorization: Basic` header; `401` with `WWW-Authenticate` if absent
  (`basic/login.go:65-70`). Password compared in plaintext against the Secret keyed by
  `User.spec.passwordRef` (`basic/login.go:107-124`).
- **ldap** — `?name=<LDAPConfig>` + JSON body `{"username","password"}` (`ldap/login.go:73-130`).
  Error mapping: not-found → `404`, multiple entries → `300`, other → `403`
  (`ldap/login.go:102-109`).
- **oauth** — `?name=<OAuthConfig>` + `X-Auth-Code` header (`oauth/login.go:71-85`). Code is
  exchanged via `golang.org/x/oauth2`; token type must be `bearer` when a RESTAction is configured
  (`oauth/login.go:105-110`).
- **oidc** — `?name=<OIDCConfig>` + `X-Auth-Code` header. Code is POSTed to the token endpoint;
  ID-token claims (`preferred_username`, `name`, `picture`, `email`, `groups`) are read, and the
  UserInfo endpoint is called only for the claims the ID token omits (`oidc/support.go:112-198`).
  `groups` is never sourced from UserInfo (`oidc/support.go:148`).

## CRDs it reads

authn owns four namespaced CRDs (group `*.authn.krateo.io`, version `v1alpha1`), rendered under
`crds/`. It only **reads** them at request time — there is no controller that writes status.

### `User` (`basic.authn.krateo.io`) — `apis/authn/basic/v1alpha1/types.go`
- `spec.passwordRef` (`SecretKeySelector`, required) — Secret holding the password.
- `spec.displayName`, `spec.avatarURL`, `spec.groups[]`.

### `LDAPConfig` (`ldap.authn.krateo.io`) — `apis/authn/ldap/v1alpha1/types.go`
- `spec.dialURL` (required), `spec.baseDN` (required).
- `spec.bindDN` + `spec.bindSecret` (optional; omit for anonymous bind).
- `spec.tls` (optional bool — `StartTLS` with `InsecureSkipVerify`, see [gotchas.md](gotchas.md)).
- `spec.graphics` (optional).

### `OIDCConfig` (`oidc.authn.krateo.io`) — `apis/authn/oidc/v1alpha1/types.go`
- `spec.clientID`, `spec.clientSecret` (`SecretKeySelector`), `spec.redirectURI`.
- `spec.discoveryURL`, `spec.authorizationURL`, `spec.tokenURL`, `spec.userInfoURL`,
  `spec.additionalScopes`.
- `spec.restActionRef` (optional `ObjectRef` — enrich the identity via snowplow).
- `spec.graphics` (optional).

### `OAuthConfig` (`oauth.authn.krateo.io`) — `apis/authn/oauth/types.go`
- `spec.clientID`, `spec.clientSecretRef`, `spec.authURL`, `spec.tokenURL`, `spec.redirectURL`,
  `spec.scopes[]`.
- `spec.authStyle` (optional int; `0` = auto-detect how creds are sent).

Shared types (`Graphics`, `ObjectRef`, `SecretKeySelector`) — `apis/core/core.go`.

## Integration contracts

### Kubernetes CSR API (the cluster)
On every successful login the generator creates a `CertificateSigningRequest` with signer
`kubernetes.io/kube-apiserver-client` and **self-approves it** (`config/gen.go:36-64`,
`certs.go:171-173`). This requires the broad CSR RBAC in `manifests/rbac.csr.yaml` (create / get /
list / watch / approve / delete / update on `certificatesigningrequests`, plus `approve` on the
signer). The minted cert's CN = username and O = groups become the caller's Kubernetes identity.

### snowplow (identity enrichment)
When an `OIDCConfig`/`OAuthConfig` has a `restActionRef`, authn enriches the identity by calling
snowplow's `/call` endpoint for that RESTAction (`internal/helpers/restaction/resolver.go:25-65`),
authenticating with the long-lived `authn` service JWT (`main.go:153-161`). Two resolution paths:
- **`Resolve`** — calls the RESTAction directly, passing the user's bearer token as an `extras`
  param (`resolver.go:25-65`).
- **`LegacyResolve`** — fallback used when `Resolve` doesn't return a usable `name`: it deep-copies
  the RESTAction + its endpoint Secrets per-user, calls it, then deletes the copies
  (`resolver.go:67-185`). The returned `status` map may override `name`, `email`,
  `preferredUsername`, `groups`, `avatarURL` (`oidc/support.go:225-315`); any other key is rejected
  (`checkKeys` default case, `oidc/support.go:309-311`).

The snowplow base URL comes from `SNOWPLOW_SERVICE_HOST`/`SNOWPLOW_SERVICE_PORT` or
`URL_SNOWPLOW` (default `http://snowplow.krateo-system.svc.cluster.local:8081`, `main.go:57-65`).

### Stored AuthInfo (the `/info` endpoint)
Each `Generate` persists an `AuthInfo` (cert/key/CA/server) via the storage layer
(`config/build.go:150-160`); `/info?name=` reads it back (`info/info.go:45-79`). `name` is required
(`400` otherwise).

### Frontend
The frontend consumes `/strategies` to render the login page and the login response (`data` +
`accessToken`) to authenticate the user. CORS is enabled by default with `*` origin and the custom
`X-Auth-Code` header allowed (`main.go:192-204`).
