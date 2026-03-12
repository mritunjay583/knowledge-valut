---
title: "WebRTC"
layout: default
parent: Protocols
nav_order: 4
---

# WebRTC

> A comprehensive guide to WebRTC — why it exists, what problems it solves, how it compares to HTTP, gRPC, and WebSockets, architecture internals, and real-world production use cases. Written for system design interviews and engineering decisions.

---

## What is WebRTC?

WebRTC (Web Real-Time Communication) is a **peer-to-peer, real-time communication protocol** that enables audio, video, and arbitrary data to flow directly between browsers (or native apps) without going through a server.

- Open-source project by Google (originally acquired from Global IP Solutions)
- **Browser-native** — no plugins, no installs (`navigator.mediaDevices`, `RTCPeerConnection`)
- Primarily **peer-to-peer** (P2P), but can use servers when needed
- Built on **UDP** for low-latency media transport
- Handles the hard stuff: codec negotiation, encryption, NAT traversal, adaptive bitrate
- Standardized by W3C (API) and IETF (protocols)

---

## Why WebRTC? What Problem Does It Solve?

### The Problem Before WebRTC

Before WebRTC, real-time audio/video in browsers required:
- **Flash** — dead, security nightmare, not mobile-friendly
- **Java Applets** — dead, slow startup, security issues
- **Plugins** (Oovoo, Skype plugin) — install friction, cross-platform pain
- **Server relay for everything** — all media routed through servers = high latency + massive bandwidth cost

```
Old way (server relay):
  User A → [Upload video to Server] → [Server relays] → [Download to User B]
  Latency: 200-500ms+
  Server bandwidth: 2x every stream
  Cost: $$$$$

WebRTC way (peer-to-peer):
  User A ←→ User B (direct connection)
  Latency: 50-150ms
  Server bandwidth: ~0 (just signaling)
  Cost: $
```

### What WebRTC Solves

| Problem | Before WebRTC | With WebRTC |
|---------|---------------|-------------|
| Browser video calls | Needed plugins/Flash | Native, zero install |
| Latency | 200-500ms (server relay) | 50-150ms (P2P, UDP) |
| Server bandwidth cost | Server relays all media | P2P — media goes direct |
| Encryption | Optional, inconsistent | Mandatory (DTLS + SRTP) |
| NAT traversal | DIY, extremely hard | Built-in (ICE, STUN, TURN) |
| Codec negotiation | Manual | Automatic (SDP offer/answer) |
| Adaptive quality | Manual | Built-in (bandwidth estimation, adaptive bitrate) |

### Interview Tip 💡
> WebRTC isn't just "video calling in the browser." It's a full stack of protocols for real-time media AND data. The P2P nature means servers don't touch the media in most cases — that's the key cost and latency advantage.

---

## WebRTC Architecture — The Protocol Stack

WebRTC isn't a single protocol. It's a stack of protocols working together:

```
┌─────────────────────────────────────────────┐
│              Application (JS API)            │
│  getUserMedia, RTCPeerConnection, DataChannel│
├─────────────────────────────────────────────┤
│              SRTP / SRTCP                    │  ← Encrypted media
│         (Secure Real-time Transport)         │
├─────────────────────────────────────────────┤
│              DTLS                            │  ← Key exchange & encryption
│    (Datagram Transport Layer Security)       │
├─────────────────────────────────────────────┤
│              SCTP (for DataChannel)          │  ← Reliable/unreliable data
│    (Stream Control Transmission Protocol)    │
├─────────────────────────────────────────────┤
│              ICE / STUN / TURN               │  ← NAT traversal
│    (Interactive Connectivity Establishment)   │
├─────────────────────────────────────────────┤
│              UDP (primary) / TCP (fallback)  │  ← Transport
└─────────────────────────────────────────────┘
```

### Core Components Explained

**1. Signaling (NOT part of WebRTC spec)**

WebRTC doesn't define how peers find each other. You need a signaling server to exchange:
- **SDP (Session Description Protocol)** — what codecs, media types, and network info each peer supports
- **ICE Candidates** — possible network paths to reach each peer

