---
layout:        post
title:         Ethereum's biggest issue no one is talking about
date:          2021-11-07 20:00:00
summary:       Private orderflow and MEV is a deadly combination, but trustless light clients might be able solve it
categories:    blog
tags:          Private Orderflow, Ethereum, Lightclient, MEV
---

We recently had a long workshop in Bogota with the Flashbots team and other Relay runners.
Afterwards I started thinking a bit about how to get rid of flashbots and democratize MEV even further.
One inital idea I had was to create a geth fork that could run bunch of basic MEV strategies (like arbitrage, liquidations, etc).
All the strategies would be open source under a very strict license s.th. interested developers could add their strategies and no one (at least not big companies) is able to modify the code and not upstream their contributions.
This would bring back power to the validators. 

A single liquidation can bring a validator > 100 ETH. 
All of this value is currently extracted by block builders and they only have to pay just enough to get their transactions into a block.
This value is also not reinvested in the ecosystem. 
Tokens are dumped and ETH is (usually) sold.
If this value would go to the validators, they would (likely) reinvest it into validators, making Ethereum more secure.

So far so good, why are we not doing this then?
This is where we arrive at the crux of the problem; **private orderflow**.
If we moved the block production back to the individual validators, the public mempool would start to matter more.
In order to find the arbitrage and liquidation opportunities, the validators would need to see those transactions.
Fact is however, that most transactions are currently send via public RPC endpoints. 
Thus a handfull of centralized services have control over the orderflow. 
They essentially choose which nodes to supply these transactions to.

In a world where block building is done by the validators, access to the private orderflow becomes crucial.
Thus big staking pools might start to make deals with RPC providers to get access to the orderflow.
This is bad and will result in centralization around the RPC providers as he who controls the transactions also controls the MEV.

TBH. private orderflow is also a problem right now with centralized block builders.
The builders that get access to the most transactions can extract the most MEV so they can pay more for transactions etc.
But there might be a solution.

Trustless light clients are not very feasible on Ethereum right now.
They would require merkle proofs for all state changes in a block which can easily result in > 500MB of proof data for a single block.
The move from merkle trees to verkle trees will enable two things.
Light clients will be able to **trustlessly** follow the chain by downloading the headers and a proof for the transactions in the block.
But light clients will also be able to **trustlessly** execute `eth_call` by asking full nodes (or centralized providers) for certain parts of the state (and proofs for it). 
Thus light clients will be able to create transactions and estimate their gas usage for example.

How does this relate to private orderflow?
Given a strong and easy to use implementation of trustless light clients, we might see wallets moving away from defaulting to centralized RPC providers to running their own light clients in the background.
Having an army of hundred thousands of light clients running on users computers (and maybe mobile phones) everywhere will make it way more difficult to capture the orderflow.

In conclusion private orderflow is bad, because whomever controls the transactions can extract the most MEV and thus become a central player.
We can fight private orderflow by sending more transactions directly to the public mempool by running our own nodes.
If trustless light clients become usable and economic we will hopefully see wallets move away from centralized RPC providers.
In order to enable trustless light clients we need verkle trees and a lot of engineering work.