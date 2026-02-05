# HTTP Request Smuggling – Basic TE.CL Vulnerability

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

***Level** — Practitioner<p align="center"></i></p>
<br>

**Category** — HTTP Request Smuggling<p align="center"></i></p>
<br>
**PortSwigger Link** — [https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te](https://portswigger.net/web-security/request-smuggling/lab-basic-te-cl)<p align="center"></i></p>
<br>
**Completed** — February 6 2026<p align="center"></i></p>
<br>
**Tools** — Burp Repeater (Update Content-Length unchecked), HTTP Smuggler extention (for
confirmation)<p align="center"></i></p>
<br>

# Table of Contents

- [Overview](#overview)
- [Vulnerability Details](#vulnerability-details)
- [Root Cause Analysis](#root-cause-analysis)
- [Exploitation](#exploitation)
- [Impact](#impact)
- [Conclusion](#conclusion)
- [Mitigation](#mitigation)
- [Tools and Resources](#tools-and-resources)
- [References](#references)

## Overview
In the **TE.CL** variant of HTTP Request Smuggling, the front-end server prefers
**Transfer-Encoding: chunked** (ignores or overrides Content-Length), while the back-end
server strictly honors **Content-Length** (ignores chunked encoding). This mismatch lets an
attacker smuggle a second request inside the chunked body: the front-end sees one complete
request (ending at 0 chunk), but the back-end sees extra content after the stated
Content-Length as a new, separate request.
This direction of desync is especially powerful in environments where front-ends (reverse
proxies, CDNs) default to chunked parsing, while legacy or misconfigured back-ends stick to
Content-Length — a common real-world combination.

## Aha Moment
The breakthrough came when I realized the parsers weren’t just disagreeing on length — they
were **choosing different headers** based on their own internal rules. In CL.TE the front-end
stops early (CL), back-end reads more (TE). In TE.CL it’s reversed: front-end reads everything
(TE), back-end stops early (CL).
That direction flip wasn’t a minor variation — it was a completely different attack surface. Once I
internalized that many real-world setups have front-ends (Nginx, Cloudflare) preferring chunked
and back-ends (Apache, old Tomcat) sticking to Content-Length, the whole smuggling class felt
less like tricks and more like a predictable consequence of inconsistent HTTP specs. Now every
time I see mixed CL+TE headers in a proxy chain, I immediately test both directions.

# Exploitation Steps (My Journey)
After finishing CL.TE, I thought TE.CL would be almost identical — just flip the headers. Sent
the same payloads with TE first... and got nothing but 400s and timeouts. Tried increasing
chunk sizes, adding dummy headers, even copying CL.TE payloads verbatim — still dead.
About 30 minutes in I was doubting if TE.CL was actually different or if I’d broken something.
Then I unchecked “Update Content-Length” in Repeater (huge oversight from CL.TE muscle
memory) and rebuilt the payload carefully: small CL (4), long chunked body ending with 0, full
GPOST inside. Sent twice... and the second response was a 302 to /admin. I stared at it for a
second, then laughed — it actually worked. That feeling when the smuggled request landed as
the victim user after the flip was incredibly satisfying.

- [Probe requests showing inconsistent desync](screenshots/01-probe-te-cl.png)
- [Final TE.CL payload in Repeater](screenshots/02-te-cl-payload.png)
- [Smuggled GPOST executed – 302 to admin](screenshots/03-smuggled-redirect.png)
- [Captured admin response & lab solved](screenshots/04-solved.png)
## Root Cause
- Front-end prefers Transfer-Encoding: chunked (ignores Content-Length when both present)- Back-end prefers Content-Length (ignores Transfer-Encoding)
- No rejection or normalization of ambiguous requests with conflicting headers
## Recommended Mitigations
- Normalize header precedence consistently across all servers (e.g., always prefer TE or always
CL)
- Reject requests containing both Content-Length and Transfer-Encoding
- Disable HTTP/1.1 smuggling vectors in proxies (strict mode in Nginx/Apache)
- Prefer HTTP/2 end-to-end (many H2 → H1 gateways mitigate smuggling)
- Monitor for desync patterns (unexpected redirects, response smuggling)
## Reflection
Switching from CL.TE to TE.CL drove home how subtle header precedence can completely
invert an attack. The moment the second request smuggled through after the flip was a
reminder that these vulns aren’t about one trick — they’re about exploiting disagreement
between systems that should speak the same language. It’s one of those topics that feels
abstract until you see both sides, then it becomes terrifyingly practical.
**Skills Demonstrated**:
- Understanding bidirectional desync (CL.TE vs TE.CL)
- Manual Repeater crafting with precise header control
- Reasoning about front-end vs back-end HTTP parser differences
- Chaining smuggling to privilege bypass / response theft
