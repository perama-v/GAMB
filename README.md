---
eip: <to be assigned>
title: Generic Attributable Manifest Broadcaster
description: A contract for announcing newly published metadata.
author: perama (@perama-v)
discussions-to: <URL>
status: Draft
type: Standards Track
category (*only required for Standards Track): ERC
created: <date created on, in ISO 8601 (yyyy-mm-dd) format>
requires (*optional): <EIP number(s)>
---

> Note: This is formulated in the style of an EIP as a thought exercise for now.
It is not clear that the concept is suited for an EIP. If that changes, I'll update this
comment.

## Abstract

A smart contract for publishing a content identifier (CID) for a manifest file for a under a topic
name. The contract provides a method to retrieve a content identifier for a given publisher
and topic.

A contract is compliant if it:
- Accepts a topic-CID pair in a transaction call.
- Returns a CID for a given topic-publisher pair as a read method.

## Motivation

The Ethereum ecosystem involves data structures that are periodically updated and distributed
between peers. Users who follow a protocol to update the data according to some protocol
need to broadcast this to interested parties. Storage of the CID in a smart contract under a topic
allows parties to detect updates by different publishers and retrieve data without relying
on a single provider.

## Specification
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
described in RFC 2119 and RFC 8174.

### Smart contract

The contract MUST provide a write method:
```
publishHash(string TOPIC, string HASH)
```
Where:
- `TOPIC` string identifier unique to a data type, such as a partcicular database or index (such as
"my-special-index").
- `HASH` a string representing the Interplanetary File System (IPFS) CID of a manifest file.
Any CID version MAY be used.

The `publishHash` method MUST emit an event with the following signature:

```
event HashPublished(address PUBLISHER, string TOPIC, string HASH)
```

Where:
- `PUBLISHER` is `msg.sender` in the call to `publishHash`.

The contract MUST provide a read method that returns `HASH`.

```
manifestHashMap(address PUBLISHER, string TOPIC)
```

The contract MUST NOT have the ability to modify or remove any methods or through any upgrade,
self-destruct or proxy pattern that would alter the aforementioned methods.

### Manifest file

The manifest is a JavaScript-Object Notation (JSON) file with CID equal to `HASH`.

The Manifest MUST have a "version" field, whose value is RECOMMENDED to represent the semantic
version of the data structure.

The Manifest MUST have a "schemas" field, whose value is RECOMMENDED to either represent the CID
of the schema
and/or protocols that define the data and its preparation and use.

It is RECOMMENDED that the Manifest has a "publish_as_topic" field, whose value is `TOPIC`.

It is RECOMMENDED that the manifest contains a list of CIDs, each representing the smallest
distributable unit of the data.

Other fields MAY be used.

The layout of the Manifest MUST be documented in the aforementioned linked schema.

## Rationale

In existing systems where data is published with content identifiers (CIDs)
such as the Interplanetary File
System (IPFS), the publisher may update records with a naming system such as then InterPlanetary
Name System (IPNS) or the Ethereum Name System (ENS). This becomes brittle if the main
provider ceases their operation.

The manifest broadcaster is a solution that allows
different publishers to advertise databases in the same place. Competing or complementing
broadcasters provide user choice. There is low operational overhead for becoming a new publisher,
including one-off or infrequent publishing.

## Reference Implementation

The following compliant contract is deployed at `0x0c316b7042b419d07d343f2f4f5bd54ff731183d`.

The contract was deployed by TrueBlocks to publish the manifest
of the UnchainedIndex as a public good. It may be used by other parties to publish
data under topics of their choosing.
```
/**
 *Submitted for verification at Etherscan.io on 2022-06-13
*/

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract UnchainedIndex_V2 {
    constructor() {
        owner = msg.sender;
        emit OwnerChanged(address(0), owner);

        manifestHashMap[msg.sender][
            "mainnet"
        ] = "QmP4i6ihnVrj8Tx7cTFw4aY6ungpaPYxDJEZ7Vg1RSNSdm"; // empty file
        emit HashPublished(
            msg.sender,
            "mainnet",
            manifestHashMap[msg.sender]["mainnet"]
        );
    }

    // Note: this is purposefully permissionless. Anyone may publish a hash
    // and anyone my query that hash by a given publisher. This is by design.
    // End users themselves must determine who to believe. We suggest it's us,
    // but who's to say?
    function publishHash(string memory chain, string memory hash) public {
        manifestHashMap[msg.sender][chain] = hash;
        emit HashPublished(msg.sender, chain, hash);
    }

    // If, at a certain point, we decide to disable or redirect donations. Otherwise,
    // owner no other purpose. "isOwner isAMistake!"
    function changeOwner(address newOwner) public returns (address oldOwner) {
        require(msg.sender == owner, "msg.sender must be owner");
        oldOwner = owner;
        owner = newOwner;
        emit OwnerChanged(oldOwner, newOwner);
        return oldOwner;
    }

    function donate() public payable {
        require(owner != address(0), "owner is not set");
        emit DonationSent(owner, msg.value, block.timestamp);
        payable(owner).transfer(address(this).balance);
    }

    event HashPublished(address publisher, string chain, string hash);
    event OwnerChanged(address oldOwner, address newOwner);
    event DonationSent(address from, uint256 amount, uint256 ts);

    mapping(address => mapping(string => string)) public manifestHashMap;
    address public owner;
}
```
As a demonstration the above contract may be used to fetch the manifest of the
UnchainedIndex for mainnet by using the read method `manifestHashMap(PUBLISHER, TOPIC)` with the following values:

- `PUBLISHER`: `0xf503017d7baf7fbc0fff7492b751025c6a78179b`
- `TOPIC`: `mainnet`

```
> QmPshzTJCHdskAH6XWAbZgd5ebfwghfens1BHn8nYVn6Xh
```


## Security Considerations

The contract is non-upgradeable to prevent the modification of data after broadcasting.

Independent parties may use the same topic to publish different data. It is the responsibility
of the user to find and provide a publisher. For example, the topic "mainnet" may be used
for both the UnchainedIndex and a different index without disrupting each other.

Data linked to by a publisher by the manifest may be incorrect, out of date or inaccurate.
The verion and schema sections of the data provide a means for users to navigate breaking changes
to the structure and content of the data.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).