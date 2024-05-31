# RGB++ Script Standard

Authors: Cipher Wang, JJY

Contributors: CyberOrange, Ian, Jan

# Overview

## Requirements and Limitations on Isomorphic Binding

Isomorphic Binding requires that RGB++ transactions must be submitted on BTC chain, and that the user use single-use seals on BTC to describe the operation on RGB++ cells. The user needs to construct the CKB raw tx and the RGB++ commitment first, then submit the commitment to BTC, and finally send the CKB TX on-chain. 

However, there are some constraints:

- The BTC UTXO information that represents ownership must be specified in the `lock` of the RGB++ Cell (`btc_tx` + `index`).
- The RGB++ Cell in `ckb_tx` depends on `btc_tx`; if `btc_tx.commitment` includes `ckb_tx`, it results in a deadlock. Thus, **the commitment can only contain these following information of the CKB transaction**:
    - Commitment includes only the first N Inputs and Outputs;
    - Commitment must cover all Inputs and Outputs where `Type` is not null;
    - After the initial N Inputs and Outputs, the CKB transaction can include additional Inputs and Outputs with a null `Type`. This rule allows for modifications to the transaction fee.
- Cell is created without lock script validation, allowing anyone to create an RGB++ Cell using any BTC UTXO as `LockArgs`. This process essencially transfers ownership of the Cell to a BTC UTXO. Also, the operation of this type of Cell follows the same principles outlined above.

# Contract Requirements

The following contract is required:

- `RGBPP_lock`: this is designed to handle the unlocking process of the RGB++ Cell;
- `BTC_TIME_lock`: this is a time lock used to secure a specified number of blocks when assets leap from Layer 1 (L1) to Layer 2 (L2).

## **Config Cell** of the Contract

