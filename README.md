# Relay web proxy

A small, dependency-free web proxy that runs in Node and is controlled from a browser. It fetches public `http://` and `https://` URLs server-side, rewrites common HTML links and resources through the proxy, and returns other assets unchanged.

## Run

```bash
npm start
```

On Linux, `npm start` launches Chromium in headed mode inside a virtual display;
Relay streams that browser to the page, so no desktop window is needed. Then
open <http://localhost:3000>.

For environments without `xvfb-run`, use `npm run start:headless`. Some sites
may still reject automated browser sessions regardless of headless mode. Relay
does not solve or skip Cloudflare challenges; when Cloudflare presents a human
verification, complete it in the remote browser viewport.

For sites with origin-bound authentication such as Xbox, open <http://localhost:3000/remote> (or select **Use real browser mode**). This uses a real Playwright Chromium session with its own cookies, JavaScript, redirects, and a live WebSocket viewport.

In browser mode, turn on **Low data** to reduce the live viewport stream to
960x540 JPEG frames at lower quality and one third of the normal frame rate.
It also stops remote audio from being sent to that session. This reduces the
streaming cost, but does not limit data fetched by the site itself.

Set `PORT` to use another port:

```bash
PORT=8080 npm start
```

To run headed Chromium manually, start it through a virtual display:

```bash
xvfb-run -a env HEADLESS=false npm run start:headless
```

The **Enable mic** control streams the local microphone into a PulseAudio
virtual capture device. Chromium uses that device as its native microphone,
which is compatible with sites that inspect browser audio devices before
calling `getUserMedia` for voice chat. It does not connect directly to an Xbox
console.

To return remote browser audio to the local page, install PulseAudio utilities
in the host or container:

```bash
sudo apt-get update && sudo apt-get install -y pulseaudio pulseaudio-utils
```

Relay creates a `relay_output` virtual sink and captures its monitor at 48 kHz
stereo. Click **Enable sound** in the relay page before opening party chat.
Set `AUDIO_CAPTURE=false` to disable this capture path.

## Boundaries

This is intended for local use or a trusted environment. It allows public targets only and rejects obvious local/private network destinations, but it is not a complete production SSRF defense. HTML rewriting is deliberately lightweight; sites that depend heavily on client-side routing, signed requests, or complex security policies may not work correctly.

## Loading metrics

Every proxied response includes diagnostic headers:

- `X-Proxy-Metrics`: JSON timings for DNS validation, upstream fetch, headers, body processing, total duration, byte count, and redirects.
- `X-Proxy-Upstream`: the final URL after redirects.
- `X-Proxy-Bytes`: the size of the rewritten response body.
- `Server-Timing` is not required; the JSON header is available directly in browser network tools and scripts.

HTML `href`, `src`, `srcset`, form targets, and CSS `url(...)` references are rewritten to stay inside the proxy. This improves compatibility with large sites such as Microsoft and Xbox, while highly dynamic authenticated applications may still require a full browser automation proxy.

The remote browser uses Chromium CDP screencasting over WebSockets. Mouse, wheel, double-click, drag, and keyboard events are dispatched directly to Chromium, so the page is interactive without screenshot polling. It requires the `playwright` and `ws` packages plus the installed Chromium runtime.

JavaScript responses also rewrite absolute URLs belonging to the current proxied site to local paths. This keeps single-page application API and route requests inside Relay while leaving third-party service URLs unchanged.

For single-page applications, Relay also mirrors the upstream pathname into the browser history and remembers the upstream origin for same-origin route requests. This prevents client-side routers from interpreting `/proxy?url=...` as their application path.

Opening `/` always returns the Relay homepage and clears the previous site routing cookie, so an old asset or Azure error cannot hijack a new session.

Upstream session cookies are copied to the local browser origin and forwarded on later requests, which helps refreshes on sites that use session-based routing or bot protection. Non-2xx upstream responses are returned with their original status and body so the upstream diagnostic page remains visible instead of being replaced by a Relay error.

Known-invalid empty Azure `comp` query parameters are removed before forwarding, including on redirect hops; valid query parameters are preserved.