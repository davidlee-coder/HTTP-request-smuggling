# HTTP/2 Request Smuggling – H2.TE Variant


**Level** — Expert  
**Category** — HTTP Request Smuggling / HTTP/2  
**PortSwigger Link** — https://portswigger.net/web-security/request-smuggling/advanced/http2-request-smuggling  
**Completed** — February 2026  
**Tools** — Burp Suite (with HTTP/2 support), HTTP Smuggler extension (for H2 testing)


## Overview


HTTP/2 request smuggling (specifically the **H2.TE** variant) exploits inconsistencies between how front-end servers (e.g., reverse proxies, load balancers) translate HTTP/2 frames into HTTP/1.1 for back-end servers. In H2.TE, the front-end converts HTTP/2 headers and body into HTTP/1.1 using **Transfer-Encoding: chunked**, while the back-end may interpret the request using **Content-Length** (or ignore chunked encoding entirely).  


This allows an attacker to smuggle additional requests by crafting HTTP/2 messages with misleading length indicators — the front-end forwards the full H2 stream as one request, but the back-end sees extra content as a new request after the stated length.  


The lab typically involves smuggling via H2 headers (e.g., pseudo-headers, padding, or body framing) to bypass front-end controls and reach back-end endpoints as a privileged user.


## Aha Moment


The breakthrough came when I realized HTTP/2 smuggling isn’t just “HTTP/1.1 with frames” — it’s a completely different attack surface because the front-end **rewrites** the protocol during H2 → H1 translation.  


I kept trying classic CL.TE/TE.CL payloads over HTTP/2 and getting nowhere — 400s, timeouts, or nothing. Then it clicked: the front-end was **adding** Transfer-Encoding: chunked during conversion, but the back-end was **ignoring** it and using Content-Length from the original H2 headers (or calculating it wrong).  


That moment when I crafted a payload with a short Content-Length in H2 pseudo-headers + a long chunked body, and the smuggled request executed on the back-end as admin — I literally paused and said “this is insane.” It wasn’t about header precedence anymore; it was about how proxies mangle H2 into H1. Now I see every H2 front-end as a potential smuggling gateway until proven otherwise.


## Root Cause


- Front-end servers (e.g., Nginx, Envoy, Cloudflare) translate HTTP/2 streams to HTTP/1.1 for back-end communication, often injecting or preferring **Transfer-Encoding: chunked**.  
- Back-end servers may ignore Transfer-Encoding (especially older or strict parsers) and rely solely on **Content-Length** — or miscalculate length from H2 DATA frames.  
- HTTP/2 pseudo-headers (:method, :path, :authority) and stream framing allow attackers to craft ambiguous length signals that survive translation differently on each side.  
- Lack of strict H2 → H1 normalization or rejection of ambiguous framing creates the desync opportunity.


## Impact


- **Bypass front-end security controls** (WAF, rate limiting, IP blocks) → smuggle requests reach back-end as privileged users.  
- **Response smuggling** → steal responses intended for other users (e.g., admin sessions, API keys).  
- **Cache poisoning** via H2 paths (especially when front-end caches based on H2 pseudo-headers).  
- **Privilege escalation** → execute admin-only actions (delete users, access internal endpoints).  
- **Chain with other vulns** → amplify impact (e.g., smuggle XSS payloads, SSRF, or desync into business logic flaws).  
In real environments (CDN + app server), this can lead to account takeover, data theft, or full compromise.

# Exploitation

I used POST /x to ensure the front-end would forward a request body during the HTTP/2 to HTTP/1.1 downgrade. By smuggling a GET request to a non-existent /x endpoint, I created a unique signature (404) that allowed me to verify the socket poisoning without interfering with valid application endpoints.
Initially, the socket poisoning failed because my smuggled HTTP/1.1 prefix lacked a Host header. While the front-end (H2) doesn't require a Host header in the same way, the back-end (H1.1) is RFC-compliant and requires it. This resulted in a 400 Bad Request, causing the server to close the connection before the victim's request could be prepended. I resolved this by crafting a valid HTTP/1.1 request within the H2 body, ensuring the back-end would hold the request in the buffer for the next incoming user.

My smuggling payload looked perfect, but the admin session just wouldn't trigger. After a few failed attempts, I realized I’d made a classic protocol mistake: Initially, I didn't get an obvious error message. I was sending my payload and receiving standard 200 OKs and 404s, but the admin session never appeared. It was a 'silent' failure.

I realized that by omitting the Host header in my smuggled HTTP/1.1 prefix, I hadn't crashed the connection, but I also hadn't created a valid request for the back-end to process. It was essentially 'junk' data sitting in the buffer. The server was ignoring my smuggled instructions and just processing my subsequent requests as if nothing had happened.

## Recommended Mitigations


- **Prefer end-to-end HTTP/2** (disable H2 → H1 downgrade where possible).  
- **Strict header normalization** in front-end proxies: reject ambiguous requests, enforce consistent length handling.  
- **Disable Transfer-Encoding: chunked** injection during H2 → H1 conversion (or force back-end to honor it).  
- **Use HTTP/2-only back-ends** or gateways with smuggling mitigations (e.g., Envoy strict mode, Nginx with `http2` + `proxy_http_version 1.1` + strict checks).  
- **Monitor for desync indicators** (unexpected 400/timeout patterns, anomalous response lengths).  
- **Vendor hardening** — apply latest patches for known H2 smuggling vectors in proxies/CDNs.


## Reflection


H2.TE felt like the next level after classic smuggling — same idea, but the protocol translation layer adds a whole new dimension of nastiness. The moment I saw the smuggled request execute over HTTP/2 after failing with H1 payloads was a reminder that modern stacks introduce new attack surfaces even as they try to improve security. It’s one of those vulns that makes you appreciate how fragile the entire HTTP ecosystem still is.


**Skills Demonstrated**:
- Understanding HTTP/2 → HTTP/1.1 translation desync  
- Effective use of HTTP Smuggler for H2 testing  
- Crafting H2-specific smuggling payloads  
- Reasoning about proxy rewriting behavior  
- Chaining H2 smuggling to privilege bypass
