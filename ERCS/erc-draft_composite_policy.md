---
title: Composite On-Chain Policies with Arbitrary Rules
description: Simple onchain policy engine utilizing generic rule artifacts within DAG evaluator
author: Vlad Pryadko <vpriadko@lacero.io>, Vitali Grabovski <v.grabovski@lacero.io>
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2025-03-14
requires: 1167 ??
---

## Abstract

An onchain policy engine proposal with simple yet efficient approach. Policies are split into simple rules - artifacts - and then represented onchain as a DAG. The DAG is then recursively evaluated starting from a top-tier artifact (root node, effectively making it a tree). The evaluation result can be treated as a final policy evaluation result.
This proposal describes approaches, standard interfaces and conventional traits, facilitating policies and artifacts interoperability for common usage.

## Motivation

There are a large number of software systems that rely on smart contracts, from simple vaults to decentralized exchanges and oracles. All of these systems should be regulated in one way or another - limiting the right of an administrator to use them, restricting the methods allowed, limiting the amount of withdrawals. Restrictions, rules, and prohibitions are all essential parts of any financial (and other) system.
Current approaches to smart contract programming allow creating simple rules in the form of algorithmic constraints and modifiers. But more complex rules - dynamic, composite, different depending on different conditions - become more difficult to create as the number of inputs to these rules grows. Moreover, some problems - for example, interactive composition of simple rules - cannot be solved with current methods due to the lack of Reflection. Reusing complex rules, changing them on the fly, and grouping them is also out of the question.
The proposed approach aims to solve this problem without changing any of the network layers.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Definitions

**Policy** - a rule or set of rules combined in such a way as to be perceived as one coherent rule. Policy is a term that is clearly defined only in connection with the action to which it applies. For example, a particular bank transfer must always comply with the bank's policy for that type of transfer. This particular policy may be a single rule or a composite of several. Moreover, a policy may be a composite of several other policies, which in this case are perceived as rules.
So, to summarize, a policy is a rule for a specific transaction, no matter how complex or simple.

In this context, the elementary particle that makes up a policy is called an **artifact**. An artifact - is also a rule, but it is indivisible within the policy. Inside, an artifact can be as complex or as simple as its creator wants it to be.

For example, the policy “must be over 21 years old and a citizen” consists of the artifacts “must be over 21 years old” and “must be a citizen”. Obviously, it is possible to define the artifact “to be over 21 and to be a citizen” if desired, but it is recommended to choose a more natural granularity.
Moreover, artifacts in the context of the current proposal are arbitrary contracts, so they can be not only rules, but also arbitrary transactions. It follows that “should be” and “and” are also artifacts.
From all of the above, it turns out that politics is a constructor consisting of simple blocks - artifacts.

The technical details of such a constructor - one that allows you to change blocks and dynamically supply them with data - are defined by this standard.

### Interfaces

Artifacts can implement any logic, but they must adhere to a standard interface that will allow the policy controller to integrate them correctly and ensure an unmodified data flow between them:

```solidity
interface IArbitraryDataArtifact {
    function exec(bytes[] memory data) external returns (bytes memory);

    function init(bytes memory data) external;

    function getExecDescriptor()
        external
        pure
        returns (string[] memory argsNames, string[] memory argsTypes, string memory returnType);

    function getInitDescriptor()
        external
        pure
        returns (string[] memory argsNames, string[] memory argsTypes);

    function description() external pure returns (string memory desc);

  }
```

Other interfaces are more implicit, being conventions and approaches reasoned and explained in Rationale.

## Rationale

Since rules often require heterogeneous data (both existing on the blockchain and purely off-chain), the system must be well suited for integration from both off-chain and on-chain sides. This integrability is ensured by the architecture of the artifact handler, which is able to properly arrange artifacts and data, resulting in a correctly computed policy. In fact, an instance of the handler is an instance of the policy.

### All bytes

Since artifacts are contracts, their method arguments must be clearly typed. On the other hand, the handler cannot handle all types in a uniform manner. To avoid massive ad hoc duplications, it is suggested that all types be encoded into bytes before being supplied to and returned from artifacts.
This means that each variable or result of an artifact can be perceived as bytes by both the handler and the likely offline code, which makes life much easier

