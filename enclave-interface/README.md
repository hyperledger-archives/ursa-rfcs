# - Feature Name: enclave-interface
- Start Date: 2020-02-11
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 0.1
- Contributors: Mike Lodder, Jon Geater, Mick Bowman, Carsten Stöcker, Hart Montgomery, Brent Zundel

# Summary
[summary]: #summary
Hardware security modules(HSM)/Trusted Execution Environments(TEE)/Secure enclaves are becoming more common for specialized cryptography key management.
This RFC describes a common API that can be used for interacting with enclaves in Ursa. This API allows for
HSM/TEE/Enclave providers to be plug and play from a consumer's perspective. This model allows for the programming interface to be
fully harmonised but the implementation can be swapped out at the direction of the application user.


# Motivation
[motivation]: #motivation
Using enclaves is difficult. Each provider has unique methods for performing cryptographic operations.
It is the primary aim to abstract away many of the implementation details from developers with the goal of programming an enclave to be easy to 
do and misuse resistant. Developers are not usually cryptographic nor key management experts so this API should be
very difficult for people to misuse and create insecure protocols. Ursa will provide a single, central crypto interface
for all to adopt while allowing multiple implementations of that interface that embody different operational or non-functional
characteristics such as instrumentation, tuned performance, or hardware protection.
A good implementation of this pattern should allow different users of the same 
compiled code to change the crypto implementation with no ill-effect, and no conditional code in the application itself.

