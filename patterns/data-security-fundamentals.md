# Data Security Fundamentals

Reference for thinking through data protection at the architecture level. Covers the distinctions and decision frameworks that surface in design reviews and security debates.

## Encoding ≠ Encryption

This is the most common confusion in security discussions and worth being precise about.

| | Encoding (e.g., base64) | Encryption (e.g., AES) |
|---|---|---|
| Purpose | Represent bytes in a different alphabet (e.g., binary as text) | Make bytes unreadable without a key |
| Reversal | Anyone with knowledge of the scheme | Only with the correct key |
| Key required? | No | Yes |
| Security value | Zero | Real, depending on key management |
| Example use | Wrapping varbinary in JSON | Protecting data at rest or in flight |

If someone says "decrypt the base64," they almost certainly mean either:

- "Decode the base64" (just `Convert.FromBase64String` — no key needed), or
- "The data is base64-encoded ciphertext; decode to get bytes, then decrypt with the key"

These are very different operations. Push for clarity when the distinction is muddled — confusing the two tends to end with either security theater (assuming base64 protects something) or unnecessary work (writing decryption code when only decoding is needed).

## Defense in depth

The principle: assume any single security layer can fail, and design so that no single failure exposes the protected asset.

Practical implications:

- **Network controls** (IP allowlists, firewalls) protect against attackers reaching the system at all
- **Identity controls** (AD, OAuth, MFA) ensure only authenticated users connect
- **Authorization controls** (RBAC, ACLs) ensure authenticated users only access their data
- **Encryption at rest** protects against threats that bypass the previous layers (backup theft, disk theft, DBA reads)
- **Encryption in flight** (TLS) protects against network observers
- **Audit logging** detects exploitation after the fact

Each layer has a different threat it addresses well and others it addresses poorly. The art is matching layers to threats — not just stacking everything.

### Common anti-patterns

- **"We have authentication, so we don't need authorization"** — once authenticated, every user has access to everything. Authentication and authorization are different problems.
- **"We have TLS, so we don't need encryption at rest"** — TLS protects in-flight data, not data sitting on disk or in backups.
- **"We have encryption at rest, so backups are safe to share"** — only true if the keys aren't in the backup. Most backup systems include the encryption keys for the database.
- **"We have permissions, so DBAs can't see the data"** — DBAs typically have admin rights; permissions don't apply to them by design. Only column-level encryption with externally-held keys protects against DBA reads.

## Threat modeling — what each control buys you

A simple framework for evaluating whether a security measure is doing useful work: list the threats it addresses and the threats it doesn't.

| Threat | Network controls | Identity / RBAC | TLS | Encryption at rest (server-held keys) | Encryption at rest (external keys, e.g. Always Encrypted, app-layer) |
|---|---|---|---|---|---|
| External attacker with no credentials | ✓ | ✓ | ✓ | — | — |
| Compromised user account | ✗ | partial | — | — | — |
| Insider with DB access (DBA) | — | ✗ | — | ✗ | ✓ |
| Backup theft / restore to lower env | — | — | — | depends | ✓ |
| Network eavesdropping | — | — | ✓ | — | — |
| Disk theft | — | — | — | ✓ | ✓ |
| Compromised app server with DB credentials | — | — | — | partial | partial |
| Memory dump of running process | — | — | — | ✗ | ✗ |

Use this to answer "what does this layer actually protect against?" When someone proposes a control, the right question is which threat it addresses and whether that threat is already covered by something else.

## Where keys live matters more than that they exist

The strength of any encryption is bounded by the strength of the key management. Common patterns ranked by strength:

1. **Hardware Security Module (HSM)** — Keys never leave the HSM. Encryption/decryption happens via API calls. Highest assurance, highest cost.
2. **Cloud KMS / Key Vault with strict access policies** — Keys are managed by the cloud provider; access controlled by IAM. Operations cached but keys not exfiltrated to clients. High assurance for typical threat models.
3. **Cloud KMS / Key Vault with permissive access** — As above but the access policies grant broad read. Strength depends on who has access.
4. **Server-held with external cert chain** — Keys live in SQL Server but protected by a cert chain that requires admin password to unlock. Reasonable assurance against DBA reads if cert chain is properly protected.
5. **Server-held with default protection** — Keys live in SQL Server protected only by the default service master key. Any sysadmin can unlock and read.
6. **App config with file-system permissions** — Keys in a config file readable by the app process. Vulnerable to anyone who can read the filesystem.
7. **Hardcoded in source** — Keys in source code or compiled binaries. Effectively no protection; treat as no encryption.

Asking "where is the key actually stored, and who can access it?" is more productive than asking "are we using AES-256?" The cipher is rarely the weak link; key handling almost always is.

## Authenticated vs unauthenticated encryption

When you encrypt data, you also typically need a way to detect tampering. Two approaches:

- **Unauthenticated (e.g., AES-CBC alone)** — Encrypts but doesn't detect modification. An attacker who can write to the ciphertext can flip bits in the plaintext (sometimes predictably) without being detected.
- **Authenticated (e.g., AES-GCM, AES-CCM, ChaCha20-Poly1305)** — Encrypts and produces a tag that detects any modification. Decryption fails on tampering.

Always prefer authenticated encryption when available. AES-GCM is the modern default. Older code using AES-CBC without an HMAC is a substitution-attack risk.

SQL Server's `EncryptByKey` has an optional **authenticator** parameter that provides a similar guarantee — binding ciphertext to a row identifier so an attacker can't swap rows.

## When encryption is required vs nice to have

Compliance frameworks often dictate encryption requirements. The most common drivers:

- **NIST 800-53 SC-28** — Protection of Information at Rest. Common driver in government and regulated environments.
- **HIPAA Security Rule** — Encryption of ePHI at rest and in transit (addressable, not mandatory, but de-facto required).
- **PCI DSS Requirement 3** — Cardholder data must be encrypted; specific requirements for key management.
- **GDPR Article 32** — "Appropriate technical measures" — interpreted as encryption for sensitive personal data.
- **State data breach laws** — Some (e.g., California, New York) provide safe harbor against breach notification if breached data was encrypted.

When encryption is compliance-driven rather than risk-driven, "we have great access controls" doesn't substitute. The control says encrypt, so encrypt — even if you've judged the residual risk to be low.

## Costs of encryption

Worth being honest about what you give up:

- **Search** — Encrypted columns can't be substring-searched, range-queried, or full-text indexed without giving up the encryption guarantee
- **Joins** — Joins on encrypted columns work only if both sides use the same key and deterministic encryption
- **Performance** — Negligible for most workloads with hardware AES, but stored-proc roundtrips add latency for batch operations
- **Operational complexity** — Key rotation, certificate renewal, key store availability become deployment concerns
- **Debugging** — Reading production data for incident response requires going through the decryption path; raw queries return ciphertext
- **Backup/restore complexity** — Restoring backups across environments requires the key chain; mismatched keys mean unreadable data
- **Disaster recovery testing** — Must verify the key chain is restorable, not just the data

These aren't reasons not to encrypt; they're reasons to know what you're signing up for and design around it.

## Practical "do you actually need encryption here" checklist

Before adding encryption to a column or system:

1. **What's the threat?** Be specific. "Backup theft when sent to vendor for support" is concrete; "general security" is not.
2. **Is the threat in scope for our architecture?** A purely internal app with strong network controls has a different threat profile than a public-facing API.
3. **Is encryption the right control for this threat?** Sometimes the answer is access controls, audit logging, network segmentation, or data minimization.
4. **What's the operational cost?** Search limitations, key management overhead, debugging friction.
5. **What's required by compliance?** If the framework says encrypt, this becomes a checkbox question rather than an analysis question.
6. **Where will keys live?** This is the question that determines whether the encryption actually does work or is theater.

If you can answer these, you have a defensible architectural decision either way.
