- Feature Name: language-bindings
- Start Date: 2019-03-14
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 1

# Summary
[summary]: #summary

The Rust programming language was chosen so Ursa can be portable and
consumed by multiple programming languages. This RFC details how this is
to be done.

# Motivation
[motivation]: #motivation

Hyperledger projects will want to start using Ursa. Those that use Rust
will have immediate access to the library's functionality. Those projects
using other programming languages will need other methods.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Rust programming language allows exporting functions to be called
from in other languages. Most programming languages can consume "C"
callable libraries. This RFC covers how Ursa will enable its code to be
consumable by other languages.

When exporting to other languages there are two types of wrappers that
can be created: thin and idiomatic. Example code written
in Rust below will be used to illustrate the difference.

```rust
pub struct Ed25519 {}

impl Ed25519 {
    pub fn sign(message: &[u8], private_key: &[u8]) -> Vec<u8> {
        ...
    }
    pub fn verify(message: &[u8], public_key: &[u8], signature: &[u8]) -> bool {
        ...
    }
}
```

The purpose of the wrapper is to expose `sign` and `verify`. A thin wrapper
will be the simplest method like so:

```c
int ursa_bls_sign(unsigned char *signature, const unsigned long long signature_len,
                  const unsigned char *const message, const unsigned long long message_length,
                  const unsigned char *const private_key, const unsigned long long private_key_length);

int ursa_bls_verify(const unsigned char *const message, const unsigned long long message_length,
                    const unsigned char *const public_key, const unsigned long long public_key_length,
                    const unsigned char *const signature, const unsigned long long signature_length);
```

The C code now exposes those two functions so C# can use them as a thin wrapper

```csharp
using System;
using System.Runtime.InteropServices;

namespace Hyperledger.Ursa.Api
{
    internal static class NativeMethods
    {
        [DllImport("ursa", CharSet = CharSet.Ansi)]
        internal static extern int ursa_bls_sign(out byte[] signature, long signature_len,
                                                 byte[] message, long message_length,
                                                 byte[] private_key, long private_key_length);

        [DllImport("ursa", CharSet = CharSet.Ansi)]
        internal static extern int ursa_bls_verify(byte[] message, long message_length,
                                                   byte[] public_key, long public_key_length,
                                                   byte[] signature, long signature_length);
    }
}
```

Thin wrappers are not desirable by end users because it requires more knowledge
about the native library than they perhaps want to know. It also does not allow
language developers to use what they are most familiar with. However, thin wrappers
are required because they handle marshaling between the two languages. Idiomatic wrappers
are written to hide the nastiness of the thin wrapper and allow language developers to
stick to what they like and are familiar with. Continuing the example above, an idiomatic
wrapper can be written like this

```csharp
using System;
using System.Text;
using System.Runtime.InteropServices;

using Hyperledger.Ursa.Api;

namespace Bls
{
    public static class Bls
    {
        private const int SIGNATURE_BYTES = 32;

        public function byte[] Sign(string message, byte[] privateKey)
        {
            return Sign(Encoding.UTF8.GetBytes(message), private_key);
        }

        public function byte[] Sign(byte[] message, byte[] privateKey)
        {
            var signature = new byte[SIGNATURE_BYTES];

            if (NativeMethods.ursa_bls_sign(out signature, SIGNATURE_BYTES, message, message.Length, privateKey, privateKey.Length) == 0)
            {
                return signature;
            }
            throw new Exception("An error occurred while signing message");
        }

        public function bool Verify(string message, byte[] publicKey, byte[] signature)
        {
            return Verify(Encoding.UTF8.GetBytes(message), publicKey, signature);
        }

        public function bool Verify(byte[] message, byte[] publicKey, byte[] signature)
        {
            switch (NativeMethods.ursa_bls_verify(message, message.Length,
                                                  privateKey, privateKey.Length,
                                                  signature, signature.Length))
            {
                case 0: return true;
                default: return false;
            }
        }
    }
}
```

The idiomatic wrapper provides more logic and parameters for convenience than the thin wrapper does.
It can also be expanded to include other types but ultimately maps all inputs to
the expected thin wrapper types.

Thin wrappers are composed of many functions using the [foreign function interface (FFI)](https://doc.rust-lang.org/nomicon/ffi.html).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Thin wrappers can be problematic to write by hand and maintain as APIs change.
It is desirable to have thin wrappers generated programmatically.
Rust supports generating FFI using `bindgen`. `bindgen` can map to other languages like
C and WASM. Thus thin wrappers should be created and maintained as much as possible using features like `bindgen`.
Thin wrappers should be included in the Ursa project itself in a folder called `wrappers`. This will allow them to
stay in-sync with any other changes to the core library.

Idiomatic wrappers usually cannot be generated by hand as they require more in depth language experience
than automatic generators can produce. Therefore, idiomatic wrappers should be written by project contributors
and maintained in separate repositories. These wrappers also SHOULD include tests for interacting with
native Ursa.

Here are general guidelines for producing wrappers:

`<language>` should follow the naming convention as described [here](https://support.codebasehq.com/articles/tips-tricks/syntax-highlighting-in-markdown)

- Thin wrappers
    - Produced by code
    - Output into the `libursa/wrappers/<language>/folder`
- Idiomatic wrappers
    - Written by developers
    - Implemented as best deemed by contributors/community
    - Stored in separate repositories using the naming convention `ursa-wrapper-<language>`
    - Keep certain things invariant
        - The reason is to make wrappers similar to one another, and to avoid unnecessary documentation burden or mental model translation between wrappers (which may have light doc) and ursa (where the investment in doc will be substantial).
        - If someone learns how to use a wrapper of the API in language 1, and then go look at the wrapper of the API in language 2, one should see the same basic concepts, and be able to predict the function names one should call and the sequence of calls one should use.
        - Each wrapper should preserve all the terminology of the main interface: don't change `encrypt` to `seal` or `decrypt` to `open`
        - All preconditions on parameters, all data formats (e.g., json, config files, etc), all constraints on input should be identical in all wrappers and in the base API.
        - The "level" of the wrapper API can be generally "higher" than the C-callable functions, but it should expose the same amount of functionality and flexibility as the C-callable functions, so the higher level functions, if they exist, should be an addition to lower-level ones, not substitutes for them. In other words, helpers and convenience methods are great, but don’t allow them to entirely hide the core functions which provide granular control.
    - The wrapper should document the earliest and latest version of ursa that it knows to be compatible.
    - The wrapper should document what platforms it targets, what use cases it's built for, etc. A wrapper should be able to find ursa using default OS methods (e.g., in the system PATH), but should also provide a way for a specific path to ursa to be specified, such that the wrapper can work either from an OS-wide install of ursa or a version in a particular directory.

# Drawbacks
[drawbacks]: #drawbacks

- Some developers may want all ursa code written in the language of their needs to avoid issues presented by FFI.
- Some languages cannot generate thin wrappers yet using `bindgen`.

# Prior art
[prior-art]: #prior-art

Languages have consumed C callable libraries for many years. Other languages have also compiled to C callable libraries like other system languages.

# Unresolved questions
[unresolved]: #unresolved-questions

# Changelog
[changelog]: #changelog

- [14 Mar 2019] - v1 - Initial version
