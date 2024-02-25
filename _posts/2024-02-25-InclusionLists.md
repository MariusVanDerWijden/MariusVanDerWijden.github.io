---
layout:        post
title:         In support of simple inclusion lists
date:          2024-01-11 16:00:00
summary:       This post should illustrate my ideas around inclusion lists and why a simpler approach might be better in the short term over existing proposals
categories:    blog
tags:          ethereum inclusion lists pbs mev
---

Since I joined the Foundation in April 2020, a feeling grew stronger that we are playing whack-a-mole with the protocol. Optional access-lists to un-break a few contracts, preserving self-destruct if called within a deployment, Push 0 and many more. Yes they enable legitimate use-cases, but are they really worth the effort and complexity that they add to the protocol?

Inclusion lists have been proposed in order to preserve the censorship resistance of the protocol. The current proposal (as far as I understand) proposes that validators that want to have certain transactions included, will add them to a list and the proposer for the next slot will need to include them or miss their block proposal.

This will, in theory, force block producers to include transactions into their block, that they otherwise wouldn't and allow them to argue that they are forced by the protocol to include those transactions even if they don't want to. I disagree, because I think in practice it would mean that they will be forced to not participate in the protocol anymore. Forcing someone to include anything against their will, will not work very long and validators and block producers will be forced to stop playing the game.

There might be workable schemes in the future, but the proposals I have seen were not great in my opinion and would not move the status quo. 

I am in favor of implementing an optional inclusion list feature in clients as follows:

- Validators that want to include transactions to or from certain accounts can configure these accounts in their Execution Layer Client.
- Before proposing a block, validators ask their EL client for a list of transactions matching their criteria, the inclusion list
- Validators send the inclusion list to the builders that they are connected to via MEV-Boost
- Relays and Builders can decide whether they would like to build a block with those transactions
- If they refuse to build a block with those transactions, they can just not bid on it and the block will be built by willing relayers/builders or the local execution layer client
- If they accept to build the block, they send an inclusion proof of the transactions and the blinded block to the validator, as usual in MEV-Boost
- The validator signs the highest bidding block that has the inclusion proof for the transactions that they required

The inclusion proof is very easy to build, since the transaction hashes are merkelized in the transaction root of the execution block. 

This scheme gives us the best of both worlds in my opinion:

- Validators do not miss out on MEV-blocks if no censorship is happening
- Relays and Block builders can enforce their policies of not building certain blocks
- Relays and Block builders are incentivized to not censor as they will loose significant market-share otherwise
- Validators can prioritize their own transactions even if they pay below market tips.

This scheme can be implemented quite quickly without a hard-fork by adding optional fields to the messages between relay and CL client, and adding two new endpoints to the Execution Layer clients: `engine_setLocalAccounts` and `engine_getLocalTransactions`. 

In conclusion: I would prefer implementing an optional protocol that does not need a hard-fork and rolling it out as soon as possible over adding a complicated scheme under consensus were we don't know whether Relays and Block builders will follow it and where having multiple consecutive blocks gives bigger stakers and pools an advantage over smaller operations. 