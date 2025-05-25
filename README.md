# Alpenglow: The future of consensus on Solana

**Alpenglow is the biggest change to Solana's core protocols in the history of the blockchain.** Developed by Prof. Dr. Roger Wattenhofer, Quentin Kniep and Kobi Sliwinski from the ETH Zurich and unveiled at Accelerate 2025, Alpenglow represents a fundamental shift in Solana's consensus mechanism, replacing long-established mechanisms such as TowerBFT and Proof-of-History.

The most compelling characteristic of Alpenglow is a **dramatic reduction in finalization latency**. In the current system, finalizing a block takes 12.8s on average. Once Solana has transitioned to Alpenglow, finality can be reached in a median time of 150ms. That's a staggering 85x improvement over the current system, rivaling Web2 infrastructure in responsiveness.

The protocol achieves this speed-up while also **bolstering network security and resilience**. Under realistic assumptions, the network will remain operational even if up to 40% of nodes are faulty. This is achieved through its unique 20+20 model, which marks a paradigm shift in attack modeling.

In this guide, we explore how it works and what it means for Solana's future.

## What is Alpenglow?

Alpenglow is a **novel consensus protocol** specifically designed for high-performance proof-of-stake blockchains. Like any consensus protocol, its purpose is to create agreement on the ledger's state between the nodes of the network.

While **maintaining Solana's fundamental structure**, Alpenglow replaces some of Solana's components with improved versions and introduces some key innovations.

Following the structure of the current protocol, time is segmented into slots, each with a designated leader. The leader is responsible for receiving transactions and constructing them into blocks.

To efficiently disseminate the blocks to the network, Alpenglow uses **Rotor**, a newly introduced block propagation protocol.

Once a block has been propagated, nodes start voting on it. The voting process is governed by the second new protocol, **Votor**. The voting finishes after one or maximally two rounds of voting, ending with a decision on whether to accept the block or not.

## Rotor: Block propagation

A core challenge in blockchain technology is **efficient block propagation**: how does a leader distribute a newly created, potentially large block (128MB in Solana) to the entire network without being constrained by its own bandwidth?

**For now, Solana's solution to this is [Turbine](https://www.helius.dev/blog/turbine-block-propagation-on-solana).** Turbine arranges all nodes in a hierarchical structure called the turbine tree. The leader sends the block to the tree's root and each node then forwards the data to a unique subset of nodes in the next layer, as determined by the turbine tree. This approach minimizes communication overhead compared to sequential or flooded propagation. It is crucial for Solana's high throughput and scalability. 

In the future, Solana will adopt Rotor, a newly designed efficient block propagation protocol.

**Rotor's design is inspired by Turbine and retains its core strengths, but uses a simplified architecture.** Unlike Turbine's multi-layered structure, Rotor uses a single layer of relay nodes to disseminate data from the leader to all other nodes. The relay nodes are regular nodes selected by a predefined method. For each use of Rotor, a different subset of nodes is selected to function as relays. One innovation of Alpenglow is a novel selection method that is particularly resilient.

The rationale behind Rotor's single-layer design stems from the network delay introduced by each additional layer in Turbine's architecture. Each additional layer requires one more network hop, which becomes a significant bottleneck in practice (the speed of light is too slow!). So, the goal of the design is to minimize the number of network hops, which can be achieved by sticking to a single retransmission layer.

**In total, Rotor can be seen as a simplified and optimized version of Turbine**. This is reflected in its name: it's a single rotor as opposed to a turbine which consists of a stack of rotating blades.

## Votor: Voting algorithm

After a block has been distributed by Rotor, the nodes start to vote on it. **This voting process is governed by Alpenglow's novel voting protocol, Votor.**

**The core idea behind Votor is to run two voting paths at the same time and let nodes pick the one that's faster for *them*.** This means: Every node always starts voting on both paths. But it only needs to complete one of these paths to finalize a block. As a result, it automatically picks the path that is faster for it. This ensures a relatively low latency for all nodes, independent of their geographical location. It also allows the system to flexibly adapt to changes in network topology. **The overall effect is a dramatic decrease in average latency.**

In Votor, nodes operate on two voting paths:
* Path 1: If at least 80% of stake participates, the block is finalized after one round of voting
* Path 2: If at least 60% of stake participates, the block is finalized after two rounds of voting

The dominant factor in how long a round of voting takes is the network delay. This is mostly determined by the geographical location of a node relative to the other nodes.

If a node is part of a **geographically close high-stake cluster** (e.g. with a latency of 5ms), then the two rounds of voting happen much faster (in ca. 10ms) than a single round that needs to include remote nodes with high latencies (e.g. 100ms).

Conversely, if a **node is far away from most other nodes** (more precisely, stake), then the latency is so high that one round of voting finishes much earlier than two rounds (where the high latency is effectively doubled).

By executing both voting paths concurrently, Votor dynamically adapts to network conditions, ensuring rapid consensus at all times and for all nodes.

## Fault tolerance

### Three types of nodes

Before we can reason about the fault tolerance of Alpenglow's network, we need to introduce a model of node types. Usually, a model only distinguishes between two types of nodes: correct and Byzantine. **The cleverness of Alpenglow comes from splitting up faulty nodes into Byzantine and down nodes.** So, in total, the model considers three types of nodes:

**Correct node**: A node that strictly adheres to the consensus protocol

**Byzantine node**: A node that is faulty in an arbitrary way. It might be actively malicious, send syntactically incorrect messages, confirm invalid blocks or try to disrupt the consensus process.

**Down Node**: A node that is failing to participate in the network. It's silent rather than malicious. This could be due to a software crash, hardware failure or loss of network connectivity.

Why do we treat down nodes as a separate case? In real-world blockchain systems, Byzantine nodes are very rare since nodes have no incentive to behave that way. Instead, bad behavior often comes from machine misconfigurations, software crashes, hardware issues and network or power outages. In other words, large-scale faults are likely accidents rather than coordinated attacks. Evidently, down nodes are easier to manage than Byzantine nodes. **By separating these two failure modes, we can make stronger guarantees about the network's fault tolerance.**

### 20+20 resilience

Alpenglow has a **unique 20+20 model of network resilience**. It guarantees the network's operability as long as less than 20% of stake is controlled by Byzantine nodes and up to 20% of stake is controlled by down nodes. This means that, even under harsh conditions, the network will continue to function.

While Alpenglow does not reach the best possible Byzantine fault tolerance of 33% (like today's Solana), it vastly improves on the most common and realistic case of down nodes.

This design choice reflects a practical observation: in large-scale systems, many failures are due to mishaps like network outages or hardware crashes, not intentional malice. **Alpenglow aims to handle a high volume of these "accidental" failures in addition to a base level of malicious behavior.**

### Illustration

It can be tricky to really understand what the 20+20 model means in practice.

The key observation is that a down node can be seen as a Byzantine node since being unavailable to the network is a special case of a Byzantine fault.

So, we can say that the network remains stable if the faulty nodes can be split up into < 20% Byzantine and <= 20% down nodes.

Some examples:

|Scenario                    | Network              | Explanation                              |
|----------------------------|----------------------|------------------------------------------|
|5%  Byzantine, 15% down     | stable               | both conditions hold                     |
|21% Byzantine, 0%  down     | breaks               | >= 20% Byzantine nodes                   |
|10% Byzantine, 25% down     | stable               | count 5% of down nodes as Byzantine      |
|10% Byzantine, 33% down     | breaks               | can't count the excess 13% as Byzantine  |

In the common case, the portion of Byzantine stake is very low (<5%), leaving plenty of room to deal with network outages and crashed nodes.

When can an attacker successfully shut down the network?
* When they get > 20% stake,
* Or, if they acquire p% stake, but also manage to shut down more than 40-p% of nodes (e.g. 10% and 31%)

## Latency

**How long does it take for Alpenglow to finalize a block?** More precisely: how long does it take for a leader to send out a block and then for the nodes to finalize it?

Here, the fundamental lower bound is the base network latency. That's the time to send an arbitrary data packet from the leader to a node. You can't go any lower than that.

[Results](https://drive.google.com/file/d/1y_7ddr8oNOknTQYHzXeeMD2ProQ0WjMs/view) from experiments on a reference implementation indicate the following rule of thumb: **the time it takes to distribute and finalize a block is roughly 2x the base network latency.** So, if the latency is 75ms, it takes 150ms to finalize a block.

Keep in mind that this refers to **finality**, i.e. the final decision on whether to put a transaction into the blockchain -- not optimistic confirmation or similar.

Also, keep in mind that we're talking about executing transactions on a **globally distributed Layer-1 blockchain** managing assets worth billions of dollars.

Taking this into account, **this is a remarkably low figure!**

This drastic reduction in latency **directly leads to a significantly improved user experience**, with transactions reaching finality almost instantly and users seeing results almost in real-time.

## Multiple concurrent leaders

Alpenglow's approach will likely simplify the [implementation](https://x.com/aeyakovenko/status/1924814494808891843) of multiple concurrent leaders.

This would allow Solana to [minimize the amount of MEV](https://x.com/aeyakovenko/status/1810222589991583922) is extracted from users: If users can choose between multiple leaders, they can choose the most favorable offer, which leads to competition among leaders to minimize MEV in their proposed block.

This would also make it feasible to develop an on-chain [central limit order book](https://www.anza.xyz/blog/the-path-to-decentralized-nasdaq).

This way, Alpenglow's technological revolution sets the stage for more even improvements on Solana.

## Conclusion

With the advent of Alpenglow, Solana is poised for a new era of performance and capability.

This innovative consensus protocol will lead to a dramatic reduction in latency, unlocking unprecedented levels of performance and enabling the development of a new generation of real-time applications.

Alpenglow achieves this while also bolstering network security.

Furthermore, it paves the way for future improvements, such as support for multiple concurrent leaders.

These advancements promise a bright future for the Solana ecosystem, offering benefits to both users and developers. They solidify Solana's position as the leading high-performance proof-of-stake blockchain.

## Additional Resources

[Accelerate 2025 Presentation: Introducing Alpenglow](https://www.youtube.com/watch?v=x1sxtm-dvyE)

[Alpenglow: A New Consensus for Solana](https://www.anza.xyz/blog/alpenglow-a-new-consensus-for-solana)

[Solana Alpenglow Whitepaper](https://drive.google.com/file/d/1y_7ddr8oNOknTQYHzXeeMD2ProQ0WjMs/view)
