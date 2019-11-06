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
good APIs that are simple enough to avoid confusion. This defines how software encryption
APIs will be used in Ursa. Hardware interfaces may be similar but are discussed in a different RFC.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In  this  document,  we  describe  a  philosophy  and  approach  for  interfaces  for  Ursa.   Our
primary goals are twofold:  make an interface that is easy for developers skilled in cryptography
to customize, but difficult for others to misuse.  To that end, we describe two sets of interfaces:

**Developer Interface.**  The external interface will be what the developers who are primarily
focused  on  using  cryptography  (and  are  not  cryptographic  experts)  see.   As  many  details  as
possible will be hidden from these interfaces.
It should be very difficult for people to misuse this layer and create insecure protocols.

**Cryptographer Interface.** This is the internal interface that will be used by cryptographers
(in  all  senses,  both  cryptography  researchers  and  cryptography  engineers).   Interfaces  will  be
flexible, but as descriptive as possible.  In addition, interfaces (and protocols) will be as
compos-
able
as possible.  Our goal is to make it easy for people to choose which underlying cryptosystems
to build their protocols from in this layer.
It will be potentially possible for people to misuse this layer and create insecure protocols,
so we want to advise that only experts tamper with these “hazardous materials.

Authenticated encryption methods are strongly preferred and any other methods should be considered lower level–only used by those who know what they are doing.

## Developer Interface

Our idea for the naming convention for the external interface is purely functionality-based.  We
will start each function with a reminder of the current cryptographic protocol being implemented
in camel case.  So this will be something like PublicKeyEncryptor, for a public key encryption
protocol and Encryptor for symmetric key encryption.
Functions belonging to each type of protocol will be listed after the designation of the protocol
as snake case.  For PublicKeyEncryptor, this would be things like key
gen, encrypt, and decrypt.
So an example interface would look like the following:

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

```rust
mod pk {
    pub enum PublicKeyEncryptorType {
        RSA,
        ElGamal
    }
    
    pub trait NewPublicKeyEncryptor {
        type PublicKeySize = ArrayLength<u8>;
        type PrivateKeySize = ArrayLength<u8>;
        
        fn new(publickey: GenericArray<u8, Self::PublicKeySize>,
               privatekey: GenericArray<u8, Self::PrivateKeySize>) -> Self;
    }
    
    pub trait PublicKeyEncryptor {        
        fn encrypt<A: AsRef<[u8]>(&self, in: A) -> Result<Vec<u8>, Error>;
        fn encrypt_buffer<I: Read, O: Write>(&self, in: A, out: O) -> Result<(), Error>;
        fn decrypt<A: AsRef<[u8]>(&self, in: A) -> Result<Vec<u8>, Error>;
        fn decrypt_buffer<I: Read, O: Write>(&self, in: I, out: O) -> Result<(), Error>;
    }
    
    pub struct Rsa2048OaepSha256 {
        public_key: GenericArray<u8, U294>;
        private_key: GenericArray<u8, U2048>;
    }
    
    impl NewPublicKeyEncryptor for Rsa2048OaepSha256 {
        type PublicKeySize = U294;
        type PrivateKeySize = U2048;
        
        fn new(publickey: GenericArray<u8, Self::PublicKeySize>,
               privatekey: GenericArray<u8, Self::PrivateKeySize>) -> Self {
            Self {
                publickey,
                privatekey
            }
        }
    }
    
    impl PublicKeyEncryptor for Rsa2048OaepSha256 {
        ...
    }
}

mod symm {
    pub struct Encryptor {
        fn new(params: EncryptorParams) -> Self;
        fn key_gen(&self) -> Result<Key, Error>;
    
        fn encrypt<A: AsRef<[u8]>(&self, headers: A, plaintext: A, key: &Key) -> Result<Vec<u8>, Error>;
        fn encrypt_buffer<A: AsRef<[u8], I: Read, O: Write>(&self, headers: A, in: I, out: O, key: &key);
        fn decrypt(&self, headers: Option<&[u8]>, in: &[u8], key: &Key) -> Result<Vec<u8>, Error>;
        fn decrypt_buffer(&self, headers: Option<&[u8], in: &BufReader, key: &Key, out: &BufWriter) -> Result<bool, Error>;
    }
}
```

In this case, PublicKeyEncryptorParams would denote pre-specified configurations of algorithms that a developer could use.  For the external user, this would typically be a single, simple
string and not get into the algorithmic details.  As an example, there could be PublicKeyEncryptorParams::IndyDefault , which a external user would not need to know anything about.  In
addition, this interface would make it very easy for a user to change their implementations–only
the call to the “new” function would need to be modified.

## Cryptographer Interface

