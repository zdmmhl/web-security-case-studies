# Historical report: Mobile Desktop Xss Chain

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## NetQuocca3

### Summary

NetQuocca was found to be vulnerable to a sophisticated exploit chain involving cross-site scripting (XSS) and cross-origin resource sharing (CORS) misconfigurations across its dual-frontend architecture. The application serves different interfaces for desktop and mobile users, both of which contained distinct vulnerabilities in how they rendered transaction descriptions. By chaining a high-interaction XSS on the mobile platform with a stored XSS on the desktop platform, an attacker could bypass strict CORS policies protecting the `/flag` API endpoint. This allowed for the unauthorized retrieval of the administrator's flag, which was then exfiltrated using the application's own internal transaction functionality.

### Exploitation

The vulnerability stems from the inconsistent and insecure handling of user-supplied transaction descriptions across the `lab.example.invalid` (Desktop) and `lab.example.invalid` (Mobile) domains.

#### 1. Technical Analysis of Vulnerable Components

**Desktop Sanitizer Bypass:** The desktop site uses a custom `sanitize()` function to process HTML descriptions. However, the logic is flawed: by prefixing a payload with an unknown or empty HTML tag, subsequent dangerous tags can bypass the filter. While this allows for stored XSS, it requires the victim to manually open a specific transaction modal, making it a passive threat on its own.

**Mobile Markdown Injection:** The mobile frontend includes a critical vulnerability in its React-based Markdown renderer. When processing image tags, the application decodes the `src` attribute and injects it directly into the `innerHTML` of a span element:

JavaScript

```

useEffect(() => {

  if(ref.current) {

    ref.current.innerHTML = `<img src="${decodeURIComponent(src ?? "")}" title="${title}">${children ?? ""}</img>`

  }

}, [ref.current])

```

*Listing 1: Decompiled mobile bundle showing the unsafe innerHTML sink*

#### 2. The Multi-Stage Exploit Chain

A direct attack on the `/flag` endpoint from the mobile domain fails due to CORS restrictions; the API does not allow the `m.` subdomain to access sensitive resources. To overcome this, the exploit was split into two stages.

**Stage 1: The Mobile Redirector (Transaction ID: 4831)** An attacker creates a transaction with a malicious Markdown image tag. When the administrator views this transaction on a mobile device, the `onerror` event triggers an automatic redirection to the desktop site:

```

[Original screenshot omitted.]

```

*Listing 2: Payload used to force a cross-domain jump to the desktop interface*

This payload uses the `?desktop` parameter to prevent the server from redirecting the user back to the mobile site, while the `&transaction=4830` parameter ensures the second stage of the attack is loaded immediately.

**Stage 2: Desktop Flag Retrieval and Exfiltration (Transaction ID: 4830)** Once redirected to the desktop site, the administrator's browser executes the second payload. This script fetches the flag from the API—now within the permitted same-origin context—and uses the page's existing transfer form to send a new transaction back to the attacker containing the flag:

```

<x></x><img src=x onerror="fetch(BASE_URL+'/flag').then(r=>r.json()).then(j=>(f=toAccountBsb.form,f[1].value='000-000',f[2].value='LAB_ACCOUNT',f[3].value=j.msg,f[4].value=.1,f.requestSubmit()))">

```

*Listing 3: Final payload for automated flag exfiltration via internal API calls*

This exfiltration method does not require an external webhook. Instead, it reuses the application's own transfer workflow to send the flag back to the attacker.

The attack successfully resulted in the disclosure of the flag:

```

[REDACTED LAB FLAG]

```

[Original screenshot omitted.]

*Listing 4: The retrieved flag confirming successful cross-frontend exploitation*

### Remediation

The primary defense must be the implementation of robust, context-aware encoding and sanitization. Custom sanitizers should be replaced with industry-standard libraries such as DOMPurify to prevent bypasses like the one observed in the desktop frontend.

For the mobile application, the use of `innerHTML` with decoded user input must be eliminated. Developers should use safe DOM APIs or React's built-in property binding which automatically handles character encoding.

Furthermore, the CORS policy should be strictly audited. While the mobile subdomain was correctly restricted from reading the flag, the presence of the XSS vulnerability rendered this protection moot. Implementing a strict Content Security Policy (CSP) that prohibits `unsafe-inline` scripts and limits `connect-src` would have prevented the exploit from either executing or exfiltrating data to the attacker's account.

### Impact and Risk

The impact of this vulnerability is critical. By chaining the mobile and desktop issues together, an attacker can reach the protected `/flag` endpoint and exfiltrate the returned data through the application's own functionality.

The risk is increased by the fact that the exploit depends on behaviour across two related frontends rather than a single page. Once the malicious transactions are stored, the attack becomes reliable when viewed by the administrator in the required sequence.
