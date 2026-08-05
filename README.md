# Fitgirl Downloader

A desktop application for Tauri v2 that downloads FitGirl Repacks from the
[fuckingfast.co](https://fuckingfast.co) hoster — automatically solving the
Cloudflare Turnstile challenge, resolving direct links, and downloading files
with parallel segmented connections.

Built with **Rust** (Tauri v2, `wreq` HTTP client with Firefox 148 emulation)
on the backend and **React 18 + TypeScript + Vite** on the frontend.

> **Disclaimer:** This tool only automates the download process for content
> hosted on third-party file hosters. The authors do not host, distribute, or
> promote any copyrighted content. You are responsible for using this software
> in accordance with your local laws and the terms of service of the sites it
> interacts with.

---

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Development](#development)
- [Building a Release](#building-a-release)
- [Usage](#usage)
- [Options](#options)
- [Controls](#controls)
- [Debug Logging](#debug-logging)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [License](#license)

---

## Features

- **One-click setup** — paste a FitGirl repack page URL or a list of direct
  `fuckingfast.co` links; the app extracts every file link automatically.
- **Automatic Cloudflare Turnstile solving** — a hidden WebView2 window loads
  the hoster page, auto-clicks the download button, waits for the `dlpass`
  cookie, and POSTs to the internal `/f/{id}/go` endpoint to read the
  `HX-Redirect` header that reveals the direct download URL.
- **Parallel segmented downloads** — each file is split into `N` ranges
  (default 4, up to 8) downloaded concurrently to maximize bandwidth.
- **Resilience** — per-part automatic retries (5 attempts) that resume from
  the last written offset, plus a 20 s idle timeout that detects and recovers
  from stuck/connection-hung connections.
- **Queue management** — download up to 5 files concurrently, with per-item
  **pause / resume / cancel** controls.
- **Cleanup on cancel** — cancelling a download deletes the partially written
  file.
- **Live progress** — per-file progress bar, MB downloaded, total size, and a
  smooth (EMA-smoothed) transfer-rate readout.
- **Selective download** — choose individual parts, select/deselect all, or
  skip `optional` files in one click.
- **Cross-platform packaging** — builds Windows installers (MSI + NSIS) via
  Tauri bundler.

---

## How It Works

### 1. Link extraction

Given a FitGirl repack page URL, the app fetches the HTML and looks for the
`fuckingfast` hoster link inside `div.entry-content ul > li:nth-child(2) > a`.
If the link hides more links inside a spoiler (`su-spoiler-content`), those
are extracted, sorted naturally, and deduplicated.

Each extracted link has the form:

```
https://fuckingfast.co/<file_id>#<filename>
```

The fragment (`#<filename>`) carries the real output filename.

### 2. Cloudflare resolution

The hoster page is protected by Cloudflare Turnstile, so a plain HTTP client
cannot obtain the direct link. The app opens a **hidden WebView2 window**
(`ff_resolver_<file_id>_<timestamp>`) pointed at the link and injects
`RESOLVER_JS`, which:

1. Blocks pop-up ad windows (`window.open = () => null`).
2. Waits for the `a.gay-button` element to become clickable (not disabled via
   `opacity:0.5` / `not-allowed`).
3. Clicks it and waits for the `dlpass` cookie.
4. POSTs `form` data to `/f/{file_id}/go` with the headers
   `HX-Request: true`, `HX-Current-URL`, `Origin`, and
   `Content-Type: application/x-www-form-urlencoded` (empty body).
5. Reads the `HX-Redirect` response header, which is the direct
   `https://dl.fuckingfast.co/dl/...` URL.

The window's navigation is intercepted (`on_navigation`); any navigation to
`dl.fuckingfast.co` is captured and forwarded to the Rust backend, which closes
the resolver window and starts the real download.

If auto-solving exceeds **180 seconds**, the window becomes **visible** so you
can solve the Turnstile manually; it then waits another 180 seconds.

### 3. Parallel download

The backend probes the file size with a `Range: bytes=0-0` request (expects
`206 Partial Content`), pre-allocates the file, and splits the download into
`parts` ranges. Each part:

- Opens its own file handle and seeks once to its start offset.
- Streams its range, writing sequentially (no shared locks, no per-chunk
  seeking → stable throughput).
- Updates global progress every 256 KB.
- On connection loss or idle (no data for 20 s) retries **from the last
  written byte**, up to 5 times with backoff.

---

## Prerequisites

| Requirement | Version / Notes |
| --- | --- |
| [Node.js](https://nodejs.org) | ≥ 18 (for frontend build) |
| [Rust toolchain](https://rustup.rs) | stable, recent |
| [Tauri CLI](https://tauri.app) | v2 (`@tauri-apps/cli`) |
| [NASM](https://www.nasm.us) | **required** to compile `btls-sys` (wreq's BoringSSL backend). Must be on `PATH` at build time. |
| [Microsoft WebView2 Runtime](https://developer.microsoft.com/en-us/microsoft-edge/webview2/) | required at runtime (preinstalled on most Windows 10/11 systems) |

> **NASM note:** if `cargo build` fails while compiling `btls-sys` with a NASM
> error, make sure `nasm.exe` is reachable, e.g. add
> `C:\Users\Black\AppData\Local\bin\NASM` to your `PATH` (install via
> `winget install NASM.NASM`).

---

## Development

```bash
# Install frontend dependencies
npm install

# Run the app in development mode (starts Vite + Tauri)
npm run tauri dev

# Or run the frontend alone (hot reload only, no Tauri backend)
npm run dev
```

The Vite dev server listens on `http://localhost:1420` (configured in
`src-tauri/tauri.conf.json`).

---

## Building a Release

```bash
# 1. Build the frontend (tsc type-check + vite build)
npm run build

# 2. Build and bundle the app
npm run tauri build
```

Output:

- Executable: `src-tauri/target/release/fitgirl-tauri.exe`
- Installers: MSI + NSIS under `src-tauri/target/release/bundle/`

---

## Usage

1. **Launch the app.** Select a save directory with **BROWSE** (the choice is
   remembered for next time).

2. **Enter links.** Paste either:
   - A FitGirl repack page URL (e.g. `https://fitgirl-repacks.site/...`), or
   - A list of direct `fuckingfast.co` links, one per line, in the
     `#filename` format.

   Click **PARSE** to load the files into the list.

3. **Choose which files to download.** Use the per-file checkboxes or the
   buttons:
   - **SELECT ALL** — check every file.
   - **DESELECT OPTIONAL** — uncheck files whose names contain *optional*.
   - **DESELECT ALL** — clear the selection.

4. **Configure the download** (before starting):
   - **Max:** number of files downloaded concurrently (1–5, default 1).
   - **Conn:** number of parallel connections (ranges) per file
     (1, 2, 3, 4, 6, 8 — default 4).

5. Click **DOWNLOAD**. Each file goes through:
   - `resolving` — Cloudflare Turnstile is being solved in the hidden window.
   - `downloading` — ranges are being fetched in parallel; progress bar, MB
     and transfer rate update live.
   - `done` — completed successfully.

> Files are written with the exact name from the link fragment, directly into
> the selected save directory.

---

## Options

| Option | Values | Default | Description |
| --- | --- | --- | --- |
| **Max** | 1 – 5 | 1 | Max concurrent files downloaded at the same time. |
| **Conn** | 1, 2, 3, 4, 6, 8 | 4 | Parallel range connections per file. More connections usually mean more throughput, but some servers throttle per connection. |

---

## Controls

While a file is downloading you can use the per-item buttons:

- **Pause** — stop that file temporarily (connection is paused).
- **Resume** — continue a paused file.
- **Cancel (✕)** — stop and **delete the partial file**.

The main button becomes **CANCEL DOWNLOADS** while anything is active and
cancels the whole queue (each partial file is cleaned up).

---

## Debug Logging

The app writes a timestamped log to:

```
%TEMP%\fitgirl_debug.txt
```

Entries include resolver navigation events (`nav: <url>`), the direct link
captured from the resolver window, transfer-rate samples every 2 s
(`speed_sample: <B/s> delta=<B> <downloaded>/<total>B`), and per-part retries
(`retry part [start-end] from <offset> attempt <n>: <error>`). Manual
Turnstile solves are logged as `resolved (manual): <url>`.

On a Rust panic, a crash dump is written to `%TEMP%\fitgirl_panic.txt`.

---

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| `Cloudflare/DDoS protection` | The site returned 403 to a plain HTTP request. Restart the download — the resolver window handles Turnstile. |
| Download starts but seems stuck | A connection may be hanging. The 20 s idle timeout now detects this and retries from the last byte. Check `fitgirl_debug.txt` for `retry part ...` entries. |
| A window appears after ~3 minutes | Auto Turnstile solving timed out; solve the challenge manually in the now-visible window. |
| Resolver errors (`resolver closed`, timeout) | Cloudflare could not be solved in time. Try again, or solve manually when the window appears. |
| Slow / fluctuating speed | Try increasing **Conn**, or lowering it if the server throttles per connection. The speed readout is EMA-smoothed, so short spikes are expected. |
| Cancel does not delete the file | Handles may still be open for a moment; deletion is retried for up to 60 s. If it persists, the file is deleted once the last connection closes. |
| `btls-sys` build failure (NASM) | NASM is missing from `PATH`. Install it and re-run the build. |

---

## Project Structure

```
fitgirl-tauri/
├── src/                        # React frontend
│   ├── App.tsx                 # UI, queue, polling, speed (EMA) display
│   ├── App.css                 # styles
│   └── main.tsx                # React entry
├── src-tauri/
│   ├── src/lib.rs              # All backend logic (Rust)
│   ├── Cargo.toml              # wreq, scraper, tokio, tauri, ...
│   └── tauri.conf.json         # App config, bundler targets
└── package.json                # Frontend scripts & deps
```

### Key backend pieces (`src-tauri/src/lib.rs`)

| Symbol | Purpose |
| --- | --- |
| `get_links_from_page` | Extracts `fuckingfast.co` links (incl. spoilers) from a repack page. |
| `resolve_via_webview` | Hidden WebView2 window + `RESOLVER_JS` Turnstile auto-solving. |
| `resolve_download_url` | Turns `fuckingfast.co` link into the direct `dl.fuckingfast.co` URL. |
| `probe_total` | HEAD-style `Range: bytes=0-0` size probe. |
| `download_part` | Downloads one range, writes sequentially, retries from last byte. |
| `parallel_download` | Splits the file, spawns part tasks, samples speed, cleans up. |
| `single_download` | Fallback single-stream download (used when size probing fails). |
| `start_download` | Entry command: resolve → probe → dispatch parallel/single. |

---

⚠️ Disclaimer / Warning
Please read this section carefully before using the software:

Use at Your Own Risk: Anyone who chooses to use this software takes full responsibility for their actions.
No Liability: The authors/developers are not responsible for any legal consequences caused by the use of this tool.
Modifications: We accept no liability for actions taken using either the original source code or any modified versions of it.
By using this software, you acknowledge that you are solely responsible for compliance with any relevant laws and terms of service.
