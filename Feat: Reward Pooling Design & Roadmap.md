by IPFSForce

Nov 2021

## Background

As the storage power of filecoin network surpassed the [network baseline](https://filecoin.io/blog/posts/filecoin-network-crosses-baseline-sustainability-target-for-first-time/) and kept growing at a [record pace](https://filscan.io/statistics/power) after the [hyperdrive upgrade](https://filecoin.io/blog/posts/filecoin-v13-hyperdrive-network-upgrade-unlocks-10-25x-increase-in-storage-onboarding/), small to medium sized storage provider may find their reward stream become [increasingly unstable](https://filecoinproject.slack.com/archives/CEGN061C5/p1626916976253400). Pooling rewards between storage providers has been [discussed](https://github.com/filecoin-project/lotus/discussions/4005) as a potential solution to this problem, but no one has taken on the challenge of realizing it just yet. After much research efforts combined with a variety of [experiences](https://filecoin.io/blog/posts/welcome-venus-to-the-filecoin-mainnet/) gained from operating storage systems, the IPFSForce is pleased to take on the task of designing and implementing a reward pooling feature for Filecoin!

Reward pooling has been on IPFSForce' plate for quite a while. It is based on the implementation of Filecoin - Venus, taking advantage of its distributed architecture and consolidated block producing capability. This document will present the design of reward pooling of the internal alpha version which is not final and subject to changes. Comments and suggestions are always welcomed!

## Approach

Before diving into the specifics, we will walk you through the general overview of the current approach.

### Setup

We decided to take a shared owner address approach where storage provider joining the reward pool either change their owner address to a designated owner address provided by the reward pool admin or have the admin init a new miner id for them. From this point, the storage provider would then continue their storage providing operation as usual.

![image-20211108160607462](https://i.loli.net/2021/11/08/sp5xmXHuFaqthTV.png)

### Reward pool

The reward of each individual storage provider will be tracked off-chain by the reward pool, which follows token economy of Filecoin where 25% is immediate and the rest of 75% will be vesting linearly over 180 days. When new block is mined, rewards will be distributed to each storage provider's account according to how much storage power they contribute to the pool. For example, contributing 30% of the storage power of the pool translates into getting 30% of the total rewards. Rewards then can be withdrawn from the reward pool on demand.

![image-20211108161748395](https://i.loli.net/2021/11/08/ZpOG31dV9tr6laU.png)

## Goals

There are two aspects of reward pooling, namely the pooling and the distribution. Pooling is the act of literally pooling the rewards of participating storage providers and distribution dictates how storage provider have their fair share of the pool. It's worth mentioning that the roadmap reward pool has envisioned is broken into two parts as the following.

### Part 1: Distribution

Distribution of the rewards consists of many moving pieces. It gets triggered when a storage provider mined a block. Reward pool would then resort to on-chain data to derive how much exactly each participating storage provider's fair share is and update the off-chain records. Note that 75% of the block reward will be vesting over 180 days linearly whereas 25% will be immediate. The following sequence diagram shows a high-level view of how a mined block gets translated to the balance of each participating storage provider.

![image-20211108135058418](https://i.loli.net/2021/11/08/1ZHGvEozwi94Fhx.png)

### Part 1: Pooling

As reward pool has access to the shared owner address of all participating storage provider, it will withdraw an amount from each storage provider that corresponds to the immediate and vesting rewards storage provider earns that day everyday. For example, if a storage provider mined a block today, then at midnight of that day reward pool will withdraw the immediate reward of that block plus the amount released from vesting (from previously mined blocks) if present. The following diagram shows a high-level sequence diagram of pooling.

![image-20211108143648376](https://i.loli.net/2021/11/08/I1ZtS3TGoVxQ4BF.png)

### Part 2: fvm

With future introduction of fvm in 2022, all pooling and distribution logic could potentially be written into a smart contract. Everyone can audit the contract or even create their own version of the reward pool. Eventually, we hope that joining a reward pool would mean simply invoking a smart contract, saving the need of operation overheads in changing owner and withdraw rewards from the pool. Very much like what [@raulk](https://github.com/raulk) has described in this [comment](https://github.com/filecoin-project/FIPs/issues/113#issuecomment-955205874).

![image-20211108165637883](https://i.loli.net/2021/11/08/2xZrQsJRfbjOhuw.png)

## Design ideas

More details will be elaborated on in this section and all are subject to further tuning.

### Distribution

- Minimum of 10TiB of raw power to be eligible for rewards;
- Each storage provider's power share in a reward pool is calculated using the power data at the height where the block is mined;
- For example, if a block is mined at height 88888, then a storage provider share of the reward would equal to its power at height 88888 divided by total power of participating storage providers at height 88888;
- Token economy works same as Filecoin where 25% is immediate and 75% will be vesting linearly over 180 days;
- Off-chain DB will hold a record of all blocks mined by participating storage providers including but not limited to information like block CID, block producer, epoch, block reward and win tickets, and a record of participating storage providers associated with the mined blocks including but not limited to information like miner ids, power, power of the reward pool;

### Pooling

- As stated above, token economy works same as Filecoin which means the amount need to be pooled everyday from a participating storage provider consist of both the immediate reward from mined block(s) and vesting reward gained that day;
- If the withdraw fail to meet the amount dictated by the token economy backed by on-chain data, the storage provider will be deducted the difference and be potentially in-debt;

### Debt

- One in debt, reward pool will actively check if storage provider has repaid the debt;
- Once debt is repaid, storage provider will still be able to reclaim the rewards they won during the time they are debted;

### Fairness

- When a storage provider fail to produce a block when it's eligible to do so, it would be responsible for the missing rewards;
- If failure to produce is due to errors of chain service, then the admin of the reward pool (also the operator of the chain service) is responsible for the missing rewards;
- Finally if failure to produce is due to force majeure, then the block is forfeited and the rewards will not be pooled;
- When a storage provider is to exit a reward pool:
  * If no block produced, it can exit but no withdraw can be made;
  * If block produced, it has to pool all the remaining vesting amount before it can withdraw and exit;
- A fee (small percentage of rewards) to be collected to cover operation cost of a reward pool;

### Reward flow

From storage provider's perspective

[SP] Change owner address and join a reward pool;

[SP] Mined a block;

[RP] Pool the reward from mined block and potential vesting rewards (daily);

[SP] Withdraw rewards on demand;

### Dependencies

1. `venus` module to provide node services;

2. `venus-messager` module to provide data services;

3. `venus-gateway` to provide signature services;

![模块图](https://i.loli.net/2021/09/08/7UxfVujcNPmszyR.jpg)

## Risks

- Granularity of the reward pooling is a day, which may or may not be ideal.
- Community's concerns that reward pooling needs a [killer logic](https://github.com/filecoin-project/lotus/discussions/4005#discussioncomment-486520).
- How storage provider can build trust toward the operators of the reward pool.
- How to withdraw collaterals released from expired sectors.
- Cost of pooling which involves gas of withdraw messages everyday.

