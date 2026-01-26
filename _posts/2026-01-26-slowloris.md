---
layout: post
title: How One Laptop Can Take Down an Entire Server
date: 2026-01-24 00:00:00
description: A deep dive into Slowloris HTTP attacks - how they work, why they're dangerous, and how to defend against them
tags: http tcp apache nginx ddos security
categories: cybersecurity
---

For some reason, the word "Denial of Service" always sounded cool to me. This led me to explore more about them and eventually dive deeper into the world of cybersecurity too. Growing up, I used to read about Denial of Service attacks happening quite regularly to almost every major organisation. While they are still quite common, the modern-day internet is distributed and robust enough to handle most of these attacks at scale. 

The ingenuity behind some of these attacks is what interested me the most, exploiting fundamental protocol behaviors and slowly building up something to take down a large-scale infrastructure. So that is what I will be discussing in this post. Coincidentally, this was also the topic of my first research internship.

Most DDoS attacks are loud, high-bandwidth floods. **Slowloris** is the silent killer—an insidious attack that exhausts server resources using minimal bandwidth. In a world of brute force, it proves that being slow can be a devastating weapon. To defend against it, you must ensure that slow clients are cheap for you and expensive for the attacker.

---

## The Anatomy of DoS Attacks

Before diving into Slowloris, let's understand the broader landscape. DoS attacks can be categorized by what resource they target:

### Bandwidth-Based Attacks

The brute force approach. Imagine your server has a 50 Mbps download capacity—it can only process 50 megabits of data per second. An attacker with 100 Mbps upload speed simply floods you with more data than you can handle.

The server spends all its time processing junk traffic, leaving nothing for legitimate users. ISPs will throttle you, firewalls will choke, and your users see nothing but timeouts.

**Reality check:** This requires massive resources. That's why attackers use botnets—fleets of compromised "zombie" devices all attacking simultaneously. Think SYN floods, DNS amplification, and the classic volumetric DDoS.

### Connection-Based Attacks

Every HTTP request is a TCP connection underneath. TCP is *stateful*—your server must remember each client: allocating memory, CPU cycles, file descriptors.

Servers can't maintain unlimited connections. They set a cap—maybe 150 concurrent connections for Apache. Hit that limit, and new users get "503 Service Unavailable."

**The naive attack:** Open 150 connections and do nothing. But servers aren't stupid—they timeout idle connections after a few seconds. Connection killed. Nice try.

**The clever attack:** What if you stay *just* active enough? Send one byte... wait... send another byte. The server thinks you're a slow client on a bad network. It waits patiently. Timer resets. Connection lives.

This is **Slowloris**.

### Vulnerability-Based Attacks

The surgical strike. Instead of exhausting resources, exploit a bug in the software itself—a malformed request that crashes the process, a regex that causes catastrophic backtracking, a buffer overflow.

One carefully crafted packet. Backend down. Cold restart takes minutes while users stare at error pages.

Examples: Node.js prototype pollution, OpenSSL Heartbleed, regex denial-of-service (ReDoS).

**This is the most dangerous category**—it requires the least bandwidth and causes the most damage.

---

## What Attackers Really Want

The goal is always the same: **exhaust resources** (CPU, memory, network bandwidth) so the backend can't serve legitimate users. The creativity lies in *how*:

| Strategy | How It Works | Difficulty |
|----------|--------------|------------|
| **Long-running requests** | Requests that take forever to process, starving others | Easy |
| **Crash the backend** | Exploit vulnerabilities to take the server down | Medium |
| **Exhaust max connections** | Hold all connection slots hostage | Medium |
| **Large responses** | Bloated payloads (1.5MB initial loads!) exhaust bandwidth | Easy |
| **Flood with requests** | Classic botnet volumetric attack | Hard (needs resources) |
| **Complex requests** | CPU-intensive operations (bad regex, heavy queries) | Medium |

Slowloris sits in the sweet spot: **easy to execute, hard to detect, devastating in impact**.

---

## What is Slowloris?

Slowloris, developed by security researcher Robert "RSnake" Hansen in 2009, is an application-layer denial-of-service attack that overwhelms web servers by opening multiple connections and keeping them alive with **partial HTTP requests**. Instead of flooding a server with traffic, it exhausts the server's connection pool by never completing requests.

**Key Characteristics:**
- **Attack Complexity:** Low (single machine, minimal bandwidth)
- **Detection Difficulty:** High (appears as legitimate slow traffic)
- **Impact Severity:** Critical (complete service unavailability)

---

## How Slowloris Works

At the TCP/HTTP level, Slowloris opens many TCP connections and then **never completes the HTTP request**. 

A normal HTTP request looks like this:
```http
GET / HTTP/1.1\r\n
Host: example.com\r\n
\r\n
```

