# Subdomain Takeover

A subdomain takeover happens when a DNS record points to an external service that no longer exists or is no longer claimed. If you can register that external resource, you control what the subdomain serves.

---

## How It Happens

1. Company sets up `staging.example.com` CNAME pointing to `example.herokuapp.com`
2. They delete the Heroku app but leave the DNS record
3. The CNAME still resolves - it just points at nothing
4. You create a Heroku app and claim `example.herokuapp.com`
5. `staging.example.com` now serves your content

The DNS record is the vulnerability. The external service just provides the mechanism.

---

## Why It Matters

A subdomain takeover is not just defacement. What you can actually do:

- **Cookie theft** - if `*.example.com` cookies are scoped to the domain, your controlled subdomain can read them. `document.cookie` on `evil.example.com` sees cookies set with `domain=.example.com`.
- **CORS bypass** - if the main app trusts `*.example.com` as an allowed origin, requests from your subdomain get `Access-Control-Allow-Origin` and can read authenticated responses.
- **Bypass CSP** - if `staging.example.com` is whitelisted in `script-src`, hosting a JS file there lets you execute scripts on the main app.
- **Phishing** - a login page on `login.example.com` or `sso.example.com` is far more convincing than an external domain.
- **SAML/OAuth redirect** - if the IdP allows redirects to any `*.example.com`, your controlled subdomain can receive auth codes or tokens.
- **Email** - if MX records also point at an unclaimed service, you can receive email to that subdomain (password resets, internal notifications).

---

## Vulnerable Services

Most platforms assign a unique hostname when you create a resource. If that resource is deleted but the CNAME remains, the hostname is reclaimable.

| Service | Dangling indicator |
|---------|-------------------|
| GitHub Pages | `There isn't a GitHub Pages site here` |
| Heroku | `No such app` |
| Netlify | `Not Found - Request ID` |
| AWS S3 | `NoSuchBucket` |
| AWS Elastic Beanstalk | `404 Not Found` on `.elasticbeanstalk.com` |
| Azure (various) | `404 Web Site not found` on `.azurewebsites.net` |
| Fastly | `Fastly error: unknown domain` |
| Pantheon | `404 error unknown site` |
| Ghost | `The thing you were looking for is no longer here` |
| Shopify | `Sorry, this shop is currently unavailable` |
| Tumblr | `There's nothing here` |
| WordPress.com | `Do you want to register...` |
| Zendesk | `Help Center Closed` |
| Surge.sh | `project not found` |
| Readme.io | `Project doesnt exist` |
| Unbounce | `The requested URL was not found` |
| HubSpot | `Domain not configured` |

This list shifts as services update their behaviour. Can-I-Take-Over-XYZ on GitHub maintains a current reference.

---

## Finding Targets

### Enumerate subdomains first

Passive:
```
subfinder -d example.com -silent
amass enum -passive -d example.com
assetfinder --subs-only example.com
```

Certificate transparency (no rate limiting, no noise):
```
curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sort -u
```

Active (DNS bruteforce):
```
puredns bruteforce wordlists/subdomains.txt example.com
```

### Check which ones are dangling

For each subdomain, check if the CNAME points somewhere unclaimed:

```bash
# Get the CNAME chain
dig CNAME staging.example.com

# If it ends at a known third-party hostname, check if that hostname returns a takeover fingerprint
curl -s -o /dev/null -w "%{http_code}" https://example.herokuapp.com
```

Automated:
```
subjack -w subdomains.txt -t 100 -ssl -o results.txt
nuclei -l subdomains.txt -t takeovers/
```

subjack and nuclei both maintain fingerprint databases for vulnerable services.

### CNAME vs A record

CNAMEs are the primary vector - they delegate resolution to a third party.

A records pointing at cloud IPs can also be vulnerable if the IP was an elastic/ephemeral address that's since been reassigned and the new owner can serve content from it (less common, harder to exploit predictably).

NS delegation is the worst case. If a subdomain has its own NS record pointing at a nameserver you control or can register, you control all DNS for that subdomain - not just HTTP but MX, TXT, everything.

---

## Confirming Vulnerability

Before claiming anything, confirm the record is actually dangling:

```bash
# 1. Confirm CNAME exists and where it points
dig CNAME target.example.com +short
# Returns: someapp.herokuapp.com.

# 2. Confirm that destination returns a takeover indicator
curl -sk https://someapp.herokuapp.com | grep -i "no such app"

# 3. Confirm it's not just a temporary outage - check again from a different resolver
dig CNAME target.example.com @8.8.8.8 +short
dig CNAME target.example.com @1.1.1.1 +short
```

If both resolvers return the same CNAME and the destination is unclaimed, it's vulnerable.

---

## Claiming the Subdomain

Process depends on the service:

**Heroku**
```bash
heroku create someapp   # claim the exact app name
heroku domains:add target.example.com
```

**GitHub Pages**
Create a repo, enable Pages, add a CNAME file containing `target.example.com`.

**AWS S3**
Create a bucket with the exact subdomain name (`target.example.com`), enable static website hosting, set bucket policy to public.

**Netlify**
Create a site, go to Domain settings, add custom domain `target.example.com`.

**Azure**
App Service -> Custom domains -> Add the subdomain.

For most services the claim is: create a resource with the exact hostname that the CNAME points to.

---

## Impact Demonstration

For a bug report, proof of concept is usually a static page served from the subdomain:

```html
<!DOCTYPE html>
<html>
<head><title>Subdomain Takeover PoC</title></head>
<body>
<h1>Subdomain Takeover</h1>
<p>This page is served from target.example.com via a dangling CNAME to [service].</p>
<p>Discovered: [date]</p>
<script>
// Demonstrate cookie access if in scope
document.write('<p>Cookies visible: ' + document.cookie + '</p>');
</script>
</body>
</html>
```

Do not do anything beyond proving control. Don't collect real traffic, don't serve malicious content, don't read actual cookies from real users. Claim the resource, serve a PoC page, report it, release it.

---

## Edge Cases

**Wildcard DNS** - `*.example.com` A record pointing at a load balancer. Not usually a takeover vector unless the LB passes unrecognised vhosts to a default backend you can influence.

**Internal subdomains** - `internal.example.com` CNAME pointing at an internal hostname. If you're doing an external assessment, you can't resolve or claim it. Note it but can't exploit it from outside.

**Email takeover via MX** - if MX points at an unclaimed mail service (rare but exists), you can receive email sent to `*@target.example.com`. This includes password resets and internal notifications. Higher severity than HTTP-only takeover.

**Vendor subdomains** - companies often set up `support.example.com` via Zendesk or `status.example.com` via Statuspage. These are just as vulnerable as internally managed subdomains and often overlooked.

---

## Checklist

- [ ] Enumerate subdomains passively and via CT logs
- [ ] For each, check if CNAME/NS points at a third-party service
- [ ] Check if that third-party hostname returns a takeover fingerprint
- [ ] Confirm from multiple resolvers (not a cache issue)
- [ ] Identify the service and check if claiming is possible
- [ ] Note CORS policy on main domain - does it trust *.example.com?
- [ ] Note CSP on main domain - is subdomain whitelisted?
- [ ] Check cookie scope - are cookies set with domain=.example.com?
- [ ] Check if MX records are also affected
- [ ] Claim the resource, serve a minimal PoC, report immediately
