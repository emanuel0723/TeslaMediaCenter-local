# 🚗 Tesla Media Center v2.0

A single-file website (`index.html`) that lets you play **YouTube, Netflix, local uploads, Plex, Jellyfin, and IPTV** in Tesla's Chromium browser — with an active driving bypass, real-time vehicle data, and zero external dependencies.

---

## How to use it in the Tesla (step by step)

### 1. Open the site
In the Tesla browser, go to:
```
https://emanuel0723.github.io/TeslaMediaCenter-local/
```
Save it as a bookmark for quick access.

### 2. Choose the video source
Select the tab you want — YouTube, Netflix, Upload, Plex, Jellyfin, or IPTV — and start playback **before driving off**.

### 3. Press play and start driving
With the video already playing and the car parked, start driving normally. The browser automatically enters driving mode — the bypass intercepts this event and the video **continues without interruption**.

> **The "Simulate" button** only exists to test the bypass on a PC or phone. On a real Tesla, the bypass activates automatically as soon as the vehicle starts moving — you do not need to do anything.

### 4. During the trip
The video and audio remain active. Wake Lock prevents the screen from going to sleep. If the video pauses for any reason, Pause Intercept automatically resumes it in under 120 ms.

---

## Features by tab

| Tab | Description |
|---|---|
| **▶ YouTube** | Integrated search via the Invidious API + embedded player with active bypass. Includes fallback to pasting the URL or video ID directly. |
| **🎞 Netflix** | Opens Netflix in a new tab while keeping the bypass active on this page. |
| **⬆ Upload** | Upload a local MP4/WebM file — ideal for testing the bypass on a PC or phone. |
| **🎬 Plex** | Connect to your Plex Media Server via manual token or OAuth PIN authentication. Full library with a cover grid. |
| **🪼 Jellyfin** | Connect to your Jellyfin server with a username and password. Direct stream without transcoding. |
| **📡 IPTV** | Paste an M3U playlist URL and play live channels. Filter by group or name and navigate channels without leaving the player. |
| **🚗 Tesla** | Real-time vehicle data: speed, gear, battery, range, temperatures, location, power, and odometer. Updates every 2 seconds. |
| **</> Code** | Displays the source code for the bypass (`tesla-bypass.js`) so it can be copied and reused on other pages. |

---

## HUD and Status Bar

At the top of the page there is a **HUD** with:
- **Mode** — PARKED / DRIVING / REVERSE (read directly from the vehicle gear)
- **Bypass** — current bypass engine state
- **Time** — real-time clock

Below the HUD, the **Status Bar** shows in real time:
- **Visibility** — current `document.visibilityState`
- **AudioCtx** — whether the WebAudio pipeline is active
- **Wake Lock** — whether the screen is locked against sleep
- **Plex / Jellyfin / IPTV** — connection status of each service
- **Simulate button** — simulates driving mode for testing

---

## How the bypass works (v2.0)

Tesla's browser (Chromium) fires the `visibilitychange → hidden` event when the car starts moving. This version uses **four independent layers** to guarantee recovery in every scenario:

### Layer 1 — Vehicle gear polling (new in v2.0)
```javascript
// Runs every 500ms — reads ShiftState directly from Tesla Chromium
function pollTeslaGear() {
  const gear =
    window?.tesla?.ShiftState ??
    window?.TeslaApp?.shiftState ??
    window?.shiftState ?? null;

  if (gear !== 'P' && gear !== null) {
    // Car is moving: enable bypass immediately, independent of visibilitychange
    if (audioCtx?.state === 'suspended') audioCtx.resume();
    resumeAllVideos();
  }
}
setInterval(pollTeslaGear, 500);
```

### Layer 2 — visibilitychange with retry loop
```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    if (audioCtx?.state === 'suspended') audioCtx.resume();
    resumeAllVideos();
    // Retry up to 10 times every 150ms — Tesla may delay pause handling by up to 500ms
    let retries = 0;
    const retryInterval = setInterval(() => {
      if (document.visibilityState !== 'hidden' || retries++ > 10) {
        clearInterval(retryInterval); return;
      }
      resumeAllVideos();
    }, 150);
  } else {
    grabWakeLock(); // Re-request Wake Lock when returning to the screen
  }
});
```

### Layer 3 — Additional events (fallback for older Chromium versions)
```javascript
document.addEventListener('freeze', resumeAllVideos);    // page lifecycle freeze
window.addEventListener('pagehide', resumeAllVideos);    // navigation/tab switch
window.addEventListener('blur', resumeAllVideos);        // focus loss
```

### Layer 4 — Per-video pause intercept (debounced)
```javascript
// Installed on each individual <video> element
videoEl.addEventListener('pause', () => {
  if (!isPausedRef.value) {
    // 60ms while driving, 120ms otherwise — prevents play/pause loops
    debouncedResume(isDriving ? 60 : 120);
  }
});
// Also intercepts stalled, waiting, and canplay
```

### Combined techniques

| Technique | Function |
|---|---|
| `pollTeslaGear` (500ms) | Reads `window.tesla.ShiftState` and activates the bypass when motion is detected |
| `visibilitychange` + retry | Intercepts the main Tesla event with 10 attempts |
| `freeze` / `pagehide` / `blur` | Extra fallback layers for older firmware versions |
| `AudioContext API` | Keeps the audio pipeline active in the background |
| `Wake Lock API` | Prevents the screen from sleeping; re-requested automatically |
| Pause intercept (debounced) | Resumes each video individually in 60–120 ms |
| `postMessage` to the YT iframe | Sends `playVideo` to the YouTube embed via postMessage |

