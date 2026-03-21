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
