## Interventions

Interventions live in `browser/extensions/webcompat/data/interventions/` as **individual JSON files**, one per site. Each file lists the Bugzilla bug numbers, the affected URLs, the type of breakage, and the interventions to apply.

The full schema reference is `browser/extensions/webcompat/intervention_schema.json`. At build time, `browser/extensions/webcompat/codegen.py` validates each JSON file against the schema and emits the final `run.js` plus any generated content scripts into `injections/generated/`.

### File structure

Intervention JSON:
```
browser/extensions/webcompat/data/interventions/{bug_id}-{domain}.json
```

Bug-specific JS (only when no reusable script in `injections/js/` fits):
```
browser/extensions/webcompat/injections/js/bug{bug_id}-{domain}-{purpose}.js
```

CSS is authored inline in the JSON's top-level `"css"` object (see "CSS injection" below). Generated CSS files are written to `injections/generated/` by `codegen.py`.

## Choosing an intervention type

Pick the intervention type that fits the breakage. When more than one type would work, prefer the least-invasive one —
for example, hiding an unsupported-browser banner with CSS or `hide_messages` is preferable to shipping a `ua_string` override, even if the UA override would also resolve the issue. UA overrides are powerful but might change behavior across the whole site, so reach for them only when narrower options aren't viable.

Common options:

- **Inline CSS** in the top-level `"css"` object — for unsupported-browser banners, layout fixes, anything purely visual.
- **`hide_alerts`** — for `alert()`/`confirm()` dialogs warning that Firefox is unsupported.
- **`hide_messages`** — for removing inline DOM warning banners by CSS selector plus text content (when the selector alone would also affect unrelated elements).
- **`modify_meta_viewport`** — for viewport meta tag overrides.
- **A reusable script** from `injections/js/` (e.g. `use_chrome_useragent.js`, `define_window_chrome.js`, `disable_FastClick.js`). List the bare filename(s) in `content_scripts.js`.
- **`ua_string`** UA override — when the site gates on the UA string and shimming `window.chrome` / `navigator.userAgent` from JS isn't enough.
- **A new bug-specific JS file** — when nothing reusable fits.

## Mapping bug signals to intervention types

Use the bug's symptoms (from Bugzilla comments, keywords, or live reproduction) to narrow down the intervention type:

- `impact:blocked` → reusable UA / `window.chrome` JS shims if the gate is client-side; `ua_string` override if the server blocks Firefox before any JS runs
- `impact:unsupported-warning` → inline CSS, or `hide_messages` / `hide_alerts`
- "works with Chrome mask" → reusable UA + `window.chrome` scripts (`use_chrome_useragent.js`, `define_window_chrome.js`)
- "alert recommending Chrome" → `hide_alerts: ["chrome"]`
- Missing API → check `injections/js/` for an existing shim before writing a new one
- Anything else → ask the user

## Intervention types

### 1. UA override

For sites that block Firefox but work with a Chrome UA string.

`ua_string` is **not** the same as the reusable UA scripts in `injections/js/`:

- **`ua_string`** rewrites the outgoing `User-Agent` HTTP header. The server sees the new UA, and client-side JS also reads it via `navigator.userAgent`. Use this when the breakage comes from the server (e.g. a Firefox-blocking response, or a different page variant served by UA).
- **Reusable UA scripts** (`use_chrome_useragent.js`, `add_chrome_to_useragent.js`, `remove_firefox_from_useragent.js`, etc.) only override `navigator.userAgent` and related JS-visible properties at runtime; the HTTP header is unchanged. Use these when the breakage is purely client-side (e.g. an in-page script that shows an unsupported-browser banner based on `navigator.userAgent`).

```json
{
  "label": "example.com",
  "bugs": {
    "1234567": {
      "issue": "firefox-blocked-completely",
      "matches": ["*://example.com/*"]
    }
  },
  "interventions": [
    {
      "platforms": ["android"],
      "ua_string": ["Chrome_with_FxQuantum"]
    }
  ]
}
```

Common `ua_string` values: `["Chrome_with_FxQuantum"]`, `["Chrome"]`, `["change_Firefox_to_Chrome"]`, `["add_Chrome"]`.

### 2. CSS injection (inline)

Define CSS in the top-level `"css"` object, keyed by an arbitrary name, then reference it from an intervention via `"css": ["key_name"]`. Codegen emits a real CSS file under `injections/generated/`.

```json
{
  "label": "example.com",
  "bugs": {
    "1234567": {
      "issue": "unsupported-warning",
      "matches": ["*://example.com/*"]
    }
  },
  "css": {
    "hide_browser_notice": "#browser-modal { display: none !important; }"
  },
  "interventions": [
    {
      "platforms": ["all"],
      "css": ["hide_browser_notice"]
    }
  ]
}
```

Long-form (when `all_frames` or `match_origin_as_fallback` is needed):

```json
"interventions": [
  {
    "platforms": ["all"],
    "css": {
      "all_frames": true,
      "match_origin_as_fallback": true,
      "which": ["hide_browser_notice"]
    }
  }
]
```

