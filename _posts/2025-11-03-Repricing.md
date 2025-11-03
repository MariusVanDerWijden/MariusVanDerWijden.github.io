---
layout:        post
title:         Why we should prioritize repricings in Glamsterdam
date:          2025-11-03 17:00:00
summary:       This post explains my current view on the Ethereum roadmap, the proposed repricing EIPs and why we should prioritize them in Glamsterdam
categories:    blog
tags:          ethereum glamsterdam repricing EVM
---

Glamsterdam is in the process of being scoped at the moment. The two headliner features have been locked in: EPBS and BAL, now the real work begins in deciding on adding non-headliner features that will complement the headliners. Non-headliners should in my opinion be judged on three dimensions: **Impact**, **Readiness** and **Ease of Implementation**.

EIPs that excel in only of these categories are, in my opinion, not great candidates for non-headliner features. In this post, I want to argue, why the repricing EIPs are the most important upgrades we can make to Ethereum at this point in time.

# Impact
Repricings have a big impact on scalability. They let us redjust the cost of different resources. This way we can utilize the existing resources of the network better and we are not forced to increase the requirements for node operators when we increase the throughput of Ethereum. 

![Geth benchmarks](https://raw.githubusercontent.com/MariusVanDerWijden/mariusvanderwijden.github.io/master/_posts/pricing-benchmarks.png)

As you can see in the picture, the throughput of different operations varies widely. While some operations are executable at ~50 Megagas/s, others could be executed at more than 3 Gigagas/s. With the repricing we are trying to harmonize the different operations in order to safely increase the gas limit. 

# Readiness
This is the weakest point of the repricing proposals at the moment in my opinion. We have a very clear view of which operations and areas we want to reprice, but the numbers are not reliable yet. 

We started a significant effort at the Ethereum Foundation in collaboration with Nethermind in order to produce benchmarks that show which operations are over or underpriced and I am very confident that within the next few weeks, we will start to have better numbers and be able to finalize them by the beginning of next year. 

# Ease of Implementation
Most of the repricing EIPs are very easy to implement for client teams as they are only changing a few constants in the code. The bulk of the work is in proposing the numbers and verifying that all Execution Layer Clients are able to execute with the expected speed. 

I am anticipating that the repricing efforts will result in client devs finding a lot of improvements in their code during the implementation and benchmarking process, which will make clients even faster than today!

# Repricing EIPs
Now to the actual EIPs and my **personal** ranking of them.
The Glamsterdam [meta EIP](https://eips.ethereum.org/EIPS/eip-7773) currently contains 44 EIPs that are proposed of inclusion. 10+ of them are related to gas repricings. I would like to categorize them in three different categories: _crucial_, _important_ and _nice to have_.

### Crucial
The crucial category contains EIPs that I think are very crucial to the goal of repricings. Without them, the repricing efforts are not worth it in my opinion.

- [EIP-7904: General Repricing](https://eips.ethereum.org/EIPS/eip-7904) harmonizes costs across compute operations
- [EIP-8038: State-access gas cost update](https://eips.ethereum.org/EIPS/eip-8038) increase (and harmonize) the cost of operations that access state.
- [EIP-8037: State Creation Gas Cost Increase](https://eips.ethereum.org/EIPS/eip-8037)  increase (and harmonize) the cost of operations that create state.
- [EIP-2780: Reduce intrinsic transaction gas](https://eips.ethereum.org/EIPS/eip-2780) harmonize the cost of ETH transfer with the compute, data and state operations.
- [EIP-7981: Increase access list cost](https://eips.ethereum.org/EIPS/eip-7981) increase the cost of access lists.
- [EIP-7976: Increase Calldata Floor Cost](https://eips.ethereum.org/EIPS/eip-7976) increase the cost of call data.
- [EIP-7778: Block Gas Accounting without Refunds](https://eips.ethereum.org/EIPS/eip-7778) refunds are applied _after_ the block, thus the block gaslimit can not be cicumvented.  

These EIPs are trying to harmonize the costs between the three key resources Ethereum has: Compute, Data and State. They are trying to address the worst case blocks that attackers could create. Removing these bottlenecks will allow us to significantly increase the gas limit without the fear of attacks. 

### Important

- [EIP-8032: Size-Based Storage Gas Pricing](https://eips.ethereum.org/EIPS/eip-8032) decrease gas costs for storage in smaller contracts
- [EIP-8059: Gas Units Rebase for High-precision Metering](https://eips.ethereum.org/EIPS/eip-8059) anchors the gas cost to a different anchor than currently in order to increase precision.


### Nice-to-Have
- [EIP-8058: Contract Bytecode Deduplication Discount](https://eips.ethereum.org/EIPS/eip-8058) with significant increases in costs for creating state, we can give refunds to duplicated code, which makes up a significant share of contracts.
- [EIP-2926: Chunk-Based Code Merkleization](https://eips.ethereum.org/EIPS/eip-2926) Introduces code-chunking which allows for initcode and code size increases.
-[EIP-8011: Multidimensional Gas Metering](https://eips.ethereum.org/EIPS/eip-8011) lets us meter different resource independently which allows for better control of the key resources.

### Argue against
I would argue against the following EIPs:

-[EIP-7971: Hard Limits for Transient Storage](https://eips.ethereum.org/EIPS/eip-7971) this EIP is incompatible with AA unfortunately.
- [EIP-8053: Milli-gas for High-precision Gas Metering](https://eips.ethereum.org/EIPS/eip-8053) its an alternative to EIP-8059 and I think introducing decimals in the EVM will make the protocol unnecessary complex.
- [EIP-8057: Inter-Block Temporal Locality Gas Discounts](https://eips.ethereum.org/EIPS/eip-8057) this would enshrine the caching behavior of clients in protocol.
- [EIP-7923: Linear, Page-Based Memory Costing](https://eips.ethereum.org/EIPS/eip-7923) unnecessarily complicates the memory model and leads to significant redesigns of the EVM implmentations.

# Conclusion
In conclusion, I think repricings are the most important feature to include in Glamsterdam as it will allow us to significantly improve the throughput of Ethereum. The EIPs are very easy to implement and while the numbers are not ready yet, we have invested a lot in the testing and benchmarking infrastructure to get them quickly and reliably.

So contact your local core developer and ask them to support repricings for Glamsterdam! :P