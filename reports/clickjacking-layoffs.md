# Historical report: Clickjacking Layoffs

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# Layoffs Writeup

## Challenge Overview

The goal of this challenge was to trick an administrator bot into performing a privileged action—firing an employee—via a Clickjacking (UI Redressing) attack.  

The application exposed two key areas:

1. A **public forum** where users could leave comments.

2. A **restricted admin panel** (`/admin`), accessible only to the administrator, which displayed a grid of employee cards with corresponding "fire me" buttons.

At first glance, the application seemed secure since normal users could not access the `/admin` panel. However, further testing showed that it lacked basic frame-busting protections.

---

## Initial Analysis

Testing the forum comment submission revealed that raw HTML tags were rendered directly on the page without sanitization. This meant I could inject an `<iframe>` into my posts.

Checking the HTTP response headers of the `/admin` endpoint showed that it was entirely missing standard anti-framing headers:

- No `X-Frame-Options: DENY` (or `SAMEORIGIN`)

- No `Content-Security-Policy: frame-ancestors 'none'`

Because the `/admin` page could be framed, the objective became clear: embed the `/admin` page as an invisible iframe over my comment, and align one of the "fire me" buttons exactly over a benign element that the admin bot was programmed to click (in this case, the `reply` link on my comment).

---

## Failed Approaches

Several ideas were tested and discarded before reaching the intended solution.

### 1. The "Red Box" (Nested Clipping) Method

My initial strategy was to create a highly targeted Clickjacking payload using a small, tightly constrained container (`overflow: hidden`) with a red border. The idea was to "clip" out only the "fire me" button from the iframe and position it precisely over the `reply` link.

The payload looked something like this:

```html

<style>

  #wrap { position: relative; }

  #win { position: absolute; left: 292px; top: -37px; width: 30px; height: 16px; overflow: hidden; }

  #frm { position: absolute; left: -452px; top: -293px; }

</style>

<div id="wrap"><div id="win"><iframe id="frm" src="/admin"></iframe></div></div>

```

**Why it failed:** This approach introduced a severe CSS Coordinate System shift. By wrapping the iframe in a `position: relative` container, the origin `(0,0)` was forced to the exact point where the text injection began. Calculating the precise negative offset required to drag the iframe's button into that small clipping window proved to be incredibly difficult and error-prone.

------

## Key Insight

The key weakness was that the browser naturally handles overlay positioning if you keep the coordinate system simple.

Instead of fighting with nested containers and relative coordinate shifts, the most robust approach was to use a **macroscopic, full-frame overlay**.

By injecting a single `<iframe>` with `position: absolute` directly into the post, the iframe floated freely. By temporarily setting `opacity: 0.5`, I could visually overlay the target `/admin` page on top of the forum thread.

This turned the problem from "mathematical clipping" into a simple visual "nudging" exercise using the browser's DevTools.

------

## Exploitation Strategy

I injected the iframe and used the browser's DevTools (Elements panel) to adjust the `top` and `left` CSS properties using the arrow keys.

1. I started with a rough guess: `left: -160px; top: -330px;`. This aligned the button for the user `hamish`.

2. To target a different user (`melon`), who was located further to the left in the admin panel grid, I needed to shift the iframe to the right.

3. After fine-tuning in the DevTools, the exact coordinates to align `melon`'s "fire me" button over the `reply` link were found.

4. I changed the opacity to `0.001` to make the iframe completely invisible while still intercepting clicks.

The final payload was:

HTML

```

<iframe

  src="/admin"

  style="

    position:absolute;

    top:-310px;

    left:80px;

    width:800px;

    height:600px;

    opacity:0.001;

    z-index:999;

  ">

</iframe>

```

------

## Why the Attack Worked

The application assumed that hiding the `/admin` panel behind session authentication was sufficient protection.

However, the implementation failed because web browsers allow cross-origin framing by default unless explicitly told not to.

As a result:

- the attacker could frame the authenticated `/admin` page

- the bot would willingly visit the attacker's forum post

- the CSS `opacity: 0.001` rendered the iframe invisible to the bot/user, but the browser still routed pointer events (clicks) to the top-most layer (the iframe)

- when the bot attempted to click the benign `reply` button, the click was hijacked by the `fire me` button hovering directly above it.

Once the collision occurred, the application accepted the authenticated POST request and executed the firing action.

------

## Result

After submitting the Clickjacking payload to the forum and waiting for the admin bot to click the `reply` button on the post, the action succeeded and the challenge displayed the flag in the forum thread:

```

[REDACTED LAB FLAG]

```

The employee was fired, confirming that the UI redressing had succeeded.

------

## Security Impact

This challenge demonstrates that Clickjacking is effective if the target page is:

- missing framing protections

- executing state-changing actions based on simple clicks

- relying solely on session cookies without secondary confirmations (like re-authentication or unpredictable CSRF tokens that can't be auto-submitted via clicks)

Authentication alone does not protect against UI redressing, as the victim's browser automatically includes their session cookies when the iframe loads.

------

## Remediation

A secure implementation should:

1. Send the `Content-Security-Policy: frame-ancestors 'none'` HTTP response header on all sensitive pages to prevent framing entirely.

2. Send the legacy `X-Frame-Options: DENY` header for backwards compatibility with older browsers.

3. Use `SameSite=Lax` or `SameSite=Strict` attributes on session cookies to prevent them from being sent in third-party iframe contexts.

4. Require secondary confirmation (e.g., a modal prompt or re-entering a password) for highly destructive actions.

------

## Final Notes

The important lesson from this challenge is that **a visual disguise can turn an authorized user into a confused deputy.** If framing is allowed, an attacker can manipulate the visual context of the application, forcing victims to perform actions they never intended to execute.