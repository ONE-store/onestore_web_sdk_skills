# Checklists: A (status check) and B (pre-release check)

For both lists, **read only. Do not edit files.**

## List A: status check (doctor)

Use exactly four verdicts: **present / missing / not applicable / needs confirmation**.

- **present**: you confirmed it at a file and line
- **missing**: you checked where it should be, and it is not there
- **not applicable**: this game does not need it (give the reason in one line)
- **needs confirmation**: the code cannot tell (server settings, ONE store registration, etc.). Say what to look at

The verdict says whether the thing named in the item exists. A14, A15, and A19 name things that
must **not** be there, so for them **missing** is the good result and **present** is a problem to fix.
Do not invent other verdicts such as "passed".

Show the full table; the point is to show what is already done too. Cite evidence as `path:line`.

| # | Item | What to check |
|---|---|---|
| A1 | SDK load | The SDK is imported in a `<script type="module">` and **runs before the game scripts** (a game script loaded with a plain `<script src>` runs first even if its tag comes later). The URL is the official CDN, `https://h5sdk.onestore.net/lib/{{version}}/…` |
| A2 | Entry point | `createSDK()` is called (or `window.ONEstoreH5` is used) |
| A3 | Call order | `initializeAsync` → `setLoadingProgress` → `startGameAsync`, in order and chained |
| A4 | Unsupported environment | The `err` in the `initializeAsync` result is checked (it resolves, it does not reject) |
| A5 | Feature checks | `sdk.ads.isSupported()` / `sdk.iap.isSupported()` before showing ad or purchase UI, called **after init** |
| A6 | Save on exit | An `on("exit")` listener exists, the save looks synchronous and within 0.5 seconds, and `quit()` is not called again |
| A7 | Save before quit | If the game calls `quit()` itself, it saves before the call (`exit` does not fire on that path) |
| A8 | Back key | `onBackPressed` is registered and is a **synchronous function** (no async) |
| A9 | pause / resume | Listeners exist, handling is idempotent, `ringerSilent` from `resume` is applied |
| A10 | Rewarded ads | `requestId` is passed to `showRewardedAsync`, and the reward branch is `status === "rewarded"` only |
| A10-1 | Ad preload | `loadRewarded` / `loadInterstitial` is called early, when the game starts (recommended; without it the first ad takes longer) |
| A11 | Reward granting | Whether the client grants currency directly (if so, needs confirmation: granting through the server postback is the policy) |
| A12 | Purchase | `reason` branches for `purchase` (`user_cancelled`, etc.) and the hook for server-side `purchaseToken` verification |
| A12-1 | developerPayload | `purchase` requests include `developerPayload`, **the slot for the information needed to grant the item (such as a user identifier).** It is optional, but without it the purchase result and PNS notification cannot be matched for granting. Also check that it is at most 200 bytes in UTF-8 |
| A13 | External navigation | `openDeeplink` is used instead of `window.top.location` or `window.open` |
| A14 | Frame-busting code | Frame-busting code such as `window.parent !== window` checks (should be **missing**) |
| A15 | Forbidden APIs | Use of `alert` / `confirm` / `prompt` / `print` (should be **missing**) |
| A16 | Viewport | `100dvh`, `overscroll-behavior: none`, `viewport-fit=cover`, and safeArea are applied |
| A17 | Frame-allow headers | No `X-Frame-Options`, and `frame-ancestors` is set. These are server settings, so if the code does not show them: **needs confirmation** |
| A18 | Cookies and storage | If the session depends on cookies, `SameSite=None; Secure`; saves are keyed by `playerId` |
| A19 | Debug leftovers | A `setDebugLogger` call left in the release code (should be **missing**) |
| A20 | Listener timing | `pause` / `resume` / `exit` listeners are registered before `initializeAsync()` (recommended; events can arrive right after init) |
| A21 | Calls into the game | When SDK code and game code are separate scripts, SDK code calls the game object only after it exists (waits with `whenGame` from SKILL.md or an equivalent). The `initializeAsync()` result can arrive before the game script runs |

If items that need values (placementId, productId, SDK URL, how requestId is issued) are empty, give
the fill-in form from SKILL.md under the table.

## List B: pre-release check

**List only what needs fixing.** Show passed items as a count. Order: blockers → non-blockers →
needs confirmation → passed count.

### Blockers: shipping as is causes incidents

| # | Item | Why it blocks |
|---|---|---|
| B1 | Rewards granted from the client-side `rewarded` alone | Reproducible abuse. The grant is confirmed through SSV → verification → server postback |
| B2 | Items granted without server-side `purchaseToken` verification | No protection against forged payments. Granting and consumption are the server's job, and the payment notification (PNS) also goes to the server |
| B3 | Frame-busting code left in | The game does not load at all |
| B4 | An `X-Frame-Options: DENY/SAMEORIGIN` response, or `frame-ancestors` without both `https://*.onestore.co.kr` and `https://*.onestore.net` | The game does not load, or **registration review rejects it** (server setting; if the code does not show it, needs confirmation) |
| B5 | Served over HTTP, or redirected to another origin | Blocked as mixed content, or may fail to load or communicate |
| B6 | `requestId` missing for rewarded ads | Every ad fails with `invalid_request` |
| B7 | The SDK is loaded from somewhere other than the official CDN (`https://h5sdk.onestore.net/lib/{{version}}/…`) | Depends on a self-hosted or unofficial copy. Replace it with the official URL |
| B8 | SDK code calls the game object before the game script has run | The game may never start: always in a regular browser, and in the app depending on timing |

### Non-blockers: works, but risky

| # | Item | Result |
|---|---|---|
| B9 | No save in `on("exit")` | Progress is lost on exit and restart |
| B10 | The exit save is async or takes over 0.5 seconds | The save is cut off (the app closes the screen within that time) |
| B11 | `onBackPressed` not registered, or async | The exit popup on every back press, or answers over 200 ms count as not consumed |
| B12 | Rewards given on `dismissed` or `no_fill` | Violates the policy; reward only on `rewarded` |
| B13 | `user_cancelled` shown as an error | An error popup after the user cancelled. Causes support tickets |
| B14 | `ringerSilent` ignored | Sound in silent mode. Causes support tickets |
| B15 | `alert` / `confirm` / `prompt` used | Does not work or is blocked in the iframe |
| B16 | safeArea not applied | UI hidden behind the system bars |
| B17 | The session depends on cookies or storage affected by partitioning | Sign-in and saves are lost intermittently |

### Needs confirmation: outside the code

- ONE store registration: test URL, ad placements (placementId), in-app products (productId)
- Postback endpoint registration and **idempotent handling** (postbacks can be resent)
- Frame-allow headers from the real server (`curl -sI <game URL> | grep -i -e x-frame -e content-security`)
