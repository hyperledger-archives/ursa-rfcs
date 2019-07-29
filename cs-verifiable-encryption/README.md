- Feature Name: cs-verifiable-encryption
- Start Date: 2019-07-24
- RFC PR: 
- Ursa Issue: 
- Version: 1

# Summary
[summary]: #summary

[Camenisch-Shoup verifiable encryption](https://www.shoup.net/papers/verenc.pdf) is a protocol for proving "properties" 
about encrypted data. The "properties" are discrete log relations and the encryption is public key encryption. The party 
creating the ciphertext proves in zero knowledge that the ciphertext can be decrypted by the designated party. The RFC 
talks about 2 things, the encryption scheme itself and a protocol to prove that the encrypted data satisfies specific 
discrete log relation(s). The CS Verifiable Encryption is the most efficient scheme for verifiable encryption known to the author.

# Motivation
[motivation]: #motivation

Zero Knowledge Proofs (ZKP) provide anonymity but in some cases the verifier of the ZKP agrees on provisional anonymity meaning 
that on pre-agreed terms, the anonymity of the prover can be broken. To break this anonymity the verifier takes the proof to a mutually 
trusted 3rd party who is capable of deanonymizing the prover. Hyperledger Aries will like to use this capability with their anonymous credentials. Verifiable encryption will be one of the predicates to satisfy while creating proofs from credentials. Like the way we have proof requests that say "reveal attributes `a` and `b`, prove `c` lies in `[m, n]`, prove `d` is one of these ", there will be an additional predicate "..., prove that you have encrypted attribute `e` and `f` for the auditor". The next 2 sections have details on how that will be achieved.   
Consider an example of an online forum where posting requires presenting a ZKP 
of required age from a credential that the forum (verifier) verifies. Now, the forum might have rules against posting illicit content and wants to identify the poster in such cases. The forum will thus demand that the poster in addition to presenting a ZKP of required age, also encrypts its SSN (or other government or reliable ID) **that is present in the credential** for an external auditor. In case of an illicit post, the forum can identify the poster with the auditor's help. But unless the auditor decrypts the ciphertext, the forum cannot identify the poster.  
Another example is of an ecommerce site that demands that the buyer has certain characteristics but does not need to learn the shipping address. The ecommerce site will request a proof from the buyer where he proves using a credential that he has the desired characteristics and correctly encrypts the shipping address **from the credential** for the shipping company. Now the ecommerce site cannot learn the address but only the shipping company can.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

*Some terminology*: The entity which can decrypt the ciphertext is called *auditor*. The entity demanding a proof (ZKP) is 
the *verifier* and the entity creating the proof is called the *prover*.   
The *prover* holds 1 or more credentials and creates proof using them. The *auditor* has a public key for encryption and both the *prover* and *verifier* have access to it. This is a long term key and the *auditor* can have the same public key for several independent provers and verifiers. It is important that the public key should be unambiguously known to both the *prover* and *verifier*. One way to achieve that would be for the *auditor* to put it on a tamper evident database accessible to both the *prover* and *verifier*. A blockchain is an eligible option. Now when the *verifier* is only allowing for provisional anonymity (like in examples above), the prover while creating the ZKP, encrypts a pre-decided (between *prover* and *verifier*) attribute(s) from the credential(s) using the *auditor*'s public key. The *verifier* will now verify the ZKP and also use *auditor*'s public key to check that the pre-decided attribute(s) from the credential(s) used to create the ZKP have been encrypted and the *auditor* when asked can decrypt them.  
**Note**: The *prover* should never encrypt his link secret for the auditor.
The verifiable encryption follows the Sigma protocol approach where the prover first creates the ciphertext of the attributes and ciphertext of blindings for those attributes. These blindings should match the ones used in the overall proof. After generating/receiving the challenge, it computes the response. The verifier will then use challnge and response to reconstruct the ciphertext over blindings and check for equality. The reason to structure it this way is for composability such that the main proving protocol should be able to use it.  
This verifiable encryption is meant to be treated as a predicate.  
The next section will describe the API of this protocol.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This protocol has been implemented in this [PR](https://github.com/hyperledger/ursa/pull/40). The PR contains tests to demonstrate how the protocol works.

1. The auditor can create his encryption keypair using `CSKeypair::new`. This method takes the maximum number of attributes that can be encrypted using the keypair. The following example creates a keypair whose public key can be used to encrypt at max 3 attributes. Trying to encrypt more will result in an error.  The `CSKeypair` is an struct of 2 fields, a public key `CSEncPubkey` and a private key `CSEncPrikey`.  

    ```rust
    let keypair: CSKeypair = CSKeypair::new(3)?;
    let pub_key: &CSEncPubkey = &keypair.pub_key;
    let pri_key: &CSEncPrikey = &keypair.pri_key;
    ```

1. Say the prover now wants to encrypt 3 attributes. He creates a sequence of his 2 attributes uses the above keypair's public key to call `CSKeypair::encrypt`. The `CSKeypair::encrypt` in addition to the public key takes a `label` which is a public value and indicates the condition underwhich the ciphertext should be decrypted. The same `label` should be used while decrypting this ciphetext. The `CSKeypair::encrypt` function returns a `CSCiphertext` if there is no error. 
    
    ```rust
    let pub_key: &CSEncPubkey = &keypair.pub_key;
    let attributes = vec!["Attribute1", "Attribute2", "Attribute3"];
    let ciphertext: CSCiphertext = CSKeypair::encrypt(&attributes, "only_decrypt_when_*".as_bytes(), &pub_key)?;
    ```

1. To decrypt the ciphertext from above, the auditor will use the label, ciphertext and his keys in `CSKeypair::decrypt`. If successful, it returns the decrypted attributes

    ```rust
    let decrypted_messages: Vec<BigNumber> = CSKeypair::decrypt(label, &ciphertext, &keypair.pub_key, &keypair.pri_key)?;
    ```

1. The prover can now execute the verifiable encryption protocol. He will first either create or retrieve `blindings` for the attributes he wants to encrypt and then create the ciphertext over the attributes, over the `blindings` using `CSKeypair::encrypt_and_prove_phase_1`. He also gets some random values that he will use while creating the response to the challenge.

    ```rust
    let (ciphertext, blindings_ciphertext, r, r_tilde) = CSKeypair::encrypt_and_prove_phase_1(
            &attributes,
            &blindings,
            label,
            &keypair.pub_key,
        )?;
    ```    

    The prover sends both `ciphertext` and `blindings_ciphertext` to the verifier. Incase of a non-interactive ZKP, both `ciphertext` and `blindings_ciphertext` should be hashed in the challenge.

1. When the prover receives the challenge, it creates the response to it using `CSKeypair::encrypt_and_prove_phase_2`. The challenge could have come from a verifier in case of an interactive ZKP or it could be the result of hashing all the commitments of the main proving protocol (including this one).

    ```rust
    let r_hat: BigNumber = CSKeypair::encrypt_and_prove_phase_2(
            &r,
            &r_tilde,
            &challenge,
            &keypair.pub_key,
            Some(&mut ctx),
        )?;
    ```    

1. The verifier reconstructs the ciphertext over blindings and matches the received ciphertext over blindings using `CSKeypair::reconstruct_blindings_ciphertext`. The `attribute_responses` in the example below is sent by the prover as part of the main protocol for all attributes. The verifier picks those that were encrypted.

    ```rust
    let blindings_ciphertext: CSCiphertext = CSKeypair::reconstruct_blindings_ciphertext(
            &ciphertext,
            &attribute_responses,
            &r_hat,
            &challenge,
            label,
            &keypair.pub_key,
        )
    ```

1. The private key `CSEncPrikey`, public key `CSEncPubkey` and ciphertext `CSCiphertext` are serializable using serde-json.

# Drawbacks
[drawbacks]: #drawbacks

The prover should ensure that the auditor will not decrypt his attributes unless the pre-agreed terms are satisfied. If the verifier tricks the prover to encrypt his attributes for a colluding auditor then the prover's privacy is broken. Also the prover should be careful about what attributes he is encrypting and not encrypt sensitive attributes like link secret. The sensitivity of same attributes maybe different depending on the auditor. Though Ursa APIs should not (and cannot) dictate which attributes cannot be encrypted, the consumer of Ursa should take measures to prevent accidental encryption of 
such attributes. Eg, Hyperledger Aries can expose 2 methods for verifiably encrypting, like `ver_enc` and `ver_enc_unsafe`, `ver_enc` will not allow encrypting link secret.

# Rationale and alternatives
[alternatives]: #alternatives

- CS Verifiable Encryption offers chosen ciphertext security.
- Even though the cryptographic scheme dates back to 2003, the RFC author does not know of a more efficient scheme or a 
more recent scheme either. 
- A possible but inefficient alternative is to use a generic proving system like ZK-SNARKS or Bulletproof and prove 
that the ciphertext can be decrypted and has the desired properties in an arithmetic circuit. The encryption scheme can 
be ElGamal. But the arithmetic circuit will need to have constraints for decryption which will be expensive. 
- No publicly implementation exists of this scheme. The [Idemix site(https://idemix.wordpress.com/) specifies a 
[link to the Idemix library](http://prime.inf.tu-dresden.de/idemix/) but it times out.

# Prior art
[prior-art]: #prior-art

As the CS Verifiable Encryption paper mentions, the previous schemes do not provide chosen ciphertext 
security and are less efficient. Most of the content in the RFC is taken from CS Verifiable Encryption 
[paper](https://www.shoup.net/papers/verenc.pdf), a research report on the ["Specification of the Identity 
Mixer Cryptographic Library"](https://domino.research.ibm.com/library/cyberdig.nsf/papers/EEB54FF3B91C1D648525759B004FBBB1/$File/rz3730_revised.pdf), and some other [introductory](https://idemix.wordpress.com/2009/08/18/quick-intro-to-credentials/) 
and [architecture level](https://www.freehaven.net/anonbib/cache/idemix.pdf) documents. 


# Unresolved questions
[unresolved]: #unresolved-questions

- Please refer the [PR](https://github.com/hyperledger/ursa/pull/40) to see the concerns.
