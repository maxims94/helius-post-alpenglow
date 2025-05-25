# Alpenglow: The future of Solana's consensus

Alpenglow is the **biggest change to Solana's core protocols** that the blockchain has ever seen. Developed by Roger Wattenhofer, Quentin Kniep and Kobi Sliwinski from the ETH Zurich and unveiled at Accelerate 2025, Alpenglow represents a fundamental shift in Solana's consensus mechanism, replacing long-established mechanisms such as TowerBFT and Proof-of-History.

The most compelling characteristic of Alpenglow is a **dramatic reduction in finalization latency**. In the current system, it takes 12.8s on average to finalize a block. Once Solana has transitioned to Alpenglow, finality can be reached in a median time of 150ms. That's a staggering 100x improvement over the current system, rivaling Web2 infrastructure in responsiveness.

The protocol achieves this speed-up while also **bolstering network security and resilience**. Under some realistic assumptions, the network will remain operational even if up to 40% of nodes are faulty. This is achieved through its unique 20+20 model, which marks a paradigm shift in attack modeling.

Let's explore how it works and what it means for Solana's future.

## What is Alpenglow?

Alpenglow is a **novel consensus protocol** specifically designed for high-performance proof-of-stake blockchains. Like any consensus protocol, its purpose is to create agreement on the ledger's state between the nodes of the network.

While maintaining Solana's fundamental structure, Alpenglow replaces some of its components with enhanced versions and introduces some key innovations.

As in the current protocol, time is segmented into slots, each with a designated leader. The leader is responsible for receiving transactions and constructing them into blocks.

To efficiently disseminate the blocks to the network, they use **Rotor**, a new protocol introduced by Alpenglow.

Once a block has been propagated, nodes start to engage in a voting process. This is governed by the second new protocol, **Votor**. The voting process occurs over a 1-2 rounds of voting, ending with the decision to either append the block to the blockchain or reject it.

## Rotor: Block propagation

A **core challenge** in blockchain technology is efficient block propagation: how does a leader distribute a newly created, potentially large (128MB in Solana) block to the entire network without being constrained by its own bandwidth?

Currently, Solana's solution to this is [Turbine](https://www.helius.dev/blog/turbine-block-propagation-on-solana). Turbine arranges all nodes in a hierarchical structure called the turbine tree. The leader sends the block to the tree's root and each node then forwards the data to a unique subset of nodes in the next layer, as determined by the turbine tree. This approach minimizes communication overhead compared to sequential or flooded propagation. It is crucial for Solana's high throughput and scalability. 

In the future, Solana will adopt Rotor, a newly designed efficient block propagation protocol.

**Rotor's design is inspired by Turbine and retains its core strengths, but simplifies the architecture.** Unlike Turbine's multi-layered structure, Rotor uses a single layer of relay nodes to disseminate data from the leader to all other nodes. The relay nodes are regular nodes that are selected according to a predefined method. For each use of Rotor, a different subset of nodes is selected to function as relays. One innovation of Alpenglow is a novel selection method that is particularly resilient.

The rationale behind Rotor's single-layer design stems from the network delay introduced by each additional layer in Turbine's architecture. Each additional layer leads to an additional network hop, which becomes a significant bottleneck in practice (the speed of light is too slow!). So, the goal of the design is to minimize the number of network hops, which can be achieved by sticking to a single retransmission layer.

In total, **Rotor can be seen as a simplified and optimized version of Turbine**.

## Votor: Voting algorithm

After a block has been distributed via Rotor, the voting process starts. For this, Alpenglow uses its novel voting protocol, Votor.

Votor runs two voting path concurrently: a primary path for the typical network situation and a fallback path for less ideal conditions. All nodes always start both paths. Only one path needs to finish for a node to finalize a block. This way, we take the faster of the two and thereby reduce latency.

Votor has two voting paths:
* Path 1: If at least 80% of stake participates, the block is finalized after one round of voting
* Path 2: If at least 60% of stake participates, the block is finalized after two rounds of voting

The dominant factor in how long a round of voting takes is the network delay. This is mostly determined by the geographical position of a node relative to the other nodes.

If a node is part of geographically close cluster (e.g. with a latency of 5ms), then the two rounds of voting may happen much faster (in ca. 10ms) than a single round that needs to include remote notes.

Conversely, if a node is far away from most other nodes, then the latency is so high that one round of voting is done much earlier than two rounds.

By concurrently executing both paths, Votor dynamically adapts to network conditions, ensuring rapid consensus.

This concurrent approach reduces latency. With longer network delays, the single-round path is often faster. In scenarios with low network delays, the two-round path can be quicker. Votor's design allows it to adapt to varying network conditions to ensure efficient consensus.

## Fault tolerance

### Three types of nodes

The cleverness of Alpenglow comes from splitting up faulty nodes into Byzantine and down nodes. In total, our model considers three types of nodes:

Correct node: A node following the consensus protocol

