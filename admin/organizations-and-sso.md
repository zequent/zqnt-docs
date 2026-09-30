# Organizations, Users & Single Sign-On

This page covers standing up a new deployment's first organization and users, and connecting an
organization to its own OIDC identity provider (SSO) instead of the platform's built-in
email+password login.

Every platform user belongs to exactly one organization (except `system_admin`, a platform
operator role that isn't scoped to any one organization). Local email+password authentication is
always the default — SSO is opt-in, per organization, and never required.

## 1. Bootstrapping the first `system_admin`

The first time the Connector Service starts against an empty database, it seeds exactly one
`system_admin` user automatically. There is no manual step for this.

| Variable | Applies to | Default | Notes |
| --- | --- | --- | --- |
| `AUTH_SYSTEM_ADMIN_EMAIL` | Connector Service | `system-admin@zequent.local` | Only read on that first boot |

The generated password is printed once, to the Connector Service's own startup log, and nowhere
else — it is not recoverable after that. Capture it from your deployment's log aggregator on first
boot.

Log in as this user against the Admin Console API to get a bearer token for every step below:

```bash
curl -s -X POST http://localhost:8005/api/admin-console/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"system-admin@zequent.local","password":"<the generated password>"}'
```

The response is `{"token": "...", "expiresAt": "...", "organizationId": null, "roles": ["system_admin"]}`.
Tokens are valid for **1 hour**; log in again once one expires. Use it as `Authorization: Bearer
<token>` on every request below.

## 2. Creating an organization

An organization is created from its license. Zequent issues each license for one organization,
by name, and the license carries that organization's id. In the Admin Console, as `system_admin`:

1. **Organizations → Create from license**.
2. Paste the license key, or drop the organization's `.zqnt` license file onto the field.

The organization is created with exactly the id and name the license names, and its license is
active immediately — no redeploy, no restart. If that organization already exists on this
installation, nothing is created and its license is activated instead. The license names the
organization only once: afterwards the name is yours to change in the console.

To license an organization that already exists (for example the default organization, or one
created with **Add Organization**), ask Zequent for a license issued for that organization's id,
then use **Activate license** on the organization. An organization admin can activate their own
organization's license on the **License** screen. A license issued for another organization is
refused, and the message names the organization it belongs to.

The organization's `id` (a UUID) is the `organizationId` every step below needs; the
Organizations screen shows it.

## 3. Deciding local auth vs. SSO for the organization

Every organization defaults to local email+password authentication with no further setup. Only
configure SSO (below) for an organization whose users should authenticate against their own
company identity provider instead.

## 4. Creating additional users

For a **local-auth** organization, create users directly:

```bash
curl -s -X POST http://localhost:8005/api/admin-console/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <system_admin token>" \
  -d '{
    "email": "operator@acme.example",
    "organizationId": "<organizationId from step 2>",
    "roles": ["operator"]
  }'
```

This consumes one seat on the organization's active license and returns a one-time temporary
password in the response — it is not shown again, so hand it to the user immediately. Valid roles
are `org-admin`, `operator`, `approver`, `viewer` (`system_admin` is granted by direct database
action only, not through this endpoint).

For an **SSO** organization, you don't create users this way at all — see below.

## 5. Configuring SSO for an organization

Point an organization at its own OIDC identity provider (Okta, Microsoft Entra ID, Google
Workspace, Keycloak, Auth0, or any standards-compliant OIDC provider). This is `system_admin`-only.

```bash
curl -s -X PUT http://localhost:8005/api/admin-console/identity-providers \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <system_admin token>" \
  -d '{
    "organizationId": "<organizationId from step 2>",
    "issuerUrl": "https://your-org.okta.com",
    "clientId": "<the OIDC client id your IdP gave you>",
    "clientSecret": "<the OIDC client secret your IdP gave you>",
    "emailDomains": ["acme.example"],
    "roleClaimName": "groups",
    "claimRoleMapping": {
      "Acme-Operators": "operator",
      "Acme-Admins": "org-admin"
    },
    "enabled": true
  }'
```

There is no self-service path yet for an organization to configure its own SSO — this whole
endpoint is `system_admin`-only for now.

| Field | Required | Notes |
| --- | --- | --- |
| `organizationId` | Yes | The organization from step 2 |
| `issuerUrl` | Yes | The IdP's OIDC issuer — its `/.well-known/openid-configuration` document must be reachable at `<issuerUrl>/.well-known/openid-configuration` |
| `clientId` / `clientSecret` | Yes (client secret on first save only) | From your IdP's own OIDC application/client setup. `clientSecret` is write-only — re-saving other fields later without it keeps the existing secret |
| `emailDomains` | Yes | Which email domains route to this IdP on the login screen. A domain can belong to at most one organization |
| `roleClaimName` | No, defaults to `groups` | Which claim on the IdP's ID token carries group/role membership |
| `claimRoleMapping` | No | Claim value → platform role (`org-admin`/`operator`/`approver`/`viewer`). A claim value with no entry here grants no role from it; a user matching none of these gets `viewer` |
| `enabled` | Yes | Set `false` to keep the configuration without activating it |

You must also register the redirect URI below as an **allowed redirect URI** on the OIDC client
you create at the IdP:

| Variable | Applies to | Default | Notes |
| --- | --- | --- | --- |
| `OIDC_REDIRECT_URI` | Admin Console API | `http://localhost:3000/auth/callback` | Set to your deployment's real Admin Console UI origin + `/auth/callback` in production. One value for the whole installation, not per organization |

With SSO configured, a user with a matching email domain never needs to be created up front —
their first successful login through the IdP auto-provisions their platform account, with roles
resolved from `claimRoleMapping` and re-resolved on every subsequent login (so a role change in
your own directory takes effect here automatically). They still consume a license seat on first
login, exactly like a locally-created user does.

### Login flow

The Admin Console UI's login screen handles this automatically — enter an email, and it either
shows a password field or redirects to the configured IdP. If you're integrating against the API
directly:

1. `POST /api/admin-console/auth/discover` with `{"email": "..."}` → `{"mode": "local"}` (show a
   password field, call `/auth/login` as normal) or `{"mode": "oidc", "authorizationUrl": "..."}`
   (redirect the browser there).
2. The IdP redirects the browser back to `OIDC_REDIRECT_URI` with `code`/`state` query parameters.
3. `POST /api/admin-console/auth/oidc/callback` with `{"code": "...", "state": "..."}` → the same
   `{"token", "expiresAt", "organizationId", "roles"}` shape `/auth/login` returns.

## 6. Testing SSO locally with Keycloak

You don't need a real customer IdP to test this end to end — Keycloak runs standalone in Docker and
speaks the same OIDC protocol any real IdP does. This is a **testing convenience only**: it plays
the role of "a customer's own IdP" for local development, the same way you'd use any throwaway test
identity provider. It is not a platform dependency and nothing else needs it running.

```bash
docker run -d --name test-keycloak \
  -p 8180:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

Then, from Keycloak's admin console (`http://localhost:8080` inside the container, `:8180` on your
host — default credentials `admin`/`admin`):

1. Create a realm.
2. Create a client: confidential (not public), Standard Flow enabled, with a redirect URI matching
   your `OIDC_REDIRECT_URI` (`http://localhost:3000/auth/callback` for a local Admin Console UI dev
   server). Note the generated client secret.
3. Add a protocol mapper to that client of type **Group Membership**, claim name `groups`, so the
   ID token actually carries a `groups` claim — Keycloak doesn't include one by default.
4. Create a group (e.g. `Zequent-Operators`) and a test user in that group, with a password.

Then configure the organization exactly as in [step 5](#5-configuring-sso-for-an-organization),
using:

- `issuerUrl`: `http://localhost:8180/realms/<your realm>`
- `clientId` / `clientSecret`: from the client you created
- `emailDomains`: your test user's email domain
- `claimRoleMapping`: `{"Zequent-Operators": "operator"}` (or whatever group/role you chose)

To automate this setup instead of clicking through Keycloak's admin console, export the realm as
JSON (i.e. one client with a `groups` mapper, one group, one user with a password credential) and
start Keycloak with `start-dev --import-realm`, mounting the file to
`/opt/keycloak/data/import/<name>.json` — Keycloak imports it automatically on boot.

Remove the container when you're done: `docker rm -f test-keycloak`.
