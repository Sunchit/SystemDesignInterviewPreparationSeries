# Forward Proxy vs Reverse Proxy: The Gatekeepers of Modern Internet Architecture
### Day 29 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 29!

Yesterday, we mastered consistent hashing for resharding distributed systems. Today, we explore **Forward Proxy and Reverse Proxy** — two fundamental networking concepts that often confuse even experienced engineers. Understanding the difference is crucial for system design interviews and real-world architecture.

> The key insight: Forward proxies act on behalf of **clients**, while reverse proxies act on behalf of **servers**. This single distinction explains everything else.

---

## 🔄 THE FUNDAMENTAL DIFFERENCE

```
┌─────────────────────────────────────────────────────────────────┐
│                 PROXY DIRECTION MATTERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   FORWARD PROXY (Client-Side)                                    │
│   ────────────────────────────                                   │
│   Client → [Forward Proxy] → Internet → Server                  │
│                                                                  │
│   • Proxy knows the CLIENT                                       │
│   • Server sees PROXY's IP, not client's                        │
│   • Client KNOWS it's using a proxy                             │
│                                                                  │
│   ═══════════════════════════════════════════════════════════   │
│                                                                  │
│   REVERSE PROXY (Server-Side)                                    │
│   ────────────────────────────                                   │
│   Client → Internet → [Reverse Proxy] → Server                  │
│                                                                  │
│   • Proxy knows the SERVER                                       │
│   • Client sees PROXY's IP, not server's                        │
│   • Client DOESN'T KNOW there's a proxy                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔵 FORWARD PROXY: CLIENT'S REPRESENTATIVE

### What is a Forward Proxy?

A forward proxy sits between clients and the internet, acting as an intermediary that makes requests **on behalf of clients**.

```
┌─────────────────────────────────────────────────────────────────┐
│                 FORWARD PROXY FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌────────┐     ┌───────────────┐          ┌────────────┐     │
│   │        │     │               │          │            │     │
│   │ Client │────▶│ Forward Proxy │─────────▶│  Internet  │     │
│   │        │     │               │          │            │     │
│   └────────┘     └───────────────┘          └─────┬──────┘     │
│   IP: 192.168.1.5    IP: 203.0.113.50             │            │
│                                                    ▼            │
│                                            ┌────────────┐       │
│                                            │   Server   │       │
│                                            │            │       │
│                                            └────────────┘       │
│                                            Sees: 203.0.113.50   │
│                                            (Proxy IP, not       │
│                                             client IP!)         │
│                                                                  │
│   KEY POINT: Server has NO IDEA who the real client is!        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Forward Proxy Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| **Anonymity** | Hide client's real IP address | VPN services, Tor network |
| **Access Control** | Block employees from certain sites | Corporate firewalls |
| **Caching** | Cache frequently accessed content | ISP caching proxies |
| **Bypass Restrictions** | Access geo-blocked content | Accessing Netflix from different regions |
| **Monitoring** | Log all outbound traffic | Corporate compliance |
| **Bandwidth Savings** | Cache and compress responses | School/university networks |

### Real-World Forward Proxy Examples

```
┌─────────────────────────────────────────────────────────────────┐
│                 FORWARD PROXY EXAMPLES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. CORPORATE PROXY                                             │
│   ──────────────────                                             │
│   Employee → Corporate Proxy → Internet                          │
│   • Blocks social media during work hours                       │
│   • Logs all browsing history                                   │
│   • Scans downloads for malware                                 │
│                                                                  │
│   2. VPN (Virtual Private Network)                               │
│   ────────────────────────────────                               │
│   Your Device → VPN Server → Internet                           │
│   • Encrypts all traffic                                        │
│   • Masks your real IP                                          │
│   • Bypasses geo-restrictions                                   │
│                                                                  │
│   3. TOR NETWORK                                                 │
│   ──────────────                                                 │
│   You → Node1 → Node2 → Node3 → Internet                        │
│   • Multiple layers of encryption                               │
│   • Extreme anonymity                                           │
│   • Each node only knows previous/next hop                      │
│                                                                  │
│   4. SQUID PROXY (Enterprise)                                    │
│   ───────────────────────────                                    │
│   • Caches web content                                          │
│   • Reduces bandwidth usage                                     │
│   • Content filtering                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Forward Proxy Configuration Example

```python
# Python - Using a forward proxy with requests
import requests

