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

Ursa should distinguish between two methods for encryption: one for asymmetric algorithms
like RSA, and one for symmetric like AES. However the interfaces will be very similar.

[Libsodium](https://libsodium.gitbook.io/doc/) for example uses these two types as public-key encryption and secret key encryption.

Public key encryption is generally applied for key exchange or secret key wrapping. It requires two different keys–*public* and *private*, linked together with a one way mathematical
relationship. The *private* key is used to decrypt a message and cannot be recovered from the *public* key and MUST remain private. The *public* key is used to encrypt a message.

Secret key encryption uses the same key to encrypt and decrypt. It is only shared among all involved parties as a *shared secret key*.

Users will create keys using one of these two types and pass them to the appropriate method.
Rust typing will prevent a user from passing a secret key to a public key encryption method and visa versa.

Public-Key encryption methods MUST use the naming convention **FUNCTION-SCHEME-PARAMETERS**. Below are some examples
that could be implemented:

- Rsa2048Oaep
    - Encryption with an RSA 2048 bit public key using [Optimal Asymmetric Encryption](https://tools.ietf.org/html/rfc8017#section-7.1).
    - Decryption with an RSA 2048 bit private key.
    - This method does not use a MAC or authentication tag.

Secret-key encryption methods MUST use the naming convention **FUNCTION-SCHEME-PARAMETERS**. Below are some examples
that could be implemented:

- Aes256Gcm
- Chacha20Poly1305
- Twofish256CbcHmacSha2_256
- Blowfish448HmacSha2_256

Secret-key encryption methods generally require the use of *nonces* or *initialization vectors (iv)*, the two terms are often used interchangeably.
We use the term *nonce* here. Nonces must be unique for a given key and message pair for certain encryption schemes to be prevent certain types of attacks.
To simplify the interfaces for Ursa users, *nonce* generation will be handled by Ursa and not a required concern for developers.
A lower level layer can expose this for more advanced crypto consumption like key homomorphic PRFs where nonce reuse might be desirable.

Authenticated encryption methods are strongly preferred and any other methods should be considered lower level–only used by those who know what they are doing.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All proposed code will be implemented in **libursa**.
**Libursa** will use existing traits and structures where possible.
`KeyGenOption`, `PublicKey`, `PrivateKey`, `SecretKey`, and `MacKey`
are already implemented.

Rust structs and traits will look and work like so:
```rust
pub trait PublicKeyEncryptionScheme {
    fn new() -> Self;
    fn keypair(&self, options: Option<KeyGenOption>) -> Result<(PublicKey, PrivateKey), CryptoError>;
    fn encrypt(&self, plaintext: &[u8], pk: &PublicKey) -> Result<Vec<u8>, CryptoError>;
    fn decrypt(&self, ciphertext: &[u8], sk: &PrivateKey) -> Result<Vec<u8>, CryptoError>;
    fn private_key_size() -> usize;
    fn public_key_size() -> usize;
}

pub trait SecretKeyEncryptionScheme {
    fn new() -> Self;
    fn genkey(&self, options: Option<KeyGenOption>) -> Result<SecretKey, CryptoError>;
    fn encrypt(&self, plaintext: &[u8], sk: &SecretKey) -> Result<Vec<u8>, CryptoError>;
    fn decrypt(&self, ciphertext: &[u8], sk: &SecretKey) -> Result<Vec<u8>, CryptoError>;
    fn key_size() -> usize;
    fn tag_size() -> usize;
}
```

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

# Prior art
[prior-art]: #prior-art

[Rust Crypto](https://crates.io/crates/rust-crypto) implements a number of crypto primitives but is not for the faint of heart.
It also does not allow for optionally compiling selected algorithms. It doesn't cover other algorithms that are under consideration to be added.

[Libsodium](https://libsodium.gitbook.io/doc/) only utilizes Ed25519 curves.

[Openssl](https://www.openssl.org) is a library that has many cryptographic algorithms but is written in "C".

# Unresolved questions
[unresolved]: #unresolved-questions

- Is there a better naming scheme that we can follow for the algorithms?

# Changelog
[changelog]: #changelog

- [18 Feb 2019] - v2 - Incorporate changes from Lovesh and HartM
- [6 Feb 2019] - v1 – Initial draft
