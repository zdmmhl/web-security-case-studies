# Historical report: Jinja Persistent State

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## Jinja Ninja v2

### Summary

The Jinja Ninja v2 challenge exposed a server-side template injection (SSTI) vulnerability in a Jinja rendering feature. Although each payload was restricted by a length limit, the application also preserved writable server-side state across requests for a period of time. By abusing the writable `config` object, an attacker could store intermediate objects and function references across multiple submissions, gradually reconstruct a full exploitation chain, and finally execute `/getflag`. The vulnerability therefore did not arise only from SSTI itself, but also from the unsafe persistence of attacker-controlled state.

### Exploitation

The starting point was a standard Jinja SSTI condition: user input was rendered as a template rather than treated as plain text. In a typical Jinja SSTI scenario, an attacker may try to build an execution chain such as:

```

{{ lipsum.__globals__.os.popen('/getflag').read() }}

```

Listing 1: Example of a one-shot SSTI execution chain

> This illustrates the general goal of the attack: use Jinja object access to reach Python runtime objects and eventually execute a system command. In this challenge, however, the length limit made a one-shot payload impractical, so the execution chain had to be reconstructed across multiple requests using the writable `config` object.

The important observation was that the challenge did not reset state after every request. Instead, it explicitly indicated that the state would only be reset after a period of inactivity. This meant the application preserved attacker-controlled changes long enough to support staged exploitation.

A second key observation was that `config` was not only accessible from the template context, but also writable through `config.update(...)`. As a result, `config` could be used as a persistent storage container for intermediate references between requests.

[Original screenshot omitted.]

Listing 2: Server-side configuration data exposed through the `config` object

> This output shows that the `config` object was directly accessible from the template context and exposed internal server-side configuration values. It also demonstrated that `config` was not just a hidden backend object, but an object that could be inspected and later abused during exploitation.

The exploitation strategy was therefore to split the full object chain into small stages and save each stage into `config`. Conceptually, the attack reconstructed the following chain over multiple requests:

```

u = config.update

l = lipsum

g = l.__globals__

o = g.os

p = o.popen

p('/getflag').read()

```

Listing 3: Conceptual staged object chain used for exploitation

> Instead of building the full chain in one template expression, the attacker stored short intermediate references inside `config` and then reused them in later requests.

The attack was performed in six short requests.

```

{{config.update(u=config.update)}}

{{config.u(l=lipsum)}}

{{config.u(g=config.l.__globals__)}}

{{config.u(o=config.g.os)}}

{{config.u(p=config.o.popen)}}

{{config.p('/getflag').read()}}

```

[Original screenshot omitted.]

Listing 4: Multi-stage payloads and final command execution result

> The first five payloads gradually store useful objects and function references inside `config`. The final payload calls the cached `popen` function to execute `/getflag` and read its output.

This works because each step is short enough to fit within the payload limit, but together the steps reconstruct the same capability as a much longer one-shot SSTI payload. The writable and persistent `config` object effectively becomes a cross-request variable store controlled by the attacker.

The most important point is that the length limit itself was not the real protection mechanism. Once attacker-controlled state could persist across requests, the limit only forced the exploit to become multi-stage rather than preventing exploitation. In addition, because `config` was shared and writable, this behavior exposed more than a single render operation: it allowed attacker-controlled values to remain available over time and potentially be visible across later interactions.

### Remediation

The most important fix is to stop rendering untrusted input as Jinja templates. User-controlled content should be treated as plain text, not evaluated as executable template code.

The template environment should also be restricted so that dangerous objects and runtime entry points are not exposed. Objects such as `config`, helper functions like `lipsum`, and access paths into Python globals should not be available to untrusted templates.

In addition, template context objects should not be writable by user input. A configuration object should not be modifiable from inside a rendered template, and attacker-controlled template expressions should never be able to change server-side state.

Any request-specific state should also be isolated and cleared after each request. Persistent shared state should not be reused across users or across separate template submissions. This is especially important here, because the staged exploit depended on previously stored references remaining available over multiple requests.

As a defence-in-depth measure, the application should enforce strict sandboxing for template rendering, including attribute access restrictions and denial of access to Python internals or operating-system interfaces.

### Impact and Risk

The impact of this vulnerability is high because it allows more than a single template injection. In this challenge, the attacker can gradually build an execution chain across multiple requests and eventually run a server-side command. In a real system, this could expose application secrets, configuration values, or other sensitive server-side data.

The risk is increased further by the fact that the writable `config` object persists across requests. This means the issue is not limited to one isolated render. In practice, attacker-controlled state could remain available for later use, interfere with subsequent requests, or expose shared server-side information to other users.

The likelihood of exploitation is also high once the attacker recognises that state is preserved and that `config` can be modified. The challenge does not require administrator access, memory corruption, or unusual conditions. It mainly relies on the combination of SSTI and unsafe persistent state, which makes the attack practical even under a strict input-length limit.

Overall, this is a serious vulnerability because the application turns a template renderer into a persistent execution surface. The length limit may make exploitation less direct, but it does not meaningfully reduce the underlying risk when attacker-controlled state can be stored and reused across requests.