proxies = {
    'http': 'http://proxy.company.com:8080',
    'https': 'http://proxy.company.com:8080',
}

# All requests go through the proxy
response = requests.get('https://api.example.com/data', proxies=proxies)

# With authentication
proxies_auth = {
    'http': 'http://user:password@proxy.company.com:8080',
    'https': 'http://user:password@proxy.company.com:8080',
}
```

```bash
# Environment variables for system-wide proxy
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,.internal.company.com"
```

---

## 🟢 REVERSE PROXY: SERVER'S REPRESENTATIVE

### What is a Reverse Proxy?

A reverse proxy sits in front of servers, accepting requests from clients and forwarding them to backend servers. Clients don't know they're talking to a proxy.

```
┌─────────────────────────────────────────────────────────────────┐
│                 REVERSE PROXY FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌────────┐          ┌───────────────┐     ┌────────────┐     │
│   │        │          │               │     │  Server 1  │     │
│   │ Client │─────────▶│ Reverse Proxy │────▶│────────────│     │
│   │        │          │               │     │  Server 2  │     │
│   └────────┘          └───────────────┘     │────────────│     │
│   Sees: proxy.com     IP: 203.0.113.50      │  Server 3  │     │
│                                              └────────────┘     │
│                                              Hidden from        │
│                                              client!            │
│                                                                  │
│   Client thinks: "I'm talking to proxy.com"                     │
│   Reality: Proxy forwards to actual servers                     │
│                                                                  │
│   KEY POINT: Client has NO IDEA about backend servers!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Reverse Proxy Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| **Load Balancing** | Distribute traffic across servers | Nginx, HAProxy |
| **SSL Termination** | Handle HTTPS at proxy level | Reduces server CPU load |
| **Caching** | Cache responses to reduce server load | CDN edge servers |
| **Security** | Hide backend infrastructure | DDoS protection |
| **Compression** | Compress responses before sending | Gzip at proxy level |
| **API Gateway** | Route, authenticate, rate limit | Kong, AWS API Gateway |
| **A/B Testing** | Route users to different versions | Canary deployments |

### Real-World Reverse Proxy Examples

```
┌─────────────────────────────────────────────────────────────────┐
│                 REVERSE PROXY IN ACTION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. CLOUDFLARE                                                  │
│   ─────────────                                                  │
│   User → Cloudflare Edge → Your Origin Server                   │
│   • DDoS protection (absorbs attack traffic)                    │
│   • CDN caching (serves static content)                         │
│   • SSL/TLS termination                                         │
│   • WAF (Web Application Firewall)                              │
│                                                                  │
│   2. NGINX AS REVERSE PROXY                                      │
│   ──────────────────────────                                     │
│   User → Nginx → [App Server 1, App Server 2, App Server 3]     │
│   • Load balancing (round-robin, least-conn)                    │
│   • Health checks (remove unhealthy servers)                    │
│   • Request buffering                                           │
│                                                                  │
│   3. AWS APPLICATION LOAD BALANCER                               │
│   ────────────────────────────────                               │
│   User → ALB → [EC2 instances, ECS containers, Lambda]          │
│   • Path-based routing (/api → service A, /web → service B)    │
│   • SSL termination with ACM certificates                       │
│   • Auto-scaling integration                                    │
│                                                                  │
│   4. KUBERNETES INGRESS                                          │
│   ─────────────────────────                                      │
│   User → Ingress Controller → [Pod 1, Pod 2, Pod 3...]          │
│   • Routes based on hostname and path                           │
│   • TLS termination                                             │
│   • Service discovery                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ REVERSE PROXY: DETAILED CAPABILITIES

### 1. Load Balancing

```
┌─────────────────────────────────────────────────────────────────┐
│                 LOAD BALANCING STRATEGIES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Round Robin                                                    │
│   ────────────                                                   │
│   Request 1 → Server A                                          │
│   Request 2 → Server B                                          │
│   Request 3 → Server C                                          │
│   Request 4 → Server A  (cycles back)                           │
│                                                                  │
│   Least Connections                                              │
│   ─────────────────                                              │
│   Server A: 10 connections                                       │
│   Server B: 5 connections  ← New request goes here              │
│   Server C: 8 connections                                        │
│                                                                  │
│   IP Hash (Sticky Sessions)                                      │
│   ─────────────────────────                                      │
│   hash(client_ip) % num_servers = target_server                 │
│   Same client always hits same server                           │
│                                                                  │
│   Weighted                                                       │
│   ────────                                                       │
│   Server A (weight: 5): Gets 50% of traffic                     │
│   Server B (weight: 3): Gets 30% of traffic                     │
│   Server C (weight: 2): Gets 20% of traffic                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. SSL/TLS Termination

