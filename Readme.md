---
eip: XXX
title: Minimal Avatar Smart Wallet (MASW)
description: A standardized smart‑wallet interface for EIP‑7702 account‑code delegation.
status: Draft
type: Standards Track
category: ERC
requires: 7702, 712
author: 0xMostafas <0xMostafas@proton.me>
discussions-to: https://ethereum-magicians.org/t/erc-tbd-minimal-avatar-smart-wallet-masw-delegate-wallet-for-eip-7702/24761?u=mostafas
created: 2025-07-08
---

## Abstract

Minimal Avatar Smart Wallet (MASW) is an immutable delegate‑wallet that any EOA can designate via `EIP‑7702` (txType `0x04`). Once designated, the wallet’s code remains active for every subsequent transaction until the owner sends a new `0x04` to clear or replace it. During each delegated call the EOA is the avatar and MASW’s code executes as the delegate at the same address, enabling atomic batched calls (EIP‑712‑signed) and optional sponsor gas reinbursment in ETH or ERC‑20.

The contract offers one primary function, `executeBatch`, plus two plug‑in hooks: a Policy Module for pre/post guards and a Recovery Module for alternate signature validation. Replay attacks are prevented by a global metaNonce, an expiry, and a chain‑bound `EIP‑712` domain separator. Standardising this seven‑parameter ABI removes wallet fragmentation while still allowing custom logic through modules.

## Motivation

A single‑transaction code‑injection model (EIP‑7702) grants EOAs full implementation freedom, but unconstrained diversity would impose high coordination costs:

- **Interoperability** – Divergent ABIs and fee‑settlement conventions force dApps and relayers to maintain per‑wallet adapters, increasing integration complexity and failure modes.
- **Economic alignment** – Gas‑sponsorship relies on deterministic fee‑reimbursement paths; heterogeneity erodes relayer incentives and throttles sponsored‑transaction volume.
- **Tooling precision** – Indexers, debuggers, and static‑analysis frameworks achieve optimal decoding and gas estimation when targeting a single, fixed byte‑code and seven‑field call schema.
- **Extensibility focus** – Constraining variability to two module boundaries (Policy, Recovery) localises complexity, allowing research and hardening efforts to concentrate on higher‑level security primitives rather than re‑engineering core wallet logic.

By standardising the immutable byte‑code, signature domain, and minimal ABI while exposing clearly defined extension hooks, MASW minimises fragmentation and maximises composability across the Ethereum tooling stack.

## Specification

### Overview of Delegation Flow

1. **Deploy** `MASW` with constructor argument `_owner = EOA`.
2. The owner sends an EIP‑7702 transaction (txType `0x04`) referencing the contract’s byte‑code hash.
3. After that transaction the EOA acts as the **avatar wallet** while the MASW logic executes as the **delegate wallet** at the _same_ address.

### Public Interface

```solidity
function executeBatch(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[]   calldata calldatas,
    address token,
    uint256 fee,
    uint256 expiry,
    bytes   calldata signature
) external;

function setPolicyModule(address newModule)   external;
function setRecoveryModule(address newModule) external;

event BatchExecuted(bytes32 indexed structHash);
event ModuleChanged(bytes32 indexed kind, address oldModule, address newModule);
```

### Transaction Type Hash

```solidity
bytes32 constant BATCH_TYPEHASH = keccak256(
  "Batch(address[] targets,uint256[] values,bytes[] calldatas,address token,uint256 fee,uint256 exp,uint256 metaNonce)"
);
```

### Storage Layout

| Slot | Name             | Type    | Description                              |
| ---: | ---------------- | ------- | ---------------------------------------- |
|    0 | `metaNonce`      | uint256 | Monotonically increasing meta‑nonce      |
|    1 | `_entered`       | uint256 | Re‑entrancy guard flag                   |
|    2 | `policyModule`   | address | Optional `IPolicyModule` (zero = none)   |
|    3 | `recoveryModule` | address | Optional `IRecoveryModule` (zero = none) |

