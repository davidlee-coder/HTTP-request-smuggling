# HTTP request smuggling, confirming a CL.TE vulnerability via differential responses

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

**Level** — Practitioner<p align="center"></i></p>
<br>

**Category** — HTTP Request Smuggling<p align="center"></i></p>
<br>
**PortSwigger Link** — [https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te](https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te)<p align="center"></i></p>
<br>
**Completed** — February 3 2026<p align="center"></i></p>
<br>
**Tools** — Burp Repeater (Update Content-Length unchecked),HTTP Smuggler extention (for
confirmation), Documentation: [Flameshot](https://flameshot.org) (Screen capture & annotation) <p align="center"></i></p>
<br>

## Table of Contents

- [Overview](#overview)
- [Vulnerability Details](#vulnerability-details)
- [Root Cause Analysis](#root-cause-analysis)
- [Exploitation](#exploitation-assessment)
- [Impact](#impact)
- [Conclusion](#conclusion)
- [Mitigation Strategies](#mitigation-strategies)

# Overview

HTTP Request Smuggling is more than a header bypass; it is a desynchronization primitive born from protocol-level disagreements between server tiers. In a CL.TE configuration, the front-end treats the Content-Length header as the source of truth for request boundaries, while the back-end honors Transfer-Encoding: chunked.
By crafting a request that satisfies the front-end's byte count but includes a "0" chunk EOF for the back-end, I can strand malicious data in the back-end's TCP buffer. This "smuggled" data is then prepended to the next legitimate request. This allows an attacker to effectively hijack the execution context of subsequent users—enabling front-end security bypasses, response queue poisoning, and session hijacking.

# My Aha Moment 

For a long time, I viewed HTTP Request Smuggling as just a clever header trick—messing with Content-Length and Transfer-Encoding to confuse a server. But the moment the lightbulb truly clicked was when I stopped seeing it as a bug and started seeing it as a desynchronization primitive.
The breakthrough was realizing that the front-end and back-end aren't 'misreading' headers, they are perfectly following two different sets of logic on the exact same byte stream. When the front-end (CL) and back-end (TE) disagree on where Request A ends, the 'leftover' bytes don’t just vanish they become the prefix for the next request.
I stopped looking for 'smuggling' and started looking for protocol-level disagreement. Now, every time I encounter a reverse proxy/app server architecture, I don't just check for headers I ask: 'How does this stack maintain state on a multiplexed connection, and what happens if I force these two parsers to disagree on the boundary of the truth?'
That shift—from 'how do I bypass a filter?' to 'how do I desynchronize the stream?'—is what turned a complex vulnerability into a repeatable methodology

# Root Cause 

- Front-end server priotizes Content-Length over Transfer-Encoding or vice versa.

- Back-end server uses different precedence usually TE prefered. 

- No normalization or rejection ofambiguous requests with both headers present.

# Exploitation

I was poking around the homepage and wanted a faster way to spot desync points, so I fired up the HTTP Request Smuggler extension. I wasn't just looking for bugs I was looking for timing offsets where the server hung just a second too long, signaling a protocol-level disagreement.
Once the extension flagged a CL.TE discrepancy, I moved the request over to Repeater to prove it wasn't a false positive:
<img width="1363" height="686" alt="image" src="https://github.com/user-attachments/assets/99d9b489-224d-4e39-8130-d11cbdeac3d2" />

<img width="1031" height="417" alt="image" src="https://github.com/user-attachments/assets/c31b17ef-09b2-4ad9-ae5e-152c3239f586" />

<img width="1083" height="707" alt="image" src="https://github.com/user-attachments/assets/969e9bbe-4664-4c6b-9a2a-168e30941808" />
<p align="center"></i></p>
<br><br>


While testing in Repeater, I was trying to trigger a desync but kept hitting a wall. I realized I was still sending the request over HTTP/2, which handles request boundaries much more strictly than its predecessor. For a CL.TE attack to actually work in this environment, I had to force the connection back to HTTP/1.1.
Once I toggled the protocol and re-aligned my Content-Length and Transfer-Encoding headers, the attack clicked. Debugging that 'invisible' protocol mismatch was a great reminder that smuggling isn't just about the payload—it's about the specific way the front-end and back-end talk to each other.
<img width="1360" height="685" alt="image" src="https://github.com/user-attachments/assets/bea8b3ea-11c3-4b11-baa3-5531b99e820e" />

<img width="1361" height="686" alt="image" src="https://github.com/user-attachments/assets/bf442804-c5dc-424a-bc8b-a3f18e4f9a7e" />
<p align="center"></i></p>
<br><br>

Using Burp Repeater, I issued a dual-identity payload:

POST / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 27
Transfer-Encoding: chunked

0

GET /404 HTTP/1.1
X:Y

 1. The Front-end View (The "Envelope")
The reverse proxy acts as the gatekeeper. It relies on the Content-Length header.

    Boundary: It counts exactly 27 bytes (starting from the 0 down to the X:Y).
    Action: It sees one "legal" POST request and forwards the entire block to the back-end over a persistent connection.

2. The Back-end View (The "Splice")
The back-end server is configured to prioritize Transfer-Encoding: chunked.

    Request A (Termination): It starts reading the body and immediately sees the 0\r\n\r\n. In chunked encoding, this is the EOF (End-of-File). It processes the POST request as having an empty body and finishes.
    The "Leftovers": The back-end stops parsing at the 0, but the remaining bytes—GET /404 HTTP/1.1 don't disappear. They are left stranded in the TCP buffer.
<p align="center"></i></p>
<br>

By issuing the request twice, I essentially acted as both the attacker and the victim to prove the desync. The first request felt like a normal
200 OK, but it was secretly "priming" the back-end's TCP buffer with my smuggled GET /404 prefix.
When I sent the second request, the back-end reached into its buffer, grabbed those leftover bytes, and tacked them onto the start of my new request. Instead of seeing the POST I just sent, the server processed the smuggled command first, resulting in a 404 Not Found. Receiving that error on a perfectly valid POST request was the definitive proof that the request queue was poisoned.:
<img width="1361" height="686" alt="image" src="https://github.com/user-attachments/assets/9ebb4263-ea5f-41da-b364-fac6a82b5c66" />

<img width="1028" height="688" alt="image" src="https://github.com/user-attachments/assets/43e165a4-7554-4d3a-8ef8-9ab1cb3d2207" />

<img width="1349" height="685" alt="image" src="https://github.com/user-attachments/assets/a82aa951-2861-4ef9-8b15-9732b007bee3" />
<p align="center"></i></p>
<br><br>


# Impact

Request smuggling isn't just about a 404 error, it's about context hijacking. Because the "smuggled" request is prepended to the next user’s legitimate request, an attacker can:

-Bypass Front-end Security: Gain access to internal-only admin pages like /admin that the front-end proxy normally prevents access to, as the back-end believes the request is coming from a trusted source.

- Session Hijacking: Smuggle in a request that "captures" the next user's session cookie by prefixing a request to a logging endpoint that the attacker controls.

- Web Cache Poisoning: Force the front-end cache to store a malicious version of a page (like a login page with a script injection) by desynchronizing the response-to-request mapping.

# The Mitigations

The root cause is the "Logic Gap" between servers. To fix it, you have to close that gap.

-Enforce HTTP/2 End-to-End: The most robust fix is to use HTTP/2 for the entire request chain. Its binary framing mechanism makes it virtually impossible to "smuggle" boundaries.

- Disable HTTP/1.1 smuggling vectors: Reject ambiguous requests with both CL and TE.

- Strict Validation: Ensure the front-end and back-end are both configured to reject ambiguous requests that contain both a Content-Length and a Transfer-Encoding header.
    
- Use a WAF: Modern Web Application Firewalls (like AWS WAF or Cloudflare) have built-in protections to normalize or drop malformed "dual-length" requests before they hit your infrastructure.

**Happy (ethical) Hacking!**
