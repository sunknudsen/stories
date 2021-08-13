<!--
Title: Exploring the password policy rabbit hole
Description: Exploring the password policy rabbit hole… how to avoid getting pwned while keeping some level of convenience.
Cover image: office.jpg
Publication date: 2021-08-11T11:27:23.819Z
Listed: true
-->

<span class="drop-cap">S</span>ecuring one’s computers and accounts (or one’s organization) is a challenging balancing act between convenience and security.

Implementing too much security (such as using very long passwords) hurts productivity while too little will likely result in getting [pwned](https://haveibeenpwned.com/).

Striking the right balance and avoiding [common misconceptions](#common-misconceptions) requires a deep understanding of how password security works… which requires time… a rare commodity among humans.

In this story, I will guide you down the password policy rabbit hole shedding light on [password entropy](#password-entropy), [hardware random number generators](#hardware-random-number-generator-hrng), [key derivation functions](#key-derivation-function-kdf), [secure elements](#secure-element) and why deeply understanding these topics is fundamental when adopting (or drafting) password policy suited to one’s [threat model](#threat-model).

I would like to thank [TrustToken](https://www.trusttoken.com/) for supporting this research. 🙌

## Common misconceptions

### 1. Using 5-word passphrase is less secure than 8-character password.

When truly random, a 5-word passphrase generated using [EFF’s Short Wordlist #1](https://eff.org/files/2016/09/08/eff_short_wordlist_1.txt) has more [entropy](#password-entropy) than 8-character password, therefore is **more secure**.

### 2. Using 8-character password is never secure enough.

It really depends on which [key derivation function](#key-derivation-function-kdf) and [secure element](#secure-element) is used, if any, to harden password.

In the context of macOS, password is secured using PBKDF2-SHA512 (generally using over 25,575 iterations) and, when FileVault is enabled on Macs equipped with Apple T2 Security Chip, media key (used to encrypt data) is wrapped (double-encrypted) using key derived from password and unique ID (UID) fused to [secure element](#secure-element).

On older Macs that do not have T2 chip, brute-forcing password using [hashcat](https://hashcat.net/hashcat/) on [AWS](https://aws.amazon.com/)’ most powerful instance type (currently [p4d.24xlarge](https://aws.amazon.com/ec2/instance-types/p4/)) costs over $3M USD (at discounted [spot pricing](https://aws.amazon.com/ec2/spot/pricing/)).

On recent T2-equipped Macs with FileVault enabled, brute-forcing password is theoretically impossible.

**This is likely secure enough for most [threat models](#threat-model).**

### 3. One should never use same password to unlock computer and password manager.

It really depends on operating system and password manager.

[1Password](https://1password.com/), for example, encrypts user data using both [Secret Key](https://support.1password.com/secret-key-security/) and password while never sending either over the Internet thanks to a [Secure Remote Password](https://en.wikipedia.org/wiki/Secure_Remote_Password_protocol) (SRP) [implementation](https://support.1password.com/secure-remote-password/).

In the context of macOS, given both operating system and 1Password share same [user space](https://en.wikipedia.org/wiki/User_space), using two separate passwords yields little, if any, additional security.

### 4. One should rotate passwords as often as possible.

Security researchers have [proven](https://www.ftc.gov/news-events/blogs/techftc/2016/03/time-rethink-mandatory-password-changes) scheduling password rotations is generally **less secure** given human nature (unless password has been compromised).

**Entering the rabbit hole…**

## Password entropy

Password entropy is a measure of [password strength](https://en.wikipedia.org/wiki/Password_strength).

Say we have two six-sided dices, how many combinations are there?

> Calculating number of combinations is done using the following formula.
>
> **(number of possibilities per value)<sup>^</sup>(number of values)**

For two six-sided dices, number of combinations equals **6<sup>^</sup>2** or **36**.

> Calculating password entropy is done using the following formula.
>
> **log<sub>2</sub>(number of combinations)**

For two six-sided dices, entropy equals **log<sub>2</sub>(36)** or **~5.17 bits**.

The command line utility [bc](https://www.gnu.org/software/bc/manual/html_mono/bc.html) is very useful when calculating entropy.

```console
$ echo 'l(6^2)/l(2)' | bc -l
5.16992500144231236295
```

Passwords should be truly random and should contain enough entropy to mitigate [bruce-force attacks](https://en.wikipedia.org/wiki/Brute-force_attack) to the extent of one’s [threat model](#threat-model).

Statistically, an attacker needs to try 2<sup>entropy-1</sup> combinations to [deterministically](https://en.wikipedia.org/wiki/Deterministic_algorithm) brute-force password.

For two six-sided dices, number of combinations to try equals **2<sup>5.17-1</sup>** or **18**.

Now, what is entropy of truly random 8-character password (when using “a-z, A-Z, 0-9 and \_-;:!?.@\*/#%$” character set)?

```console
$ echo 'l(75^8)/l(2)' | bc -l
49.83054952396704701806
```

Entropy equals **~49.83 bits** resulting in having to try **2<sup>49.83-1</sup>** or **500,373,945,961,594** combinations to brute-force password.

## Hardware random number generator (HRNG)

Humans are inherently bad at generating truly random data (using their mind) from which truly random passwords can be derived.

Being aware of this, humans developed
[hardware random number generators](https://en.wikipedia.org/wiki/Hardware_random_number_generator) designed to generate truly random data derived from physical properties such as phase drift of ring oscillators.

Hardware random number generators tend to generate bits of entropy (random data in other words) slowly. As a result, generated bits are often used as seed of [cryptographically-secure pseudorandom number generators](https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator).

Most (if not all) computers now ship with built-in hardware random number generator and most (if not all) [operating systems](https://en.wikipedia.org/wiki/Operating_system) now include public [application programming interfaces](https://en.wikipedia.org/wiki/API) (APIs) applications such as browsers and password managers use to generate truly random data.

For example, [secure element](#secure-element) found in Apple T2 Security Chip includes a hardware random number generator used as entropy source.

## Key derivation function (KDF)

In order to use the [Advanced Encryption Standard](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) (better known as AES) one needs a fixed-length key of 128, 192 or 256 bits.

One can generate a 256-bit key (encoded as [hexadecimal](https://en.wikipedia.org/wiki/Hexadecimal) string) using `openssl` command.

```console
$ openssl rand -hex 32
ce7df8f96e691ec46e7cd2c7e1bad90f4432a6d6fb6e7a8ce880a841bed4d394
```

Remembering above key would be hard right? Thankfully, keys can also be derived from passwords with less entropy using [cryptographic hash functions](https://en.wikipedia.org/wiki/Cryptographic_hash_function) such as [SHA-256](https://en.wikipedia.org/wiki/SHA-2) (generally not recommended) or [key derivation functions](https://en.wikipedia.org/wiki/Key_derivation_function) such as [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) (used by macOS to store passwords and 1Password to [secure](https://support.1password.com/pbkdf2/) master password) or [Argon2](https://en.wikipedia.org/wiki/Argon2) (used by [KeyPassXC](https://keepassxc.org/) to secure password).

Using cryptographic hash function is only recommended when key is [wrapped](https://support.apple.com/en-ie/guide/security/sec4c6dc1b6e/1/web/1#secde6cf5956) in additional layer(s) of encryption (ever wondered why one can change encryption password instantly or how more than one user or recovery key can unlock [FileVault](https://en.wikipedia.org/wiki/FileVault) or [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) volumes?).

```console
$ echo "coral barn wing armor acid" | openssl dgst -sha256
e5b3ad893826d624f821b4a1a6fea1fbe3bf8ab5649a69ba880cf7b99a1bd42c

$ echo "coral barn wing armor acid" | argon2 $(openssl rand -base64 32)
Type:		Argon2i
Iterations:	3
Memory:		4096 KiB
Parallelism:	1
Hash:		d45ae284815c65e8101fd64c14b07b241d589ace493aad45845c8ebed3d53b28
Encoded:	$argon2i$v=19$m=4096,t=3,p=1$ZXE3dElZazR6SFJHUWxkR09ra0pucHVqVFdtOTYzR1NMTUFqNGVyYmlMQT0$1FrihIFcZegQH9ZMFLB7JB1Yms5JOq1FhFyOvtPVOyg
0.009 seconds
Verification ok
```

Using Argon2 (with default settings), hash (which is also 256-bit key) is derived from password in 0.009 seconds.

Let’s run above commands 1000 times each and see how long it takes.

```console
$ time (for i in {1..1000}; do echo "coral barn wing armor acid" | openssl dgst -sha256 > /dev/null; done)
( for i in {1..1000}; do; echo "coral barn wing armor acid" | openssl dgst  >)  1.52s user 1.96s system 99% cpu 3.485 total

$ salt=$(openssl rand -base64 32); time (for i in {1..1000}; do echo "coral barn wing armor acid" | argon2 $salt > /dev/null; done)
( for i in {1..1000}; do; echo "coral barn wing armor acid" | argon2 $salt > )  15.76s user 3.16s system 94% cpu 20.090 total
```

Using Argon2 (with default settings) vs SHA-256 takes 15.76s vs 1.52s. **Longer is better to mitigate brute-force attacks.**

Most, if not all, key derivation functions have a feature called **iterations** designed to slow things down (configured using `-t` in the context of Argon2).

```console
$ echo "coral barn wing armor acid" | argon2 $(openssl rand -base64 32) -t 100
Type:		Argon2i
Iterations:	100
Memory:		4096 KiB
Parallelism:	1
Hash:		3c4f8af639f388be91b6b9176d00b15e1b449b4d240ea5fbd66bc6f0c20297e3
Encoded:	$argon2i$v=19$m=4096,t=100,p=1$Y29FYUJQc2ZJeGp5S2lueWQyVndRdnhteTNzS0FITGxaZFlhVTA2M2JzOD0$PE+K9jnziL6RtrkXbQCxXhtEm00kDqX71mvG8MICl+M
0.231 seconds
Verification ok

$ echo "coral barn wing armor acid" | argon2 $(openssl rand -base64 32) -t 1000
Type:		Argon2i
Iterations:	1000
Memory:		4096 KiB
Parallelism:	1
Hash:		3946b19e54c219bc60a81325099a980b64c0a4c1628c6c6f913253ef9a5bee4b
Encoded:	$argon2i$v=19$m=4096,t=1000,p=1$V0dOWklOeGtVbG4zQ2lIcE9naXVPWDBKNmFoQ285YUhOTlN3aHpESUV1MD0$OUaxnlTCGbxgqBMlCZqYC2TApMFijGxvkTJT75pb7ks
2.271 seconds
Verification ok
```

Notice how increasing iterations tenfold (from 100 to 1000) resulted in key derivation taking 10 times longer (2.271 seconds vs 0.231 seconds)?

By the way, you may be wondering why derived hashes (`3c4f8af6…` and `3946b19e…`) are not identical.

Key derivation functions also have a feature called a **salt** which, when generated randomly (using `openssl rand -base64 32` in above examples), mitigates [rainbow table](https://en.wikipedia.org/wiki/Rainbow_table) attacks. A rainbow table is a precomputed database of hashes with corresponding passwords.

Argon2 improves upon PBKDF2 by being more configurable and resistent to [time–memory trade-off](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff) (TMTO) attacks.

When using Argon2, one can adjust the time (`-t`), memory (`-m`) and parallelism (`-p`) “cost” of attack to mitigate one’s [threat model](#threat-model).

## Secure element

Secure elements are specialized [systems on a chip](https://en.wikipedia.org/wiki/System_on_a_chip) (SoC) designed to compartmentalize sensitive data and computing away from the [central processing unit](https://en.wikipedia.org/wiki/Central_processing_unit) (CPU) and, when applicable, [memory](https://en.wikipedia.org/wiki/Computer_memory).

Secure elements often self-generate cryptographic keys [fused](https://support.apple.com/en-ca/guide/security/sec59b0b31ff/web) to system on a chip during manufacturing making them (theoretically) safe from [supply chain attacks](https://en.wikipedia.org/wiki/Supply_chain_attack).

In the context of [Apple T2 Security Chip](https://www.apple.com/ca/mac/docs/Apple_T2_Security_Chip_Overview.pdf), a [unique ID](https://support.apple.com/en-ca/guide/security/sec59b0b31ff/web#sec293d3d1f5) (UID) is self-generated during manufacturing using [built-in](https://support.apple.com/en-ca/guide/security/sec59b0b31ff/web#sec285763cef) hardware random number generator and fused to system on a chip. UID is used to encrypt media key used to encrypt internal SSD (using [built-in](https://support.apple.com/en-ca/guide/security/sec59b0b31ff/web#sec26bcbfbb8) AES engine).

Ever wondered why enabling FileVault on T2-equipped Macs is immediate (compared to enabling FileVault on pre-T2 Macs)?

> If FileVault isn’t enabled on a Mac with the T2 chip during the initial Setup Assistant process, the volume is still encrypted, but the volume key is protected only by the hardware UID in the Secure Enclave. If FileVault is enabled later—a process that is immediate since the data was already encrypted—an anti-replay mechanism prevents the old key (based on hardware UID only) from being used to decrypt the volume. The volume is then protected by a combination of the user password with the hardware UID as previously described.

Media key is stored outside secure enclave using effaceable storage so it can be quickly (and [permanently](https://en.wikipedia.org/wiki/Data_erasure)) destroyed (a process called [crypto-shredding](https://en.wikipedia.org/wiki/Crypto-shredding)).

This is how iOS, iPadOS and macOS devices are [remotely wiped](https://support.apple.com/en-ca/guide/deployment-reference-ios/apd713df1b14/web#apddc756c463).

Same logic applies when “Erase Data” is enabled on iOS or iPadOS and failed passcode attempts are exhausted.

> The media key is located in effaceable storage and designed to be quickly erased on demand; for example, via remote wipe using Find My Mac or when enrolled in a mobile device management (MDM) solution. Effaceable storage accesses the underlying storage technology (for example, NAND) to directly address and erase a small number of blocks at a very low level. Erasing the media key in this manner renders the volume cryptographically inaccessible.

Secure elements also tend to be more resistant to [side-channel attacks](https://en.wikipedia.org/wiki/Side-channel_attack).

## Threat model

Evaluating one’s threat model is essential to strike the right balance between convenience and security.

For example, an average person likely doesn’t need to use 7-word passphrase (~90.5 bits of entropy) to secure computer but a [whistleblower](https://en.wikipedia.org/wiki/Whistleblower) likely does.

One needs to evaluate how much an adversary is willing to spend to breach one’s security while acknowledging that, in the context of passwords and physical security, [5 dollar wrench attacks](https://en.bitcoin.it/wiki/Storing_bitcoins#The_5_dollar_wrench_attack) or [rubber-hose cryptanalysis](https://en.wikipedia.org/wiki/Rubber-hose_cryptanalysis) are very effective (and low cost).

Putting physical security aside, computers are inherently insecure given their design and number of apps one installs increasing attack surface to unmanageable levels.

As a result, **compartmentatlization is one’s first line of defence**. In other words, one should use [air-gapped](<https://en.wikipedia.org/wiki/Air_gap_(networking)>) devices (such as security keys or hardware wallets) to compartmentalize sensitive use cases.

No matter how secure one’s passwords are, if [key logger](https://en.wikipedia.org/wiki/Keystroke_logging) is running on computer, one has been pwned.

## Recommendations

Here are a few high value low opportunity cost recommendations that one should always follow.

- Always generate truly random passwords (vs making them up) such as passphrase generated using [dices](https://www.eff.org/dice) or cryptographically-secure command line utilities such as [passphraseme](https://github.com/micahflee/passphraseme)
- Always enable [full disk encryption](https://en.wikipedia.org/wiki/Disk_encryption) (and never store recovery key using online services such as iCloud)
- Always enable [multi-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) (when possible) and favor [FIDO U2F](https://en.wikipedia.org/wiki/Universal_2nd_Factor) or [FIDO2](https://en.wikipedia.org/wiki/FIDO2_Project) (to mitigate [phishing](https://en.wikipedia.org/wiki/Phishing) attacks) unless physical attack (such as targeted theft) is a non-theoretical threat model (and never store [TOTP](https://en.wikipedia.org/wiki/Time-based_One-Time_Password) hashes in password manager)
- Always use password manager to generate long random passwords for online accounts (and never reuse password for more than one account)
- Always compartmentalize sensitive data and computing (for example, using security key to secure one’s PGP private keys or using hardware wallet to secure one’s crypto)
- Always lock screen when away and shutdown device when in transit (especially when going through customs).
- Never schedule password rotations (unless password has been compromised)

Hope this research is helpful to other.

Stay safe!
