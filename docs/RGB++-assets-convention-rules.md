> This document is a repost of  [_xUDT Information Convention RFC_](https://talk.nervos.org/t/xudt-information-convention-rfc/8030)  post on NervosTalk forum, 
with a few refinements for greater consistency and readability.

## Background

xUDT (RGB++) assets on CKB similar to ERC20 assets on Ethereum in that they lack protocol-level uniqueness constraints for token names. 
Considering the conventions of the Bitcoin ecosystem, Considering the conventions of the Bitcoin ecosystem, it is necessary to introduce 
standards regarding uniqueness and length.


## Scope
- The following conventions cover **inscription info cell** and **unique cell**, combining the two for processing.
  For example, each asset has a unique name among them.
- These conventions consider only the `Symbol` and do not consider the `Name` field.

```yaml
# Unique cell
{
  "code_hash": "0x2c8c11c985da60b0a330c61a85507416d6382c130ba67f0c47ab071e00aec628",
  "hash_type": "data1",
  "out_point": {
    "tx_hash": "0x67524c01c0cb5492e499c7c7e406f2f9d823e162d6b0cf432eacde0c9808c2ad",
    "index": "0x0"
  },
  "dep_type": "code"
}
```

## Name Conflict
- Only the first occurrence of an asset name on-chain is acknowledged; subsequent occurrences are marked as “duplicate”.
- Case sensitivity is ignored during duplicate comparison, for example: “Seal” is considered the same as “SEAL”.
- Any addition of invisible characters (including spaces) to the name, regardless of whether there are similar assets,
  is treated as a “**duplicate**”.
- For wallets and exchanges, duplicate assets are directly hidden.
- For browsers:
  - “duplicate assets” are labeled as “Risk Asset - RGB++ incompatible”;
  - non-duplicate assets are labeled as “RGB++ Compatible”;
  - In places where the aforementioned text is too long to display, symbols such as [!] and [+] are used for marking.


## Letters
- For exchanges, only the ASCII visible character subset is supported.
- For wallets and browsers, the supported character range includes:
  - ASCII visible characters subset
  - A subset of emojis: to be determined
- For assets that use unsupported characters, the asset name is uniformly displayed as “INVALID”.

## Length Range
- Only assets with Symbol length of 4-5 characters are processed normally.
- Assets falling outside this range are treated as “duplicate assets”.

## Verification
- Whitelisted assets are set to bypass the rules mentioned above.
- To get whiltelist, To get on the whitelist, project owners should submit their token information to Cell Studio’s GitHub repository.
  After verification, the token information will be reviewed and released. Assets on the whitelist are given priority for display
  regardless of the above rule, and any other assets with the same name are automatically labeled as "duplicate assets".
- As of now, the whitelist submission is not open yet. 


## Assets Mark
Ecosystem apps should add label / mark for every asset to provide some critical information:
- Issuance mark: `L1 issue` / `L2 issue` to identify where the coins are issued, on Bitcoin with `rpgpp_lock` or CKB with normal lock;
- Supply mark: `Limited supply` / `Unlimited supply` to identify whether a coin could be issued arbitrarily.

  > Note: The [issue For Support tags of xudt #620](https://github.com/Magickbase/ckb-explorer-public-issues/issues/620) defines the
  existing tags and display of xUDTs on the CKB Explorer.
