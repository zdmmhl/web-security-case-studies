# CRLF response splitting and stored DOM XSS

Retrospective English edition of the `report` coursework writeup. Target URLs, certificate details, personal names, infrastructure addresses and challenge flags are omitted. The application and its evidence are not bundled; this is a historical analysis rather than a runnable reproduction.

## Application and access model

The lab required a client certificate. A connection without it produced a TLS certificate-required alert. After authorized access, `/report` accepted a report and stored its UUID in a Flask session; `/view/<uuid>` displayed the report. Decoding the observable session payload did not mean that its signature could be forged.

The report view returned a lab `flag` cookie with `HttpOnly` and `Secure`. JavaScript could not obtain that cookie through `document.cookie`. The eventual disclosure combined two separate defects.

## The stored DOM-XSS path

The view decoded report content from Base64 and sanitized elements using a short blacklist. The source snippet removed `script` and `object` elements, stripped attributes matching `on*`, and checked `src` values for `javascript:`.

The blacklist did not cover an iframe's `srcdoc` attribute. HTML inside `srcdoc` is parsed as a nested document; checking only the outer element's tag and selected attributes did not sanitize that nested content. The historical probe demonstrated script execution through this path.

No CSP header or CSP meta element was observed in the original response inspection. The challenge centered on incomplete sanitization and the remaining cookie disclosure barrier, rather than the CDN-based CSP chain in the companion case.

## The response-header injection path

The server inserted the user-controlled `name` into a response header resembling:

```http
X-Meta: target=demo-target; time=...
```

A single CRLF probe introduced an extra response header. A double-CRLF probe terminated the apparent header section early. In the original response, later server-generated headers appeared in the browser-visible body:

```http
HTTP/1.1 200 OK
X-Meta: target=demo-target
X-Test: yes

<body>demo</body>; time=...
Set-Cookie: flag=[REDACTED]; Path=/; HttpOnly; Secure
...
```

This depended on the lab server's response construction and header ordering. It does not establish that a conforming modern framework will accept CRLF in header values.

## Combining the defects

The historical chain used response splitting to expose the `Set-Cookie` value as body text. The DOM-XSS path then read that text and extracted the lab value.

1. Submit a `name` containing the response-splitting sequence.
2. Submit `content` using the unsanitized nested-document path.
3. Confirm that the report body contains the cookie-header text.
4. Have the challenge browser visit the report.
5. Read and decode the exposed body value and collect the lab result through a temporary HTTPS webhook.

The script read a string in the document body. It did not defeat the browser's `HttpOnly` cookie API restriction. The server had disclosed the same value through another channel.

## Collection and historical outcome

The notes describe unsuccessful or inconsistent HTTP collection probes followed by an HTTPS endpoint. Mixed-content handling was a possible explanation for the earlier behavior, not a conclusively isolated cause.

The original writeup records an administrator-browser callback containing the decoded challenge flag. The webhook, source IP, certificate identifier, personal name, raw encoded flag and decoded flag are omitted. No new callback was collected during publication.

## Remediation

- Reject CR and LF in user-derived response-header values and use framework APIs that enforce this boundary.
- Render report text as text. If HTML is required, use a maintained sanitizer with explicit handling of nested documents.
- Preserve `HttpOnly` and `Secure`, while ensuring sensitive cookie values are never rendered into response bodies.
- Treat a review bot's access to untrusted reports as a separate trust boundary and restrict its privileges.

## Verification boundary

The historical reasoning was reviewed and edited. The target application, raw response captures and administrator browser are not available in this repository. The chain has not been freshly reproduced.
