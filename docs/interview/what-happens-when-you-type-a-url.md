---
title: What Happens When You Type a URL
description: Follow a browser request through DNS, TCP, TLS, HTTP, server processing, and rendering with commands that expose each step.
tags:
  - networking
  - dns
  - tls
  - http
---

# What Happens When You Type a URL

A URL request crosses several systems before a page appears. During an incident, split that path into DNS, connection, TLS, HTTP, and application stages so you can prove which stage is failing.

## Quick Reference

| Command | What it does |
|---------|--------------|
| `getent ahosts example.com` | Resolves the hostname through the host's configured name service |
| `dig example.com A` | Shows DNS answer records and response metadata |
| `dig +trace example.com` | Walks delegation from the root servers to the authoritative server |
| `openssl s_client -connect example.com:443 -servername example.com` | Opens a TLS connection with SNI and prints the certificate chain |
| `curl -v https://example.com/` | Shows connection, TLS, request, and response details |
| `curl -I https://example.com/` | Fetches response headers without downloading the body |

## Core Concepts

### Parse and resolve

The browser splits the URL into scheme, hostname, port, path, and query. It checks browser and operating-system caches before asking the resolver configured for the host. The recursive resolver may answer from cache or query root, top-level-domain, and authoritative servers until it finds an address.

Resolve through the same name-service path used by most applications:

```bash
getent ahosts example.com
```

### Connect and negotiate TLS

HTTPS normally opens TCP port 443 with a three-way handshake. TLS then negotiates a protocol and cipher, validates the server certificate against trusted certificate authorities, and derives symmetric session keys. SNI carries the hostname before HTTP so a shared IP can select the correct certificate.

Inspect the negotiated TLS version, cipher, certificate subject, and issuer:

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E '^(subject=|issuer=|New,|Protocol|Cipher)'
```

### Send HTTP and render

The browser sends an HTTP request containing the method, path, headers, and cookies. A CDN or reverse proxy may terminate TLS and forward the request to an application, which may call caches, databases, or other services before returning a status, headers, and body. The browser parses HTML, fetches referenced resources, builds the page, runs JavaScript, and paints pixels.

Show request headers, response headers, redirects, and protocol negotiation:

```bash
curl -vL --max-redirs 5 https://example.com/ -o /dev/null
```

## Common Scenarios

### DNS points to the wrong address

Compare the normal recursive answer with the authoritative delegation path:

```bash
dig example.com A +noall +answer && dig +trace example.com
```

### The certificate fails only for one hostname

Send the hostname with SNI and print the chain verification result:

```bash
echo | openssl s_client -connect example.com:443 -servername example.com -verify_return_error
```

### The request is slow

Break total latency into name lookup, TCP connection, TLS, first byte, and total time:

```bash
curl -sS -o /dev/null -w 'dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s first_byte=%{time_starttransfer}s total=%{time_total}s\n' https://example.com/
```

### DNS is correct but the target server must be tested directly

Override DNS for one request while preserving the URL hostname, Host header, and TLS SNI:

```bash
IP=$(getent ahostsv4 example.com | awk 'NR==1 {print $1}'); curl -v --resolve "example.com:443:$IP" https://example.com/ -o /dev/null
```

## Gotchas

- **DNS cache layers differ**: The browser, operating system, local resolver, and recursive resolver can hold different answers until their TTLs expire.
- **A successful ping proves little**: ICMP can be blocked while TCP works, and a ping does not test TLS, HTTP, or the application.
- **An IP-only TLS test can mislead**: Shared endpoints need SNI to select the expected certificate and virtual host.
- **HTTP success can hide dependency latency**: A `200` response does not show whether the application spent most of its time waiting on a database.
- **HTTP/3 changes the transport**: A browser may use QUIC over UDP after discovering support, while `curl` on Ubuntu 22.04 commonly tests HTTP/2 or HTTP/1.1 over TCP.

## Related Challenges

<div class="practice-cta" markdown>

**No matching hands-on challenge is published yet.**

Practice other Linux and production failure modes in an isolated terminal.

[Browse Paged Again challenges](https://pagedagain.com/incidents?utm_source=runbooks&utm_medium=concept&utm_campaign=what-happens-when-you-type-a-url){ .md-button .md-button--primary }

</div>

<a class="star-cta" href="https://github.com/pagedagain/sre-handbook">Found this useful? <span class="star-cta-link">Star the handbook repo</span> to help other SREs find it.</a>
