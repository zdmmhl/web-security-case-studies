# Web Security Case Studies

Two historical lab writeups examining stored XSS, browser-side execution constraints, and a CRLF/DOM-XSS vulnerability chain.

## Cases

- [Stored XSS and allowed script sources](cases/stored-xss-and-csp.md)
- [CRLF response splitting and DOM XSS](cases/crlf-and-dom-xss.md)
- [Impact and remediation notes](cases/remediation.md)

The external script-hosting site is described in [the existing XSS lab repository](https://github.com/zdmmhl/zdmmhl.github.io). It hosts browser-side content; the historical webhook was the collection endpoint.

Credentials, certificate containers, tokens, flags and screenshots are not bundled. Original lab URLs are replaced with placeholders. No request has been sent to the old services during preparation.


## Provenance

Originated in UNSW COMP6443. The larger assessment report is team work; these selected writeups require final confirmation of individual case attribution.

## Verification status

These are retrospective notes, not freshly reproduced experiments. Targets, original evidence and supplied course materials are not bundled. No current-service behavior or new experimental result is claimed.
