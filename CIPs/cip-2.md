---
cip: <to be assigned>
title: Generic Conditions
description: Enable generic conditions to fulfill a transfer.
author: Layne Haber (@LayneHaber) <layne@connext.network>, Arjun Bhuptani (@ArjunBhuptani) <arjun@connext.network>, Rahul Sethuram (@rhlsthrm) <rahul@connext.network>
status: Draft
type: Standards Track
category (*only required for Standards Track): Core
created: 2021-12-16
requires (*optional): N/A
---

## Abstract
This construction would allow contracts to evaluate whether a transaction should be cancelled or fulfilled. The current contracts restrict the fulfillment and cancellation conditions to verifying a user signature.

## Motivation
Delegating the logic of fulfilling or cancelling transactions to external contracts rather than exclusivley user signatures will allow for easier native protocol integrations to nxtp. Without a way to safely delegate the ability to mark a transaction as "fulfilled" to an arbitrary contract, many protocol integrations would require a Connext-specific adapter rather than native integration.

## Specification
The `ProposedOwnable.sol` contract MUST:
- be responsible for maintaining a whitelist of conditions for that chain
- allow for ownership over the whitelist to be transferred or renounced

The changes to the interface MUST include:
```ts
abstract contract ProposedOwnable {
    ...

    bool private _conditionOwnershipRenounced;
    uint256 private _conditionOwnershipTimestamp;

    event ConditionOwnershipRenunciationProposed(uint256 timestamp);
    event ConditionOwnershipRenounced(bool renounced);
    
    /**
    * @notice Returns the timestamp when validation ownership was last proposed to be renounced
    */
    function conditionOwnershipTimestamp() public view virtual returns (uint256) {
        return _conditionOwnershipTimestamp;
    }

    /** 
    * @notice Indicates if the ownership of the validation whitelist has
    * been renounced
    */
    function isConditionOwnershipRenounced() public view returns (bool) { ... }

    /** 
    * @notice Indicates if the ownership of the validation whitelist has
    * been renounced
    */
    function proposeConditionOwnershipRenunciation() public virtual onlyOwner { ... }

    /** 
    * @notice Indicates if the ownership of the validation whitelist has
    * been renounced
    */
    function renounceConditionOwnership() public virtual onlyOwner { ... }
}
```

The `InvariantTransactionData` struct MUST:
- include arbitrary information which may be relevant to a condition's resolution
- include information about the sending and receiving chain conditions

The updated struct should be:
```ts
// Holds all data that is constant between sending and
// receiving chains. The hash of this is what gets signed
// to ensure the signature can be used on both chains.
struct InvariantTransactionData {
    address receivingChainCondition;
    address sendingChainCondition;
    bytes encodedConditionData;
    ...
}
```

The `TransactionManager` MUST:
- include `onlyOwner` methods to add and remove generic conditions
- include sanity checks around conditions when `prepare`-ing a transaction
- check if a transaction can be fulfilled using the `shouldFulfill` function on the specified condition interpreter
- check if a transaction can be cancelled pre-expiry using the `shouldCancel` function on the specified condition interpreter
- allow `unlockData` to be revealed on `fulfill` or `cancel` to satisfy a given condition

All `ConditionInterpreters` MUST satisfy the following interface:
```ts
interface IConditionInterpreter {
    function shouldFulfill(
        TransactionData calldata txData,
        bytes calldata unlockData,
        uint256 transactionManagerChainId
    ) external view returns (bool);

    function shouldCancel(
        TransactionData calldata txData,
        bytes calldata unlockData,
        uint256 transactionManagerChainId
    ) external view returns (bool);
}
```

