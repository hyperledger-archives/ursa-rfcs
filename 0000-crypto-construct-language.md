 Feature Name: crypto_construct_language
- Start Date: 2020-01-29
- RFC PR:
- Ursa Issue:
- Version: 0

# Summary [summary]: #summary

This RFC describes a new feature, inspired by Bitcoin script, for serializing
all cryptographic constructs as a series of data and command tokens that
describe the use/application of the construct. For instance, a secret key would
be serialized as an encrypted key with commands for unwrapping the key. Anybody
wanting to access the secret key can execute the script and if they provide the
proper inputs, the result will be the unwrapped key. This applies for all
cryptographic constructs from simple constructs such as key storage up to the
most complex such as multi-factor key rotation schemes.

# Motivation [motivation]: #motivation

By storing cryptographic constructs in "functional" form, we abstract away the
underlying cryptographic privitive/operation providers as well as provide a
standardized language for describing arbitrarily complex cryptographic
constructs in serialized form (e.g. DID document, blockchain transaction
records, encrypted packets, etc).

This came from the DID document standardization effort when we ran into the
need for storing complex cryptographic constructs for different controller
binding and key rotation operations. Those operations typically have one or
more ways that a controller can prove control over an identity and/or force a
rotation away from a compromised key. This RFC seeks to standardize the
language we use to describe those constructs so that regarless of the
underlying crypto library, the construct can be recreated and the operation
executed.

This also came from the did:git DID method spec effort where we ran into the
need for describing cryptographic constructs for enforcing governance rules for
a repo. An example rule would be: changes to the MAINTAINERS file requires a
commit that is signed by at least 2 of the maintainers of the project. The
rules need to be serialized in a form that is machine parsable and machine
executable with context specific references to files and commits to read data
used in the construct creation. This RFC again seeks to standardize the
language we use to describe those rules.

To further expand on the topic of enforcing an M-of-N signatures of existing
maintainers for any commits that change the MAINTAINERS file in a repo, the
follow example shows how it could be accomplished. The MAINTAINERS file can
contain a number of different CCLang scripts for the governance of that file
itself. Or the rules can be stored in another GOVERNANCE file. It doesn't
matter.

Let's assume the CCLang M-of-N check script is stored in the MAINTAINERS file.
It assumes that the list of maintainer public keys will first be pushed onto
the stack before it gets executed and it just adds up the number of maintainer
public keys and tests to see if that sum is greater-than-or-equal to the M
threshold.

Now, if the CCLang parallel multi-sig on the commit is written so that it
leaves a copy of the validating public key on the stack for every valid digital
signature, then to enforce M-of-N, the commit hook script just needs to append
the M-of-N CCLang script from the MAINTAINERS file to the end of the CCLang
multi-sig in the commit and then execute.

The first half will be the CCLang multi-sig from the commit and it will leave
the stack with public keys for all of the valid signatures. Then the second
half will be the M-of-N check script that checks each public key against the
list of public keys from the MAINTAINERS file and adds up the number of matches
and compares that for >= against the M threshold. If, at the end, the stack has
`TRUE` left then the commit passes the M-of-N multi-sig of maintainers check.
If the stack is left with `FALSE` it does not pass the check and the commit
should be rejected.

If a code repo requires that all commits be signed by an identity already
stored in the repo itself and all of the commit hooks enforce the cryptographic
checks using CCLang, and if all of the cryptographic check CCLang scripts are
also stored in files in the repo, then the entire process is self-certifying
and guaranteed to be correct.

# Guide-level explanation [guide-level-explanation]: #guide-level-explanation

The crypto constructs language (CCLang) is used to serialize all cryptographic
constructs and uses operations that map directly to the cryptographic
algorithms and operations that Ursa offers. Ursa abstracts away multiple
implementations of different algorithms (e.g. SHA256, RSA encryption, etc). At
compile time the developer chooses which implementation they want included in
the final Ursa binary. CCLang is an extension of the Ursa API by encoding Ursa
API calls as a series of data and tokens that direct the use of the Ursa API.

CCLang is inspired by Bitcoin script and is a reverse-polish notation (RPL)
scripting language design to execute on a virtual stack machine. CCLang scripts
contain data tokens, argument tokens and operation tokens to serialize the
steps needed to execute a specific cryptographic operation (e.g. verify a
digital signature, unwrap a secret key, validate a key rotation operation,
etc). Anywhere a developer would normally encode data used for a cryptographic
operation they would instead store a CCLang script.

