# Historical report: Clients Regex Filter

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# Clients 1 Writeup

## Challenge Overview

Clients 1 was a stored XSS challenge. User controlled input supplied to the client creation feature was later rendered back into the application’s HTML. The vulnerable field was displayed inside table markup, which meant that attacker controlled content was parsed in a full `text/html` context rather than being safely treated as text. Browsers are required to parse `text/html` using the HTML parser’s tokenization and tree construction rules, and the standard explicitly documents parser error handling for malformed markup, including strange cases involving tables and broken tags. 

The practical consequence was that once attacker controlled input escaped the original table cell context, the browser would still attempt to build a DOM from malformed HTML. This made the challenge fundamentally different from a simple reflected string injection bug.

## Initial Analysis

At first, straightforward payloads such as normal `<script>`, `<img>`, `<svg>`, and `<a>` tags did not survive unchanged. Instead, the application returned distorted fragments such as `lert(1)`, `rc=x ...`, `nmouseover=...`, or `mg src=...`. This strongly suggested that the backend was not performing proper output encoding, but was instead trying to destroy suspicious HTML with a regex style filter.

That distinction matters. Proper output encoding would have converted attacker controlled `<` and `>` into HTML entities, making them harmless text. Here, by contrast, the application left behind broken HTML fragments, which is typical of destructive string based filtering rather than safe rendering.

## Reverse Engineering the Filter

The best fit model for the backend filter was:

```

input.replace(/<[a-zA-Z][^>]*>[^a-zA-Z]*[a-zA-Z]/g, "")

```

This pattern has four important properties.

First, it only starts matching at a `<` that is immediately followed by an ASCII letter.

 Second, it then consumes everything up to the next `>`.

 Third, after that `>`, it consumes any number of non letters.

 Fourth, it finally consumes the first ASCII letter that appears after those non letters.

This explains the observed damage pattern very well. The filter does not simply remove a whole tag. Instead, it removes from the start of an opening tag through the next `>` and then continues far enough to delete the first letter that follows. Because `[^a-zA-Z]*` can include `<`, `>`, `/`, quotes, whitespace, and other punctuation, the match can cross tag boundaries and eat the first letter of the next tag or the first letter of following text.

For example:

<script>alert(1)</script> can match as <script>a, leaving lert(1)</script>.

<img src=x alt=TEST> can match as <img s, leaving rc=x alt=TEST.

<a onmouseover=1> can match as <a o, leaving nmouseover=1>.

This was the key insight. The application was not sanitising HTML. It was deleting a regex match that often extended beyond the intended tag and into neighbouring content.

A subtle but important point is that the `/g` flag means a global replacement across all non overlapping matches in a pass. It does not, by itself, prove that the developer wrote an outer `while` loop to repeat the filtering until the string stabilised. In practice, however, a single global replace is already enough to explain the multiple deletions seen during testing.

## Why the Naive Payloads Failed

Many payloads failed because the first opening tag was hit directly by the filter. Once that happened, the opening tag was destroyed before the browser ever saw a usable element.

This is why normal OWASP style probes such as malformed `<a>` or `<img>` tests did not immediately work here, even though those ideas are valid in general. The OWASP XSS Filter Evasion Cheat Sheet includes examples based on malformed tags, relaxed browser parsing, extraneous angle brackets, and half open markup. In particular, the “Malformed A Tags”, “Malformed IMG Tags”, “Extraneous Open Brackets”, and “Half Open HTML/JavaScript XSS Vector” sections all rely on the fact that browsers tolerate broken HTML and continue parsing it. 

In this challenge, however, the backend filter often destroyed the opening token so aggressively that the browser never received a usable `<script>`, `<img>`, or `<a>` opener in the first place. Therefore, a direct payload was usually reduced to inert residue.

## Why the Successful Bypass Worked

The successful bypass used a decoy structure:

```

<<fred><sscript>alert(1)</script>

```

This worked because the first literal `<` was not followed by a letter, so it did not begin a regex match. The filter instead began matching at the second `<`, namely `<fred>`. From there, the regex consumed:

```

<fred><s

```

Removing that substring left:

```

<script>alert(1)</script>

```

In other words, the extra leading `<` survived, while the decoy tag and the first `s` of `sscript` were removed. After replacement, those surviving pieces joined together into a valid `<script>` start tag.

This is exactly why the bypass felt so specific. It was not random obfuscation. It was carefully aligned against the filter’s matching rule. The decoy tag absorbed the destructive part of the regex, and the surviving characters reassembled into executable markup.

Conceptually, this is closest to the OWASP “Extraneous Open Brackets” idea. OWASP notes that extra opening brackets can defeat simplistic detection engines that reason about angle brackets in a crude way rather than performing proper context aware parsing. 

## Why Browser Parsing Still Matters

The bypass did not rely only on the regex. It also relied on the browser continuing to parse malformed HTML after the backend had mangled it.

The HTML standard explicitly defines an HTML parser, a tree construction stage, and separate handling for parser errors and strange cases such as misnested tags, unexpected markup in tables, and unclosed formatting elements. That means malformed output does not simply fail closed. It is often repaired or at least interpreted consistently enough to produce a usable DOM. 

That behaviour was particularly relevant here because the injection point sat inside table markup. The challenge therefore combined two weaknesses: a regex based destructive filter on the server, and tolerant HTML parsing in the browser.

## Exploitation Summary

The full exploitation path was:

1. Identify that user supplied HTML was stored and later rendered inside the clients table.

2. Confirm that the backend used destructive tag filtering rather than safe output encoding.

3. Reverse engineer the filter and model it as a regex that deletes from an opening tag through the first subsequent ASCII letter after the closing `>`.

4. Build a payload where a sacrificial malformed tag is matched and removed, while the surviving characters reassemble into a valid script opener.

5. Execute JavaScript in the privileged viewing context and use that execution to complete the challenge objective.

The important lesson is that the payload was not simply “a weird script tag”. It was a string engineered to survive a very specific regex transformation.

## Root Cause

The root cause was unsafe output handling. The application attempted to neutralise HTML using regex based deletion instead of context appropriate output encoding or a trusted sanitisation library.

This is fundamentally brittle for three reasons.

First, regular expressions do not understand HTML parsing rules.

 Second, destructive deletion can accidentally create new dangerous strings after transformation.

 Third, browsers still parse malformed HTML according to well defined recovery rules.

The challenge was therefore a textbook example of why ad hoc XSS filtering is unreliable.

## Remediation

The correct defence would have been to HTML encode untrusted data before rendering it into the page. If rich HTML input was genuinely required, the application should have used a mature allowlist based HTML sanitiser rather than deleting substrings with regexes.

In addition, sensitive cookies should be marked `HttpOnly` so that successful script execution cannot directly read them. That would not fix the XSS itself, but it would reduce the impact of the bug.

## Final Conclusion

Clients 1 was not solved by finding a rare HTML element. It was solved by reverse engineering a broken regex filter and then designing input that transformed into a valid script tag after filtering.

The key technical observation was that the backend did not escape attacker controlled HTML. Instead, it applied a global regex deletion of the form:

```

/<[a-zA-Z][^>]*>[^a-zA-Z]*[a-zA-Z]/g

```

Because this pattern can cross tag boundaries and delete the first letter of the next tag or text fragment, it was possible to introduce sacrificial markup and cause the surviving characters to reassemble into executable HTML. The browser then did the rest by parsing the damaged markup into a DOM according to normal HTML parsing and error recovery rules.