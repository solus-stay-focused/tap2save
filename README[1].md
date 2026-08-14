# Tap2Save.net — Backend

A small Express API that turns your frontend's "Demo Mode" into a real
downloader, using [yt-dlp](https://github.com/yt-dlp/yt-dlp) for
extraction (YouTube, Instagram, TikTok, Twitter/X, Facebook) and
`ffmpeg` for merging/audio conversion. Nothing is stored on the server —
every file is streamed straight through to the browser.

## 1. Install prerequisites on the server

```bash
# Node deps
cd tap2save-backend
npm install

# yt-dlp (Python package, easiest way to keep it updated)
pip install -U yt-dlp
# or: sudo apt install ffmpeg && sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp && sudo chmod +x /usr/local/bin/yt-dlp

# ffmpeg (needed to merge video+audio and to make mp3s)
sudo apt install ffmpeg   # Debian/Ubuntu
# brew install ffmpeg     # macOS
```

Confirm both are on PATH: `yt-dlp --version` and `ffmpeg -version`.

## 2. Configure

```bash
cp .env.example .env
# edit ALLOWED_ORIGINS to match your frontend's domain
```

## 3. Run

```bash
npm start
# -> Tap2Save backend listening on http://localhost:8787
```

For production, run it behind a process manager (pm2/systemd) and a
reverse proxy (nginx/Caddy) that terminates HTTPS. Since responses are
streamed, make sure your proxy doesn't buffer the whole response body
(e.g. in nginx: `proxy_buffering off;` on the `/api/download` location).

## 4. API

### `POST /api/info`
```json
{ "url": "https://www.tiktok.com/@user/video/123..." }
```
Returns:
```json
{
  "id": "…",
  "originalUrl": "…",
  "platform": "TikTok",
  "title": "…",
  "thumbnailUrl": "…",
  "durationSeconds": 34,
  "uploader": "…",
  "formats": [
    { "formatId": "137", "label": "1080p", "ext": "mp4", "kind": "video", "hasAudio": false, "height": 1080, "filesize": 8345213 },
    { "formatId": "bestaudio", "label": "Audio 128kbps", "ext": "m4a", "kind": "audio", "hasAudio": true, "filesize": 1200000 }
  ],
  "recommended": { "video": "137", "audio": "bestaudio" }
}
```

### `GET /api/download?url=<encoded url>&formatId=<id>&type=video|audio&filename=<name>`
Streams the file back with `Content-Disposition: attachment`, so a plain
`<a href="...">` or `fetch().blob()` on the frontend triggers a normal
browser download.

Rate limits: 20 `/api/info` calls/min and 10 `/api/download` calls/min
per IP (tune in `server.js`).

## 5. Wiring it into your existing `index.html`

Right now the frontend fabricates sample data. Replace the mock
extraction step with a real call, and point each queue item's
`streamUrl`/`downloadUrl` at `/api/download` instead of the sample MP4s.
Rough shape (adapt names to match your actual `processUrls`/render code):

```js
const API_BASE = 'https://api.tap2save.net'; // your deployed backend

async function extractVideoInfo(url) {
  const res = await fetch(`${API_BASE}/api/info`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url }),
  });
  if (!res.ok) throw new Error((await res.json()).error || 'Extraction failed');
  return res.json();
}

function buildDownloadUrl({ originalUrl, formatId, type, title }) {
  const params = new URLSearchParams({
    url: originalUrl,
    formatId: formatId || '',
    type,
    filename: title,
  });
  return `${API_BASE}/api/download?${params.toString()}`;
}
```

Then in your queue-processing code, swap the call that currently builds
fake `formats`/`streamUrl` for `extractVideoInfo(url)`, and set each
format's `downloadUrl` via `buildDownloadUrl(...)`. The rest of your UI
(queue, ZIP bundling with JSZip, history, etc.) keeps working as-is since
it already just does `fetch(item.streamUrl).blob()`.

Also worth updating: the "⚠ Demo Mode" banner and the FAQ answer in the
guide modal that says this build has no backend, since that'll no longer
be true.

## 6. A note on scope

This only works reliably for public, non-DRM-protected videos. Private
posts, age-gated content, and some platforms' anti-bot protections may
cause `yt-dlp` to fail or require cookies/login — that's expected and
not something worth trying to defeat. Also keep in mind downloading
other people's content from these platforms generally isn't allowed by
their Terms of Service, so this is best treated as a personal-use /
own-content tool rather than a public redistribution service.
