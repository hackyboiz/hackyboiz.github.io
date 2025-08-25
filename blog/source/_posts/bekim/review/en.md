---
title: "[Research] Paper Review (USENIX ’24) Speculative DoS Attacks in Ethereum (en)"
author: bekim
tags: [review, blockchain, ethereum, usenix, Dos, bekim]
categories: [Research]
date: 2025-08-25 17:00:00
cc: true
index_img: /2025/08/25/bekim/review/en/image.png
---


# 1. Introduction

Hello, this is bekim! I was writing articles related to blockchain when I got a job and became too busy, so I will review some papers! I searched for papers from Usenix Security 24 or 25, and chose topics related to blockchain security, which I have been dealing with on a regular basis. So, I will review the papers assuming that the readers of this article have a basic understanding of blockchain.

![](en/image.png)

Today's review is on a paper titled **Speculative DoS Attacks in Ethereum**. This paper presents a new form of denial-of-service (DoS) attack that could occur in Ethereum. The researchers demonstrate how three main attack methods **(ConditionalExhaust, MemPurge, and GhostTX)** can exploit Ethereum's transaction processing and resource management structure. All experiments were conducted on the Ethereum testnet.

# 2. Paper Review

In my previous research paper [From Paper to Code: Smart Contract](https://hackyboiz.github.io/2025/07/13/bekim/smartcontract/en/)”, I briefly explained the gas mechanism in the section describing how Ethereum works. If you find this concept difficult to understand, you may want to read it first!

## 2.1 Background

Ethereum primarily utilizes the gas mechanism to prevent excessive computation and provides incentives to nodes (validators) operating the network. When executing a transaction, gas must be paid based on the computational load, and this gas has served as a mechanism to protect network resources. Therefore, until now, there has been a perception that DoS attacks are very difficult on Ethereum.

However, this paper presents a different perspective.

Due to the Turing-complete nature of smart contracts, nodes must execute a transaction to verify whether it can actually be included in a block.
The costs incurred in this process are not compensated by fees unless the transaction is included in the block. In other words, attackers can create malicious transactions that “force work but do not provide compensation.”

The paper introduces three new attack techniques to demonstrate this.

- **ConditionalExhaust**: A resource exhaustion attack that occurs only under specific conditions.
- **MemPurge:** An attack that forces normal transactions out of the validator/builder's mempool.
- **GhostTX:** An attack that disrupts the reputation system of Ethereum's PBS (Proposer-Builder Separation) ecosystem.

In particular, using ConditionalExhaust and Mempurge simultaneously can deplete the computational resources of validators while blocking the mempool, preventing victims from including transactions in blocks. This could result in victims generating empty blocks and threatening the network's activity.

## 2.2 Proposed Attacks

### 2.2.1 The ConditionalExhaust Attack

The first attack we will introduce is the ConditionalExhaust attack, a conditional resource depletion attack.

The core idea of the attack is simple. Validators or block builders following censorship do not include transactions interacting with a specific censored address (σ) in the block. This is because, following the 2022 U.S. OFAC (Office of Foreign Assets Control) sanctions against the smart contract address associated with the Tornado Cash mixing service, some Ethereum validators began filtering out transactions related to that address from blocks. The problem is that **to determine whether a transaction is related to a sanctioned entity (σ), the transaction must be executed unconditionally**. This execution process is exploited by ConditionalExhaust.

![](en/image1.png)
> Complete Overview of Decred's Structure [source: https://medium.com/decred/blockchain-governance-how-decred-iterates-upon-bitcoin-3cc7030c655e]
>

Speculative DoS Attacks in Ethereum (USENIX Security 2024): ConditionalExhaust is a conditional REA, in which
an attacker creates transactions that invoke computationally complex code if the victim cannot include them in a block, for example due to its censoring policy

The attack is divided into two main stages.

First, in the **deployment stage**, the attacker uploads a specially designed smart contract to the blockchain (1).

This contract contains the following conditions:

1) If the executor is a censoring validator, perform complex and heavy computations (2). (3)
2) If not, it performs simple operations and incurs minimal gas costs.

