## Interventions

Interventions are defined inside `browser/extensions/webcompat/data/interventions/` folder and use **individual JSON files** as the source of truth.
Each intervention file lists related bugzilla bug numbers, the affected URLs, the rough type of breakage resolved, and the actual interventions which should be applied under which circumstances.

If any CSS or JS changes are also required, they must be placed in separate files bundled with the extension, with the names of all such files then also listed in the definition in `content_scripts` sections in the appropriate interventions. These content scripts behave like any other Web Extension content scripts, and may be loaded in all frames as necessary (`"all_frames": true`).


### File Structure

Intervention file:
```
browser/extensions/webcompat/data/interventions/{bug_id}-{domain}.json
```
JS or CSS Injection (if needed)
```
browser/extensions/webcompat/injections/js/bug{bug_id}-{domain}-{purpose}.js
browser/extensions/webcompat/injections/css/{bug_id}_{domain}-{purpose}.css
```

## Intervention Types

These are the most common intervention types:

### 1. UA Override

When site blocks Firefox but works with Chrome UA string / Chrome Mask extension.

JSON structure:
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
      "ua_string": ["Chrome"]
    }
  ]
}
```

Common `ua_string` values: `["Chrome"]`, `["change_Firefox_to_Chrome"]`, `["add_Chrome"]`

### 2. JS Injection

When you need to shim a missing method or suppress alerts/warnings.

```json
{
  "interventions": [
    {
      "platforms": ["all"],
      "content_scripts": {
        "js": ["bug1234567-example.com-prevent-alert.js"]
      }
    }
  ]
}
```

### 3. CSS Injection
When you need to hide unsupported warnings or fix layout issues with css.

```json
{
  "interventions": [
    {
      "platforms": ["android"],
      "content_scripts": {
        "css": ["1234567_example.com-hide-unsupported-warning.css"]
      }
    }
  ]
}
```

## JS Injection Best Practices

### Guard Pattern (when needed)

Use a guard pattern when modifying global APIs or prototypes to prevent the code from running multiple times:

```js
if (!window.__firefoxWebCompatFixBug1234567) {
  Object.defineProperty(window, "__firefoxWebCompatFixBug1234567", {
    configurable: false,  // Can't be deleted or changed
    value: true,          // Mark as applied
  });

  // ... actual intervention code here (e.g., override setTimeout, addEventListener, etc.)
}
```

**When to use:**
- Overriding global APIs (setTimeout, addEventListener, etc.)
- Modifying prototypes that would conflict if wrapped multiple times
- Any code that would error or cause issues if run twice

**Naming convention:**
- Use `__firefoxWebCompatFix` prefix
- Add bug number: `Bug1234567` OR descriptive name: `Fastclick`
- Examples: `__firefoxWebCompatFixBug1756970`, `__firefoxWebCompatFixFastclick`

Many interventions are already implemented and do a range of interesting things. For instance, these are examples of what's possible:

* Re-writing a response header with a regexp:
```json
{
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
}
```

* Spoofing navigator.userAgentData:

```js
if (!navigator.userAgentData) {
  const ua = navigator.userAgent;
  const mobile = ua.includes("Mobile") || ua.includes("Tablet");

  // Very roughly matches Chromium's GetPlatformForUAMetadata()
  let platform = "Linux";
  if (mobile) {
    platform = "Android";
  } else if (navigator.platform.startsWith("Win")) {
    platform = "Windows";
  } else if (navigator.platform.startsWith("Mac")) {
    platform = "macOS";
  }

  const version = (ua.match(/Firefox\/([0-9.]+)/) || ["", "58.0"])[1];

  // These match Chrome's output as of version 126.
  const brands = [
    {
      brand: "Not/A)Brand",
      version: "8",
    },
    {
      brand: "Chromium",
      version,
    },
    {
      brand: "Google Chrome",
      version,
    },
  ];

  navigator.userAgentData = {
    brands,
    mobile,
    platform,
  };
}
```
* Spoofing window.chrome and navigator.vendor:

```js
if (!window.chrome) {
  window.chrome = {};

  const nav = Object.getPrototypeOf(navigator);
  const vendor = Object.getOwnPropertyDescriptor(nav, "vendor");
  vendor.get = () => "Google Inc.";
  Object.defineProperty(nav, "vendor", vendor);
}