It is important to state that CCLang's scope is limited to cryptographic
operations and is not intended to support other operations such as contacting
oracles, network lookups, or anything else that wouldn't be handled by an
existing cryptographic library. It is intended to normalize how we all share
complex cryptographic constructs through serialized form (e.g. network packets,
DID documents, etc).

## Some Examples

I think the best way to really understand how CCLang is different is to look at
some comparisons between existing serialization schemes and how the same
construct would be serialized in CCLang. Don't worry about understanding all of
the tokens in these examples because they will be described in full detail in
the reference section below.

### Key Material

The most basic of cryptographic constructs is storing a public key. In the
proposed DID document specification a public key is serialized in JSON format
with the encoding type in the key:

```
{ "type": "Ed25519", "hexkey": "0a7d1d784358af1f8073ba07eb5ae2fc7272a860ec4547de8bc13d04259cd59a" }
```

Secure Scuttlebutt does something different. They use a sigil character to
identify a public key---`@`---and then add a suffix to define the algorithm the
key is intended to be used with:

```
@Cn0deENYrx+Ac7oH61ri/HJyqGDsRUfei8E9BCWc1Zo=.ed25519
```

In CCLang, a public key alone doesn't need any decoration. It is just stored as
the key data. The serialization changes however if the key must be encoded for
the given format. When storing in a text format, public keys are typically
encoded using hexidecimal characters or a binary-to-text translation scheme
like Base64 or Base58. So in a text format, a Base64 encoded public key in
CCLang form would look like this:

```
Cn0deENYrx+Ac7oH61ri/HJyqGDsRUfei8E9BCWc1Zo= Base64 DECODE
```

The first token is the Base64 encoded public key data followed by the
identifier for the encoding scheme (e.g. "Base64") and lastly an opcode
executing the decode operation. Since CCLang is processed using a stack machine
this is an RPN representation of the operation to decode the key into bytes.

To execute this, the tokens are processed left-to-right. First the encoded key
data is pushed onto the stack. Then the identifier specifying the
text-to-binary scheme is pushed onto the stack. The DECODE opcode pops the
encoding scheme identifier and the encoded bytes off of the stack, executes the
correct decoding function and then pushes the resulting bytes onto the stack.
Any software that understands CCLang would execute this script to get the
public key bytes in memory ready for use in other operations.

## Digital Signatures

Digital signatures are when we start getting into slightly complex
cryptographic constructs. The most common way to create a digital signature is
to first hash the data being signed and then using a nonce with public key
cryptography to encrypt the hash of the data. The resulting digital signature
is usually the combination of the original data, the encrypted hash, the public
key of the signer and the nonce used when generating the signature. To verify a
digital signature, the verifier uses the nonce and public key to decrypt the
hash and then compares that with the hash they calculate for the signed data.

In Verifiable Credentials, they store something called a "Linked Data
Signature" that contains the digital signature of the credential. They are
encoded in JSON-LD and look like this:

```
{
  "@context": "https://www.w3.org/2018/credentials/examples/v1",
  "title": "Hello World!",
  "proof": {
    "type": "Ed25519Signature2018",
    "proofPurpose": "assertionMethod",
    "created": "2019-08-23T20:21:34Z",
    "verificationMethod": "did:example:123456#key1",
    "domain": "example.org",
    "jws": "eyJ0eXAiOiJK...gFWFOEjXk"
  }
}
```

Notice how the type is a complex identifier that implies a hash function and
encryption function and relies on external documentation to specify exactly how
an `Ed25519Signature2018` is constructed. Implementors will have to read this
external documentation before they will know how to process each field to
verify this signature. Linked data signatures are only lightly self-describing.
The steps required to verify the signature are left up to the implementor and
assumed to be widely known. The linked data signature specification fails to
describe exactly the steps to take to verify the signature and presents a
problem for implementers new to cryptography.

In Secure Scuttlebutt, for some reason they forego the sigil character that
they use for everything else and instead signal that something is a signature
by adding a ".sig" string as part of the suffix. The hash algorithm used is
specified in the Secure Scuttlebutt protocol guide and is not encoded in the
signature itself. This limits the ability for Secure Scuttlebutt to adopt new
signature schemes without massive disruption caused by breaking changes and
incompatible implementations. The binary-to-text encoding scheme is also
specified in the protocol specification as non-URL-safe Base64. Again, this
limits the opportunity for future revisions to the protocol. The only part that
is self-describing is the encryption algorithm identifier in the suffix. A
Secure Scuttlebutt signature looks like the following:

