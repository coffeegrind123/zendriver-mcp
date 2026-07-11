# Browser Automation Learnings

Append-only file. Add entries as you discover non-obvious behaviors,
site-specific quirks, timing requirements, or selector patterns.

## Format

```
## YYYY-MM-DD: <brief title>
- **Context**: What you were doing
- **Learning**: What was non-obvious
- **Rule**: The new rule to follow
```

---

<!-- Add learnings below this line -->

## 2026-03-21: React/SPA forms need native value setter in Chrome
- **Context**: Filling Steam login form (React-based) — standard `type_text()` put both values in the same field
- **Learning**: React inputs don't respond to normal DOM value changes. Use `execute_js` with native setter: `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(input, value)` then `dispatchEvent(new Event('input', {bubbles: true}))`
- **Rule**: For React/Vue forms in Chrome, use native value setter via execute_js. For Firefox/Camoufox, use Playwright's `fill()` method instead.

## 2026-03-21: Cookie consent banners block all interaction
- **Context**: Steam login page had a cookie consent overlay covering the input fields
- **Learning**: Cookie banners use fixed/absolute positioning and cover interactive elements. Clicks silently hit the banner instead of the target element. No error is thrown — it just doesn't work.
- **Rule**: Always check for and dismiss cookie banners immediately after navigation: `click(text="Accept All")` or similar. Do this before any other interaction.

## 2026-03-21: JWT tokens in localStorage, not cookies
- **Context**: After Steam OpenID login to a Vue SPA, `get_cookies()` returned empty but the app was authenticated
- **Learning**: The SPA stored a JWT in `localStorage.token` and sent it via `Authorization: Bearer` header. Many modern apps do this — cookies are only for server-rendered sites.
- **Rule**: When investigating auth, check both `get_cookies()` AND `get_local_storage()`. Look for `token`, `jwt`, `access_token` keys.

## 2026-03-21: Auth tokens can appear in redirect URL params
- **Context**: After completing Steam OpenID flow, the app redirected to `/return?token=eyJ...`
- **Learning**: Some apps pass the auth token in the URL query string on the OAuth redirect, before the SPA processes it into localStorage. If localStorage is empty, check `window.location.href` for `?token=` params.
- **Rule**: When extracting auth tokens, check URL params first, then localStorage, then cookies.

## 2026-03-21: get_interaction_tree() can return empty on SPAs
- **Context**: Vue.js SPA (Vuetify framework) — interaction tree returned `[]`
- **Learning**: The DOM walker may not find elements in Shadow DOM or dynamically-rendered SPA components. `find_buttons()` and `find_inputs()` use different detection logic and may succeed where the tree fails.
- **Rule**: If `get_interaction_tree()` returns empty, try `find_buttons()`, `find_inputs()`, or `execute_js` to query the DOM directly.

## 2026-07-11: low_memory launch flags are Cloudflare-detectable
- **Context**: Inspecting play-cs.com (Cloudflare Turnstile). `bypass_cloudflare` timed out ("could not solve the checkbox") when the browser was started with `low_memory=true`.
- **Learning**: `low_memory` adds `--disable-gpu` / software-WebGL flags that anti-bot fingerprinting flags as automation, so the Turnstile never clears. Relaunching WITHOUT `low_memory` (plain `start_browser`) let the checkbox solve on the same site.
- **Rule**: For any Cloudflare/anti-bot site, launch `start_browser` WITHOUT `low_memory` (accept the higher memory use). Only use `low_memory` on constrained hosts for non-protected pages.

## 2026-07-11: execute_js does NOT await Promises (async returns {})
- **Context**: Ran an `async` IIFE in `execute_js` (a `fetch()` to grab a favicon) — the tool returned `{}` with no data.
- **Learning**: The wrapper serializes the return value synchronously; an `async` function returns a Promise that serializes to `{}`. `await`/`.then` results never come back.
- **Rule**: Keep `execute_js` synchronous. For network, either navigate directly to the resource URL and read it from the resulting document, or use a synchronous `XMLHttpRequest` (`xhr.open(..., false)`). Never rely on `async`/`await` inside `execute_js`.

## 2026-07-11: site-wrapped Turnstile needs a manual form submit after bypass
- **Context**: `bypass_cloudflare` returned "solved" and the widget showed a green "Success!", but the page stayed on "Verification required" — a site-custom challenge, not the standard CF interstitial.
- **Learning**: Some sites embed Turnstile in their OWN `<form action="/verify_captcha" method="post">` with a populated hidden `<input name="cf-turnstile-response">` (the solved token, ~700 chars) and a manual "Verify" submit button. `bypass_cloudflare` only solves the checkbox; it does not submit the form, and `click(text="Verify")` may not trigger the submit.
- **Rule**: After `bypass_cloudflare` shows "Success!" but the page doesn't advance, inspect `document.forms` for a challenge form with a filled `cf-turnstile-response`, then submit it directly: `execute_js("(function(){document.forms[0].submit();return true})()")`.

## 2026-07-11: static assets often bypass the JS challenge — fetch with curl
- **Context**: Needed each site's favicon; some sites were Cloudflare-gated on the HTML.
- **Learning**: CF challenges HTML *navigations*, not always static assets. `curl -A "<a real Chrome UA>" --compressed <favicon-url>` returned the real PNG (HTTP 200) even on a challenged domain. Without `--compressed` you get raw gzip bytes (file says "gzip compressed data", not "PNG").
- **Rule**: To pull static assets (favicons/images/css) from a CF-protected site, try `curl -A "<browser UA>" --compressed` on the host first — it's far simpler than driving the browser. Always include `--compressed`.

## 2026-07-11: cross-origin image → base64 needs a same-origin document
- **Context**: Tried to canvas a favicon into base64 from a page on a different origin — `canvas.toDataURL()` throws a taint SecurityError.
- **Learning**: Canvas read-back is blocked for cross-origin images without CORS. But navigating the browser DIRECTLY to the image URL makes the generated document same-origin with the image, so `document.images[0]` canvases cleanly. `https://www.google.com/s2/favicons?domain=<host>&sz=128` is also same-origin (google.com) and canvas-readable — but it often returns a generic/greyscale icon, so prefer the site's real favicon URL.
- **Rule**: To get a cross-origin image as base64, `navigate` to the image URL itself, then canvas `document.images[0]`. Prefer downloading the raw bytes with curl (previous entry) when you just need the file.
