- Feature Name: (mal000001, encryption-api)
- Start Date: 6-Feb-2019
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 1

# Summary
[summary]: #summary

Encryption is considered a primary use for cryptography which Ursa does not
currently provide. There is demand for providing easy to use encryption/decryption
interfaces in Ursa. Ursa already has similar interfaces for signing/verification.
The implemented algorithms provide programmers choices that aim
to avoid bad parameter interactions. Keys are passed around as data blobs
allowing consumers to protect them as they see fit.

# Motivation
[motivation]: #motivation

Hyperledger projects will need to be able to encrypt/decrypt data using
industry standards. To minimize mistakes and allow other projects good
encryption schemes, Ursa will provide known good implementations with
good APIs that are simple enough to avoid confusion.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This is explained in greater detail in the [LateX Document](main.tex)

Authenticated encryption methods are strongly preferred and any other methods should be considered lower level–only used by those who know what they are doing.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All proposed code will be implemented in **libursa**.
**Libursa** will use existing traits and structures where possible.
`KeyGenOption`, `PublicKey`, `PrivateKey`, `SecretKey`, and `MacKey`
are already implemented.

This is explained in greater detail in the [LateX Document](main.tex)

# Rationale and alternatives
[alternatives]: #alternatives

Other libraries tie encryption methods to key objects instead of algorithm objects.
The disadvantage of this model becomes apparent when multiple keys must be used for encryption
and when safeguarding them.

The key object model requires creating a new object then calling the encryption on the object.
Our model considers keys as data blobs and only need one instance of an encryption scheme which allows any
key to be passed without creating a new instance.

Safeguarding should be easier where the key is a data blob that can be stored in secure locations.
Key objects tend to encode more than just data when serialized.

Other crypto libraries like libsodium and secp256k1 also follow this model with openssl beginning to adopt it as well.

# Unresolved questions
[unresolved]: #unresolved-questions

- Is there a better naming scheme that we can follow for the algorithms?

# Changelog
[changelog]: #changelog

- [2 Apr 2019] - v1 – Initial draft