```
QYOR/zU9dxE1aKBaxc3C0DJ4gRyZtlMfPLt+CGJcY73sv5abKKKxr1SqhOvnm8TY784VHE8kZHCD8RdzFl1tBA==.sig.ed25519
```

With CCLang, digital signatures are fully self-describing and do not rely on an
external specification to store the procedure for verifying the digital
signature. CCLang digital signatures are scripts that, when executed, verify
the digital signature. The primary challenge with CCLang digital signatures is
the reference to the external data that was hashed when calculating the
signature.  CCLang uses an "open-read-close" sequence along with
application-specific identifiers to specify what data storage unit to open
(e.g. file, stream, object), and which bytes to read from the data storage unit
for the hash calculation.

If we assume that the following is a digital signature for a text file named
`foo.txt` and the entire file was signed and that the binary-to-text encoding
scheme used is hexidecimal and the hash algorithm used was SHA256 and the
public key encryption algorithm used was Ed25519, then the CCLang digital
signature would look like the following:

```
418391ff353d77113568a05ac5cdc2d03278811c99b6531f3cbb7e08625c63bdecbf969b28a2b1af54aa84ebe79bc4d8efce151c4f24647083f11773165d6d04
Hex DECODE 1425ffb6c0cba6e6c23ca29f22bc3881cf924241dc683d7bb3b188ea2ff38966 Hex
DECODE foo.txt OPEN 0 $ READ CLOSE Ed25519 VERIFY
```

To understand and verify this digital signature it needs to be executed like
so:

1) The digital signature hexidecimal is pushed followed by the `Hex` encoding
   identifier and the opcode to decode the text to binary. The result is the
   digital signature binary data on the top of the stack.
2) Next the public key of the signer is decoded from hex and also left on the
   stack.
3) Then the name of the file---`foo.txt`---is pushed and the OPEN opcode pops
   the name, opens the file, and pushes the stream handle onto the stack.
4) Next the starting index of the read---`0`---followed by the number of bytes
   to read---`$`---are pushed onto the stack. The `$` symbol means "end of
   stream" which will cause the READ opcode to read all of the bytes from the
   file and push them onto the stack followed by the stream handle.
5) Then the file stream is closed which closes the file and pops the stream
   handle from the stack leaving the bytes from the file on top.
6) The last step is to push the signature algorithm identifier 'Ed25519' onto
   the stack and then execute the 'VERIFY' opcode to verify the signature from
   the signature data, the public key, and the data that was signed. The
   'VERIFY' opcode results in 'TRUE' or 'FALSE' being pushed onto the stack
   whether the signature is valid or not.

By executing the CCLang digital signature script, we validate the digital
signature by putting the signature, public key and signed data onto the stack
and then running the signature verification function. It is important to point
out that the parameter for the `OPEN` opcode---`foo.txt` in this case---is
entirely application specific. The above CCLang script is a digital signature
over a file in a filesystem and therefore uses a relative path to reference the
file. It is also assumed that the signature is stored in a separate file in the
same directory as what is called a "detached signature". If this signature was
stored as part of the `foo.txt` file, then the `READ` opcode would not use `$`
but would instead specify the number of bytes to read up to, but not including
the digital signature itself.

If this was a digital signature over a transaction in a blockchain data store,
the parameter for the `OPEN` operation would be the identifier for the
transaction---either the hash address of the transaction or whatever addressing
scheme the blockchain uses.

One other note is required. In systems where the signed data is encoded in JSON
and the signature itself is also part of the JSON, the CCLang script can use
multiple reads to cause the bytes on before and after the digital signature to
be read for hashing. This is far superior to the way Secure Scuttlebutt and the
Verifiable Credential specifications expect digital signatures to be verified
and does not require the full parsing and manipulation of the JSON document.

For insance, in Secure Scuttlebutt, digital signatures are stored as:
`"signature": "<base64>.sig.ed25519"`. The specification says to parse the
JSON, remove the signature field, then canonicalize the JSON before hashing.
With CCLang, if the signature data starts at offest 152 and is 100 bytes long,
then the CCLang version of the signature would have two `READ` operations
followed by a `CONCAT` like so:

```
0 152 READ 251 $ READ CONCAT
```

This would leave the stack with the JSON bytes before and after the signature
line on the stack ready to be parsed, canonicalized, and then hashed. If the
JSON is assumed to already be canonicalized, then the hash can be directly
calculated.

### Multi-Sig Signatures

Where it becomes obvious that CCLang is superior to both Verifiable Credentials
and Secure Scuttlebutt is in the encoding of multi-sig signatures. Multi-sig
signatures are digital signatures constructed as a conglomeration of signatures
from two or more identities. There are two kinds of multi-sig signatures:
parallel and serial.

Parallel signatures are when multiple identities sign the same data. This is a
simple agreement protocol. Think of it like multiple parties signing the same
contract. They are all agreeing to the data they are signing. Serial signatures
are when one identity signs some data and then a second identity signs both the
data concatenated with the first signature. The next signature would sign the
concatenation of the data and all previous signatures. This is an endorsement
protocol whereby each subsequent signature endorses the data and the previous
signatures. This is useful in supervisory roles such as project maintainers in
open source projects. Developers submit signed commits and then the maintainers
sign the commit and the developer's signature when merging it into the main
branch of the code, thus endorsing the contribution.

Currently, the Verifiable Credentials spec doesn't directly specify how either
type of multi-sig is to be serialized. It is left up to the implementor as long
as each individual signature follows the spec, it is a compliant signature.

In Secure Scuttlebutt protocol specification, multi-sig signatures are not
addressed at all. There is some discussion on supporting this but no consensus
has emerged mostly because all obvious solutions are ugly hacks and so far the
need is low.

With CCLang, both kids of multi-sig signatures are trivial. By using an
`IF-ELSE-FI` opcode trio it is easy to append digital signatures to form a
multi-sig signature. For instance, let's start with a basic digital signature
that looks like the following:

```
<sig hex> Hex DECODE <pub key hex> Hex DECODE foo.txt OPEN 0 $ READ CLOSE Ed25519 VERIFY
```

Let's say another signer wants to add a parallel signature to the one above.
They would calculate their own signature and append it using `IF-ELSE-FI` to
create a CCLang script that validates both signatures over the data:

```
<first sig hex> Hex DECODE <first pub key hex> Hex DECODE OPEN 0 $ READ CLOSE ED25519 VERifY IF <second sig hex> Hex DECODE <second pub key hex> Hex DECODE foo.txt OPEN 0 $ READ CLOSE Ed25519 VERIFY ELSE FALSE FI
```

This is a parallel multi-sig. The first part is a normal CCLang digital
signature validation. After the `VERIFY` opcode, the stack will either have a
`TRUE` on top if the signature is valid or a `FALSE` on top if it is not. The
`IF` opcode pops the top and if it is `TRUE`, the script between `IF` and
`ELSE` is executed, otherwise the script between `ELSE` and `FI` is executed.
Between the `IF` and `ELSE` is a second digital signature validation script
that will leave `TRUE` on top if it is valid. So if both signatures are valid,
the script will end with `TRUE` on top of the stack. If the first signature is
invalid, then the code between `ELSE` and `FI` will get executed. That script
just pushes `FALSE` onto the stack to indicate that the signature checks
failed. Any number of parallel signatures can be appended to the digital
signature independently.  This allows for asynchronous multi-sig operations
where one person signs some data and then forwards the data and their signature
to the next person who then appends their signature and so on until all
signatures are created and appended.

For serial multi-sig, the subsequent signature must include not only the data
but the previous signatures. To accomplish this, the CCLang script of the
endorsing signature must be set up with reads of the data and reads of the
previous signature data before the validaton opcode executes.

Any number of digital signatures can be generated and appended to the existing
signature using the `IF-ELSE-FI` pattern. It is also possible to mix both
parallel and serial digital signatures in any arbitrary arrangement. Let's say
a commit comes in that is signed off by two authors that collaborated. That
commit has two parallel signatures, one from each author. Then the maintainer
of the project merges it with their serial signature that covers both the
commit and the two signatures from the authors.

# Reference-level explanation [reference-level-explanation]: #reference-level-explanation

## The Stack

