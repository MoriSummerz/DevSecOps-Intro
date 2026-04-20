# Lab 11 — Reverse Proxy Hardening: Nginx Security Headers, TLS, and Rate Limiting

## Task 1 — Reverse Proxy Compose Setup

### Why a reverse proxy improves security

Placing Nginx in front of Juice Shop gives operations a single hardened ingress point without touching application code:

- **TLS termination** — certificates, protocol versions, and cipher suites are configured once at the edge instead of being re-implemented in every app container.
- **Security header injection** — the proxy adds `X-Frame-Options`, `X-Content-Type-Options`, HSTS, Referrer/Permissions policies, COOP/CORP, and CSP on every response, including errors and redirects (`add_header … always`). This is especially valuable when the upstream application either omits headers or emits conflicting ones — Nginx strips them via `proxy_hide_header` and re-applies a consistent policy.
- **Request filtering / rate limiting** — `limit_req_zone` + `limit_req` provides a defence-in-depth layer against credential stuffing and brute force that the app would otherwise have to implement itself.
- **Single access point / smaller attack surface** — the app binds only to the internal Docker network; all inbound traffic is forced through Nginx's audited configuration. Vulnerability scanners, misconfigurations, and admin endpoints on the app container are not directly reachable from the host.

### Why hiding direct app ports reduces attack surface

Juice Shop in `docker-compose.yml` uses `expose: 3000` (container-to-container only), not `ports: 3000:3000`. That means:

- Port 3000 is **not published** on the host — an attacker on the LAN cannot hit the unhardened Node.js app and bypass Nginx's headers, TLS, and rate limits.
- All traffic must traverse the Nginx policy layer; no shadow endpoints can exist.
- Only ports `8080` (HTTP → redirect) and `8443` (HTTPS) leave the host, which matches the least-privilege principle for network exposure.

### `docker compose ps` evidence

```
NAME            IMAGE                           COMMAND                  SERVICE   CREATED         STATUS         PORTS
lab11-juice-1   bkimminich/juice-shop:v19.0.0   "/nodejs/bin/node /j…"   juice     4 minutes ago   Up 4 minutes   3000/tcp
lab11-nginx-1   nginx:stable-alpine             "/docker-entrypoint.…"   nginx     4 minutes ago   Up 4 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp, 80/tcp, 0.0.0.0:8443->8443/tcp, [::]:8443->8443/tcp
```

Only Nginx has `0.0.0.0:<host>->…` mappings. Juice Shop shows `3000/tcp` with no host-side binding — reachable only inside the Compose network.

The initial HTTP probe also confirms correct redirect behaviour:

```
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8080/
HTTP 308
```

---

## Task 2 — Security Headers

### Headers observed on HTTPS (`analysis/headers-https.txt`)

```
HTTP/2 200
server: nginx
strict-transport-security: max-age=31536000; includeSubDomains; preload
x-frame-options: DENY
x-content-type-options: nosniff
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), geolocation=(), microphone=()
cross-origin-opener-policy: same-origin
cross-origin-resource-policy: same-origin
content-security-policy-report-only: default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'
```

HTTP responses (`analysis/headers-http.txt`) carry the same header set **except** `Strict-Transport-Security`, which is correctly scoped to the HTTPS server block only (HSTS over plaintext is ignored by browsers and RFC 6797 forbids it).

### What each header protects against

- **X-Frame-Options: DENY** — prevents the page from being rendered inside any `<iframe>`. Defeats classic **clickjacking** / UI-redress attacks where an attacker overlays an invisible Juice Shop frame on their own page and tricks the victim into clicking admin controls.
- **X-Content-Type-Options: nosniff** — disables browser MIME-type sniffing. Stops attacks where an uploaded or reflected asset declared as `text/plain` (or a missing type) is re-interpreted by the browser as JavaScript/HTML and executed, a vector used in stored-XSS-via-file-upload chains.
- **Strict-Transport-Security (HSTS): max-age=31536000; includeSubDomains; preload** — instructs the browser to refuse plaintext HTTP to this origin for 1 year, covering subdomains and opting in to the browser preload list. Defends against **SSL-stripping MITM** and accidental HTTP leaks after the first visit.
- **Referrer-Policy: strict-origin-when-cross-origin** — sends the full URL on same-origin requests but only the origin (scheme+host+port) on cross-origin, and nothing at all on HTTPS→HTTP downgrades. Prevents leakage of session tokens, password-reset tokens, or internal paths through third-party `Referer` headers.
- **Permissions-Policy: camera=(), geolocation=(), microphone=()** — explicitly denies the page (and any iframes it may embed) access to powerful browser APIs. Shrinks the blast radius if a future XSS lands — the injected script cannot silently turn on the camera or geolocate the user.
- **COOP: same-origin / CORP: same-origin** — Cross-Origin-Opener-Policy isolates Juice Shop's browsing context so cross-origin popups cannot reach `window.opener`, mitigating Spectre-class side-channels and tab-napping. Cross-Origin-Resource-Policy prevents other origins from embedding Juice Shop resources (`<img>`, `<script>`, etc.), reducing data exfiltration and XSSI risk.
- **Content-Security-Policy-Report-Only** — in Report-Only mode the browser does not block violations but would report them if a `report-uri`/`report-to` endpoint were configured. This is the safe on-ramp for CSP: we can observe what Juice Shop actually loads (the app is very JS-heavy and needs `'unsafe-inline'`/`'unsafe-eval'`) before switching to enforcement. Once enforced, CSP is the single strongest browser-side mitigation against **XSS and data-injection attacks**.

