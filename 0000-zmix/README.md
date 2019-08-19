- Feature Name: (mal000002, zmix)
- Start Date: 05-Feb-2019
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 1

# Summary
[summary]: #summary

Zmix is a library for expressing, constructing, and verifying non-interactive
zero-knowledge proofs (ZKPs). A broad class of zero-knowledge proofs is 
supported. Schnorr-type proofs are used for many parts and to glue pieces
together, while individual parts of a proof may use other zero-knowledge
techniques. 

# Motivation
[motivation]: #motivation

Zero-knowledge proofs form a cryptographic building block with many 
applications. Within hyperledger, Indy relies heavily on ZKPs to construct
anonymous credentials. Fabric uses ZKPs to authenticate transactions when
using the identity mixer membership service provider. Implementing ZKPs
correctly however is challenging. Zmix aims to offer a secure
implementation of ZKPs which can be consumed through a simple API, making
it easier for other projects to use ZKPs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Introduction

A zero-knowledge proof enables a prover to convince a verifier of the truth
of some statement about certain values, without revealing those values. For
example, a prover may know that values `A` and `B` have an equal discrete
logarithm. That is, for some `x`, `A = g^x` and `B = h^x`. It could
convince a verifier that this is true by revealing `x`. If the prover would
rather not disclose `x`, it could instead construct a zero-knowledge proof
of this statement, and send the resulting proof to the verifier. The 
verifier can now be sure that `log_g(A) = log_h(B)`, without knowing the
discrete log `x`.

In the example above, we have three pieces of data: 

- What do we prove? We prove that there exists an `x` such that `A = g^x`
and `B = h^x`.

- How do we know this is actually true? We know this because we know `x`
for which this is true. 

- The actual zero-knowledge proof that the prover produces and the verifier
verifies.

In zmix, we have data types exactly corresponding to these three points:

- `ProofSpec`: specify what we want to prove

- `Witness`: contains the secrets that satisfies the statement of a
`ProofSpec`.

- `Proof`: the zero-knowledge proof. 

The library provides two features: constructing and verifying
zero-knowledge proofs. To construct a proof, zmix requires a `ProofSpec s`
and a `Witness w` (which is a valid witness for `p`), and will output a
`Proof p`. To verify a proof, zmix requires input a `ProofSpec s` and a
`Proof p`, and outputs a boolean indicating whether proof was valid or
invalid.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Conceptually, the zmix library will offer the functions:

* `generate_proof(s: ProofSpec, w: Witness) -> Result<Proof, Error>`
* `verify_proof(p: Proof, s: ProofSpec) -> Result<(), Error>`

where the proof specification contains

1. the number of secrets involved in the zero-knowlege proof (we call these
_messages_), and
1. a list of _statements_, which represent sub-parts of the overall
zero-knowlege proof:

```
pub struct ProofSpec {
    message_count: usize,
    statements: Vec<Statement>,
    ...
}
```

While the library will offer various kinds of statements, the most
important ones are for signatures. In general, signatures contain at least

1. a public key under which the signature can be verified, and
1. an *ordered* list of values that are either hidden (that is, secret) or
revealed (that is, public).

For example, the library will offer a statement type for Boneh Boyen
Shacham (BBS) signatures as follows:

```
pub enum Statement {
    SignatureBBS {
        public_key: Vec<u8>,
        messages: Vec<HiddenOrRevealedValue>,
    },
    ...
}

pub enum HiddenOrRevealedValue {
    HiddenValueIndex(usize),
    RevealedValue(Vec<u8>),
}
```

While for each revealed value in the signature statement simply contains
the actual value, hidden values are qualified with an _index_.
This index must be smaller or equal to the overall number of secrets
involved in the zero-knowledge proof as specified in the field
`message_count` in the proof specification.

This index-based approach allows for referring to the *same* secret in the
overall zero-knowledge proof from *different* statements. For example,
another important statement type are commitments, such as the following
Pederson commitment:
```
pub enum Statement {
    ...
    PedersenCommitment {
        message_index: usize,
        commitment: Vec<u8>,
        ...
    },
```
Consider a proof specification with a `message_count` of 2 and the
following two statements:
1. a `SignatureBBS` over three values where the first is hidden referring
to index 0, the second is hidden referring to index 1, and the third is the
revealed value 42, and
1. a `PedersenCommitment` referring (via `message_index`) to index 0.

The fact that both the signature and the commitment refer to index 0 means
that they both refer to the same secret in the overall list of secrets (of
size `message_count`) involved in the zero-knowledge proof.

In this respect, the case when *different signature statements* refer to
the *same* index is special: this means that the underlying secret values
in the signatures are *equal*.

## Proof Modules

As mentioned previously, zmix will offer various kinds of statement types.
For each statement type, the library will provide a respective 
*proof module* that handles all aspects of this statement type. Proof
modules must be implemented according to the following interface:
```
pub trait ProofModule {
    fn get_hash_contribution(
        statement: Statement,
        witness: StatementWitness,
        message_r_values: Vec<Vec<u8>>,
    ) -> Result<(HashContribution, ProofModuleState), ZkLangError>;
    fn get_proof_contribution(
        state: ProofModuleState,
        challenge_hash: Vec<u8>,
    ) -> Result<StatementProof, ZkLangError>;
    fn recompute_hash_contribution(
        challenge_hash: Vec<u8>,
        proof: StatementProof,
    ) -> Result<HashContribution, ZkLangError>;
}

#[transparent]
pub struct HashContribution(pub Vec<u8>);
pub struct ProofModuleState {
    state: Vec<u8>,
}
```

Given this proof module interface, a central *proof orchestrator* within
zmix will coordinate the generation and the verification of zero-knowledge
proofs as follows:

![Generation flow](zmix_proof_generation.png)

![Verification flow](zmix_proof_verification.png)

### Module types

Zmix plans to offer the following proof modules

- Signatures
    - Boneh Boyen Shacham
    - Pointcheval Saunders
- Pedersen commitments
- Bulletproof intervals
- Bulletproof set membership inclusive and exclusive
- SNARKs set memberships
- Verifiable encryption 

# Changelog
[changelog]: #changelog

- [15 Aug 2019] - v1.1 - Update following recent discussions.

- [5 Feb 2019] - v1 - Initial version.
