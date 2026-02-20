# JamSync

**A Decentralized, Real-Time WebRTC Audio Synchronization Network for Chromium Browsers**

JamSync is a peer-to-peer audio broadcasting system built entirely on standard web technologies. It allows a single Host to capture the live audio output of any browser tab — Spotify, YouTube, Apple Music, SoundCloud, or any other web-based audio source — and stream it simultaneously to an unlimited number of remote Listeners with sub-second latency, without routing audio through any central media server.

Listeners do not install anything. They receive a QR code or a URL, open it on any smartphone or computer browser, and hear the Host's audio within seconds. The system is entirely self-contained: the Chrome Extension runs on the Host's machine, and the Listener Progressive Web App is served from a globally distributed CDN.

---

## Table of Contents

1. [How It Works — The Big Picture](#1-how-it-works--the-big-picture)
2. [Repository Structure](#2-repository-structure)
3. [System Architecture Deep Dive](#3-system-architecture-deep-dive)
   - 3.1 [The Chrome Extension — Multi-Process Model](#31-the-chrome-extension--multi-process-model)
   - 3.2 [The Offscreen Document Engine](#32-the-offscreen-document-engine)
   - 3.3 [The Content Script Observer](#33-the-content-script-observer)
   - 3.4 [The Background Service Worker](#34-the-background-service-worker)
   - 3.5 [The Popup UI](#35-the-popup-ui)
   - 3.6 [The PWA Listener App](#36-the-pwa-listener-app)
4. [Audio Transport Pipeline](#4-audio-transport-pipeline)
   - 4.1 [Tier 1 — Direct WebRTC MediaStream](#41-tier-1--direct-webrtc-mediastream)
   - 4.2 [Tier 2 — TURN Server Relay](#42-tier-2--turn-server-relay)
   - 4.3 [Tier 3 — AudioWorklet Data Channel PCM Fallback](#43-tier-3--audioworklet-data-channel-pcm-fallback)
5. [Network Topology](#5-network-topology)
6. [Session Lifecycle — Step by Step](#6-session-lifecycle--step-by-step)
7. [Bidirectional Control System](#7-bidirectional-control-system)
8. [Data Channel Message Protocol](#8-data-channel-message-protocol)
9. [NAT Traversal Strategy](#9-nat-traversal-strategy)
10. [Installation — Chrome Extension (Host)](#10-installation--chrome-extension-host)
11. [Installation — PWA Listener (Vercel)](#11-installation--pwa-listener-vercel)
12. [Local Development Setup](#12-local-development-setup)
13. [Configuration Reference](#13-configuration-reference)
14. [Known Issues and v1.1 Hotfix](#14-known-issues-and-v11-hotfix)
15. [Technical Constraints and Limitations](#15-technical-constraints-and-limitations)
16. [Glossary](#16-glossary)

---

## 1. How It Works — The Big Picture

When you open Spotify in Chrome and click "Start Broadcasting" in the JamSync extension, several things happen in rapid sequence. The extension spawns a hidden background document that captures the raw audio stream from your Spotify tab. That audio is fed into a WebRTC peer connection. At the same time, the extension registers a short, unique Room ID with a public signaling server and generates a QR code pointing to the JamSync web app with that Room ID embedded in the URL.

When a friend scans the QR code, their browser opens the JamSync Progressive Web App, connects to the same signaling server, and negotiates a direct peer-to-peer connection with your machine. Once that connection is established, the signaling server is no longer needed. Audio flows directly from your browser to theirs — no server in the middle — and they start hearing whatever you are playing.

If their network blocks direct connections (a corporate firewall, a carrier-grade NAT on a mobile data plan), the system automatically falls back to routing audio through a relay server. If even that fails, a third-tier fallback kicks in that encodes the raw audio data manually and sends it over WebRTC's data messaging channel.

While all of this is happening, a metadata channel is open between you and every Listener. When your track changes on Spotify, every Listener's screen updates to show the new song title within milliseconds. Any Listener can press Play, Pause, Skip, or Previous on their phone, and those commands travel back to your machine and physically click the corresponding button in your Spotify tab.

The entire system — audio capture, peer connection management, encoding fallbacks, metadata sync, and remote control — runs inside two JavaScript environments: your Chrome Extension and a static website hosted on Vercel.

---

## 2. Repository Structure

```
jamsync/
│
├── jam/
│   │
│   ├── extension/                  The Chrome Extension (Host-side)
│   │   ├── manifest.json           Extension manifest (Manifest V3)
│   │   ├── background.js           Service Worker — lifecycle orchestrator
│   │   ├── offscreen.html          Hidden DOM document shell
│   │   ├── offscreen.js            Audio engine and WebRTC connection manager
│   │   ├── audio-relay-processor.js  AudioWorklet for PCM Data Channel fallback
│   │   ├── controller.js           Content script — DOM observer and actuator
│   │   ├── popup.html              Extension popup interface shell
│   │   └── popup.js                Popup logic — broadcasting controls and QR
│   │
│   └── web/                        The PWA Listener (Vercel-hosted)
│       ├── package.json
│       ├── public/
│       │   ├── index.html          PWA shell
│       │   ├── app.js              All listener logic — WebRTC, audio, UI
│       │   └── manifest.json       PWA install manifest
│       └── ...
│
└── README.md
```

The two top-level subdirectories represent two completely separate applications that communicate with each other exclusively through WebRTC. The `extension/` directory is never served over the network — it is loaded directly into Chrome as an unpacked extension. The `web/` directory is a static site deployed to Vercel.

---

## 3. System Architecture Deep Dive

JamSync's architecture is shaped by a specific constraint: Chrome's Manifest V3 extension specification, which removed background scripts' access to all DOM APIs. This matters because the three APIs that JamSync depends on — `AudioContext`, `MediaStream`, and `chrome.tabCapture` — are all DOM-domain APIs. Under Manifest V3, the background script runs as a Service Worker, which has no DOM access whatsoever.

The architectural solution is the `chrome.offscreen` API: a hidden HTML page that the Service Worker spawns programmatically, which runs with full DOM access, full Web API access, and full WebRTC access, but with no user-visible rendering surface. This hidden document is the heart of JamSync.

### 3.1 The Chrome Extension — Multi-Process Model

The Chrome Extension is not a single program. It is four separate JavaScript contexts, each running in its own process, each with different permissions, each communicating with the others via Chrome's asynchronous message-passing API. Understanding this separation is fundamental to understanding the codebase.

| Process | File | Chrome API Access | Runs In |
|---|---|---|---|
| Service Worker | `background.js` | Extension APIs, messaging, storage | Background (no DOM) |
| Offscreen Engine | `offscreen.js` | DOM, WebRTC, tabCapture, AudioContext | Hidden HTML page |
| Content Script | `controller.js` | Limited DOM access to the music tab | Injected into music tabs |
| Popup UI | `popup.js` | Extension APIs, messaging | Popup window |

These four processes communicate using `chrome.runtime.sendMessage()` and `chrome.tabs.sendMessage()`. There is no shared memory, no shared variables, no direct function calls between them. Every piece of coordination happens over these asynchronous message channels.

### 3.2 The Offscreen Document Engine

`offscreen.js` is the most complex file in the codebase. It owns all audio and network logic.

When the Service Worker decides to start broadcasting, it calls `chrome.offscreen.createDocument()` which instantiates `offscreen.html` in a hidden process. The `offscreen.js` script attached to that page immediately does the following:

First, it creates a `PeerJS` node. PeerJS is a library that wraps the WebRTC API with a simpler connection interface and uses a public signaling server (`0.peerjs.com`) to exchange connection handshake data between peers. The PeerJS node generates a unique Room ID — a 6-character alphanumeric string — that identifies this Host on the signaling server. This ID is sent back to the Background Service Worker via `chrome.runtime.sendMessage()`.

Second, it calls `chrome.tabCapture.capture()` with the ID of the active music tab. This API returns a live `MediaStream` object containing the raw audio output of that tab. This stream is stored in memory. It is not played back locally; it is held ready to be inserted into outgoing WebRTC peer connections.

Third, it sets up a listener for incoming PeerJS connection events. When a Listener connects, PeerJS fires an event, and `offscreen.js` responds by creating a new `RTCPeerConnection` for that peer, calling `addTrack()` to attach the live audio stream to it, opening a Data Channel for metadata and control messages, and registering the peer in an internal connection registry.

The connection registry is a key data structure. It is a Map from peer ID to an object containing the `RTCPeerConnection` instance and its associated `DataChannel`. When the engine needs to broadcast a metadata update to all Listeners, it iterates this map and sends the message over each registered Data Channel. When a peer disconnects, its entry is removed from the map.

### 3.3 The Content Script Observer

`controller.js` is injected by Chrome into any tab whose URL matches the domains listed in `manifest.json` — the Spotify Web Player, YouTube, Apple Music, and SoundCloud. It runs inside the tab's JavaScript environment, with direct access to that tab's DOM.

It serves two functions: observation and actuation.

For observation, it sets up `MutationObserver` instances pointing at the specific DOM elements that display the currently playing track title and artist name on each supported music service. When the displayed title changes — because the user skipped a track, because autoplay advanced, because the playlist looped — the observer callback fires, extracts the new text, and sends it upstream with `chrome.runtime.sendMessage()` to the Background Service Worker.

For actuation, it listens for incoming messages from the Background Service Worker. When it receives a `PLAYBACK_CONTROL` message with an action like `NEXT`, `PREV`, or `TOGGLE`, it locates the relevant button in the music tab's DOM and calls `.click()` on it. This simulates a physical user click, activating the native playback control without requiring any API credentials or special permissions beyond basic DOM access.

The DOM selectors used to find playback buttons and track title elements are service-specific. Spotify's player structure differs from YouTube's, which differs from Apple Music's. These selectors are defined as constants at the top of `controller.js` and are the most likely part of the codebase to need updating as music services change their front-end markup.

### 3.4 The Background Service Worker

`background.js` is intentionally thin. Its philosophy is to delegate everything it possibly can to the Offscreen Engine or the Content Script. Its own responsibilities are:

Managing the extension's operational state — whether broadcasting is active or inactive, what the current Room ID is, and how many Listeners are connected. This state is held in memory and also synced to `chrome.storage.local` so the Popup can read it.

Responding to messages from the Popup. When the user clicks "Start Broadcasting", the Popup sends a message to the Background Worker, which creates the Offscreen Document. When the user clicks "Stop Broadcasting", the Background Worker destroys the Offscreen Document and clears state.

Relaying messages between the Offscreen Engine and the Content Script. Because the Offscreen Document cannot directly message a specific tab's Content Script (it does not have access to `chrome.tabs`), it routes control commands through the Background Worker, which calls `chrome.tabs.sendMessage()` to reach the right tab.

### 3.5 The Popup UI

`popup.html` and `popup.js` implement the small window that appears when you click the JamSync extension icon in Chrome's toolbar. This is the only user-facing surface of the extension.

Its state reflects the current broadcasting status. When broadcasting is inactive, it shows a single "Start Broadcasting" button. When broadcasting is active, it shows the Room ID as copyable text, a rendered QR code (generated client-side using a QR code library) pointing to `https://jamsync-web.vercel.app?room=XXXXXX`, a live count of connected Listeners, a "Stop Broadcasting" button, and, if the fallback transport is active, an indicator showing which tier is in use.

The Popup communicates with the Background Worker via `chrome.runtime.sendMessage()` and reads persistent state from `chrome.storage.local` for cases where the popup was closed and reopened mid-session.

### 3.6 The PWA Listener App

`app.js` inside `jam/web/public/` is the entire Listener application. It is a single JavaScript file implementing a Vanilla JS Progressive Web App with no framework dependency.

On page load, it reads the `room` query parameter from the URL. If no parameter is present, it shows an error screen instructing the user to scan a valid QR code. If a Room ID is found, it instantiates a PeerJS client and initiates a connection to the Host identified by that Room ID.

Once the WebRTC connection is established, the incoming `MediaStream` is attached to a hidden `<audio>` element and playback starts automatically. A `MediaStreamSourceNode` feeds an `AnalyserNode` which runs a `requestAnimationFrame` loop to draw real-time frequency-domain data onto a Canvas element — the audio visualizer.

The app also registers with the browser's `MediaSession` API, which controls lockscreen media card behavior on Android and iOS. This registration allows the track title received over the Data Channel to appear on the device's lock screen alongside standard media controls.

---

## 4. Audio Transport Pipeline

Audio delivery is the core challenge of the system. WebRTC works reliably between two devices on the same Wi-Fi network, but real-world conditions — mobile data plans, corporate firewalls, university networks, VPNs — introduce NAT traversal failures that prevent direct peer connections. JamSync uses a three-tier pipeline that degrades gracefully through these conditions.

### 4.1 Tier 1 — Direct WebRTC MediaStream

This is the default and preferred path. The Offscreen Engine calls `RTCPeerConnection.addTrack()` with the captured `MediaStreamTrack`, and Chrome's built-in WebRTC implementation handles everything else: codec negotiation (Opus is selected automatically), variable bitrate encoding, forward error correction, jitter buffering, and hardware-accelerated encode/decode on both ends.

The audio quality on this path is excellent. Opus at 128kbps provides near-transparent audio quality with very low CPU overhead. End-to-end latency is typically 200–500ms on open Wi-Fi.

This path works when both the Host and Listener have publicly reachable IPs, or when a STUN server can help them discover their public addresses and punch through their respective NATs. It fails when both peers are behind symmetric NATs that block UDP hole-punching.

### 4.2 Tier 2 — TURN Server Relay

When STUN-based hole-punching fails, the ICE framework automatically promotes TURN candidates. A TURN server is a known public server that relays UDP packets between two peers who cannot reach each other directly. JamSync ships with authenticated credentials for a Metered.ca TURN instance configured in the `ICE_CONFIG` objects in both `offscreen.js` and `app.js`.

From the application's perspective, Tier 2 is identical to Tier 1. The same Opus codec, the same `MediaStream` track, the same `RTCPeerConnection`. The only difference is the routing path. Latency increases by roughly 100–200ms due to the relay hop.

The TURN server sees encrypted audio packets (SRTP) but cannot decrypt them. It is a relay, not a proxy; it does not inspect audio content. TURN relay incurs bandwidth cost on the Metered.ca account, billed per gigabyte relayed. At Opus 128kbps, one hour of relay for one Listener consumes approximately 57MB.

### 4.3 Tier 3 — AudioWorklet Data Channel PCM Fallback

Tier 3 activates when both Tier 1 and Tier 2 fail — which happens on networks that block all UDP traffic entirely, including TURN relay, and when the ICE gathering timeout expires. This is uncommon but occurs on particularly aggressive enterprise firewalls.

The WebRTC Data Channel uses SCTP over DTLS, which can tunnel over TCP as a fallback, giving it significantly broader firewall traversal than RTP media streams. Tier 3 exploits this by abandoning the native media pipeline entirely and manually encoding audio data as messages through the Data Channel.

The encoding process works as follows:

Inside the Offscreen Document, `audio-relay-processor.js` is registered as an `AudioWorkletProcessor`. This is a special class that runs on the audio thread inside Chrome's audio processing pipeline. It intercepts the raw PCM buffer from the tab capture at intervals of 4096 frames per processing call.

Each frame of audio is a 32-bit floating-point number in the range -1.0 to +1.0. The processor multiplies each value by 32767 and rounds to the nearest integer, producing a 16-bit signed integer representation. These integers are packed into an `Int16Array`. The array's underlying binary buffer is then Base64 encoded, producing a string of ASCII characters that is safe to send over the Data Channel without binary framing issues. This Base64 string is wrapped in a JSON object and dispatched via `conn.send()`.

On the Listener side, `app.js` receives these JSON messages on the Data Channel. It decodes the Base64 string back to an `ArrayBuffer`, reads the `Int16` values, and divides them by 32767 to restore the original floating-point range. These samples are pushed into a circular `Float32Array` ring buffer. A `ScriptProcessorNode` (or `AudioWorkletNode` in updated contexts) continuously reads from the front of this ring buffer and outputs the samples to the `AudioContext.destination`, producing continuous audio playback.

The ring buffer depth is tuned to absorb network jitter without introducing excessive latency. If the buffer runs empty (a momentary gap in packet delivery), silence is inserted rather than crashing the audio thread.

Tier 3 latency is higher — typically 700–1200ms — because of the encoding overhead and the buffering required to smooth out SCTP delivery irregularities. Audio quality is also slightly lower due to the 16-bit quantization compared to Opus, but remains acceptable for music listening.

---

## 5. Network Topology

JamSync uses a Star topology. The Host node is at the center. Every Listener connects directly to the Host, not to each other. Listeners have no awareness of other Listeners at the protocol level.

```
                        [ HOST ]
                           |
          ┌────────────────┼────────────────┐
          |                |                |
    [Listener A]     [Listener B]     [Listener C]
```

This was a deliberate design decision with three specific motivations.

The first is fault isolation. In a full mesh topology, every peer connects to every other peer. If Listener A has a bad connection causing packet loss and retransmission storms, that congestion affects Listeners B and C. In a star topology, A's problems affect only A's connection to the Host; B and C are unaffected.

The second is mobile efficiency. Listeners are frequently mobile devices on cellular connections. In a full mesh, each Listener would need to both transmit audio to and receive audio from every other peer, multiplying bandwidth usage by the number of connected peers. In a star topology, Listeners only receive. Their uplink carries only small Data Channel control messages, not audio.

The third is control surface simplicity. The Host's connection registry is the single authoritative source of all peer state. The Host can mute a specific Listener, send a targeted Data Channel message, or remove a peer from the session without coordinating with any other node.

The trade-off is that the Host's upstream bandwidth scales linearly with Listener count. At Opus 128kbps, 10 Listeners consume approximately 1.28Mbps of the Host's upload bandwidth. A typical 100Mbps home upload connection can support approximately 780 simultaneous Listeners before saturating. In practice, the cap is set at 20 Listeners per session for quality and manageability reasons.

---

## 6. Session Lifecycle — Step by Step

**Step 1: Host opens a music tab**

The user opens Spotify (or YouTube, Apple Music, or SoundCloud) in a Chrome tab. The JamSync Content Script is automatically injected into the tab because its domain matches the whitelist in `manifest.json`. The Content Script begins observing the DOM for track metadata but does not yet communicate with the rest of the system.

**Step 2: Host clicks "Start Broadcasting"**

The Popup sends a `{type: 'START_BROADCAST', tabId: <active tab id>}` message to the Background Service Worker. The Background Worker calls `chrome.offscreen.createDocument()`, spawning the Offscreen Engine. It also passes the tab ID to the Offscreen Engine so it knows which tab to capture.

**Step 3: Offscreen Engine initializes**

`offscreen.js` runs. It instantiates a PeerJS node, which connects to `0.peerjs.com` and receives a unique Room ID in response. It also calls `chrome.tabCapture.capture({tabId: <id>})`, which returns a live `MediaStream` containing the tab's audio output. Both operations complete within 1–2 seconds. The Room ID is sent back to the Background Worker, which stores it and passes it to the Popup for display.

**Step 4: QR code appears in Popup**

The Popup receives the Room ID, generates a QR code encoding the URL `https://jamsync-web.vercel.app?room=XXXXXX`, and renders it. The Host shows this QR code to their friends.

**Step 5: Listener scans the QR code**

The Listener's browser opens the PWA, which parses the `room` parameter and connects to `0.peerjs.com`. The PeerJS client sends an SDP Offer — a structured document describing the Listener's supported audio codecs, network addresses, and WebRTC capabilities — to the signaling server, addressed to the Host's Room ID.

**Step 6: WebRTC handshake**

The Offscreen Engine receives the SDP Offer via its PeerJS connection callback. It creates a new `RTCPeerConnection`, attaches the captured `MediaStreamTrack` via `addTrack()`, opens a `DataChannel` named `"jamsync-control"`, generates an SDP Answer, and returns it through the signaling server to the Listener. Both sides then exchange ICE candidates — network address/port pairs — and the ICE framework selects the best available path (direct, TURN-relayed, or failing that, Data Channel fallback).

**Step 7: P2P connection established**

Once the ICE negotiation completes and the peer connection reaches the `"connected"` state, the signaling server plays no further role. All communication from this point forward flows directly between the Host and Listener (or through the TURN relay if needed). The Listener's `<audio>` element receives the incoming `MediaStreamTrack`, begins playback, and the Listener hears the Host's audio.

**Step 8: Track metadata sync**

The Content Script's `MutationObserver` has been watching Spotify's DOM for title changes. When the track changes, it fires, extracts the new title and artist, and sends them to the Background Worker, which relays them to the Offscreen Engine. The Engine iterates its connection registry and sends `{type: "NOW_PLAYING", title: "...", artist: "..."}` over every connected Listener's Data Channel simultaneously. Each Listener's PWA updates its UI header and registers the new title with the device's `MediaSession` API for lockscreen display.

**Step 9: Host stops broadcasting**

The user clicks "Stop Broadcasting" in the Popup. The Background Worker calls `chrome.offscreen.closeDocument()`, destroying the Offscreen Engine. All `RTCPeerConnection` instances are closed, all Listeners' audio stops, and all Data Channels are terminated. The Popup returns to idle state.

---

## 7. Bidirectional Control System

One of JamSync's core features is that Listeners are not passive. They can control the Host's playback. The following describes the exact message path for a "Skip Track" command issued by a Listener.

The Listener presses "Next" on their PWA. `app.js` sends the following over the Data Channel:

```json
{ "type": "CONTROL", "action": "NEXT" }
```

The Offscreen Engine receives this on its Data Channel `"message"` event handler. It recognizes the `CONTROL` type and calls:

```javascript
chrome.runtime.sendMessage({ type: 'PLAYBACK_CONTROL', action: 'NEXT' });
```

The Background Service Worker receives this and calls:

```javascript
chrome.tabs.sendMessage(activeMusicTabId, { type: 'PLAYBACK_CONTROL', action: 'NEXT' });
```

The Content Script, injected into the Spotify tab, receives this and executes:

```javascript
document.querySelector(SELECTORS.spotify.nextButton).click();
```

Spotify advances to the next track. Its DOM updates with the new track title. The Content Script's `MutationObserver` fires, detects the change, and sends the new metadata upstream. The Offscreen Engine broadcasts the new title to all Listeners via Data Channel. Every Listener's screen updates.

The entire round-trip from button press to all-Listener UI update typically takes 300–700ms on a local network, with most of that time being Spotify's own animation delay before the DOM updates.

---

## 8. Data Channel Message Protocol

All non-audio communication between Host and Listeners flows over named WebRTC Data Channels. Every message is a JSON string. The following message types are defined:

**Host to Listener messages:**

`NOW_PLAYING` — Sent whenever the track changes on the Host's music tab.

```json
{ "type": "NOW_PLAYING", "title": "Track Title", "artist": "Artist Name" }
```

`HOST_STATE` — Sent when a new Listener connects, immediately providing them with current session state.

```json
{ "type": "HOST_STATE", "title": "Current Track", "artist": "Artist", "isPlaying": true }
```

`FALLBACK_ACTIVE` — Sent to a specific Listener when the Host detects the MediaStream path has failed and the AudioWorklet PCM path is being engaged for that peer.

```json
{ "type": "FALLBACK_ACTIVE" }
```

**Listener to Host messages:**

`CONTROL` — Sent when a Listener presses a playback control button.

```json
{ "type": "CONTROL", "action": "TOGGLE" }
{ "type": "CONTROL", "action": "NEXT" }
{ "type": "CONTROL", "action": "PREV" }
```

`PING` — Sent periodically to keep the Data Channel alive on networks that aggressively time out idle connections.

```json
{ "type": "PING" }
```

---

## 9. NAT Traversal Strategy

Network Address Translation (NAT) is the technology routers use to allow many devices to share a single public IP address. It creates a fundamental problem for peer-to-peer connections because neither peer knows the other's true public address or whether their router will accept unexpected incoming connections.

WebRTC's ICE (Interactive Connectivity Establishment) framework solves this through a structured discovery process. JamSync configures ICE with three types of candidates:

**Host Candidates** — The device's actual local network IP address. Works only when both peers are on the same local network.

**Server Reflexive (STUN) Candidates** — The device's public IP address and port as observed by a STUN server on the open internet. Discovered by sending a binding request to `stun.l.google.com:19302`. Works for most home broadband connections where the NAT type is "cone" (allows incoming connections to a port that the device has previously used for outbound traffic).

**Relay (TURN) Candidates** — An address on the TURN relay server that will forward packets to the device. Used when both peers are behind symmetric NATs (which assign a different port for each destination, making STUN hole-punching impossible) or when UDP is blocked entirely. JamSync uses Metered.ca as its TURN provider, with credentials configured in the `iceServers` array.

ICE tries all available candidates simultaneously and selects the highest-priority path that successfully exchanges packets. The priority order is: direct > STUN > TURN. If you need to use a different TURN provider — for example, a self-hosted Coturn server on your corporate infrastructure — update the `iceServers` array in both `offscreen.js` and `app.js` with your server's address and credentials.

---

## 10. Installation — Chrome Extension (Host)

**Prerequisites:** Google Chrome browser (version 88 or later for full Manifest V3 and `chrome.offscreen` support).

**Step 1:** Download the repository and locate the `jam/extension/` directory, or download the pre-built `jamsync.v1.1.zip` archive and extract it. You need the directory that contains `manifest.json`, not a zip file.

**Step 2:** Open Chrome and navigate to `chrome://extensions/` in the address bar.

**Step 3:** In the top-right corner of the Extensions page, toggle on the "Developer mode" switch.

**Step 4:** Click the "Load unpacked" button that appears after enabling Developer mode. A file picker opens.

**Step 5:** Navigate to and select the `jam/extension/` directory (the one containing `manifest.json`). Click "Select Folder".

**Step 6:** JamSync appears in your extensions list. A small icon appears in the Chrome toolbar. If you do not see it, click the puzzle-piece "Extensions" icon in the toolbar and pin JamSync.

**Step 7:** Chrome will prompt for permissions when you first start broadcasting. Accept all prompts. The required permissions are:

- `tabCapture` — required to capture the audio output of browser tabs
- `offscreen` — required to spawn the hidden Offscreen Document for DOM API access
- `scripting` — required to inject the Content Script into music tabs
- `storage` — required to persist session state across popup open/close cycles

These permissions are also declared in `manifest.json`. The extension does not request any permissions beyond what is listed there.

---

## 11. Installation — PWA Listener (Vercel)

**No installation is required for Listeners.**

The production PWA is live at `https://jamsync-web.vercel.app`. Listeners do not install anything. They navigate to the URL provided by the Host (either by scanning the QR code in the extension popup, or by following a direct link) and the application loads in their browser.

The PWA is installable as a standalone app. On Android with Chrome, a banner will appear after the first visit offering to "Add to Home Screen". On iOS with Safari, open the share menu and tap "Add to Home Screen". The installed app will appear on the home screen and run in a standalone browser window without the address bar.

---

## 12. Local Development Setup

**Prerequisites:** Node.js 18 or later, npm 9 or later.

**Chrome Extension — local changes:**

The extension loads directly from the filesystem. To make changes:

1. Edit files in `jam/extension/`.
2. Navigate to `chrome://extensions/` in Chrome.
3. Click the refresh icon on the JamSync extension card.
4. Reload any tabs that have the Content Script injected (Spotify, YouTube, etc.).

There is no build step for the extension. All JavaScript is loaded directly by Chrome.

**PWA — local development server:**

```bash
cd jam/web
npm install
npm run start
```

This starts a local development server (default port 3000 with hot reload). The extension in its default configuration points to `https://jamsync-web.vercel.app`. To test against your local server, temporarily change the base URL in `popup.js` from the production Vercel URL to `http://localhost:3000` and regenerate QR codes accordingly.

**Deploying the PWA to Vercel:**

```bash
cd jam/web
npm install
npx vercel --prod
```

Vercel will prompt for your account on first deploy and assign a project URL. Update the base URL in `popup.js` to your new Vercel project URL if you are deploying your own instance.

---

## 13. Configuration Reference

**TURN / STUN Server Configuration**

Both `jam/extension/offscreen.js` and `jam/web/public/app.js` contain an `ICE_CONFIG` constant near the top of the file:

```javascript
const ICE_CONFIG = {
    iceServers: [
        {
            urls: 'stun:stun.l.google.com:19302'
        },
        {
            urls: 'turn:relay.metered.ca:443',
            username: 'YOUR_METERED_USERNAME',
            credential: 'YOUR_METERED_CREDENTIAL'
        },
        {
            urls: 'turn:relay.metered.ca:443?transport=tcp',
            username: 'YOUR_METERED_USERNAME',
            credential: 'YOUR_METERED_CREDENTIAL'
        }
    ]
};
```

Replace the Metered.ca credentials with credentials from your own TURN provider if needed. To use a self-hosted Coturn server, replace `relay.metered.ca:443` with your server's address. Adding `?transport=tcp` to the TURN URL enables TCP fallback for networks that block UDP entirely.

**Maximum Listener Limit**

The maximum number of Listeners per session is enforced in `offscreen.js`. Search for the `MAX_LISTENERS` constant:

```javascript
const MAX_LISTENERS = 20;
```

Increase this value with caution. Each additional Listener adds approximately 128kbps to the Host's upstream bandwidth consumption, plus a new `RTCPeerConnection` instance in memory.

**Supported Music Domains**

The Content Script is injected only into tabs whose URLs match the list in `manifest.json` under the `content_scripts` host permissions. To add support for a new music service, add its domain pattern here and add the corresponding DOM selectors to `controller.js`.

**AudioWorklet Buffer Size**

In `audio-relay-processor.js`, the processing buffer size is defined as:

```javascript
const BUFFER_SIZE = 4096;
```

Smaller values reduce latency on Tier 3 but increase CPU overhead and message send frequency. Larger values reduce CPU overhead but increase Tier 3 latency. 4096 frames at 44100Hz represents approximately 93ms of audio per processing interval.

---

## 14. Known Issues and v1.1 Hotfix

**Issue JS-019 — Global stream termination on local mute (FIXED in v1.1)**

This was a critical bug present in all v1.0.x releases. When a Listener pressed the "Pause For Me" button — which was intended to mute audio on their device only — audio stopped for every Listener connected to the session.

The root cause is a WebRTC protocol behavior. When a `<audio>` element's `.muted` property is set to `true`, Chromium and WebKit browsers dispatch an RTCP (RTP Control Protocol) Receiver Report packet upstream to the sender, signaling that the receiving track is suspended. This is a bandwidth conservation feature: if a receiver is no longer consuming audio, there is no point in the sender continuing to encode and transmit it.

In JamSync, the Host's Offscreen Engine shares a single `MediaStreamTrack` across all outgoing `RTCPeerConnection` instances. When one Listener's RTCP suspension signal arrived at the Host and Chromium suspended the shared track, audio stopped flowing to every connected peer simultaneously.

The fix, deployed in v1.1, replaces the `.muted = true` approach with volume coercion:

```javascript
// v1.0 — DEFECTIVE: triggers RTCP Receiver Report suspension
$('remoteAudio').muted = true;

// v1.1 — CORRECT: silences audio locally without RTCP side effects
$('remoteAudio').volume = 0;

// v1.1 also handles the DataChannel PCM fallback path explicitly:
if (relayAudioCtx && relayAudioCtx.state === 'running') {
    relayAudioCtx.suspend();
}
```

Setting `.volume = 0` keeps the audio element actively consuming the WebRTC stream at the receiver level, preventing any RTCP suspension signal from being generated. From the Host's perspective, the Listener is still receiving audio normally — it simply has its output attenuated to zero locally. Other Listeners are completely unaffected.

The `relayAudioCtx.suspend()` call handles the Tier 3 AudioWorklet fallback path. When a Listener is on this path, audio is flowing through an `AudioContext` that is entirely separate from the `<audio>` element. Suspending it halts the audio processing graph locally and conserves mobile CPU cycles, without producing any network-level side effects (it is a local Web Audio API operation, not a WebRTC operation).

This fix was deployed exclusively to the Vercel PWA. No Chrome Extension update was required. All Listeners received the fix automatically on their next page load.

---

## 15. Technical Constraints and Limitations

**Chrome-only on the Host side.** The `chrome.tabCapture` API and `chrome.offscreen` API are Chrome-specific. The extension cannot be ported to Firefox or Safari without replacing the tab audio capture mechanism entirely. Listeners, however, can use any modern browser that supports WebRTC — Chrome, Firefox, Safari, Edge, and most mobile browsers.

**One tab per session.** The extension captures exactly one tab's audio per broadcasting session. Switching to a different tab as the audio source requires stopping and restarting the broadcast. The active tab ID is captured at broadcast-start time and does not dynamically follow focus changes.

**CORS and autoplay policies.** Some mobile browsers block audio autoplay until the user interacts with the page. The PWA's audio element has `autoplay` and `playsinline` attributes set, and the audio playback is triggered in response to the WebRTC `ontrack` event, which most browsers treat as a user-initiated action sufficient to bypass autoplay restrictions. If a Listener opens the PWA and hears nothing, they should tap anywhere on the screen to trigger a user interaction, which will allow the audio element to play.

**No end-to-end encryption for metadata.** Audio transmitted over WebRTC is encrypted by the SRTP (Secure Real-time Transport Protocol) layer — this is mandatory in all WebRTC implementations. However, track metadata (song titles, artist names) transmitted over Data Channels is encrypted only by the DTLS layer provided by WebRTC, not by any application-layer encryption. This is adequate for typical use but should be considered if you are deploying JamSync in a context where metadata privacy matters.

**PeerJS signaling server dependency.** The free public instance at `0.peerjs.com` is used for initial handshakes. This server is not operated by JamSync and has no uptime SLA. If the signaling server is down, new Listeners cannot join (the initial handshake fails), but existing sessions that have already connected continue to function because post-handshake communication is entirely P2P. For production deployments, consider running your own PeerJS signaling server — it is an open-source Node.js application available at [peerjs/peerserver](https://github.com/peers/peerserver).

**Maximum 20 Listeners per session.** This limit is a configurable constant in `offscreen.js`. It is not a technical ceiling imposed by WebRTC but a practical quality-of-service constraint. Each additional peer connection increases Host memory usage, CPU usage for connection maintenance, and upstream bandwidth consumption.

---

## 16. Glossary

**AudioContext** — A Web Audio API object that represents the audio processing graph. All audio operations in the browser — decoding, processing, encoding, playback — run through an AudioContext.

**AudioWorklet / AudioWorkletProcessor** — A way to run custom JavaScript code directly on the browser's audio thread with access to raw audio sample buffers. Used in JamSync's Tier 3 fallback to intercept PCM samples before they reach the output.

**chrome.offscreen** — A Chrome Extension API (Manifest V3) that creates a hidden HTML document with full DOM access, used to work around Service Workers' lack of DOM API access.

**chrome.tabCapture** — A Chrome Extension API that captures the MediaStream (audio and/or video) of a specific browser tab.

**Data Channel** — A WebRTC feature that enables bidirectional text or binary message passing between peers over the same connection as the media streams. JamSync uses Data Channels for metadata and control messages.

**ICE (Interactive Connectivity Establishment)** — The WebRTC framework for discovering viable network paths between two peers. It gathers Host, STUN, and TURN candidates and selects the best path.

**Manifest V3** — The current Chrome Extension specification, which replaced Manifest V2. Its defining change was replacing persistent background pages with short-lived Service Workers, removing DOM API access from background scripts.

**MediaSession API** — A browser API that allows web pages to register metadata (title, artist, album art) and event handlers with the operating system's media controls, enabling lock screen and hardware media key integration.

**MutationObserver** — A browser API that watches for changes to the DOM and fires a callback when elements are added, removed, or modified. Used in JamSync's Content Script to detect track title changes.

**Opus** — An open audio codec designed for real-time internet audio transmission. It is the default audio codec for WebRTC and provides excellent quality at low bitrates with low encode/decode latency.

**PCM (Pulse-Code Modulation)** — Raw, uncompressed audio data. A PCM audio buffer is a sequence of numbers, each representing the audio signal amplitude at one instant in time.

**PeerJS** — An open-source JavaScript library that wraps the WebRTC API with a simpler peer-connection interface and provides a hosted signaling server for initial handshakes.

**Progressive Web App (PWA)** — A website that uses modern browser features (Service Workers, Web App Manifest) to behave like a native app: installable to the home screen, capable of running offline, and able to register with system-level media controls.

**RTCP (RTP Control Protocol)** — The control channel that runs alongside RTP audio streams in WebRTC. RTCP carries statistics and control signals between peers, including Receiver Reports that indicate the state of incoming media track reception.

**RTCPeerConnection** — The core WebRTC API object representing a connection between two peers. It manages ICE negotiation, DTLS encryption handshake, codec negotiation, media track transmission, and Data Channel transport.

**SDP (Session Description Protocol)** — A text format used in WebRTC to describe the capabilities of a peer: which codecs it supports, which IP addresses and ports it can receive on, and what media types it wants to exchange. Peers exchange SDP Offer and SDP Answer documents during the handshake phase.

**SRTP (Secure Real-time Transport Protocol)** — The encrypted transport layer that carries audio (and video) data in WebRTC connections. All WebRTC audio is SRTP-encrypted by default — this is mandated by the WebRTC specification and not optional.

**STUN (Session Traversal Utilities for NAT)** — A protocol and server type that helps a device discover its public IP address and port as seen from the internet, enabling NAT hole-punching for peer-to-peer connections.

**TURN (Traversal Using Relays around NAT)** — A protocol and server type that relays UDP (or TCP) packets between two peers who cannot establish a direct connection. TURN is the last-resort ICE candidate type used when STUN hole-punching fails.

**WebRTC (Web Real-Time Communication)** — A browser standard that enables direct peer-to-peer audio, video, and data transfer between browsers without plugins or external software. All audio transmission in JamSync uses WebRTC.

---

*JamSync v1.1 — Built on WebRTC, Chrome Extension APIs, and Progressive Web App standards.*
