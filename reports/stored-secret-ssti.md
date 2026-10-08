# Historical report: Stored Secret Ssti

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## Secrets 2

### Summary

Secrets 2 exposed a server-side template injection (SSTI) vulnerability in an administrator-only feature used to inspect users’ stored `secret` values. Instead of displaying those values as plain text, the application rendered them as Jinja template content. Because a normal user could control their own `secret`, an attacker could store a malicious template payload and then use the admin view to trigger server-side execution. This allowed access to Python runtime objects, directory enumeration, and ultimately arbitrary file read of `/app/flag`.

### Exploitation

The vulnerable behaviour appeared in the administrator function for viewing another user’s `secret`. A normal user-controlled value was not treated as inert data. Instead, it was evaluated as template code when shown in the admin interface.

The attack first used a reconnaissance payload to confirm that SSTI was working and to identify the file layout on the server:

```

{% for x in lipsum.__globals__['os'].listdir('/app') %}{{x}} {% endfor %}

```

Listing 1: Reconnaissance payload used to enumerate the `/app` directory

> This payload uses `lipsum.__globals__` to reach the Python `os` module and call `os.listdir('/app')`. Its purpose is to confirm that the template engine is executing attacker-controlled expressions on the server.

When this payload was rendered through the administrator view, it returned directory contents including:

```

app.py static templates db.py db-redacted.py helpers.py flag views.py

```

Listing 2: Output showing successful directory enumeration

> This output confirms two important facts: first, SSTI execution was successful; second, the application flag file was located at `/app/flag`.

After the target path had been identified, the `secret` value was replaced with a file-read payload:

```

{% for x in [lipsum.__globals__['__bui' ~ 'ltins__']['o' ~ 'pen']('/app/flag').read()] %}{{x}}{% endfor %}

```

Listing 3: SSTI payload used to read `/app/flag`

> This payload accesses Python built-in file operations through `lipsum.__globals__`, while splitting strings such as `__builtins__` and `open` to avoid simple keyword filtering. It then reads the contents of `/app/flag` and returns them in the rendered template output.

Once the administrator page was used to view that user’s `secret`, the server executed the payload and returned the flag:[Original screenshot omitted.]

```

[REDACTED LAB FLAG]

```

Listing 4: Output showing successful file read and flag disclosure

> This confirms that the vulnerable template rendering path allowed attacker-controlled input to execute on the server and access sensitive local files.

Overall, the exploit chain was straightforward: a normal user stored a malicious Jinja template in their own `secret`, and the administrator feature later triggered its execution by rendering that stored value as template code.

### Remediation

The most important fix is to stop rendering user-controlled data as templates. Stored `secret` values should be treated as plain text only and escaped appropriately before display.

The template environment should also be restricted so that dangerous objects and runtime entry points are not exposed. Access paths such as `lipsum.__globals__`, Python built-ins, file operations, and operating-system modules should not be reachable from untrusted template input.

In addition, administrator functionality must not assume that stored user data is safe. Any data originally supplied by users should still be treated as untrusted, even when it is later viewed by an administrator.

As a defence-in-depth measure, sensitive files should not be directly readable by the web application process. Secrets and flags should be isolated from the application runtime wherever possible.

### Impact and Risk

The impact of this vulnerability is high because it turns a stored user-controlled string into server-side code execution within the template engine. In this challenge, the attacker was able to enumerate server files and then read `/app/flag`. In a real application, the same issue could expose configuration files, credentials, source code, or other sensitive data stored on the server.

The likelihood of exploitation is also high once an attacker can control a value that is later rendered by the administrator interface. The attack does not require unusual conditions or advanced memory corruption. It mainly relies on unsafe template rendering and the exposure of dangerous objects inside the Jinja execution context.

Overall, this is a serious vulnerability because it converts stored user input into executable server-side template code and allows direct access to sensitive local files.