Rules:
- Every key in the top-level `"css"` object must be referenced by at least one intervention; extras fail the build.
- CSS values must be non-empty strings.

### 3. `hide_alerts` — suppress unsupported-browser dialogs

```json
"interventions": [
  {
    "platforms": ["all"],
    "hide_alerts": ["chrome"]
  }
]
```

Matches are lowercase substring checks against `alert()` / `confirm()` text. With options:

```json
"hide_alerts": {
  "all_frames": true,
  "alerts": ["chrome", "unsupported browser"]
}
```

### 4. `hide_messages` — remove inline DOM warnings

```json
"interventions": [
  {
    "platforms": ["desktop"],
    "hide_messages": [
      { "container": ".chakra-alert", "message": "Chrome" }
    ]
  }
]
```

`message` is a case-sensitive `innerText.includes(...)` check. Optional `click_adjacent` clicks a sibling instead of removing the container.

### 5. `modify_meta_viewport`

```json
"interventions": [
  {
    "platforms": ["android"],
    "modify_meta_viewport": {
      "all_frames": true,
      "modify": {
        "interactive-widget": {
          "value": "resizes-content",
          "only_if_not_equals": "resizes-content"
        }
      }
    }
  }
]
```

Supported keys: `height`, `width`, `initial-scale`, `minimum-scale`, `maximum-scale`, `user-scalable`, `interactive-widget`, `viewport-fit`.

### 6. Reusable JS scripts via `content_scripts.js`

Common spoofs live in `injections/js/` as plain (non-`bug`-prefixed) files. Reference them by bare filename — `codegen.py` combines multiple referenced scripts into a single generated `bug{n}-{label}.js`.

```json
"interventions": [
  {
    "platforms": ["all"],
    "content_scripts": {
      "js": [
        "spoof_as_windows.js",
        "use_chrome_useragent.js",
        "define_minimal_window_chrome.js"
      ]
    }
  }
]
```

Catalog (check `injections/js/` for the live list):

- **UA / vendor / platform**: `use_chrome_useragent.js`, `use_chrome_vendor.js`, `use_chrome_appVersion.js`, `use_linux_appVersion.js`, `use_win64_platform.js`, `add_chrome_to_useragent.js`, `add_samsung_to_useragent.js`, `remove_firefox_from_useragent.js`, `spoof_as_windows.js`, `define_userAgentData.js`
- **`window.chrome` shims**: `define_window_chrome.js`, `define_minimal_window_chrome.js`
- **API defines / shims**: `define_InstallTrigger.js`, `undefine_InstallTrigger.js`, `define_MozAppearance.js`, `define_SpeechRecognition.js`, `define_TouchEvent.js`, `define_WebKitMutationObserver.js`, `define_mozRTC_APIs.js`, `unprefix_mozCaptureStream.js`
- **Behavior toggles**: `disable_FastClick.js`, `disable_legacy_FastClick.js`, `disable_PACE_eventLag.js`, `disable_PDFJS_workers.js`, `disable_navigator_clipboard_read.js`, `disable_resizeMode_in_GDM.js`, `clear_navigator_plugins.js`, `deduplicate_registerProtocolHandler.js`, `emulate_MouseWheel_events.js`, `increase_performance_now_precision.js`, `load_mParticle_synchronously.js`, `predefine_OnetrustActiveGroups.js`, `round_pageYOffset.js`, `run_late_window_load_listeners.js`, `slow_down_wheel_events.js`, `support_nonstandard_timezones.js`, `undefine_firefox_properties.js`

If a reusable script almost fits but needs tweaks, prefer extending it (so other sites benefit) over copying it into a new bug-specific file.

#### Combining reusable scripts

A single intervention often needs a small combination of shims working together — list them in the same `content_scripts.js` array and `codegen.py` will concatenate them into one generated file.

```json
"content_scripts": {
  "js": ["use_chrome_useragent.js", "use_chrome_vendor.js"]
}
```

Pairings seen in the codebase (grep `injections/js/` references in existing intervention files for more):

- `use_chrome_useragent.js` + `use_chrome_vendor.js` — the most common combo, for sites that check both `navigator.userAgent` and `navigator.vendor`.
- `use_chrome_useragent.js` + `define_minimal_window_chrome.js` — when the site also probes `window.chrome`.
- `define_window_chrome.js` + `use_chrome_vendor.js` + `define_userAgentData.js` — fuller Chrome impersonation; use the non-minimal `define_window_chrome.js` here.

Start with the smallest combination that fixes the issue, and add more shims only if the site still detects Firefox.

You can mix reusable JS scripts with the special keys (`hide_alerts`, `hide_messages`, `modify_meta_viewport`) in the same intervention — codegen merges them all into one generated script.

### 7. Bug-specific JS file

Only when nothing in `injections/js/` is reusable. Filename: `injections/js/bug{bug_id}-{domain}-{purpose}.js`. Reference it from `content_scripts.js` by bare filename.

## Header / request rewriting

Available for cases where the breakage lives in network metadata or bodies. See `intervention_schema.json` for full shape.

