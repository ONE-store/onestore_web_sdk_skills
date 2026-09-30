# ONE store H5 Game SDK: API facts

This file is everything the skill knows about the SDK. **Do not invent classes, methods, fields, or
values that are not here.** The sources are the public type declarations of SDK v1.1.0 and the
official integration guide (v1.1.0, revised 2026-09-18). Public docs:
<https://onestore-dev.gitbook.io/dev/tools/web-sdk>

## What it is

A web SDK that connects an HTML5 game to the ONE store app (Android WebView). The ONE store hosting
page loads the game page in an iframe, and the SDK handles communication with the app. **The game
never deals with postMessage or the app bridge directly.**

- **The only supported environment is the WebView inside the Android ONE store app.** Regular
  browsers and iOS are not supported. Detect the environment with `err` in the `initializeAsync()`
  result and with each feature's `isSupported()`
- Feature areas: **lifecycle** (init, loading, start, pause, exit), **ads** (rewarded, interstitial),
  **in-app purchase**

## Loading and entry point

```html
<script type="module">
  import { createSDK } from "https://h5sdk.onestore.net/lib/{{version}}/onestore-h5-sdk.min.js";
  const sdk = createSDK();
</script>
```

- There are **two distributed bundles**: `onestore-h5-sdk.js` and `onestore-h5-sdk.min.js`
  (recommended). Put the version in `{{version}}`. It starts with `v`; if the user was not given
  one, use **the latest, `v1.1.0`**
- **The SDK must load before the game scripts.** It connects to the hosting page as soon as it
  loads; if it loads late, it can miss events that arrive right after initialization
- **Put the script that calls the SDK at the top level of the game document.** Initialization fails
  inside an iframe the game page creates itself
- `createSDK()` returns the same instance no matter how many times it is called. It is the same
  object as `window.ONEstoreH5`
- ES modules (`type="module"`) are blocked by CORS on the `file://` origin. Test locally over HTTP

## Environment and state

```ts
sdk.getSDKVersion(): string
sdk.getState(): "IDLE" | "INITIALIZING" | "READY" | "LOADING" | "PLAYING" | "PAUSED" | "TERMINATED"
```

- Detect the environment with `err` in the `initializeAsync()` result (unsupported environment) and
  each feature's `isSupported()`. **Feature support is checked per feature:** `sdk.ads.isSupported()`,
  `sdk.iap.isSupported()`

## Required call order (nothing works if it is broken)

```
initializeAsync() → setLoadingProgress(0–100) → startGameAsync() → (play) → quit() / restart()
```

```ts
sdk.initializeAsync(): Promise<InitResult>
sdk.setLoadingProgress(percentage: number): void
sdk.startGameAsync(): Promise<InitResult>
sdk.quit(): void      // the game exits on its own. No exit event; save before calling
sdk.restart(): void   // reload. An exit event gives a chance to save, then the app reloads the page
```

- `initializeAsync` is idempotent: calling it again returns the same Promise
- **In an unsupported environment it resolves with `err` instead of rejecting**, so the game's boot
  flow is not interrupted. Branch with `if (info.err)`
- Calling `startGameAsync` without init rejects with reason `sdk_not_initialized`. On failure the
  state returns to IDLE, so it can be retried
- After `restart()`, start again from `createSDK()` → `initializeAsync()`

### InitResult (returned by initializeAsync and startGameAsync)

| Field | Meaning |
|---|---|
| `playerId?` | **Pseudonymous per-game user ID.** The key the game server stores progress under |
| `locale?` | Device UI language (BCP-47, e.g. `ko-KR`) |
| `ringerSilent?` | Whether the device is in silent mode. Later changes arrive with the `resume` event |
| `safeArea?` | `{top, bottom, left, right}` (px): insets to keep clear of the system bars |
| `err?` | **Present only in unsupported environments** (e.g. `"Platform Not Supported"`). When set, the fields above are not provided |

## Lifecycle events

```ts
sdk.on(event, fn) / sdk.once(event, fn) / sdk.off(event, fn?)
```

**Register listeners before calling `initializeAsync()`.** Events can arrive in a row right after
initialization.

| Event | Payload | Rule |
|---|---|---|
| `pause` | `{ reason: "background" \| "ad" \| "payment" }` | Fires automatically when the app goes to the background, an ad shows (`"ad"`), or a purchase is in progress (`"payment"`). The game only needs to handle it idempotently |
| `resume` | `{ ringerSilent?: boolean }` | The field is present only if silent mode changed between pause and resume |
| `exit` | none | **Only save. No need to call `quit()` again.** The SDK and the app finish the exit |

`exit` rules (all important):

- Saving must finish **synchronously within 0.5 seconds**. The app closes the game screen or reloads
  the page within that time. Async saves (fetch, IndexedDB, and so on) are not guaranteed to finish
- It also fires on the `restart()` path. Unsaved progress is lost
- It does not fire when the game calls `quit()` itself. In that case, save before calling it
- A game with no listener exits immediately when an exit is requested
- `onClose(fn)` is an alias of `on("exit", fn)`. **Use only `on("exit")` in code and guidance**, so
  the game does not end up registering the same handler twice

