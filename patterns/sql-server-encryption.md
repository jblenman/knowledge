# SQL Server Encryption Patterns

Reference for the encryption mechanisms in SQL Server / Azure SQL, what each protects against, and how to integrate them with .NET applications.

## Four architectural patterns at a glance

| Pattern | Where decryption happens | Where keys live | Key C# behavior |
|---|---|---|---|
| **Column-level (symmetric key + cert)** | Server (in stored proc) | Inside SQL Server (cert chain) | C# only sees ciphertext or already-decrypted cleartext. No crypto in app |
| **Always Encrypted** | Client (.NET SqlClient driver) | External (Key Vault, Windows cert store) | Driver decrypts transparently when reading rows |
| **Application-layer** | Client (your app) | Wherever the app gets them (Key Vault, KMS, file) | App uses `System.Security.Cryptography` directly |
| **Transparent Data Encryption (TDE)** | At storage layer (transparent) | DB master key chain | None — invisible to apps. Protects only at-rest, not in flight or in memory |

Picking the right one depends on threat model and search/query needs more than on technical capability.

## Column-level encryption (symmetric key + certificate)

The classic pattern. T-SQL syntax:

```sql
-- One-time setup (typically by a DBA in a controlled session)
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'strong-password-not-stored-in-source';
CREATE CERTIFICATE MyCert WITH SUBJECT = 'Cert for column encryption';
CREATE SYMMETRIC KEY MyKey WITH ALGORITHM = AES_256 ENCRYPTION BY CERTIFICATE MyCert;

-- Encrypt
OPEN SYMMETRIC KEY MyKey DECRYPTION BY CERTIFICATE MyCert;
UPDATE Table SET EncryptedCol = EncryptByKey(KEY_GUID('MyKey'), @plaintext);
CLOSE SYMMETRIC KEY MyKey;

-- Decrypt
OPEN SYMMETRIC KEY MyKey DECRYPTION BY CERTIFICATE MyCert;
SELECT CONVERT(varchar(max), DecryptByKey(EncryptedCol)) FROM Table;
CLOSE SYMMETRIC KEY MyKey;
```

Key chain: data is encrypted by a symmetric key → that key is encrypted by a certificate → that certificate is encrypted by the database master key → which is protected by the service master key. Everything lives inside SQL Server.

Wrap encryption/decryption in stored procedures so application code never has to know the key names or cert chain.

### `EncryptByKey` output structure

The varbinary `EncryptByKey` returns has a known structure useful for diagnostics:

- Bytes 0–15: GUID of the key (`KEY_GUID`)
- Byte 16: header version (typically `0x01`)
- Bytes 17–19: reserved / padding
- Bytes 20–35: 16-byte IV (random per call by default)
- Bytes 36+: ciphertext + 20-byte SHA-1 integrity hash

Total length is roughly plaintext length + 36–52 bytes of overhead. If you ever need to identify whether a base64-decoded blob is from `EncryptByKey`, the leading 16 bytes look like a GUID followed by `01`.

### The authenticator parameter — substitution attack defense

`EncryptByKey` has an optional 3rd/4th argument set:

```sql
EncryptByKey(KEY_GUID('MyKey'), @plaintext, 1, @authenticator)
```