## Rationale
The generic `ConditionInterpreters` are conceptually similar to the `TransferDefinitions` used in the [vector](https://github.com/connext/vector) protocol.

The `ConditionInterpreter` and changes to the `TransactionData` struct are designed to allow for more generalized unlocking conditions. Imagine the following scenario -- a DeFi protocol wants to allow a crosschain transaction to be fulfilled *only after* some other condition is met on their contracts. In the current construction, you would have two options:

1. Users monitor this condition themselves. This is undesirable as it places the assumption of an "honest user" onto the protocol, which is rarely a safe assumption to make.
2. Create a specific an adapter that calls some `isFulfillable` function on the protocol contracts before proceeding to unlock the transaction on the `TransactionManager`. This approach has a few drawbacks. First, it requires the protocol implementers to come up with a safe adapter to evaluate the condition. Additionally it requires changes to the frontend code to both (1) collect the user signature and (2) conceal the user signature until the `isFulfillable` function is met.

## Backwards Compatibility

This change should **NOT** be considered backwards compatible as it changes the contracts, the `TransactionData` struct, and the `fulfill` / `cancel` interfaces. However, the user-experience of completing a crosschain transaction would be more or less consistent.

These introductions require the following to be updated:
1. *Subgraph mappings* -- changes in the `TransactionData` struct will change the event signatures, and subgraph mappings will have to be updated.
2. *Messaging payloads* -- changes in the `TransactionData` will require changes in the messaging payloads, specifically for the auction and meta transaction requests.
3. *SDK interface* -- changes in the `TransactionData` will require changes in the sdk interface. It could be possible, though perhaps not advantageous in the long term, to keep these interfaces consistent.

As this is not a backwards compatible chain routers will have to activate cleanup mode to remove any active transactions before updating to watch the new contracts. Once updated, they should only respond to bids from users who are running compatible versions.

## Test Cases
N/A.

## Reference Implementation
You can see [here](https://github.com/connext/nxtp/pull/433) for a draft PR of what these contract changes could look like. 

*NOTE*: Not a complete implementaiton!

## Security Considerations
There are three primary security considerations to be discussed with this type of implementation:

***Selecting Condition Interpreters***
As proposed, there is a `sendingConditionInterpreter` and a `receivingConditionInterpreter`. This opens up an attack vector where users could specify a `sendingConditionInterpreter` that is logically different than the `receivingConditionInterpreter`. If the router prepares this transfer, they may not be able to use the same `unlockData` to fulfill the transfer on the `sendingChain` as the user was able to fulfill with on the `receivingChain`. This means upon transaction expiry, the user would be able to claim their deposited funds from the sending chain in addition to successfully claiming the routers funds on the `receivingChain`.

There are two strategies to mitigate this concern:
1. *ConditionRegistry*: A `ConditionRegistry` would operate in a very similar manner as the vector [`TransferRegistry`](https://github.com/connext/vector/blob/main/modules/contracts/src.sol/TransferRegistry.sol). The primary advantage is that it would allow you to choose a condition interpreter by *name* rather than by *address*. Logically consistent interpreters across multiple chains would be given the same `name` and the `sendingConditionInterpreter`/`receivingConditionInterpreter` fields could be replaced with a single `name`. Implementing this solution would add complexity to the condition registration process, and increase the gas. However, the use of a single field for both chains would eliminate this type of attack as long as the interpreter was correctly added to the registry.
2. *Offchain Checks*: Routers are ultimately motivated to check the validity of interpreters offchain before preparing a transaction. This would save gas and keep the adding/removing of interpreters simple, but relies on offchain rather than onchain logic.

***Condition Interpreter Limitations***
Condition interpreters have to follow some generic rules so they are safe-to-use by the protocol. Condition interpreters should abide by the following:
- *No time-related conditions*: The sending and receiving chain have to be able to unlock with the same results using the same inputs, at different times. Anything that adds a time-related element (i.e. funds disbursed dependent on the time elapsed) will not have symmetric results on the sending and receiving chains.
- *msg.sender checks*: Because the `msg.sender` will be different on both the sending and receiving chains, there should be no dependencies on the sender for fulfilling successfully.
- *pure or view*: The interpreters should *not* be stateful, or have second order effects, because they could have different states between chains. Instead, all functions should be `pure` or `view` functions

Due to the sensitive nature of interpreters, they should be managed by a whitelist. This concern is not much different than the concerns around adding arbitrary tokens to the protocol, so should be an acceptable concession.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Questions
- Registry vs. offchain checks?
- Return `bool` or a `Balance` type object