# For Jamie

Facts from the files in this repo, from `dig` and `curl` on 2026-11-16, and from a local headless Chrome check the same day. GitHub Pages is not serving these files yet.

## 1. What this is

This repo is a static GitHub Pages site. It sends `nutanixguy.com` and `www.nutanixguy.com` to `https://cv.nutanixvibes.com`. There is no build and no app server.

Analogy: Pages is a load balancer that can only return a file, not a 301 to another origin. `index.html` and `404.html` are that file. The browser does the forward. DNS in Microsoft 365 is the VIP.

## 2. Architecture

```mermaid
flowchart LR
  browser[Browser]
  dns["Microsoft 365 DNS"]
  pages[GitHub Pages]
  html["index.html or 404.html"]
  dest["cv.nutanixvibes.com"]

  browser --> dns
  dns -->|"apex A records"| pages
  dns -->|"www CNAME to nutanixguy.github.io"| pages
  pages --> html
  html -->|"script, or meta refresh"| dest
```

`dig` on 2026-11-16 showed Microsoft name servers (`ns1` through `ns4.bdm.microsoftonline.com`). The apex A records are the four GitHub Pages addresses listed in GitHub's custom-domain docs: `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`. No AAAA records came back. `www` is a CNAME to `nutanixguy.github.io`. An MX record points at `nutanixguy-com.mail.protection.outlook.com`. This change did not edit DNS. Leave that MX record, and the rest of the mail records, alone.

`CNAME` names the custom domain Pages should bind: `nutanixguy.com`. `/` is `index.html`. Any other missing path is `404.html`. Pages is expected to return that file as the 404 body and leave the browser URL alone, so `location.pathname` is still `/foo`. That URL-preserving 404 was tested on a local stand-in, not on Pages.

Both HTML files run this script before the refresh meta:

`window.location.replace('https://cv.nutanixvibes.com' + location.pathname + location.search + location.hash)`

Without JavaScript, the meta refresh and the visible link go to `https://cv.nutanixvibes.com/` and drop the path.

GitHub Pages cannot send a server-side 301 to `https://cv.nutanixvibes.com`. A meta refresh plus the script is how this host moves the browser. After merge, set Settings > Pages to deploy from `main` / root, then tick Enforce HTTPS once the certificate is issued. On 2026-11-16, `https://nutanixguy.com` still presented a `*.github.io` certificate, so the name check failed. `http://nutanixguy.com` and `http://www.nutanixguy.com` both returned GitHub's generic 404, not these files.

## 3. Codebase map

- `index.html` — redirect for `/`.
- `404.html` — the same document, for unknown paths.
- `CNAME` — one line, `nutanixguy.com`.
- `.nojekyll` — empty, so Pages does not run Jekyll.
- `README.md` — what this repo does, and the warning not to touch mail DNS.
- `FORJAMES.md` — this file.

## 4. Tech stack and why each piece was chosen

GitHub Pages from `main`, folder `/`, because there is nothing to build. Plain HTML, because Pages will not emit a 301 to another site. `.nojekyll`, because a Jekyll pass is not needed and must not rewrite the files. No dependencies, analytics, or external scripts. The only script is the inline `location.replace` in `index.html` and `404.html`.

## 5. Decisions log

| Date | Decision | Why | Alternatives rejected |
| --- | --- | --- | --- |
| 2026-11-16 | Same meta refresh, canonical link, and script in `index.html` and `404.html` | A deep path is not a real file. The 404 document still sees the requested URL, so one script can keep pathname, search, and hash. | Jekyll `redirect_from` (still a meta refresh, and it needs Jekyll). GitHub Actions (nothing to generate). A 301 from this repo (Pages will not send one to another origin). |
| 2026-11-16 | `CNAME` contains only `nutanixguy.com` | The apex is the custom domain. `www` is already a DNS CNAME to `nutanixguy.github.io`. | Putting `www.nutanixguy.com` in `CNAME`, which would make www the primary name. |
| 2026-11-16 | Script before the refresh meta | The script keeps the path. The meta tag always targets the CV root and is the no-JS fallback. | Meta tag first, which can navigate to the root before the script runs. |

## 6. How to run and test it

No install. The files are the site. Python's `http.server` is not a fair stand-in: it does not return `404.html` for a missing path. Use a server that does, and that does not redirect the browser to another local URL.

Smoke test, run on 2026-11-16 with headless Chrome against that kind of local server:

1. `CNAME` is the single line `nutanixguy.com`. `.nojekyll` is 0 bytes. `index.html` and `404.html` are the same bytes.
2. JavaScript on, request `/`. The browser landed on `https://cv.nutanixvibes.com/`. That URL returned HTTP 200.
3. JavaScript on, request `/foo?x=1#y`. The stand-in returned `404.html` with status 404 and left the URL in place. The browser landed on `https://cv.nutanixvibes.com/foo?x=1#y`. That URL returned HTTP 404.
4. JavaScript off, same `/foo?x=1#y` request. The meta refresh landed on `https://cv.nutanixvibes.com/` (HTTP 200). The path was not kept.
5. The fallback is the anchor in the body of `index.html` and `404.html`. No CSS hides it. A screenshot was not taken: the refresh delay is `0`, so the browser leaves immediately.

GitHub Pages was not asked to serve `404.html`. This smoke test does not prove the live deploy.

## 7. Lessons learned

No defect turned up in the site files. With JavaScript off, `/foo?x=1#y` ended at the CV root. With JavaScript on, the same request kept `/foo`, the query, and the hash. That is the split the script order is there to produce.

## 8. Open questions / known gaps

- This commit does not enable Pages. After merge: Settings > Pages, deploy from `main` / root, then Enforce HTTPS once the cert exists. HTTPS for the apex failed its name check on 2026-11-16.
- GitHub's www-to-apex redirect was not observed. Over HTTP, both names returned GitHub's generic 404. GitHub's docs say that redirect starts once the apex is the configured custom domain and both names resolve to Pages.
- `cv.nutanixvibes.com` is also a CNAME to `nutanixguy.github.io`. On 2026-11-16, `/` returned 200 and `/foo` returned 404. A preserved path still 404s on the CV site when that path is not there.
- This repo is `nutanixguy/nutanixguy-redirect`, not `nutanixguy/nutanixguy.github.io`. A request to `https://nutanixguy.github.io/nutanixguy-redirect/foo` would keep the `/nutanixguy-redirect` prefix. That URL is not the front door. It was not tested.
- No AAAA records were in the dig answer. GitHub's docs also list four IPv6 addresses. They were not added.
