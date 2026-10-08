# School mailer contract
Providers smtp, gmail (Google OAuth gmail.send API), appsscript (signed/encrypted token push). Old appsscript URL relay remains readable as interim compatibility until admin switches after token confirmation.
PUT /admin/settings mailer fields provider, fromName, gmailUser, fromEmail, clientId, clientSecret, refreshToken, smtpHost, smtpPort, smtpSecure, smtpUser, smtpPassword. Secrets nullable to remove; absent keeps. Existing url/secret compatibility retained, not offered in new UI.
GET /admin/settings and /admin/mailer redact secret, clientSecret, refreshToken, smtpPassword, expose corresponding *Set booleans and configured. No token/private key in any ordinary GET.
POST /admin/mailer/appsscript/generate {} rotates dedicated RSA2048 identity and HKDF generation salt; invalidates prior tokens/scripts. Returns {codeGs,appsscriptJson,keyId}. This is the ONLY HTTP operation disclosing script secrets; Cache-Control no-store. Does NOT switch mailer or change other settings.
GET /admin/mailer/appsscript/status returns {generated,keyId,tokenSet,tokenValid,expiresAt,lastPushAt}. These endpoints require existing school-admin bearer sessions.
PUT /admin/settings {mailer:{provider:"appsscript",appsscriptMode:"token",gmailUser:"school address"}} only after status tokenValid. New UI can select token mode automatically in save. Interim relay remains if appsscriptMode absent and URL exists.
Public POST /mailer/appsscript/token envelope {ciphertext,iv,ts,signature}, strict JSON. Fields ciphertext/iv/signature canonical unpadded base64url; ciphertext includes trailing16-byte GCM tag, iv12 bytes, signature256 bytes; ts13-digit integer UTC epoch ms.
RSA-SHA256 PKCS1v1.5 signs ASCII concatenation ciphertext||iv||decimal(ts), with no delimiter. Fixed IV16chars + ts13digits and ciphertext strict bounded encoding make framing unambiguous. Verify RSA before AES decrypt. AES256GCM no AAD, key HKDF-SHA256(server PKCS8 DER signing key, random32byte generation salt, info UTF8 carpschool-appsscript-aes,32). Never embed server signing key; embed only derived AES key and dedicated RSA PKCS8 private key. Server stores public key, salt, keyId in private mailer state collection.
Encrypted UTF8 JSON {token,expiresAt,ts,nonce}; expiresAt integer ms within now+60min, >now+30seconds; ts matches envelope ±5min; nonce UUIDv4. Persistent unique nonce insert atomic, TTL >= signed timestamp acceptance window. Token state encrypted at rest under separately HKDF-derived key info carpschool-appsscript-at-rest; atomic CAS keyId prevents rotation race and monotonic ts prevents older pushes replacing newer tokens.
Script fetches Google tokeninfo to obtain actual expires_in (ScriptApp does NOT promise 60min), checks gmail.send, advertises expiry min(actual-60sec,50min), installs one every30min trigger, pushes immediately. Trigger interruptions or short-lived tokens can cause gaps; server fails closed, no relay fallback in token mode. No logs contain token or raw errors. Generated IV uses locked persisted counter with generation-specific prefix, no Math.random.

## UPDATE: no client credentials in any UI
Central Google client id/secret are env-only (no DB, no admin UI). Mailer UI has no client ID/secret/refresh-token inputs at all.
Legacy 'gmail' is shown read-only (gmailUser only); saving it sends only {provider, fromName}, so stored legacy creds stay untouched until migration.
School settings view no longer needs clientId/clientSecretSet/refreshTokenSet (web ignores them).

## UPDATE: actual backend Apps Script contract (per backend, web adapted)
POST /admin/mailer/appsscript/generate {regenerate?} -> {codeGs, appsscriptJson, keyId}; 409 when key exists and regenerate!==true; no-store.
GET /admin/mailer/appsscript/status -> {generated, keyId, tokenSet, tokenValid, expiresAt, lastPushAt}.
Settings view mailer.push no longer used by web.

Security: SMTP host/port permits school admin network access; only trusted admins may configure it. SMTP security explicitly supports starttls (TLS required), tls (implicit TLS), and none (explicit plaintext, trusted private relays only); STARTTLS fails closed if TLS cannot be negotiated and never silently downgrades to plaintext. No TLS verification bypass. Script editors have mail authorization; never share/clone or clear Script Properties without regenerating. RSA generation has CPU cost and is throttled. Public body limited by existing Express JSON parser; endpoint existing global rate limit applies. Server token admission validates possession of generated keys, not Google account identity; Gmail API ultimately validates actual authorization.

## Backend implementation reconciliation
New explicit provider appsscript_push delivers using signed token push. Existing provider appsscript with no mode or mode relay still uses the deployed legacy URL/secret relay, unchanged. Compatibility explicit appsscriptMode token remains accepted, but new UI should save provider appsscript_push. Token-mode saves require current valid server token. Generation never changes provider.
Public POST /mailer/appsscript/iv accepts {keyId,ts,nonce,signature}. RSA-SHA256 signs ASCII carpschool-appsscript-iv:<keyId>:<ts>:<nonce>. Strict canonical encoding, 5-minute timestamp window, atomic persistent unique request nonce, generation-bound Mongo counter. Returns {iv} (12-byte unpadded base64url). IV is SHA256(keyId) first8 bytes plus big-endian uint32 counter. Counter survives script copies/resets; generation changes AES key. Script-local counter is NOT used.
Central Google broker callback target https://api.carpschool.ca/mailer/google/callback. Broker implementation is pending, do not assume this route is deployed. Proposed school UI endpoints POST /admin/mailer/google/connect -> {url}, GET /admin/mailer/google/status -> {connected,email,expiresAt}, POST /admin/mailer/google/disconnect -> {ok:true}. No client-credential inputs; legacy gmail configuration preserved until explicit migration.

## Google (actual, per backend, supersedes mailer.google above)
- POST /admin/mailer/google/connect -> {url}  NO BODY; backend uses allowlisted request Origin; callback /admin/school?mailer=connected|error
- GET  /admin/mailer/google/status  -> {connected, email, expiresAt}
- POST /admin/mailer/google/disconnect

Google disconnect response

POST /admin/mailer/google/disconnect deletes local Google mailer state before contacting the broker. Returns {ok:true,localDisconnected:true,revocation}, where revocation is revoked after a verified signed acknowledgement, unconfirmed on central/Google failure, or not_required without a stored refresh token. Unconfirmed does not retain locally usable tokens or disclose upstream details; it does not assert that the Google grant was revoked.