```
┌─────────────────────────────────────────────────────────────────┐
│                 SSL TERMINATION                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WITHOUT SSL Termination:                                       │
│   ─────────────────────────                                      │
│   Client ══HTTPS══▶ Proxy ══HTTPS══▶ Server 1                   │
│                           ══HTTPS══▶ Server 2                   │
│                           ══HTTPS══▶ Server 3                   │
│   Problem: Each server needs SSL certificate, CPU overhead      │
│                                                                  │
│   WITH SSL Termination:                                          │
│   ─────────────────────                                          │
│   Client ══HTTPS══▶ Proxy ──HTTP──▶ Server 1                    │
│                          ──HTTP──▶ Server 2                     │
│                          ──HTTP──▶ Server 3                     │
│   Benefit: One certificate, reduced server CPU, easier mgmt    │
│                                                                  │
│   ⚠️ Security Note: Internal traffic is unencrypted            │
│   Solution: Use private network or re-encrypt (SSL passthrough) │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Caching

```
┌─────────────────────────────────────────────────────────────────┐
│                 REVERSE PROXY CACHING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   First Request:                                                 │
│   ──────────────                                                 │
│   Client → Proxy → Server                                        │
│                  ← Response (Cache-Control: max-age=3600)       │
│          ← Response (also stored in proxy cache)                │
│                                                                  │
│   Subsequent Requests (within 1 hour):                          │
│   ─────────────────────────────────────                          │
│   Client → Proxy                                                 │
│          ← Response (served from cache!)                        │
│          Server never touched!                                   │
│                                                                  │
│   Benefits:                                                      │
│   • Reduced server load                                          │
│   • Faster response times                                        │
│   • Lower bandwidth costs                                        │
│                                                                  │
│   Cache Headers:                                                 │
│   • Cache-Control: max-age=3600                                 │
│   • ETag: "abc123"                                              │
│   • Last-Modified: Wed, 21 Oct 2024 07:28:00 GMT               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Security Shield

```
┌─────────────────────────────────────────────────────────────────┐
│                 SECURITY BENEFITS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. HIDE BACKEND INFRASTRUCTURE                                 │
│   ──────────────────────────────                                 │
│   Attacker sees: proxy.company.com                              │
│   Attacker doesn't see: 10.0.1.5, 10.0.1.6, 10.0.1.7           │
│                                                                  │
│   2. DDoS PROTECTION                                             │
│   ──────────────────                                             │
│   Attack traffic → Proxy (absorbs/filters) → Clean traffic     │
│   Proxy can handle millions of connections                      │
│   Backend servers stay protected                                │
│                                                                  │
│   3. WEB APPLICATION FIREWALL (WAF)                             │
│   ─────────────────────────────────                              │
│   Proxy inspects requests for:                                   │
│   • SQL injection attempts                                       │
│   • XSS attacks                                                  │
│   • Malformed requests                                           │
│   Bad requests blocked before reaching servers                  │
│                                                                  │
│   4. RATE LIMITING                                               │
│   ────────────────                                               │
│   Proxy tracks requests per IP/user                             │
│   Blocks excessive requests (429 Too Many Requests)             │
│   Protects against brute force attacks                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 SIDE-BY-SIDE COMPARISON

```
┌─────────────────────────────────────────────────────────────────┐
│              FORWARD vs REVERSE PROXY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Aspect          │ Forward Proxy    │ Reverse Proxy            │
│   ────────────────┼──────────────────┼────────────────────────  │
│   Position        │ Client-side      │ Server-side              │
│   Acts for        │ Clients          │ Servers                  │
│   Hides           │ Client identity  │ Server identity          │
│   Client aware?   │ Yes              │ No                       │
│   Main purpose    │ Privacy/Access   │ Load/Security            │
│   Who configures? │ Client/Network   │ Server admin             │
│   Examples        │ VPN, Squid, Tor  │ Nginx, Cloudflare, ALB   │
│                                                                  │
│   ═══════════════════════════════════════════════════════════   │
│                                                                  │
│   MEMORY HOOK:                                                   │
│   ─────────────                                                  │
│   Forward = Client FORWARDING requests through proxy            │
│   Reverse = Requests REVERSED/redirected to hidden servers     │
│                                                                  │
│   OR SIMPLY:                                                     │
│   ───────────                                                    │
│   Forward Proxy = "Hide ME from the server"                     │
│   Reverse Proxy = "Hide SERVERS from me"                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ NGINX CONFIGURATION EXAMPLES

