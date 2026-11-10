| Topic | Level | Reading Time | Prerequisites |
|---|---|---|---|
| SSH Key Authentication and Privilege Escalation | Intermediate | ~25 min | Password Cracking Techniques, Linux basics |

> **Goal of this section:**  
> You'll understand how SSH verifies that you actually own a particular key, the fundamental difference between the passphrase, the private key, and the public key, how Linux file permissions control the use of these keys, and how a simple misconfiguration in sudo permissions can lead to a full system compromise.


## Learning Objectives

By the end of this section, you will be able to:

- Determine whether a PEM-format private key is **passphrase-protected** by inspecting its header.
- Explain how tools like **ssh2john** and **John the Ripper** attempt to recover a key's passphrase offline.
- Describe the **SSH public key authentication handshake**, step by step, from connection to verified signature.
- Explain the critical distinction between a **passphrase**, a **private key**, and a **public key**, and why only one of them ever reaches the server.
- Read and modify **Linux file permissions** using both symbolic notation and **octal notation** (`chmod`).
- Explain why SSH refuses to use an overly permissive private key file.
- Describe a common **privilege escalation** path caused by a misconfigured `sudo` rule.

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Recognizing a Passphrase-Protected Key](#recognizing-a-passphrase-protected-key)
  - [Reading the PEM Header](#reading-the-pem-header)
  - [Practical Confirmation](#practical-confirmation)
- [Cracking a Key's Passphrase Offline](#cracking-a-keys-passphrase-offline)
  - [Converting the Key to a Crackable Hash](#converting-the-key-to-a-crackable-hash)
  - [What the Cracking Tool Actually Does](#what-the-cracking-tool-actually-does)
  - [Why This Works Entirely Offline](#why-this-works-entirely-offline)
- [How SSH Public Key Authentication Works](#how-ssh-public-key-authentication-works)
  - [Prerequisite: authorized_keys](#prerequisite-authorized_keys)
  - [Steps 1 Through 4: Negotiation](#steps-1-through-4-negotiation)
  - [Steps 5 Through 8: Proving Ownership](#steps-5-through-8-proving-ownership)
- [Critical Concept: Passphrase Versus Authentication](#critical-concept-passphrase-versus-authentication)
  - [Three Different Roles](#three-different-roles)
  - [Why Encrypt the Key at All](#why-encrypt-the-key-at-all)
- [Connecting with a Cracked Key](#connecting-with-a-cracked-key)
- [Linux File Permissions: The Basics](#linux-file-permissions-the-basics)
  - [Reading a Permission String](#reading-a-permission-string)
- [chmod and Octal Notation](#chmod-and-octal-notation)
  - [The Three Numeric Values](#the-three-numeric-values)
  - [Combining Values](#combining-values)
- [Why SSH Enforces Key File Permissions](#why-ssh-enforces-key-file-permissions)
- [Privilege Escalation Through Misconfigured sudo Rules](#privilege-escalation-through-misconfigured-sudo-rules)
  - [Checking What You Can Run as Root](#checking-what-you-can-run-as-root)
  - [Why an Unrestricted Command Is Dangerous](#why-an-unrestricted-command-is-dangerous)
  - [The Full Attack Chain](#the-full-attack-chain)
- [Career Path Connections](#career-path-connections)
- [Key Terms Glossary](#key-terms-glossary)
- [Summary](#summary)

## Recognizing a Passphrase-Protected Key

### Reading the PEM Header

An unencrypted **PEM** key starts with:

```text
-----BEGIN RSA PRIVATE KEY-----
```

**Immediately followed** by the key data itself.

A **passphrase-encrypted** key, on the other hand, has **two extra lines** right after `BEGIN`:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,A1B2C3D4E5F6...
```

These two lines say that the bytes below **are the raw key, encrypted with a symmetric cipher**, using a key derived from the **passphrase + the salt** shown in `DEK-Info`.

```mermaid
flowchart TD
    A["PEM file starts with BEGIN RSA PRIVATE KEY"] --> B{"Proc-Type and DEK-Info lines present?"}
    B -->|"No"| C["Key is unencrypted, usable immediately"]
    B -->|"Yes"| D["Key is encrypted, needs passphrase to decrypt"]
```

### Practical Confirmation

A practical confirmation: if you try to use an encrypted key, SSH **will explicitly ask you for the passphrase** instead of logging you in right away:

```bash
ssh -i mykey.pem user@target
```

```text
Enter passphrase for key 'mykey.pem':
```

> [!TIP]
> This single prompt is the fastest way to confirm whether a given key is encrypted, without even opening the file. If SSH logs you in immediately with no prompt, the key is unencrypted.

## Cracking a Key's Passphrase Offline

### Converting the Key to a Crackable Hash

**John the Ripper** doesn't understand the structure of a PEM file directly. A helper tool called **ssh2john** extracts the algorithm, the salt (from `DEK-Info`), and a sample of the encrypted key data, and puts them into **a single hash line** that John can process:

```bash
ssh2john mykey.pem > key_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt key_hash.txt
```

### What the Cracking Tool Actually Does

For each candidate password in the wordlist, John does the following:

- It runs **the same key-derivation function** that the PEM encryption spec uses, feeding it the candidate password + the extracted salt, which produces a **candidate symmetric key**.
- It uses that candidate key to try to **decrypt a check portion** of the private key data.
- It checks whether the result has a **valid RSA key structure** (correct ASN.1/DER padding). If it does, the passphrase is correct. If it's garbage, it tries the next candidate.

```mermaid
flowchart TD
    A["Candidate password from wordlist"] --> B["Run key-derivation function with password + salt"]
    B --> C["Candidate symmetric key"]
    C --> D["Attempt to decrypt check portion of key data"]
    D --> E{"Valid RSA structure (ASN.1/DER padding)?"}
    E -->|"Yes"| F["Passphrase found"]
    E -->|"No"| G["Try next candidate"]
```

### Why This Works Entirely Offline

This process is **entirely local, without any connection to any server**, and that's exactly why GPU acceleration makes it so fast. The attacker doesn't need to send real requests to the server; they just compute and check locally, thousands or millions of times per second.

> [!IMPORTANT]
> Offline cracking is far more dangerous than online brute-forcing because there is no rate limiting, no lockout policy, and no logging on the target system to detect the attempt. The entire attack happens on the attacker's own machine, against a file they already possess.

## How SSH Public Key Authentication Works

### Prerequisite: authorized_keys

Before anything else, the server must have a `~/.ssh/authorized_keys` file holding **the public keys** allowed to log in as this user.

### Steps 1 Through 4: Negotiation

| # | Step | What Happens |
|---|---|---|
| 1 | Connection initiation | The client runs `ssh user@server`, which starts a connection request |
| 2 | Capability negotiation | The server replies, stating which authentication methods it supports (password, public key, keyboard-interactive) |
| 3 | Client offers public key auth | The client says "I want to authenticate with a public key" and sends its public key |
| 4 | Server checks authorized_keys | No match = immediate rejection. A match = it continues (this only confirms the key is **registered**, not that the client actually owns the private key) |

```mermaid
flowchart LR
    A["Client: ssh user@server"] --> B["Server: lists supported auth methods"]
    B --> C["Client: offers public key"]
    C --> D{"Match in authorized_keys?"}
    D -->|"No"| E["Rejected immediately"]
    D -->|"Yes"| F["Proceed to ownership proof"]
```

> [!WARNING]
> Step 4 only confirms that the offered public key is registered. It proves nothing yet about who the client actually is. An attacker who somehow obtained only the public key (which is not secret by design) still cannot pass the next step.

### Steps 5 Through 8: Proving Ownership

| # | Step | What Happens |
|---|---|---|
| 5 | Challenge issued | The server generates a **random challenge (nonce)** and sends it, asking for proof of ownership |
| 6 | Client must sign the challenge | The client needs to use its private key to cryptographically **sign** the challenge, and sends the signature |
| 7 | The passphrase problem | If the key is passphrase-encrypted, **it can't sign anything while it's still encrypted**; the passphrase must decrypt it **locally first** |
| 8 | Signature verification | The server verifies the signature using the stored public key. A correct mathematical match = proof that the client really has the private key = successful authentication |

```mermaid
flowchart LR
    A["Server sends random challenge (nonce)"] --> B{"Is private key encrypted?"}
    B -->|"Yes"| C["Passphrase decrypts it locally first"]
    B -->|"No"| D["Key is already usable"]
    C --> E["Client signs the challenge with the private key"]
    D --> E
    E --> F["Server verifies signature using stored public key"]
    F --> G["Authentication succeeds"]
```

## Critical Concept: Passphrase Versus Authentication

### Three Different Roles

```mermaid
flowchart LR
    A["Passphrase: decrypts key locally"] --> B["Private key: signs server's challenge"]
    B --> C["Public key: on server, verifies signature"]
```

| Element | Role | Travels to the Server? |
|---|---|---|
| Passphrase | Decrypts the key **locally** only | **No, never** |
| Private key | Signs the challenge | No, it stays on the client machine |
| Public key | Verifies the signature | Yes, stored in advance in `authorized_keys` |

> [!IMPORTANT]
> The passphrase does NOT authenticate you to the server. The server never sees it, and it plays no role whatsoever in server-side verification. Its only job is local: decrypting the private key file on your own machine so the key material becomes usable for signing.

### Why Encrypt the Key at All

If the passphrase plays no role on the server, why encrypt the key at all?

**A stolen unencrypted key** lets the attacker **impersonate you instantly**, with no obstacle at all. **A passphrase-protected key** is **useless to any thief** unless they also know the passphrase.

```mermaid
flowchart TD
    A["Attacker steals the private key file"] --> B{"Is it passphrase-protected?"}
    B -->|"No"| C["Can impersonate the owner instantly"]
    B -->|"Yes"| D["Key alone is useless without the passphrase too"]
```

## Connecting with a Cracked Key

Once you know the passphrase, you can connect:

```bash
chmod 600 mykey.pem
ssh -i mykey.pem user@target
```

SSH will ask for the passphrase, decrypt the key locally, sign the challenge, and complete the handshake.

## Linux File Permissions: The Basics

### Reading a Permission String

```bash
ls -l mykey.pem
```

```text
-rw-r----- 1 alice alice 1679 Oct 11 02:11 mykey.pem
```

| Segment | Meaning |
|---|---|
| `-` (the first character) | The file type: `-` = regular file, `d` = directory, `l` = symbolic link |
| `rw-` (Owner) | The owner (alice) can read and write, no execute |
| `r--` (Group) | Members of the group that owns the file can only read |
| `---` (Others) | Anyone else on the system **has no permissions at all** |

Every file has **exactly three permission sets**, always in the same order: **Owner → Group → Others**. Each set has **3 slots**: read/write/execute, showing `-` when a permission is absent.

```mermaid
flowchart LR
    A["Permission string: -rw-r-----"] --> B["Owner: rw-"]
    A --> C["Group: r--"]
    A --> D["Others: ---"]
```

## chmod and Octal Notation

### The Three Numeric Values

```bash
chmod 600 mykey.pem
```

| Value | Meaning |
|---|---|
| `4` | Read (r) |
| `2` | Write (w) |
| `1` | Execute (x) |

### Combining Values

The `chmod` command takes **three digits, Owner-Group-Others**, each of which is **the sum of the values** of the permissions you want:

| Digit | Calculation | Result |
|---|---|---|
| `6` | `4 + 2` | Read + Write, no execute (this is the Owner digit in `chmod 600`) |
| `0` | — | No permissions at all, used here for Group and Others |

`chmod 600` means: **Owner: rw- (6), Group: --- (0), Others: --- (0)**. The equivalent symbolic form: `chmod u=rw,go= mykey.pem`.

```mermaid
flowchart LR
    A["chmod 600"] --> B["Owner digit: 6 = read(4) + write(2)"]
    A --> C["Group digit: 0 = no permissions"]
    A --> D["Others digit: 0 = no permissions"]
```

## Why SSH Enforces Key File Permissions

SSH enforces **its own security check** on private key files **before it even tries to use them**.

If a key is **readable or writable by the Group or Others**, SSH **refuses to use it**, **and won't even ask for the passphrase**.

**The reason:** on a **multi-user** system, a key with open permissions could be **read by any other local user**, which would let them **impersonate you** on any server that trusts that key.

```bash
chmod 600 mykey.pem
```

This sets **exactly**: Owner can read and write, Group nothing, Others nothing, which is what SSH requires.

Only after this check passes does `ssh -i` continue to the handshake and ask for the passphrase.

The typical rejection message:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for 'mykey.pem' are too open.
```

> [!IMPORTANT]
> This permission check happens before the passphrase prompt, not after. A key file with loose permissions gets rejected instantly, regardless of how strong the passphrase is. Permissions and passphrase strength are two separate, independent layers of protection.

## Privilege Escalation Through Misconfigured sudo Rules

### Checking What You Can Run as Root

Once you're logged in to a Linux machine with any account, an important step is to check **what you're allowed to run as root without a password**:

```bash
sudo -l
```

An example of a dangerous result:

```text
(ALL) NOPASSWD: /bin/cat
```

### Why an Unrestricted Command Is Dangerous

The `cat` command **has no built-in restrictions** on which file it reads. **Giving it blanket sudo permission** effectively means **granting root-level read access to the entire filesystem**.

```bash
sudo cat /etc/shadow
```

This command reveals **the password hash of every user on the system, including root itself**.

```mermaid
flowchart TD
    A["sudo -l reveals: NOPASSWD /bin/cat"] --> B["cat has no file restrictions"]
    B --> C["sudo cat /etc/shadow"]
    C --> D["Reveals password hashes for every user, including root"]
    D --> E["Crack the root hash offline, same tools as before"]
    E --> F["su root with the cracked password"]
```

> [!WARNING]
> Granting unrestricted `sudo` access to a seemingly harmless command like `cat`, `less`, `vim`, or `find` is a classic privilege escalation path. The danger is never in the command's normal purpose; it is in the fact that the command can read (or sometimes write) any file on the system once it runs as root.

### The Full Attack Chain

These steps can be combined into a complete chain:

```mermaid
flowchart LR
    A["Crack a password-protected private key offline"] --> B["Authenticate via SSH as a normal user"]
    B --> C["Check sudo -l for misconfigured permissions"]
    C --> D["Abuse an unrestricted command to read /etc/shadow"]
    D --> E["Crack the root password hash offline"]
    E --> F["Switch to root with su"]
```

> [!TIP]
> Every stage in this chain reuses the same core idea from earlier in this series: a hash alone, whether it belongs to a user account or a private key, is only as safe as the strength of the password behind it, and offline cracking removes every rate limit or lockout that would normally slow an attacker down.

## Career Path Connections

| Career Path | How This Topic Applies |
|---|---|
| SOC Analyst | Monitors for unusual sudo rule usage and SSH authentication attempts as key indicators of post-exploitation activity |
| Penetration Tester | SSH key cracking and sudo misconfiguration abuse are standard techniques in internal assessments and CTF-style engagements |
| GRC | File permission standards and least-privilege sudo policies are direct, auditable security controls |
| Cloud Security | SSH key management and least-privilege IAM roles follow the exact same principles in cloud compute environments |

## Key Terms Glossary

| Term | Definition |
|---|---|
| PEM | A common text-based encoding format for cryptographic keys and certificates |
| Passphrase | A secret that decrypts a private key file locally; it never reaches the server |
| Private Key | The secret half of a key pair, used to sign a server's authentication challenge |
| Public Key | The non-secret half of a key pair, stored on the server to verify signatures |
| authorized_keys | The file on an SSH server listing public keys permitted to authenticate as a given user |
| Nonce | A random, single-use challenge value generated by the server during authentication |
| Key-Derivation Function | An algorithm that transforms a password and salt into a usable cryptographic key |
| ASN.1/DER | A structured binary encoding format used to validate that decrypted key data is genuine |
| chmod | The Linux command used to change file permission settings |
| Octal Notation | A numeric shorthand (0-7) representing combinations of read, write, and execute permissions |
| sudo | A Linux command allowing a permitted user to run commands with elevated (root) privileges |
| Privilege Escalation | Gaining higher-level access than originally granted, often through misconfiguration |

## Summary

- **A PEM header reveals whether a key is encrypted:** `Proc-Type` and `DEK-Info` after `BEGIN` mean the key is passphrase-protected.
- **ssh2john bridges PEM and John the Ripper:** it extracts the algorithm, the salt, and a sample of the encrypted data into a single crackable hash format.
- **Offline cracking tries each candidate independently:** key derivation, partial decryption, and checking for a valid ASN.1/DER structure.
- **The SSH handshake has two phases:** first negotiation and key registration (steps 1-4), then the actual proof of ownership through the signature (steps 5-8).
- **Only three things matter, with three completely different roles:** the passphrase is local only, the private key signs, and the public key verifies on the server.
- **The passphrase never reaches the server, ever.** It's purely a local defense against theft of the key itself.
- **Linux permissions follow Owner-Group-Others, always in that order**, and chmod takes three octal digits representing the sum of each category's permissions.
- **SSH enforces strict key file permissions before even prompting for a passphrase:** a key with open permissions is rejected immediately.
- **A single unrestricted sudo rule can compromise an entire system:** an "innocent" command like `cat` with blanket root permission exposes all of `/etc/shadow`.
