# Krateo AuthN Service

## List available strategies

The `GET /strategies` endpoint shows all available authentication strategies.

> When a particular authentication strategy requires additional configuration parameters, these will be exposed under the _"extensions"_ key. Otherwise this attribute will not be present.

Example:

```sh
$ curl https://api.krateoplatformops.io/authn/strategies
```

```json
[
  {
    "kind": "basic",
    "path": "/basic/login"
  },
  {
    "kind": "ldap",
    "name": "forumsys",
    "path": "/ldap/login"
  },
  {
    "kind": "ldap",
    "name": "openldap",
    "graphics": {
      "icon": "key",
      "displayName": "Login with LDAP",
      "backgroundColor": "#ffffff",
      "textColor": "#000000"
    },
    "path": "/ldap/login"
  },
  {
    "kind": "oauth",
    "name": "github-example",
    "graphics": {
      "icon": "fa-brands fa-github",
      "displayName": "Login with Github",
      "backgroundColor": "#ffffff",
      "textColor": "#000000"
    },
    "path": "/oauth/login",
    "extensions": {
      "authCodeURL": "https://github.com/login/oauth/authorize?client_id=XXXX&redirect_uri=http%3A%2F%2Flocalhost%3A8888%2Fgithub%2Fgrant&response_type=code&scope=read%3Auser+read%3Aorg&state=YYYY",
      "redirectURL": "http://localhost:30080/auth?kind=oauth"
    }
  },
  {
    "kind": "oidc",
    "name": "oidc-example",
    "graphics": {
      "icon": "fa-brands fa-windows",
      "displayName": "Login with Azure",
      "backgroundColor": "#4444ff",
      "textColor": "#ffffff"
    },
    "path": "/oidc/login",
    "extensions": {
      "authCodeURL": "https://login.microsoftonline.com/XXXX/oauth2/v2.0/authorize?client_id=XXXX\u0026redirect_uri=http%3A%2F%2Flocalhost%3A8080%2Foidc%2Fcallbacl\u0026response_mode=query\u0026response_type=code\u0026scope=openid+email+profile+User.Read",
      "redirectURL": "http://localhost:8080/auth?kind=oidc"
    }
  }
]
```

## Authentication

Regardless of the strategy used, the response will always be a json with the following structure:

```json
{
   "code":200,
   "user":{
      "displayName":"John Doe",
      "username":"johndoe",
      "avatarURL":"https://avatars.githubusercontent.com/u/585381?v=4"
   },
   "groups": [
      "devs"
   ],
   "data":{
      "apiVersion":"v1",
      "clusters":[
         {
            "cluster":{
               "certificate-authority-data":"<base64-ca-cert-data>",
               "server":"https://127.0.0.1:51461"
            },
            "name":"krateo"
         }
      ],
      "contexts":[
         {
            "context":{
               "cluster":"krateo",
               "user":"johndoe"
            },
            "name":"krateo"
         }
      ],
      "current-context":"krateo",
      "kind":"Config",
      "users":[
         {
            "user":{
               "client-certificate-data":"<base64-user-cert-data>",
               "client-key-data":"<base64-user-cert-key-data>"
            },
            "name":"johndoe"
         }
      ]
   }
}
```

### Login with Basic Authentication

The Authorization header field is constructed as follows:

- username and password are combined with a single colon
  - this means that the username itself cannot contain a colon

- the resulting string is encoded using a variant of Base64 (+/ and with padding)

- the authorization method and a space character (e.g. "Basic ") is then prepended to the encoded string.

For example, if the username is Aladdin and the password is open sesame, then the field's value is the Base64 encoding of Aladdin:open sesame, or QWxhZGRpbjpvcGVuIHNlc2FtZQ==

Then the Authorization header field will appear as: _Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==_

Example:

```sh
curl https://reqbin.com/echo
   -H "Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ=="
```

### Login with OAuth Authorization Code Flow

> Let's take _Github_ as example, the same concept applies to all authentication systems of this type (authorization code flow).

