# LuckyLick

[![Türkçe Dokümantasyon](https://img.shields.io/badge/Dok%C3%BCmantasyon-T%C3%BCrk%C3%A7e%20%C4%B0%C3%A7in%20T%C4%B1klay%C4%B1n%C4%B1z-red?style=for-the-badge&logo=readme)](./README_TR.md)

> [🇹🇷 **Türkçe dokümantasyonu okumak için tıklayınız → README_TR.md**](./README_TR.md)

> **⚠️ This repository is public for documentation purposes only.**
> The full source code is maintained in a private repository.
> Contact: [SliferSoft](mailto:info@slifersoft.com)

---

## TL;DR

LuckyLick is an **experimental research project** investigating ISP-level censorship bypass without relying on a VPN. Modeled after the interactive experience of theuselessweb.com, clicking the action button triggers a chance-driven selection that retrieves a random adult video from curated data pools and upstream platform APIs. Adult video platforms were specifically chosen as the test subject because they face the strictest regulatory bans in Turkey and feature multi-layered CDN architectures, presenting the ultimate real-world benchmark for traffic tunneling algorithms. The project's core accomplishment is streaming this strictly blocked media directly in a standard browser without requiring any VPN, proxy extensions, or network configuration—achieved through local proxy routing and DNS-over-HTTPS (DoH) tunneling.

---

## What is LuckyLick?

**LuckyLick** is an experimental web platform that demonstrates **ISP-level censorship bypass without a VPN**, using a local multi-threaded reverse proxy server combined with DNS-over-HTTPS (DoH) resolution.

The project proves a single core thesis:

> *It is possible to access ISP-blocked content from a standard browser — with zero VPN, zero browser extensions, and zero special configuration — purely through server-side DNS tunneling and HTTPS proxy routing.*

---

## The Technical Problem

In Turkey (and many other countries), ISPs block access to certain websites by poisoning their DNS responses or by deep-packet inspection (DPI). The typical workaround is a VPN, which adds latency, requires third-party trust, and creates friction for end users.

LuckyLick explores an alternative: **a locally-running Python proxy server** that:

1. **Bypasses DNS sinkholes** via hardcoded IP maps and Cloudflare/Google DNS-over-HTTPS fallback
2. **Establishes direct HTTPS connections** to blocked hosts using their real IPs, correct TLS SNI (`_server_hostname`), and valid `Origin`/`Referer` headers
3. **Rewrites all embedded sub-resources** (scripts, images, media) to route through itself, so the browser never makes a direct request to any blocked domain
4. **Streams media natively** via HTTP 206 Partial Content with chunked streaming and full `Range` request support — no buffering, no re-encoding
5. **Serves a minimal SPA frontend** with a zero-configuration experience — open `http://localhost:5500` and everything works

---

## Architecture Overview

```
Browser (localhost:5500)
        │
        ▼
┌──────────────────────────────────────┐
│   ThreadingHTTPServer (Python)        │
│   port 5500                           │
│                                      │
│  ┌─────────────────────────────────┐ │
│  │  Static File Serving (SPA)      │ │
│  │  index.html / script.js /       │ │
│  │  styles.css / assets/           │ │
│  └─────────────────────────────────┘ │
│                                      │
│  ┌─────────────────────────────────┐ │
│  │  /api/stream_info               │ │
│  │  • Fetches embed page HTML      │ │
│  │  • Extracts direct MP4/HLS URL  │ │
│  │  • Proxies all source URLs      │ │
│  └─────────────────────────────────┘ │
│                                      │
│  ┌─────────────────────────────────┐ │
│  │  /api/proxy?url=<target>        │ │
│  │  • Transparent reverse proxy    │ │
│  │  • DoH-based DNS resolution     │ │
│  │  • SNI + correct headers        │ │
│  │  • 206 Partial / Range support  │ │
│  │  • HTML/JSON URL rewriting      │ │
│  └─────────────────────────────────┘ │
└──────────────────────────────────────┘
        │
        ▼
 Real IP (via DoH)  →  Blocked Host CDN
```

---

## Key Technical Innovations

### 1. DoH + Pre-seeded IP Map
Standard DNS queries to ISP resolvers are intercepted and sinkholed. The server bypasses this entirely:

```python
DNS_MAP = {
    "www.example-blocked.com": "93.x.x.x",  # hardcoded real IP
    ...
}

# Fallback: Cloudflare 1.1.1.1 / Google DoH over HTTPS
doh_url = f"https://1.1.1.1/dns-query?name={host}&type=A"
```

### 2. TLS SNI Injection
Python's `http.client.HTTPSConnection` doesn't pass SNI by default when connecting to an IP. This is fixed with:

```python
conn = http.client.HTTPSConnection(real_ip, 443, context=ssl_ctx)
conn._server_hostname = host  # forces correct TLS SNI
```

### 3. Platform-Aware Origin/Referer Spoofing
CDN servers validate the request origin. The proxy sets correct headers per platform:

```python
if "phncdn.com" in host:
    headers["Origin"]  = "https://www.pornhub.com"
    headers["Referer"] = "https://www.pornhub.com/"
```

### 4. Accessible CDN Detection
Multiple CDN mirrors exist per video. The server tests each CDN's TCP reachability (~0.8s timeout) before committing to it:

```python
def is_host_accessible(src_url):
    s = socket.socket(); s.settimeout(0.8)
    s.connect((real_ip, 443))  # non-blocking port test
```

### 5. Client-Side JS Interception Injection
When proxying HTML pages, a script is injected to intercept `fetch()`, `XMLHttpRequest`, and `HTMLMediaElement.src` assignments — redirecting all outbound URLs through the local proxy at runtime:

```javascript
window.fetch = function(inp, init) {
  inp = toProxy(inp); // rewrites URL before it leaves the browser
  return _fetch.call(this, inp, init);
};
```

### 6. HTTP 206 Partial Content Streaming
Video files are streamed in 64 KB chunks with full `Range` header passthrough, enabling instant playback start and smooth seeking without downloading the entire file:

```python
while True:
    chunk = res.read(65536)
    if not chunk: break
    self.wfile.write(chunk)
    self.wfile.flush()
```

---

## Content Choice and Chance Mechanic

The research required a test target that is reliably blocked by Turkish ISPs and technically demanding enough that an end-to-end bypass is non-trivial. Adult video platforms satisfy both criteria: they face blanket nationwide regulatory blocks in Turkey, and their multi-CDN architectures, signed expiry tokens, and adaptive streaming manifests present a rigorous benchmark for traffic tunneling.

The chance mechanic (inspired by [theuselessweb.com](https://theuselessweb.com/)) drives the discovery workflow: clicking the button queries curated pools and upstream platform APIs to select a random video, streaming it instantly without any VPN. The content category serves strictly as a high-friction censorship benchmark.

---

## Running the Server

**Requirements:** Python 3.10+, no external dependencies.

```bash
python server.py 5500
```

Open `http://localhost:5500` in any browser. No VPN, no extensions, no configuration.

---

## Project Status

This project was a **research experiment**, not a production service. The findings were:

- ✅ Full bypass achievable with pure Python + stdlib
- ✅ Native HTML5 video playback (no iframe, no ads)
- ✅ Zero client-side configuration required
- ✅ Works across Chrome, Firefox, Edge, Safari
- ⚠️ CDN token expiry means stream URLs are time-limited (~2h validity)
- ⚠️ Some CDN nodes are geofenced at TCP level (unreachable even by IP)

---

## Repository

The full source code (proxy server, harvester scripts, SPA frontend, data pools) is maintained in a **private repository** to prevent abuse. This README is provided for technical documentation and portfolio reference only.

**Developer:** [SliferSoft](https://slifersoft.com) — Ersin Sandal  
**Contact:** info@slifersoft.com