### Artifact dataflow traits

Artifacts have two main methods - init and exec.
The init method is called once at the stage of policy initialization by the handler. Since artifacts can be reused across multiple policies, each new policy makes its own copy of each artifact that needs init using erc1167. This ensures that the artifact has a clean state.
Both init and exec methods take arguments (exec also has a return value). According to the all bytes approach, these arguments are encoded as bytes. But they are serialized differently: exec arguments are an array of byte-encoded values, while init arguments are byte-encoded values.
exec args = [abi.encode(uint256), abi.encode(string)]
init args = abi.encode(uint256, string)

These values should be decoded accordingly:

```solidity
function exec(bytes[] memory data) external pure override returns (bytes memory) {
        uint256 argA = abi.decode(data[0], (uint256));
        string memory argB = abi.decode(data[1], (string));

        return abi.encode(doSomething(argA, argB));
    } 
```

```solidity
function init(bytes memory data) external override {
        (bool init1, address init2, bytes memory init3, uint256 init4, string memory init5) = abi
            .decode(data, (bool, address, bytes, uint256, string));
}
```

### External complience traits

Artifacts should speak for themselves.
The getExecDescriptor method should return an array of signatures of the exec method parameters and a description of the return value. The getInitDescriptor method should return an array of signatures of the init method parameters.

```solidity
function getExecDescriptor()
        public
        pure
        override
        returns (string[] memory argsNames, string[] memory argsTypes, string memory returnType)
    {
        uint256 argsLength = 2;
        argsNames = new string[](argsLength);
        argsNames[0] = "argA";
        argsNames[1] = "argB";
        argsTypes = new string[](argsLength);
        argsTypes[0] = "uint256";
        argsTypes[1] = "uint256";
        returnType = "bool";
    }

```

The description method returns a constant string that describes the logic of the artifact - what it does, whether it is statefull, where it can be used, and so on. The description format is not strictly defined and is intended for human perception.
The getExecDescriptor and getInitDescriptor methods provide information for automated systems that automatically encode arguments for further policy computation. An example of such a system can be found in the reference implementation.

```solidity
function description() external pure override returns (string memory desc) {
        desc = "Stateful artifact used to validate signatures from a predefined list of approvers. Requires a quorum of valid signatures to approve. First parameter - messageHash packed as bytes, second one - signatures packed as bytes array. Returns bool representing whether enough valid signatures were provided.";
    }
```


## Backwards Compatibility

No backward compatibility issues found.
One may associate the current approach with erc-2746, but they are not intended to be a replacement for each other or related standards, so backward compatibility is not provided.
Even though erc-2746 describes a similar concept of rule perception, the current standard declares a completely different approach to implementing a rule engine on the on-chain, and interfaces have no similarities.

## Reference Implementation

See https://github.com/GuardianLabs/policy-sdk/tree/dev/packages/contracts/contracts (move to assets?)

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.
  If the reference implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed.

  TODO: Remove this comment before submitting
-->

## Security Considerations

Since the standard sets out a scheme (protocol) for the interaction of contracts with the main orchestrator (processor), the standard itself is not a source of security problems. Each implementation of this policy engine has to justify the security of its use on its own. This section lists only hypothetical common security issues that may be common to many implementations:
1. Untrusted artifact. If any elementary rule that is part of a policy is unreliable, the entire policy is unreliable and can be exploited to obtain a false authorization for an operation. An unreliable artifact can be an artifact with updated logic (proxy), an artifact with unknown code, an artifact that is vulnerable to attacks, etc.
2. Incorrect initialization of the state. Statefull artifacts use a new proxy to get a clean state. This approach may not be obvious and, if you get confused, you can get empty slots where you expected data and vice versa. The reference instantiation uses a minimal proxy (any approach can be used to generate a pseudo-copy) that uses delegatecall, which is also worth mentioning.
3. Attack on the handler. If the implementation of the policy graph handler is vulnerable, any policy can be replaced by a single artifact policy that always returns true. Thus, special attention should be paid to the authorization of handler actions.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
