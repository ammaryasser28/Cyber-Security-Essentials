| Topic | Level | Reading Time | Prerequisites |
|---|---|---|---|
| SSH Key Authentication and Privilege Escalation | Intermediate | ~25 min | Password Cracking Techniques, Linux basics |

> **الهدف من الـ Section ده:**  
> هتفهم إزاي SSH بيتحقق إنك فعلاً صاحب مفتاح معين، وإيه الفرق الجوهري بين الـ passphrase والـ private key والـ public key، وإزاي صلاحيات ملفات Linux بتتحكم في استخدام المفاتيح دي، وإزاي misconfiguration بسيط في صلاحيات sudo ممكن يوصّل لاختراق كامل للنظام.


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

مفتاح **PEM** غير مشفر بيبدأ بـ:

```text
-----BEGIN RSA PRIVATE KEY-----
```

**مباشرة متبوع** ببيانات المفتاح نفسها.

أما المفتاح **المشفر بـ passphrase**، فيه **سطرين إضافيين** مباشرة بعد `BEGIN`:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,A1B2C3D4E5F6...
```

السطرين دول بيقولوا إن البايتات اللي تحت دي **هي المفتاح الخام، مشفر بـ symmetric cipher**، باستخدام مفتاح مشتق من **passphrase + الـ salt** الموضح في `DEK-Info`.

```mermaid
flowchart TD
    A["PEM file starts with BEGIN RSA PRIVATE KEY"] --> B{"Proc-Type and DEK-Info lines present?"}
    B -->|"No"| C["Key is unencrypted, usable immediately"]
    B -->|"Yes"| D["Key is encrypted, needs passphrase to decrypt"]
