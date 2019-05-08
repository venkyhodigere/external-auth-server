# `external-auth-server`

`eas` (pronounced `eez`) is primarily focused on lowering the barrier to
using various authentication schemes in a kubernetes environment (but it works
with any reverse proxy supporting external/forward auth). `eas` can be
deployed once and protect many services using disperse authentication methods
and providers. The goal is to make enabling authentication as easy as:

1. generating a new `config_token` (see below)
1. adding an `annotation` to an `Ingress` with the `config_token`
1. benefit

# Authentication Plugins

- htpasswd
- LDAP
- OpenID Connect
- oauth2
- request param
- request header

# Features

- works with any proxy server (traefik, nginx, ambassador, etc) that supports
  forward/external auth
- works with any `OpenID Connect` provider (tested predominantly with
  `keycloak` but it should be agnostic)
- only requires 1 installation to service any number of
  providers/configurations/vhosts/domains
- passes tokens to the backing service via headers
- automatically refreshes tokens

# Usage

## Prerequisites

- `eas` must be able to access `OIDC Provider`

- `user-agent` must be able to access `OIDC Provider`
- `user-agent` must be able to access `proxy`
- `user-agent` must be able to access `eas` (if `redirect_uri` is directly
  pointing to `eas` service `/oauth/callback` endpoint)

- `proxy` must be able to access `eas`
- `proxy` must send `X-Forwarded-Host` (localhost:8000) to `eas` in sub-request
- `proxy` must send `X-Forwarded-Uri` (/anything/foo/bar?test=foo) to `eas` in
  sub-request
- `proxy` must send `X-Forwarded-Proto` (http) to `eas` in sub-request
- `proxy` should send `X-Forwarded-Method` (GET) to `eas` in sub-request
- `proxy` must return non `2XX` responses from `eas` to browser
- `proxy` may forward `2XX` auth header `X-Id-Token` to backing service
- `proxy` may forward `2XX` auth header `X-Userinfo` to backing service
- `proxy` may forward `2XX` auth header `X-Access-Token` to backing service
- `proxy` may forward `2XX` auth header `Authorization` to backing service

If running multiple instances (HA) you will need a shared cache/store (see
redis below).

## Launch the server

If running with docker (`docker pull travisghansen/oauth-external-auth-server`)
just launch the container with the appropraite env variables.

```
EAS_CONFIG_TOKEN_SIGN_SECRET="foo" \
EAS_PROXY_ENCRYPT_SECRET="bar" \
EAS_ISSUER_ENCRYPT_SECRET="blah" \
EAS_ISSUER_SIGN_SECRET="super secret" \
EAS_SESSION_ENCRYPT_SECRET="baz" \
EAS_COOKIE_SIGN_SECRET="hello world" \
EAS_COOKIE_ENCRYPT_SECRET="something" \
EAS_PORT=8080 \
node src/server.js
```

## Generate a token

```
# please edit the values in bin/generate-config-token.js to your situation
# ie: issuer disovery URL, client_id, client_secret, etc
# also make sure to use the same secrets used when launching the server
EAS_CONFIG_TOKEN_SIGN_SECRET="foo" \
EAS_PROXY_ENCRYPT_SECRET="bar" \
node bin/generate-config-token.js
```

## Configure your reverse proxy

```
# See full examples in the ./examples/ directory
# particularly nginx has some particular requirements
# NOTE: run over https in production

# traefik
address = http://<oeas server ip>:8080/verify?config_token=<token output from above>

# nginx
proxy_pass "http://<oeas server ip>:8080/verify?redirect_http_code=401&config_token=<token output from above>";
```

## Endpoints

Configure the external auth URL to point to the services `/verify`
endpoint. The URL supports the following query params:

- `config_token=the encrypted configuration token`
- `redirect_http_code=code` (only use with nginx to overcome external auth
  module limitations (should be set to `401`), otherwise omitted)
- `fallback_plugin=plugin index` if all plugins fail authentication which
  plugin response should be returned to the client

If your provider does not support wildcards you may expose `eas` directly and
set the `config_token` `redirect_uri` to the `eas` service at the
`/oauth/callback` path.

## redis

No support for sentinel currently, see `bin/generate-store-opts.js` with further options.

- https://www.npmjs.com/package/redis#options-object-properties

```
EAS_STORE_OPTS='{"store":"redis","host":"localhost"}'
```

## Kubernetes

A `helm` chart is supplied in the repo directly.

```
helm upgrade \
--install \
--namespace=kube-system \
--set configTokenSignSecret=<random> \
--set proxyEncryptSecret=<random> \
--set issuerEncryptSecret=<random> \
--set issuerSignSecret=<random> \
--set sessionEncryptSecret=<random> \
--set cookieSignSecret=<random> \
--set cookieEncryptSecret=<random> \
--set storeOpts.store="redis" \
--set storeOpts.host="redis.lan" \
--set storeOpts.prefix="eas:" \
--set ingress.enabled=true \
--set ingress.hosts[0]=eas.example.com \
--set ingress.paths[0]=/ \
eas ./chart/
```

Annotate a `traefik` ingress

```
ingress.kubernetes.io/auth-type: forward
ingress.kubernetes.io/auth-url: "https://eas.example.com/verify?config_token=CONFIG_TOKEN_HERE"
ingress.kubernetes.io/auth-response-headers: X-Userinfo, X-Id-Token, X-Access-Token, Authorization
```

# Design

Really tring to alleviate the following primary challenges:

- needing to deploy an additional `proxy` (`oauth2_proxy`,
  `keycloak-gatekeeper`, etc)
- static configurations
- issuer/provider specific implementations
- reverse proxy specific implementations
- inability to make complex assertions on the claims/tokens

Development goals:

