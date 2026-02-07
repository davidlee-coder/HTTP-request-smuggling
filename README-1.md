# HTTP Request Smuggling – Basic TE.CL Vulnerability

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

**Level** — Practitioner<p align="center"></i></p>
<br>

**Category** — HTTP Request Smuggling<p align="center"></i></p>
<br>
**PortSwigger Link** — [https://portswigger.net/web-security/request-smuggling/lab-basic-te-cl](https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te)<p align="center"></i></p>
<br>
**Completed** — February 6 2026<p align="center"></i></p>
<br>
**Tools** — Burp Repeater (Update Content-Length unchecked),HTTP Smuggler extention (for
confirmation), Documentation: [Flameshot](https://flameshot.org) (Screen capture & annotation)<p align="center"></i></p>
<br>

# Table of Contents

- [Overview](#overview)
- [My Aha Moment](#my-aha-moment)
- [Exploitation](#exploitation)
- [Root Cause](#root-cause)
- [Impact](#impact)
- [Mitigation](#mitigation)

# Overview
In the TE.CL variant of HTTP Request Smuggling, the front-end server prefers
Transfer-Encoding: chunked and ignores or overrides Content-Length, while the back-end server strictly honors Content-Length and ignores chunked encoding. This mismatch lets an attacker smuggle a second request inside the chunked body: the front-end sees one complete request ending at 0 chunk, but the back-end sees extra content after the stated Content-Length as a new, separate request. This direction of desync is especially powerful in environments where front-ends reverse proxies, CDNs default to chunked parsing, while legacy or misconfigured back-ends stick to Content-Length a common real-world combination.

# Aha Moment
The breakthrough came when I realized the parsers weren’t just disagreeing on length they were choosing different headers based on their own internal rules. In CL.TE the front-end stops early (CL), back-end reads more (TE). In TE.CL it’s reversed: front-end reads everything (TE), back-end stops early (CL). That direction flip wasn’t a minor variation it was a completely different attack surface. Once I internalized that many real-world setups have front-ends (Nginx, Cloudflare) preferring chunked and back-ends (Apache, old Tomcat) sticking to Content-Length, the whole smuggling class felt less like tricks and more like a predictable consequence of inconsistent HTTP specs. Now every time I see mixed CL and TE headers in a proxy chain, I immediately test both directions.

# Exploitation
 
I was messing around the homepage and wanted a quik way to spot desync points, so I fired up the HTTP Request Smuggler extension. I wasn't just looking for bugs I was looking for timing offsets where the server hung just a second too long, indicating a protocol-level disagreement. Once the extension flagged a TE.CL discrepancy, I moved the request over to Repeater to prove it wasn't a false positive:
 <img width="1255" height="683" alt="image" src="https://github.com/user-attachments/assets/61858b11-1f52-4d7b-b48d-94defa982531" />

 <img width="1030" height="485" alt="image" src="https://github.com/user-attachments/assets/2f6d5598-4e9f-4ce8-9810-a3f0545d685f" />

<img width="1083" height="658" alt="image" src="https://github.com/user-attachments/assets/9e5fd536-865f-4e71-95c1-b1b08910ae39" />
<p align="center"></i></p>
<br><br>


During the verification phase in Burp Repeater, I encountered a 400 Bad Request indicated in the third screenshot. This was a pivotal moment where I realized that Request Smuggling is a game of byte-perfect alignment.To exploit a H2.TE vulnerability, you have to force the front-end and back-end to disagree. I had to ensure Burp Suite didn't 'fix' my payload. By unchecking 'Update Content-Length', I essentially took off the safety rails. This gave me full control over the raw byte stream, allowing me to smuggle a hidden GET request inside the body of a POST without the tool correcting my intentional 'errors' back to a valid state.
In a TE.CL attack, the Transfer-Encoding chunk size (in hex) must account for every character of the smuggled request, including the trailing \r\n. If the hex value is 1e (30 bytes), but the smuggled body is 31 bytes, the front-end parser desynchronizes from the payload itself, leading to an 'Invalid Request' error. 
3\n\r
GET /DAVID HTTP/1.1 \n\r 
X=Y\n\r
0\n\r
\n\r
<img width="1366" height="682" alt="image" src="https://github.com/user-attachments/assets/f5311599-46ba-40bd-b130-69a23741c0ac" />
<img width="1366" height="685" alt="image" src="https://github.com/user-attachments/assets/5130d929-cb0a-4b3c-806b-8ec1d5a4d24c" />
<img width="1358" height="660" alt="image" src="https://github.com/user-attachments/assets/8e1e171b-ec24-495a-ba77-c9986adf216a" />
<p align="center"></i></p>
<br><br>

Next, I resolved this by using the 'Hex Editor' tab in Burp inspector panel to ensure the byte count matched the hex chunk header exactly:

1a\n\r 
GET /DAVID HTTP/1.1 \n\r 
X=Y\n\r
0\n\r
\n\r
<img width="1030" height="659" alt="image" src="https://github.com/user-attachments/assets/39b823d4-9ff4-462a-8fab-5b89ee6f50fe" />

<p align="center"></i></p>
<br><br>

Initially, my attack appeared successful at the protocol level (returning 200 OK), but I wasn't seeing the 'smuggled' behavior (e.g., a 404 or session hijacking). I realized that while I had successfully desynchronized the servers, I hadn't prepared the payload for the back-end application logic.
To 'swallow' a victim's request, I had to ensure two things:

    Content-Type: application/x-www-form-urlencoded: This tells the back-end to treat the smuggled body as a series of parameters.
    Content-Length (Smuggled): 20: By setting a length larger than the actual smuggled body, I forced the back-end to 'hang' and wait for more data. When the victim's request arrived on the same connection, the back-end appended their headers to my smuggled parameter, effectively capturing their data and returning the response meant for my smuggled request to them: 
<img width="1027" height="697" alt="image" src="https://github.com/user-attachments/assets/4453850e-608e-4fcd-8bda-483bc6ed066c" />
<p align="center"></i></p>
<br><br>

To confirm the exploit, I executed a back-to-back request sequence. I sent the initial 'attack' request to poison the back-end socket buffer, immediately followed by a standard, legitimate POST request.
The result was a 404 Not Found response for the second request. Since the second request was targeting a valid path, receiving a 404 proved that the back-end had prepended my smuggled GET /DAVID (which does not exist) to the start of the legitimate request. This confirmed a successful socket poisoning and demonstrated how an attacker can hijack the execution context of any user sharing the same connection pool:
<img width="1030" height="659" alt="image" src="https://github.com/user-attachments/assets/e29dfdf3-693a-4f6c-8ef1-a87f8df07746" />

<img width="1027" height="697" alt="image" src="https://github.com/user-attachments/assets/4d85e75c-73eb-4752-84f5-f47ba0ee5eaf" />
<p align="center"></i></p>
<br><br>



# Root Cause
- Front-end prefers Transfer-Encoding: chunked (ignores Content-Length when both present)- Back-end prefers Content-Length (ignores Transfer-Encoding)
- No rejection or normalization of ambiguous requests with conflicting headers

# Impact

- Mass Account Takeover (Session Hijacking): By "swallowing" a victim's request, an attacker can capture the Session Cookies or Authorization Bearer tokens of legitimate users. These credentials are leaked into the attacker's smuggled parameters, allowing them to impersonate any user on the platform. PortSwigger's research on capturing requests highlights this as a primary critical risk.

- Bypassing Security Controls (WAF/ACL): Request smuggling allows an attacker to bypass front-end security filters. Because the front-end sees a "benign" request while the back-end executes the "smuggled" malicious request, internal-only endpoints (like /admin or /internal-api) become accessible to unauthorized external users.

- Web Cache Poisoning: If the site uses a Content Delivery Network (CDN) or a front-end cache, a smuggled response can be stored in the cache. This results in the "smuggled" content (such as a malicious JavaScript redirect) being served to thousands of users from the cache, long after the initial attack has ended.

- Credential Theft via Data Reflection: In scenarios where the application reflects user input (like a search bar or profile field), the "swallowed" victim data can be written directly to the database or displayed on-screen, allowing the attacker to exfiltrate private data at scale.

# Mitigations

- Header Normalization: Ensure that all servers in the chain (Load Balancers, WAFs, and Back-ends) follow the same precedence rules. Under RFC 9112, Transfer-Encoding must always take precedence over Content-Length.

- Strict Validation (Reject Ambiguity): Configure the front-end to reject (400 Bad Request) any incoming request that contains both Content-Length and Transfer-Encoding headers. This removes the "choice" that leads to desynchronization.

-Protocol Hardening (Strict Mode): Enable "strict" parsing in proxy software. For example, using Nginx's ignore_invalid_headers or Apache's HttpProtocolOptions Strict prevents attackers from using obfuscation (like spaces before colons) to hide headers.

- End-to-End HTTP/2: Transitioning to HTTP/2 significantly reduces risk because it uses a binary framing layer rather than string-based delimiters to define message boundaries. However, beware of H2.CL / H2.TE "downgrade" attacks if the back-end still uses HTTP/1.1.

- Connection Termination: Instead of reusing TCP connections (Keep-Alive) for multiple users, configure the proxy to use a fresh connection for each back-end request. While this impacts performance, it completely eliminates socket poisoning.


**Happy (ethical) Hacking!**