---

## Task 3 — TLS, HSTS, Rate Limiting & Timeouts

### TLS / testssl.sh summary (`analysis/testssl.txt`)

**Protocols offered:**
- SSLv2 — not offered (OK)
- SSLv3 — not offered (OK)
- TLS 1.0 — not offered
- TLS 1.1 — not offered
- **TLS 1.2 — offered (OK)**
- **TLS 1.3 — offered (OK), final**
- ALPN: `h2, http/1.1`

**Cipher suites offered (server-preferred order):**

| Version | Suite                              | Key exch.  | Enc.        | Bits |
|---------|------------------------------------|-----------|-------------|------|
| TLS 1.2 | ECDHE-RSA-AES256-GCM-SHA384        | ECDH P-256 | AES-GCM     | 256  |
| TLS 1.2 | ECDHE-RSA-AES128-GCM-SHA256        | ECDH P-256 | AES-GCM     | 128  |
| TLS 1.3 | TLS_AES_256_GCM_SHA384             | ECDH/MLKEM | AES-GCM     | 256  |
| TLS 1.3 | TLS_CHACHA20_POLY1305_SHA256       | ECDH/MLKEM | ChaCha20    | 256  |
| TLS 1.3 | TLS_AES_128_GCM_SHA256             | ECDH/MLKEM | AES-GCM     | 128  |

All suites offer **AEAD + Forward Secrecy**. Weak categories (NULL, anonymous, EXPORT, LOW/DES/RC4, 3DES, CBC) are all *not offered*.

**Why TLSv1.2+ is required (prefer 1.3):**
- TLS 1.0/1.1 are formally deprecated by RFC 8996 and by every major browser since 2020 — they rely on HMAC-SHA-1/MD5 for integrity and mandatory CBC-mode cipher suites vulnerable to BEAST, Lucky13, and POODLE-TLS.
- TLS 1.2 (with the AEAD-only cipher whitelist in `nginx.conf`) removes those weaknesses.
- TLS 1.3 drops all legacy key exchanges, forces AEAD ciphers and forward secrecy, eliminates renegotiation, and cuts the handshake to 1-RTT (0-RTT with care) — faster **and** smaller attack surface.

**Known-vulnerability scan (all clean):** Heartbleed, CCS Injection, Ticketbleed, ROBOT, CRIME, BREACH, POODLE, SWEET32, FREAK, DROWN, LOGJAM, BEAST, LUCKY13, Winshock, RC4 — **not vulnerable** in every case. Secure renegotiation is supported and client-initiated renegotiation is disabled.

