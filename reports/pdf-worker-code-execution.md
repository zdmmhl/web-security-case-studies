# Historical report: Pdf Worker Code Execution

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## Job

### Summary

The Job application was found to be vulnerable to critical client-side code execution within its PDF rendering engine. The application allows users to upload PDF resumes, which are rendered by an administrator using an outdated version of `pdf.js` (v1.8.188). By exploiting **CVE-2018-5158**, an attacker can inject malicious JavaScript into the PDF's `PostScript` compiler. When the administrator views the crafted resume, the payload executes within the browser's worker thread. Because the worker can still issue same-origin requests, the attacker can trigger unauthorized actions such as self-approving a job application under the administrator's session.

### Exploitation

The exploitation process involved identifying the correct injection point and bypassing the constraints of the Web Worker environment.

#### 1. Exploration and Discovery

**Phase 1: Initial XSS Testing**

- Early attempts focused on standard HTML injection within the PDF's metadata and name objects, such as `/BaseFont`.

- Injecting tags like `<script>` or `<img onerror=...>` resulted in either silent failures or PDF parsing errors.

- A critical error was observed: `Warning: name token is longer than allowed by the spec: 199`, confirming a strict 127-byte limit on PDF Name objects that made complex JS payloads impossible.

**Phase 2: Version Fingerprinting and CVE Identification**

- Analysis of the `pdfjs.js` file and logs revealed the version as `1.8.188`.

- Research into this specific version led to **CVE-2018-5158**, which involves the `constructPostScriptFromIR` function in the worker thread.

- Source code auditing of `pdfjs.worker.js` (line 5383) confirmed that PostScript code is compiled via `new Function()`, and that the `/Domain` array from the PDF is concatenated into the function string without sanitization.

**Phase 3: Local Probing and Validation**

- To verify the execution, a "probe" PDF was generated with a harmless payload: `0)),fetch('/hit2018').catch(function(){});//`.

- The attacker hosted a local HTTP server and monitored logs while rendering the PDF in a local test environment.

- The receipt of a `GET /hit2018` request confirmed that `Domain[1]` was a stable injection point and that the JS was indeed executing.

#### 2. Technical Analysis of the Sink

The vulnerability stems from how the `visitArgument` function (line 5811) handles PostScript compilation:

```

// Vulnerable logic in pdfjs.worker.js

visitArgument: function (arg) {

  this.parts.push('Math.max(', arg.min, ', Math.min(', arg.max, ', src[srcOffset + ', arg.index, ']))');

}

```

*Listing 1: The vulnerable sink where the Domain values are used as arg.min/max*

By setting `arg.max` to `0)),payload;//`, the attacker breaks out of the `Math.min` call and executes their own logic.

#### 3. The Multi-Stage Exploit Chain

**Stage 1: Breaking the Worker Sandbox**

- The exploit runs in a Web Worker, meaning there is no `window` or `document` object to interact with the UI.

- The discovery that the `Approve` button on the main page simply performs a `fetch()` to `/approve` was the turning point.

- Since the Worker shares the same origin, it can still perform this `fetch()` with the administrator's credentials automatically included.

**Stage 2: Automation and Exfiltration**

- The final payload was designed to be "idempotent" using `self.__p` to prevent multiple execution attempts from crashing the worker.

- It performs a relative fetch to `/applications/LAB_USER`, regexes the application ID from the HTML, and then issues a `POST` to the `/approve` endpoint:

```

0)),self.__p||(self.__p=1,fetch('/applications/LAB_USER').then(function(r){return r.text()}).then(function(d){

    var m=/app-([0-9]+)/.exec(d);

    if(m) return fetch('/review/'+m[1]+'/approve',{method:'POST'});

}).catch(function(){}))//

```

*Listing 2: Final payload for automated internal approval*

The attack resulted in the disclosure of the flag after the bot's automated approval:

```

[REDACTED LAB FLAG]

```

*Listing 3: The retrieved flag confirming successful cross-environment exploitation*

### Remediation

- **Update Library:** Replace `pdf.js 1.8.188` with a modern version (v2.x or later) that does not use `new Function()` for PostScript compilation.

- **CSRF Protection:** Implement unique, unpredictable tokens for state-changing actions like `/approve` to prevent them from being triggered by unauthorized client-side code execution.

- **Content Security Policy (CSP):** Deploy a CSP that restricts `script-src` to exclude `unsafe-eval`, effectively disabling the `new Function()` sink utilized by this exploit.

### Impact and Risk

The impact is critical because an unauthenticated user can upload a malicious resume that causes privileged state-changing actions when reviewed by an administrator. In this challenge, that was enough to approve the attacker's own application using the reviewer's session.

The risk is high because the payload is stored in the uploaded file and executes automatically during a normal review workflow. The administrator does not need to click any extra element inside the page for the attack to succeed.