```
* Running late window.load event listeners:

```js
if (!window.__firefoxWebCompatFixBug1756970) {
  Object.defineProperty(window, "__firefoxWebCompatFixBug1756970", {
    configurable: false,
    value: true,
  });

  const { prototype } = window.EventTarget;
  const { addEventListener } = prototype;
  prototype.addEventListener = function (type, b, c, d) {
    if (
      this !== window ||
      document?.readyState !== "complete" ||
      type?.toLowerCase() !== "load"
    ) {
      return addEventListener.call(this, type, b, c, d);
    }
    console.log("window.addEventListener(load) called too late, so calling", b);
    try {
      b?.call();
    } catch (e) {
      console.error(e);
    }
    return undefined;
  };
}
```
* Loading a particular script sync instead of async:

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

* Overriding a site's own JS property:

```
if (!window.$feprelist_C) {
  let value = undefined;

  function check(obj) {
    if (obj === null) {
      return false;
    }
    if (Array.isArray(obj)) {
      for (const v of obj) {
        if (check(v)) {
          return true;
        }
      }
      return false;
    }
    if (typeof obj === "object") {
      if ("barcodeEnabled" in obj) {
        obj.barcodeEnabled = true;
        return true;
      }
      for (const v of Object.values(obj)) {
        if (check(v)) {
          return true;
        }
      }
    }
    return false;
  }

  Object.defineProperty(window, "$feprelist_C", {
    configurable: true,

    get() {
      return value;
    },

    set(v) {
      try {
        if (check(v)) {
          console.info("enabled");
        }
      } catch (_) {}
      value = v;
    },
  });
}

```
* ignoring a specific setTimeout call

```
if (!window.__firefoxWebCompatFixBug1943898) {
  Object.defineProperty(window, "__firefoxWebCompatFixBug1943898", {
    configurable: false,
    value: true,
  });

  window.setTimeout = function (fn, time) {
    const text = "" + fn;
    if (
      text.includes("var el = document.getElementById('alwaysFetch');") &&
      text.includes("el.value = el.value ? location.reload() : true;")
    ) {
      document.getElementById("alwaysFetch").value = "";
    }
    return window.setTimeout(fn, time);
  };
}
```
* fixing a site with issues with the FastClick JS library (or one with legacy FastClick):

```js
if (!window.__firefoxWebCompatFixFastclick) {
  Object.defineProperty(window, "__firefoxWebCompatFixFastclick", {
    configurable: false,
    value: true,
  });

  const proto = (window.CSSStyleProperties ?? window.CSS2Properties).prototype;
  const descriptor = Object.getOwnPropertyDescriptor(proto, "touchAction");
  const { get } = descriptor;

  descriptor.get = function () {
    if (new Error().stack?.includes("notNeeded")) {
      return "none";
    }
    return get.call(this);
  };

  Object.defineProperty(proto, "touchAction", descriptor);
}
```


## Common "issue" Values
- `"firefox-blocked-completely"` - site completely blocks Firefox
- `"unsupported-warning"` - Firefox unsupported warning is displayed
- `"broken-interactive-elements"` - broken buttons, dropdowns, form inputs
- `"broken-layout"` - layout or content rendering issues
- `"desktop-layout-not-mobile"` - desktop layout shown on mobile instead of mobile-optimized layout
- `"page-fails-to-load"` - page fails to load or loads blank
- `"extra-scrollbars"` - duplicate scrollbars appear
- `"broken-scrolling"` - scrolling functionality broken
- `"broken-meetings"` - video conferencing/meeting functionality broken
- `"blocked-content"` - specific content blocked from displaying
- `"broken-videos"` - video playback broken
- `"broken-images"` - images fail to load or display incorrectly
- `"user-interface-frustration"` - UI elements cause frustration or poor UX
- `"broken-redirect"` - redirect functionality broken
- `"broken-printing"` - print functionality broken
- `"broken-login"` - login flow broken
- `"broken-audio"` - audio playback broken
- `"broken-zooming"` - zoom functionality broken
- `"broken-map"` - map functionality broken
- `"incorrect-viewport-dimensions"` - viewport size calculated incorrectly

Other less common values: `"slow-performance"`, `"frozen-tab"`, `"broken-font"`, `"broken-editor"`, `"broken-comments"`, `"broken-captcha"`

Check existing interventions in `browser/extensions/webcompat/data/interventions/` for more examples.

## URL Match Patterns

The `matches` field uses WebExtension match patterns:
- `"*://example.com/*"` - All protocols, exact domain
- `"https://example.com/*"` - HTTPS only
- `"*://*.example.com/*"` - All subdomains
- `"*://example.com/path/*"` - Specific path prefix

It is also possible to used `"exclude_matches"`  to exclude a specific subdomain while using a wildcard subdomain `"*://*.example.com/*"`.