```

### Practical Confirmation

تأكيد عملي: لو حاولت تستخدم مفتاح مشفر، SSH **هيطلب منك الـ passphrase صراحةً**، بدل ما يدخلك على طول:

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

أداة **John the Ripper** مش فاهمة بنية ملف الـ PEM مباشرة. فيه أداة مساعدة اسمها **ssh2john** بتستخرج الـ algorithm، والـ salt (من `DEK-Info`)، وعينة من بيانات المفتاح المشفرة، وتحطهم في **سطر hash واحد** يقدر John يعالجه:

```bash
ssh2john mykey.pem > key_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt key_hash.txt
```

### What the Cracking Tool Actually Does

لكل password مرشّح في الـ wordlist، John بيعمل التالي:

- بيشغّل **نفس دالة اشتقاق المفتاح (key-derivation function)** اللي مواصفات تشفير PEM بتستخدمها، وبيدخّل فيها الـ password المرشّح + الـ salt المستخرج، فينتج **symmetric key مرشّح**.
- بيستخدم المفتاح المرشّح ده في محاولة **فك تشفير جزء فحص (check portion)** من بيانات المفتاح الخاص.
- بيتحقق هل الناتج عنده **بنية RSA key صحيحة** (padding صحيح بصيغة ASN.1/DER). لو صح، يبقى الـ passphrase صحيح. لو كلام غير مفهوم (garbage)، يجرب المرشّح التالي.

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

العملية دي **محلية بالكامل، من غير أي اتصال بأي server**، وده بالظبط سبب إن تسريع الـ GPU بيخليها سريعة جداً. المهاجم مش محتاج يبعت طلبات حقيقية للسيرفر؛ هو بس بيحسب وبيتحقق محلياً، آلاف أو ملايين المرات في الثانية.

> [!IMPORTANT]
> Offline cracking is far more dangerous than online brute-forcing because there is no rate limiting, no lockout policy, and no logging on the target system to detect the attempt. The entire attack happens on the attacker's own machine, against a file they already possess.

## How SSH Public Key Authentication Works

### Prerequisite: authorized_keys

قبل أي شيء، السيرفر لازم يكون عنده ملف `~/.ssh/authorized_keys` بيحمل **الـ public keys** المسموحلها تدخل كهذا المستخدم.

### Steps 1 Through 4: Negotiation

| # | Step | What Happens |
|---|---|---|
| 1 | Connection initiation | الـ client بيشغّل `ssh user@server`، وده بيبدأ طلب اتصال |
| 2 | Capability negotiation | السيرفر بيرد موضّح أنهي طرق authentication بيدعمها (password, public key, keyboard-interactive) |
| 3 | Client offers public key auth | الـ client بيقول "عايز أثبت هويتي بـ public key"، وبيبعت الـ public key بتاعه |
| 4 | Server checks authorized_keys | مفيش تطابق = رفض فوري. فيه تطابق = يكمل (ده بس بيأكد إن المفتاح **مسجّل**، مش إن الـ client فعلاً مالك الـ private key) |

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
| 5 | Challenge issued | السيرفر بيولّد **challenge عشوائي (nonce)** ويبعته، طالباً إثبات الملكية |
| 6 | Client must sign the challenge | الـ client محتاج يستخدم الـ private key بتاعه عشان **يوقّع (sign)** الـ challenge تشفيرياً، ويبعت التوقيع |
| 7 | The passphrase problem | لو المفتاح مشفر بـ passphrase، **مايقدرش يوقّع أي حاجة وهو لسه مشفر**؛ الـ passphrase لازم تفك تشفيره **محلياً أولاً** |
| 8 | Signature verification | السيرفر بيتحقق من التوقيع باستخدام الـ public key المخزّن. التطابق الرياضي الصحيح = إثبات إن الـ client فعلاً عنده الـ private key = نجاح الـ authentication |

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
| Passphrase | بتفك تشفير المفتاح **محلياً** بس | **لأ، أبداً** |
| Private key | بتوقّع الـ challenge | لأ، بتفضل على جهاز الـ client |
| Public key | بتتحقق من التوقيع | نعم، مخزّنة مسبقاً في `authorized_keys` |

> [!IMPORTANT]
> The passphrase does NOT authenticate you to the server. The server never sees it, and it plays no role whatsoever in server-side verification. Its only job is local: decrypting the private key file on your own machine so the key material becomes usable for signing.

### Why Encrypt the Key at All

لو كده الـ passphrase ملهاش دور على السيرفر، ليه أصلاً نشفّر المفتاح؟

**مفتاح غير مشفر مسروق** بيخلي المهاجم يقدر **ينتحل هويتك فوراً**، من غير أي عائق. **مفتاح محمي بـ passphrase** بيبقى **عديم الفائدة لأي حرامي** من غير ما يعرف الـ passphrase كمان.

```mermaid
flowchart TD
    A["Attacker steals the private key file"] --> B{"Is it passphrase-protected?"}
    B -->|"No"| C["Can impersonate the owner instantly"]
    B -->|"Yes"| D["Key alone is useless without the passphrase too"]
```

## Connecting with a Cracked Key

بمجرد معرفة الـ passphrase، تقدر تتصل:

```bash
chmod 600 mykey.pem
ssh -i mykey.pem user@target
```

SSH هيطلب الـ passphrase، يفك تشفير المفتاح محلياً، يوقّع الـ challenge، ويكمل الـ handshake.

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
| `-` (الحرف الأول) | نوع الملف: `-` = ملف عادي، `d` = مجلد، `l` = symbolic link |
| `rw-` (Owner) | المالك (alice) يقدر يقرا ويكتب، مفيش execute |
| `r--` (Group) | أعضاء المجموعة المالكة للملف يقدروا يقروا بس |
| `---` (Others) | أي حد تاني على النظام **مفيش له أي صلاحية خالص** |

كل ملف عنده **بالظبط ثلاث مجموعات صلاحيات**، دايماً بنفس الترتيب: **Owner → Group → Others**. كل مجموعة فيها **3 خانات**: read/write/execute، بتظهر `-` لو مش موجودة.

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

الأمر `chmod` بياخد **ثلاث أرقام، Owner-Group-Others**، كل واحد هو **مجموع قيم** الصلاحيات المطلوبة:

| Digit | Calculation | Result |
|---|---|---|
| `6` | `4 + 2` | Read + Write، من غير execute (ده الـ Owner digit في `chmod 600`) |
| `0` | — | مفيش صلاحيات خالص، مستخدم هنا للـ Group وOthers |

`chmod 600` معناها: **Owner: rw- (6)، Group: --- (0)، Others: --- (0)**. الصيغة الرمزية المقابلة: `chmod u=rw,go= mykey.pem`.

```mermaid
flowchart LR
    A["chmod 600"] --> B["Owner digit: 6 = read(4) + write(2)"]
    A --> C["Group digit: 0 = no permissions"]
    A --> D["Others digit: 0 = no permissions"]
