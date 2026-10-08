# Review Notes for Historical Web Reports

The archival reports preserve historical reasoning, including uncertain conclusions. They are not a fresh audit of any current service. Lab flags, identities, collection endpoints, and screenshots are removed; snippets are illustrative.

- **CSP Fun:** The original notes infer COOP or sandboxing from a browser error. The original response headers and browser trace are not bundled, so this diagnosis is unverified. An about:blank helper does not universally defeat COOP; opener access depends on origin and browsing-context-group behavior. Do not interpret the reported result as a general COOP bypass. The embedded CSP and nested script have formatting damage.
- **Engineering:** Missing base-uri can permit a base element to change resolution of a nonce-authorized relative script. This is a document-injection chain, not a cryptographic nonce break. The original notes alternate between a directory base and a file base; both examples must be interpreted with URL resolution rules and script placement. See [CSP base-uri](https://www.w3.org/TR/CSP/#directive-base-uri).
- **Job:** Code running in a worker and issuing same-origin requests has not necessarily escaped a worker sandbox. CSRF tokens do not reliably stop same-origin XSS. Do not use the original “v2.x or later” wording as a current dependency recommendation; use a maintained release and current vendor guidance.
- **Layoffs:** The payload frames a same-origin administrative page. SameSite cookies address cross-site contexts and do not stop this same-site case. A genuine click on a framed form may submit its legitimate CSRF token; CSRF protection alone does not prevent clickjacking.
- **NetQuocca:** Moving execution to an origin permitted by the API exploits XSS in a trusted frontend. It is not proof that the API's CORS implementation was itself defective. The displayed mobile/desktop hostnames and account numbers are placeholders.
- **Report:** HttpOnly is not directly bypassed through document.cookie. The reported chain exposes header-like secret text in the body, then reads that text. mTLS does not sanitize content or prevent script execution in an authenticated browser.
- **Clients:** The regex is a best-fit hypothesis from observed transformations, not verified server source.
- **GCC and Gone Fishing:** Arbitrary file access and cross-protocol requests depend on the historical compiler, filesystem, Redis configuration, and request construction. References to unrelated incidents are context, not evidence for these labs.
- **All reports:** Severity and likelihood language in the source was written for challenge conditions. It is not a CVSS assessment or a claim about production exposure.

[Scripting restrictions in CSP](https://www.w3.org/TR/CSP/) should be interpreted alongside output encoding, sanitization, origin boundaries, and the actual browser behavior.
