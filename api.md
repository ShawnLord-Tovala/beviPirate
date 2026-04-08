# Bevi Touchless API — Reverse Engineering Notes

Reverse engineered from the Bevi touchless web app JS bundle (`main-Cr80hG9-.js`, `index-30IuMcDy.js`).

## QR Code URL Format

```
https://bevitouchless.co/{deviceType}/{tabletId}/{token}
```

| Field | Example | Notes |
|-------|---------|-------|
| `deviceType` | `rey` | Known values: `tl`, `su20`, `rey` |
| `tabletId` | `6602a94e39f756d9` | Identifies the physical machine |
| `token` | `CBmCxelQSWGcgiNS3I75uQ==` | Base64 session token — scoped to a single scan session |

---

## WebSocket (real-time control)

### Connection

```
wss://bevitouchless.co/user-side/{tabletId}/{token}
```

- Token must be passed **raw** (no `encodeURIComponent`) — it appears verbatim in the path.
- Immediately after `onopen`, send a device hash to establish context (see Auth below).

### Auth — Device Hash

On open, the client sends a UUID v4 stored in `localStorage` under the key `bevi.touchless.deviceid`.
The server will reject all subsequent commands with `{"cmd":"error","value":"bevi context doesn't match"}` if this is not sent first.

```js
// Generate once, persist forever
const id = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
  const r = (Date.now() + Math.random() * 16) % 16 | 0;
  return (c === 'x' ? r : (r & 3 | 8)).toString(16);
});
localStorage.setItem('bevi.touchless.deviceid', id);

// Send on every WS open
ws.send(JSON.stringify({ cmd: 'hash', value: id }));
```

### Commands sent TO server

| `cmd` | Payload fields | Description |
|-------|---------------|-------------|
| `hash` | `value: <uuid>` | **Must be sent first** — establishes bevi context |
| `dispense` | `value: true/false`, `position: N` | Start / stop dispensing. Server times out after ~5s; re-send `true` every 3s to keep pouring. |
| `flavor` | `value: true/false`, `position: N`, `code: "str"` | Select / deselect a flavor |
| `intensity` | `value: 1–5`, `position: N`, `code: "str"` | Set flavor intensity |
| `water` | `value: <waterType>` | Set water type |
| `sparkling` | `value: true/false` | Toggle sparkling |
| `noop` | _(none)_ | Keepalive — send every ~20s |
| `pong` | `value: <timestamp>` | Reply to server `ping` |

### Messages received FROM server

| `cmd` | Key fields | Description |
|-------|-----------|-------------|
| `ack` | `value: "register"` | Confirms session is live after `hash` |
| `ack` | `value: <cmd>` | Echoes every command you send |
| `ping` | `value: <timestamp>` | Heartbeat — reply with `pong` |
| `subtext` | `code`, `value` | One message per available flavor; use these to build the flavor list |
| `flavor` | `code`, `position`, `value` | Machine-side flavor state changed |
| `intensity` | `code`, `position`, `value` | Intensity changed |
| `water` | `value: <waterType>` | Active water type |
| `waters` | `DISPENSE_STILL`, `DISPENSE_SPARKLING`, etc. | Which water types this machine supports |
| `sparkling` | `value: bool` | Sparkling on/off |
| `max_selectable` | `overall`, `flavors`, `enhancements` | Max simultaneous selections |
| `locale` | `value`, `metric`, `french` | Machine locale settings |
| `unregister` | `reason` | Session ended (timeout or machine takeover) |
| `error` | `value: "bevi context doesn't match"` | Sent if `hash` was not sent or is unknown |

### Water type values

```
DISPENSE_STILL
DISPENSE_SPARKLING
DISPENSE_LOW_SPARKLING
DISPENSE_AMBIENT
DISPENSE_HOT
```

---

## Flavor List — REST (CORS blocked from external origins)

The app fetches the flavor catalog over HTTP, but this endpoint is same-origin only — CORS blocks it from external web apps. Flavor data is available via the WebSocket `subtext` messages instead.

```
GET https://bevitouchless.co/flavors/{tabletId}/{token}
Headers:
  Accept: application/json
  Content-Type: application/json
  X-Api: true
```

The flavor `code` field follows the pattern `{name}_{id}`, e.g. `peach_mango_133`. Strip the trailing numeric ID and title-case the rest to get a display name.

---

## Session Lifecycle

1. User opens QR code → browser navigates to `https://bevitouchless.co/rey/{tabletId}/{token}`
2. App connects WebSocket to `wss://bevitouchless.co/user-side/{tabletId}/{token}`
3. Client sends `hash` → server responds `ack: register`
4. Server pushes machine state: `waters`, `subtext` (flavors), `water`, `sparkling`, `locale`, `max_selectable`
5. User interacts → client sends `flavor`, `intensity`, `water`, `dispense` commands
6. Server sends `unregister` on session timeout (~30s idle) or when machine is taken over
7. QR token is single-use per scan session — re-scan to get a new token

---

## Reference Files

The original minified JS bundles used for this analysis are in `reference/`:

| File | Description |
|------|-------------|
| `main-Cr80hG9-.js` | Main app bundle — Redux store, WebSocket middleware, API calls |
| `index-30IuMcDy.js` | Core React/Redux libraries + app config (contains `webSocketUrl`, `baseUrl`) |
| `App-CYYjerZF.js` | Root app component |
| `startRecording-DDj8pFof.js` | Session recording / analytics |
| `tablet-CWj2A276.js` | Device type enum: `tl`, `su20`, `rey` |
| `6a5c0228-8d35-44fb-b7f8-0c182711c8bc` | Compression/binary utility (zlib-style) |
