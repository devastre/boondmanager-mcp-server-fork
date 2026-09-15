# OAuth2 (HTTP transport)

The HTTP transport of the BoondManager MCP server is an **OAuth2 protected
resource** (per the [MCP Authorization 2025-06-18 spec][mcp-auth] and
[RFC 9728][rfc-9728]). The stdio transport keeps the existing
JWT / BasicAuth env-var methods documented in the [README](../README.md);
this guide covers HTTP only.

[mcp-auth]: https://spec.modelcontextprotocol.io/specification/2025-06-18/basic/authorization/
[rfc-9728]: https://datatracker.ietf.org/doc/html/rfc9728

## Architecture in one diagram

```
┌──────────────┐  1. discovery (unauth)   ┌────────────────────┐
│  MCP client  │ ─────────────────────►   │   MCP server       │
│ (Claude…)    │ ◄─── auth_server URL ─── │   (/.well-known/…) │
│              │                          └────────────────────┘
│              │
│              │   2. OAuth dance (browser)
│              │ ─────────────────────────► ┌───────────────────┐
│              │ ◄────── access_token ───── │   BoondManager    │
│              │                            │   (auth server)   │
│              │                            └─────────▲─────────┘
│              │   3. MCP request                     │
│              │      Authorization: Bearer <token>   │
│              │ ─────────────────────────► ┌─────────┴─────────┐
│              │                            │   MCP server      │
│              │                            │  (this repo)      │
│              │                            └─────────┬─────────┘
│              │                                      │
│              │                                      │  4. API call
│              │                                      │     Authorization: Bearer <same token>
│              │                                      ▼
│              │                            ┌───────────────────┐
│              │                            │   BoondManager    │
│              │                            │   (resource API)  │
│              │                            └───────────────────┘
└──────────────┘
```

The MCP server **holds no OAuth state**: no `client_secret`, no refresh
token, no per-user storage. It validates that a Bearer token is present
on every request, pushes it into an `AsyncLocalStorage` context, and
forwards it verbatim to BoondManager when tool calls fire. If
BoondManager rejects the token (401), the failure surfaces back through
the MCP response. Multi-tenant by construction — each user's actions are
attributed to *their* Boond identity in Boond's audit log.

---

## 1. Register an App in BoondManager

OAuth2 is configured per *App* in BoondManager:

1. **Administration → Apps → Security tab.**
2. Toggle **OAuth2** on.
3. Add **at least one redirect URL** — the URL your MCP client uses to
   receive the OAuth callback. For local Claude Desktop / Claude Code,
   that's typically a `localhost` or `127.0.0.1` URL on whichever port
   the client listens on (consult your client's docs). For browser-based
   MCP clients, that's the client's own public callback URL.
4. Note the **Client ID**, **Authorization URL**, and **Access Token URL**
   shown by the Security tab. They are what the MCP client needs in step 3.
5. Configure **Authorized APIs** — these become the scopes the App can
   request.

> The MCP server does **not** need the `client_secret`. That secret lives
> with the MCP client (or in your IdP / OAuth broker, depending on how
> you wire authentication into the MCP client).

---

## 2. Run the MCP server

No bootstrap, no login CLI — just start it.

```bash
export MCP_TRANSPORT=http
export MCP_HTTP_HOST=0.0.0.0
export MCP_HTTP_PORT=3000
# Required when behind a reverse proxy so discovery advertises the
# externally-reachable URL.
export MCP_HTTP_PUBLIC_URL=https://mcp.example.com/mcp
npx boondmanager-mcp-server
```

That's it. The server is now:

- accepting OAuth2 Bearer tokens at `https://mcp.example.com/mcp`
- publishing discovery metadata at
  `https://mcp.example.com/.well-known/oauth-protected-resource`
  (also reachable at the path-suffixed variant
  `…/.well-known/oauth-protected-resource/mcp` per RFC 9728 §3.2).

Optional env vars for the discovery metadata:

| Var | Purpose |
|-----|---------|
| `MCP_HTTP_PUBLIC_URL` | Public URL advertised as the OAuth2 resource identifier. Defaults to `http://<host>:<port><path>` — set this whenever you front the server with a reverse proxy. |
| `BOOND_OAUTH_AUTHORIZATION_SERVER` | Issuer URL of the BoondManager authorization server, advertised in `authorization_servers`. Defaults to `https://ui.boondmanager.com`. |
| `BOOND_OAUTH_SCOPES` | Space/comma-separated list of scope hints advertised in `scopes_supported`. Empty = clients negotiate scopes directly with Boond. |
| `BOOND_BASE_URL` | API base URL used when forwarding tool calls. Defaults to `https://ui.boondmanager.com/api`. |

---

## 3. Configure the MCP client

The MCP client discovers the OAuth flow automatically by fetching the
protected-resource metadata. With a spec-compliant MCP client, the
typical sequence is:

1. Client connects to `https://mcp.example.com/mcp`, gets back `401` +
   `WWW-Authenticate: Bearer resource_metadata="…"`.
2. Client fetches the metadata URL, learns the authorization server is
   `https://ui.boondmanager.com`.
3. Client fetches the authorization server's metadata
   (`.well-known/oauth-authorization-server`) — or uses pre-configured
   `authorize` / `token` URLs — and opens the browser for user consent.
4. User authorizes the App in BoondManager. The browser redirects back
   to the client with an `authorization_code`.
5. Client exchanges the code for an `access_token` (and `refresh_token`)
   at the Boond token endpoint, using the App's `client_id` (and
   `client_secret` if non-public).
