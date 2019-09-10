- Feature Name: ps-signatures
- Start Date: 2019-08-08
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 0.1

# Summary
[summary]: #summary

Pointcheval-Sanders signature scheme from the paper [Short Randomizable signatures](https://eprint.iacr.org/2015/525) with protocol for proof of knowledge of signature. A user can get signature on known or committed messages from the signer and then prove knowledge of signature and optionally reveal some messages under the signature. Implementation in this [PR](https://github.com/hyperledger/ursa/pull/46). This feature depends on the [PR](https://github.com/hyperledger/ursa-rfcs/pull/10) for RFC for proof of knowledge of committed messages in a commitment. 

# Motivation
[motivation]: #motivation

Anonymous credentials consists of issuer's signature over credential attributes with the property that it is efficient to prove knowledge of signature without revealing the signature itself. Additionally, it should be possible to reveal some of the attributes of the credential to the verifier. Anonymous credentials are needed in Hyperledger Aries.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are 3 mainly entities in anonymous credentials, the *issuer* which signs a credential (a vector of attributes) and gives the signature to the *holder* which then uses the signature to prove its knowledge to a *verifier* without revealing the signature. The issuer might not know all the attributes it is signing and the holder might have only given a commitment to those attributes. In this case, the issuer gives a multi-message blind signature to the holder which the holder can unblind. The issuer owns a signing key and a verification key. The verification key is published so that it is accessible by the holder and verifier.  The verification key is meant to support a certain number of messages only meaning that a verification key intended to be used for verifying signature over a vector of 5 messages cannot be used to verify signature over 4 or 6 messages.  
- `SignatureGroup` and `OtherGroup` are 2 type aliases that can refer to groups G1 and G2 or groups G2 and G1 depending on whether feature `G2G1` or `G1G2` is enabled. These features are mutually exclusive. The `SignatureGroup` is the group for the elements of `Signature` or `Sigkey`. `Verkey` contains elements of both groups. 
- The signer calls the `keygen` method to create a 2-tuple of signing key and verification key (`Sigkey`, `Verkey`). The method takes the number of messages to support and a label that is hashed to create generators. The `Verkey` size grows linearly with the number of messages that it has to support but the `Sigkey` has only 1 group element.
    ```rust
    pub fn keygen(count_messages: usize, label: &[u8]) -> (Sigkey, Verkey)
    ```
- The signer can create a new signature on a collection of known messages using `Signature::new`. The number of messages should be compatible with the provided `Verkey`. Regardless of the number of messages being signed, the signature always consists of 2 group elements.
    ```rust
    impl Signature {
        pub fn new(
            messages: &[FieldElement],
            sigkey: &Sigkey,
            verkey: &Verkey,
        ) -> Result<Self, PSError>
    }
    ```
    Refer test [test_signature_all_known_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R122)
- The signer can create a new blind signature on a collection of known and committed messages using `Signature::new_with_committed_messages`. This method accepts a Pedersen commitment (`commitment` below) to the committed messages created by the holder and the collection of known messages. Note that the commitment could be for several messages. This method **must** only be used if there is one or more committed messages.
    ```rust
    impl Signature {
        pub fn new_with_committed_messages(
            commitment: &SignatureGroup,
            messages: &[FieldElement],
            sigkey: &Sigkey,
            verkey: &Verkey,
        ) -> Result<Self, PSError>
    }
    ```
    The signer should verify that the commitment is well formed by executing a proof of knowledge protocol as described in the test [test_sig_committed_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR200).
- Once a holder has received a blind signature created with the `Signature::new_with_committed_messages` method, he unblinds the signature with `Signature::get_unblinded_signature`. For this the holder must have the randomness he used in the commitment he sent to the issuer.
    ```rust
    impl Signature {
        pub fn get_unblinded_signature(&self, blinding: &FieldElement) -> Self
    }
    ```
- To verify a signature, `Signature::verify` must be used. It takes the messages over which the signature is created.
     ```rust
    impl Signature {
        pub fn verify(&self, messages: &[FieldElement], verkey: &Verkey) -> Result<bool, PSError>
    }
    ```
    Refer tests [test_signature_many_committed_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R151) and [test_signature_known_and_committed_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R171) for the unblinding an verification of signature.  
- To prove knowledge of the signature, `PoKOfSignature` is used. The protocol has 2 phases, in the pre-challenge phase, the signature is transformed to an aggregate signature and a PoK of the underlying multi-message is created using `PoKOfSignature::init`. The `revealed_msg_indices` specifies the indices of messages that are revealed to the verifier and hence do not need to be included in the proof of knowledge.
    ```rust
    impl PoKOfSignature {
        pub fn init(
            sig: &Signature,
            vk: &Verkey,
            messages: &[FieldElement],
            revealed_msg_indices: HashSet<usize>,
        ) -> Result<Self, PSError>
    }
    ```
- After the challenge is generated (or received in case of an interactive protocol), `PoKOfSignature::gen_proof` is used to create the proof `PoKOfSignatureProof`. 
    ```rust
    impl PoKOfSignature {
        pub fn gen_proof(self, challenge: &FieldElement) -> Result<PoKOfSignatureProof, PSError>
    }
    ```
- The verifier can then verify the proof using `PoKOfSignatureProof::verify`.
    ```rust
    impl PoKOfSignatureProof {
        pub fn verify(
            &self,
            vk: &Verkey,
            revealed_msgs: HashMap<usize, FieldElement>,
            challenge: &FieldElement,
        ) -> Result<bool, PSError>
    }
    ```
    Refer the tests [test_PoK_sig](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR252) and [test_PoK_sig_reveal_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR276) for the proof of knowledge of signature and proof of knowledge of signature with revealing some messages respectively.
- Another test that shows a more elaborate scenario where a signature is requested on a mix of known and committed messages and then the knowledge of that signature is proved in addition to revealing some messages is shown in test [test_scenario_1](https://github.com/hyperledger/ursa/pull/46/files#diff-3b79c2d1f6accb7035421822bd62d6c6R10).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation that the RFC is describing is the implementation of sections 6.1 and 6.2 of the [PS signatures paper](https://eprint.iacr.org/2015/525) and an attempt has been made to use the same symbol names in the code as used in the paper. Some implementation optimizations have been done like the use of multi-pairings or multi-exponentiations (both constant and variable time). The PoK protocol for Pedersen commitments used is described in this [RFC](https://github.com/hyperledger/ursa-rfcs/pull/10). 

The `Signature` implementation is based on section 6.1 of the paper and implements "A Multi-Message Protocol" with the enhancement being the support of signer-known messages also as explained below:
    - Say the verification key supports 5 messages, the relevant generators are Y<sub>1</sub>, Y<sub>2</sub>, Y<sub>3</sub>, Y<sub>4</sub>, Y<sub>5</sub>
    - The holder wants to reveal only last 3 message to the issuer but commits to the first 2 messages in a commitment C = g<sup>t</sup>Y<sub>1</sub><sup>m1</sup>Y<sub>2</sub><sup>m2</sup> (t is the randomness and should be preserved for unblinding the sig) and sends C to the issuer (along with a proof of knowledge).
    - The issuer now creates D = C * Y<sub>3</sub><sup>m3</sup>Y<sub>4</sub><sup>m4</sup>Y<sub>5</sub><sup>m5</sup> where m3, m4 and m5 are known to him. The issuer will now choose a random u and create the signature (g<sup>u</sup>, (XD)<sup>u</sup>) as described in the paper.
    - The holder will unblind the signature as described in the paper.

The `PoKOfSignature` is based on section 6.2 of the paper and the implementation follows the paper almost as-it-is. However, the product of multiple pairings is replaced with a multi-pairing for efficiency. The proof of knowledge (symbol pie in the paper) is done using [this protocol](https://github.com/hyperledger/ursa/pull/46/files#diff-168ead191b84bf0856f3d153e9da3ba5R14). 


# Drawbacks
[drawbacks]: #drawbacks

Not aware of any.

# Rationale and alternatives
[alternatives]: #alternatives

- This is meant to be one of the supported signature schemes for anonymous credentials. 
- BBS+ is another signature scheme to be supported but PS signatures can potentially be used to build threshold signatures (there is an unpublished work).


# Prior art
[prior-art]: #prior-art

The RFC author is not aware of any implementation (open or closed source) of PS signatures. 

# Unresolved questions
[unresolved]: #unresolved-questions

- It is expected that proof of knowledge of signature API will be refined further through broader discussion.
- Proving properties about signed messages (range proof) or relationships among signed messages (equality, inequality) has been considered out of scope.

# Changelog
[changelog]: #changelog

- 