```
┌────────┐     Signaling Server      ┌────────┐
│ Peer A │ ──── (WebSocket/HTTP) ───→│        │
│        │                           │ Server │
│        │ ←── (WebSocket/HTTP) ────│        │
└────┬───┘                           └───┬────┘
     │                                   │
     │         Signaling Server          │
     │    ┌────────────────────────┐     │
     │    │ 1. A sends SDP Offer   │     │
     │    │ 2. Server forwards to B│     │
     │    │ 3. B sends SDP Answer  │     │
     │    │ 4. Server forwards to A│     │
     │    │ 5. Exchange ICE cands  │     │
     │    └────────────────────────┘     │
     │                                   │
     │      Direct P2P Connection        │
     │ ═══════════════════════════════   │
     │    (media flows directly,         │
     │     server is no longer needed)   │
     └───────────────────────────────────┘
```

**2. SDP (Session Description Protocol) — The Offer/Answer**

```
// SDP Offer (simplified)
v=0
o=- 123456 2 IN IP4 127.0.0.1
s=-
t=0 0
m=audio 49170 RTP/SAVPF 111 103
a=rtpmap:111 opus/48000/2        ← I support Opus audio codec
a=rtpmap:103 ISAC/16000          ← I also support iSAC
m=video 51372 RTP/SAVPF 96 97
a=rtpmap:96 VP8/90000            ← I support VP8 video codec
a=rtpmap:97 H264/90000           ← I also support H.264
a=ice-ufrag:abc123               ← My ICE credentials
a=fingerprint:sha-256 AB:CD:...  ← My DTLS certificate fingerprint
```

The offer says "here's what I support." The answer says "here's what I picked from your list."

**3. ICE (Interactive Connectivity Establishment) — NAT Traversal**

The hardest problem WebRTC solves. Most devices are behind NATs/firewalls and can't receive direct connections.

```
ICE Candidate Types (in preference order):

1. Host Candidate — direct local IP (works on same network)
   192.168.1.5:12345

2. Server Reflexive (srflx) — public IP discovered via STUN
   203.0.113.5:54321 (my public IP, discovered by STUN server)

3. Relay Candidate — traffic relayed through TURN server
   turn-server.com:3478 (last resort, adds latency)
```

**STUN (Session Traversal Utilities for NAT):**
```
Peer A → STUN Server: "What's my public IP?"
STUN Server → Peer A: "You're 203.0.113.5:54321"
Peer A shares this with Peer B via signaling
Peer B tries to connect directly to 203.0.113.5:54321
```
- Works for ~80% of NAT types (cone NATs)
- Fails for symmetric NATs (common in corporate networks)
- STUN servers are lightweight — just echo back the public IP

**TURN (Traversal Using Relays around NAT):**
```
Peer A → TURN Server → Peer B
```
- Fallback when direct connection fails (~20% of cases)
- TURN server relays ALL media — expensive in bandwidth
- This is the most expensive part of WebRTC infrastructure

**ICE Flow:**
```
1. Gather all candidates (host, srflx via STUN, relay via TURN)
2. Exchange candidates via signaling
3. Try all candidate pairs simultaneously (connectivity checks)
4. Pick the best working pair (prefer direct > STUN > TURN)
5. If network changes, ICE restarts and finds new path
```

### Interview Tip 💡
> STUN is cheap (just IP discovery, pennies/month). TURN is expensive (relays all media, bandwidth costs). A well-designed system minimizes TURN usage. In practice, ~80% of connections work via STUN, ~20% need TURN. Budget your TURN servers accordingly.

---

## The WebRTC Connection Flow (Complete)

