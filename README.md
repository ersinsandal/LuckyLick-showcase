# LuckyLick

> **⚠️ This repository is public for documentation purposes only.**  
> The full source code is maintained in a private repository.  
> Contact: [SliferSoft](mailto:info@slifersoft.com)

🇹🇷 [Türkçe dokümantasyon için tıklayın → README_TR.md](./README_TR.md)

---

## What is LuckyLick?

**LuckyLick** is an experimental web platform that demonstrates **ISP-level censorship bypass without a VPN**, using a local multi-threaded reverse proxy server combined with DNS-over-HTTPS (DoH) resolution.

The project proves a single core thesis:

> *It is possible to access ISP-blocked content from a standard browser — with zero VPN, zero browser extensions, and zero special configuration — purely through server-side DNS tunneling and HTTPS proxy routing.*

### On the Name

The name **LuckyLick** is a deliberate double entendre:

- **Lucky** + **Click** → *LuckyLick* — the core UX mechanic is a single click that yields an unpredictable result, like a roll of the dice
- **Lick** carries a secondary erotic connotation — a nod to the platform's adult content test subject, woven into the brand without being explicit

The name was chosen to be memorable, slightly irreverent, and accurate to both the technical mechanic and the content domain.

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

## Content Choice Rationale

For an effective censorship bypass test, the target content must satisfy two conditions:

1. **Reliably blocked by ISPs** in the test country (Turkey)
2. **Rich in sub-resources** — scripts, CDN media, HLS playlists — so that the bypass is genuinely non-trivial

Adult video platforms were chosen as the test target because they meet both conditions perfectly. They are among the most consistently and thoroughly blocked sites in Turkey, operating under blanket regulatory bans rather than case-by-case takedowns. Their technical infrastructure is also particularly demanding: video delivery involves multiple independently-hosted CDN subdomains, signed token URLs with short expiry windows, player authentication handshakes, and adaptive bitrate (HLS/DASH) manifests — making a successful end-to-end bypass technically meaningful and non-trivial to replicate.

The adult content subject matter is therefore incidental to the research goal: it was selected because it is the hardest censorship target to bypass correctly, not because of the content itself.

The randomized "roulette" UX element (inspired by [theuselessweb.com](https://theuselessweb.com/)) was added to give the experiment a playful, explorable character — one click, an unknown outcome. The luck/chance mechanic served as both a creative framing device and a lightweight pool-sampling mechanism for iterating over a large content dataset.

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

| Status | Finding |
|--------|---------|
| ✅ | Full bypass achievable with pure Python + stdlib |
| ✅ | Native HTML5 video playback (no iframe, no ads) |
| ✅ | Zero client-side configuration required |
| ✅ | Works across Chrome, Firefox, Edge, Safari |
| ⚠️ | CDN token expiry means stream URLs are time-limited (~2h validity) |
| ⚠️ | Some CDN nodes are geofenced at TCP level (unreachable even by IP) |

---

## Repository

The full source code (proxy server, harvester scripts, SPA frontend, data pools) is maintained in a **private repository** to prevent abuse. This README is provided for technical documentation and portfolio reference only.

**Developer:** [SliferSoft](https://slifersoft.com) — Ersin Sandal  
**Contact:** info@slifersoft.com
