- Feature Name: proof-of-knowledge-of-discrete-logarithms
- Start Date: 2019-08-03
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 0.1

# Summary
[summary]: #summary

This is a design document for constructing proofs of knowledge of committed values using the Sigma protocol. The prover is able to construct both interactive and non-interactive proofs of knowledge.

# Motivation
[motivation]: #motivation

Several protocols require proving knowledge of discrete logarithms. Like proving knowledge of messages and randomness in a Pedersen commitment, a vector Pedersen commitment. Such protocols then become part of larger protocols like proving knowledge of a signature or selectively revealing some of the messages in the commitment. These protocols form the basis for anonymous credentials in Hyperledger Aries.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Sigma protocol is used to prove knowledge of discrete logarithms and relations among them. So it can be used to prove knowledge of `x` in g<sup>`x`</sup> without revealing `x`. Or it can be used to prove knowledge of `x`, `y` and `z` in g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup> without revealing `x`, `y` or `z`. A slight variation of the statement is to reveal that g<sub>2</sub>'s exponent is `y` in g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup>. This is accomplished by proving knowledge of `x` and `z` in g<sub>1</sub><sup>`x`</sup>g<sub>3</sub><sup>`z`</sup> and the verifier can check that product of g<sub>2</sub><sup>`y`</sup> and g<sub>1</sub><sup>`x`</sup>g<sub>3</sub><sup>`z`</sup> is indeed g<sub>1</sub><sup>`x`</sup>g<sub>2</sub><sup>`y`</sup>g<sub>3</sub><sup>`z`</sup>.   
Being able to prove such statements is important for anonymous credentials because in those the credential signature is of a form Zg<sub>1</sub><sup>`m1`</sup>g<sub>2</sub><sup>`m2`</sup>g<sub>3</sub><sup>`m3`</sup>...g<sub>n</sub><sup>`n`</sup>h<sup>`r`</sup> where `Z` is derived from signer's secret key and each of m<sub>i</sub> is a credential attribute and `r` is the randomness. There might be more elements in the signature but the aforementioned is of interest here. When using such credentials for zero knowledge proofs, the prover proves knowledge of the signature and the attributes in there. Depending on the signature scheme used, the prover randomizes the signature in a particualar way before creating a proof of knowledge of the messages and signature. This RFC will give examples in the context of PS signatures which are implemented [here](https://github.com/lovesh/signature-schemes/tree/master/ps).  
Interactive Sigma protocols are 3-step, 
- in the first step, prover commits to some randomness and sends it to the verifier.
- in 2nd step, prover gets a challenge from the verifier.
- in 3rd step, prover creates a response involving the randomness, its secret and the challenge and sends to the verifier.
- the verifier will now check correctness of a relation involving the commitment from 1st step, challenge and response.

Non interactive Sigma protocols modify the 2nd step. Rather than getting the challenge from the verifier, they compute the challenge by hashing the commitments from 1st step.   
Consider an example of prover proving knowledge of `x` in g<sup>`x`</sup>. Lets call g<sup>`x`</sup> as Y. Only the prover knows `x` but both prover and verifier know `Y` and `g`. 
- prover picks a random number `r`, creates commitment T = g<sup>`r`</sup> and sends T to the verifier.
- verifier sends challenge c to prover.
- prover computes response `s = r - c*x` and sends `s` to verifier.
- verifier checks if g<sup>`s`</sup>*Y<sup>`c`</sup> equals T. If it does then verifier is convinced of prover's knowledge of `x`.

In the non-interactive version prover will hash g, Y and T to compute challenge c. There are more details and variations for the above protocol which can be seen [here](https://en.wikipedia.org/wiki/Proof_of_knowledge#Sigma_protocols).  Similar protocols can be developed more more complex relations that can be seen [here](https://medium.com/@loveshharchandani/zero-knowledge-proofs-with-sigma-protocols-91e94858a1fb).


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section will describe how such proofs of knowledge are to be constructed in reference to [PS signatures](https://eprint.iacr.org/2015/525), 6.2. The code for the proofs of knowledge are [here](https://github.com/lovesh/signature-schemes/blob/master/ps/src/pok.rs).  
- For the 1st step (commitment), the prover first creates a `ProverCommitting` object. This object holds the generators and random values used during this step. 
- Prover calls `ProverCommitting::commit` for each proof of knowledge that he wishes to do. This function takes a generator (`g`) and optionally takes a randomness. If not supplied, it creates its own.
- Once the prover has committed for all its secrets, it calls `ProverCommitting::finish` method that results in creation of `ProverCommitted` object after consuming `ProverCommitting`. 
- `ProverCommitted` marks the end of commitment phase and has the final commitment.
- Now if this is a non-interactive protocol and the prover wishes to compute the challenge, `ProverCommitted` has a method `ProverCommitted::gen_challenge` to generate the challenge by hashing all generators and commitment. It is optional to use this method as the challenge may come from a super-protocol or from verifier. It takes a vector of bytes that it includes for hashing for computing the challenge. In an interactive proof, the prover can simply receive the challenge.
- For the 3rd step, the prover calls `ProverCommitted::gen_proof` that takes the challenge (which was either computed by hashing or received from verifier) and the secrets (`x`) to compute a `Proof`. 
- Now the verifier can verify the proof by calling `Proof::verify`. 

To see the above protocol in action, have a look at this [test](https://github.com/lovesh/signature-schemes/blob/master/ps/src/pok.rs#L348).  
The above protocol can be used in a proof of knowledge of signature as can be seen [here](https://github.com/lovesh/signature-schemes/blob/master/ps/src/pok.rs#L23). Here the prover is randomizing the signature while keeping it verifiable and proving the knowledge of attributes using the above protocol. Test for [proving knowledge of signature and attributes](https://github.com/lovesh/signature-schemes/blob/master/ps/src/pok.rs#L460), [proving knowledge of signature and attributes and revealing some attributes](https://github.com/lovesh/signature-schemes/blob/master/ps/src/pok.rs#L484).    


# Drawbacks
[drawbacks]: #drawbacks

Not aware of any drawback.

# Rationale and alternatives
[alternatives]: #alternatives

- An expensive alternative is to generic proving systems like SNARKs or Bulletproofs which allow proving much more complex relations.
- If we decided not to use Interactive ZKP ever, we can utilize [Merlin](https://github.com/dalek-cryptography/merlin) for managing the transcript.


# Prior art
[prior-art]: #prior-art

- Projects like [libscapi](https://github.com/cryptobiu/libscapi) and [schnorr-nizk](https://github.com/adjoint-io/schnorr-nizk) provide examples of how to do interactive and non-interactive protocols respectively.


# Unresolved questions
[unresolved]: #unresolved-questions

- Should we provide an implementation of such a protocol in Ursa so that it can be readily consumed? Should that interface support logic for doing different kinds of Sigma protocols and their composition.