- maintain original host/port/path for _all_ callbacks to ensure return to the
  proper location (callbacks detected by setting GET param on original URI with
  query stripped)
- signed: ensures only trusted apps/proxies can use the service
- encrypted: allows for identity operators to hide client\_{id,secret} (and
  other configuration options) from reverse proxy operators
- config aud: ensures users cannot use token (cookie) from one
  configuration/site and use it with another

# Challenges

## kong-oidc

- not cache'ing the discovery docs
- does not allow for deeper validation on iss/groups/other attrs/etc
- `redirect_uri` when set on multiple hosts/routes becomes difficult
  (https://github.com/nokia/kong-oidc/issues/118)
- not generic to work with all proxies

## oauth2_proxy

- cumbersome to deploy and intrusive to the overall process (sidecars in
  kubernetes, etc)
- must be deployed unique to each service (ie, new deployment of the proxy for
  each `client_id` and `client_secret` etc)

# TODO

## 0.2.0

- cache jwks keys?
- support better logic for original URI detection `Forwarded` header and `X-Forwarded-For`, etc
- ensure sessions (guid) does not already exist however unlikely
- implement logout (both local and with provider)
- configuration for custom assertions (lodash?)
- allow for built-in assertions (`config_token`)
- allow for run-time (ie: URL params) assertions
- configuration for turning on/off redirects (probably a query param like `redirect_http_code`) (this may simply be a verify_strategy)
- nonce?
- config_token revocation (blacklist specific jti's)
- support for verifying Bearer requests/tokens
- support RSA signing in addition to signing key
- appropriately handle invalid/changed secrets for signing/encryption
- implement proper logger solution
- support self-signed certs
- document proper annotations for common ingress controllers (traefik, nginx, ambassador, etc)
- ~~Authorization header with id_token for kube-dashboard~~
- ~~support static redirect URI (https://gitlab.com/gitlab-org/gitlab-ce/issues/48707)~~
- support for encyprted cookie
- cookie as struct {id: foo, storage_type: cookie|backend}?
- update to 3.x `openid-client`
- replace `jsonwebtoken` with `@panva/jose`
- implement verify_strategy (cookie_only, bearer, cookie+token(s), etc)
- ensure empty body in responses
- server-side `config_token`(s) to overcome URL length limits and centrally manage (provide a `config_token_id` instead)
- redis integration into helm chart

## 0.1.0

- ~~cache discovery/issuer details~~ (this is automatically handled by the client lib)
- ~~support custom issuer endpoints~~
- ~~use key prefix for discovery and sessions~~
- ~~support manual issuer configuration~~
- ~~support client registration~~
- ~~refresh access token~~
- ~~checks to see if refresh token is present or not~~
- ~~configuration to enable refreshing access token~~
- ~~configuration to enable userInfo~~
- ~~configuration to enable refreshing userInfo~~
- ~~configuration for cookie domain~~
- ~~configuration for cookie path~~
- ~~configuration for scopes~~
- ~~proper ttl for cached sessions~~
- ~~state csrf cookie check~~
- ~~support redis configuration~~
- ~~build docker images and publish to docker hub~~
- ~~support static `redirect_uri` for providers that do not support wildcards~~
- ~~support `/oauth/callback` handler for the static `redirect_uri`~~
- ~~fixup refresh_access_token config option name~~
- ~~fixup introspect access_token config option name?~~
- ~~figure out why discovery requests are not being cached by the client~~
- ~~cache issuer and client objects~~
- ~~figure out refresh token when URL has changed~~

## Ideas

- allow per-path and/or per-method checks
  (https://www.keycloak.org/docs/latest/securing_apps/index.html#_keycloak_generic_adapter)

# Links

- https://www.keycloak.org/docs/latest/securing_apps/index.html#_keycloak_generic_adapter
- https://docs.traefik.io/configuration/entrypoints/#forward-authentication
- https://www.getambassador.io/reference/services/auth-service/
- https://github.com/ajmyyra/ambassador-auth-oidc/blob/master/README.md
- https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-openid-connect-code
- https://bl.duesterhus.eu/20180119/
- https://itnext.io/protect-kubernetes-dashboard-with-openid-connect-104b9e75e39c

- https://developer.okta.com/authentication-guide/auth-overview/#authentication-api-vs-oauth-2-0-vs-openid-connect
- https://developer.okta.com/authentication-guide/implementing-authentication/auth-code/

- https://github.com/oktadeveloper/okta-kong-origin-example
- https://connect2id.com/learn/openid-connect
- https://www.jerney.io/secure-apis-kong-keycloak-1/amp/

- https://redbyte.eu/en/blog/using-the-nginx-auth-request-module/
- https://nginx.org/en/docs/http/ngx_http_auth_request_module.html
- https://github.com/openresty/lua-nginx-module#readme
- https://nginx.org/en/docs/varindex.html
- https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- https://forum.nginx.org/read.php?29,222609,222652
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#external-authentication

- https://tools.ietf.org/html/rfc6750#section-3

- https://developer.okta.com/blog/2017/07/25/oidc-primer-part-1
- https://developer.okta.com/blog/2017/07/25/oidc-primer-part-2
- https://developer.okta.com/blog/2017/08/01/oidc-primer-part-3

- https://blog.runscope.com/posts/understanding-oauth-2-and-openid-connect

- https://developers.google.com/identity/protocols/OpenIDConnect

- https://tools.ietf.org/html/rfc6265#section-4.1.1
- Servers SHOULD NOT include more than one Set-Cookie header field in the same response with the same cookie-name.
- ^ why we do not allow setting the cookie on multiple domains

- https://devforum.okta.com/t/oauth-2-0-authentication-and-redirect-uri-wildcards/1015/2

- https://github.com/keycloak/keycloak-gatekeeper
- https://github.com/pusher/oauth2_proxy