### Reverse Proxy Configuration

```nginx
# Basic reverse proxy
http {
    upstream backend_servers {
        # Load balancing with health checks
        server 10.0.1.10:8080 weight=5;
        server 10.0.1.11:8080 weight=3;
        server 10.0.1.12:8080 backup;  # Only used if others fail
    }

    server {
        listen 80;
        listen 443 ssl;
        server_name api.example.com;

        # SSL configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        location / {
            # Forward to backend
            proxy_pass http://backend_servers;
            
            # Preserve original client info
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # API routing
        location /api/users {
            proxy_pass http://user-service:8080;
        }

        location /api/orders {
            proxy_pass http://order-service:8080;
        }

        # Static file caching
        location /static/ {
            proxy_pass http://backend_servers;
            proxy_cache_valid 200 1d;
            proxy_cache_use_stale error timeout;
        }
    }
}
```

### Forward Proxy Configuration

```nginx
# Forward proxy (requires ngx_http_proxy_connect_module)
server {
    listen 8080;
    
    # DNS resolver
    resolver 8.8.8.8;
    
    location / {
        proxy_pass http://$http_host$request_uri;
        proxy_set_header Host $http_host;
        
        # Access control
        allow 192.168.1.0/24;  # Only internal network
        deny all;
    }
}
```

---

## 🔧 CODE EXAMPLES

### Python - Detecting Proxy Usage

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/whoami')
def whoami():
    """
    Detect if request came through a proxy
    """
    # Direct client IP (might be proxy)
    remote_addr = request.remote_addr
    
    # Original client IP (set by reverse proxy)
    x_forwarded_for = request.headers.get('X-Forwarded-For')
    x_real_ip = request.headers.get('X-Real-IP')
    
    # Protocol (http or https)
    x_forwarded_proto = request.headers.get('X-Forwarded-Proto', 'http')
    
    if x_forwarded_for:
        # X-Forwarded-For can contain multiple IPs: client, proxy1, proxy2
        client_ip = x_forwarded_for.split(',')[0].strip()
    elif x_real_ip:
        client_ip = x_real_ip
    else:
        client_ip = remote_addr
    
    return {
        'remote_addr': remote_addr,
        'client_ip': client_ip,
        'x_forwarded_for': x_forwarded_for,
        'x_real_ip': x_real_ip,
        'protocol': x_forwarded_proto,
        'behind_proxy': x_forwarded_for is not None or x_real_ip is not None
    }
```

### Java - Spring Boot Behind Reverse Proxy

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.web.filter.ForwardedHeaderFilter;

@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    
    /**
     * Enable forwarded header processing.
     * This allows Spring to correctly identify:
     * - Client IP from X-Forwarded-For
     * - Protocol from X-Forwarded-Proto
     * - Host from X-Forwarded-Host
     */
    @Bean
    public FilterRegistrationBean<ForwardedHeaderFilter> forwardedHeaderFilter() {
        FilterRegistrationBean<ForwardedHeaderFilter> bean = 
            new FilterRegistrationBean<>();
        bean.setFilter(new ForwardedHeaderFilter());
        return bean;
    }
}
```