The authenticator (typically the row's primary key as `varchar`) binds the ciphertext to its row. Without it, an attacker with write access to the encrypted column can swap one row's ciphertext for another's, and `DecryptByKey` will happily return the wrong row's plaintext. Always use the authenticator when row identity matters — which is almost always for sensitive data.

Decrypt with matching authenticator:

```sql
DecryptByKey(EncryptedCol, 1, @authenticator)
```

If the authenticator doesn't match, `DecryptByKey` returns NULL — substitution defense.

### Random IV per call

By default, `EncryptByKey` uses a random IV per encryption. This is good (semantic security) but means:

- Same plaintext encrypts to different ciphertext each call
- Cannot search for ciphertext matches (no equality predicates on encrypted columns)
- Cannot find duplicates within a column (same value → different ciphertext rows)

If you need searchable encrypted columns, use **Always Encrypted with deterministic encryption** instead. There's no straightforward way to make `EncryptByKey` deterministic.

### Base64 wrapping idiom

Stored procs that return encrypted values often base64-encode the varbinary for transport (so the API can return JSON-friendly strings). The portable idiom across SQL Server versions:

```sql
-- varbinary → base64 string
DECLARE @encrypted varbinary(max) = EncryptByKey(KEY_GUID('MyKey'), @plaintext);
SELECT CAST(N'' AS xml).value('xs:base64Binary(sql:variable("@encrypted"))', 'varchar(max)');

-- base64 string → varbinary (input as @b64 varchar(max))
SELECT CAST(N'<x>' + @b64 + N'</x>' AS xml).value('xs:base64Binary((/x)[1])', 'varbinary(max)');
```

Works on every SQL Server since 2005. SQL Server 2022 added native `BASE64_ENCODE` / `BASE64_DECODE` functions which are cleaner if you don't need version portability.

## Always Encrypted

Microsoft's client-side encryption feature. Designed for "decrypt at the latest possible moment" requirements where the DB server itself shouldn't see cleartext.

### Architecture

- **Column Master Key (CMK)** lives outside the DB — Azure Key Vault, Windows Certificate Store, or HSM
- **Column Encryption Key (CEK)** lives in DB metadata, encrypted by the CMK
- The .NET SqlClient driver encrypts on insert and decrypts on read transparently
- DB only ever sees ciphertext

### Enabling it

Connection string flag:

```
Server=...;Database=...;Column Encryption Setting=Enabled;
```

Plus key store provider registration in app code:

```csharp
SqlColumnEncryptionAzureKeyVaultProvider provider =
    new SqlColumnEncryptionAzureKeyVaultProvider(tokenCredential);
SqlConnection.RegisterColumnEncryptionKeyStoreProviders(
    new Dictionary<string, SqlColumnEncryptionKeyStoreProvider>
    {
        { SqlColumnEncryptionAzureKeyVaultProvider.ProviderName, provider }
    });
```

After that, encryption/decryption is automatic.

### Two encryption types

| Type | Searchability | Use when |
|---|---|---|
| **Deterministic** | Equality predicates work (`WHERE col = @val`) | You need to look up rows by encrypted value |
| **Randomized** | No predicates work — column is opaque to queries | You only retrieve the value, never search by it |

Deterministic uses a fixed IV derived from the plaintext + key, so same plaintext → same ciphertext. Slightly weaker than randomized (allows frequency analysis) but enables search.

### Query limitations

Even with deterministic encryption, you can't do:

- `LIKE` patterns on encrypted columns
- Range queries (`>`, `<`, `BETWEEN`)
- Joins on encrypted columns of different types
- Aggregates (`SUM`, `AVG`, `MIN`, `MAX`)
- Full-text search

This is the main reason teams reject Always Encrypted — query limitations bite when the encrypted column is anything you actually want to search on.

## Application-layer encryption

App generates its own keys, encrypts before storing, decrypts after reading. DB just stores ciphertext as `varbinary(max)`.

```csharp
using var aes = Aes.Create();
aes.Key = keyBytes; // From Key Vault, KMS, etc.
aes.GenerateIV();

using var encryptor = aes.CreateEncryptor();
byte[] ciphertext = encryptor.TransformFinalBlock(plaintext, 0, plaintext.Length);

// Store: aes.IV (16 bytes) || ciphertext (and optionally an HMAC for authentication)
```

Most flexible — same pattern works for SQL, Solr, blob storage, Cosmos, anything. Most responsibility — you own all key management.

Use **AES-GCM** if available (`AesGcm` class in .NET 8+) for authenticated encryption rather than AES-CBC + separate HMAC. GCM is simpler and harder to misuse.

## Diagnostic: ciphertext or cleartext?

When an API returns base64-encoded data and you don't know if the bytes after decoding are cleartext or ciphertext:

```csharp
var bytes = Convert.FromBase64String(payload);

// Test 1: UTF-8 readability
var asUtf8 = Encoding.UTF8.GetString(bytes);
Console.WriteLine($"As UTF-8: {asUtf8.Substring(0, Math.Min(200, asUtf8.Length))}");

// Test 2: byte distribution
int printable = bytes.Count(b => b >= 0x20 && b <= 0x7E);
Console.WriteLine($"Printable ratio: {printable * 100.0 / bytes.Length:F1}%");

// Test 3: header structure (EncryptByKey signature)
Console.WriteLine($"First 20 bytes: {BitConverter.ToString(bytes.Take(20).ToArray())}");
```

Interpreting:

- Readable text or JSON in `asUtf8`, printable ratio > 90% → **cleartext**
- Garbage in `asUtf8`, printable ratio 25–40% → **ciphertext** (high entropy)
- First 16 bytes look like a GUID (8-4-4-4-12 hex pattern) followed by `01` → specifically SQL Server `EncryptByKey` ciphertext

## Solr + encrypted fields

Solr can't search inside encrypted content — the inverted index requires tokenizable cleartext. So encryption + search requires hybrid patterns:

- Index searchable metadata (IDs, status, timestamps) as cleartext
- Store sensitive payload encrypted, retrievable but not searchable
- For exact-match search on encrypted fields, use deterministic encryption (Always Encrypted deterministic mode) and encrypt the search term before submitting
- For partial-match or full-text search on sensitive fields, the answer is usually "don't encrypt at the index layer — apply different access controls instead"

## Threat model checklist

Before recommending column encryption, verify which threats it actually addresses for your setup:

| Threat | Column encryption (server-side) helps? | Always Encrypted helps? | App-layer helps? |
|---|---|---|---|
| External attacker with no DB access | ✓ (defense in depth) | ✓ | ✓ |
| DBA reading raw tables | ✓ if cert/key not accessible to DBA | ✓ | ✓ |
| Backup theft / restore to lower env | ✓ if cert chain not in backup | ✓ if CMK not in backup | ✓ |
| Compromised app account with EXECUTE on decrypt proc | ✗ — proc decrypts for them | ✗ if app has CMK access | ✗ if app has key |
| Memory dump of running app process | ✗ | ✗ | ✗ |

Encryption only protects against threats where the attacker doesn't have the decryption path. If your app's DB user has `EXECUTE` permission on the decrypt proc, then anyone who compromises the app has decryption — same as no encryption from that angle.

## Performance considerations

- Stored proc encryption/decryption adds a roundtrip per encrypted operation. For batch-heavy workloads, batch encryption inside a single proc call.
- `EncryptByKey` is fast (AES-256 hardware accelerated on most CPUs) but the `OPEN SYMMETRIC KEY` step has fixed overhead. Open once per session, not per row.
- Always Encrypted has parameter-encryption overhead per query plus client-side CPU. CMK access (Key Vault) caches but the first call pays a network round-trip.
- TDE has near-zero overhead for typical workloads — the encryption happens at the I/O layer with hardware acceleration.

## References

- [SQL Server `EncryptByKey`](https://learn.microsoft.com/en-us/sql/t-sql/functions/encryptbykey-transact-sql)
- [Always Encrypted overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)
- [Always Encrypted with .NET](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/sqlclient-support-always-encrypted)
- [TDE overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption)
- [`System.Security.Cryptography.AesGcm`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.aesgcm)