So why another interface? [PKCS11](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html)
Microsoft CNG, Java JCA, [Parsec](https://github.com/parallaxsecond/parsec), and others do not handle cross platform,
extensible crypto like Ursa does. For example, HSMs do not currently support BLS signature keys and it is desirable for
applications to take advantage of using hardware-backed or virtual implementations.
Therefore, services that want to use an enclave that have special crypto needs will be able to do so when using Ursa.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The types of hardware-backed or virtual implementations envisaged are:

1. Simple hardware-accelerations (AES-NI, HW RNGs, FPGAs)
2. Personal HSMs (Yubikey, Ledger, smart card. Typically locally connected, single instance, simple integrated key management,
OS keyrings like Mac OS X Keychain, Gnome-Keyring, KWallet, Windows Credential Manager)
3. Enterprise HSMs (Thales, Iron Core Labs, nCipher, Utimaco, Securosys, Jitsuin.
    May be network connected, multi-user and redundent configurations, complex key management with roles and access policies. Logging and auditing requirements)
4. Cloud key management (Microsoft Azure KeyVault, Amazon AWS KMS, Hashicorp Vault, Box KeySafe)
5. Trusted Execution Environments (Intel SGX, ARM Trustzone, RISC-V Multivisors, Unbound Key Control)

Use of each different type of implementation comes with its own unique set of security questions. Superficially a user
may simply want to know **"is my crypto code running in a secure environment or hardware?"**, but for true security
it is necessary to verify many more of the details:

1. Where did this enclave come from? How was it initialized?
1. What's the administrative status of this service? Who has authority to change the code/firmware/configuration?
1. Can changes to code handling the crypto be detected?
1. How are keys handled? How were they generated? Who has authority to make backups or export or change them?
1. Are whole protocol blocks implemented atomically in the enclave or just individual primitives like encryption?
1. How to prove the enclave is really being used and not bypassed?

It is very important not to expose the details of the implementation through the API itself: it gets very messy, very quickly
to try to accommodate all the small details of each potential implementation in a generic API. Instead, it is preferable
to provide attestation commands that return provider-specific information which can be verified out-of-band.

## Why can't we just invisibly use the encalve we find on our system?

Because, with the possible exception of things like AES-NI or OS TRNG, that's just not possible.
There may be multiple hardwares available, or they may not be discoverable, or they may require out-of-band explicit handling to make them work, or ... (many other reasons).

## Provider Loading, Program Context, and Session Management

As soon as a library grows connections or dependencies outside its static code base it becomes necessary to burden it with concepts like context and sessions.
At the very least this is required in order to hold on to things like network connections or other open ports in the case of discrete hardware, but other considerations are:

1. Login information or live authenticated sessions
1. Use of keys that have extra authorization requirements for use (eg password, MFA)
1. Loadbalancing/failover state

It should be possible to hide most of this in the provider implementation (putting the burden on the developers and not the user) but inevitably some of this contextual state will need to be carried through the API calls.

One particular thing to consider is whether we support multiple implementations to be accessed concurrently by the same application.
Consider the case of a Chinese or Russian implementer who needs all the standard core crypto for basic Hyperledger operation but also require GOST or SMx family algorithms for certain purposes.
There are 4 possible naive routes to take here: 
1. Do not support sovereign or other 'special' algorithms in Ursa
1. Require all Ursa providers to implement all algorithm families
1. Allow (potentially) multiple providers to be loaded, and have **implicit** selection of which to use depending on the call made
1. Allow (potentially) multiple providers to be loaded, and have **explicit** selection of which to use depending on the knowledge of the developer

The first three of these are technically feasible but not practically useful.
Ursa will adopt approach #4 and provide an explicit initialization function that specifies which provider is to be used,
and initializes a context object that the application must manage and maintain.
This is, again, in keeping with popular mature crypto architectures such as PKCS11, JCA, and CNG.
The precise manifestation of this will depend on the language bindings for the particular application.

## Provider requirements and constraints

Contexts and sessions should **safely** zero out sensitive data when used in memory or disk used to limit side channel attacks like
malicious network and snapshot adversaries or honest but curious system administrators.

While the main purpose for having a provider architecture is to enable difference between different implementations, it is important to also enforce a significant degree of sameness.
The following constraints are suggested:
1. All providers must provide an implementation of the entire Ursa core interface
1. While a provider may support interactive or 2FA authentication for key use, they must offer a configuration that makes Ursa operation non-interactive
1. All providers must observe data format compatibility in any data element of the published APIs. Assuming the same key is available to both providers, then the output of one provider shall be consumable as input to any other - for example, a signature from provider A shall be verifiable in provider B, and data encrypted in provider B shall be decryptable in provider C.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Before any cryptographic operations can occur, programmers must have a connection to the enclave. Enclave connections
come in one of the following forms:

1. Software Development Kits (SDK) or provider libraries. (SGX, PSP, Trustzone, Mac OS X Keychain)
1. Physical connections (USB)
1. Networked (HTTP, ethernet, bluetooth)

Therefore initial piece of the interface is the connection. Given each provider has different connection methods, 
the connection create process takes a configuration unique to that enclave. The goal of this API design is to minimize
changes to it to maintain compatibility.

We show all code samples written as Rust but can be adapted to other languages as desired.

Most hardware implementations will not allow key material to be passed through the API, even in encrypted form, so a system of external references is required that allows keys to be referenced in a way that supports:
1. **Consistency** - The same ID refers to the same key every time (according to the key management paradigm of the underlying hardware)
1. **Naming schemes** - Organizations often name their keys according to some formal naming scheme - such as **legal.europe.documentencryption.May2019-1**.
1. **Key blocks** - Some HSMs not only allow but actively require keys to be passed as formatted and encrypted keyblocks.  In this case we not only need to support the simple data type of a binary key block but also (potentially) identify which KEK it's encrypted under.

In keeping with the drive for Ursa to be simple and hard to mess up, the proposal is to make KeyIDs in the Ursa interface be simple UTF-8 string names, and leave the underlying provider implementation to deal with the complexities of translation, key rollover, duplication and so on.  Keyblocks will not be supported until really demanded.

The generic trait that all provides the specific APIs is the trait `EnclaveLike`.
```rust
pub trait EnclaveLike {
    /// Create connection to enclave, returns a session/context useful
    /// for issuing commands to the enclave
    fn connect(connector: EnclaveConnector) -> Result<Self, Error>;
    /// Terminate the current session/context
    fn close(self);
    /// Get a list of capabilities supported by this enclave
    fn capabilities(&self) -> Result<Vec<EnclaveOperation>, Error>;
    /// Create a new enclave key. Settings and key type and supported operations are 
    /// defined in the `EnclaveCreateKeyOptions`. 
    fn create_key(&self, options: EnclaveCreateKeyOptions) -> Result<EnclaveMessage, Error>;
    /// Delete an enclave key. This will error if current session is not allowed 
    fn delete_key(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    /// Perform a cryptographic operation like encryption, signing,
    /// export wrapped, import wrapped, generate attestation, or other crypto algorithms
    /// defined in the `EnclaveOperation`.
    fn execute(&self, message: EnclaveOperation) -> Result<EnclaveMessage, Error>;
    /// Get information about the specified "key_id". Such information is
    /// key type (Ed25519, RSA, BLS), operations allowed, created date,
    /// valid until date, authorized users
    fn get_key_info(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    /// Return a public key given a private key with "key_id"
    fn get_public_key(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    /// Return "bytes" random data bytes generated in the enclave
    fn get_random(&self, bytes: usize) -> Result<Vec<u8>, Error>;
    /// Return a list of all key_ids the current session is allowed to use
    fn list_keys(&self) -> Result<EnclaveMessage, Error>;
    /// Ping the enclave to keep the session/context alive. Returns an error if the session/context is closed
    /// or otherwise unavailable
    fn ping(&self) -> Result<EnclaveMessage, Error>;
    /// Put a raw key into the enclave. Similar to `create_key` except the key was generated outside the enclave.
    /// Usually this is used for inserting public or symmetric keys
    fn put_key(&self, message: PutKeyOptions) -> Result<EnclaveMessage, Error>;
    /// Reset the enclave to a factory default state, clearing all stored objects and restoring the default auth key is applicable.
    /// WARNING: This wipes all keys and other data (logs, policies) from the enclave. 
    fn reset(&self) -> Result<EnclaveMessage, Error>;
    /// Useful for non crypto related queries like get audit logs, get storage capacity, or device info as well as
    /// creating other credentials, changing permissions and policies.
    fn send_message(&self, message: InputEnclaveMessage) -> Result<EnclaveMessage, Error>;
}
```

Each connection is provided by the `EnclaveConnector` enumeration. The enumeration defines the connection type.
This list is not exhaustive but is defined enough for the reader to get a general idea of how to add new connection types.
This is where the specifics about the enclave must be known by the developer.
```rust
pub enum EnclaveConnector {
    OsKeyRing(OsKeyRingConnector),
    YubiHsm(YubiHsmConnector),
    SGX(SGXConnector),
    Trustzone(TrustzoneConnector),
    AwsKms(AwsKmsConnector),
    AzureKeyVault(AzureKeyVaultConnector),
    HashicorpVault(HashicorpConnector)
}
```

The `OsKeyRingConnector` contains the options for connecting to the OS Keyring which may be backed by a hardware
enclave. Each of these fields may be empty which means the OS will use the default keyring and prompt the consumer
for a username and password. 

```rust
pub struct OsKeyRingConnector {
    path: Option<String>,
    username: Option<String>,
    password: Options<String>
}
```

The `YubiHsm` connector contains the options for connecting to a portable Yubikey which may be networked or connected
via USB. Other methods might be provided by the [Yubico SDK](https://developers.yubico.com/YubiHSM2/Releases/)

```rust
pub struct Credentials {
    authentication_key_id: u16,
    authentication_key: Key, //derived from PBKDF2 or symmetric key
}

pub struct UsbConnector {
    address: u8,
    bus_number: u8,
    credentials: Credentials,
    context: UsbContext,
    product_name: String,
    serial_number: u32,
    timeout_ms: u64,
}

pub struct HttpConnector {
    address: String, //IP address or DNS name
    credentials: Credentials,
    port: u16,
    timeout_ms: u64
}

pub enum YubiHsmConnector {
    Usb(UsbConnector),
    Http(HttpConnector)
}
```

Once a `EnclaveConnector` has been defined, the connection is created. The connection enables all other operations.
Enclave providers must be queryable for capabilities which can then be used by the caller to determine what its allowed
to use. Common capabilities with existing enclaves are in the following rust code. These are not mutually exclusive
```rust
pub enum EnclaveOperation {
    Attestation(AttestationParams),
    GenerateAsymmetricKey(GenerateAsymmetricParams),
    GenerateSymmetricKey(GenerateSymmetricParams),
    GenerateRandom(RandomParams),
    KeyAgreement(AgreementParams),
    Sign(SigningParams),
    Verify(VerifyParams),
    Encrypt(EncryptParams),
    Decrypt(DecryptParams),
    DeleteKey(DeleteKeyParams),
    WrapKey(WrapParams),
    UnwrapKey(UnwrapParams),
    ExportWrappedKey(ExportWrappedParams),
    ImportWrappedKey(ImportWrappedParams),
    KeyInfo(KeyInfoParams),
    Audit(AuditParams),
    Log(LogParams),
    DeviceInfo
}

/// Key agreement derivation parameters
pub enum AgreementParams {
    /// Key agreement using PKCS3
    Pkcs3(Pkcs3Params),
    /// Key agreement using ECDH
    Ecdh(EcdhParams),
    /// Key agreement using Post-Quantum algorithms
    Pq(PostQuantumParams)
}

/// PKCS#3 parameters according to v1.4
pub struct Pkcs3Params {
    /// Prime, must be 2048, 3072, 4096, 6144, or 8192
    p: Pkcs3DhP,
    /// Base
    g: BigNum,
    /// This enclave's key id
    id: String,
    /// Mask generating function
    mgf: Pkcs3Mgf,
    /// Other's public key
    peer: BigNum,
}

/// Pkcs3 diffie hellman prime
pub struct Pkcs3DhP {
    value: BigNum
}

/// Valid mask generating functions for Pkcs3
pub enum Pkcs3Mgf {
    Sha224,
    Sha256,
    Sha384,
    Sha512
}

/// Elliptic Curve Diffie Hellman parameters
pub struct EcdhParams {
    /// The curve to use
    curve: EccCurve,
    /// This enclave's key id
    id: String,
    /// Mask generating function
    mgf: EcdhMgf,
    /// Other's public key as an uncompressed point for curves that support compressed points
    /// typically is 57, 65, 97, 129, 133, 193 bytes
    peer: EcPoint
}

/// Valid elliptic curves see https://eprint.iacr.org/2018/193.pdf 
/// and https://eprint.iacr.org/2005/133.pdf
/// for information about BN and BLS curves
pub enum EccCurve {
    /// Curve25519
    Curve25519,
    /// Curve41417
    Curve41417,
    /// Curve448 AKA Goldilocks 
    Curve448,
    /// Barreto-Lynn-Scott embedding degree 12 with prime field 381 bits
    Bls381,
    /// Barreto-Lynn-Scott embedding degree 12 with prime field 461 bits
    Bls461,
    /// Barreto-Lynn-Scott embedding degree 24 with equivalent security of AES-192
    Bls24,
    /// Barreto-Lynn-Scott embedding degree 48 with equivalent security of AES-256 with 581 bit prime
    /// see https://tools.ietf.org/id/draft-kato-threat-pairing-01.html#rfc.section.3
    Bls48,
    /// Barreto Naehrig with prime field 254 bits. About 100 bits of security
    Bn254,
    /// Barreto Naehrig with prime field 256 bits. See https://cryptojedi.org/papers/dclxvi-20100714.pdf
    /// and https://moderncrypto.org/mail-archive/curves/2016/000740.html for more details
    BnP256,
    /// Barreto Naehrig with prime field 384 bits. See https://cryptojedi.org/papers/dclxvi-20100714.pdf
    BnP384,
    /// Barreto Naehrig with prime field 512 bits as specified in ISO15946-5
    BnP512,
    /// Barreto Naehrig with prime field 638 bits as specified in
    /// https://fidoalliance.org/specs/fido-v2.0-rd-20180702/fido-ecdaa-algorithm-v2.0-rd-20180702.html#supported-curves-for-ecdaa
    BnP638,
    /// NIST P-224 (secp224r1)
    EcP224,
    /// NIST P-256 (secp256r1, prime256v1)
    EcP256,
    /// NIST P-384 (secp384r1)
    EcP384,
    /// NIST P-521 (secp521r1)
    EcP521,
    /// secp256k1
    EcK256,
    /// Brainpool 224
    EcBP224,
    /// BrainPool 256
    EcBP256,
    /// BrainPool 384
    EcBP384,
    /// BrainPool 512
    EcBP512,
}

/// Valid mask generating functions for Ecdh
pub enum EcdhMgf {
    Sha2_224,
    Sha2_256,
    Sha2_384,
    Sha2_512,
    Sha3_224,
    Sha3_256,
    Sha3_384,
    Sha3_512,
    Blake2_224,
    Blake2_256,
    Blake2_384,
    Blake2_512,
    Blake3_256,
}
```

Some capabilities offer multiple options like `DeriveDiffieHellman` which can be either PKCS#3 DHParameter structure
or Elliptic-Curve based.

This model allows the flexibility to add new operations and types without changing the APIs.

Each operation will return an Enclave Message that contains the results or an error upon failure. For example, 
Encryption will return the ciphertext and possibly an authentication tag. Signing returns the signature.

# Drawbacks
[drawbacks]: #drawbacks
For developers intimately familiar with enclave programming, this API may be a more expensive/complicated scheme than
using the methods they are already used to using. Care must be taken so this scheme does not change frequency with each
enclave provider that is to be supported.

Another possibility is that this approach is too flexible and requires intimate knowledge about crypto algorithms.
To mitigate this, predefined ciphers can be created for end consumers like RSA-3072-PSS-SHA256 or
AES-256-GCM, AES-128-CBC-HMAC-SHA256, XCHACHA20-POLY1305, ED25519, ECDSA-SHA256, or ECIES-SHA512-AES-GCM.
This reduces algorithmic agility that is an inherent problem
with many cryptographic libraries. 

# Rationale and alternatives
[alternatives]: #alternatives

# Prior art
[prior-art]: #prior-art
This provider model is extremely common in the crypto world with implementations like [PKCS11](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html)
Microsoft CNG, Java JCA, [Parsec](https://github.com/parallaxsecond/parsec), and others. 


# Unresolved questions
[unresolved]: #unresolved-questions