```
Step 1: Get Media
  Peer A → navigator.mediaDevices.getUserMedia({video: true, audio: true})
  
Step 2: Create Peer Connection
  Peer A → new RTCPeerConnection(iceServers: [{urls: "stun:stun.example.com"}])
  
Step 3: Add Tracks
  Peer A → peerConnection.addTrack(audioTrack, stream)
  Peer A → peerConnection.addTrack(videoTrack, stream)
  
Step 4: Create Offer
  Peer A → peerConnection.createOffer() → SDP Offer
  Peer A → peerConnection.setLocalDescription(offer)
  
Step 5: Send Offer via Signaling
  Peer A → SignalingServer → Peer B: SDP Offer
  
Step 6: Peer B Receives Offer
  Peer B → peerConnection.setRemoteDescription(offer)
  Peer B → peerConnection.createAnswer() → SDP Answer
  Peer B → peerConnection.setLocalDescription(answer)
  
Step 7: Send Answer via Signaling
  Peer B → SignalingServer → Peer A: SDP Answer
  
Step 8: Exchange ICE Candidates (trickle ICE)
  Peer A → onicecandidate → SignalingServer → Peer B → addIceCandidate()
  Peer B → onicecandidate → SignalingServer → Peer A → addIceCandidate()
  
Step 9: ICE Connectivity Checks
  Both peers try candidate pairs until one works
  
Step 10: DTLS Handshake (encryption)
  Peers exchange keys, verify certificate fingerprints from SDP
  
Step 11: Media Flows (SRTP)
  Peer A ←══ Encrypted Audio/Video ══→ Peer B
```

---

## Media in WebRTC

### Audio Codecs
| Codec | Bitrate | Latency | Quality | Notes |
|-------|---------|---------|---------|-------|
| Opus | 6-510 kbps | ~20ms | Excellent | Default, adaptive, mandatory in WebRTC |
| G.711 | 64 kbps | ~1ms | Good | Legacy telephony, no compression |
| G.722 | 48-64 kbps | ~40ms | Good | Wideband, legacy |

### Video Codecs
| Codec | Notes |
|-------|-------|
| VP8 | Mandatory in WebRTC, open-source (Google), good quality |
| VP9 | Better compression than VP8, higher CPU |
| H.264 | Hardware acceleration on most devices, patent-encumbered |
| AV1 | Next-gen, best compression, high CPU (hardware support growing) |

### Adaptive Bitrate
WebRTC automatically adjusts quality based on network conditions:
```
Good network (5 Mbps) → 720p, 30fps, high bitrate
Network degrades (1 Mbps) → 480p, 24fps, lower bitrate
Network recovers (3 Mbps) → 720p, 30fps, medium bitrate
```
This uses **REMB (Receiver Estimated Maximum Bitrate)** or **Transport-CC** feedback.

### Simulcast
Send multiple quality layers simultaneously:
```
Peer A sends:
  Layer 1: 720p (high)
  Layer 2: 360p (medium)
  Layer 3: 180p (low)

SFU picks which layer to forward to each receiver based on their bandwidth
```

---

## DataChannel — The Underrated Feature

WebRTC isn't just audio/video. `RTCDataChannel` provides a P2P data pipe.

```javascript
const dataChannel = peerConnection.createDataChannel("game-state", {
  ordered: false,        // Don't guarantee order (faster for gaming)
  maxRetransmits: 0,     // Don't retry lost packets (unreliable mode)
});

dataChannel.onopen = () => {
  dataChannel.send("Hello peer!");
  dataChannel.send(binaryArrayBuffer);  // Binary data too
};
```

### DataChannel vs WebSocket