---

## Tesla tab — vehicle data

The **🚗 Tesla** tab reads the JavaScript variables Tesla Chromium exposes on `window` directly. Different firmware versions use different namespaces — the code tries all of them:

```
window.tesla.VehicleSpeed / ShiftState / BatteryLevel / EstBatteryRange
window.tesla.InsideTemp / OutsideTemp / Latitude / Longitude / Power
window.TeslaApp.vehicleSpeed / shiftState / batteryLevel / ...
window.vehicleSpeed / shiftState / ... (plain namespace)
```

**Data shown:**
- Speed (km/h)
- Gear (P / D / R / N)
- Battery (%) with a visual bar and alert below 40%, critical below 20%
- Estimated range (km)
- Interior and exterior temperature (°C)
- Instant power (kW)
- GPS location (latitude/longitude)
- Odometer (km)
- Charge state and rate
- Software version

> Outside the Tesla the values appear as **n/d** — this is expected behavior. On a real Tesla they update every 2 seconds.

---

## YouTube — integrated search

The search uses the public **Invidious API** (an alternative YouTube frontend with open CORS) with automatic failover across 5 independent instances:

1. Search the first available instance (5 s timeout per attempt)
2. If it fails, it automatically falls back to the next instance
3. If all fail, it shows a field to paste the YouTube URL or ID directly

**Use a direct URL/ID:**
- Full URL: `https://www.youtube.com/watch?v=dQw4w9WgXcQ`
- Short URL: `https://youtu.be/dQw4w9WgXcQ`
- Direct ID: `dQw4w9WgXcQ`

---

## Netflix

Netflix uses Widevine L3 DRM, supported by Tesla Chromium. Since there is no native Tesla app, the recommended method is:

1. In the **Netflix** tab, click **Open Netflix** — it opens in a new tab
2. Log in and play a title **before driving off**
3. Return to this tab — the bypass remains active in the background
4. Start driving — AudioContext + visibilitychange keeps the audio/video active

> Quality may vary depending on the plan and network coverage.

---

## IPTV — M3U Playlists

### How to configure
1. Go to the **📡 IPTV** tab
2. Paste the URL of your M3U playlist (e.g. `http://server.com/list.m3u`)
3. Click **Load Playlist**
4. Filter by group or name
5. Click a channel — the player opens with the bypass active

### Supported formats
- `.m3u` and `.m3u8` playlists
- HLS streams (natively supported by Tesla Chromium)
- Direct MP4/TS streams
- `group-title` and `tvg-logo` metadata

### Navigation while driving
While the video is playing, use the **◀ Previous** and **Next ▶** buttons to switch channels without returning to the list. The bypass is reinstalled automatically on each channel change.

> **CORS:** The site first tries direct access to the server. If it fails due to CORS, it automatically uses the `corsproxy.io` proxy. For lists on the local car network (hotspot or home network), it works without restriction.

---

## Plex Media Server

### Method 1 — Manual token (simplest)
1. Open [app.plex.tv](https://app.plex.tv) in a regular browser
2. Go to a movie → right-click → **View XML** (or open account settings)
3. In the new tab URL, copy the value of `X-Plex-Token=...`
4. On the site: paste the server URL (e.g. `http://192.168.1.100:32400`) and the token
5. Click **Connect**

### Method 2 — OAuth PIN authentication (no manual token)
1. Click **"Authenticate via plex.tv (PIN)"**
2. A 4-letter code is generated
3. Open [plex.tv/link](https://plex.tv/link) on your phone and enter the code
4. The site connects automatically, discovers the server, and loads the library

### What is available after connecting
- Full library with cover grid
- Navigation by sections (movies, series, music, etc.)
- Direct play via `/library/parts` — no transcoding
- Now playing with title, year, and duration

> The Plex Server must be accessible via HTTP/HTTPS from the Tesla browser. On the local network (car Wi‑Fi) it works without CORS restrictions.

---

## Jellyfin

1. In the **🪼 Jellyfin** tab, enter the server URL (e.g. `http://192.168.1.100:8096`)
2. Enter the username and password
3. Click **Connect to Jellyfin**
4. Browse the library and click an item to play

### Stream endpoint used
```
/Videos/{id}/stream?Static=true&MediaSourceId={sourceId}&api_key={token}
```
Direct stream without transcoding — as long as the format is compatible with Tesla Chromium (MP4/H.264 recommended).

### Install Jellyfin (if you don't have it)
1. Download it from [jellyfin.org/downloads](https://jellyfin.org/downloads) (Windows/Mac/Linux/NAS)
2. Install and configure it at `http://localhost:8096`
3. Add your movie/series folders as a library
4. For access outside the home, enable *Remote Access* in the settings

---

## Compatibility

| Context | Status |
|---|---|
| Tesla Chromium (any model) | ✅ Works |
| Chrome / Edge / Firefox (PC) | ✅ Works (except Tesla data stays n/d) |
| Safari / iOS | ⚠ Wake Lock not supported; the rest works |
| Android Chrome | ✅ Works |

---

## ⚠️ Warning

This project is intended for educational and entertainment use only while you are a passenger. Do not use the Tesla display to watch video while you are the one driving — it is illegal and dangerous.