`owner` and `DOMAIN_SEPARATOR` are `immutable` and occupy no storage slots.

### Domain Separator Construction

```solidity
DOMAIN_SEPARATOR = keccak256(
  abi.encode(
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
    keccak256("MASW"),
    keccak256("1"),
    block.chainid,   // MUST be the live chain‑ID; using 0 is disallowed
    _owner           // keeps separator stable before & after delegation
  )
);
```

### Batch Execution (`executeBatch`)

| Stage                 | Behaviour                                                                                                                                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Validation**        | ‑ `targets.length == values.length == calldatas.length > 0`<br>‑ `block.timestamp ≤ expiry`<br>‑ `metaNonce` matches then increments<br>‑ EIP‑712 digest recovers `owner` **or** is approved by `recoveryModule` |
| **Policy pre‑hook**   | If `policyModule != address(0)`, `preCheck` **must** return `true`; a revert or `false` vetoes the batch                                                                                                         |
| **Calls**             | For each index _i_: `targets[i].call{value:values[i]}(calldatas[i])`; revert on first failure                                                                                                                    |
| **Policy post‑hook**  | Same semantics as pre‑hook                                                                                                                                                                                       |
| **Fee reimbursement** | If `fee > 0`: native transfer (`token == address(0)`) or ERC‑20 `transfer` with OpenZeppelin‑style return‑value check                                                                                            |
| **Emit**              | `BatchExecuted(structHash)`                                                                                                                                                                                      |

#### Gas Sponsorship

The relayer and owner agree off‑chain on `(token, fee)` prior to submission.  
Because the fee is part of the signed batch, a relayer cannot unilaterally raise it.  
If a rival relayer broadcasts the same signed batch first, they earn the fee and the original relayer’s transaction reverts—aligning incentives naturally.  
Relayers **must** confirm the avatar’s balance up‑front; insufficient funds render the transaction invalid in the mem‑pool.

### Modules

#### Policy Module

```solidity
interface IPolicyModule {
  function preCheck (address sender, bytes calldata rawData) external view returns (bool);
  function postCheck(address sender, bytes calldata rawData) external view returns (bool);
}
```

- A module **may** veto by reverting _or_ by returning `false`.
- Aggregator designs are encouraged: forward to child policies and stop on first failure (revert or return `false`).

#### Recovery Module

```solidity
interface IRecoveryModule {
  function isValidSignature(bytes32 hash, bytes calldata sig) external view returns (bytes4);
}
```

Must return `0x1626ba7e`.

### Nonce‑Race Consideration

A single global `metaNonce` is used. Two relayers submitting the same nonce concurrently results in one success and one revert. The `expiry` field (wallets typically set ≤ 30 s) makes such races low‑impact, but UIs should surface the failure.

## Rationale

- **Immutable logic** minimises upgrade risk; a new version requires an explicit 7702 `0x04` call.
- A **two‑module** boundary captures common customisations without growing byte‑code.=
- No hard `maxTargets`; advanced users can bundle many calls, while conservative users install a size‑capping Policy module.
- Domain separator binds the real `chainId` to mitigate cross‑chain replays.

## Security Considerations

| Threat                      | Mitigation                                                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| Same‑chain replay           | Global `metaNonce`                                                                                |
| Cross‑chain replay          | Chain‑bound domain separator                                                                      |
| Fee grief / over‑charge     | Fee is part of signed data; front‑running risk sits with relayer                                  |
| Batch gas grief             | Optional Policy can reject oversized batches                                                      |
| ERC‑20 non‑standard returns | OpenZeppelin SafeERC20 transfer check                                                             |
| Re‑entrancy                 | `nonReentrant` guard; state mutated only before external calls (nonce++) and after (fee transfer) |
| Malicious Module            | Core logic immutable; swapping modules needs an owner‑signed tx                                   |

## Reference Implementation

Reference implementation can be found here [`MASW.sol`](https://github.com/MostafaS/MASW/blob/main/MASW.sol).

## Copyright

Copyright and related rights waived via [CC0](https://eips.ethereum.org/LICENSE).
