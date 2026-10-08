# Historical report: Gone Fishing Ssrf

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## Gone Fishing

### Summary

The Gone Fishing challenge exposed a webhook feature that could be abused to send server-side requests to internal services. By controlling both the target URL and the HTTP method, an attacker could first direct the server to a local Redis instance and then manipulate the request format so that Redis interpreted part of the HTTP request as a valid command. This allowed the attacker to retrieve the value of the `flag` key from Redis. The vulnerability is therefore not just a simple SSRF issue, but a chain involving SSRF, input validation failure, CRLF injection, and cross-protocol command execution.

### Exploitation

The attack began with reconnaissance. When inspecting responses in Burp Suite, the application returned the header:

[Original screenshot omitted.]

Listing 1: Response header disclosing the internal Redis host

> This response header disclosed the internal host address used by the backend service. Since Redis commonly listens on port `6379`, it suggested that the webhook functionality might be abused to reach a local Redis instance.

This was an important clue because it narrowed the likely target from “some internal service” to a probable Redis server running on `127.0.0.1:6379`.

The next step was to inspect the webhook creation request. Burp Suite showed that both the `site` parameter and the `method` parameter were user-controlled.

```

POST /create HTTP/2

Host: lab.example.invalid

Content-Type: application/x-www-form-urlencoded

site=http://127.0.0.1:6379&method=GET+flag%0d%0a

```

Listing 2: Malicious webhook creation request targeting the local Redis service

> This request demonstrates two important weaknesses. First, the server accepts an arbitrary destination through the `site` parameter, creating an SSRF primitive. Second, the `method` field is not restricted to normal HTTP verbs and even allows CRLF characters, making request manipulation possible.

By setting `site` to `http://127.0.0.1:6379`, the webhook feature caused the application server to connect to a local service that external users would not normally be able to access directly. This is a standard SSRF condition. However, SSRF alone was not enough to retrieve the flag. The second issue was that the application did not validate the `method` field properly.

Normally, an HTTP method should be something like `GET` or `POST`. In this challenge, the method could be changed to:

```

GET flag\r\n

```

If the server constructed the outbound request in a form similar to:

```

<METHOD> / HTTP/1.1

Header: value

...

```

then the actual first line sent to `127.0.0.1:6379` became:

```

GET flag

 / HTTP/1.1

...

```

Redis does not understand HTTP, but it does support simple inline commands. As a result, it interpreted the first line, `GET flag`, as a valid Redis command. The remaining HTTP request data was treated as invalid extra input.

[Original screenshot omitted.]

```

$29

[REDACTED LAB FLAG]

-ERR unknown command '/', with args beginning with: 'HTTP/1.1' 

-ERR unknown command 'X-Mtls-Username:', with args beginning with: 'LAB_USER' 

-ERR unknown command 'Accept-Language:', with args beginning with: 'en-US,en;q=0.5' 

-ERR unknown command 'Referer:', with args beginning with: 'https://waugh.zip' 

-ERR unknown command 'Origin:', with args beginning with: 'https://featherbear.cc' 

-ERR unknown command 'Content-Type:', with args beginning with: 'application/x-www-form-urlencoded' 

-ERR unknown command 'Content-Length:', with args beginning with: '0' 

-ERR unknown command 'Connection:', with args beginning with: 'keep-alive' 

```

Listing 3: Redis response showing successful retrieval of the `flag` value

> The response shows that Redis successfully executed the first injected command, `GET flag`, and returned the flag value. The later `-ERR unknown command` lines appear because the rest of the HTTP request was not valid Redis syntax, but they do not affect the success of the first command.

This explains why the response first returned the flag and only then showed multiple Redis errors. Even though most of the HTTP request was meaningless to Redis, the first injected line was enough to retrieve the sensitive value.

Overall, the exploit chain combined several weaknesses: the webhook feature allowed requests to arbitrary destinations, local addresses were reachable, the `method` parameter was fully user-controlled, CRLF characters were not filtered, and the application treated all targets as if they were normal HTTP services. Together, these flaws turned a webhook request into an arbitrary command sent to a local Redis instance.

### Remediation

The most important fix is to restrict the webhook destination. The application should not allow requests to `127.0.0.1`, `localhost`, private address ranges, link-local addresses, metadata services, or other sensitive internal targets. This validation should be performed both before and after DNS resolution to prevent bypasses such as DNS rebinding.

The `method` parameter should also be strictly validated. Only a fixed whitelist of legitimate HTTP methods, such as `GET`, `POST`, `PUT`, and `DELETE`, should be accepted. Arbitrary strings must never be allowed in this field.

In addition, the application should reject carriage return and newline characters in all request components that are influenced by user input, especially the method, URL, and headers. This would prevent request injection and CRLF-based protocol manipulation.

The service should also avoid manually constructing raw HTTP requests. Instead, it should use a well-tested HTTP client library that enforces valid request structure automatically. This makes it much harder for attacker-controlled input to alter the low-level request format.

As a defence-in-depth measure, the webhook system should only be allowed to access approved protocols and approved destination ports. If the feature is only intended for HTTP and HTTPS webhooks, then arbitrary ports such as `6379` should never be reachable.

### Impact and Risk

The impact of this vulnerability is high because it allows more than simple network probing. By abusing the webhook service, an attacker can reach internal services that are not meant to be exposed, and in this case can directly retrieve sensitive data from Redis. In a real system, this kind of access could expose application secrets, session data, cached user information, or other backend data. A well-known example is the 2019 Capital One breach, where abuse of a server-side request flaw contributed to the theft of a very large amount of sensitive customer data and later led to major regulatory penalties (U.S. Department of Justice 2022; U.S. Securities and Exchange Commission 2020).

The likelihood of exploitation is also high. The attack does not require administrator access or unusual conditions. It mainly relies on a webhook feature that accepts attacker-controlled destinations and request fields, so once the attacker identifies the internal service and understands how the outbound request is constructed, the exploit can be reproduced in a practical way.

Overall, this is a serious vulnerability because it turns a webhook feature into a way of sending attacker-controlled commands to an internal service and retrieving sensitive data from it.

### References

U.S. Department of Justice 2022, *Former Seattle Tech Worker Convicted of Stealing Personal Information from More Than 100 Million People in Capital One Data Breach*, viewed 25 March 2026. 

U.S. Securities and Exchange Commission 2020, *Capital One Financial Corporation to Pay $80 Million Penalty for Data Breach that Affected More Than 100 Million Individuals*, viewed 25 March 2026. 