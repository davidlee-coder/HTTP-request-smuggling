# HTTP/2 Request Smuggling – H2.TE Variant
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

**Level** — Expert  
**Category** — HTTP Request Smuggling / HTTP/2  
**PortSwigger Link** — https://portswigger.net/web-security/request-smuggling/advanced/http2-request-smuggling  
**Completed** — February 8 2026  
**Tools** — Burp Suite (with HTTP/2 support), HTTP Smuggler extension (for H2 testing)


# Overview

HTTP/2 request smuggling (specifically the **H2.TE** variant) exploits inconsistencies between how front-end servers (e.g., reverse proxies, load balancers) translate HTTP/2 frames into HTTP/1.1 for back-end servers. In H2.TE, the front-end converts HTTP/2 headers and body into HTTP/1.1 using **Transfer-Encoding: chunked**, while the back-end may interpret the request using**Content-Length** (or ignore chunked encoding entirely). This allows an attacker to smuggle additional requests by crafting HTTP/2 messages with misleading length indicators — the front-end forwards the full H2 stream as one request, but the back-end sees extra content as a new request after the stated length.  
The lab typically involves smuggling via H2 headers (e.g., pseudo-headers, padding, or body framing) to bypass front-end controls and reach back-end endpoints as a privileged user.

# Aha Moment

The breakthrough came when I realized HTTP/2 smuggling isn’t just “HTTP/1.1 with frames” it’s a completely different attack surface because the front-end **rewrites** the protocol during H2 → H1 translation. I kept trying classic CL.TE/TE.CL payloads over HTTP/2 and getting nowhere 400s, timeouts, or nothing. Then it clicked: the front-end was **adding** Transfer-Encoding: chunked during conversion, but the back-end was **ignoring** it and using Content-Length from the original H2 headers (or calculating it wrong). That moment when I crafted a payload with a short Content-Length in H2 pseudo-headers and a long chunked body, and the smuggled request executed on the back-end as admin I literally paused and said “this is insane.” It wasn’t about header precedence anymore; it was about how proxies mangle H2 into H1. Now I see every H2 front-end as a potential smuggling gateway until proven otherwise.

# Root Cause

- Front-end servers (e.g., Nginx, Envoy, Cloudflare) translate HTTP/2 streams to HTTP/1.1 for back-end communication, often injecting or preferring **Transfer-Encoding: chunked**.  
- Back-end servers may ignore Transfer-Encoding (especially older or strict parsers) and rely solely on **Content-Length** — or miscalculate length from H2 DATA frames.  
- HTTP/2 pseudo-headers (:method, :path, :authority) and stream framing allow attackers to craft ambiguous length signals that survive translation differently on each side.  
- Lack of strict H2 → H1 normalization or rejection of ambiguous framing creates the desync opportunity.


# Impact

- **Bypass front-end security controls** (WAF, rate limiting, IP blocks) → smuggle requests reach back-end as privileged users.  
- **Response smuggling** → steal responses intended for other users (e.g., admin sessions, API keys).  
- **Cache poisoning** via H2 paths (especially when front-end caches based on H2 pseudo-headers).  
- **Privilege escalation** → execute admin-only actions (delete users, access internal endpoints).  
- **Chain with other vulns** → amplify impact (e.g., smuggle XSS payloads, SSRF, or desync into business logic flaws).
    
In real environments (CDN + app server), this can lead to account takeover, data theft, or full compromise.

# Exploitation

After discovering a restricted /admin endpoint returning a 401 Unauthorized, I pivoted my strategy. I wasn't just looking for a simple bypass; I was hunting for a protocol-level disagreement between the front-end and back-end.
I utilized the HTTP Request Smuggler extension to scan for discrepancies, and it flagged a potential H2.TE vulnerability. To verify this wasn't a false positive, I moved the request into Burp Repeater for manual confirmation.

To 'poison' the stream, I changed the method to POST and injected a chunked end-marker (0) followed by an arbitrary prefix—GINGER—into the HTTP/2 body.
http

POST / HTTP/2
Host: LAB-ID.web-security-academy.net
Transfer-Encoding: chunked

0

GINGER
<img width="1366" height="611" alt="HOME PAGE" src="https://github.com/user-attachments/assets/9f4e4d1f-ae30-4f8a-8ba1-d4a4708d4d9f" />  
<img width="1033" height="685" alt="image" src="https://github.com/user-attachments/assets/c2fb9f1d-8183-4767-843a-9994ee381cbd" />
<img width="1055" height="658" alt="image" src="https://github.com/user-attachments/assets/41b93b29-ed65-4ba9-872c-c87485c82059" />
<img width="1034" height="679" alt="image" src="https://github.com/user-attachments/assets/fb3c1b6d-5434-426f-898c-1d8b2db05cf6" />
<p align="center"></i></p>
<br><br>