Byzantine node: A node exhibiting a Byzantine fault is faulty in an arbitrary way. It might be sending incorrect messages, wrongly confirming blocks, or trying to disrupt the consensus process.

Down Node: A node that is failing to participate in the network. It's silent rather than malicious. This could be due to a software crash, hardware failure or loss of network connectivity.

Why treat down nodes specially / separate them out? Since truly malicious nodes are very rare (there's little incentive to), but it happens frequently that nodes go down (hardware issue, network issue, software crashes)

So, in a real large scale distributed blockchain system, we will probably see significantly less than 33% byzantines. Instead, realistic bad behavior often comes from machine misconfigurations, software bugs, and network or power outages. In other words, large scale faults are likely accidents rather than coordinated attacks.

### 20+20 resilience

Alpenglow has a unique 20+20 model of network resilience. It guarantees the network operability as long as less than 20% of stake is controlled by Byzantine nodes and up to 20% of stake is controlled by down nodes.

This allows the protocol to operate even in harsh network conditions.

While Alpenglow does not reach the best theoretically possible fault tolerance against Byzantine nodes of 33% (like today's Solana), it vastly improves on the most common case of down nodes.

This is a design choice that reflects a practical observation: in large-scale systems, many failures are due to things like network outages or hardware crashes, not necessarily intentional malice. Alpenglow aims to handle a high volume of these "accidental" failures in addition to a base level of malicious behavior.

### Illustration

It can be hard to wrap your head around what the 20+20 model means for the network's robustness in practical terms. Let's understand it better.

The key observation is that a down node can be seen as a Byzantine node since being unavailable to the network is a special case of a Byzantine fault.

So, you can say that the network is stable if the faulty nodes can be split up into < 20% Byzantine and <= 20% down nodes.

Some examples:

|Scenario                    | Network              | Explanation                            |
|----------------------------|----------------------|----------------------------------------|
|5%  Byzantine, 15% down     | stable               | both conditions hold                   |
|21% Byzantine, 0%  down     | breaks               | too many Byzantine nodes               |
|10% Byzantine, 25% down     | stable               | count 5% of down nodes as Byzantine    |
|10% Byzantine, 33% down     | breaks               | can't count 13% as Byzantine           |

In the common case, the portion of Byzantine stake is very low (<5%), leaving room to deal with network outages and crashed nodes.

When can an attacker successfully shut down the network?
* When they get > 20% stake,
* Or, if they acquire p% stake, but also manage to shut down more than 40-p% of nodes (e.g. 10% and 31%)

## Latency

How long does it take Alpenglow to finalize a block? More precisely, how long does it take for a node to receive a block via Rotor and then finalize it via Votor?

The fundamental lower bound is the base network latency. That's the time to send one packet from the leader to a node. You can't go lower than that.

As a rule of thumb, the time it takes to finalize a block is roughly 2x the base network latency. So, if the latency is 75ms, then it takes 150ms to finalize the block.

Keep in mind that this refers to *finality*, i.e. the final decision on whether to put a transaction into the blockchain -- not optimistic confirmation or similar.

This is a remarkably low figure for a globally distributed Layer-1 blockchain.

This dramatic reduction in latency directly translates to a significantly improved user experience, with transactions reaching finality almost instantaneously and users seeing almost real-time results.

## Multiple concurrent leaders

It's likely that Alpenglow's approach makes it easier to [implement](https://x.com/aeyakovenko/status/1924814494808891843) multiple concurrent leaders.

This would allow Solana to [minimize how much MEV](https://x.com/aeyakovenko/status/1810222589991583922) is extracted from users: If users can choose between multiple leaders, they can pick the best offer from leaders, which leads to competition between them

It would also make it feasible to create an on-chain [central limit order book](https://www.anza.xyz/blog/the-path-to-decentralized-nasdaq).

In that way, Alpenglow's technical revolution sets the stage for more improvements to Solana in the future.

## Conclusion

Alpenglow marks a major advancement for the Solana blockchain.

Its new consensus mechanism enables a drastic reduction in latency, unlocking an unprecedented level of performance and enabling a new generation of real-time applications.

Alpenglow achieves this while also providing enhanced security guarantees.

Furthermore, it paves the way for future improvements, such as support for multiple concurrent leaders.

These advancements promise significant changes for both users and developers, signaling a bright future for the Solana ecosystem. 

In broader terms, this comprehensive redesign of Solana's consensus architecture unlocks unprecedented performance levels.  This positions Solana to support a new generation of real-time applications and solidifies its position as a leading global high-performance proof-of-stake blockchain. 

## Additional Resources

[Accelerate 2025 Presentation: Introducing Alpenglow](https://www.youtube.com/watch?v=x1sxtm-dvyE)

[Alpenglow: A New Consensus for Solana](https://www.anza.xyz/blog/alpenglow-a-new-consensus-for-solana)

[Solana Alpenglow Whitepaper](https://drive.google.com/file/d/1y_7ddr8oNOknTQYHzXeeMD2ProQ0WjMs/view)
