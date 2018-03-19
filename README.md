# Distributing authenticated mappings

This repository contains some foundational work for generalizing and distributing authenticated mappings with user-defined semantics. 

## What does that mean?

The internet is built off of mappings; in particular, DNS manages IP mappings onto human-meaningful aliases.

The problem that most internet infrastructure faces (largely DNS) is that many of these mappings are unauthenticated. DNSSEC attempts to solve this issue by providing signed records, but has yet to see widespread adoption. The point solution we have instead adopted is the current WebPKI ecosystem, with Certificate Authorities running large identity-to-public-key mappings and Certificate Transparency logs auditing their behavior.

This CA + CT infrastructure solution for mapping domain names to public keys took many years to get right, because PKI is hard and secure systems break backwards compatibility. It works well enough that people want to distribute *other types* of mappings that are built on top of this. MTA-STS policies bootstrap off of HTTPS to distribute policies for a particular domain. Mozilla's Binary Transparency project bootstraps CT to distribute binary hashes.

Another point: we may want to distribute name-to-public-key mappings for things other than TLS; for instance, PGP keys for e-mail addresses or public keys for Signal numbers.

Lastly, transparency logs are great but not perfect. It relies on clients to continually audit these authorities-- otherwise, others trying to connect to those clients can still be MitM'ed by a network-level adversary who has also co-erced the authority to subvert existing mappings. In addition, although they allow us to catch when a central authority is misbehaving, it is uncertain what we do if this happens. Do we migrate these certificates to a different authority and distrust the original? How do we notify users?

## Ok, tell me about this project

Our aim is to create a distributed database of mappings by
1. performing distributed consensus over
2. signed statements according to explicitly defined semantics.

For (1), any byzantine-fault tolerant distributed consensus will work. For starters, we can assume a fixed-membership cluster that runs PBFT, similar to a set of CAs performing BFT consensus over CSR submissions. You could also run this on a large, decentralized, distributed ledger, but this comes with the costs of such a system. We think SCP (federated byzantine agreement) might make sense for such a system, because it could sensibly bootstrap off of existing trust relationships.

For (2), how do we define these semantics?

### Defining mappings

Suppose I want the network to perform consensus over my binary hashes! I would submit the following transaction to the network:

```
CreateMapping {
    create_transition_rule: ["OpOwnerSignature"],
    update_transition_rule: ["OpOwnerSignature"],
    key_type: "PublicKey",
    value_type: "SHA256Hash",
}
```

This defines the key => value types, as well as which transition rules need to be fulfilled for this mapping to succeed. The network creates this mapping and returns an Id (a uniquely identifiable string), using which you can identify mappings.

### Submitting entries and enforcing transitions

With the "Id" of a particular mapping type (for instance, binary hashes), we can submit entries to it. Suppose the Id for binary hashes is "abcd", and Mozilla wants to sign off on a new release of Firefox. They might submit:

```
UpdateRequest {
    mapping_id: "abcd",
    key: (<ref/URL to hashed contents>, <name>),
    value: <firefox hash>,
    signature: <this request, signed, by public key for name>
}
```

The network would enforce the transitions specified for this mapping type-- namely, `OpStatement`, which validates that the signature was indeed valid for the public key specified in the request.

### Other operation types

These operations help define whether a Create or Update transition is valid for a particular mapping type.

```
    OpCASignature:         
        Enforces that the creation is signed, by a public key with a valid
        chain to a root CA (from ca-certificates package).
    OpOwnerSignature:
        Enforces that update signature is signed by the owner of the associated name
        in this database's key store.
    OpNOfMSignatures:   
        Enforces that the update signatures contain N valid signatures from
        the M identities specified by the transaction.
```
[ops.proto](ops.proto) contains the documentation for the above.

### Network RPCs

```
CreateMapping
Update
Create
Get
```
[mapping.proto](mapping.proto) describes these in more detail.

### Other

For more examples, see [examples_requests.proto](examples_requests.proto). For the other mapping specs, see [mapping.proto](mapping.proto).

### Future work

Most interface updates will be pushed to this repository. In order to stabilize the interface & network RPCs, we're working on implementing the validation as 1) a state machine over a PBFT cluster and 2) an ERC20 contract as a proof-of-concept. Some preliminary work on (1) can be found on our implementation of PBFT [here](https://github.com/sydneyli/keynet/tree/master/src/pbft).