## Back key

```ts
sdk.onBackPressed(handler: (() => boolean) | null): void
```

- Return `true` (consumed) if the game closed its own UI (a popup, etc.), otherwise `false`, which
  shows the app's exit confirmation popup
- **Must be a synchronous function.** The app waits 200 ms for the answer. No async
- No handler, or a handler that throws, counts as not consumed
- Only an exact `true` counts as consumed

## Ads (`sdk.ads`)

```ts
sdk.ads.isSupported(type?: "rewarded" | "interstitial"): boolean  // valid only after init
sdk.ads.loadRewarded({ placementId }): void        // preload: accepted only, no completion callback
sdk.ads.loadInterstitial({ placementId }): void    // (recommended, see below)
sdk.ads.isReadyAsync(type, placementId): Promise<boolean>  // a hint for showing the ad button
sdk.ads.showRewardedAsync({ placementId, requestId, timeoutMs? }): Promise<AdResult>
sdk.ads.showInterstitialAsync({ placementId, timeoutMs? }): Promise<AdResult>
```

`isSupported()` with no argument is true only when both rewarded and interstitial are supported.

### AdResult

| Field | Meaning |
|---|---|
| `status` | `"rewarded"` \| `"completed"` \| `"dismissed"` \| `"failed"` |
| `requestId?` | The value passed in the request, returned as is. **Returned on failure and timeout too.** Not present for interstitial ads |
| `reason?` | Failure reason (`no_fill`, `already_showing`, …). Branch on this |

Key rules (the most misunderstood area):

- **`placementId` must be issued in advance.** ONE store issues it per game, separately for rewarded
  and interstitial. A request with an unissued placement fails with no fallback
- **Preloading (`loadRewarded` / `loadInterstitial`) early, when the game starts, is recommended**, so
  the ad shows immediately when the button is pressed. Without it the ad still loads when requested,
  but the first one takes longer. Calling it repeatedly for the same placement does not load twice
- Rewarded results are `rewarded` | `dismissed` | `failed`; interstitial results are `completed` | `failed`
- **Show the reward only on `rewarded`.** `dismissed` (left early) and `failed` (including `no_fill`)
  are not rewarded. The policy is no reward when an ad could not be shown for lack of inventory
- **Do not grant rewards from the client-side `rewarded` result.** The grant is confirmed through
  SSV (server-side verification) → verification → the game company's server postback. `rewarded` is
  only a signal for the reward effect
- **`requestId` is required for rewarded ads.** Without it the request fails with `invalid_request`
  without calling the app. The SDK does not generate it. The postback has to identify the user, so
  **issuing it on the game company's server and mapping it to the user is recommended** (not required)
- Failures **do not throw**. They come back as `status: "failed"` with a `reason`
- A missing `placementId`, an unregistered placement, or a placement type mismatch (a rewarded ID
  used for an interstitial, or the reverse) fails with `invalid_request`
- Repeated taps are blocked by the SDK (`reason: "already_showing"`). Only one full-screen UI (ad or
  purchase) can be shown at a time
- `pause` / `resume` fire automatically around an ad. The game does not need to do anything extra
- `timeoutMs` is a backstop **until the ad finishes**. Default 600000 (10 minutes), allowed
  1000–1800000 (30 minutes); out-of-range values fall back to the default. Usually not needed
- An ad can still show late after a `timeout` failure. Avoid requesting the same placement again
  right away, and guard against double grants with the returned `requestId`
- Requesting an ad while unsupported returns `{ status: "failed", reason: "platform_not_supported" }`
  without calling the app
- `isReadyAsync` returns `false` for an unregistered placement, a placement type mismatch, a lookup
  failure, or a missing `placementId` (it does not throw)

## In-app purchase (`sdk.iap`)

```ts
sdk.iap.isSupported(feature?: "purchase" | "getProductDetails"): boolean   // valid only after init
sdk.iap.purchase({ productId, productName?, developerPayload? }): Promise<PurchaseResult>
sdk.iap.getProductDetailsAsync(productIds: string[]): Promise<ProductDetails[]>
```

**`isSupported()` with no argument means `"purchase"`** (whether purchase is supported). This differs
from ads, where no argument means both. Always check product lookup support with
`isSupported("getProductDetails")`: some app versions support purchase but not lookup.

Preparation: register products in **ONEconsole: 수익화 > 인앱 상품 (Monetization > In-App
Products)**. Web products support **managed products only** (no subscriptions).

- `productId` is required: the custom product ID registered in the Developer Center. `productName`
  is optional (if omitted, the registered product name is shown)
- `developerPayload` **holds the information needed to grant the item.** It is optional but **very
  important**: when the game receives the purchase result or the server receives the PNS (payment
  notification), this value identifies which user and which purchase it is. Put in what is needed
  to grant the item, such as a user identifier, and map it on the server. **Up to 200 bytes in
  UTF-8.** Longer values reject with `invalid_request` without calling the app
- It waits until the purchase UI ends (success, failure, or cancel). **There is no SDK timeout**
- **Purchase requests trigger `pause` / `resume`, and the `pause` reason is `"payment"`.** The
  purchase call site does not need to pause the game; handle it consistently in the `pause` listener