| Aspect | WebSocket | DataChannel |
|--------|-----------|-------------|
| Topology | Client ↔ Server | Peer ↔ Peer (P2P) |
| Transport | TCP | SCTP over DTLS over UDP |
| Reliability | Always reliable (TCP) | Configurable (reliable or unreliable) |
| Ordering | Always ordered | Configurable |
| Latency | Server hop adds latency | Direct P2P, minimal latency |
| Encryption | Optional (wss://) | Always encrypted (DTLS) |
| Server cost | Server handles all traffic | Zero (P2P) |
| NAT traversal | Not needed (server has public IP) | Needed (ICE/STUN/TURN) |

### Use Cases for DataChannel
- **P2P file sharing** — send files directly between browsers (no server upload)
- **P2P gaming** — game state sync with unreliable mode for speed
- **P2P chat** — messages without a server (privacy-focused)
- **Screen sharing data** — mouse/keyboard events alongside video

### Interview Tip 💡
> DataChannel's killer feature is **configurable reliability**. You can choose ordered+reliable (like TCP), unordered+unreliable (like UDP), or anything in between. For gaming, you want unreliable — a dropped position update is irrelevant because the next one is already coming.

---

## Scaling WebRTC — P2P Doesn't Scale

### The Mesh Problem
```
2 participants: 1 connection each (total: 1)
3 participants: 2 connections each (total: 3)
4 participants: 3 connections each (total: 6)
5 participants: 4 connections each (total: 10)
N participants: N-1 connections each (total: N*(N-1)/2)
```

Each participant must **encode and upload** their stream N-1 times. At 5+ participants, this destroys CPU and bandwidth.

```
Mesh (P2P) — works for 2-4 people:
  A ←→ B
  A ←→ C
  B ←→ C
  (everyone sends to everyone)
```

### Solution 1: SFU (Selective Forwarding Unit) — The Industry Standard

```
┌────────┐         ┌─────────┐         ┌────────┐
│ Peer A │ ──1x──→ │   SFU   │ ──────→ │ Peer B │
│        │ ←────── │ Server  │ ←─1x─── │        │
└────────┘         │         │         └────────┘
                   │ Forwards│
┌────────┐         │ streams │         ┌────────┐
│ Peer C │ ──1x──→ │ selectively      │ Peer D │
│        │ ←────── │         │ ←─1x─── │        │
└────────┘         └─────────┘         └────────┘
```

- Each participant uploads their stream **once** to the SFU
- SFU **forwards** (not transcode) streams to other participants
- SFU decides which streams/quality layers to forward (using simulcast)
- **CPU cost:** Low (just forwarding packets, no encoding/decoding)
- **Bandwidth cost:** Moderate (server handles distribution)
- **Scales to:** 50-100+ participants easily

**This is what Zoom, Google Meet, Microsoft Teams, and Discord use.**

### Solution 2: MCU (Multipoint Control Unit) — Legacy Approach

```
┌────────┐         ┌─────────┐         ┌────────┐
│ Peer A │ ──1x──→ │   MCU   │ ──1x──→ │ Peer B │
│        │ ←─1x─── │ Server  │ ←─1x─── │        │
└────────┘         │         │         └────────┘
                   │ Decodes │
                   │ Mixes   │
                   │ Encodes │
                   └─────────┘
```

- MCU **decodes all streams**, mixes them into one composite, **re-encodes**, sends one stream to each participant
- Each participant receives a single mixed stream (low download bandwidth)
- **CPU cost:** Extremely high (decode + mix + encode for every participant)
- **Bandwidth cost:** Low for clients
- **Scales to:** Limited by server CPU
- Used in legacy telephony systems, rarely in modern WebRTC

### SFU vs MCU

| Aspect | SFU | MCU |
|--------|-----|-----|
| Server CPU | Low (forwarding) | Very High (transcoding) |
| Client bandwidth (down) | Higher (N-1 streams) | Low (1 mixed stream) |
| Client CPU | Higher (decode N-1) | Low (decode 1) |
| Latency | Lower | Higher (mixing adds delay) |
| Flexibility | Each client picks layout | Server controls layout |
| Cost | Moderate | Expensive |
| Used by | Zoom, Meet, Teams | Legacy conferencing |

### Interview Tip 💡
> When asked "how would you design Zoom," the answer is SFU + simulcast. Each sender uploads 3 quality layers (high/med/low). The SFU forwards the appropriate layer to each receiver based on their bandwidth and screen size. The speaker gets high quality, thumbnails get low quality.

---

## WebRTC vs HTTP vs gRPC vs WebSocket — Four-Way Comparison

| Aspect | HTTP | gRPC | WebSocket | WebRTC |
|--------|------|------|-----------|--------|
| Transport | TCP | TCP (HTTP/2) | TCP | UDP (primary) |
| Topology | Client → Server | Client → Server | Client ↔ Server | Peer ↔ Peer (or via SFU) |
| Latency | Medium-High | Low-Medium | Low | Ultra-Low (~50ms) |
| Direction | Request-Response | 4 patterns | Full-duplex | Full-duplex |
| Data type | Any | Protobuf | Any | Audio/Video/Data |
| Encryption | Optional (TLS) | Optional (TLS) | Optional (WSS) | Mandatory (DTLS+SRTP) |
| Browser native | ✅ | ❌ | ✅ | ✅ |
| P2P capable | ❌ | ❌ | ❌ | ✅ |
| NAT traversal | N/A | N/A | N/A | Built-in (ICE) |
| Server cost | Per request | Per RPC | Per connection | Minimal (P2P) or SFU |
| Best for | APIs, web pages | Microservices | Real-time messaging | Audio/video, P2P data |

### Where WebRTC Wins Over Everything Else

| Scenario | Why WebRTC Wins |
|----------|-----------------|
| Video/audio calls | Built for this — codecs, adaptive bitrate, echo cancellation |
| P2P file transfer | No server bandwidth cost, direct transfer |
| Ultra-low latency needs | UDP-based, ~50ms vs ~200ms+ for TCP-based |
| Privacy-sensitive P2P | Data never touches a server (in true P2P mode) |
| Screen sharing | Native browser API, optimized encoding |

### Where WebRTC Loses

| Scenario | Better Alternative | Why |
|----------|-------------------|-----|
| Chat/messaging | WebSocket | Simpler, no NAT traversal complexity |
| APIs | HTTP/REST | Request-response is the right model |
| Microservices | gRPC | Type safety, deadlines, no P2P needed |
| Server push (notifications) | SSE / WebSocket | WebRTC is overkill |
| Broadcasting to 10K+ viewers | HLS/DASH (HTTP streaming) | WebRTC doesn't scale for broadcast |
| Reliable ordered data | WebSocket / TCP | WebRTC's UDP base can lose/reorder packets |

---

## Real-World Production Use Cases

### 1. Google Meet
- **SFU architecture** with simulcast
- Each participant sends 3 quality layers
- SFU forwards appropriate layer based on receiver's bandwidth and UI layout
- Uses VP8/VP9 codecs, Opus for audio
- **Scale:** Supports up to 500 participants (most receiving only)

### 2. Zoom
- Hybrid **SFU + selective MCU** approach
- SFU for most cases, MCU-like mixing for gallery view on weak clients
- Custom congestion control algorithms
- End-to-end encryption option (media encrypted before reaching SFU)
- **Scale:** Up to 1000 participants

### 3. Discord
- WebRTC for **voice channels** and video
- WebSocket for signaling and text chat
- SFU (Selective Forwarding) for group calls
- Custom Opus settings optimized for gaming voice chat (low latency > quality)
- **Scale:** Voice channels with 25+ simultaneous speakers

### 4. Twitch / Facebook Live (Low-Latency Ingest)
- Streamers use WebRTC to **ingest** their stream to the server (~sub-second latency)
- Server then transcodes and distributes via HLS/DASH to viewers (5-30s latency)
- **Why WebRTC for ingest:** Ultra-low latency from streamer to server
- **Why HLS for distribution:** Scales to millions of viewers via CDN

### 5. Figma (Cursor/Voice in Collaboration)
- WebRTC for **voice chat** during collaboration sessions
- DataChannel for low-latency cursor position sharing
- **Why WebRTC:** P2P voice is cheaper than server-relayed, cursor sync needs ultra-low latency

### 6. Clubhouse / Twitter Spaces
- WebRTC for **real-time audio rooms**
- SFU architecture — speakers upload, listeners receive
- Agora (third-party WebRTC infrastructure) powers many of these
- **Scale:** Thousands of listeners, dozens of speakers

### 7. CloudFlare / Google Stadia (Cloud Gaming)
- WebRTC for **streaming game video** from cloud to player
- DataChannel for **controller input** (unreliable mode — speed over reliability)
- **Why WebRTC:** Sub-100ms round trip is critical for playable gaming
- H.264 hardware encoding on server, hardware decoding on client

### 8. Miro / Excalidraw (Collaborative Whiteboards)
- DataChannel for **real-time drawing sync** between collaborators
- P2P when possible, SFU for larger rooms
- **Why WebRTC DataChannel:** Lower latency than WebSocket for drawing strokes

### Production Architecture — Video Conferencing at Scale
```
                    ┌──────────────────┐
  Browsers ────────→│ Signaling Server │ (WebSocket)
  (WebRTC)          │ (room mgmt, SDP) │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
     │   SFU Node 1   │ │   SFU Node 2   │ │   SFU Node 3   │
     │ (media routing) │ │ (media routing) │ │ (media routing) │
     └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
             │                  │                  │
             └──────── Cascading (SFU-to-SFU) ─────┘
                    (for cross-region rooms)
     
     ┌─────────┐  ┌─────────┐
     │  STUN   │  │  TURN   │  (NAT traversal infrastructure)
     │ Servers │  │ Servers │
     └─────────┘  └─────────┘
     
     ┌──────────────────┐
     │  Recording/      │  (optional: record streams for playback)
     │  Transcoding     │
     └──────────────────┘
```

---

## Common Interview Questions

### Q: Design a video conferencing system like Zoom/Google Meet
1. **Signaling:** WebSocket server for room management, SDP exchange, ICE candidates
2. **Media:** SFU architecture — each participant sends 1 stream (with simulcast layers), SFU forwards selectively
3. **NAT traversal:** STUN servers (cheap, handle 80% of cases) + TURN servers (expensive fallback)
4. **Scaling rooms:** Each SFU handles ~50-100 participants. For larger rooms, cascade SFUs
5. **Global distribution:** Deploy SFUs in multiple regions, cascade between them for cross-region rooms
6. **Adaptive quality:** Simulcast (3 layers) + SFU selects layer per receiver based on bandwidth/UI
7. **Recording:** Tap into SFU, forward streams to a recording service
8. **Screen sharing:** Separate media track, higher resolution, lower framerate

### Q: How does NAT traversal work in WebRTC?
1. **Gather candidates:** Host (local IP), Server Reflexive (STUN → public IP), Relay (TURN)
2. **Exchange via signaling:** Both peers share all their candidates
3. **Connectivity checks:** ICE tries all candidate pairs simultaneously (STUN binding requests)
4. **Select best pair:** Prefer host > srflx > relay (lowest latency wins)
5. **Consent freshness:** Periodically verify the path still works
6. **ICE restart:** If network changes (WiFi → cellular), restart ICE to find new path

### Q: WebRTC vs WebSocket for a chat application?
- **WebSocket wins.** Chat is text messages that need reliable, ordered delivery. WebSocket (TCP) gives you that natively.
- WebRTC DataChannel CAN do chat, but adds unnecessary complexity (ICE, STUN/TURN, P2P connection setup) for something a simple WebSocket server handles trivially.
- **Exception:** If you need P2P privacy (no server sees messages), DataChannel makes sense.

### Q: How would you build a live streaming platform (like Twitch)?
- **Ingest:** Streamer → WebRTC (or RTMP) → Media Server (ultra-low latency capture)
- **Transcode:** Media Server → FFmpeg → Multiple quality levels (1080p, 720p, 480p, 360p)
- **Package:** HLS/DASH segments (2-6 second chunks)
- **Distribute:** CDN (CloudFront, Fastly) → Viewers
- **Why NOT WebRTC for viewers?** WebRTC is P2P/SFU — doesn't scale to millions. HLS/DASH over HTTP scales infinitely via CDN caching.
- **Low-latency option:** WebRTC for <1000 viewers (e.g., Cloudflare Calls), HLS-LL for larger audiences

### Q: How do you handle packet loss in WebRTC?
- **Audio (Opus):** Forward Error Correction (FEC) — redundant data in each packet to recover lost ones
- **Video:** 
  - NACK — receiver requests retransmission of specific lost packets
  - FEC (like FlexFEC) — parity packets to reconstruct lost ones
  - PLI (Picture Loss Indication) — request a new keyframe if too many packets lost
- **Adaptive bitrate:** If loss is high, reduce quality to reduce congestion
- **Jitter buffer:** Buffer a few packets to smooth out arrival time variations

### Q: What's the difference between SRTP and DTLS?
- **DTLS (Datagram TLS):** TLS but for UDP. Handles the key exchange and establishes encryption parameters. Runs once at connection setup.
- **SRTP (Secure RTP):** Encrypts the actual media packets using keys from DTLS. Runs on every packet.
- **Analogy:** DTLS is the handshake that agrees on the lock. SRTP is the lock on every package.

### Q: How would you add recording to a WebRTC video call?
- **Server-side (standard):** SFU taps into the media streams, forwards copies to a recording service that writes to storage (S3)
- **Composite recording:** MCU-like service decodes all streams, mixes into one video, encodes, stores
- **Individual recording:** Store each participant's stream separately, compose later (cheaper, more flexible)
- **Client-side:** `MediaRecorder` API in the browser — simple but unreliable (user can close tab)

### Q: Explain Oonnection migration in WebRTC vs HTTP/3
- **HTTP/3 (QUIC):** Connection ID survives IP changes. Seamless, automatic.
- **WebRTC (ICE restart):** When network changes, ICE restarts — gathers new candidates, runs connectivity checks again. Brief interruption (~1-3 seconds) but recovers.
- HTTP/3 is smoother because QUIC was designed for this. WebRTC's ICE restart is a heavier process.

---

## WebRTC Infrastructure Costs

Understanding costs is critical for system design interviews:

| Component | Cost | Notes |
|-----------|------|-------|
| STUN server | ~Free | Lightweight, Google offers free ones (`stun:stun.l.google.com:19302`) |
| TURN server | $$$$ | Bandwidth-intensive — relays all media. ~$0.04-0.10/GB |
| SFU server | $$$ | CPU for packet routing, bandwidth for forwarding |
| Signaling server | $ | Just WebSocket, minimal resources |
| Recording | $$ | Storage + compute for transcoding |

**Cost optimization:**
- Minimize TURN usage (good ICE implementation, try all candidates)
- Use simulcast to reduce SFU bandwidth (forward low quality to thumbnails)
- Deploy SFUs close to users (reduce latency, avoid cross-region bandwidth)
- Use hardware codecs where possible (VP8/H.264 hardware encoding)

---

## Decision Framework

```
Do you need audio/video?
├── Yes → Is it 1-to-1 or small group (<5)?
│         ├── Yes → WebRTC P2P (mesh) ✅
│         └── No → Is it a meeting (<100 people)?
│                   ├── Yes → WebRTC + SFU ✅
│                   └── No → Is it a broadcast (1000+ viewers)?
│                             ├── Yes → WebRTC ingest + HLS/DASH distribution
│                             └── No → WebRTC + cascaded SFUs
├── No → Do you need P2P data transfer?
│         ├── Yes → WebRTC DataChannel ✅
│         └── No → Do you need real-time server push?
│                   ├── Yes → WebSocket or SSE
│                   └── No → HTTP/REST or gRPC
```

---

## Further Reading
- http — the protocol WebRTC's signaling often runs on
- websockets — commonly used for WebRTC signaling
- grpc — alternative for structured service-to-service streaming
- tcp — the transport WebSocket uses vs WebRTC's UDP
- dns — resolving STUN/TURN server addresses
- load-balancing — distributing users across SFU nodes

---

*Last updated: 2025-03-13*
