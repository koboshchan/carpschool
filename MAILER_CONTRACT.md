# Mailer API contract (proposed by web, Oct 7 2026)

All routes school-server, AdminGuard, responses Cache-Control: no-store.

## GET /admin/settings  -> mailer (sanitized view, never returns secrets)
{
  provider: 'smtp' | 'gmail' | 'appsscript_push' | 'appsscript' | 'test',   // 'gmail' = Google OAuth (kept for compat), 'appsscript' = legacy interim relay
  fromName: string,
  configured: boolean,                       // provider-specific
  // smtp
  smtpHost?: string, smtpPort?: number, smtpSecurity?: 'tls'|'starttls'|'none', smtpUser?: string, smtpFrom?: string, smtpPasswordSet: boolean,
  // google oauth (unchanged)
  gmailUser?: string, clientId?: string, clientSecretSet: boolean, refreshTokenSet: boolean,
  // legacy relay (unchanged, interim)
  url?: string, secretSet: boolean,
  // token push
  push: { keyCreatedAt: string|null, lastTokenAt: string|null, lastTokenStatus: 'never'|'ok'|'expired'|'rejected', tokenExpiresAt: string|null, lastError?: string /* short, non-secret */ }
}

## PUT /admin/settings {mailer: patch}
Patch keys: provider, fromName, smtpHost, smtpPort, smtpSecurity, smtpUser, smtpFrom, smtpPassword (string to set | null to clear | omitted keep),
gmailUser, clientId, clientSecret, refreshToken, url, secret (same write-only semantics as today).
Switching provider must NOT clear other providers' stored config (legacy relay stays usable until push works).

## POST /admin/mailer/appsscript/generate  body {regenerate?: boolean}
- No key yet: creates RSA2048 keypair + dedicated secrets, returns 200 {code: string, manifest: string, keyCreatedAt: string}
- Key exists and regenerate !== true: 409 {error:'KEY_EXISTS'}
- regenerate === true: rotates key, invalidates old script, returns 200 as above.
- Re-showing an existing script without rotating is NOT supported (secrets only in the explicit generate response). No GET for code.
- code = minified Code.gs (HKDF-AES256GCM, RSA2048, no server signing key), manifest = minified appsscript.json.

## GET /admin/mailer/appsscript/status -> push object above (for polling every ~10s while tab open)

