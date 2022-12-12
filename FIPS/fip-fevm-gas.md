---
fip: "<to be assigned>" <!--keep the qoutes around the fip number, i.e: `fip: "0001"`-->
title: Update gas charging schedule for FEVM
author: Steven Allen (@stebalien)
discussions-to: <URL>
status: Draft
type: <Technical (Core, Networking, Interface, Informational)  | Organizational | Recovery | FRC>
category (*only required for Standard Track): <Core | Networking | Interface >
created: <date created on, in ISO 8601 (yyyy-mm-dd) format>
spec-sections:
  - <section-id>
  - <section-id>
requires: 0032, PR-512 (draft)
---

<!-- You can leave these HTML comments in your merged FIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new FIPs. Note that a FIP number will be assigned by an editor. When opening a pull request to submit your FIP, please use an abbreviated title in the filename, `fip-draft_title_abbrev.md`. The title should be 44 characters or less. -->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->

With the introduction of user programmable smart contracts, we need to ensure that the gas model accurately reflects the cost of on-chain computation. In the current network, users can only trigger course-grained behaviors by submitting messages to built-in actors. However, with the introduction of FEVM, users will be able to deploy new smart contracts with fine-grained control over FVM execution. If we don't carefully tune the gas model to accurately charge for computation, an attacker could carefully construct a contract to take advantage that and slow down or even stop chain validation.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->

This FIP proposes a few adjustments to the gas charging schedule, adjusting both the costs of certain system calls and some WebAssembly instructions. Specifically, this FIP:

1. Adds charges to syscalls that were previously "free".
2. Adds variable charges (scaling with the input) to some syscalls that perviously charged a fixed fee.
3. Adds memory latency charges to all operations that can potentially read from random memory locations.
4. Adds bulk memory copy fee to all instructions that initialize and/or copy memory.

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

With the introduction of FEVM, users will be able to directly trigger:

1. Operations that have runtime cost varying in the size of the inputs (bulk memory copies, hashing, etc.).
2. Random memory reads.

Currently:

- There is a fixed fee but no per-byte charge for hashing, memory copies, fills, etc.
- Memory reads are priced like all other instructions (4 gas per instruction) but _random_ memory reads may take up to ~10ms (100gas) if the value being read is not in the CPU's cache.

It's not currently possible to take advantage of these limitations _today_ (before FEVM) however...

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

Notes:

- Filecoin targets 10 gas/ns. That is, an operation that takes 1 nanosecond should cost 10 gas.
- The FVM supports gas charges with _milligas_ precision. That is, we can charge for gas in increments of 0.001 gas.

### System Calls

TODO:

- Some of this will be covered in the FEVM FIP itself.
- Update block charges as well?

#### Hashing

Before this FIP, hashing cost a flat 31355 gas. This FIP proposes the following gas fee schedule:

| Hash Function | Flat | Per-Byte |
|---------------|------|----------|
| SHA2-256      | TODO | TODO     |
| Blake2b-256   | TODO | TODO     |
| Blake2b-512   | TODO | TODO     |
| Keccak256     | TODO | TODO     |
| Ripemd160     | TODO | TODO     |


### Instructions

This FIP introduces two new instruction-level charges:

1. A memory access fee of 100 gas.
2. A memory copy fee of 0.5 gas/byte.

The memory copy cost was first introduced in FIP0032 but only applied to memory copying within the FVM itself.

It also makes a few no-op operations free.

TODO: Re-run gas benchmarks in light of this and consider reducing the per instruction cost (previously averaged in memory operations, etc.

#### Memory Access

A memory access fee of 100 gas is charged _once_ for every instruction that reads a user-specified memory location. Instructions that read onto the stack and perform no other operations are _only_ charged the memory access fee while instructions that perform an additional operation are charged the default instruction fee (currently 4 gas) in addition to the memory access fee.

Specifically, the following instructions are charged a memory access fee (100 gas) only:

1. Instructions that read from tables: `br_table`, `table.get`, `call_indirect`.
2. Instructions that load from memory to the stack: `$t.load`, `$t.loadN_u` (non sign extending).

The following instructions are charged both a memory access fee and the default instruction fee (100+4 gas):

1. Instructions that copy tables and memory: `{table,memory}.{init,copy}`.
2. Instructions that load from memory then operate on the value: `$t.loadN_s` (sign extension).

#### Memory Copy

In addition to any memory _access_ fees, instructions that operate on variable-sized memory chunks are charged a per-byte memory copy fee: `{table,memory}.{init,fill,copy,grow}`.

Additionally, these fees are charged for memory and table allocation when the actor is instantiated (started).

#### No-Ops

Costs have been removed from some operations that would have little to no cost on a traditional CPU. These rules are _valid_ under FEVM because the user cannot trigger _specific_ sequences of WebAssembly instructions but must be revisited before introducing full user programmability as some pathologically unoptimized programs may be able to find a way to make these instructions actually take time.

##### Locals & Globals

Setting/getting locals and globals is now free because these instructions are mostly just telling other instructions which registers to use. Furthermore, in FEVM, the user cannot directly control and/or create new locals/globals.

##### Casts

Logic-less casts have been made free as they generally don't turn into actual instructions (or, when they do, these instructions basically free and tied to some other more expensive operation).

1. `i64.extend32_u` - re-interprets the input as a u64 (without sign extension).
2. `i32.wrap_i64`- casts an i64 to an i32.
3. `$t.reinterpret_$t` - bitwise casts.

##### Constants

All operations that push constants onto the WebAssembly stack (`$t.const`) have been made free. In most cases, these will compile down to "immediate" arguments, or, in the worst case, _very_ predictable loads from read-only memory.

## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.

### Memory Gas Derivation

The memory access cost is derived from an expected worst-case memory latency of 10ns:

```
10ns * 10gas/ns
= 100 gas
```


1. Re-interpreting stack operations that change the type of a stack element but don't actually operate on it.
2. "Constant" operations.
2. Operations to load and store globals and locals.

The memory copy cost was derived from benchmarking and the data transfer rate of 3200mhz RAM:

```
3200Mhz * 8B
= 25.6GB/s
= 25.6B/ns

25.6B/ns / 10gas/ns
= 2.56B/gas
= 0.39gas/B
```

The value was then rounded up to 0.5 to account for variability in the execution environment.

## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This FIP is consensus breaking, but should have no other impact on backwards compatibility.

## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

TODO: Will be covered by conformance tests (the only reasonable way to cover gas charging, unfortunately).

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

This FIP charges zero for some operations. These operations should consume no CPU time, but this should be rigorously verified before allowing users to deploy arbitrary WebAssembly contracts.

Furthermore, this FIP does not charge variable amounts for different types of mathematical operations. For example, in the future, we'll likely want to charge more for floating point operations, and significantly more for the "sqrt" operation.

However, the proposed gas schedule should be "good enough" for FEVM because:

1. Users cannot deploy new WebAssembly contracts, so there's no way to write custom WebAssembly that creates sequences of "free" instructions that actually have runtime costs.
2. Floating point instructions, including the square-root instruction, is not available in the EVM.

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->

TODO

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->

This FIP will likely increase gas fees in some cases and needs to be carefully benchmarked.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

TODO

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
