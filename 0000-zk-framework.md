[//]: # (SPDX-License-Identifier: CC-BY-4.0)

- Feature Name: r1cs-select
- Start Date: 2021-11-10
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 1

# Summary
[summary]: #summary
Generic zero knowledge libraries such as bellman, bulletproof, etc come with two components frontends and backends. Application programmers write their business logic using the frontend. The backend handles the underlying cryptography. Traditional zero knowledge libraries do not allow mix and match between different backends and frontends. In this zero knowledge framework one can use rust r1cs library as front end and either bellman or bulletproof as backend. In future versions we plan to support more backends.

# Motivation
[motivation]: #motivation

Finding the appropriate zero knowledge framework (specifically the backend) is a non-trivial design choice for application developers. Different backends have different performance numbers depending on deployment options which might not be obvious to the application developer. Moreover, application developers can have different preferences for different frontends. To make life easy for application developers ideally we should have seemless compatibility between different frontend and backends. In the first version of this framework we support rust r1cs as front end, bellman and bullet proof as backend.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This project demonstrates R1CS logic with backends to Bellman and Bulletproofs.

Use the following commands to execute each step of the process. The steps are setup (bellman), prove, and verify.

Setup

`cargo run -- setup --backend=bellman | tee params.json`

Prove

`echo '"1234"' > witness.hex`
`cargo run -- prove --backend=bellman --parameters=params.json --witness=witness.hex | tee proof.json`

or 

`cargo run -- prove --backend=bulletproofs --witness=witness.hex | tee proof.json`

Verify

`cargo run -- verify --backend=bellman --parameters=params.json --input=proof.json`

or

`cargo run -- verify --backend=bulletproofs --input=proof.json`

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

main.rs: this file contains sample code on how to use the framework, get_circuit function defines the circuit for which zero knowledge proofs will be generated
r1csbellman.rs, r1csbulletproof.rs: modules that convert r1cs spec to underlying backends


# Drawbacks
[drawbacks]: #drawbacks

We have chosen r1cs-select as frontend, which might not be preferred frontend for all users.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?
- For incorporating new protocol implementations what other implementations
  exist and why were they not selected?
- For new protocols, what related protocols exist and why do the not satisfy
  requirements?

# Prior art
[prior-art]: #prior-art

zk-interface is the defacto prior art. It has extensive features. However, it is not being actively maintained and set up is relatively complex.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

# Changelog
[changelog]: #changelog

- [10 Jan 2019] - v2 - a one-line summary of the changes in this version.
