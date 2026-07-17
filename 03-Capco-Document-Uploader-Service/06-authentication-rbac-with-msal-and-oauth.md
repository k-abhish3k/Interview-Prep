# 06 — Authentication and RBAC with MSAL and OAuth 2.0

## Where this chapter sits

Chapters 01–02 covered the API surface and the data layer; chapter 05 covered secrets and established
that **Azure AD/Entra ID** is this service's identity provider. This chapter is the piece that sits
between them: how a caller proves who they are (**authentication**, via **OAuth 2.0** and **OpenID
Connect**, using Microsoft's **MSAL** library), and how the service decides what that caller is allowed
to do (**authorization**, via **Role-Based Access Control** — RBAC). This is a confirmed part of how
the service was built, not a hypothetical addition: every path operation that reads or writes a
document sits behind an authenticated, role-checked FastAPI dependency.

## OAuth 2.0 fundamentals: two flows this service actually needs

OAuth 2.0 is an authorization framework — it defines how a client obtains a token that proves it's
allowed to call an API on someone's behalf, without that client ever seeing the resource owner's
password. It defines several **grant types** ("flows"); this service uses two of them, because it has
two categories of caller:

**1. Authorization Code flow with PKCE — for interactive, human callers.** When a bank employee uses a
client application (an admin UI, a portal) to upload or manage documents, that application redirects
the user to Azure AD to authenticate interactively (username, password, MFA), Azure AD redirects back
with an authorization code, and the client exchanges that code for an access token. **PKCE** (Proof Key
for Code Exchange) adds a client-generated secret (`code_verifier`) and its hash (`code_challenge`) to
this exchange specifically so that a public client — a single-page app or mobile app with no way to
keep a client secret confidential — can't have its authorization code intercepted and redeemed by a
different app. PKCE is the modern default for interactive flows, including confidential clients, per
current OAuth security best practice.

**2. Client Credentials flow — for service-to-service callers.** This document-uploader service isn't
only called by interactive bank staff — it's also plausibly called by other backend systems (a batch
import job, another microservice that needs to attach a generated report as a document) with no human
present to authenticate. The Client Credentials flow lets a registered *application* — identified by its
own client ID and secret or certificate — authenticate directly to Azure AD and receive an **app-only
access token** representing the application itself, not any particular user. The token's role claims
(below) then come from **app roles** assigned to the calling application's service principal, rather
than to a human user.

Both flows terminate in the same place from this API's point of view: an incoming request carries a
**bearer access token** in the `Authorization: Bearer <token>` header, and the API's job is to validate
it and extract claims from it — it doesn't need to know or care which flow produced that token.

## OpenID Connect: identity layered on top of OAuth

OAuth 2.0 by itself answers "is this request authorized to call this API" — it doesn't standardize
"who is this person." **OpenID Connect (OIDC)** is a thin identity layer on top of OAuth 2.0 that adds
an **ID token** (a JWT containing the authenticated user's identity claims — `sub`, `name`, `email`,
`oid`) alongside the OAuth access token. Azure AD is an OIDC provider, so the same authentication
exchange that gets this service its access token also gets the client an ID token it can use to display
"logged in as ..." without a separate API call. For this service's own backend validation, the access
token (not the ID token) is what matters — the ID token is consumed by the client application, the
access token is what's presented to this API and validated on every request.

## MSAL: acquiring tokens against Azure AD

**MSAL** (Microsoft Authentication Library) is Microsoft's official library for implementing the OAuth
2.0/OIDC flows above against Azure AD/Entra ID, available for Python as the `msal` package. It handles
the flow mechanics, token caching, and refresh so application code doesn't hand-roll HTTP calls to
Azure AD's token endpoint.

**`PublicClientApplication` vs. `ConfidentialClientApplication`.** MSAL exposes two client types
matching the two categories of caller above:

- **`PublicClientApplication`** — for clients that can't securely hold a secret (desktop apps, mobile
  apps, single-page apps). Used for the interactive Authorization Code + PKCE flow. It has no client
  secret configured; PKCE is what protects the code exchange instead.
- **`ConfidentialClientApplication`** — for clients that *can* securely hold a secret or certificate
  (a backend service, a daemon). Used for the Client Credentials flow, and also usable for the
  Authorization Code flow when the calling application is itself a confidential client (e.g., a
  server-rendered admin portal that can keep a client secret out of the browser).