The heart of the CCLang system is the stack machine used to execute CCLang
scripts.  In general, the CCLang stack machine is an _abstract_ stack machine
that accepts data of any type. Long strings get stored and references to the
strings are pushed onto the stack. Opcodes may or may not pop arguments from
the stack and may or may not push results onto the stack. If for some reason an
opcode cannot be executed due to incorrect types of input parameters or the
values of the input parameters are invalid, the execution of the CCLang script
will halt immediately and an appropriate error code will be returned to the
caller of the CCLang interpreter.

## Documenting Opcodes

The rest of this reference uses the standard notation for documenting commands
in the Forth programming language. Forth is also a RPN stack based programming
language. It uses a simple form of documenting commands that looks like the
following:

```
/ a1 -- argument one is data of any type.
/ a2 -- second argument of any type.
= ( a1 a2 -- TRUE|FALSE )
```

Comment lines being with a single `/` character and come before the command and
stack documentation to give explanations of the parametes int he stack diagram.
The stack diagram follow the command---in this case `=`---and consists of a
right parenthesis `(` followed by the parameters poped off of the stack
with `--` separator followed by the return parameters pushed onto the stack. In
the case of the `=` opcode, it pops two parameters of any type---a1 and a2---
and compares them for equality and pushes either `TRUE` if they are equal or
`FALSE` if they are not.

Another example would be the DECODE opcode:

```
/ e -- encoded data.
/ t -- encoding type identifier.
/ b -- decoded binary data.
DECODE ( e t -- b )
```

The DECODE opcode pops the encoded data and the encoding type identifier from
the stack, executes the correct decode operation to turn the text to binary and
then pushes the binary onto the stack. This is only used in text serialization
formats such as JSON, YAML, or XML.

## Opcodes

### Logical Comparison

```
/ a1 -- data of any type.
/ a2 -- data of any type.
= ( a1 a2 -- TRUE|FALSE )
```

The `=` operator pops two arguments of any type and compares them for equality.
It pushes TRUE if they are equal, FALSE if not.

```
/ a1 -- numerical data.
/ a2 -- numerical data.
< ( a1 a2 -- TRUE|FALSE )
```

The `<` operator pops two numerical arguments off of the stack and compares
them to see if the first argument popped is less than the second argument
popped. It pushes TRUE if that is true, FALSE if not.

```
/ a1 -- numerical data.
/ a2 -- numerical data.
> ( a1 a2 -- TRUE|FALSE )
```

The `>` operator pops two numerical arguments off of the stack and compares
them to see if the first argument popped is greater than the second argument
popped. It pushes TRUE if that is true, FALSE if not.

There are also the `<=`, `>=`, and `!=` opcodes that are combinations of the
above logical comparisons. They test for less-than-or-equal, greater-than-or-
equal, and not-equal respectively.

### Binary-to-text Operations

```
/ e -- text-encoded data.
/ t -- encoding type identifier.
/ b -- decoded binary data.
DECODE ( e t -- b )
```

The `DECODE` opcode is used to decode binary data that has been encoded in some
binary-to-text encoding system such as Base64, Base58, and/or Hexidecimal. The
list of supported encoding types is listed below.

```
/ b -- binary data.
/ t -- encoding type identifier.
/ e -- binary data encoded as text.
ENCODE ( b t -- e )
```

The `ENCODE` opcode is used to encode binary data into a text format using the
specified binary-to-text encoding scheme. The result is the text encoded binary
data.

### Encryption

```
/ e -- binary encrypted data.
/ k -- binary key data.
/ o.. -- optional binary parameters required by the given algorithm (e.g. nonce).
/ i -- encryption algorithm identifier.
/ b -- decrypted binary data.
DECRYPT ( e k o.. i -- b )
```

The `DECRYPT` opcode is used to decrypt the encrypted data using the key and
any other required data for decrypting using the algorithm specified by the
algorithm identifier. The result is the decrypted binary data.

```
/ b -- binary data.
/ k -- binary key data.
/ o.. -- optional binary parameters.
/ i -- encryption algorithm identifier.
/ e -- encrypted binary data.
```

The `ENCRYPT` opcode is used to take binary and encrypt it using the specified
encryption algorithm and key and algorithm-specific parameters. The result is
the encrypted data.

### Signing

```
/ s -- binary signature data.
/ k -- binary public key.
/ d -- binary data that was signed.
/ o.. -- optional binary signature parameters.
/ i -- signature algoirthm identifier.
VERIFY ( s k o.. i -- TRUE|FALSE )
```

