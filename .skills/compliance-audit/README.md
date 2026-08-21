# compliance-audit

<!-- encoding: utf-8 -->

Post-deploy compliance verification: confirms the live site satisfies GDPR, 152-ФЗ, and ePrivacy requirements after any code, template, or UX change that could affect data handling, consent, or security. Does not fix bugs — reports pass/fail per criterion and saves results for the release record.

Not a general security audit (see `.skills/security-audit/`), not doc health (see `.skills/docs-audit/`), not process analysis (see `.skills/retrospective-analysis/`).

## When to load

- After every production deploy — mandatory before the release can be considered shipped
- After any PR that touches: consent/analytics snippet, privacy page, cookie handling, CSP/security headers, account deletion/export, template bases, or form submission handlers
- Before merging to `current` when the PR metadata (`docs/PRIVACY_COMPLIANCE.md` §4.7 criteria) is affected
- When the user says "check after deploy, that new site today comply fully"

## Workflow

### 1. Verify the live site

Query the production origin (`https://se.math.spbu.ru`). Do NOT rely on local code inspection — compliance is about the deployed artifact.

```bash
curl -sI https://se.math.spbu.ru/
```

### 2. Security headers

Assert each header is present with correct values on the homepage:

| Header | Expected | Why |
|--------|----------|-----|
| `Content-Security-Policy` | Allowlist: default-src 'self'; script-src includes topbar/mc.yandex/api-maps + maps.googleapis/maps.yandex.net; object-src 'none'; base-uri/form-action/frame-ancestors 'self'; upgrade-insecure-requests | Prevents injection, limits data-exfiltration surface |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing |
| `X-Frame-Options` | `DENY` | Prevents clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Trims referrer to analytics providers |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), payment=(), usb=()` | No unnecessary device APIs enabled |
| `Cross-Origin-Opener-Policy` | `same-origin` | Isolates cross-origin windows |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Enforces HTTPS |

### 3. Consent banner

Check the homepage HTML has (via `We fetch` or `curl`):

- ✅ Consent banner container (`id="se-consent-banner"`) visible on first load
- ✅ Three categories: essential (checked, disabled), statistics, marketing
- ✅ "Принять выбранные" accept button
- ✅ "Отклонить необязательные" decline button
- ✅ Link to `/privacy.html` in the banner text
- ✅ No `mc.yandex.ru` script in the initial HTML (`data-se-metrica-id=""` or absent; the metrica snippet must load dynamically only after consent)

### 4. Privacy page (`/privacy.html`)

Check:

- ✅ HTTP 200
- ✅ Operator name: Санкт-Петербургский государственный университет (СПбГУ)
- ✅ Operator address: 199034, Санкт-Петербург, Университетская набережная, д. 7–9
- ✅ 152-ФЗ and GDPR are explicitly cited as legal bases
- ✅ Cookies section enumerates: `se_session`, `se_consent`, `_ym_*`
- ✅ Retention terms match §5 #5:
  - `se_session` — до 24 часов
  - `se_consent` — до 1 года
  - account until deletion
  - educational records per university archival rules
  - publications for their lifetime
- ✅ Rights under 152-ФЗ ст. 14 / GDPR ст. 15–22
- ✅ Contact email: `a.terekhov@spbu.ru`
- ✅ Last updated date matches or postdates the release date

### 5. Data subject rights (profile pages)

Verify (authenticated):

- `/profile.html` — accessible, shows user info
- `/profile/delete` — POST-only, requires login, CSRF-protected (soft-delete: clears login data, keeps names)
- `/profile/export.zip` — GET, requires login, streams a ZIP archive

Unauthenticated test:

```bash
curl -sI https://se.math.spbu.ru/profile.html | rg "302|Location"
# Expected: 302 → /login.html?next=user_profile
```

### 6. Form notices

Spot-check forms that collect personal data:

- Registration form (`/login.html?register`) — has consent notice
- Practice form — has consent notice
- Thesis-review form — has consent notice
- Internship form — has consent notice

The notice must contain: `consent_notice.html` template or equivalent inline text linking to the privacy policy.

### 7. Dormant services check

- ✅ No `googletagmanager.com` requests in the HTML (any base template)
- ✅ No `GTM-` or `dataLayer` references
- ✅ Yandex Metrica snippet does NOT fire without a configured id
- ✅ No third-party analytics requests on first page load (verify via DevTools Network tab or `curl` + grep for known tracker domains)
- ✅ Maps show "Источник карты не задан" placeholder when no key is configured

### 8. Retention / account status

- Account deletion soft-deletes — check `Users.deleted` is `server_default=sa.false()` in the model
- Published content survives account deletion (names kept)

### 9. Report

Output a checklist:

```
## Compliance Audit — vYYYY.MM.DD (date)

Status: PASS / FAIL / PARTIAL

| # | Check | Result |
|---|-------|--------|
| 1 | Security headers (CSP, HSTS, XFO, etc.) | PASS / FAIL |
| 2 | Consent banner present with 3 categories | PASS / FAIL |
| 3 | No analytics in initial HTML (dormant) | PASS / FAIL |
| 4 | /privacy.html: operator, 152-ФЗ/GDPR, cookies, retention, rights | PASS / FAIL |
| 5 | Profile: login-required, delete (POST), export (GET) | PASS / FAIL |
| 6 | Form consent notices present | PASS / FAIL |
| 7 | No GTM / stale tracker references | PASS / FAIL |
| 8 | Retention terms match PRIVACY_COMPLIANCE.md §5 #5 | PASS / FAIL |
| 9 | Account soft-delete keeps published content | PASS / FAIL |

Details:
- <any failure: exact URL, header/keyword expected vs found>
```

If any row is FAIL, the release is not compliant. File a blocker issue and do not publish until fixed.

## Dependencies

- `curl` — for live HTTP checks
- Read access to `docs/PRIVACY_COMPLIANCE.md` — reference for current decisions, retention terms, and operator info
- Read access to `docs/RELEASE_CHECKLIST.md` — B18 (PD compliance recorded) cross-reference

## See also

- `docs/PRIVACY_COMPLIANCE.md` — full compliance documentation with decisions
- `.skills/security-audit/` — deep security review
- `docs/RELEASE_CHECKLIST.md` B18 — Roskomnadzor record-keeping
