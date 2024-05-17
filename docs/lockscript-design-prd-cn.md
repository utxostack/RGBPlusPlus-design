# RGB++ 合约规范

Authors: Cipher Wang, JJY

Contributors: CyberOrange, Ian, Jan

# 概述

## 关于同构绑定的要求和限制

同构绑定本质上要求 RGB++ 相关的每一个 BTC UTXO 和 CKB CELL 对应。鉴于交易的流程是先构造一个 CKB raw tx，然后把它的部分信息作为 commitment 放到 BTC TX 中，最后再将 CKB TX 上链。这里面的一些约束条件有：

- CKB Cell 的 lock 需要有完整的 BTC UTXO 信息（btc_tx + index）
- ckb_tx 的计算依赖所有的 cell 信息，后者依赖 btc_tx，如果 btc_tx.commitment 包含 ckb_tx，这就死锁了，因此，**commitment 只能包含 ckb tx 的部分信息**
    - 为了方便 ckb tx 更新手续费重发，commitment 中不应该包含 output[].capacity
- Cell 的一个额外问题是，创建 cell 时，lock 脚本并不执行，因此如果没有额外的限制或约束，任何人都可以创建多个使用相同 btc_utxo 作为 lockargs 的 cell，形式上与同一个 btc utxo 绑定
    - 但似乎多个 cell 于一个 btc_utxo 对应不会引入安全问题，因此我们允许这种情况出现

# 合约需求

需要如下合约，

- RGB_lock 用来处理与 BTC TX 的同构映射的 CELL 的解锁
- BTC_TIME_lock，用来当资产从 L1 jump 到 L2 时进行锁定一定区块数再允许用户操作

## 合约的 Config Cell

RGB_lock / BTC_TIME_lock 合约需要读取轻节点 cells，因此我们必须保存相关合约的 type_hash。由于不希望引入硬编码的合约依赖，我们引入 Config Cell 的概念来解决此类配置问题。


部署合约时要求合约输出以及 config cell 输出都在同一笔交易内完成部署。

```yaml
# BTC_TIME_lock
inputs: any cells
outputs:
  BTC_TIME_lock code cell
  time_lock_config cell
...

# RGB_lock
inputs: any cells
outputs:
  RGB_lock code cell
  rgb_lock_config cell
...
```
合约通过以下方式找到 config cell

```yaml
1. load_script 找到目前的 合约的 type_hash
2. 通过 type_hash 找到 cell dep 符合且 out_point.index == 1 的 cell deps 的 index
3. load 这个 cell dep 的 data 即得到全局配置

全局配置
```

```rust
struct RGBPPConfig {
  # Type hash of bitcoin light client
  bitcoin_lc_type_hash: Byte32,
  # Type hash of bitcoin time lock contract
  bitcoin_time_lock_type_hash: Byte32,
}
```
每次更新合约都必须和 config cell 一起更新，并且遵守更新规则。

## 合约数据结构

### RGB_lock

```yaml
RGB_lock:
  code_hash: 
    RGB_lock
  args:
    out_index | %bitcoin_tx%
```

- RGB_lock:
    - out_index：指定一个可消费自己的 utxo 中的 index
    - bitcoin_tx: 指定一个可消费自己的 utxo 中的 btc_txid，该数值不包含在计算 commitment 内

### BTC_TIME_lock

```yaml
BTC_TIME_lock:
  args: lock_script | after | %new_bitcoin_tx%
```

- BTC Time lock:
    - lock_script 为目标 lockscript
    - after 要求 new_bitcoin_tx 已经超过 after 个确认
    - 解锁后的 cell 持有人的 lock 符合 lock_script

## RGB_Lock 解锁逻辑

<aside>
💡 该 Lock 放在每一个 RGB++ 资产在 CKB 的映射 Cell 上，用于 L1 地址(btc_utxo)持有 RGB++ 资产
</aside>

**Cell 解锁验证流程**

![uib](./assets/lock-verify.png)

- 存在一个与当前 CKB TX 对应的 `btc_tx`，它满足：
    - 包含在 CKB 上的 BTC 轻客户端中
    - inputs 中包含一个与要解锁的 cell.lock 对应的 btc utxo input，即  `btc_tx.inputs[i] == previous_bitcoin_tx | out_index`
    - outputs 中有且仅有一个 OP_RETURN，包含 `commitment`
    - `self.lockargs.%new_bitcoin_tx% = btc_tx`
- 该 `commitment`为以下内容的 hash，算法为 `double sha256(”RGB++” | messages)`
  - `version: u16`，必须为 0
  - `inputs_len:u8`
    - 表示 commitments 包含前 n 个 inputs
    - 必须 >= 1
    - 所有 type 不为空的 input cell 必须被包含在 inputs_len 中
  - `outputs_len:u8`
    - 表示 commitments 包含前 n 个 outputs
    - 必须 >= 1
    - 所有 type 不为空的 output cell 必须被包含在 outputs_len 中
  - `CKB_TX.inputs[:inputs_len]`
  - `CKB_TX.outputs_sub[:outputs_len]`, 包含全部数据，除了
    - 不包含 `lockargs.%new_bitcoin_tx% = btc_tx` 
- 验证 L1 的资产仍然被 RGB++ 逻辑保护，即所有 outputs 中 type 不为空的 cells 必须使用以下两种 lock 之一
  - RGB_lock
  - BTC_TIME_lock
      - 要求 `lockargs.after ≥ 6`
      - 要求 `lockargs.new_bitcoin_tx == btc_tx`

**tips**

- 由最后一个有 type 的 input / output cell 的位置就可以计算出 inputs_len / outputs_len 值
- 如果所有 inputs / outputs 都没有 type 则至少填 `1`，commitment 至少包含一个 input out point
- SDK 增加手续费时可以在 commitment 之外的 cells 里增加，这样即使提交不成功也可以修改这个 cell。

