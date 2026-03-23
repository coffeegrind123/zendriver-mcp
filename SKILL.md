---
name: browser-automation
description: Browser automation using browser MCP tools for web scraping, form filling, testing, and security auditing. Provides 35+ tools for controlling a Chromium browser — navigation, element interaction, DOM querying, tab management, network monitoring, and JavaScript execution. Use when asked to browse, scrape, fill a form, automate a website, check a page, open a browser, take a screenshot, monitor network requests, run a security audit, or interact with any web page programmatically. Features a token-optimized DOM walker that reduces HTML to compact JSON with numeric IDs.
metadata:
  author: coffeegrind123
  version: "1.1"
---

# Browser Automation

## Lifecycle: Every Session Follows This

```
start_browser() → navigate(url) → wait(1-2) → [work] → stop_browser()
```

No tool works until `start_browser()` completes. No page tools work until
`navigate()` completes. Always `stop_browser()` when finished.

| Scenario | Call |
|----------|------|
| Default (headful) | `start_browser()` |
| Headless scraping | `start_browser(headless=true)` |
| With proxy | `start_browser(proxy="socks5://host:port")` |
| Authenticated proxy | `start_browser(proxy="http://user:pass@host:port")` |
| Persist session | `start_browser(user_data_dir="/path/to/profile")` |
| Check if running | `get_browser_status()` |

## Primary Discovery: get_interaction_tree()

**ALWAYS use `get_interaction_tree()` as the FIRST step to understand a page.**

This is the single most important tool. It returns a token-optimized JSON
of all interactive elements — 96% smaller than raw HTML. Each element gets
a numeric ID you can click/type into directly.

```json
{"id": 1, "t": "btn", "l": "Submit", "r": "main"}
{"id": 2, "t": "in",  "l": "Email",  "r": "main", "v": ""}
{"id": 3, "t": "link", "l": "Sign Up", "r": "nav"}
```

- `t` = type (`btn`, `link`, `in`, `sel`, `chk`, `rad`, `tab`, `mnu`, `el`)
- `l` = label (inferred from aria-label/placeholder/text/title/alt)
- `r` = region (`hdr`, `nav`, `main`, `side`, `ftr`, `dlg`)
- `v` = current value (inputs only)

**Use numeric IDs directly:**
- `click(selector="1")` — clicks element with id 1
- `type_text(text="hello", selector="2")` — types into element with id 2

See `reference/dom-walker.md` for full type/region codes and label inference.

## Tool Selection Decision Tree

### "I need to find something on the page"

```
Interactive element (button, link, input)?
  YES → get_interaction_tree()
    Only buttons? → find_buttons(filter_text="optional")
    Only inputs?  → find_inputs(filter_type="text|email|...")
  NO → Visible text content?
    YES → get_text_content()
    NO  → get_content() [raw HTML — use sparingly, large tokens]
```

### "I need to interact with something"

```
Click → get_interaction_tree() → click(selector="<id>")
      → OR click(text="Button Text") if you know exact text
Fill form → fill_form(json={"#email": "a@b.com", "#pass": "123"})
          → OR type_text(text="value", selector="<id>")
Dropdown  → select_option(selector="<css>", value="option_value")
Upload    → upload_file(selector="input[type=file]", file_path="/path")
Submit    → submit_form() or press_enter()
```

### "I need to wait for something"

```
Page load       → wait(seconds=2)
Element appear  → wait_for_element(selector="css", timeout=10, visible=true)
Network idle    → wait_for_network(timeout=5)
Specific request → wait_for_request(url_pattern="api/data")
```

### "I need to extract data"

```
DOM data     → execute_js("document.querySelector('.price').textContent")
Visible text → get_text_content()
API response → clear_logs() → [action] → wait_for_request(url_pattern="api/")
Screenshot   → screenshot(save_path="/tmp/page.png")
```

## General Rules

1. **Autonomy**: Execute the full browsing workflow without asking for
   confirmation at each step. Ask only when genuinely ambiguous.

2. **Token discipline**: Prefer `get_interaction_tree()` over `get_content()`.
   Raw HTML burns context fast. Only use `get_content()` for HTML structure
   the interaction tree doesn't expose.

3. **Wait after navigate**: Always `wait(1-2)` after `navigate()`. SPAs need
   hydration time. Use `wait_for_element()` for dynamic content.

4. **Numeric IDs are ephemeral**: They change on every `get_interaction_tree()`
   call. Get the tree, use IDs immediately. Never cache across navigations.