```yaml
# application.yml - Trust proxy headers
server:
  forward-headers-strategy: framework
  tomcat:
    remote-ip-header: X-Forwarded-For
    protocol-header: X-Forwarded-Proto
```

### Node.js - Express Behind Proxy

```javascript
const express = require('express');
const app = express();

// Trust the reverse proxy (important for req.ip and req.protocol)
app.set('trust proxy', true);
// Or be specific: app.set('trust proxy', 'loopback, linklocal, uniquelocal');

app.get('/whoami', (req, res) => {
    res.json({
        // With 'trust proxy', these reflect the real client
        clientIP: req.ip,
        protocol: req.protocol,
        host: req.hostname,
        
        // Raw headers for debugging
        headers: {
            'x-forwarded-for': req.headers['x-forwarded-for'],
            'x-forwarded-proto': req.headers['x-forwarded-proto'],
            'x-real-ip': req.headers['x-real-ip']
        }
    });
});

app.listen(3000);
```

---

## 🌐 REAL-WORLD ARCHITECTURE

### Netflix Architecture (Simplified)

```
┌─────────────────────────────────────────────────────────────────┐
│                 NETFLIX PROXY LAYERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User                                                           │
│     │                                                            │
│     ▼                                                            │
│   ┌─────────────────────────────────────────┐                   │
│   │           CDN (Open Connect)            │ ← Reverse Proxy   │
│   │   • Caches video content at edge        │   Layer 1         │
│   │   • 95% of traffic served from here     │                   │
│   └─────────────────────────────────────────┘                   │
│     │                                                            │
│     ▼ (only API calls, not video)                               │
│   ┌─────────────────────────────────────────┐                   │
│   │          AWS ELB / CloudFront           │ ← Reverse Proxy   │
│   │   • SSL termination                     │   Layer 2         │
│   │   • Geographic routing                  │                   │
│   └─────────────────────────────────────────┘                   │
│     │                                                            │
│     ▼                                                            │
│   ┌─────────────────────────────────────────┐                   │
│   │          Zuul API Gateway               │ ← Reverse Proxy   │
│   │   • Authentication                      │   Layer 3         │
│   │   • Rate limiting                       │                   │
│   │   • Request routing                     │                   │
│   └─────────────────────────────────────────┘                   │
│     │                                                            │
│     ▼                                                            │
│   ┌─────────────────────────────────────────┐                   │
│   │        Microservices (hundreds)         │                   │
│   │   • User service, Recommendation, etc.  │                   │
│   └─────────────────────────────────────────┘                   │
│                                                                  │
│   LAYERS OF REVERSE PROXIES:                                    │
│   1. CDN - Content caching at edge                              │
│   2. Load Balancer - Traffic distribution                       │
│   3. API Gateway - Security & routing                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Corporate Network with Both Proxies

```
┌─────────────────────────────────────────────────────────────────┐
│                 CORPORATE ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   INTERNAL NETWORK                           INTERNET            │
│   ────────────────                           ────────            │
│                                                                  │
│   ┌──────────────┐                                              │
│   │   Employee   │                                              │
│   │   Laptop     │                                              │
│   └──────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│   ┌──────────────┐     ┌──────────────┐                         │
│   │   Forward    │────▶│   Internet   │  Accessing external     │
│   │   Proxy      │     │   (Google,   │  websites through       │
│   │  (Squid)     │     │   etc.)      │  forward proxy          │
│   └──────────────┘     └──────────────┘                         │
│                                                                  │
│   ════════════════════════════════════════════════════════════  │
│                                                                  │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│   │   Internet   │────▶│   Reverse    │────▶│   Internal   │   │
│   │   User       │     │   Proxy      │     │   App        │   │
│   │              │     │  (Nginx)     │     │   Servers    │   │
│   └──────────────┘     └──────────────┘     └──────────────┘   │
│                                                                  │
│   External users access internal apps through reverse proxy     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Practice

### Question 1:
> "What's the difference between a forward proxy and a reverse proxy?"