```python
import msal

# Service-to-service caller: another backend system acquiring an app-only token
confidential_app = msal.ConfidentialClientApplication(
    client_id=settings.client_id,
    client_credential=settings.client_secret,   # or a certificate, preferred for production
    authority=f"https://login.microsoftonline.com/{settings.tenant_id}",
)

result = confidential_app.acquire_token_for_client(
    scopes=["api://doc-uploader-svc/.default"],
)
access_token = result["access_token"]
```

**Token caching.** MSAL maintains an in-memory (or pluggable, persistent) token cache keyed by scope
and identity, and `acquire_token_silent(...)` checks that cache before making a network call to Azure
AD — returning a still-valid cached token instantly, or transparently using a cached refresh token to
get a new access token if the old one expired. This matters operationally: without it, naive code would
re-authenticate against Azure AD on every single outbound call, adding latency and needlessly hammering
the identity provider.

**On-behalf-of vs. app-only tokens.** When this service itself needs to call a *downstream* API using
the calling user's identity (e.g., calling Microsoft Graph to resolve the uploader's department, chapter
05) rather than its own app identity, MSAL's **On-Behalf-Of (OBO) flow**
(`acquire_token_on_behalf_of(...)`) exchanges the token this service received for a new token scoped to
that downstream API, still representing the original user — as opposed to `acquire_token_for_client`,
which gets a token representing the service itself with no user context at all. Which one to use depends
on whether the downstream call should be attributed to "this service" or "the user this service is
acting on behalf of" — a real distinction for audit logging in a banking context.

## How the FastAPI backend validates an incoming token

MSAL is what a *client* uses to acquire a token; this service, as the **resource server**, does the
opposite job — it receives a bearer token on every request and must independently verify it's valid
before trusting anything in it. A caller-supplied token is never trusted just because it's
well-formed JSON; validation means:

1. **Signature validation against Azure AD's JWKS endpoint.** Azure AD publishes its current signing
   keys at a well-known JWKS (JSON Web Key Set) URL
   (`https://login.microsoftonline.com/{tenant_id}/discovery/v2.0/keys`). The service fetches (and
   caches, with periodic refresh) that key set and verifies the token's signature against it — proving
   the token was actually issued by Azure AD and hasn't been tampered with, not just that it decodes as
   valid JSON.
2. **Claim checks**: `aud` (audience) must match this API's own registered App ID URI — a token issued
   for a *different* API must be rejected even if it's validly signed by the same tenant. `iss`
   (issuer) must match the expected `https://login.microsoftonline.com/{tenant_id}/v2.0` for this
   specific tenant — relevant given the multi-tenant HSBC/Bank of America context, since a token issued
   by the wrong tenant's Azure AD must never be accepted. `exp` (expiry) must be in the future — an
   expired token is rejected regardless of how recently it was valid.
3. **Extracting the `roles` claim** — covered below.

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt          # PyJWT; python-jose is an equally common alternative
from jwt import PyJWKClient

bearer_scheme = HTTPBearer()
jwks_client = PyJWKClient(
    f"https://login.microsoftonline.com/{settings.tenant_id}/discovery/v2.0/keys"
)

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
) -> "AuthenticatedUser":
    token = credentials.credentials
    try:
        signing_key = jwks_client.get_signing_key_from_jwt(token)
        claims = jwt.decode(
            token,
            signing_key.key,
            algorithms=["RS256"],
            audience=settings.api_app_id_uri,
            issuer=f"https://login.microsoftonline.com/{settings.tenant_id}/v2.0",
        )
    except jwt.PyJWTError as exc:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED,
                             detail="invalid or expired token") from exc

    return AuthenticatedUser(
        object_id=claims["oid"],
        email=claims.get("preferred_username", claims.get("upn", "")),
        roles=claims.get("roles", []),
        tenant_id=resolve_tenant_from_claims(claims),   # chapter 05 — never client-supplied
    )