5. **CSS selectors for stability**: When referencing the same element across
   page changes, use a CSS selector instead of a numeric ID.

6. **execute_js syntax**: Scripts must NOT start with `return`. Simple
   expressions directly. Complex logic: `(function(){ var x = 1; return x; })()`

7. **Network monitoring**: Call `clear_logs()` BEFORE the action you want to
   monitor, then `get_network_logs()` or `wait_for_request()` AFTER.

8. **Tab hygiene**: Close tabs opened with `new_tab()` when done.

9. **Error recovery**: If a click or navigation fails, `screenshot()` to
   visually diagnose. The page may have changed or a modal may block.

10. **Auth persistence**: Use `get_cookies()` / `set_cookies()` for cookie-based auth.
    For JWT-based SPAs, use `get_local_storage()` / `set_local_storage()` — many
    modern apps store tokens in localStorage, not cookies.

## Common Workflows

### Web Scraping
```
start_browser(headless=true) → navigate(url) → wait(2)
→ get_interaction_tree()     → get_text_content()
→ execute_js("...")          → stop_browser()
```

### Form Filling
```
start_browser() → navigate(url) → wait(2)
→ get_interaction_tree()
→ fill_form(json={"#email": "user@ex.com", "#password": "secret"})
→ click(selector="<submit_id>") → wait_for_network()
→ screenshot() → stop_browser()
```

### Multi-Tab
```
start_browser() → navigate("site-a.com") → wait(2) → [work]
→ new_tab(url="site-b.com") → wait(2) → [work]
→ list_tabs() → switch_tab(tab_id=0)
→ close_tab(tab_id=1) → stop_browser()
```

### API Response Interception
```
start_browser() → navigate(url) → wait(2)
→ clear_logs() → click(selector="<trigger_id>")
→ wait_for_request(url_pattern="api/v2/data")
→ stop_browser()
```

### Security Audit
```
start_browser() → navigate(url) → wait(2)
→ run_security_audit() → stop_browser()
```

## Error Handling

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| "Browser not started" | Forgot lifecycle | `start_browser()` first |
| "No page loaded" | Forgot navigation | `navigate(url)` + `wait()` |
| Click does nothing | Element not visible | `scroll_to_element()` then retry |
| Element not found | Page not loaded yet | `wait_for_element()` then re-query |
| Stale numeric ID | Page mutated | Re-call `get_interaction_tree()` |
| execute_js fails | Started with `return` | Remove `return`, use IIFE |
| Network logs empty | Didn't clear first | `clear_logs()` before action |
| Form submit no effect | Need explicit submit | `submit_form()` or `press_enter()` |
| Interaction tree empty | SPA/React not hydrated | `wait(3-5)` then retry, or use `find_inputs()`/`find_buttons()` |
| Clicks blocked by overlay | Cookie banner or modal | Dismiss banner first: `click(text="Accept All")` |

## Critical Pitfalls

- ❌ Do NOT call any tool before `start_browser()` — everything will fail
- ❌ Do NOT use `get_content()` as primary discovery — wastes tokens massively
- ❌ Do NOT cache numeric IDs across navigations — they regenerate each call
- ❌ Do NOT start `execute_js` scripts with `return` — use bare expression or IIFE
- ❌ Do NOT forget `wait()` after `navigate()` — SPAs need hydration time
- ❌ Do NOT leave tabs open — close with `close_tab()` when done
- ❌ Do NOT read network logs without clearing first — stale entries mislead
- ❌ Do NOT use `mouse_click(x,y)` unless no other option — coordinates are fragile
- ❌ Do NOT call `get_interaction_tree()` then ignore the IDs — that's the whole point
- ❌ Do NOT interact with elements before dismissing cookie consent banners — they block clicks
- ❌ Do NOT assume auth is in cookies — check `get_local_storage()` for JWT tokens too
- ❌ Do NOT take screenshots without a timeout — screenshots can hang indefinitely on slow or unresponsive pages, blocking the session. Always set a timeout (e.g. `timeout: 10000`)

## Self-Refinement Protocol

After completing a browser automation task, if you discovered something
non-obvious (a selector pattern, a timing requirement, a site-specific
quirk), append it to `LEARNINGS.md`:

```
## YYYY-MM-DD: <brief title>
- **Context**: What you were doing
- **Learning**: What was non-obvious
- **Rule**: The new rule to follow
```

See `reference/tools.md` for complete tool documentation.
See `reference/patterns.md` for additional automation recipes.
See `reference/dom-walker.md` for interaction tree internals.