Both the `RGBPP_lock` and `BTC_TIME_lock` require the [BTC light client](https://github.com/ckb-cell/ckb-bitcoin-spv-contracts/blob/master/contracts/ckb-bitcoin-spv-type-lock/README.md) data, necessitating the storage of the associated contract's `type_hash`. To avoid hardcoded dependencies, the concept of Config Cell is introduced here to address this configuration issue.

When deploying a contract, it is required that both the contract output and the Config Cell output be included within the same transaction.

```
# BTC_TIME_lock
inputs: any cells
outputs:
  BTC_TIME_lock code cell
  time_lock_config cell
...

# RGBPP_lock
inputs: any cells
outputs:
  RGBPP_lock code cell
  rgb_lock_config cell
...
```

Follow these steps to identify Config Cell in the contract:

```
1. Use load_script to retrieve the type_hash of the current contract;
2. Use type_hash to identify the index that its cell dep matches and has out_point.index == 1;
3. Load the data from this cell dep to fetch global configuration settings.
```

```
struct RGBPPConfig {
  version: Uint16,
  // Type hash of bitcoin light client
  bitcoin_lc_type_hash: Byte32,
  // Type hash of bitcoin time lock contract
  bitcoin_time_lock_type_hash: Byte32,
}
```

Whenever the contract is updated, the Config Cell must be updated as well, and these updates must adhere to predefined rules.

## Data Structure of the Contract

### **RGBPP_lock**

```
RGBPP_lock:
  code_hash:
    RGBPP_lock
  args:
    out_index | %bitcoin_tx%
```

- `RGBPP_lock`:
    - out_index: UTXO index, the ownership of the Cell belongs to this UTXO
    - bitcoin_tx: BTC txid

### **BTC_TIME_lock**

```
BTC_TIME_lock:
  args: lock_script | after | %new_bitcoin_tx%
```

- `BTC_TIME_lock`:
    - refers to the owner of Cell once `lock_script` is unlocked;
    - `after` requires that `new_bitcoin_tx` can only be unlocked after it has received more than the number of confirmations specified by the `after` .

## **RGBPP_Lock Unlock Logic**

💡 For Cell of  the RGB++ Asset on L1 Address (`btc_utxo`)

### **Cell Unlock Validation Process**

![image](docs/assets/lock-verify.png)

The process for unlocking involves providing a `btc_tx` with the RGB++ commitment:

- It is included within the BTC light client on CKB.
- Inputs should include the BTC UTXO Input corresponding to the cell.lock to be unlocked, i.e., `btc_tx.inputs[i] == previous_bitcoin_tx | out_index`.
- Outputs must contain only one `OP_RETURN`, which include `commitment`.
- `self.lockargs.%new_bitcoin_tx% = btc_tx`.

The `commitment` is created using the `double sha256("RGB++" | messages)` method, and should satisfy the following rules :

- `version: u16` should always be 0;
- `inputs_len: u8`:
    - Specifies the commitment includes the first n input cells;
    - Must be >= 1;
    - All input cells with a non-null type must be included within `inputs_len` ;
- `outputs_len: u8`:
    - Specifies the commitment includes the first n output cells;
    - Must be >= 1;
    - All output cells with a non-null type must be included within `outputs_len` ;
- `CKB_TX.inputs[:inputs_len]` ;
- `CKB_TX.outputs_sub[:outputs_len]`includes all data except for:
    - `lockargs.%new_bitcoin_tx% = btc_tx` .

The remaining assets in the transaction is secured by the RGB++ Lock. All output cells that have a non-null type must use one of the following locks:

- `RGBPP_lock`
- `BTC_TIME_lock`, which requires:
    - `lockargs.after ≥ 6` ;
    - `lockargs.new_bitcoin_tx == btc_tx`.

Tips:

- `inputs_len` and `outputs_len` can be calculated by the position of the last input or output cell with a `type` ;
- The commitment must include at least one input, even if all input `type` are null;
- SDKs can alter cells outside of the commitment to adjust the transaction fee.

## **BTC_TIME_lock Unlock Logic**

```
lock.args: lock_hash | after | %new_bitcoin_tx%
```

- The `lock_script` identifies the recipient who will receive the assets after the lock is unlocked:
    - For each `BTC_TIME_Lock` input in a transaction, there must be a corresponding output at the same index;
    - The lock of the output must be `lock_script`, with the other fields such as `type`, `data`, and `capacity` must be identical to those in the input.
- `after` requires that the `new_bitcoin_tx` has more than the number of confirmations specified by the `after` ;
- after unlocking, the `lock` of the cell holder must match the `lock_script`.

# **Transaction Logic**

## **L1 Transfers/Operations**

**Definition**: `RGBPP_lock` refers to input and output asset cells on CKB where `type` ≠ null.

```
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
      code: RGBPP_lock
      args: btc_utxo_1 = (out_index | previous_btc_tx)

output:
  xudt:
    type: xudt
    lock:
      code: RGBPP_lock
      args: out_index = 1 | %new_bitcoin_tx%

  xudt:
    type: xudt
    lock:
      code: RGBPP_lock
      args: out_index = 2 | %new_bitcoin_tx%
```

## L1 → L2 Leap

**Definition**: On CKB, the lock for input asset cell is`RGB_lock`. For output assets cell, the lock configuration includes at least one, or possibly all, being `BTC_TIME_lock`; while the remaining output asset cells use`RGBPP_lock`.

 `BTC_TIME_lock`, as a new type of timelock on CKB, is introduced here.

```
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
      code: RGBPP_lock
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
      code: RGBPP_lock
      args: out_index=1 | %new_bitcoin_tx%
```

After enough BTC blocks have been confirmed, the `BTC_TIME_lock` cells can be unlocked. 

> Note: Each output corresponding to a `BTC_TIME_lock` input must have an unlock cell where all fields except for the lock status should remain unchanged.
> 

```
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

## **L2 → L1 Leap**

**Definition**: Input side has no `RGB_lock`, while output side has `RGBPP_lock`.

```
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

# **RGB++ Asset Issuance**

## Bitcoin L1 Issuance of RGB++ Assets

To issue RGB++ assets using Bitcoin's Layer 1, the issuer must initiate a bitcoin transaction that uses UTXO or other id as an identifier. This method enables the implementation of CSV on Layer 1, eliminating the need for Layer 2 involvement. There are various issuance methods, the following sections will discuss two primary issuance methods: direct issuance and inter-block issuance.

### **Direct Issuance**

An issuer needs to create a specific UTXO as the cell of a lock. This step bypasses isomorphic binding, allowing the cell to be used for a one-time issuance.

```
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
    RGBPP_lock:
    args: btc_utxo#0

output:
  xudt_cell:
    data: amount
    type:
      code: xudt
      args: hash(RGBPP_lock|btc_utxo#0)
    lock:
      code: RGBPP_lock
      args: btc_utxo#1
```

### **Inter-block issuance**

Inter-block issuance involves altering the issuance mode of an extensible User-Defined Token ([xUDT](https://talk.nervos.org/t/rfc-extensible-udt/5337)) from `lock` to `type`. This means creating a new xUDT or plugin so that the type.args of the issued xUDT, specifically the asset ID is lockhash but instead the parameter of some BTC-based chain.

```
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
    RGBPP_lock:
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
      code: RGBPP_lock
      args: btc_utxo#1
```

In the above example, for transactions initiated within the [`start_block, end_block`] interval, anyone can executes a fair launch on BTC L1, ensuring that everyone has equal chance of token distribution.

## **Leap to L1 Following L2 Issuance**

This process is straightforward and offers greater flexibility, hence it will not be discussed in this document.
