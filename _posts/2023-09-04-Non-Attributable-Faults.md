---
layout:        post
title:         Solving Non-Attributable Faults or how to fix MEV
date:          2023-07-16 10:00:00
summary:       In this post we discuss Non-Attributable faults in data exchange and a protocol that can fix them. As well as a protocol for fixing the non-attributable fault inherent in the current MEV builder architecture.
categories:    blog
tags:          ethereum cryptography non-attributable fault data exchange
---

_This post was inspired by the paper "FairSwap: How to fairly exchange digital goods" by Dziembowski et. al. (1)._

Many blockchain protocols suffer from an inherent non-attributable fault. Meaning a fault where the blame can not be attributed correctly. 

For example if I want to buy a piece of data from a merchant, everyone else (outside of the transacting parties) can not distinguish between two failure cases: 
- The merchant did not send the data
- I received the data but claimed that I didn't

These issues can be solved by a mutually trusted intermediary that acts as an escrow service in order to facilitate the exchange.

We can also construct a protocol that solves these faults in specific cases using smart contracts.

## Example construction

Suppose we have two players `A` and `B`.
`A` has some data `data` that they want to sell to `B`.
`B` only knows the `hash = h(data)` of the data (e.g. because the hash was notarized by a trusted authority). In this example we proof the claim that the data hashes to `hash`. We will see later that other claims about the data can be verified as well.

1. In the first step `A` encrypts `data` with key `key`:
	`encData <- ENC(data, key)`
	and computes the hash of the encrypted data:
	`hashEnc = h(encData)`
	and the hash of the key:
	`hashKey = h(key)`
	Now `A` sends `{hash, hashEnc, hashKey}` to a smart contract and the encrypted data `encData` to `B`.

2. `B` verifies that `hashEnc` matches the encrypted data they received.
	`hashEnc' = h(encData'); if hashEnc' != hashEnc : abort`
	`B` sends a conditional payment to the smart contract with the following conditions:

		If `A` fails to send key `k` that matches `hashKey` within a time period, the payment is reverted
		If `B` provides a valid fault proof within a time period, the payment is reverted
		Otherwise the payment is executed and send to `A`

3. `A` sends the key `key'` to the smart contract. The contract verifies that `h(key')` matches `hashKey`.

4. `B` now takes the key `key` and decrypts the data they received:
	`data' = DEC(encData, key)`
	If the hash of the decrypted data matches the known `hash`, B terminates and the payment is executed.
	If the hash does not match, `B` sends `data'` as a fault proof to the smart contract.
	The smart contract verifies the fault proof and reverts the payment if the fault proof was correct.

		- It computes `encData'`= ENC(data', key)` 
		- and verifies that `h(encData') == hashEnc` and `h(data') != `h(data)`

5. If the smart contract did not receive a fault proof within a certain timeout, the transaction is executed and the funds are unlocked for `A`.

![Example Construction](https://raw.githubusercontent.com/MariusVanDerWijden/mariusvanderwijden.github.io/master/_posts/non-attributable-fault.png)

### Extensions

This example construction is simplified. In practice `data` can be quite large and publishing it on the blockchain in the case of a fault would be expensive. Thus the scheme can be easily extended to use merkle trees to keep the fault proof size low.

## Non-Attributable Faults in MEV 

There is an inherent non-attributable fault in the MEV supply chain. 
Block producers pay validators to include their block. But validators have to take the word of block producers, that the block is correctly constructed and matches the blinded header that they receive first. This fault gave rise to a trusted intermediary, the relay, which connects validators and block producers and verifies that the blocks are created correctly (simplified, they also act as block producers themselves, bundling transactions from searchers). 

### Construction

In the following construction, we shorten the validator to `V` and the block producer to `B`.
Right now the bidding process is kinda backwards in my opinion. Instead of validators buying blocks, they sell block space. But we can easily redefine it. This has some implications on how blocks are constructed, because any MEV now needs to go to the validator instead of the block producer, but they pay for it.

0.1 `V` solicits bids by announcing that they are eligible to create the next block.
0.2 Block producers bid on the slot by announcing to `V` that they have a block with hash `hash` that pays the coinbase `value` ether in extracted MEV. They are willing to sell the block to `V` for `cost` ether. Thus `value - cost` is the profit for `V`, `cost` ist the profit for `B`. `V` chooses the block `n` that maximizes their `value - cost`. 

1. `V` wants to buy block with hash `hash` from `B` and notifies `B` of its intention
2. `B` sends `{hashEnc, hashKey, encData}` to `V` 
3. `V` signs a transaction `tx` that will send `cost` eth to `B` iff
	- The preimage to `hashKey` was revealed within timeout
	- No fault proof was provided within timeout
	`V` sends the transaction `{tx}` to `B`
4. `B` verifies `tx` and sends `{key}` to `V`
5. `V` decrypts block `n`, signs it and sends it to the network
6. `B` sends `tx` to the network

#### Fault Proof Construction I

The fault proof works as follows:

`B` sends the following data {txs, postState, difference} to `A`.
The fault proof contains {preState, txs, postState, difference}
All transactions are simulated in the smart contract. The smart contract verifies a proof to account of `A` in the pre-state and the post-state.
The fault proof is valid, if the difference between pre and post-state are unequal to the difference claimed by `B`. 

Problems:

Simulating all transactions of a block within a smart contract is extremely costly and won't work for big transaction bundles.

Improvements:
- Provide only the preStateHash and proofs to the preState.
- Provide only the postStateHash and proofs to the postState.
- Provide intermediate differences, so only one transaction needs to be simulated

#### Fault Proof Construction II

The fault proof contains {preStateHash, tx, postStateHash, difference}.

In this construction we only provide proofs to the preState, postState and one transaction.
`B` needs to send the following data to `V`: `{preStateHash, tx_1, postStateHash_1, difference_1, ..., sumDifference}`.

The fault proofs now has two parts:
1. if `B` claimed that `sumDifference` is higher/lower than `sum(difference_x)`:
	`V` provides merkle proofs for `{difference_1, ..., sumDifference}` to the contract
2. if `B` claimed a wrong state transition of transaction with index i:
	`V` provides merkle proofs for `{postStateHash_i-1, tx_i, postStateHash_i}` to the contract.

Problems:

Simulating a big transaction is already costly and might cost more than a full block itself.

### Conclusion

I think the only viable solution to proving transaction execution on chain is to merklelize the full transaction execution. So that every single operation is part of the proof only the state transition where `B` cheated has to be put on chain in order to proof the fault.
I'm pretty sure that is exactly how fraud proofs for optimistic rollups work, so these schemes can probably be adapted to work with this.

The only big open problem I see is that the transaction `tx` might not be valid at point 6 anymore, ideally it should be published beforehand. This would mean that the proposer would need to lock enough funds before his slot to pay out a (theoretically) unlimited bounty. 

Even if we can not solve the trustless data exchange in the case of MEV with this protocol, its still a neat technique that can be used to fairly exchange digital assets and thus should be part of the toolbox of every developer.


(1) FairSwap: How to fairly exchange digital goods: https://eprint.iacr.org/2018/740.pdf