# 07 — Secrets, Configuration, and Key Vault

## What's real vs. what's proposed in this chapter

Be precise about this distinction, because it's easy to overclaim: **this service's real configuration
mechanism is Azure App Service Application Settings (environment variables) — not Azure Key Vault.**
Everything under "What's actually configured today" below is confirmed directly from
`DEPLOYMENT_REQUIREMENTS.md` and `app.py`'s `os.environ[...]`/`os.environ.get(...)` calls throughout.
Everything under "Key Vault as a proposed hardening step" is exactly that — a proposal, framed as such,
for what you'd recommend today rather than a description of what's running. **Managed Identity is
confirmed real** (used for Blob Storage access, see below) — that part of the chapter isn't a proposal.

## What's actually configured today: App Service Application Settings

`DEPLOYMENT_REQUIREMENTS.md` states this plainly: *"a physical `.env` file is only read for local
development — production values must live in Application Settings."* `app.py` confirms this in code —
`load_dotenv()` is only called when `IS_LOCAL_MACHINE` is true; in every other environment, every
credential and config value is read straight from the process environment (`os.environ[...]`), which in
Azure App Service means values set under **Configuration → Application settings** in the portal (or via
`az webapp config appsettings set`), injected as environment variables into the running process. No
Key Vault SDK call, no `@Microsoft.KeyVault(SecretUri=...)` reference, and no `azure-keyvault-secrets`
import appear anywhere in this codebase.

The real environment variable inventory, in full (from `FULL_ARCHITECTURE.md` §11a and
`DEPLOYMENT_REQUIREMENTS.md`):

**Pre-existing (all departments):** `INGEST_API`, `IDENTITY_CLIENT_ID`,
`MICROSOFT_PROVIDER_AUTHENTICATION_SECRET`, `AZURE_TENANT_ID`, `IDENTITY_SCOPE`, `IDENTITY_REDIRECT_URI`,
`DATABASE_DRIVER`, `DATABASE_SERVER`, `DATABASE_NAME`, `ENVIRONMENT`, `IS_LOCAL_MACHINE`,
`BUILD_SOURCE`, `BUILD_VERSION`, `PORT`.

**Added for the IWPB feature:** `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`,
`SMTP_USE_TLS`, `SMTP_DISTRIBUTION_MAILBOX`, `UPLOADER_APP_URL`, `AZURE_STORAGE_ACCOUNT_URL`,
`AZURE_STORAGE_CONNECTION_STRING`, `PENDING_UPLOAD_CONTAINER`, `IWPB_MAINTENANCE_INTERVAL_SECONDS`,
`INGEST_API_SCOPE`.

Two of these are plaintext secrets sitting directly in Application Settings today with no additional
protection layer: `MICROSOFT_PROVIDER_AUTHENTICATION_SECRET` (this app's own AAD client secret) and
`SMTP_PASSWORD` (if the corporate SMTP relay requires authentication — it's optional, per
`DEPLOYMENT_REQUIREMENTS.md`). `AZURE_STORAGE_CONNECTION_STRING` is a third, if that auth mode is chosen
over the Managed-Identity-based `AZURE_STORAGE_ACCOUNT_URL` alternative.

## Managed Identity: confirmed real, for Blob Storage access

Unlike Key Vault, **Managed Identity usage is confirmed in the actual code** — `src/uploader/storage.py`
supports two auth modes for the IWPB Blob Storage staging container, and the recommended one (per
`DEPLOYMENT_REQUIREMENTS.md`, "Recommended in Azure") uses no secret at all:

```python
# src/uploader/storage.py — the Managed Identity path
elif account_url:
    service_client = BlobServiceClient(
        account_url=account_url,
        credential=DefaultAzureCredential(),
    )
```

`DefaultAzureCredential()` in an Azure-hosted process resolves to the App Service's own **system- or
user-assigned Managed Identity**, which needs the **Storage Blob Data Contributor** RBAC role granted on
the storage account (or scoped tighter, to just the `pending-iwpb-uploads` container) —
`DEPLOYMENT_REQUIREMENTS.md` §3 lists this as a one-time Azure-side role assignment, not something
configured in code. The alternative, `AZURE_STORAGE_CONNECTION_STRING`, is explicitly documented as the
"simpler option (account key/SAS-based) ... easiest for local dev/testing" — i.e., the secret-based
fallback for when Managed Identity isn't practical (mainly local development, where there's no App
Service identity to assume). This is the same core idea covered generically in other courses in this
curriculum — an app's own Azure AD identity, rather than a credential a developer manages, is what
authenticates it to a dependency — just confirmed here as actually running, specifically for Blob access,
not for Key Vault.

`app.py`'s `lifespan()` also creates a **second**, separate `DefaultAzureCredential` instance —
`azure.identity.aio.DefaultAzureCredential()`, the async variant, stored on `app.state.credential` — used
elsewhere in the app's own Azure integrations; `storage.py`'s is the synchronous `azure.identity`
variant, a deliberately separate instance since Blob access there isn't on the async request path.

## Key Vault as a proposed hardening step, not confirmed reality

If asked "would you use Key Vault here," the honest, credible answer is: **not today, but yes, as a
concrete recommendation**, specifically for the values that are real secrets sitting in plaintext
Application Settings right now:

- `MICROSOFT_PROVIDER_AUTHENTICATION_SECRET` — this app's own AAD client secret.
- `SMTP_PASSWORD` — if the SMTP relay requires authentication.
- `AZURE_STORAGE_CONNECTION_STRING` — if the connection-string auth mode is used instead of Managed
  Identity for Blob access.
- The **hardcoded `SessionMiddleware` secret_key** (`"uploader-secret-key"`, chapter 06) — today it
  isn't even in Application Settings, it's a literal string in source, which is a strictly worse
  starting point than "in Application Settings but not Key Vault."

The proposed pattern — standard, and worth being able to describe precisely rather than vaguely — is a
**Key Vault reference** directly in App Service Application Settings:

```
SMTP_PASSWORD = @Microsoft.KeyVault(SecretUri=https://<vault-name>.vault.azure.net/secrets/smtp-password/)
```

App Service resolves this at runtime via the same Managed Identity already confirmed real for Blob
access (granted a `get`/`list` access policy or RBAC role on the vault instead of the storage account),
and injects the actual secret value as a normal environment variable — so **none of `app.py`'s existing
`os.environ["SMTP_PASSWORD"]`-style code would need to change at all**. That's the strongest part of
this recommendation as a proposal: it's a configuration-only change, with the same operational win any
Key Vault migration offers — rotating a compromised secret becomes a Key Vault-side operation (update
the secret, restart the app to pick up the new value) instead of a source change and redeploy, and
there's a centralized audit trail of which secret was accessed when, which today doesn't exist for
plaintext Application Settings values.

## Tying It Back

The honest version of this chapter is more interesting than a generic "we used Key Vault" story: this
service demonstrates the **correct, secretless pattern already, for Blob Storage** (Managed Identity,
zero credentials in config), while three or four other values — an AAD client secret, an SMTP password,
optionally a storage connection string, and a literal hardcoded session-signing key — are still sitting
as either plaintext Application Settings or, worse, a source-code constant. Being able to name exactly
which values are in which state, and propose the specific, low-risk Key Vault-reference migration for
the ones that need it, is a stronger and more credible answer than either overclaiming "everything's in
Key Vault" or leaving the gap unnamed.
