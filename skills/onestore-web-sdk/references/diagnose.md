# Diagnosis from symptoms: "it doesn't work and I don't know why"

Do not start reading code with only a symptom. The cause is often outside the code (server headers,
ONE store registration, app version). **First narrow down where it stops, and ask about what the
code cannot tell.**

## Collect first

| Tool | What it shows |
|---|---|
| `sdk.setDebugLogger(msg => console.log("[H5SDK]", msg))` | The SDK's internal decisions. **The first tool for integration debugging.** It shows how far init got |
| Browser or WebView console | Frame blocking (`Refused to frame`), mixed content, and module load failures all show up here. On a device, open the WebView console with `chrome://inspect#devices` (only if the app build allows WebView debugging) |
| `sdk.getState()` | Which stage it stopped at. Check for an unsupported environment with `err` in the `initializeAsync()` result |
| `curl -sI <game URL>` | `X-Frame-Options`, `Content-Security-Policy`, redirects (3xx): server settings the code does not show |

## Symptom → how to narrow it down

### The screen is blank (the game does not appear at all)

Most likely not a code problem. Check in this order.

1. Whether `curl -sI` shows `X-Frame-Options`. `DENY` / `SAMEORIGIN` is the cause
2. Whether the game URL is HTTP, or redirects from HTTPS to another origin. That is the cause
3. `Refused to frame` or a CSP violation in the console means `frame-ancestors` is not set correctly
4. Whether frame-busting code (such as `window.parent !== window` checks) remains. Remove it
5. If none of these, get the first error in the console and decide from there

### `initializeAsync` returns `err`

Not an error: **the environment is unsupported** (it resolves, it does not reject).

- Opened in a regular browser → expected. Test in the ONE store app's web game test menu
  (`sdk-context.md`, "Device testing")
- `err` inside the app → check the SDK load order (later than the game scripts), or whether the SDK
  is called inside an iframe the game created. Initialization fails inside such an iframe

### `initializeAsync` rejects (`timeout`, etc.)

The app did not respond. Check the SDK load timing first (top level of the game document, before
the game code). If that is correct, it is an app or hosting problem: ask for the reproduction
environment (app version, device) and pass it to ONE store.

### Boot stops with `… is not a function` (the game does not start)

Check two things first.

1. **Whether a global has the same name as an element `id`.** If they match, like
   `<canvas id="game">` and `window.game`, then `window.game` is that element until the game code
   overwrites it. Rename the global
2. **Execution order.** The SDK is `type="module"` (runs after parsing), so a game script loaded with
   a plain `<script>` runs first. Conversely, if the game script was changed to a module, the SDK
   boot code may run before the game object exists. Check that it waits until the object is ready

### `sdk_not_initialized`

A call order violation: `startGameAsync()` or similar was called **before** `initializeAsync()`
completed. Find where the Promise chain breaks (calls made without `await` / `then`).

### `isSupported()` is false

- First check that it is called **after** `initializeAsync()`. Before that there is nothing to decide from
- False after init → the app version does not provide the feature. Code cannot fix it. Hiding the
  feature's UI is the right handling. For product lookup, the app must be v8.5.0 or later

### Ads always return `failed`

Narrow down by `reason`; each has a different cause.

| reason | Cause | Fixable in code? |
|---|---|---|
| `invalid_request` | Missing `placementId`, unregistered placement, placement type mismatch, or missing `requestId` for rewarded | Missing values: yes. Unregistered: **confirm the issuance with ONE store** |
| `no_fill` | No inventory | No. Only check that there is a path to continue without the reward |
| `already_showing` | Called again while an ad or purchase UI is showing | Prevent repeated taps, disable the button |
| `platform_not_supported` | This environment or app version is not supported | No. Hide the UI with `isSupported` |
| `timeout` | No response for the ad ending | Do not request the same placement again right away; guard against double grants with `requestId` |
| `show_failed` · `network_error` | Display or network failure | No. Ask for the reproduction conditions and pass them on |

### Rewards are not granted (the most common question)

**First check whether rewards are granted from the client-side `status: "rewarded"`.** That causes
most cases. The grant is confirmed through SSV → verification → the game company's server postback,
and the game has to ask its own server whether the reward was granted. If the code is right, hand
over postback endpoint registration and the `requestId` mapping (can the user be identified?) as
needs confirmation.

### Purchases fail

Narrow down by `reason` (the purchase reasons table in `sdk-context.md`).

- `already_owned` → buying an unconsumed product again. **Consumption is the game company's server's
  job.** Confirm whether the server consumed it
- `need_login` / `need_update` → the device state. Only check that there is UI telling the user what to do
- `user_cancelled` → not an error. If an error popup appears, that is what to fix
- `already_processing` → called again while an ad or purchase UI is showing. Disable the button
- `invalid_request` → check for a missing `productId` or a `developerPayload` over 200 bytes

### The back key exits the whole game

Look at the `onBackPressed` handler. With no handler, the default (exit popup) is expected. If a
handler is registered and it still happens: is it **async** (no answer within 200 ms counts as not
consumed)? Does it return exactly `true` (truthy is not enough)? Does it throw?

### The game freezes after an ad or the background

Look at the `pause` / `resume` listeners: is the handling idempotent (they can fire more than once),
and does `resume` actually restart the game loop?

### Saves are lost

Check every path.

1. No `exit` listener → there is no chance to save on exit
2. The `exit` save is async or takes over 0.5 seconds → it gets cut off. Only synchronous work
3. The game calls `quit()` itself → `exit` does not fire on that path. Does it save before the call?
4. Saves depend on cookies or localStorage → storage partitioning in a cross-origin iframe can show a
   different store. Server-side saves keyed by `playerId` are recommended

## Ask about what the code cannot tell

- Where it happened: a regular browser, or the ONE store app (web game test). The symptoms mean
  different things
- If the test menu does not appear in the app: whether the test URL and test ID are registered in ONEconsole
- Whether the placementId and products (productId) are issued and registered with ONE store
- Whether the served game URL exactly matches the test URL registered in ONEconsole
- Whether the postback and PNS endpoints are registered and the server is receiving them