**Answer:**
> "The key difference is who they represent. A **forward proxy** acts on behalf of **clients** — it sits between clients and the internet, hiding the client's identity from servers. Clients configure their browser/app to use it. Examples include corporate proxies and VPNs.
>
> A **reverse proxy** acts on behalf of **servers** — it sits between the internet and backend servers, hiding the server infrastructure from clients. Clients don't know it exists. Examples include Nginx, Cloudflare, and load balancers.
>
> Simple memory hook: Forward proxy hides 'me' (the client), reverse proxy hides 'them' (the servers)."

### Question 2:
> "Why would you use a reverse proxy instead of exposing your servers directly?"

**Answer:**
> "Multiple critical reasons:
> 1. **Load Balancing** — Distribute traffic across multiple backend servers
> 2. **SSL Termination** — Handle HTTPS at one place, reducing certificate management and server CPU
> 3. **Security** — Hide backend infrastructure, add WAF, rate limiting, DDoS protection
> 4. **Caching** — Cache responses to reduce backend load
> 5. **Single Entry Point** — One public IP instead of exposing all servers
> 6. **Zero-Downtime Deployments** — Route traffic away from servers being updated
>
> Companies like Netflix use multiple layers of reverse proxies: CDN for caching, load balancers for distribution, and API gateways for authentication and routing."

### Question 3:
> "How do you preserve the real client IP when using a reverse proxy?"

**Answer:**
> "Reverse proxies forward the original client IP using HTTP headers:
> - `X-Forwarded-For`: Contains the client IP (and chain of proxy IPs)
> - `X-Real-IP`: Contains just the client IP
> - `X-Forwarded-Proto`: Original protocol (http/https)
>
> The proxy adds these headers, and the backend application must be configured to trust them. For example, in Nginx: `proxy_set_header X-Real-IP $remote_addr`. In the application, you read from these headers instead of the direct connection IP.
>
> Security note: Only trust these headers from known proxy IPs, otherwise clients could spoof them."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 8 | Load Balancing | Reverse proxies implement load balancing |
| Day 13 | Circuit Breaker | Proxies can implement circuit breaker patterns |
| Day 22 | 7-Layer Architecture | Proxies sit at the gateway/load balancer layer |
| Day 28 | Consistent Hashing | Load balancers use consistent hashing for sticky sessions |

---

## ✅ Day 29 Action Items

1. **Set up Nginx** as a reverse proxy for a local application
2. **Implement load balancing** with multiple backend servers
3. **Configure SSL termination** at the proxy level
4. **Add caching** for static content
5. **Test X-Forwarded headers** to ensure client IP preservation

---

## 💡 Key Takeaways

| Takeaway | Why It Matters |
|----------|----------------|
| Forward = client's representative | Hides client, controls access |
| Reverse = server's representative | Hides servers, enables scaling |
| Reverse proxy is essential at scale | Load balancing, security, caching |
| Always preserve client IP | Use X-Forwarded-For headers |
| Layer your proxies | CDN → LB → API Gateway → Services |

---

## 🚀 System Design Interview Tip

> **When discussing reverse proxies in interviews, mention the layers:**
>
> "In production, we typically have multiple reverse proxy layers. The first layer is a CDN like Cloudflare for caching and DDoS protection. The second layer is a load balancer like AWS ALB for traffic distribution. The third layer might be an API Gateway like Kong for authentication, rate limiting, and request transformation. Each layer adds value without the client knowing the complexity behind that single domain name."

---

## 💡 Key Takeaway

> **Junior: "I'll just expose my application servers directly to the internet."**
>
> **Architect: "Never. We'll put a reverse proxy in front — Nginx for load balancing and SSL termination, with Cloudflare for CDN and DDoS protection. Clients see one domain, but behind it we have auto-scaling servers, zero-downtime deployments, and security at every layer. The proxy is our single point of control."**

The difference? Understanding that **proxies are not overhead — they're architecture**. In system design, the reverse proxy layer is often where the most critical cross-cutting concerns live.

---

*— Sunchit Dudeja*  
*Day 29 of 50: System Design Interview Preparation Series*

---

> 💡 **Tomorrow (Day 30):** We'll explore **Rate Limiting** — how do companies like Stripe, GitHub, and Netflix protect their APIs from abuse while ensuring fair access? Token bucket, sliding window, and distributed rate limiting at scale.