6. Client retries the MCP request with `Authorization: Bearer <access_token>`.
7. Server forwards the token to Boond on every API call; refresh happens
   entirely client-side.

If your MCP client does **not** support the discovery flow, configure
the BoondManager OAuth endpoints manually using the values from the App
Security tab.

---

## 4. Deploying to production

Because the server is stateless, deployment is trivial: just a long-running
HTTP service behind a TLS-terminating reverse proxy.

### Docker

```bash
docker run -d --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  -e MCP_HTTP_PUBLIC_URL=https://mcp.example.com/mcp \
  --name boondmanager-mcp \
  ghcr.io/fauguste/boondmanager-mcp-server:latest
```

No volume, no secret, no env var that you wouldn't paste into a Slack
channel. Restart-safe (no state).

### docker-compose

The repo ships a ready-to-use `docker-compose.yml`:

```bash
# Optional — only if you need to override MCP_HTTP_PUBLIC_URL etc.
cp .env.example .env
docker compose up -d
docker compose logs -f mcp
```

### Behind a reverse proxy

Set `MCP_HTTP_PUBLIC_URL` to the public HTTPS URL so the
`resource` field in the discovery metadata and the `realm` /
`resource_metadata` parameters in the 401 challenge advertise the
correct externally-reachable URL.

The reverse proxy must forward at least:
- `Authorization` header (Bearer token)
- `Host` header (or set `MCP_HTTP_ALLOWED_HOSTS` to include the public
  hostname)
- `Origin` header for browser-based clients. Setting `MCP_HTTP_PUBLIC_URL`
  (which you need anyway, see above) already puts its origin on the default
  allow-list, so the common proxy setup needs nothing extra; use
  `MCP_HTTP_ALLOWED_ORIGINS` only when browsers reach the server under
  *additional* origins. A request with no `Origin` is always accepted, so
  non-browser clients need nothing, and the discovery document below is
  exempt from the check so the OAuth bootstrap always completes.
- The request body for POSTs

There is nothing to protect at the network layer beyond what the OAuth
token already gates — there is no "service-account" secret stored on the
server that an attacker could steal by reaching the listener.

---

## 4bis. Spec revisions: what integrators should know

The server is a **protected resource** only — it holds no OAuth state and
issues no tokens — so the changes below require **nothing on the server
side**. They matter for the *client* half of the flow, which is where
integrators do their work.

