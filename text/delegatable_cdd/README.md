- Feature Name: Delegatable anonymous credentials, CDD scheme
- Start Date: 2019-09-06
- RFC PR: (leave this empty)
- Ursa PR: https://github.com/hyperledger/ursa/pull/51
- Version: (use whole numbers: 1, 2, 3, etc)

# Summary
[summary]: #summary

This is an implementation of delegatable anonymous credentials as described in
the CCS 17 paper [Practical UC-Secure Delegatable Credentials with Attributes
and Their Application to
Blockchain](https://acmccs.github.io/papers/p683-camenischA.pdf) by Camenisch
et.al. This allows issuers to delegate issuance rights to other entities such
that the presentations from their issued credentials do not reveal their
identities but only the identity of the the first issuer in the delegation chain.
This also allows issuers at each level to add attributes in their issued
credentials. However, downstream issuers in the delegation do learn about all
the upstream issuers, i.e. there is no privacy among issuers. This
implementation is not UC-secure but only implements the delegatable credentials
from the paper.

# Motivation
[motivation]: #motivation

Delegatable credentials are useful in saving operational costs of issuance and
can also make issuance cheaper. If an organization that wants to issue
credentials has only the head of the organization (or the relevant individual)
issuing the credential, it becomes a bottleneck. But if the head decides to
delegate the issuance rights to its subordinates, then the credential receivers
can check that credential comes from a delegatee of the head. In addition,
delegation provides privacy to issuers who might want to remain anonymous to
verifiers. This [Aries RFC](https://github.com/hyperledger/aries-rfcs/pull/218)
goes into more details of delegatable credentials.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The entity delegating the credential issuance rights and identified to the
verifier is called the *root issuer* or the *root delegator*. All issuers have a
signing and verification key but only the root issuer's verification key is
known to the verifier. The linear structure starting at the root issuer and
consisting of subsequent issuers involved in delegation is called the *issuer
chain*. For reasons explained later, an issuer is either an odd-level issuer or
an even-level issuer depending on its position in the issuer chain. The *root
issuer* is always an even-level issuer. Similarly, the linear structure starting
at the root issuer's credential and consisting of subsequent issuers'
credentials is called the *credential chain*. A credential is either an odd
level credential or even-level credential depending on the issuer; an even-level
issuer always issues an odd-level credential and vice versa. Each credentials in
the credential chain is a also called a *link* and has a level which is a
monotonically increasing number starting from 1 with no gaps. The credential
(or link) issued by the root issuer has level 1, the subsequent link will have level
2, and so on. During delegation (credential issuance), the delegator, in addition
to issuing a credential, sends its credential chain also to the delegatee. A
presentation created from the delegated credential is called *attribute token*.

Because the verification keys of even-level issuers are cryptographically different
from the verification keys of odd-level issuers, the verification key of even-level
issuers are called even-level verification keys and of odd-level issuers are called
odd-level verification keys. When an entity wants to be an issuer at both even
and odd-levels, it must own two keypairs, one for odd-level and one for even
level. Throughout this document and the code, the terms sigkey and verkey are
used to refer to signing key and verification key.
Lets look at an example. Consider an issuer chain of Alice -> Bob -> Carol ->
Dave where Alice is the root issuer. Alice will issue a delegatable credential
to Bob, Bob will issue a delegatable credential to Carol, Carol will issue a
delegatable credential to Dave and Dave will create a presentation from his
credential that will be verified by Eva. Alice is an even-level issuer who has
an even-level verkey, she will issue an odd-level credential with level 1 to Bob
who has an odd-level verkey and he will issue an even-level credential with
level 2 to Carol who has an even-level verkey and she will issue an odd-level
credential with level 3 to Dave who has an odd-level verkey. If Dave also
requests credential from an odd-level issuer, he will need to have an even-level verkey
(with a different secret key).  
During delegation, the delegator also signs the delegatee's verkey as one of the
attributes of the credential. Thus the level 1 credential above (issued to Bob)
will have Bob's odd-level verkey as one of the attributes, while the level 2
credential issued to Carol will have Carol's even-level verkey as one of the
attributes and so on. When Alice delegates to Bob, she sends one credential to
Bob, say C1, when Bob delegates to Carol, he creates a new credential C2 for
Carol and sends C1 in addition to C2 to Carol. Similarly when Carol will issue a
credential, she will also send C1 and C2 in addition to her issued credential.

The reason for the distinction between odd and even-levels is the use of *structure
preserving cryptography*. A cryptographic scheme is called structure preserving
if its all public inputs and outputs consist of group elements of bilinear
groups and the functional correctness can be verified only by computing group
operations, testing group membership and evaluating pairing product
equations.[^1] As a result, the messages are also group elements (of G1 or G2)
unlike some other schemes where the messages are elements of finite fields. The
verification key will be in a different group than the message, so if the message
is in G1, verification key will be in G2, and vice-versa. We saw above that
the delegatee's verkey is signed as part of the delegated credential. So if the
delegator has verkey in group G2, it will sign messages in group G1 which means
the delegatee needs to have a verkey in group G1. Similarly when this delegatee
issues a credential, it will sign the subsequent delegatee's verkey which needs
to be in G2 and these alternations will continue. Two variations of Groth
signatures (described in the paper) are used, the algorithms are same but one,
called `Groth1`, has messages in G1 and verkey in G2 and the other, called
`Groth2`, has messages in G2 and verkey in G1. even-level issuers use Groth1
signature and have verkey in group G2 while odd-level issuers use Groth2
signatures and have verkey in G1. Signing key is a single field element whereas
the verification key size depends linearly on the maximum number of supported
attributes.

## API

### Groth1 and Groth2 signatures

- First, some public system parameters need to be generated which are used by
  all system participants for generating keys, signing, verification, etc. These
  parameters can be generated by calling `GrothS1::setup` or `GrothS2::setup`.
  `setup` takes the maximum number of attributes that need to be supported. Keep
  it one more than the number you want to support to accommodate the public key.
    ```rust
    let max_attrs = 10;
    let label = "some label for generating NUMS".as_bytes();
    let params1: Groth1SetupParams = GrothS1::setup(max_attrs, label);
    // OR for Groth2
    let params2: Groth2SetupParams = GrothS2::setup(max_attrs, label);
    ```
- Signing keys can be generated by calling `GrothS1::keygen` or `GrothS2::keygen`. Each takes the corresponding setup parameters.
    ```rust
    let (sk1: GrothSigkey, vk1: Groth1Verkey) = GrothS1::keygen(&params1);
    // OR for Groth2
    let (sk2: GrothSigkey, vk2: Groth2Verkey) = GrothS2::keygen(&params2);
    ```
- A new signature can be created by calling `Groth1Sig:new` or `Groth2Sig:new`. 
    ```rust
    // `msgs` is a collection of elements in G1 
    let sig: Groth1Sig = Groth1Sig::new(msgs.as_slice(), &sk1, &params1)?;
    // OR for Groth2
    let sig: Groth2Sig = Groth2Sig::new(msgs.as_slice(), &sk2, &params2)?;
    ```
- An existing signature can be randomized by calling `randomize` on the signature with a randomly chosen field element.
    ```rust
    let r = FieldElement::random();
    let sig_randomized = sig.randomize(&r);
    ```
- 2 methods for signature verification, `verify` and `verify_fast`, both with the same API. `verify` computes several pairings to verify the signature whereas `verify_fast` does only 1 big multi-pairing. `verify_fast` has a negligibly higher probability of failure than `verify`. It's `O(n)/|F|` where `n` is the number of attributes and `|F|` is the order of the subgroup, `|F|` usually is > 2<sup>250</sup>.

    ```rust
    sig.verify(msgs.as_slice(), &vk, &params)?
    // Or for fast verification
    sig.verify_fast(msgs.as_slice(), &vk, &params)?
    ```

### Issuers and delegation

- Issuers are instantiated by calling `EvenLevelIssuer::new` or
  `OddLevelIssuer::new` by passing their level to the `new` function. Root
  issuers is at level 0 so can be instantiated by `EvenLevelIssuer::new(0)`.

    ```rust
    // A level 0 issuer (root issuer)
    let l_0_issuer = EvenLevelIssuer::new(0)?;
    // A level 1 issuer
    let l_1_issuer = OddLevelIssuer::new(1)?;
    // A level 2 issuer
    let l_2_issuer = EvenLevelIssuer::new(2)?;
    ```

- There is a another struct defined called `RootIssuer` which is like a proxy to
  `EvenLevelIssuer`.
    ```rust
    pub struct RootIssuer {}

    pub type RootIssuerVerkey = EvenLevelVerkey;

    impl RootIssuer {
        pub fn keygen(setup_params: &Groth1SetupParams) -> (Sigkey, RootIssuerVerkey)

        pub fn delegate(
            mut delegatee_attributes: G1Vector,
            delegatee_vk: OddLevelVerkey,
            sk: &Sigkey,
            setup_params: &Groth1SetupParams) -> DelgResult<CredLinkOdd>
    }
    ```

- Issuers generate their keys with `EvenLevelIssuer::keygen` or
  `OddLevelIssuer::keygen`. `GrothSigkey`, `Groth1Verkey` and `Groth2Verkey`
  from above are type aliased to `Sigkey`, `EvenLevelVerkey` and
  `OddLevelVerkey` respectively. Also the root issuer's verkey is always an
  `EvenLevelVerkey`, `RootIssuerVerkey` is type aliased to `EvenLevelVerkey`.

    ```rust
    // Params should be generated only once and then published in public
    let params1 = GrothS1::setup(max_attributes, label);
    let params2 = GrothS2::setup(max_attributes, label);

    // Even level issuer's keys
    let (l_even_issuer_sk: Sigkey, l_even_issuer_vk: EvenLevelVerkey) = EvenLevelIssuer::keygen(&params1);
    // Odd level issuer's keys
    let (l_odd_issuer_sk: Sigkey, l_odd_issuer_vk: OddLevelVerkey) = OddLevelIssuer::keygen(&params2);

    // A root issuer could either use EvenLevelIssuer::keygen for generating its keys or use RootIssuer::keygen
    let (root_issuer_sk: Sigkey, root_issuer_vk: RootIssuerVerkey) = RootIssuer::keygen(&params1);
    ```

- A credential is a called a link and the credentials issued by
  `EvenLevelIssuer`s are called `CredLinkOdd` and credentials issued by
  `OddLevelIssuer`s are called `CredLinkEven`.
- Issuers can delegate by calling `delegate` method that takes the attributes to
  sign, who to delegate to resulting in a credential. The even-level credential
  is called `CredLinkEven` and odd-level credential is called `CredLinkOdd`. 

    ```rust
    // An even-level issuer delegating to an issuer with odd-level verkey `delegatee_verkey` resulting in an odd-level credential
    let attributes_1: G1Vector = ....;
    let cred_link_1: CredLinkOdd = even_level_issuer
            .delegate(
                attributes_1.clone(),
                delegatee_verkey.clone(),
                &delegator_sigkey,
                &params1,
            )?;
    
    // An odd-level issuer delegating to an issuer with even-level verkey `delegatee_verkey` resulting in an even-level credential
    let attributes_2: G2Vector = ....;
    let cred_link_2: CredLinkEven = odd_level_issuer
            .delegate(
                attributes_2.clone(),
                delegatee_verkey.clone(),
                &delegator_sigkey,
                &params2,
            )?;
    
    // The root issuer can either use the `EvenLevelIssuer`'s delegate method or use the delegate function on `RootIssuer`
    let cred_link: CredLinkOdd = RootIssuer::delegate(
                attributes.clone(),
                delegatee_verkey.clone(),
                &root_issuer_sk,
                &params1,
            )
    ```

- A link stores its associated `level`, `attributes` and `signature`. The last
  element of `attributes` is the verification key of the delegatee and the
  signature is on `attributes`.
    ```rust
    pub struct CredLinkOdd {
        pub level: usize,
        pub attributes: G1Vector,
        pub signature: Groth1Sig,
    }
    ```
- To verify the correctness of link, call `verify` on it with delegator public
  key, delegatee public key and setup params.  
    ```rust
    // For odd-level links
    cred_link_1.verify(&delegatee_vk, &delegator_vk, &params1)
    // For even-level links
    cred_link_2.verify(&delegatee_vk, &delegator_vk, &params2)
    ```
- The chain of credentials is kept in `CredChain` which internally has 2 lists,
  1 for odd-level links and 1 for even. 
    ```rust
    pub struct CredChain {
        pub odd_links: Vec<CredLinkOdd>,
        pub even_links: Vec<CredLinkEven>,
    }
    ```
- Even or odd-level links can be added by calling `extend_with_even` or
  `extend_with_odd` on the chain. These methods will verify that the link has
  appropriate level.
    ```rust
    let mut chain_1: CredChain = CredChain::new();
    // Add an odd link to chain
    chain_1.extend_with_odd(cred_link_1)?
    // Add an even link to chain
    chain_1.extend_with_even(cred_link_2)?
    ```
- To verify that all delegations are valid in the chain, call
  `verify_delegations` on the chain. It takes even-level and odd-level
  verkeys involved in the delegation. In addition to verifying each link with a call
  to its `verify` method, it will also check that there are no gaps or duplicate levels
  in the chain.
    ```rust
    chain_1
        .verify_delegations(
            vec![&level_0_issuer_vk, &level_2_issuer_vk],   // even-level verkeys
            vec![&level_1_issuer_vk],   // odd-level verkeys
            &params1,
            &params2
        )?
    ```

- `CredChain`'s size can be found out by calling `CredChain::size`. The number of
  odd or even links can be found by calling `CredChain::odd_size` or
  `CredChain::even_size`.
- When only some links of the `CredChain` are needed,
  `CredChain::get_truncated_chain` can be called that returns a shorter chain of
  desired size.
    ```rust
    // Returns a shorter version of chain_1 of size 5, i.e. sum of odd and even links is 5.
    let chain_1_1: CredChain = chain_1.get_truncated(5)?;
    ```

### Attribute tokens

- Call `AttributeToken::new` with the `CredChain` and setup parameters to initialize attribute token generation.
    ```rust
    let params1: Groth1SetupParams = ...;
    let params2: Groth2SetupParams = ...;
    let chain_1: CredChain = ...;
    let at: AttributeToken = AttributeToken::new(&chain_1, &params1, &params2);
    ```
- Attribute tokens are generated for all levels of the given chain. To generate
  attribute token for a subset of the chain, first get a truncated version of
  the chain by calling `CredChain::get_truncated_chain` and then use that to
  create the attribute token. eg. if chain length is 10 and attribute token is
  needed for the first 5 levels of the chain, get a truncated chain with
  `CredChain::get_truncated_chain(5)` and then use the result in
  `AttributeToken::new`.
    ```rust
    let chain_1: CredChain = ...;
    let chain_1_1: CredChain = chain_1.get_truncated(5)?;
    let at: AttributeToken = AttributeToken::new(&chain_1_1, &params1, &params2);
    ```
- Attribute token generation happens in 2 phases. In the commitment phase all
  commitments are generated and the response phase accepts the challenge. These
  are intentionally decoupled so that this can be used in a higher level
  protocol.
- To generate the commitment, call the `commitment` method with the indices of the
  revealed attributes at each level. The indices at each level are a set of
  integers. When not revealing any attributes at a level, pass an empty HashSet at
  that level. The `commitment` method results in an `AttributeTokenComm` object
  which can be converted to bytes by calling `to_bytes` so that those bytes can
  be used in challenge computation. Or for testing,
  `AttributeToken::gen_challenge` can be used that takes `AttributeTokenComm`
    ```rust
    let chain_1: CredChain =  chain of size 2...;
    let mut at: AttributeToken = AttributeToken::new(&chain_1, &params1, &params2);

    // No attributes are revealed at any level
    let com_1: AttributeTokenComm = at.commitment(vec![HashSet::<usize>::new(); 2])?;

    // Reveal attributes 1 and 3 at 1st level, attributes 2, 3, 4 at level 2
    let mut revealed_attr_indices_1 = HashSet::<usize>::new();
    revealed_attr_indices_1.insert(1);
    revealed_attr_indices_1.insert(3);

    let mut revealed_attr_indices_2 = HashSet::<usize>::new();
    revealed_attr_indices_2.insert(2);
    revealed_attr_indices_2.insert(3);
    revealed_attr_indices_2.insert(4);
    let com_2: AttributeTokenComm = at
            .commitment(vec![
                revealed_attr_indices_1.clone(),
                revealed_attr_indices_2.clone(),
            ])?
    
    // To generate bytes for adding to challenge, call `AttributeTokenComm::to_bytes`
    let com_2: AttributeTokenComm = ...;
    let bytes = com_2.to_bytes();
    ```
- To generate response, call `AttributeToken::response`, passing the commitment, signing key of the entity creating the presentation, challenge and the even and odd-level verkeys.
    ```rust
    let resp_2: AttributeTokenResp = at
            .response(
                &com_2,
                &l_2_issuer_sk.0,
                &challenge,
                vec![&l_2_issuer_vk],
                vec![&l_1_issuer_vk],
            )?
    ```
- The verifier with then call `AttributeToken::reconstruct_commitment` to
  reconstruct the commitment that the prover would have created. He can now hash
  it into the challenge to compare with the prover's challenge.
    ```rust
    let L = 2;  // size of delegation chain
    // com_2 and resp_2 are the commitment and response sent by the prover.
    // vec![HashSet::<usize>::new(); 2] indicates that no attributes were revealed.
    let recon_com_2: AttributeTokenComm = AttributeToken::reconstruct_commitment(
            L,
            &com_2,
            &resp_2,
            &challenge,
            vec![HashSet::<usize>::new(); 2],
            &root_issuer_vk,
            &params1,
            &params2,
        )?;
    
    // If attributes were revealed, like in the example above, the call would be
    let recon_com_2: AttributeTokenComm = AttributeToken::reconstruct_commitment(
            L,
            &com_2,
            &resp_2,
            &challenge,
            vec![
                revealed_attr_indices_1,
                revealed_attr_indices_2,
            ],
            &root_issuer_vk,
            &params1,
            &params2,
        )?;
    ```
- Commitment computation can be sped up by using precomputation of some of the
  pairings. Precomputation can be done on the setup parameters which can be used
  for computing commitments involving any `CredChain` or it can be done on
  signatures as well which makes those precomputations relevant to that
  `CredChain` only.  
  Thus the commitment can be created using 3 methods, either
  `AttributeToken::commitment` that does not accept any precomputation,
  `AttributeToken::commitment_with_precomputation_on_setup_params` that only
  accepts precomputation done on setup params (using
  `PrecompOnSetupParams::new`) and
  `AttributeToken::commitment_with_precomputation` that accepts precomputation
  done on setup params (using `PrecompOnSetupParams::new`) and on `CredChain`
  using `PrecompOnCredChain::new`

    ```rust
    // Precomputation on setup params
    let precomputed_setup_params: PrecompOnSetupParams = PrecompOnSetupParams::new(&params1, &params2);

    // Precomputation on credential chain
    let precomp_sig_vals: PrecompOnCredChain = PrecompOnCredChain::new(&chain, &params1, &params2)?;

    // Computing AttributeTokenComm using PrecompOnSetupParams
    let at: AttributeToken = ...;
    let com_precomp_setup: AttributeTokenComm = at
                .commitment_with_precomputation_on_setup_params(
                    vec![HashSet::<usize>::new(); i],
                    &precomputed_setup_params,
                )?
    
    // Computing AttributeTokenComm using PrecompOnSetupParams and PrecompOnCredChain
    let com_precomp: AttributeTokenComm = at
                .commitment_with_precomputation(
                    vec![HashSet::<usize>::new(); i],
                    &precomputed_setup_params,
                    &precomp_sig_vals,
                )?
    ``` 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Most of the implemented logic follows the paper with the corrections called out
in the code.  
At several places, computations like e(a, b)<sup>c</sup> have been replaced with
e(a<sup>c</sup>, b) since they are about 30% faster. Attempts are made to
replace computations in G2 with computations in G1 inside pairings.  
Regarding the implementation of `verify_fast` function, it applies this
observation to pairings: if it needs to be checked that a == b and c == d and e
== f, then choose a random number `r` and check whether (a-b) + (c-d)\*r +
(e-f)\*r<sup>2</sup> == 0. This is because this degree 2 polynomial can have at
most 2 roots and if r is not a root then all the coefficients must be 0. If r is
a randomly chosen element from a sufficiently large set, then the chances of r
being a root are negligible.   
In a pairing scenario if verifier had to check if e(a,b) = 1, e(c, d) = 1 and e(f, g) = 1, 
    - pick a random value r in Z<sub>p\*</sub> and 
    - check e(a,b) \* e(c,d)<sup>r</sup> \* e(f,g)<sup>r<sup>2</sup></sup> equals 1 - e(a,b) \* e(c,d)<sup>r</sup> \* e(f,g)<sup>r<sup>2</sup></sup> = e(a,b) \*
    e(c<sup>r</sup>, d) \* e(f<sup>r<sup>2</sup></sup>, g). Exponent moved to 1st element of pairing since computation in group G1 is cheaper. 
    - Now use a single multi-pairing rather than 3 pairings to compute e(a,b) \*
    e(c<sup>r</sup>, d) \* e(f<sup>r<sup>2</sup></sup>, g)


# Drawbacks
[drawbacks]: #drawbacks

This solution should not be used when privacy among issuers is desired. Secondly,
since each issuer can add any attributes in its issued credential, if policies
for issuance are not properly defined, malicious issuers might be able to issue
credentials with unintended attributes (by the delegator) leading to privilege
escalation. 

# Rationale and alternatives
[alternatives]: #alternatives

An expensive alternative to delegatable credentials is the holder to get
credential directly from the root issuer. The expensiveness of this is not just
computational but operational too.  
There are some alternatives which offer privacy among issuers as well but have
different tradeoffs. [Delegatable Attribute-based Anonymous Credentials from
Dynamically Malleable Signatures](https://eprint.iacr.org/2018/340) requires
the presence of a trusted third party. [Delegatable Anonymous Credentials from
Mercurial Signatures](https://eprint.iacr.org/2018/923) does not support
credentials with attributes.


# Prior art
[prior-art]: #prior-art

Delegatable anonymous credentials have been explored since the last decade and
the first efficient (somewhat) came in 2009 by Belenkiy et al. in "Randomizable
proofs and delegatable anonymous credentials". Although this was a significant
efficiency improvement over previous works, it was still impractical. Chase et
al. gave a conceptually novel construction of delegatable anonymous credentials
in 2013 in "Complex unary transformations and delegatable anonymous credentials"
but the resulting construction was essentially as inefficient as that of
Belenkiy et al.   

# Unresolved questions
[unresolved]: #unresolved-questions

- It is expected that the API can be more refined, e.g., it may make sense to use
  Rust enums for wrapping EvenLevel(Issuer/Verkey).
- Should `verify` be removed in favour of `verify_fast`?
- Should the credential chain support more methods?
- Can the protocol be made more efficient by using recent research in structure
  preserving cryptography?

# Changelog
[changelog]: #changelog



[^1]: [Structure-Preserving Cryptography](https://www.iacr.org/archive/asiacrypt2015/94520356/94520356.pdf)