After injecting my GINGER prefix, I began a series of follow-up requests to observe the behavior of the back-end. The result was a textbook desync: every second request I sent returned a 404 response.
This 50/50 split confirmed that my smuggled data was successfully 'stuck' in the back-end's buffer. The server was attempting to process a request starting with my prefix (e.g., GINGERGET /), which predictably failed. This gave me the mathematical certainty I needed, I had full control over the back-end socket.
Now that the desync was proven, the real challenge began: turning a 'noisy' 404 into a silent session hijack by crafting a perfectly valid, smuggled HTTP/1.1 request.
<img width="1034" height="679" alt="image" src="https://github.com/user-attachments/assets/8bacc6c6-5d8d-414a-ad9c-6437eb4f7383" />
<img width="1026" height="681" alt="image" src="https://github.com/user-attachments/assets/dc2e4d8d-c573-446d-96fa-a0b846de79a6" />
<p align="center"></i></p>
<br><br>

To verify the socket poisoning without interfering with legitimate application traffic, I used a custom POST /MOM request. My goal was to ensure the front-end would forward the request body during the HTTP/2 to HTTP/1.1 downgrade. By smuggling a GET request to this non-existent endpoint, I created a unique '404 Signature.' This allowed me to definitively prove the desync was working whenever that specific 404 appeared in my response stream.
Initially, the poisoning was inconsistent. I realized that my smuggled HTTP/1.1 prefix lacked a mandatory Host header. While HTTP/2 handles hostnames via pseudo-headers (:authority), the back-end (HTTP/1.1) is strictly RFC-compliant. Without a Host header, the back-end was essentially seeing 'malformed' data and resetting the connection before the victim's request could be prepended.My smuggling payload looked perfect, but the admin session just wouldn't trigger. After a few failed attempts, I realized I’d made a classic protocol mistake: Initially, I didn't get an obvious error message. I was sending my payload and receiving standard 200 OKs and 404s, but the admin session never appeared. It was a 'silent' failure.
<img width="1025" height="690" alt="image" src="https://github.com/user-attachments/assets/630bcde5-c155-48f1-adc5-1bf35143381e" />
<p align="center"></i></p>
<br><br>

To move from a 'Proof of Concept' to a full exploit, I crafted a fully valid, RFC-compliant HTTP/1.1 request (complete with a required Host header) within the H2 body. This 'legal' request forced the back-end to maintain the socket connection and hold my smuggled data in the buffer, ready to be 'sewn' onto the next incoming user's request.
After several attempts navigating the connection-pool lottery I successfully intercepted a 302 Redirect containing the admin's post-login session cookie.
<img width="1029" height="685" alt="image" src="https://github.com/user-attachments/assets/9fe9f70e-7861-4909-9f36-767a6336ba83" />
<p align="center"></i></p>
<br><br>

I extracted the admin's session cookie and appended it to a standard HTTP/2 request. Upon receiving a 200 OK, I gained full access to the admin panel. I identified the administrative endpoint for user management (/admin/delete?username=carlos) and forwarded the hijacked session to my browser. By successfully deleting the user 'carlos,' I confirmed a critical H2.TE vulnerability leading to full administrative compromise and unauthorized privilege escalation.
<img width="1028" height="683" alt="image" src="https://github.com/user-attachments/assets/8445ddcb-3a42-44c1-b9ca-ac62578c4f1b" />
<img width="1282" height="539" alt="image" src="https://github.com/user-attachments/assets/00cde791-4ca2-49f3-bcdf-d7ec08db3e7f" />
<img width="525" height="519" alt="image" src="https://github.com/user-attachments/assets/bd987676-07ce-463d-b8e9-3d8acc7964f5" />
<img width="1326" height="576" alt="image" src="https://github.com/user-attachments/assets/1259714c-b4a4-4f3b-a6c9-a88597cd332b" />
<img width="1285" height="408" alt="image" src="https://github.com/user-attachments/assets/6d1cb3f7-f341-494b-8497-a7a9cc419df5" />
<img width="1262" height="486" alt="image" src="https://github.com/user-attachments/assets/5a8de84d-b292-4f75-b986-f46f8229bb16" />
<p align="center"></i></p>
<br><br>

# Mitigations


- **Prefer end-to-end HTTP/2** (disable H2 → H1 downgrade where possible).  
- **Strict header normalization** in front-end proxies: reject ambiguous requests, enforce consistent length handling.  
- **Disable Transfer-Encoding: chunked** injection during H2 → H1 conversion (or force back-end to honor it).  
- **Use HTTP/2-only back-ends** or gateways with smuggling mitigations (e.g., Envoy strict mode, Nginx with `http2` + `proxy_http_version 1.1` + strict checks).  
- **Monitor for desync indicators** (unexpected 400/timeout patterns, anomalous response lengths).  
- **Vendor hardening** — apply latest patches for known H2 smuggling vectors in proxies/CDNs.