**Expected "NOT ok" items (dev-cert only):**
- `Chain of trust … NOT ok (self signed)` — self-signed localhost cert by design.
- `Trust (hostname) … certificate does not match supplied URI` — testssl connects via `host.docker.internal`; SAN is `localhost, 127.0.0.1, ::1`.
- `OCSP URI / CRL / OCSP stapling / CT / CAA — not offered` — not applicable for a self-signed dev cert; eliminated by switching to a publicly-trusted CA (Let's Encrypt / mkcert) and enabling the stapling block already commented in `nginx.conf`.
- Final testssl grade is capped to **T** for these reasons; they are expected in a local lab.

**HSTS verification:** `Strict Transport Security — 365 days (=31536000 s), includeSubDomains, preload` appears on HTTPS (`headers-https.txt` line 13) but is **absent** from the HTTP response (`headers-http.txt`) — matches RFC 6797 and the `add_header` placement inside the `listen 8443 ssl` server block only.

### Rate limiting & timeouts

**Test output (`analysis/rate-limit-test.txt`, 12 sequential POSTs to `/rest/user/login`):**

```
401
401
401
401
401
401
429
429
429
429
429
429
```

6 × `401` (rejected credentials but served by the app) → 6 × `429` (blocked by Nginx before reaching the upstream). That matches the zone definition exactly: `rate=10r/m` ≈ 1 request every 6 s, `burst=5 nodelay` allows an immediate burst of 5 above the rate — so one "steady-state" request plus five burst slots = 6 admitted, everything after is rejected with `limit_req_status 429`.

**Access log excerpt (`logs/access.log`) confirming the 429s and that they never reached the upstream (no `urt=`):**

```
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.075 uct=0.005 urt=0.076
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.019 uct=0.000 urt=0.019
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.013 uct=0.001 urt=0.013
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.008 uct=0.001 urt=0.008
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.008 uct=0.001 urt=0.008
… "POST /rest/user/login HTTP/2.0" 401 26 "-" "curl/8.7.1" rt=0.005 uct=0.001 urt=0.006
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
… "POST /rest/user/login HTTP/2.0" 429 162 "-" "curl/8.7.1" rt=0.000 uct=- urt=-
```

The absent `uct=`/`urt=` on the 429 rows proves Nginx short-circuited the request — the upstream Juice Shop never saw those login attempts, which is exactly the brute-force mitigation we want.

**Why `rate=10r/m, burst=5`:**
- 10 logins per minute is well above any legitimate human rhythm (a normal user retries 1–3 times at most) but well below automated credential-stuffing throughput (thousands per minute).
- `burst=5 nodelay` tolerates genuine user mistakes (typo → correct → MFA prompt) without latency penalty, while still capping automated tools.
- Setting the limit any lower (e.g. `2r/m`) would frustrate real users on shared-IP networks (NAT, corporate egress). Setting it higher or without a burst cap makes online brute force practical. The current values are a defensible balance for a consumer web app; a banking app would tighten further and pair it with account-lockout logic.

**Timeout configuration (from `reverse-proxy/nginx.conf`):**

| Directive                 | Value | Rationale / trade-off |
|---------------------------|-------|-----------------------|
| `client_header_timeout`   | 10s   | Aggressively cuts **Slowloris** (slow-header DoS). 10 s is plenty for any real client; a mobile on 3G still sends full headers in <2 s. |
| `client_body_timeout`     | 10s   | Cuts **slow-POST / R-U-Dead-Yet** attacks. Trade-off: legitimate large uploads on bad networks may fail — acceptable here because `client_max_body_size 2m` already caps JSON/form payloads. |
| `keepalive_timeout`       | 10s   | Frees idle client connections quickly, starving connection-exhaustion tools. Cost: slight overhead for users navigating quickly as a new connection may be opened; with HTTP/2 this is effectively free. |
| `send_timeout`            | 10s   | Forces the client to continue reading the response within 10 s between successive writes. Protects against slow-read attacks. |
| `proxy_connect_timeout`   | 5s    | If the Juice Shop container is unreachable within 5 s, fail fast rather than piling requests on the proxy. |
| `proxy_read_timeout`      | 30s   | Upper bound on upstream response time. Long enough for normal Juice Shop endpoints (most are <500 ms) but short enough to stop a hung upstream from consuming proxy workers indefinitely. |
| `proxy_send_timeout`      | 30s   | Same idea in the request direction. |

The general trade-off pattern: **shorter client-side timeouts (10 s)** to defeat slow-connection DoS without hurting legitimate users, and **moderate upstream timeouts (5–30 s)** to keep the proxy responsive when the app misbehaves, with the understanding that a genuinely slow legitimate endpoint would need either a lifted per-location timeout or a redesign.

---

## Acceptance Checklist

- [x] Nginx reverse proxy running; Juice Shop not directly exposed on the host.
- [x] Security headers present over HTTP and HTTPS; HSTS only on HTTPS.
- [x] TLS enabled and scanned; TLS 1.2/1.3 only; HSTS verified; `analysis/testssl.txt` captured.
- [x] Rate limiting returns `429` on excessive login attempts (6/12 blocked); `logs/access.log` shows the 429s; timeouts documented with trade-offs.
- [x] All outputs committed under `labs/lab11/`.

## Artifacts

- `labs/lab11/docker-compose.yml`
- `labs/lab11/reverse-proxy/nginx.conf`
- `labs/lab11/analysis/headers-http.txt`
- `labs/lab11/analysis/headers-https.txt`
- `labs/lab11/analysis/testssl.txt`
- `labs/lab11/analysis/rate-limit-test.txt`
- `labs/lab11/logs/access.log`
- `labs/lab11/logs/error.log`
