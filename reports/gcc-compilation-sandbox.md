# Historical report: Gcc Compilation Sandbox

> Sanitized archival COMP6443 lab writeup. The larger assessment is team work; no exclusive individual authorship is inferred. Reported success is historical and has not been freshly reproduced. Addresses, accounts, identities, flags, encoded evidence, and screenshots are omitted or replaced. Code is illustrative and may contain draft errors. Read [review notes](../docs/report-review-notes.md).

## GCC

### Summary

The GCC challenge exposed an insecure server-side compilation service that could be abused as both a file disclosure and code execution primitive. By submitting crafted source files, an attacker could cause the compiler output to include sensitive local files such as `upload.php` and `download.php`. Once the leaked source code revealed the hardcoded `compiled-assets` storage path, the attacker could place a malicious PHP payload in that web-accessible location and execute it through a direct request. This vulnerability therefore allowed escalation from internal source code disclosure to confirmed server-side command execution, resulting in a high-risk compromise of the application.

### Exploitation

The attack began by treating the online compiler as a file disclosure primitive rather than just a build service. The submitted C files used inline assembly with `.incbin` to include arbitrary files from the server’s filesystem directly into the resulting output binary. 

For example, one payload embedded `/usr/local/apache2/htdocs/upload.php`, and another embedded `/usr/local/apache2/htdocs/download.php`. When those compiled artifacts were later downloaded, they revealed application source code that should never have been exposed to end users.  

```

asm(

".global blob_start\n"

"blob_start:\n"

".incbin \"/usr/local/apache2/htdocs/upload.php\"\n"

".global blob_end\n"

"blob_end:\n"

);

int main() { return 0; }

```

Listing 1: Payload embedding `upload.php` into the compiled artifact

> This payload abuses the `.incbin` assembler directive to embed the server-side file `upload.php` directly into the compiled artifact. As a result, the compilation service can be used as an arbitrary local file read primitive rather than a normal code compilation feature.

```

asm(

".global blob_start\n"

"blob_start:\n"

".incbin \"/usr/local/apache2/htdocs/download.php\"\n"

".global blob_end\n"

"blob_end:\n"

);

int main() { return 0; }

```

Listing 2: Payload embedding `download.php` into the compiled artifact

> A similar payload was then used to extract `download.php`, allowing further inspection of how compiled files were stored and later retrieved by the application.

To inspect how the compilation service handled uploaded files, the crafted `.c` payloads were submitted to the website for compilation. After compilation, the application returned a generated file, which was then downloaded and examined locally. Because the payload used `.incbin` to embed server-side files into the output, this process revealed the contents of files such as `upload.php` and `download.php`. Recovering `upload.php` was especially important, as it exposed the path used to store compiled outputs and allowed the attacker to determine where generated files would later be placed.

[Original screenshot omitted.]

Listing 3: Disclosed `upload.php` code revealing the compiled output path

> The leaked source code in this figure shows that compiled outputs are written to a hardcoded directory, `af381d14-a9b1-45b2-b753-d68dc37eac2b/compiled-assets/`. Although the path appears random, it is not a security control: once the source code is disclosed, the attacker can directly determine where generated files will be stored.

After identifying the compiled output path, the next step was to submit a payload whose generated artifact would be useful if interpreted by the web server. Instead of aiming to produce a normal executable, the submitted source embedded PHP code inside the output so that, if the file was later handled as PHP in the web-accessible directory, it would execute a server-side command. 

```

<?php

header('Content-Type: text/plain');

system('/getflag 2>&1');

exit;

?>

```

Listing 4: Embedded PHP payload used to trigger server-side command execution

> This payload is designed to return plain-text output and execute `/getflag` on the server. Its purpose is to turn the previously discovered storage path into a code execution primitive: once the generated file is placed in the executable web directory and interpreted as PHP, the attacker can run arbitrary server-side commands.

After the malicious artifact was generated, it could be accessed directly from the disclosed compiled-assets directory. Visiting the generated file confirmed that the web server executed the embedded payload rather than treating the file as an inert download.

[Original screenshot omitted.]

Listing 5: Accessing the generated file returns the flag

> Accessing the generated file under `af381d14-a9b1-45b2-b753-d68dc37eac2b/compiled-assets/` returned the flag, confirming that the uploaded payload was executed on the server. This demonstrates that the vulnerability chain resulted in server-side code execution.

Overall, the exploit chain combined two main weaknesses: first, arbitrary local file disclosure through compiler abuse; second, unsafe handling of uploaded or generated artifacts that enabled server-side execution.

### Remediation

The most important fix is to isolate the compilation environment from the web application and from sensitive server files. User code should be compiled inside a tightly sandboxed container or VM with a minimal filesystem, no access to application source code, no access to privileged binaries such as `/getflag`, and strict syscall and path restrictions.

Compiled artifacts must never be written into a web-executable directory. They should instead be stored outside the document root and, if downloads are required, served only as inert files through a controlled download handler with a fixed content type such as `application/octet-stream`. The server must not allow the resulting file name or extension to cause the artifact to be executed as PHP or any other server-side language.

The service should also validate and constrain the compilation process itself. Such services should not allow users to incorporate local server files into the output by leveraging compiler features. Mechanisms like `.incbin` should be disabled, or the entire compilation process should be confined to a minimal read-only isolated file system.

Finally, the application should avoid treating obscurity as protection. The UUID-like compiled-assets path was effectively secret only until the source code was disclosed. Sensitive storage paths and deployment assumptions should not be relied upon as a security boundary.

### Impact and Risk

The impact of this vulnerability is high because it allows more than simple information disclosure. By abusing the compilation service, an attacker can recover internal source files, identify where generated files are stored, and then use that knowledge to achieve server-side command execution. In a real system, this could expose sensitive data and compromise part of the server. Similar real-world incidents have shown that once attackers gain this kind of server-side access, the consequences can include theft of email and internal data, installation of additional malware, and long-term unauthorised access (Microsoft 2021; CISA 2017). 

The likelihood of exploitation is also high. The attack does not require administrator access or unusual conditions. It mainly relies on the compilation feature itself being poorly isolated, so once the attacker understands how the service works, the exploit can be repeated in a practical way.

Overall, this is a serious vulnerability because it turns a file disclosure weakness into confirmed code execution on the server.

### References

CISA 2017, *Mitigate Microsoft Exchange Server Vulnerabilities*. 

Microsoft 2021, *HAFNIUM targeting Exchange Servers with 0-day exploits*.