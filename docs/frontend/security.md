# Shieldgrid Web — Frontend Security Model

## 1. In-Memory JWT Token Storage Rationale

### The Rule
**JWT tokens MUST be stored ONLY in React memory state (`AuthProvider`). They must NEVER be written to `localStorage` or `sessionStorage`.**

### Security Rationale
- **XSS Session Theft Mitigation:** Any JavaScript running in the browser origin can inspect and read `localStorage` and `sessionStorage`. If a single Third-Party dependency or untrusted user input causes an XSS vulnerability, an attacker can extract persistent tokens from storage and compromise user sessions permanently.
- **Tradeoff:** Storing tokens in memory means a page refresh forces a re-authentication login. For a security operations platform, requiring re-login on refresh is an intentional security design choice, not a usability bug to be "fixed".

---

## 2. Untrusted Input Handling (XSS Prevention)

### Rule
All alert fields (`raw_payload`, `source`, connector logs) originate from external, untrusted sensors (Wazuh, Velociraptor).

- **Rendering Rule:** Free-text fields and raw payloads must **ALWAYS** be rendered as plain text nodes (e.g. `{alert.source}`, `<pre>{JSON.stringify(raw_payload)}</pre>`).
- **Forbidden:** Never use `dangerouslySetInnerHTML`, `v-html`, `eval()`, or unescaped HTML injection for sensor telemetry.

---

## 3. Production Bundle Inspection & Secret Audit

- The built JS bundle (`dist/assets/*.js`) is audited in CI (`scripts/ci-local.sh` and `.github/workflows/ci.yml`) using regular expression greps (`secret`, `jwt_secret`, `private_key`, `bearer ey`) to guarantee no API secrets or development keys are accidentally bundled into client code.

---

## 4. Deployment Security Headers Guidance

When serving `shieldgrid-web` in production (via Nginx, Caddy, or Cloudflare), the web server MUST be configured with the following HTTP security headers:

```nginx
# Nginx configuration example
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; connect-src 'self' http://localhost:3000 https://api.shieldgrid.internal;" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```