- **Dynamic Client Registration (DCR, [RFC 7591][rfc-7591]) is deprecated
  in the 2026-07-28 spec revision**, in favour of **Client ID Metadata
  Documents (CIMD)**: instead of POSTing a registration request, the client
  uses an HTTPS URL that resolves to its own metadata document as its
  `client_id`. Clients that rely on DCR still work wherever BoondManager
  supports it, but new integrations should prefer CIMD where available.
- **Clients must validate the `iss` parameter** returned on the
  authorization response ([RFC 9207][rfc-9207]) against the issuer they
  started the flow with. This defends against mix-up attacks when a client
  talks to several authorization servers. Purely client-side — the MCP
  server never sees the authorization response.
- The protected-resource metadata this server publishes
  (`/.well-known/oauth-protected-resource`, [RFC 9728][rfc-9728]) is
  unchanged across these revisions, as is the `WWW-Authenticate` challenge.

[rfc-7591]: https://datatracker.ietf.org/doc/html/rfc7591
[rfc-9207]: https://datatracker.ietf.org/doc/html/rfc9207

---

## 5. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `401 Unauthorized` on every MCP request, no `WWW-Authenticate` advertised by the client | MCP client doesn't yet implement the [MCP Authorization spec][mcp-auth]. Configure the BoondManager OAuth endpoints manually. |
| `401 Bearer realm=…` but the client's discovery fails | `MCP_HTTP_PUBLIC_URL` is unset behind a reverse proxy → the metadata advertises `http://0.0.0.0:3000/mcp`, which clients can't reach. Set it to the public HTTPS URL. |
| MCP server logs `BoondManager API 401`, client gets `-32603` | The forwarded access token was rejected by Boond — the user needs to re-authorize. The MCP client should handle this and trigger a new OAuth flow. |
| MCP server logs `No OAuth access token in request context` | Either a stdio code path is being hit on the HTTP transport (bug — file an issue), or `oauthContext.run(...)` was skipped (custom transport modifications). |
| `/.well-known/oauth-protected-resource` returns 404 | Wrong URL. The endpoint sits at the **root** of the public hostname (not under `/mcp`). |
| `scopes_supported` is missing from the metadata | Expected when `BOOND_OAUTH_SCOPES` is empty — clients then negotiate scopes directly with Boond. |

---

## 6. Implementation notes

