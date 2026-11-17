# nutanixguy-redirect

Static GitHub Pages site. It redirects nutanixguy.com and www.nutanixguy.com to https://cv.nutanixvibes.com. `index.html` covers `/`. `404.html` covers any other path and, when JavaScript runs, keeps the path, query, and hash. There is no build step.

DNS for the domain lives in Microsoft 365. The apex A records point at the GitHub Pages IP addresses, and www is a CNAME to nutanixguy.github.io. Mail records in that zone must never be touched.
