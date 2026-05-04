---
title: "The Silent Failure in GHEC Data Residency Proxy Setups"
date: 2026-05-04
draft: false
tags: ["github", "ghec", "data-residency", "proxy", "websocket", "networking"]
summary: "Why your GHEC DR instance loads fine but real-time updates are quietly broken — and the two commands that diagnose it in 10 seconds."
---

> Why your GHEC Data Residency instance loads fine but real-time updates are quietly broken — and the two commands that diagnose it in 10 seconds.

Everything works. Except it doesn't.

You've deployed a WAF or reverse proxy in front of your GHEC Data Residency instance. Pages load. `git push` and `git pull` work. The API responds. CI/CD runs fine. You mark the network integration as done and move on.

Weeks later, someone mentions that PR comments don't appear until they refresh the page. Actions log streaming doesn't stream — it loads in chunks after manual reload. Notification badges are always stale. Nobody filed a bug because it felt like a minor UI quirk, not a broken integration.

It's not a quirk. Your real-time updates are completely broken, and the cause is architectural — not a misconfiguration in the place you'd expect.

## The 10-second diagnostic

Before reading any further, run these two commands against your GHEC DR tenant:

```bash
nslookup alive.SUBDOMAIN.ghe.com
nslookup SUBDOMAIN.ghe.com
```

If the base domain resolves to your proxy but `alive.*` doesn't — you've found the problem. The rest of this post explains why.

## What GHEC Data Residency looks like on the wire

GHEC Data Residency doesn't serve everything from a single hostname. It distributes services across multiple subdomains under `*.SUBDOMAIN.ghe.com`. The one that matters most for this pattern is `alive.SUBDOMAIN.ghe.com` — GitHub's real-time update service that pushes live changes to the browser via WebSocket connections.

GitHub's network documentation lists `*.SUBDOMAIN.ghe.com` as a required hostname — the wildcard. But the docs don't spell out what each subdomain does, or what specifically breaks if you miss it.

This is a departure from GHES, where all services — including real-time updates — run behind a single appliance hostname. One proxy entry, one DNS record, done.

```text
GHES:     All services → single hostname → one proxy rule handles everything
GHEC DR:  Services distributed across subdomains → wildcard proxy required
```

If you've successfully proxied GHES before, your muscle memory will betray you here.

## The trap

Here's how it plays out:

1. Network team configures the proxy for `SUBDOMAIN.ghe.com` — the domain they see in the browser.
2. DNS points `SUBDOMAIN.ghe.com` to the proxy. Pages load. ✅
3. Engineers verify WebSocket config — `proxy_http_version 1.1`, `Upgrade` headers, `Connection` headers. Everything looks textbook.
4. Nobody configures DNS or proxy rules for `*.SUBDOMAIN.ghe.com` because it wasn't obvious that other subdomains needed proxying.
5. The browser tries to open a WebSocket to `alive.SUBDOMAIN.ghe.com`. DNS doesn't resolve to the proxy. The connection never happens.
6. Real-time updates silently fail. No errors. No logs. No alerts.

The result: a partially working deployment that passes every obvious test.

## Why diagnosis takes days, not minutes

This failure mode is designed to waste your time. Here's why:

- **No application errors.** The JavaScript that connects to the Alive service handles unreachable endpoints silently. The browser console shows nothing unusual.
- **No proxy logs.** The WebSocket traffic targets a different subdomain — one the proxy doesn't know about. The request never reaches the proxy, so there's nothing to log. Engineers stare at clean proxy logs and conclude the proxy isn't the problem.
- **The proxy config looks correct.** All the WebSocket directives are right — `Upgrade`, `Connection`, `proxy_http_version 1.1`. The config is fine. It's just serving the wrong scope.
- **GHES experience misleads.** Anyone who's proxied GHES knows that a single hostname handles everything. That mental model is wrong for GHEC DR, but it feels right — which makes it harder to question.
- **The symptom is easy to dismiss.** "Real-time updates don't work" sounds like a nice-to-have, not a broken deployment. Teams deprioritize it or attribute it to browser caching. It can take weeks before anyone investigates seriously.

## The fix

Once diagnosed, the fix takes minutes. Three things need to change:

### 1. DNS

Add a wildcard record so `*.SUBDOMAIN.ghe.com` resolves to the proxy — not just the base domain.

### 2. Proxy configuration

The broken config typically hardcodes the base hostname:

```nginx
# ❌ Broken — only handles the base domain
server_name  SUBDOMAIN.ghe.com;
proxy_pass   https://SUBDOMAIN.ghe.com;
Host         SUBDOMAIN.ghe.com;
```

The working config uses the wildcard and dynamic host forwarding:

```nginx
# ✅ Working — handles all subdomains
server_name  SUBDOMAIN.ghe.com *.SUBDOMAIN.ghe.com;
proxy_pass   https://$host;
Host         $host;
```

The key insight: `proxy_pass` and `Host` must use `$host` (or your proxy's equivalent) to forward requests to whichever subdomain the browser actually requested. Hardcoding the base domain means every subdomain's traffic gets misrouted.

### 3. Firewall / network rules

Ensure your firewall or NSG allows traffic to `*.SUBDOMAIN.ghe.com`, not just the base domain.

## The deeper lesson

This pattern keeps recurring because it's a topology assumption that doesn't transfer between platforms. GHES consolidates services behind one hostname. GHEC Data Residency distributes them. Both are valid architectures, but your proxy config must match the one you're actually running.

The GitHub docs do state the wildcard requirement. But a single line in a hostname table doesn't convey the operational consequence: if you miss this, your deployment will look healthy while a key capability is silently broken, and your standard diagnostic tools won't point you to the cause.

In GHEC Data Residency, **hostname topology is part of the architecture**. If your proxy only understands the base domain, you have a partial deployment — you just don't know it yet.

---

If you're running GHEC Data Residency behind a proxy or WAF, the two `nslookup` commands above take 10 seconds. It's worth checking.