The 'VERIFY' opcode executes a digital signature verification function
associated with the signature algorithm specified. It pops the signature,
public key, data that was signed and any optional paramters and the identifier
off and pushes 'TRUE' if the signature is valid and 'FALSE' if it is not.

```
/ d -- binary data to be signed.
/ k -- binary secret key to sign with.
/ o.. -- binary optional parameters for the signature scheem.
/ i -- signature algorithm identifier.
SIGN ( d k o.. i -- s )
```

The 'SIGN' opcode creates a detached digital signature over the provided data
using the secret key. All parameters are popped and the resulting signature is
pushed onto the stack.

### Hashing

```
/ b -- binary data.
/ i -- hashing algorithm identifier.
/ h -- hash of the data.
HASH ( b i -- h )
```

The `HASH` opcode takes the binary data and hashes it using the specified
hashing algorithm. The result is the hash of the data.

### Data I/O

```
/ i -- data storage object identifier (e.g. file name, transaction number, etc)
/ m -- mode to open the file under.
/ h -- handle to the opened data storage object.
OPEN ( i m -- h )
```

The `OPEN` opcode is application-specific in that the data storage object
identifier is specific to the application. In some cases it may be the file
name of a file to open or it may be the identifier of a transaction in a
blockchain application or it could any reference to a data object that makes
sense in the application. The mode is an identifier representing read, write,
and append. The result is a handle to the opened object that can be used by
other data I/O commands.

```
/ h -- handle to the opened data storage object.
/ s -- zero-indexed starting offset to begin the read.
/ n -- number of bytes to read from the object.
/ b -- binary read from the data object.
READ ( h s n -- b h )
```

The `READ` opcode takes the handle to the opened data storage object and reads
the number of bytes specified starting at the offset given. The result is the
binary data read from the data object and then the handle to the open data
object on top.

```
/ h -- handle to the opened data storage object.
/ b -- binary data to write.
WRITE ( h b -- h )
```

The `WRITE` opcode takes the handle and the data to write and writes it to the
open data object. The result is the handle to the data object.

```
/ h -- handle to the opened data storage object.
/ n -- number of bytes and direction to seek in.
SEEK ( h n -- h )
```

The `SEEK` opcode seeks the specified number of bytes. If the number of bytes
is negative, it seeks backwards towards the 0th index byte. If the number is
positive, it seeks forward towards the last byte in the object. The result is
the handle to the opened data object.

```
/ h -- handle to the opened data storage object.
CLOSE ( h -- )
```

The `CLOSE` opcode closes the opened data storage object. It pops the handle
from the stack and does not push anything to the stack.

### Data Manipulation

```
/ b -- binary data.
CONCAT ( b b -- b )
```

The `CONCAT` opcode pops two binary data arguments from the stack and
concatenates the top argument to the end of the argument below it on the stack
and pushes the resulting binary data back onto the stack.

```
/ b -- binary data.
/ o -- integer offset.
/ c -- integer count.
SLICE ( b o c -- b )
```

The `SLICE` opcode pops the count, offset, and binary data from the stack and
creates a new binary data argument starting at the offset and taking the count
number of bytes from the original binary data argument. The result is pushed
onto the stack. The offset is zero-indexed and both the offset and count are in
bytes.

```
data: abcdef0123456789
         ^ 3 offset ^ 12 count

result: def012345678
```

```
/ b -- binary data.
| ( b b -- b )
```

The `|` opcode is the bitwise _or_ between the top two binary data arguments.
The result is pushed onto the stack.

```
/ b -- binary data.
& ( b b -- b )
```

The `&` opcode is the bitwise _and_ between the top two binary data arguments.
The result is pushed onto the stack.

```
/ b -- binary data.
^ ( b b -- b )
```

The `^` opcode is the bitwise _xor_ between the top two binary data arguments.
The result is pushed onto the stack.

```
/ b -- binary data.
~ ( b -- b )
```

The `~` opcode is the bitwise inverse of the top binary data argument. The
result is pushed onto the stack.

### Stack Control

```
/ a -- argument of any type.
DUP ( a -- a a )
```

The `DUP` opcode pops the top item, duplicates it and pushes the original and
its duplicate on the top of the stack.

```
/ a -- argument of any type.
POP ( a -- )
```

The `POP` opcode pops the top item from the stack and forgets about it. This is
used for throwing away the top of the stack.

### Flow Control