```

## Why SSH Enforces Key File Permissions

SSH بيفرض **فحص أمان خاص بيه** على ملفات الـ private key **قبل ما حتى يحاول يستخدمها**.

لو مفتاح معين **قابل للقراءة أو الكتابة من الـ Group أو الـ Others**، SSH **بيرفض استخدامه**، **ومش حتى هيطلب الـ passphrase**.

**السبب:** على نظام **multi-user**، مفتاح بصلاحيات مفتوحة ممكن **أي مستخدم محلي تاني يقراه**، وده بيسمحله **ينتحل هويتك** على أي سيرفر بيثق في المفتاح ده.

```bash
chmod 600 mykey.pem
```

ده بيظبط **بالظبط**: Owner يقدر يقرا ويكتب، Group مفيش، Others مفيش، وده اللي SSH بيطلبه.

بعد ما الفحص ده ينجح بس، `ssh -i` بيكمل للـ handshake ويطلب الـ passphrase.

رسالة الرفض النموذجية:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for 'mykey.pem' are too open.
```

> [!IMPORTANT]
> This permission check happens before the passphrase prompt, not after. A key file with loose permissions gets rejected instantly, regardless of how strong the passphrase is. Permissions and passphrase strength are two separate, independent layers of protection.

## Privilege Escalation Through Misconfigured sudo Rules

### Checking What You Can Run as Root

بمجرد ما تدخل على جهاز Linux بأي حساب، خطوة مهمة هي تفحص **إيه اللي مسموحلك تشغّله كـ root من غير password**:

```bash
sudo -l
```

مثال على نتيجة خطيرة:

```text
(ALL) NOPASSWD: /bin/cat
```

### Why an Unrestricted Command Is Dangerous

الأمر `cat` **مالوش أي قيود مدمجة** على أنهي ملف يقراه. **إديت صلاحية sudo شاملة له** معناها عملياً **إعطاء وصول قراءة بمستوى root لكل نظام الملفات**.

```bash
sudo cat /etc/shadow
```

الأمر ده بيطلع **password hash كل مستخدم على النظام، بما فيهم root نفسه**.

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

الخطوات دي ممكن تتكامل لتسلسل كامل:

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

- **A PEM header reveals whether a key is encrypted:** `Proc-Type` و`DEK-Info` بعد `BEGIN` معناهم المفتاح محمي بـ passphrase.
- **ssh2john bridges PEM and John the Ripper:** بيستخرج الـ algorithm والـ salt وعينة من البيانات المشفرة في صيغة hash واحدة قابلة للكسر.
- **Offline cracking tries each candidate independently:** key-derivation، فك تشفير جزئي، والتحقق من بنية ASN.1/DER الصحيحة.
- **The SSH handshake has two phases:** تفاوض وتسجيل المفتاح أولاً (خطوات 1-4)، وبعدين إثبات الملكية الفعلية بالتوقيع (خطوات 5-8).
- **Only three things matter, with three completely different roles:** الـ passphrase محلية بس، الـ private key بيوقّع، والـ public key بيتحقق على السيرفر.
- **The passphrase never reaches the server, ever.** هي دفاع محلي بس ضد سرقة المفتاح نفسه.
- **Linux permissions follow Owner-Group-Others, always in that order**، وchmod بياخد ثلاث أرقام octal بتمثل مجموع صلاحيات كل فئة.
- **SSH enforces strict key file permissions before even prompting for a passphrase:** مفتاح مفتوح الصلاحيات بيترفض فوراً.
- **A single unrestricted sudo rule can compromise an entire system:** أمر "بريء" زي `cat` بصلاحية root شاملة بيكشف `/etc/shadow` بالكامل.