With a valid [authorization code grant](https://www.oauth.com/oauth2-servers/access-tokens/authorization-code-request/) invoke the related endpoint `path`, passing the `name` as query string parameter.

Example:

```sh
$ curl -H "X-Auth-Code: $(AUTH_CODE)" \
    https://api.krateoplatformops.io/authn/oauth/login?name=github-example
```

The `RestActionRef` is mandatory because, unlike OIDC, OAuth2 returns only the bearer token and no information regarding the user, therefore, the RESTAction is required to compile all fields. See the [RESTAction Configuration](#restaction-configuration) section for more information.

### Login with LDAP

To login using LDAP credentials must be sent as JSON using POST:

```sh
$ curl -X POST "https://api.krateoplatformops.io/authn/ldap/login?name=openldap" \
   -H 'Content-Type: application/json' \
   -d '{"username":"XXXXXX","password":"YYYYYY"}'
```

### Login with OIDC

To login using OIDC credentials, the authorization code must be sent throught the `X-Auth-Code` header field:

```sh
$ curl -H "X-Auth-Code: $(AUTH_CODE)" \
    https://api.krateoplatformops.io/authn/oidc/login?name=oidc-example
```

The authn application supports the Discovery endpoint. If you provide a Discovery endpoint the values for `authorizationURL`, `tokenURL` and `userInfoURL` are ignored and overwritten. If you do not provide a Discovery endpoint, the values for `authorizationURL`, `tokenURL` and `userInfoURL` are used.

To obtain proper groups mappings you need to configure the ID Token response on the application side. Likewise for the profile picture. Examples are listed below for Azure and KeyCloak. Alternatively, you can use the RESTAction. See the [RESTAction Configuration](#restaction-configuration) section for more information.

#### Azure
Azure can be configured to authenticate users through OIDC ([Official Azure Documentation for OIDC](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc)). To achieve this, you need to create a new app registration as follows:

On Azure:
 - Go to "App registrations" and then hit "New registration";
 - Configure the display name, account types and Redirect URI. The redirect URI must point to Krateo's frontend with an HTTPS endpoint and the path `/auth/oidc`;
 - Create a client secret in "Certificates & secrets", save the value of the secret now as it cannot be visualized afterwards;
 - In the "Authentication" menu, find and activate `Access tokens` and `ID tokens`;
 - In the "API permissions" menu, add the following: `openid`, `email`, `profile`, `User.Read` and `User.ReadBasic.All`;

To obtain groups you can do one of the following:
- Include them in the OIDC ID Token response: in the "Manifest" menu in the Azure portal, modify the value `groupMembershipClaims` to `All` ([Official Azure documentation for the groupMembershipClaims](https://learn.microsoft.com/en-us/entra/identity-platform/reference-app-manifest#groupmembershipclaims-attribute));
- Use the following RESTAction and place it in the `RestActionRef` of the OIDC custom resource:
```yaml
apiVersion: templates.krateo.io/v1
kind: RESTAction
metadata:
  name: test-rest-action
  namespace: krateo-system
spec:
  api:
  - name: groups
    verb: GET
    headers:
    - 'Accept: application/json'
    path: "v1.0/me/memberOf?$select=displayName"
    endpointRef:
      name: azure-entra
      namespace: krateo-system
    filter: .groups.value | map(.displayName)
```
To obtain groups through the rest action, add `Directory.Read.All` in the `additionalScopes`. There is a more complex example for the Microsoft Entra API that also uses pagination in the testdata folder (requires Authn >= 0.22.0).

On AuthN:
 - To obtain the user avatar/profile image include `User.Read` in the `additionalScopes` field of the OIDCConfiguration custom resource;
 - You can now configure the Authn's CR by using Azure discovery URL, which will be in the following format:
 ```
https://login.microsoftonline.com/<your-tenant-id>/v2.0/.well-known/openid-configuration
 ```

Scopes in the `additionalScopes` field can be simply separated with a space. Example for Azure groups with the RESTAction:
```yaml
  additionalScopes: User.Read Directory.Read.All
```

##### Troubleshooting
If you do not get the correct groups in the AuthN response, please verify your Azure OIDC configuration: the manifest value "groupMembershipClaims:All" adds in the JWT ID Token the value "groups", which contains an array of UIDs of the groups the user belongs. You can check the JWT ID Token returned by Azure by simulating the calls to the Authorization and Token endpoints through Postman or curl. The exact endpoints are contained in the "well-known" endpoint (Authorization and Token).

On Azure, set the redirect URI to, for example, "http://localhost:8080" for testing without HTTPS, then open the following page in a web browser:
```
https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize?client_id=<client-id>&response_type=code&redirect_uri=http://localhost:8080&response_mode=query&scope=openid email profile User.Read
```
Login, then Azure will redirect you to the redirect URI, which will error out, however, there will be a "code" query parameter in the URL. Copy this code parameter being careful not to copy other query parameters. Through cURL or Postman perform a POST to:
```
https://login.microsoftonline.com/<tentant-id>/oauth2/v2.0/token
```
with the following body values:
```
client_id=<client-id>
client_secret=<client-secret>
code=<the code from the redirect url>
redirect_uri=http://localhost:8080
grant_type=authorization_code
```
You can then decode the JWT and verify that "groups" is present with the [Microsoft offline decoder](https://jwt.ms/).

#### KeyCloak
To obtain groups, add a custom mapper of type "Group Membership" and give it the Token Claim Name "groups", uncheck `Full group path`. Add `groups` into the `additionalScopes` field of the OIDCConfiguration custom resource.
To obtain the user avatar/profile image, go to the realm settings, then "User profiles" tab, "Create Attribute", and add one with the name `picture`. Set the profile picture for the user to a URL pointing to a picture. Keycloak will now return the avatar during authentication.


### Login with a Kubernetes ServiceAccount (intra-service auth)

> This strategy is for **backend services**, not human users. It lets an in-cluster Krateo service
> authenticate to authn with its **own** Kubernetes ServiceAccount token (no password, no browser
> redirect) and receive the same JWT + kubeconfig the other strategies return. It is not listed by
> `GET /strategies` (there is no login button for it).

The caller mounts a **projected** ServiceAccount token (`TokenRequest`, audience `authn`, short TTL)
and presents it as a Bearer credential:

```sh
$ curl -X POST "https://api.krateoplatformops.io/authn/serviceaccount/login" \
   -H "Authorization: Bearer $(cat /var/run/secrets/tokens/authn-token)"
```

authn validates the token via the Kubernetes **`TokenReview`** API (checking the expected audience —
default `authn`, configurable with `--serviceaccount-audience`), then looks up a `ServiceAccount`
mapping whose `serviceAccountRef` matches the caller's SA. The mapping is the **allowlist**: an SA
with no matching `ServiceAccount` resource is rejected. The issued identity's username is the
mapping's `metadata.name` and its groups are `spec.groups` (which become the client certificate's
`O=`, scoped by standard Kubernetes RBAC).

```yaml
apiVersion: serviceaccount.authn.krateo.io/v1alpha1
kind: ServiceAccount
metadata:
  name: composition-dynamic-controller   # == the issued username
  namespace: krateo-system               # the authn operator namespace
spec:
  serviceAccountRef:                      # the k8s SA allowed to exchange its token
    namespace: krateo-system
    name: composition-dynamic-controller
  groups: [ "krateo:services" ]           # → issued cert O= → standard k8s RBAC
  displayName: Composition Dynamic Controller
```

> authn's ServiceAccount must be allowed to `create` `tokenreviews.authentication.k8s.io`, and the
> `ServiceAccount` mapping must live in the authn operator namespace.

## RESTAction Configuration
The `RESTActionRef` field in the OAuth2 and OIDC configs is mandatory and optional, respectively. It is used to compile the following fields, used to build the Kubernetes certificate required for authentication:
- `name`: string
- `email`: string
- `preferredUsername`: string - mandatory
- `avatarURL`: string
- `groups`: []string - mandatory

In the case of OAuth2, these fields are needed to compile the certificate and the ones marked as such are mandatory. They have to be included as top level fields in the RESTAction response. See the testdata folder for a Github example. In the OIDC case, these fields are all optional, and if included will overwrite the information obtained from the id token and the userinfo endpoint.

The authentication for the endpoints of the RESTAction is automatically set to bearer token, using the token obtained from the OAuth2/OIDC authentication. This token is passed to the RESTActio as a parameter, and can be used in the `Authorization` header as follows:
```yaml
    headers:
    - "${ \"Authorization: Bearer \" + .token }"
```
See [oidc-azure-pagination](./testdata/oidc-azure-pagination.yaml), [oidc-azure](./testdata/oidc-azure.yaml) and [oauth](./testdata/oauth.yaml) for more examples.

If the RESTAction does not accept a token parameter, then it will temporarily set the token in the respective endpoint.

## Graphics Configuration
The OAuth2 and OIDC authentication methods also support a `graphics` object that allows to configure how the button for the redirect to the authentication provider portal is visualized in the frontend login screen.
```yaml
  graphics:
    icon: # icon name from the fontawesome library, for icons like github and windows, use "fa-brands fa-github" or "fa-brands fa-windows"
    displayName: # text to be visualized on the button
    backgroundColor: # color of the button in hexadecimal, e.g., #0022ff
    textColor: # color of the text in the button, also in hexadecimal, e.g., #0022ff
```