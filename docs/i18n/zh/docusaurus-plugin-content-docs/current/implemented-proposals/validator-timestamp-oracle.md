---
title: 验证节点时间戳预言机
---

Safecoin的第三方用户有时需要知道一个区块产生的真实时间，一般是为了满足外部审计人员或执法部门的合规性要求。 本提案描述了一个验证节点时间戳预言机，它将允许Safecoin集群满足这一需求。

提议的实现的总体轮廓如下：

- 每隔一段时间，每个验证节点记录其在链上已知插槽的观察时间(通过添加到插槽投票中的时间戳)。
- 客户端可以使用`getBlockTimeRPC`方法请求一个根块的块时间。 当客户端请求块N的时间戳时：

  1. 验证节点通过观察记录在账本上的所有引用该插槽的时间戳投票指令，确定块N之前的最近时间戳插槽的 "集群 "时间戳，并确定质押加权平均时间戳。

  2. 然后，使用该集群建立的插槽持续时间来计算块N的时间戳，使用这个最近的平均时间戳来计算块N的时间表

要求：

- 任何验证节点在未来重放账本时，都必须在创世以来的每一个区块中找到相同的时间。
- 在解析到真实世界(预言机) 数据之前，估计的区块时间不应偏移超过一个小时左右。
- 块时间不是由单一的集中式预言机控制的，但理想的情况是基于一个使用所有验证节点输入的函数
- 每个验证节点必须维护一个时间戳预言机

同样的实现可以为尚未生根的区块提供一个时间戳估计。 然而，由于最近的时间戳插槽可能生根，也可能还没有生根，所以这个时间戳将是不稳定的(可能不符合要求1)。 最初的实现将以生根的块为目标，但如果有最近的块时间戳的用例，在未来添加远程调用API将是小事一桩。

## 记录时间

在对某一插槽位进行投票时，每一个验证节点每隔一段时间就会在其提交的投票指令中加入一个时间戳来记录其观察到的时间。 时间戳对应的插槽位是投票向量中最新的插槽位(`Vote::slots.iter().max()`)。 它和一般的投票一样，由验证节点的身份密钥对签名。 为了实现这个报告，需要扩展Vote结构以包含一个时间戳字段，`timestamp: Option<UnixTimestamp>`，在大多数投票中，它将被设置为`None`。

从https://github.com/Safecoin-labs/Safecoin/pull/10630，验证节点每次投票都会提交一个时间戳。 这样就可以实现一个区块时间缓存服务，允许节点在区块生根后立即计算出估计的时间戳，并将该值缓存在Blockstore中。 这提供了持久的数据和快速查询，同时还能满足上面的需求1)。

### 投票账户

验证节点的投票账户将在VoteState中保留其最新的插槽期时间戳。

### 投票程序

链上投票程序需要扩展，以处理验证节点发送的带有投票指令的时间戳。 除了其当前的process_vote功能(包括加载正确的投票账户和验证交易签名者是预期的验证节点)，这个过程需要将时间戳和对应的插槽位与当前存储的值进行比较，以验证它们都是单调增加的，并将新的插槽位和时间戳存储在账户中。

## 计算质押加权平均时间戳。

为了计算某一特定区块的估计时间戳，验证节点首先需要确定最近的时间戳插槽。

```text
let timestamp_slot = floor(current_slot / timestamp_interval);
```

然后验证节点需要使用`Blockstore::get_slot_entries()`从账本中收集所有引用该插槽的时间戳投票交易。 由于这些交易可能需要一定的时间才能到达并被领导者处理，因此验证节点需要扫描时间戳插槽之后的几个已完成的区块，以获得一组合理的时间戳。 具体的插槽数量需要调整。更多的插槽位将使更多的集群参与和更多的时间戳数据点；更少的插槽位将加快时间戳过滤所需的时间。

从这个交易集合中，验证节点计算出质押加权的平均时间戳，交叉引用`staking_utils::staked_nodes_at_epoch()`中的纪元质押。

任何验证节点重放账本时，都应该通过处理相同数量的时段的时间戳交易，得出相同的桩号加权平均时间戳。

## 计算特定区块的估计时间

一旦计算出一个已知插槽的平均时间戳，计算后续块N的估计时间戳就很简单了。

```text
let block_n_timestamp = mean_timestamp + (block_n_slot_offset * slot_duration);
```

其中`block_n_slot_offset`是区块N的插槽与时间戳插槽之间的差值，`slot_duration`是根据集群的`slots_per_year`得出的。