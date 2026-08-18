# MapSky Pilot & Flight Radar HUD — Web Edition

A standalone production-oriented web application built on the supported **3D Maps in the Google Maps JavaScript API**. It replaces the original Chrome-extension strategy of trying to drive the private `maps.google.com` UI.

## What was fixed from the extension prototype

1. **Real camera control** — WASD/buttons now update `Map3DElement.center`, while Q/E, R/F, +/- control heading, tilt and range.
2. **Supported Google integration** — uses `google.maps.importLibrary("maps3d")`, not consumer Google Maps internals.
3. **Photorealistic 3D** — uses `Map3DElement` in HYBRID mode, which renders satellite/photorealistic imagery where available.
4. **Error UI** — missing key, failed script download, WebGL2 failure, and Google authentication failures produce visible troubleshooting guidance.
5. **Radar backend** — OpenSky calls go through `/api/flights`, so optional OAuth credentials remain server-side and requests can be cached/rate-limited.
6. **Deployment adapters** — local/traditional Node server, Vercel function, and Netlify function are included.

## Quick start

Requires Node.js 20+.

```bash
cp .env.example .env
```

Set `GOOGLE_MAPS_API_KEY` in your shell/environment. The scripts intentionally do not parse `.env` automatically, to avoid introducing a dependency. Examples:

macOS/Linux:

```bash
export GOOGLE_MAPS_API_KEY='YOUR_KEY'
npm run dev
```

PowerShell:

```powershell
$env:GOOGLE_MAPS_API_KEY='YOUR_KEY'
npm run dev
```

Then open `http://localhost:4173`.

## Google Cloud configuration

### Required for this implementation

- A Google Cloud project.
- Billing attached to the project.
- **Maps JavaScript API** enabled.
- A browser API key restricted to your website referrers.
- API restriction on that key allowing **Maps JavaScript API**.

You do **not** need Map Tiles API for this build because 3D Maps is rendered directly by the Maps JavaScript API through `Map3DElement`.

### When Map Tiles API is needed

Enable **Map Tiles API** only if you decide to render Google's Photorealistic 3D Tiles yourself in a third-party 3D renderer such as Cesium. That is a different architecture from this project and has a different billing/usage surface.

### API key security

A JavaScript Maps key is delivered to the user's browser and therefore cannot be treated as a secret. `GOOGLE_MAPS_API_KEY` is kept out of git and injected at build time, but the production security boundary is:

- Website/HTTP-referrer restriction, e.g. `https://mapsky.example.com/*`.
- Local development referrer such as `http://localhost:4173/*`.
- API restriction allowing only Maps JavaScript API.
- Separate server-only credentials for OpenSky or any future private backend API.

Do not reuse this browser key for unrestricted server APIs.

## Controls

- W/S: forward/backward
- A/D: strafe left/right
- Q/E: rotate heading
- R/F: increase/decrease tilt
- Space/Shift: raise/lower map center altitude
- +/−: zoom by changing 3D camera range
- Mouse/touch gestures: supported directly by the 3D map
- On-screen controls: pan, zoom, rotate, tilt, reset

## OpenSky radar

Anonymous OpenSky access works without credentials but is rate limited. Optional OAuth client credentials can be configured server-side:

```bash
OPENSKY_CLIENT_ID=...
OPENSKY_CLIENT_SECRET=...
```

The browser never receives these values. Radar failure is non-fatal: the 3D map continues to operate and the HUD reports `RADAR OFFLINE`.

## Build

```bash
GOOGLE_MAPS_API_KEY='YOUR_KEY' npm run build
npm run check
```

The static frontend is emitted to `dist/`.

## Deployment

### Vercel

1. Import the repository.
2. Add `GOOGLE_MAPS_API_KEY` as a build environment variable.
3. Optionally add `GOOGLE_MAP_ID`, `OPENSKY_CLIENT_ID`, and `OPENSKY_CLIENT_SECRET`.
4. Build command: `npm run build`.
5. Output directory: `dist`.
6. Add the deployed Vercel domain to the Google API key's HTTP referrer restrictions.

The included `api/flights.mjs` provides the serverless radar endpoint.

### Netlify

1. Import the repository.
2. Configure the same environment variables.
3. `netlify.toml` already sets the build command, publish directory, function directory, and `/api/flights` redirect.
4. Add the Netlify production/preview domains you actually intend to use to the browser key's allowed referrers.

### Traditional Node server

```bash
export GOOGLE_MAPS_API_KEY='YOUR_KEY'
npm run build
npm start
```

Reverse proxy the Node process behind HTTPS with nginx/Caddy/Apache. Set `PORT` if needed.

## Troubleshooting: “it isn't working”

### `GOOGLE_MAPS_API_KEY is not configured`
The build did not receive the key. Set the environment variable and rebuild.

### `RefererNotAllowedMapError`
The current origin is missing from the API key's Website restrictions. Add the exact localhost or production origin/referrer pattern and retry after the setting propagates.

### `ApiNotActivatedMapError`
Enable Maps JavaScript API in the same Cloud project that owns the key.

### `BillingNotEnabledMapError` / `ClientBillingNotEnabledMapError`
Attach an active billing account to that Cloud project.

### `ApiTargetBlockedMapError`
The key's API restrictions do not include Maps JavaScript API.

### Grey/dark map or authentication warning
Open DevTools → Console. Google Maps reports its exact key/billing/quota error code there. The app also registers `window.gm_authFailure` to surface authentication failure in the UI.

### 3D terrain appears but buildings are flat
Photorealistic 3D surface coverage is not universal. Test a known covered major city. Terrain coverage is broader than detailed 3D surface/building coverage.

### Blank map / WebGL error
Use a current browser, enable hardware acceleration, update the GPU driver, and make sure WebGL2 is available. Corporate policies/remote desktops can disable hardware graphics.

### Map works but radar says OFFLINE
The map and OpenSky are intentionally independent. Check `/api/flights?lamin=40&lomin=-74&lamax=41&lomax=-73` directly. A 429 means OpenSky rate limiting; a 5xx can indicate upstream/network failure.

### Deployment works locally but not online
Almost always check these first:
1. Production domain is allowed in Google key HTTP-referrer restrictions.
2. The production environment variable existed **during the build**.
3. Maps JavaScript API is an allowed API on the restricted key.
4. Billing/quota is active.
5. `/api/flights` is mapped to the correct serverless function.

## Production recommendations

- Keep the Google browser key restricted by referrer and API.
- Configure Cloud billing budgets/alerts and quota caps appropriate to expected use.
- Use a custom production domain rather than allowing broad wildcard preview domains indefinitely.
- Add observability (frontend error reporting + server logs) before public launch.
- Add CSP after confirming all Google Maps domains needed by your deployed version.
- Add automated browser tests once you have a CI environment with a restricted test key.
- Do not place OpenSky secrets, database credentials, or other private keys in browser JavaScript.
