- Feature Name: ursa-python-wrappers
- Start Date: 2018-12-19
- RFC PR: 
- URSA Issue: 

# Summary
[summary]: #summary

A python wrapper for the URSA code base. 

# Motivation
[motivation]: #motivation

Many projects in hyperledger and others use python and having a simple and easy to use wrapper
will save many developers time and effort as well as making the wrapper uniform. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


The new directory structure

```
wrappers
 |--python
     |-- python3.5
          |-- test
          |-- ursa
               | -- bls.py
     |-- python3.6
          |-- test
          |-- ursa
               | -- bls.py


```
The python wrapper is pretty basic and simple to use. The directory structure includes python3.5 and python 3.6 since many
community members have asked for 3.6 support. For now it will only include bls key data structures but it is expected that
others contribute more. The test for all functions and data structures and functions will be stored in test.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This will be correcting the mistakes made in indy-crypto and replace them in current indy-crypto dependent changes.

# Prior art
[prior-art]: #prior-art

See indy-crypto/wrappers

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities


