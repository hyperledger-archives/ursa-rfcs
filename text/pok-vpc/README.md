- Feature Name: pok-vpc
- Start Date: 2019-08-03
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 0.1

# Summary
[summary]: #summary

This RFC describes constructing proofs of knowledge of committed values in a vector Pedersen commitment using Sigma protocol. The prover is able to construct both interactive and non-interactive proofs of knowledge. The code is [here](https://github.com/hyperledger/ursa/blob/2067d0543c80eb883721863d7886d4e66f1d655d/libzmix/ps/src/pok_vc.rs).

# Motivation
[motivation]: #motivation

Several protocols require proving knowledge of committed values in commitments. One of them is proving knowledge of messages and randomness in a vector Pedersen commitment. Such protocols then become part of larger protocols like proving knowledge of a signature or selectively revealing some of the messages in the commitment. These protocols form the basis for anonymous credentials in Hyperledger Aries.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Sigma protocol is used to prove knowledge of discrete logarithms and relations among them. So it can be used to prove knowledge of `x` in g<sup>`x`</sup> without revealing `x`. Or it can be used to prove knowledge of `x`, `y` and `z` in g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup> without revealing `x`, `y` or `z`. A slight variation of the statement is to reveal that g<sub>2</sub>'s exponent is `y` in g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup>. This is accomplished by proving knowledge of `x` and `z` in g<sub>1</sub><sup>`x`</sup>g<sub>3</sub><sup>`z`</sup> and the verifier can check that product of g<sub>2</sub><sup>`y`</sup> and g<sub>1</sub><sup>`x`</sup>g<sub>3</sub><sup>`z`</sup> is indeed g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup>.   
Being able to prove such statements is important for anonymous credentials because in those the credential signature is of a form Zg<sub>1</sub><sup>`m1`</sup>g<sub>2</sub><sup>`m2`</sup>g<sub>3</sub><sup>`m3`</sup>...g<sub>n</sub><sup>`n`</sup>h<sup>`r`</sup> where `Z` is derived from signer's secret key and each of m<sub>i</sub> is a credential attribute and `r` is the randomness. There might be more elements in the signature but the aforementioned is of interest here. When using such credentials for zero knowledge proofs, the prover proves knowledge of the signature and the attributes in there. Depending on the signature scheme used, the prover randomizes the signature in a particular way before creating a proof of knowledge of the messages and signature.

Interactive Sigma protocols are 3-step, 
- in the first step, prover commits to some randomness and sends it to the verifier.
- in 2nd step, prover gets a challenge from the verifier.
- in 3rd step, prover creates a response involving the randomness, its secret and the challenge and sends to the verifier.

The verifier will now check correctness of a relation involving the commitment from 1st step, challenge and response.
Non interactive Sigma protocols modify the 2nd step. Rather than getting the challenge from the verifier, they compute the challenge by hashing the commitments from 1st step.   

Consider an example of prover proving knowledge of `x`, `y` and `z` in g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup>. Lets call g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup> as P. Only the prover knows `x`, `y` and `z` but both prover and verifier know P, g<sub>1</sub>, g<sub>2</sub> and g<sub>3</sub>. 
- prover picks 3 random numbers r<sub>1</sub>, r<sub>2</sub> and r<sub>3</sub> (corresponding to x, y and z), creates commitment T = g<sub>1</sub><sup>r<sub>1</sub></sup>g<sub>2</sub><sup>r<sub>2</sub></sup>g<sub>3</sub><sup>r<sub>3</sub></sup> and sends T to the verifier.
- verifier sends challenge c to prover.
- prover computes responses s<sub>1</sub> = r<sub>1</sub> - c*x, s<sub>2</sub> = r<sub>2</sub> - c*y, s<sub>3</sub> = r<sub>3</sub> - c*z and sends s<sub>1</sub>, s<sub>2</sub> and s<sub>3</sub> to verifier.
- verifier checks if g<sub>1</sub><sup>s<sub>1</sub></sup>g<sub>2</sub><sup>s<sub>2</sub></sup>g<sub>3</sub><sup>s<sub>3</sub></sup>*P<sup>c</sup> equals T. If it does then verifier is convinced of prover's knowledge of `x`, `y` and `z`.

In the non-interactive version prover will hash g<sub>1</sub>, g<sub>2</sub> and g<sub>3</sub>, P and T to compute challenge c.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Since the commitment can be in group G1 or G2, the proof of knowledge protocol is implemented as a macro `impl_PoK_VC`. The macro will generate 3 entities (structs), `ProverCommitting`, `ProverCommitted` and `Proof` for each group that is passed to the macro. The macro takes 5 parameters, the first 3 are identifiers for these entities and the last 2 are the group element and the group element vector respectively.

```rust
macro_rules! impl_PoK_VC {
    ( $ProverCommitting:ident, $ProverCommitted:ident, $Proof:ident, $group_element:ident, $group_element_vec:ident ) => {
        ........
    }
```

The macro is defined here [here](https://github.com/hyperledger/ursa/pull/46/files#diff-168ead191b84bf0856f3d153e9da3ba5R14).  

The following describes how the 3 entities are used.
- For the 1st step (commitment), the prover first creates a `ProverCommitting` object. This object holds the generators and random values used during this step. 
- Prover calls `ProverCommitting::commit` for each proof of knowledge that he wishes to do. This function takes a generator (`gen`) and optionally takes a randomness. If not supplied, it creates its own.
    ```rust
    impl $ProverCommitting {
        .....
        pub fn commit(
            &mut self,
            gen: &$group_element,
            blinding: Option<&FieldElement>,
            ) -> usize {
                ......
        }
    }
    ```
- Once the prover has committed for all its secrets, it calls `ProverCommitting::finish` method that results in creation of `ProverCommitted` object after consuming `ProverCommitting`. 
    ```rust
    impl $ProverCommitting {
        .....
        pub fn finish(self) -> $ProverCommitted {
                ......
        }
    }
    ```
- `ProverCommitted` marks the end of commitment phase and has the final commitment.
- Now if this is a non-interactive protocol and the prover wishes to compute the challenge, `ProverCommitted` has a method `ProverCommitted::gen_challenge` to generate the challenge by hashing all generators and commitment. It is optional to use this method as the challenge may come from a super-protocol or from verifier. It takes a vector of bytes that it includes for hashing for computing the challenge. In an interactive proof, the prover can simply receive the challenge.
    ```rust
    impl $ProverCommitted {
        .....
        pub fn gen_challenge(&self, mut extra: Vec<u8>) -> FieldElement {
                ......
        }
    }
    ```
- For the 3rd step, the prover calls `ProverCommitted::gen_proof` that takes the challenge (which was either computed by hashing or received from verifier) and the secrets to compute a `Proof`. 
    ```rust
    impl $ProverCommitted {
        .....
        pub fn gen_proof(
            self,
            challenge: &FieldElement,
            secrets: &[FieldElement],
            ) -> Result<$Proof, PSError> {
                ........
        }
    }
    ```
- Now the verifier can verify the proof by calling `Proof::verify` with the Pedersen commitment
    ```rust
    impl $Proof {
        .....
        pub fn verify(
                &self,
                bases: &[$group_element],
                commitment: &$group_element,
                challenge: &FieldElement,
            ) -> Result<bool, PSError> {
                    ......
        }
    }
    ```

To see an example of the above protocol, have a look at these tests for groups [G1](https://github.com/hyperledger/ursa/pull/46/files#diff-168ead191b84bf0856f3d153e9da3ba5R195) and [G2](https://github.com/hyperledger/ursa/pull/46/files#diff-168ead191b84bf0856f3d153e9da3ba5R211).
The above protocol can be used in a proof of knowledge of signature as can be seen [here](https://github.com/hyperledger/ursa/blob/2067d0543c80eb883721863d7886d4e66f1d655d/libzmix/ps/src/pok_sig.rs).    


# Drawbacks
[drawbacks]: #drawbacks

Not aware of any drawback.

# Rationale and alternatives
[alternatives]: #alternatives

- An expensive alternative is to use generic proving systems like SNARKs or Bulletproofs which allow proving much more complex relations.
- If we decided not to use Interactive ZKP ever, we can utilize [Merlin](https://github.com/dalek-cryptography/merlin) for managing the transcript.


# Prior art
[prior-art]: #prior-art

- Projects like [libscapi](https://github.com/cryptobiu/libscapi) and [schnorr-nizk](https://github.com/adjoint-io/schnorr-nizk) provide examples of how to do interactive and non-interactive protocols respectively.


# Unresolved questions
[unresolved]: #unresolved-questions

- Should similar macros exists with logic for doing different kinds of Sigma protocols and their composition.