- Failures reject with `SDKError`. Branch on `reason` only

### PurchaseResult

| Field | Meaning |
|---|---|
| `orderId` | Order ID (receipt information) |
| `purchaseTimeMillis` | Purchase time (epoch millis) |
| `purchaseId` | Purchase ID |
| `purchaseToken` | Purchase token. **Use it for server-side verification** |
| `productId` | The requested in-app product ID |
| `developerPayload?` | Only if it was passed in the request |

Granting rules (all important):

- A successful `purchase()` response **only signals that payment completed; it is not a reason to
  grant the item.** The game company's server grants the item after verifying the payment with
  `purchaseToken`
- Payment completion is **notified to the game company's server via PNS**. PNS arrives even if the
  client never gets the response (network loss, app killed), so **build granting around the server**
- **Consumption is also the game company's server's job.** The app does not consume purchases
  automatically, so buying an unconsumed product again fails with `already_owned`. See the ONE store
  server API documentation

### Purchase reasons

| reason | Meaning |
|---|---|
| `invalid_request` | Missing required parameter, bad format, or `developerPayload` over 200 bytes |
| `user_cancelled` | The user cancelled the purchase |
| `service_unavailable` | Network or processing error on the device or server |
| `already_owned` | Buying an unconsumed product again |
| `payment_failed` | Payment failed |
| `need_login` | Not signed in to the ONE store app |
| `security_error` | Abnormal purchase request, or a required permission was denied |
| `already_processing` | A purchase or ad UI is already showing (only one full-screen UI at a time) |
| `need_update` | The ONE store service app is missing or outdated. Tell the user to update the app |
| `internal_error` | Internal error |
| `platform_not_supported` | This environment or app version does not provide the feature. Check purchase with `sdk.iap.isSupported()` and lookup with `sdk.iap.isSupported("getProductDetails")` beforehand |

### Product lookup (`getProductDetailsAsync`)

For showing prices in the game's shop. It does not show a purchase screen, so it can be called while
an ad or purchase is showing (it is not subject to the full-screen limit or pause / resume).

- **Works on ONE store service app v8.5.0 or later** (release notes, v1.1.0 update of 2026-09-17).
  On older apps `isSupported("getProductDetails")` is false; continue without showing prices
- **Up to 20 `productIds` per call.** A non-array, an empty array, more than 20, or an empty string
  fails with `invalid_request` without calling the app. For 21 or more, split into calls of 20
- The response is `ProductDetails[]`: `productId` (the requested value), `type` (`"inapp"`), `title`,
  `price` (display string), `priceAmountMicros` (price × 1,000,000, for math and sorting),
  `priceCurrencyCode` (e.g. `"KRW"`)
- **An empty array is not an error.** Products that are not on sale or not registered are left out
  (partial results), and the order does not follow the request. Match by `productId`
- `price` is a string. Do not use it for arithmetic; use `priceAmountMicros`
- Unsupported → `platform_not_supported` without calling the app. No response in 10 seconds →
  `timeout`. Other failure reasons follow the purchase table

## External links

```ts
sdk.openDeeplink(url: string): Promise<void>
```

The app validates the allowed domains. Both an unlisted domain and a format error reject with
`invalid_params` (they are not distinguished). Navigating directly, for example with
`window.top.location`, does not work inside the iframe; external navigation must go through this API.

## Error model

Errors the game sees are `SDKError`. **Branch on the `reason` string only.** Numeric error codes are
not passed to the game. `message` is supplementary text for logs; do not branch on it.

| Group | reason |
|---|---|
| Common | `invalid_params` · `unsupported_action` · `platform_not_supported` · `user_cancelled` · `rate_limited` · `timeout` · `internal_error` |
| Ads | `invalid_request` · `unsupported_screen` · `not_initialized` · `no_fill` · `show_failed` · `already_showing` · `network_error` |
| Purchase | The "Purchase reasons" table above |
| SDK checks | `sdk_not_initialized` (call order violation) |

Treat an unknown reason like `internal_error`.

## Debugging

```ts
sdk.setDebugLogger(fn: (msg: string) => void): void
```

Exposes the SDK's internal decisions as strings. **The first tool for integration debugging.**
Remove it from release builds.

## Device testing

How to run the game inside the ONE store app. Follow the order, or the menu does not appear.

1. **Register the test URL.** In ONEconsole, go to Apps > 상품관리 > Web 상품관리 > 기본정보
   (Apps > Product Mgmt. > Web App Mgmt. > Main Info), enter the game address in the test URL
   field, and save
2. **Register a test ID.** Register the account you will test with as a test ID. **Without it, the
   menu in step 3 does not appear in the app.** Choose Sandbox as the payment environment to avoid
   real charges
3. **Open web game testing.** In the ONE store app signed in with that account, go to
   마이 > 이용 안내 > 웹게임 테스트 > 내 테스트 게임 (My > User Guide > Web Game Test > My Test Games)
   and select the game
4. **Test purchases.** Follow the ONE store purchase test procedure

If the menu does not appear, check step 2 (test ID registration) first.