- `alter_request_headers` / `alter_response_headers` — regex replace on header values
- `replace_string_in_request` — substitute strings inside a response body for matching URLs/types
- `run_script_before_request` — run privileged setup before a request

Response-header rewrite example:

```json
"interventions": [
  {
    "platforms": ["all"],
    "alter_response_headers": [
      {
        "headers": ["content-disposition"],
        "replace": "filename\\*=UTF-8''([^;]+)",
        "replacement": "filename=\"$1\""
      }
    ]
  }
]
```

## JS injection best practices (for bug-specific files)

### Guard pattern

Guard any code that modifies global APIs or prototypes so it doesn't run twice:

```js
if (!window.__firefoxWebCompatFixBug1234567) {
  Object.defineProperty(window, "__firefoxWebCompatFixBug1234567", {
    configurable: false,
    value: true,
  });

  // ... intervention code here
}
```

Naming: `__firefoxWebCompatFix` prefix, plus the bug number (`Bug1234567`) or a short descriptor (`Fastclick`).

When to guard:
- Overriding global APIs (`setTimeout`, `addEventListener`, etc.)
- Wrapping prototypes that would chain-wrap on re-run
- Anything that errors or misbehaves when executed twice

### Report applied fixes via `window.__webcompat`

If the script changes observable behavior, register it so the build's auto-generated logger reports it:

```js
window.__webcompat = (window.__webcompat ?? new Set()).add("navigator.userAgent");
```

`codegen.py` automatically appends a logger script to any generated file that touches `window.__webcompat`.

### Useful patterns

Spoofing `navigator.userAgentData`:

```js
if (!navigator.userAgentData) {
  const ua = navigator.userAgent;
  const mobile = ua.includes("Mobile") || ua.includes("Tablet");

  let platform = "Linux";
  if (mobile) {
    platform = "Android";
  } else if (navigator.platform.startsWith("Win")) {
    platform = "Windows";
  } else if (navigator.platform.startsWith("Mac")) {
    platform = "macOS";
  }

  const version = (ua.match(/Firefox\/([0-9.]+)/) || ["", "58.0"])[1];

  navigator.userAgentData = {
    brands: [
      { brand: "Not/A)Brand", version: "8" },
      { brand: "Chromium", version },
      { brand: "Google Chrome", version },
    ],
    mobile,
    platform,
  };
}
```

Loading a specific script synchronously:

```js
if (!window.__firefoxWebCompatFixBug1919263) {
  Object.defineProperty(window, "__firefoxWebCompatFixBug1919263", {
    configurable: false,
    value: true,
  });

  const { prototype } = HTMLScriptElement;
  const desc = Object.getOwnPropertyDescriptor(prototype, "src");
  const origSet = desc.set;
  desc.set = function (url) {
    if (url?.includes("mparticle.js")) {
      this.async = false;
    }
    return origSet.call(this, url);
  };
  Object.defineProperty(prototype, "src", desc);
}
```

Overriding a site's own JS property:

```js
if (!window.$feprelist_C) {
  let value = undefined;

  function check(obj) {
    if (obj === null) {
      return false;
    }
    if (Array.isArray(obj)) {
      return obj.some(check);
    }
    if (typeof obj === "object") {
      if ("barcodeEnabled" in obj) {
        obj.barcodeEnabled = true;
        return true;
      }
      return Object.values(obj).some(check);
    }
    return false;
  }

  Object.defineProperty(window, "$feprelist_C", {
    configurable: true,
    get() { return value; },
    set(v) {
      try { check(v); } catch (_) {}
      value = v;
    },
  });
}
```

## Bug `issue` values

The `issue` value is constrained by `intervention_schema.json` — pick the closest match from the schema's enum, or look at neighboring intervention files for examples.

## Platforms

Valid values: `"all"`, `"android"`, `"desktop"`, `"fenix"`, `"linux"`, `"mac"`, `"windows"`.

Every intervention entry must include `platforms`. It may also include `not_platforms` to exclude a subset (e.g. `"platforms": ["all"], "not_platforms": ["android"]`).

## Targeting (matches / blocks)

The `bugs[id]` object accepts any combination of:
- `matches` — content URLs to apply the intervention to (WebExtension match patterns)
- `exclude_matches` — exclude a subset (useful with a wildcard `*://*.example.com/*`)
- `blocks` — URLs to block outright
- `exclude_blocks` — exclusions from `blocks`

Match-pattern examples:
- `"*://example.com/*"` — all protocols, exact domain
- `"https://example.com/*"` — HTTPS only
- `"*://*.example.com/*"` — all subdomains
- `"*://example.com/path/*"` — specific path prefix

## Optional intervention-level fields

An `intervention` entry may also include:
- `min_version` / `max_version` — Firefox version gating (numeric)
- `only_channels` / `not_channels` — values from `"beta"`, `"esr"`, `"nightly"`, `"stable"`
- `pref_check` — apply only when specific about:config prefs match
- `skip_if` — `"InstallTrigger_defined"`, `"InstallTrigger_undefined"`, `"relaxed_name_validation_rules"`
