# Additional Historical Web Security Cases

These edited English notes preserve additional locally saved COMP6443 writeups. The assessment includes team work; exclusive individual authorship is not inferred. Outcomes are historical claims, not fresh reproductions. Lab identities, credentials, flags, personal records, collector addresses, and screenshots are omitted.

## Handlebars template injection

The saved writeup describes reaching JavaScript's Function constructor through template-visible objects and then invoking Node.js process functionality to read a lab file. The recorded output is consistent with server-side code execution in that historical environment. Exact engine version, prototype-access settings, and server source are absent, so this is not a claim about all Handlebars deployments.

The boundary failed when submitted template syntax was executed with access to powerful runtime objects. Treat user text as data; constrain template capabilities and isolate the rendering process.

## Archive extraction traversal

A crafted tar member contained parent-directory segments and targeted a password file outside the intended extraction directory. The writeup describes constructing the archive and concludes that the password value was overwritten, but does not preserve a complete request/response trace of the successful extraction.

This illustrates an unsafe archive-extraction boundary. A public reconstruction should validate resolved paths, account for links and archive-entry types, and test that all output stays inside the chosen destination.

## Identity normalization and stored template execution

The saved Secret 1/2 narrative connects two flaws. Registration accepted a trailing-space variant of a reserved administrator name; subsequent behavior reportedly treated it as the administrator identity. The exact normalization mechanism is uncertain: application trimming and database comparison behavior are hypotheses, not verified server code.

The second stage saved template syntax in a normal user's secret field, then triggered rendering through an administrator lookup. Recorded directory and file output support the historical template-execution claim. Uniform identity handling, immutable account identifiers and explicit roles address the first boundary; rendering secrets as escaped text addresses the second.

## LLM-mediated account selection

The Teller 1 notes describe asking a banking-themed lab assistant to switch its target account to the administrator username. The recorded response exposed a protected account record. Transfer and support capabilities described by the model were not established as real backend operations, and the notes do not demonstrate an actual transfer or full administrator session.

The case illustrates how natural-language intent can influence tool parameters. Server authorization must bind every lookup to the authenticated caller rather than delegate identity or ownership decisions to generated text. The backend mechanism remains an inference because tool traces and implementation are unavailable.

## Blind stored XSS

A feedback field was reportedly rendered in an administrator bot's page, and an out-of-band request carried a lab cookie. This supports historical browser execution in a hidden review context; the request would originate from the bot's browser, not necessarily the application server.

Initial equal responses and unsuccessful delay probes did not demonstrate SQL injection, but also do not prove that every SQL injection path was absent. The case concerns unsafe rendering of feedback; fresh verification would need the review-page response and browser trace.

## JSONP callback and reflected HTML injection

The Clients 2 writeup records a controllable JSONP callback combined with an unescaped search query used to insert a same-origin script element. A filter rejected literal dots, while a runtime character construction avoided that syntactic restriction. The historical callback then executed in the reviewing browser.

JSONP is executable JavaScript, not passive JSON. Output encoding, a restricted callback identifier if legacy JSONP remains necessary, and removal of executable user-controlled responses are separate boundaries. Cookie access applies only to cookies exposed to JavaScript.

## Uploaded active content and review paths

The Profile 1/2 notes distinguish an SVG embedded as an image from the same SVG opened as an active document. The first historical chain changed an administrator review path to the uploaded document. The second blocked inline script but reportedly accepted uploaded HTML and a same-origin external script allowed by its CSP.

Success depends on response MIME types, upload handling, CSP, origin and browser context. Same-origin uploaded active documents were the relevant trust boundary; the result does not imply that any SVG image executes scripts or that all CSP policies are bypassable.

## A frontend redirect sink

The XSS Playground 4 writeup identifies a query parameter passed to a client-side location replacement function. A historical alert probe and later collector request are reported; a separate file-path probe returned a 404 page.

The saved result concerns acceptance of a script URL in that historical browser and policy context. A reconstruction should permit intended navigation destinations and schemes explicitly, and record the browser/CSP conditions. The notes do not establish that every modern browser executes the same URL.

## ClosedLearning: preserved boundary and uncertainty

The local ClosedLearning notes describe investigating a learning portal's upload and review behavior. They are retained locally as a separate source rather than silently merged into the Profile case. No new success claim is inferred here; it needs its own evidence-led reconstruction before publication as a completed case.

## Source coverage

This edition covers the local Handlebars, Orders, Secret 1/2, Teller 1, Blind XSS, Clients 2, Profile 1/2, and XSS Playground 4 writeups. Closely related variants and assessment-wide compilations remain local for provenance. See the [report index](../REPORTS.md) and [review notes](../docs/report-review-notes.md).
