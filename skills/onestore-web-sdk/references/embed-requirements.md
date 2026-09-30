# iframe embedding requirements: what the game company must do

The ONE store hosting page loads the game page in a **cross-origin iframe**. Without the steps
below, **the game does not load at all, or it loads but saving and navigation break.**

## 1. Allow framing (the most common failure)

What counts is the game URL's **final response headers** (after your CDN and WAF).

| Game URL response headers | Result |
|---|---|
| `X-Frame-Options: DENY` or `SAMEORIGIN` | Blocked |
| CSP `frame-ancestors` without the ONE store domains | Blocked |
| CSP `frame-ancestors` including the ONE store domains (including a wildcard such as `*`) | Allowed |
| Neither header present | Allowed (open to every site) |

If both headers are present, `frame-ancestors` takes precedence. `X-Frame-Options` cannot allow
specific domains only; `ALLOW-FROM` is deprecated syntax.

**Option 1: allow only the ONE store domains (recommended).** Keeps your existing blocking policy
while allowing ONE store.

```
Content-Security-Policy: frame-ancestors https://*.onestore.co.kr https://*.onestore.net
```

**Allow both domains as a whole, not individual hosts.** `frame-ancestors` checks **every ancestor
screen** that wraps the game. Besides the game runtime screen, the **registration review tool** also
wraps the game, and the two are on different ONE store domains. Listing only the runtime domain gets
the game **blocked at registration review and rejected.**

**Option 2: remove the blocking headers (alternative).** Remove **both** `X-Frame-Options` and CSP
`frame-ancestors`; if you remove only one, the other keeps blocking. This also removes clickjacking
protection, so any site could embed the game page. Choose it only if your security policy allows it.

- **Values written as `<meta http-equiv>` are ignored by browsers.** Both settings work only as HTTP
  response headers
- They are often added by default by frameworks, CDNs, or WAFs even if you never set them (Spring
  Security defaults to `DENY`, helmet to `SAMEORIGIN`). If blocking continues after checking the
  application settings, check the layers in front of it
- ONE store domains **may change with prior notice.** With option 1, check the guide for the current
  values periodically
- To check: if the screen is blank, look for `Refused to frame` or `frame-ancestors` messages in the
  browser console

## 2. URL rules

- **HTTPS is required.** The parent page is HTTPS, so an HTTP game is blocked as mixed content
- Serve the game from the URL registered as the test URL in ONEconsole. Watch for `www` and
  country-specific domain differences
- **Avoid redirects to another origin.** The game may fail to load or to communicate with the SDK

## 3. Third-party storage limits (easy to miss)

Storage rules differ in a cross-origin iframe.

- Cookies are not sent without `SameSite=None; Secure`
- Browser storage partitioning can show **a different store than when the site is visited directly**
- **Reconsider designs that keep the session or saves in cookies.** Server-side saves keyed by
  `playerId` are recommended
- Take particular care when porting a game that used to run as a top-level page

## 4. Forbidden or unavailable

| Do not | Result or alternative |
|---|---|
| Access `window.top` / `parent.document` | SecurityError |
| `window.top.location = ...` (top-level navigation) | Not possible. Navigate externally through `sdk.openDeeplink(url)` (domain allowlist check) |
| `alert` / `confirm` / `prompt` / `print` | Do not use. Replace with your own UI |
| Frame-busting code (`if (window.parent !== window) top.location = ...`) | **Must be removed.** With it, the game does not load |
| Calling fullscreen or screen-orientation lock APIs directly | May not work inside the iframe. Do not depend on them |

## 5. Screen and input

- Use `100dvh` for the viewport height (`100vh` gets cut off on mobile) and
  `overscroll-behavior: none` to stop bouncing
- Handle notches with `viewport-fit=cover` plus `safeArea` from `initializeAsync`
- Audio autoplay works only after a user gesture