The critical piece is the final `\r\n\r\n` (double CRLF) which signals "end of headers." Slowloris sends partial headers and *dribbles* more data slowly enough to reset the server's internal timeouts:

```http
GET / HTTP/1.1\r\n
Host: example.com\r\n
X-Header: partial
# no final \r\n\r\n ever sent...
```

Every 10-15 seconds, it sends another header line like `X-fake-header-N: value\r\n` to prevent the connection from timing out. The server thinks this client is just slow — and waits patiently. Eventually, **all connection slots are occupied** by these zombie requests, and legitimate users can't connect.

> **Why It's Stealthy:** Because the bandwidth used is tiny and HTTP requests look plausible (just slow), many defenses that only watch for traffic volume won't trigger alerts.

---

## Why Architecture Matters

### Apache: Thread/Process Model (Vulnerable-by-Default)

Classic Apache (prefork/worker MPMs) allocates **one thread or process per connection**. If many of these are stuck waiting for incomplete requests, you run out of workers:

```
MaxRequestWorkers 150 (default Apache 2.4)
→ 150 concurrent connections maximum
→ Slowloris opens 150 partial requests
→ All workers occupied
→ Legitimate requests rejected with "Server busy" (503)
```

Apache's default timeouts are generous — `KeepAliveTimeout` at 300 seconds on many distributions — giving Slowloris plenty of time to keep slots occupied.

**Memory Impact:** ~8-10 MB per Apache process means 150 workers can consume over 1GB of RAM just waiting for incomplete requests.

### Nginx: Event-Driven (Resilient but Not Immune)

Nginx uses an **event loop with non-blocking I/O**. It doesn't tie an OS thread to each connection — a single worker can multiplex thousands of connections efficiently:

```
worker_connections 1024 (per worker)
workers = CPU cores
→ Thousands of concurrent connections possible
→ Connections tracked in memory (~1-2 KB each)
→ Timeouts enforced at event loop level
```

This makes Slowloris far less effective against Nginx. However, even event-driven servers benefit from explicit timeouts and connection caps.

---

## Detection Indicators

Traditional volumetric DDoS indicators won't help here. Instead, watch for:

- **High number of open connections** that haven't completed headers
- **Many connections in the "reading headers" state** for extended periods
- **Connections lasting significantly longer** than normal request lifetimes
- A **sharp rise in TCP connections** without corresponding completed HTTP requests
- Unusual **distributions of connection durations** in logs

**Linux Quick Check:**
```bash
# Count connections by state
netstat -an | grep :80 | awk '{print $6}' | sort | uniq -c

# Or with ss (faster)
ss -tan | awk '{print $1}' | sort | uniq -c
```

---

## Defensive Configurations

The fundamental principle: **make slow clients expensive for attackers, cheap for you**.

What to think about:
- **Long-running requests:** Identify what might take a long time (CPU-bound? I/O-bound? Complex regex?)
- **Timeouts everywhere:** Request level, response level, database level. If something takes too long, kill it.
- **Rate limiting:** API gateways and reverse proxies are your friends
- **Assume malice:** Don't trust that clients will behave nicely

### Apache Defense: mod_reqtimeout

The primary defense for Apache is `mod_reqtimeout`, which enforces minimum data rates:

```apache
# /etc/httpd/conf.d/slowloris-hardening.conf

# Enable mod_reqtimeout
LoadModule reqtimeout_module modules/mod_reqtimeout.so

<IfModule mod_reqtimeout.c>
    # header=10-20 means: allow 10s to start, extend to 20s if client sends ≥500 B/s
    # body=10-20,minrate=500 does the same for request body
    RequestReadTimeout header=10-20,MinRate=500 body=20,MinRate=500
</IfModule>

# Tighten global timeouts
Timeout 30
KeepAliveTimeout 5
MaxKeepAliveRequests 50
```

Slow clients violating these thresholds receive a **408 Request Timeout**, freeing server resources.

**Optional: Per-IP Connection Limits with mod_antiloris:**
```apache
<IfModule mod_antiloris_module>
    IPTotalLimit 16
    ExemptIPs 127.0.0.1 ::1
</IfModule>
```

### Nginx Defense: Timeouts and Rate Limits

Nginx's event-driven architecture provides inherent protection, but tightening defaults adds another layer:

```nginx
# /etc/nginx/conf.d/slowloris-hardening.conf

http {
    # Strict timeouts for header and body reception
    client_header_timeout 10s;      # default 60s
    client_body_timeout 20s;        # default 60s
    keepalive_timeout 10s 10s;      # default 75s
    send_timeout 30s;

    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
    limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=10r/s;

    server {
        listen 80;
        
        # Limit concurrent connections per IP
        limit_conn conn_limit_per_ip 20;
        
        # Limit request rate per IP
        limit_req zone=req_limit_per_ip burst=20 nodelay;
        
        # Return 444 (close without response) for violations
        limit_req_status 444;
        limit_conn_status 444;
    }
}
```