```

This is exactly the `Depends(get_current_user)` used throughout chapter 02's router — a request with no
token, an expired token, a token for the wrong `aud`, or a token signed by a key not in Azure AD's
current JWKS all raise a `401` before any route logic runs, via the same dependency-injection mechanism
used for the database session.

## RBAC design: Azure AD App Roles and the `roles` claim

**RBAC** here is implemented with **Azure AD App Roles**, not a custom roles table this service invents
and maintains itself:

1. **App Roles are defined in the app registration's manifest** — this service's Azure AD app
   registration declares roles like `DocumentUploader.Write`, `DocumentUploader.Read`, and
   `DocumentUploader.Admin`, each with a display name, description, and an internal value string.
2. **Roles are assigned to users or groups** (or, for the client-credentials case, to other
   applications' service principals) in Azure AD's Enterprise Applications blade — an administrator
   grants a specific user or an Azure AD security group the `DocumentUploader.Write` role, rather than
   this service maintaining its own user-to-permission mapping.
3. **Assigned roles appear as a `roles` claim in the validated access token** — an array of strings,
   e.g. `"roles": ["DocumentUploader.Write"]`. Because this claim comes from Azure AD itself as part of
   a signature-validated token, the service doesn't need a separate database lookup to know a caller's
   roles; they arrive already resolved, as part of proving who the caller is.

**The FastAPI enforcement dependency:**

```python
from fastapi import Depends, HTTPException, status

def require_role(required_role: str):
    async def _check(current_user: "AuthenticatedUser" = Depends(get_current_user)) -> None:
        if required_role not in current_user.roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"missing required role: {required_role}",
            )
    return _check

@router.delete("/{doc_id}", status_code=204)
async def delete_document(
    doc_id: int,
    db: Session = Depends(get_db),
    current_user: "AuthenticatedUser" = Depends(get_current_user),
    _: None = Depends(require_role("DocumentUploader.Write")),
):
    ...
```

`require_role(...)` is a **dependency factory** — it returns a dependency function closed over the
specific role that endpoint requires, so `DELETE` can demand `DocumentUploader.Write` while `GET` only
demands that the caller be authenticated at all (no `require_role` dependency, or one requiring only
`DocumentUploader.Read`). This is the FastAPI-idiomatic way to express "authentication is required
everywhere, but authorization requirements vary per endpoint," without an if/else ladder inside every
route body.

## RBAC vs. tenant isolation: a precise distinction, and a common interviewer trap

It's tempting to describe a role like `HSBC.Uploader` (as opposed to a generic `DocumentUploader.Write`)
as "how tenant isolation is enforced," and interviewers will sometimes probe exactly this to see if a
candidate conflates two genuinely different mechanisms:

- **RBAC (via App Roles and the `roles` claim) answers "what is this caller allowed to *do*"** — can
  they write documents, only read them, or administer the service. A per-tenant role naming convention
  (`HSBC.Uploader` vs. `BofA.Uploader`) is one legitimate way roles can *interact* with tenant
  boundaries — for instance, to distinguish "an HSBC employee with upload rights" from "an HSBC employee
  with read-only rights," scoped within their own tenant.
- **Tenant isolation (via the `tenant_id` column and the repository-layer filter, chapter 05) answers
  "whose data can this caller see at all."** This is enforced at the data-access layer — every query is
  scoped to the `tenant_id` resolved from the caller's authenticated identity — not by role membership.

The trap: relying on role naming *alone* to prevent cross-tenant access (e.g., trusting that a
`BofA.Uploader` role can never see HSBC rows purely because of how the role is named, with no
enforcement in the query layer) is fragile — it depends on every future role and every future query
respecting a naming convention rather than a structural boundary. If a new role is added, or a query is
written that doesn't check the role name correctly, tenant isolation silently breaks. This service's
actual design keeps these as two independent, layered checks: RBAC decides *capability*
(`Depends(require_role(...))`), and the repository layer decides *data scope* (the `tenant_id` filter,
resolved from the token's tenant claim, applied structurally to every query regardless of which role the
caller holds). Getting this distinction right — and stating plainly that role claims are not a substitute
for row-level tenant enforcement — is exactly the kind of precision a banking-context interviewer is
listening for.

## Tying It Back

Put together with chapter 02's dependency-injection pattern and chapter 05's `tenant_id`-enforced
repository layer, a single request now passes through three independently testable, composed checks
before it touches a document: **who is this** (`Depends(get_current_user)`, validating an MSAL-acquired,
Azure AD-issued token), **what are they allowed to do** (`Depends(require_role(...))`, checking the
token's `roles` claim against Azure AD App Role assignments), and **whose data can they see**
(the repository's `tenant_id` filter, resolved from the token, never from client input). That three-layer
separation — authentication, authorization, and tenant scoping as distinct concerns, not one blurred
check — is the concrete, defensible answer to "how does this service enforce access control," and it's
the frame the next chapter's Q&A (`99-Interview-QA.md`) builds directly on.
