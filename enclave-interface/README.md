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
3. Enterprise HSMs (Thales, Iron Core Labs, nCipher, Utimaco, Securosys. May be network connected, multi-user and redundent configurations, complex key management with roles and access policies. Logging and auditing requirements)
4. Cloud key management (Microsoft Azure KeyValut, Amazon AWS KMS, Hashicorp Vault, Box KeySafe)
5. Trusted Execution Environments (Intel SGX, Microsoft CoCo, ARM Trustzone, RISC-V Multivisors, Unbound Key Control)

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
to protide attestation commands that return provider-specific information which can be verified out-of-band.

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
the connection create process takes a configuration unique to that enclave.

We show all code samples written as Rust but can be adapted to other languages as desired.

Most hardware implementations will not allow key material to be passed through the API, even in encrypted form, so a system of external references is required that allows keys to be referenced in a way that supports:
1. **Consistency** - The same ID refers to the same key every time (according to the key management paradigm of the underlying hardware)
1. **Naming schemes** - Organizations often name their keys according to some formal naming scheme - such as **legal.europe.documetencryption.May2019-1**.
1. **Key blocks** - Some HSMs not only allow but actively require keys to be passed as formatted and encrypted keyblocks.  In this case we not only need to support the simple data type of a binary key block but also (potentially) identify which KEK it's encrypted under.

In keeping with the drive for Ursa to be simple and hard to mess up, the proposal is to make KeyIDs in the Ursa interface be simple UTF-8 string names, and leave the underlying provider implementation to deal with the complexities of translation, key rollover, duplication and so on.  Keyblocks will not be supported until really demanded.

The generic trait that all provides the specific APIs is the trait `EnclaveLike`.
```rust
pub trait EnclaveLike {
    fn connect(connector: EnclaveConnector) -> Result<Self, Error>;
    fn close(self);
    fn capabilities(&self) -> Result<Vec<EnclaveCapabilities>, Error>;
    fn create_key(&self, options: EnclaveCreateKeyOptions) -> Result<EnclaveMessage, Error>;
    fn delete_key(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    fn execute(&self, message: EnclaveOperation) -> Result<EnclaveMessage, Error>;
    fn export_wrapped(&self, wrap_key_id: String, key_id: String) -> Result<EnclaveMessage, Error>;
    fn get_key_info(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    fn get_public_key(&self, key_id: String) -> Result<EnclaveMessage, Error>;
    fn get_random(&self, bytes: usize) -> Result<EnclaveMessage, Error>;
    fn import_wrapped(&self, wrap_key_id: String, wrap_message: EnclaveImportRequest) -> Result<EnclaveMessage, Error>;
    fn list_keys(&self) -> Result<EnclaveMessage, Error>;
    fn ping(&self) -> Result<EnclaveMessage, Error>;
    fn put_key(&self, message: EnclaveMessage) -> Result<EnclaveMessage, Error>;
    fn reset(&self) -> Result<EnclaveMessage, Error>;
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
    AwsKms(AwsKmsConnector)
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

Once a `EnclaveConnector` has been defined, the connection is created. The connection enables the 

# Drawbacks
[drawbacks]: #drawbacks
For developers intimately familiar with enclave programming, this API may be a more expensive/complicated scheme than
using the methods they are already used to using. Care must be taken so this scheme does not change frequency with each
enclave provider that is to be supported.

# Rationale and alternatives
[alternatives]: #alternatives



# Prior art
[prior-art]: #prior-art
This provider model is extremely common in the crypto world with implementations like [PKCS11](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html)
Microsoft CNG, Java JCA, [Parsec](https://github.com/parallaxsecond/parsec), and others. 


# Unresolved questions
[unresolved]: #unresolved-questions