This condition is included in the contract.

Next, in the **execution phase**, the attacker floods the network with multiple transactions calling this contract.

This causes censors to consume excessive CPU and memory resources for each computation, and only after execution completes do they realize the transaction cannot be included in the block because it interacts with the censored address (σ). (4) In other words, resources are wasted without any reward.

Although there is an implementation overview in the actual paper, I will not include it here due to the possibility of code misuse. Researchers designed the code to maximize resource consumption by modifying various elements, such as Ethereum's storage structure and high-cost opcodes.

As a result, while it took only microseconds for the attacker to generate and sign transactions, it took an average of 0.1 seconds for the censor to verify them, resulting in a difference of approximately 1,972 times over two hours. In actual Ethereum, a block can process up to 120 transactions in 12 seconds, but if the attacker sends just 140 transactions, it could result in an empty block being generated without including normal transactions.

The economic cost of the attack was also not significant. Deploying the contract cost $27 (36,000 KRW), and in the worst case, if the attack transactions were included in the block, the cost would be $5.3 (7,000 KRW). Upon actual calculation, the total cost required to execute 140 attack transactions in a single block was only approximately $376 (500,000 KRW).

### 2.2.2 The MemPurge Attack

The second attack is, as the name suggests, an attack that purges the node's mempool. Ethereum nodes store transactions that may be included in future blocks in a storage called **mempool**, which has limited space, so transactions with higher fees are usually prioritized.

The attacker exploits this rule to push out profitable legitimate transactions from the mempool and replace them with their own meaningless transactions.
The core of the attack is to create a transaction chain (TXchain) with consecutive nonces (numbers).

![](en/image2.png)
> Speculative DoS Attacks in Ethereum (USENIX Security 2024): The MemPurge attack lowers the cost to evict transactions from victims’ mempools.
>

Speculative DoS Attacks in Ethereum (USENIX Security 2024): The MemPurge attack lowers the cost to evict transactions from victims’ mempools.

1) First, the attacker creates the first transaction (nonce=N) to transfer funds from their account to a new address, and then adds meaningless transactions such as Nonce = N+1, N+2, ... to form a TX Chain. (1)
2) The attacker sends subsequent transactions to the victim node (2), and since the victim node receives them without any preceding transactions, it temporarily stores them in the future queue (3).
3) Finally, when the attacker sends the first transaction (N), the victim node verifies it and moves the entire previous transaction chain to the pending queue (4).
4) However, since the attacker's funds have already been withdrawn in the first transaction, subsequent transactions become invalid due to insufficient balance. As a result, the mempool is filled with meaningless transactions, and the originally profitable transactions are pushed out.
- **ConditionalExhaust + MemPurge**
Mempurge and ConditionalExhaust can be combined. By setting the to address of each MemPurge transaction to ConditionalExhaust, MemPool space is occupied, and ConditionalExhaust consumes computation.
    
By calling the ConditionalExhaust contract in the first transaction to impose CPU and memory computation burdens on the censor, and then invalidating subsequent transactions that only occupy mempool space, a more powerful attack can be executed.
    
Researchers verified this combined attack and reported that, compared to a single attack, the gas cost remained almost unchanged while maintaining the same effectiveness.
    

### 2.2.3 The GhostTX Attack

The GhostTX Attack directly targets the reputation of the Searcher in Flashbots' PBS (Proposer-Builder Separation) environment.

- **PBS** is a structure that separates the roles of block proposers and builders to efficiently construct blocks. Proposers simply select and submit blocks, while builders create the most profitable transaction combinations (bundles) and submit them to proposers.


![](en/image3.png)
> Speculative DoS Attacks in Ethereum (USENIX Security 2024): Overview of Ethereum’s PBS ecosystem actors.
>

- **Flashbots** is an implementation of PBS, providing a **Relay** service that connects Searchers (participants who create Bundles and deliver them to Builders), Builders, and Proposers. This system has a **Reputation** system that scores Searchers based on their past performance, so Bundles from Searchers with high reputations are selected first.
 
