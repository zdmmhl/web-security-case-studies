# Stored XSS and an allowed script source

Retrospective English edition of the `report-v2` coursework writeup. The original target, request-bin identifiers, authentication material and flag are omitted. The target application is not included; this article is not a runnable reproduction.

## Application flow

The lab accepted a report at `/report`, redirected to `/report/<uuid>`, and displayed a **Report this report** button. That action requested a visit from an administrator browser. The interesting trust boundary was the administrator rendering content supplied by another user.

## Identifying the injection sink

I submitted basic HTML probes in the `name` and `content` fields and inspected the returned markup. Both fields appeared without HTML escaping in structures resembling:

```html
<h2 class="mdl-card__title-text">Report against {name}</h2>
<div id="report">{content}</div>
```

This established stored HTML injection. It did not, by itself, establish JavaScript execution: the page returned a restrictive script policy.

```http
Content-Security-Policy: script-src cdnjs.cloudflare.com; base-uri 'none'; object-src 'none'
```

The historical browser blocked inline scripts and inline event-handler probes such as `onerror`. I next checked whether an allowed external library could interpret the injected markup.

## Testing the allowed library path

The experiment loaded AngularJS from the allowed CDN. A small `ng-app` element containing `{{1+1}}` rendered `2`, showing that AngularJS bootstrapped over the injected DOM.

The next probes used an autofocus input, `ng-focus`, the event's composed path and AngularJS's `orderBy` expression evaluation. In the original notes, an AngularJS 1.5.11 probe reached an alert but attempts involving `document.cookie` encountered an `isecdom` restriction. The tested 1.6.9 variant permitted the cookie-reading expression in that lab browser.

These are historical observations tied to particular library versions and the challenge environment. They do not establish the same behavior in current browsers or in every AngularJS application.

## From execution to the lab result

The original workflow used an alert to check execution, then used `navigator.sendBeacon` to send the lab cookie to a temporary request bin. The notes record that POST collection was more reliable than the tested GET collection. Autofocus supplied the trigger when the administrator opened the report.

The chain was:

1. Persist unescaped HTML in the report content.
2. Load an allowed external library.
3. Have that library interpret user-controlled DOM attributes.
4. Trigger the expression when the administrator browser focused the input.
5. Collect the lab-only result through the temporary HTTPS request bin.

The original writeup records receiving the challenge flag. No cookie, flag, active collection address or administrator-browser evidence is published here. The source's redacted final payload lost parts of its URL syntax, so this edition preserves the reasoning without presenting it as executable code.

## Security interpretation

A broad script-host allowlist can admit libraries that turn injected HTML into active behavior. CSP provides defense in depth; it does not make an unescaped HTML sink safe. Escape plain text at the rendering boundary, sanitize HTML where required, and avoid trusting arbitrary executable libraries merely because their host is allowed.

Cookie access here concerned a cookie readable by script. It should not be generalized to `HttpOnly` cookies. The [CRLF and DOM-XSS case](crlf-and-dom-xss.md) illustrates a different disclosure path.

## Verification boundary

This article was edited from the historical writeup. No requests were sent to the old lab, no administrator visit was retriggered, and no fresh exploit result is claimed.