- **`src/services/oauth.ts`** — `oauthContext` + `hybridContext` (two
  independent `AsyncLocalStorage` instances so the modes can't interfere),
  `extractBearerToken`, `buildProtectedResourceMetadata`,
  `resolveAuthorizationServer`, `resolveAdvertisedScopes`,
  `HYBRID_USER_TOKEN_HEADER`.
- **`src/transports/http.ts`** — tripartite dispatch: `staticAuth →
  hybridAuth → OAuth Bearer`. In hybrid mode, reads `X-Boond-User-Token`
  and wraps the handler in `hybridContext.run({ userToken }, …)`. OAuth
  discovery endpoint is hidden in both `staticAuth` and `hybridAuth` modes.
- **`src/services/boond-client.ts`** — `oauthContextAuth` (OAuth Bearer
  path) and `hybridContextAuth` (hybrid path). `hybridContextAuth` reads the
  `userToken` from `hybridContext`, reads `BOOND_CLIENT_TOKEN` and
  `BOOND_CLIENT_KEY` from env, and calls `buildJwt()` to mint a fresh HS256
  JWT per request.
- **No user secret** is persisted server-side. `BOOND_USER_TOKEN` is never
  stored in the server environment in hybrid mode — it travels per-request.
- Tests: `src/services/oauth.test.ts`, `src/transports/http.test.ts` (hybrid
  accept/reject, no discovery, context isolation),
  `src/services/boond-client.test.ts` (`hybridContextAuth` header, payload,
  errors, concurrent isolation, TTL, `hasHybridEnvCredentials`).

---

## 7. Mode hybride client/serveur (`BOOND_HTTP_HYBRID_AUTH`)

### Quand l'utiliser

| Critère | OAuth Bearer | Statique | **Hybride** |
|---|---|---|---|
| Secrets côté serveur | aucun | USER_TOKEN + CLIENT_TOKEN + CLIENT_KEY | CLIENT_TOKEN + CLIENT_KEY seulement |
| Secrets côté client | access_token OAuth | aucun | **USER_TOKEN** |
| Multi-tenant | ✅ (chaque user son token) | ❌ (credentials partagés) | ✅ (USER_TOKEN par requête) |
| OAuth dance requise | ✅ | ❌ | ❌ |
| Audit Boond | par user Boond | compte de service unique | **par user Boond** |
| Idéal pour | Clients MCP interactifs | CI/CD, pipelines mono-user | Intégrations où le client gère le USER_TOKEN |

### Flux

```
MCP client                              MCP server                   BoondManager API
──────────                              ──────────                   ────────────────
POST /mcp
  X-Boond-User-Token: <userToken>  ──►  lit BOOND_CLIENT_TOKEN (env)
                                        lit BOOND_CLIENT_KEY (env)
                                        buildJwt(userToken, clientToken, clientKey)
                                        ──►  X-Jwt-Client-Boondmanager: <jwt>
                                                                     ◄── réponse JSON:API
  ◄── réponse MCP
```

### Configuration serveur

```bash
export MCP_TRANSPORT=http
export BOOND_HTTP_HYBRID_AUTH=true    # active le mode hybride
export BOOND_CLIENT_TOKEN="..."       # secret serveur — ne pas exposer au client
export BOOND_CLIENT_KEY="..."         # secret serveur — ne pas exposer au client
# Optionnel : TTL JWT (recommandé pour limiter la fenêtre de rejeu)
export BOOND_JWT_TTL_SECONDS=3600
npx boondmanager-mcp-server
```

Variables d'environnement serveur :

| Variable | Obligatoire | Description |
|---|---|---|
| `BOOND_HTTP_HYBRID_AUTH` | ✅ | `true` / `1` / `yes` pour activer |
| `BOOND_CLIENT_TOKEN` | ✅ | Token client BoondManager (resté côté serveur) |
| `BOOND_CLIENT_KEY` | ✅ | Clé HMAC BoondManager (resté côté serveur) |
| `BOOND_JWT_TTL_SECONDS` | — | Durée de vie du JWT en secondes (ajoute `iat`/`exp`) |
| `BOOND_BASE_URL` | — | URL de base de l'API Boond (défaut : `https://ui.boondmanager.com/api`) |

### Configuration client MCP

Le client doit envoyer **`X-Boond-User-Token: <user_token>`** sur chaque requête MCP.

```bash
# Exemple curl
curl -X POST https://mcp.example.com/mcp \
  -H "Content-Type: application/json" \
  -H "X-Boond-User-Token: <votre_user_token_boondmanager>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}'
```

Le `USER_TOKEN` est le token utilisateur BoondManager (celui que l'utilisateur possède et peut stocker localement). Il ne doit **jamais** être envoyé au serveur comme `BOOND_USER_TOKEN` en variable d'environnement — c'est précisément ce que ce mode évite.

### Règle de priorité

Quand plusieurs variables d'activation sont posées simultanément :

```
BOOND_HTTP_STATIC_AUTH=true  →  mode statique  (priorité 1)
BOOND_HTTP_HYBRID_AUTH=true  →  mode hybride   (priorité 2)
aucun des deux               →  mode OAuth     (défaut)
```

### Troubleshooting

| Symptôme | Cause / fix |
|---|---|
| `401` avec `Missing x-boond-user-token header` | Le client n'envoie pas le header `X-Boond-User-Token`. |
| `500` avec `BOOND_CLIENT_TOKEN and BOOND_CLIENT_KEY must both be set` | Variables manquantes côté serveur. |
| `422` de Boond sur les appels API | `USER_TOKEN` invalide ou expiré — le client doit renouveler son token. |
| La discovery OAuth (`/.well-known/…`) retourne `404` | Normal en mode hybride : le endpoint n'est pas exposé pour ne pas induire les clients OAuth en erreur. |
| `500` avec `No USER_TOKEN in hybrid request context` | Bug dans le transport : `hybridContext.run(…)` n'a pas été appelé avant le dispatch MCP. |