Internally,  we  will  have  many  different  algorithms  used  to  implement  the  same  overall  cryp-
tosystems.  For instance, we may have both an LWE-based key exchange as well as a traditional
elliptic curve-based Diffie-Hellman protocol.  To do this,  we will need to use slightly different
naming conventions as well as exploit modularity.
We propose the following naming convention:

`Cryptosystem-algorithm-parameters-function`

In  this  naming  convention,  cryptosystem  describes  the  overall  scheme  being  defined.   For
instance,  things  like  “PublicKeyEncryptor,”  “MAC,”  and  “AUTH-ENCRYPT”  would  all  be
acceptable fields here.  The field “algorithm” describes the exact algorithm or paper that is being
implemented.  For instance, something like “bls-sig” could go here.  “PARAMETERS” describes
all of  the  relevant  parameters  of  the  scheme.   This  includes  things  like  security  parameters,
groups, moduli sizes, and *other dependencies* such as hash functions or other primitives that are
used in the construction of the overall cryptosystem.

The **RustCrypto** team has implemented many traits and structs that are flexible to suite our purposes
for mixing and matching to make different constructions. See [AEAD](https://github.com/RustCrypto/AEADs) 
and [miscreant-rs](https://github.com/miscreant/miscreant.rs) as examples of this.

Note that the interface is almost exactly the same in these examples, but we need fewer parameters for the more
specific function.  In particular, we don’t need to provide a reference to a 128-bit blockcipher
since we would be hard-coding AES into our implementation here.
The  idea  for  encryption  APIs  is  to  have  many  traits  that  can  be  mixed  and  matched  to
compose fixed implementations.

## Between the Two Interfaces

The beauty of this kind of system is the ability to hide the internal interface behind the external
one.  For instance,  the following describes key generation in openSSL: https://github.com/openssl/openssl/blob/master/doc/HOWTO/keys.txt.   RSA  and  elliptic  curve  keys  have  to
be  generated  in  a  completely  different  way.   This  is  something  that  we  do  not  want  to  have
burden our blockchain developers, who may not be comfortable enough with cryptography to
do something like this.
The only difficult thing we are pushing to the general developers is the “params” when calling
a new function.  However, these can be explicitly defined in simple ways by cryptographers.  For
instance,  we  used  the  example  “INDY-DEFAULT”  earlier.   Someone  could  develop  a  set  of
parameters that are the default for Indy, define them, and allow everyone else to use them in a
simple call.
When the “new” function is called on the external interface, it will use the “params” to pick
out which protocols from the internal interface to use and create an appropriate instance.  Since
we can store state in our *self* object,  we can keep track of the parameters here.  This way, a
developer that wants to change between different algorithms only has to modify the “params” on the cryptosystem description.

## Advanced Signature Interface
We next describe interfaces for our advanced signatures.

### Threshold Signatures

Our definitions are loosely based on those from [BGG+18](https://eprint.iacr.org/2017/956.pdf) and [ACC+2019](https://eprint.iacr.org/2019/1058.pdf)

```rust
pub struct ThresholdSignature {
fn new(params: TSParams) -> Self;
fn key_gen(&self) -> Result< (TSPublicParams, MasterVerifyingKey, Vec[KeyPair]),
Error>;         //needs to return an array of keys
fn part_sign(&self, sk: &SigningKey, message: &[u8]) -> Result<Vec<u8>, Error>;
fn part_verify(&self, vk: &VerifyingKey, message: &[u8], signature: &[u8]) -> bool;
fn combine(&self, pp: &TSPublicParams, SignatureList: &[u8; Self:ThresholdSize])
-> Result<Vec<u8>, Error>;
fn verify(&self, mvk: MasterVerifyingKey, message: &[u8], signature: &[u8])
-> bool // just reject failed attempts no matter the cause.
}
``` 

### Multisignatures

Our definitions in this section follow those of [BDN18](https://eprint.iacr.org/2018/483.pdf)

```rust
pub struct MultiSignature {
fn new(params: MSParams) -> Self;
fn key_gen(&self) -> Result<KeyPair, Error>; //same syntactically as regular sig
fn sign(&self, sk: &SigningKey, message: &[u8]) -> Result<Vec[u8], Error>;
//same as regular sig
fn aggregate(&self, VKList: &[VerifyingKey], SigList:&[u8]) -> Result<Vec<u8>, Error>;
fn verify(&self, VKList: &[VerifyingKey], message: &[u8], signature: &[u8]) ->
bool; //don’t return error, just reject.
}
```

If the multi-signature also supports public key aggregation, it can also extend the above struct
with the following functions.

```rust
fn agVK_gen(&self, VKList: &[VerifyingKey]) -> Result<agVerifyingKey, Error>;
fn agVK_verify(&self, agVK: agVerifyingKey, message: &[u8], signature: &[u8]) -> bool;
```

## Provider Architecture and Implementations

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