Here, reputation is calculated based on **how much profit was generated per gas unit**. Therefore, once reputation drops, the address must be changed to rebuild it, which means attackers can lower the victim's trustworthiness and force them out of the ecosystem.
ΔT: Amount paid to the Builder, pT: Fee, gT: Gas consumption 

```
r(U) = (∑ (ΔT + pT * gT)) / (∑ gT)
```

![](en/image4.png)
> Speculative DoS Attacks in Ethereum (USENIX Security 2024): GhostTX’s censorship variant exploits an inconsistency between builder and relay censorship methods
>


The attack process is divided into **two types (Proposer Variant, Non-proposer Variant)**. Simply put, in the **Proposer Variant**, if the attacker is a block proposer, they spread a bait transaction that appears to be profitable and then invalidate it in the first block to damage the Searcher's reputation.

In the case of the **Non-Proposer**, the attacker simultaneously spreads a collision transaction with the same nonce and fee as the Searcher's bait transaction, ensuring that only the attacker's transaction is recorded in the block.

1) The attacker sends a transaction that transfers funds to a sanctioned address (1)
2) The Searcher identifies this as a profitable transaction and includes it in the bundle. (2)
3) The Builder verifies the bundle (3) and adds the TX to the block (4).
4) The block is transmitted to the Relay, but the Relay detects the transaction related to the sanctioned address and discards the block (5), resulting in the Searcher's bundle being invalidated and their Reputation score decreasing. (6)

The GhostTX Attack can be executed in various ways, one of which is the **Censorship Variant**. This exploits the subtle differences in how Flashbots' Builder and Relay perform censorship.

While Builder's internal verification does not consider 0 ETH transfers using the call operator as sanction targets, Relay's verification API does consider them as such. Attackers can exploit this by creating transactions that send 0 ETH to sanctioned addresses, making them appear as normal transactions to Searcher and Builder, but only getting filtered out at the Relay stage. This results in Searcher's bundles being invalidated again and its Reputation score decreasing.

![](en/image5.png)
> Speculative DoS Attacks in Ethereum (USENIX Security 2024): Figure 8: Cost of attacking average searchers with GhostTX
>


The researchers evaluated the effectiveness of GhostTX attacks based on actual data. For top-tier Searchers, they were generating approximately 8,240 ETH in profits and 112 billion gas usage. To push them out of the top 50% in Reputation, it would require a cost of approximately 42.49 million dollars. While it is practically impossible to realistically bring down established large-scale Searchers, the results were different for average-level Searchers.

On average, they recorded approximately 0.95 ETH in profits, 3.28 million gas usage, and a Reputation score of 2.9×10¹¹. It was reported that only 9,820 dollars would be sufficient to push them down to the bottom 60% level. In other words, while the GhostTX attack is difficult to execute against large Searchers, it can relatively cheaply undermine the Reputation of medium-sized Searchers or newly entering Searchers.

# 3. Conclusion

As the authors of the paper also pointed out, there are several practical constraints that prevent the proposed attacks from being directly applied in real-world environments. For example, ConditionalExhaust requires the attacker to spread a large number of transactions simultaneously to maximize its effectiveness, which could incur significant network layer costs in practice. Additionally, there may be discrepancies between the attack costs mentioned in the paper and actual attack costs, leading to varying assessments of the threat level depending on the environment.

Personally, I wonder how realistic a threat this could be on the mainnet. ConditionalExhaust and MemPurge seem impractical for threatening the mainnet, while GhostTX appears to be a more feasible attack in reality…

The paper does propose mitigations for these attacks, but upon reading them, most seem to either degrade user experience or reduce network efficiency, so they are not yet perfect solutions. In conclusion, while these attacks may not pose an immediate, substantial threat capable of destabilizing the mainnet, the paper does highlight some structural vulnerabilities in Ethereum's architecture.

Thank you for reading this sudden paper review all the way through! 🙏 I'll be back soon with more interesting topics.