## BTC_TIME_lock 解锁逻辑

```yaml
lock.args: lock_hash | after | %new_bitcoin_tx%
```

- lock_script 为解锁后需要释放到的目标接受者
    - 解锁交易中每个 BTC_TIME_lock input 必须在相同 index 对应一个 output
    - output 的 lock 为 lock_script 其余字段 type, data, capacity 需要和 input 一致
- after 要求 new_bitcoin_tx 已经超过 after 个确认
- 解锁后的 cell 持有人的 lock 符合 lock_script

# 交易逻辑

## L1 转账/操作

**定义：CKB 上输入输出的资产 cell（定义：type ≠ null） 均为 RGB_lock**

```yaml
# BTC_TX
input:
  btc_utxo_1  # =(previous_btc_tx | out_index)
  ...
output:
  OP_RETURN: commitment
  btc_utxo_3  # =(new_bitcoin_tx | out_index)
  btc_utxo_4  # =(new_bitcoin_tx | out_index)

# CKB_TX
input:
  rgb-xudt:
    type:
      code: xudt
      args: <asset-id>
    lock:
      code: RGB_lock
      args: btc_utxo_1 = (out_index | previous_btc_tx)

output:
  xudt:
    type: xudt
    lock:
      code: RGB_lock
      args: out_index = 1 | %new_bitcoin_tx%

  xudt:
    type: xudt
    lock:
      code: RGB_lock
      args: out_index = 2 | %new_bitcoin_tx%
```

## L1 → L2 Jump 操作

**定义：CKB 上输入的资产 cell 的 lock 均为 RGB lock，输出的资产 cell 的 lock 至少一个或全部为 BTC_TIME_lock，其余为 RGB_lock**

这里需要在 CKB 上引入一种新的时间锁 Lock: **BTC_TIME_lock**

```yaml
# BTC_TX
input:
  btc_utxo_1
  ...
output:
  OP_RETURN: commitment
  btc_utxo_3

# CKB_TX
input:
  rgb_xudt:
    type: xudt
    lock:
      code: RGB_lock
      args: out_index | source_tx

output:
  rgb_xudt:
    type: xudt
    lock：
      code: BTC_TIME_lock
      args: lock_script | after | %new_bitcoin_tx%

  rgb_xudt:
    type: xudt
    lock:
      code: RGB_lock
      args: out_index=1 | %new_bitcoin_tx%
```

等到足够多的 BTC 区块确认后，可以解锁 BTC_TIME_lock 的 cell

```yaml
# CKB_TX
input:
  rgb_xudt:
    type: xudt
    lock:
      code: BTC_TIME_lock
      args: lock_script | 6 | btc_tx

output:
  rgb_xudt:
    type: xudt
    lock:
      lock_script

witness:
  # proof of 6 confirmations after #btx_tx
```

## L2 → L1 Jump 操作

**定义：输入侧没有 RGB_lock，输出侧有 RGB_lock**

```yaml
# CKB TX
input:
  xudt:
    type: xudt
    lock:
      ckb_address1

output:
  xudt:
    type: xudt
    lock:
      ckb_address2

  rgb_xudt:
    type: xudt
    lock:    
      args: btc_utxo
```
**注意，这里不需要同构绑定**

双花问题：

- 建议应用方或用户在 CKB 交易 24 块确认后再执行后续的 RGB++ L1 交易
- 否则可能因为 CKB TX 双花，导致上面 rgb_xudt(#btc_utxo) 被替换成其他输出
    - 因此 dapp 设计上，L2→L1 的资产需要等 24 个 ckb 区块才能操作，不过仅限于前端限制

# RGB++ 资产发行

## 纯 L1 方式发行 RGB++ 资产

使用 L1 方式发行 RGB++ 要求发行人使用 bitcoin 上的交易，utxo 或其他 id 作为身份标识符来发行资产，这样才可以做到无须 L2 辅助即可完全实现 CSV。具体发行方案有多种，我们这里列出两种简单方案。

### 直接发行

用户需要首先构造一个使用特定 utxo 做 lock 的 cell，作为发行人。该步骤无须经过同构绑定，后即可用这个 cell 进行一次性发行

```yaml
# BTC TX
input:
  btc_utxo#0
  ...
output:
  commitment
  btc_utxo#1
  ...

# CKB TX
input:
  issue_cell:
    RGB_lock:
    args: btc_utxo#0
      
output:
  xudt_cell:
    data: amount
    type: 
      code: xudt
      args: hash(RGB_lock|btc_utxo#0)
    lock:
      code: RGB_lock
      args: btc_utxo#1
```
### 区块区间发行

区块区间发行需要将 xudt 的发行模式从 lock 发行改为 type 发行，即创建一种新的 xudt，或插件，使得发行的 xudt 的 type.args，即资产 id 不是 lockhash，而是某些 btc 链的参数即可

```yaml
# BTC TX
input:
  btc_utxo#0
  ...
output:
  commitment
  btc_utxo#1
  ...
  
# CKB TX
input:
  issue_cell:
    RGB_lock:
      args: btc_utxo#0

output:
  xudt_cell:
    data: amount
    type: 
      code: xudt_modified
      args: 
        hash_of:
          start_block,
          end_block,
          max_per_tx,
          token_name
    lock:
      code: RGB_lock
      args: btc_utxo#1
```

上面的例子中，在[start_block, end_block] 区间发起的交易，任何人都可以在 BTC L1 上实现公平发射发行，以平等的机会获得代币。

## L2 发行后跳转到 L1

比较简单，也更灵活，不再赘述。