### Firewall Connection Limits (iptables)

At the OS level, limit connections per IP:

```bash
# Limit concurrent connections per IP to 50
iptables -A INPUT -p tcp --dport 80 \
    -m connlimit --connlimit-above 50 --connlimit-mask 32 \
    -j REJECT --reject-with tcp-reset

# Rate limit NEW connections
iptables -A INPUT -p tcp --dport 80 -m state --state NEW \
    -m recent --set --name HTTP
    
iptables -A INPUT -p tcp --dport 80 -m state --state NEW \
    -m recent --update --seconds 60 --hitcount 20 --name HTTP \
    -j DROP
```

> **Caution:** Too-low limits may block legitimate users behind corporate NATs or mobile networks.

---

## Network Edge Defenses

### Reverse Proxies and CDNs

Placing a **reverse proxy (Nginx, HAProxy) or CDN (Cloudflare, AWS CloudFront)** in front of origin servers provides the strongest protection:

1. **Request Buffering:** Edge servers buffer complete requests before forwarding to origin
2. **Large Connection Pools:** CDNs have much larger capacity to absorb connection exhaustion
3. **Built-in Rate Limiting:** Most CDN/WAF services detect and block slow-rate attacks automatically

### Layer 7 vs Layer 4 Protection

**Layer 7 (Application Layer):** Services like Cloudflare terminate your TLS connection. Your DNS points to *their* servers first. They see everything—HTTP headers, request bodies, cookies. 

This deep visibility means they can:
- Analyze request patterns for malicious behavior
- Block suspicious requests before they reach you
- Distinguish between slow attackers and legitimately slow clients

*The trade-off:* Some organizations aren't comfortable with a third party seeing all their traffic.

**Layer 4 (Transport Layer):** Some services only inspect TCP/IP—source IP, destination port, connection patterns. No visibility into the actual HTTP content.

This is faster and more privacy-preserving, but provides less protection against application-layer attacks like Slowloris. If the malicious content is *inside* the HTTP request, Layer 4 can't see it.

**Cloudflare:** Enable DDoS protection at Security → DDoS → set sensitivity to "High"

**AWS ALB:** Application Load Balancer provides built-in Slowloris protection via request buffering

---

## The Modern Twist: HTTP/2 CONTINUATION Flood

In 2024, researchers discovered a new attack variant exploiting HTTP/2's CONTINUATION frames. By sending CONTINUATION frames without the END_HEADERS flag, attackers create an infinite header stream that servers must parse and store:

- Can cause out-of-memory errors with **minimal traffic**
- Often requires only a **single TCP connection**
- Multiple CVEs issued: CVE-2024-27983, CVE-2024-31309, CVE-2024-30255

**HTTP/2 Mitigation for Nginx:**
```nginx
server {
    listen 443 ssl http2;
    
    # Limit concurrent HTTP/2 streams
    http2_max_concurrent_streams 32;   # default 128
    http2_max_field_size 16k;
    http2_max_header_size 32k;
}
```

---

## The Hardened Stack: Putting It All Together

1. **Architecture** — Use event-driven servers (Nginx) or CDN as frontend
2. **Timeouts** — Enforce strict request timeouts (mod_reqtimeout, client_header_timeout)
3. **Connection Caps** — Limit connections per IP at application and firewall layers
4. **Monitoring** — Watch connection state metrics, not just volume
5. **Edge Protection** — Add Cloud WAF/DDoS services for additional filtering

---

## Conclusion

Slowloris is a masterclass in "low and slow" attacks — minimal bandwidth, low noise, but capable of completely exhausting server resources. The good news is that modern architectures (event-driven servers, CDNs, proper timeout configurations) make it much harder to succeed.

The key insight: **be less patient with connections**. Legitimate clients complete requests quickly. Servers that wait indefinitely for stragglers become perfect targets.

---

## References

1. [Wikipedia: Slowloris (cyber attack)](https://en.wikipedia.org/wiki/Slowloris_%28cyber_attack%29)
2. [NETSCOUT: What is a Slowloris Attack?](https://www.netscout.com/what-is-ddos/slowloris-attacks)
3. [Cloudflare: Slowloris DDoS Attack](https://www.cloudflare.com/learning/ddos/ddos-attack-tools/slowloris/)
4. [Apache mod_reqtimeout Documentation](https://httpd.apache.org/docs/2.4/mod/mod_reqtimeout.html)
5. [CERT/CC: HTTP/2 CONTINUATION Flood (VU#421644)](https://kb.cert.org/vuls/id/421644)