```
/ b -- boolean argument.
IF ( b -- ) ELSE FI
```

The `IF-ELSE-FI` opcode trio and the related `IF-FI` opcode duo are used
to do conditional branching in CCLang scripts. The `IF` opcode pops the top of
the stack and evaluates it as a boolean argument.

In the case of `IF-ELSE-FI` if the argument is true then the script between
`IF` and `ELSE` is excuted followed by jumpting to the script after the `FI`.
If it is a false then the script between the `ELSE` and the `FI` is executed.

In the case of `IF-FI` if the argument is true then script between the `IF`
and `FI` is executed. If it is false then flow jumps to the script after the
`FI` opcode.

## Encoding Formats

The first version of CCLang supports the following encoding types:

* Hexidecimal -- binary represented as lower case hexidecimal characters.
* Base64 -- 62nd and 63rd characters are `+` and `/` respectively. No line
  length maximum; all data in one line with mandatory padding using `=`.
* Base64Url -- 62nd and 63rd characters are `-` and `_` respectively. No line
  length maximum; all data in one line with no padding.
* Base58 -- See definition [here](https://en.wikipedia.org/wiki/Base58).
* Base58Check -- This is a complex construct and implemented using CCLang itself.

## Encryption Algorithms

The first version of CCLang supports the following encryption algorithms:

* Curve25519
* RSA
* XSalsa20Poly1305
* AES

## Signing Algorithms

The first version of CCLang supports the following signing algorithms:

* Ed255519
* ECDSA

## Hashing Algorithms

The first version of CCLang supports the following hashing algorithms:

* SHA256
* SHA512
* SHA512-256
* Blake2b
* Argon2id

## Serialization Formats

CCLang is an abstract language definition and does not prescribe how the data
and/or opcodes are serialized in any given format. Each encoding format is
left to specify how each that is done. Below is a sample encoding specification
for JSON.

### JSON-CCLang

The first job of mapping abstract CCLang to JSON is to decide the string
constants to use for the supported encoding types, encryption algorithms,
hashing algorithms, and opcodes. Below are lists of constants in CCLang and
their string constant equivalents in JSON.

#### Encoding Constants

* Hexidecimal - `Hex`
* Base64 - `Base64`
* Base64Url - `Base64Url`
* Base58 - `Base58`

#### Encryption Algorithms

* Curve25519 - `Curve25519`
* RSA - `RSA`
* XSalsa20Poly1305 - `XSalsa20Poly1305`
* AES - `AES`

#### Signing Algorithms

* Ed25519 - `Ed25519`
* ECDSA - `ECDSA`

#### Hashing Algorithms

* SHA256 - `SHA256`
* SHA512 - `SHA512`
* SHA512-256 - `SHA512-256`
* Blake2b - `Blake2b`
* Argon2id - `Argon2id`

#### Opcodes

* = - `=`
* < - `<`
* > - `>`
* <= - `<=`
* >= - `>=`
* != - `!=`
* | - `|`
* & - `&`
* ^ - `^`
* ~ - `~`
* DECODE - `DECODE`
* ENCODE - `ENCODE`
* DECRYPT - `DECRYPT`
* ENCRYPT - `ENCRYPT`
* SIGN - `SIGN`
* VERIFY - `VERIFY`
* HASH - `HASH`
* OPEN - `OPEN`
* READ - `READ`
* WRITE - `WRITE`
* SEEK - `SEEK`
* CLOSE - `CLOSE`
* CONCAT - `CONCAT`
* SLICE - `SLICE`
* DUP - `DUP`
* POP - `POP`
* IF - `IF`
* ELSE - `ELSE`
* FI - `FI`

#### Encoding

CCLang scripts are stored as items in a JSON list. So a simple hex encoded
public string would be encoded as:

```
{
  "key": [ "0a7d1d784358af1f8073ba07eb5ae2fc7272a860ec4547de8bc13d04259cd59a", "Hex", "DECODE" ]
}
```

# Drawbacks
[drawbacks]: #drawbacks

The only drawback is that the resulting cryptographic constructs are not
compact. They can be strings that are very long and it may be difficult for a
person not experienced in RPN to follow what is being happening. The
compactness of other cryptographic construct representations is a result of
fixing details of the constructs in specifications that are not easy to update
and are not transported with the data itself. This makes all other formats
non-self-describing. It makes it hard for new systems to know exactly what they
must do to make use of any specific cryptographic construct.

# Rationale and alternatives
[alternatives]: #alternatives

The rationale for CCLang is to standardize a self-describing format for
serialized cryptographic constructs. As we develop new applications that use
ever more complicated constructs, it is become more and more important to adopt
a self-describing serialized form. Gone are the days where all we needed to
store was the public key and then write the spec to say, "always use algorithm
X to decrypt the data". With the increasing use of multi-sig signatures of all
types and zero-knowledge proofs (ZKPs), our systems can be greatly enhanced by
using a self- describing format. It also makes it easy for implementors to map
CCLang to whatever is the underlying cryptographic library they are using.

# Prior art
[prior-art]: #prior-art

Systems like Git and Secure Scuttlebutt as well as DID powered systems are
struggling to adapt to the new multi-sig and ZKP world in which they operate.
So far no good proposals have been made to adapt Git to multi-sig and Secure
Scuttlebutt has just rejected multi-sig altogether for now. There have been
some attempts at adapting Linked Data Signatures used in DIDs to support
multi-sigs. What was proposed is a little similary to CCLang but is clunky and
doesn't leverage the elegant solution of a stack machine and language.

As stated in the introduction, CCLang is inspired by Bitcoin script and takes
ideas from the Forth programming language. Bitcoin script would have been a
good alternative but it has purpose-built opcodes that only make sense in the
context of Bitcoin transactions. CCLang aims to be a more general design for
achieving the same thing as Bitcoin script. In fact, CCLang could be used to
implement all of the features in Bitcoin script but in a little more verbose
way. Bitcoin script opcodes like OP_CHECKLOCKTIMEVERIFY could be implemented
using a sequence of CCLang opcodes that read the nLockTime from the Bitcoin
transaction and uses `IF-ELSE-FI` to do the same things that
OP_CHECKLOCKTIMEVERIFY does.

There is some other prior art documented in Chrisopher Allen's post on smarter
signatures [^0]. In that blog post he discusses functional programming inspired
solutions like Forth-based scripts such a Bitcoin script. He also references
Peter Todd's Dex script that uses Lisp-like s-expression syntax [^1]. 

In the end, the benefits of a self-describing system like Bitcoin script makes
this better than the "specification assisted" systems used by Git and Secure
Scuttlebutt and DID docs. Bitcoin's script is too application-specific to be
a good general solution so CCLang has been created to fill that gap.

# Unresolved questions
[unresolved]: #unresolved-questions

* Should we focus on making sure CCLang is non-Turing-complete? These are
  essentially small "smart contracts" and therefore they are safety critical.
  Non-Turing-completeness would allow for the creation of static analysis tools
  and deterministic evaluation looking for undesirable corner case
  possibilities.
* Should CCLang include a macro-definition system where common sequences of
  CCLang opcodes can be aliased with a macro name that is used in the
  serialized versions? This would require a macro setup script that would be
  implied to run before any CCLang scripts are executed to ensure that all
  macro definitions are processed and initialized before the scripts that use
  those macros are executed. A good example of where this would be useful is
  decoding Base58Check Bitcoin addresses. The encoding and decoding of a
  Base58Check address requires concatenation, byte masking, and multiple SHA256
  hashes for the checksum piece. Having a macro called Base58CheckDecode would
  make CCLang scripts more readable.
* Some crypto libraries like NaCl hide a lot of the "sub-operations" used in
  their more complicated constructs like the sealed box. This is done on
  purpose to make the library misuse resistant and hides a lot of inner
  details. CCLang would be a challenge to map to NaCl without defining new
  opcodes that mapped directly to the secret box open/close calls in NaCl.

# References

[^0]: [http://www.lifewithalacrity.com/2016/10/smarter-signatures-experiments-in-verifications/](http://www.lifewithalacrity.com/2016/10/smarter-signatures-experiments-in-verifications/)
[^1]: [https://github.com/WebOfTrustInfo/rwot2-id2020/blob/master/topics-and-advance-readings/DexPredicatesForSmarterSigs.md](https://github.com/WebOfTrustInfo/rwot2-id2020/blob/master/topics-and-advance-readings/DexPredicatesForSmarterSigs.md)

# Changelog
[changelog]: #changelog

- [31 Jan 2020] - v1 - initial draft.
- [12 Feb 2020] - v2 - reworking signinging.
- [13 Feb 2020] - v3 - cleanup of formatting.
