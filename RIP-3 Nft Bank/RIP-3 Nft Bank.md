---
rip: 3
title: Nft Bank
author:  @butterfiolee
discussions-to: <URL>
status: Draft
created: 2022-03-21
---

## Abstract
This is a way for users to safely store their NFT's in a secure smart contract wallet outside of their main wallet.

## Motivation
Players over time will accumulate a large amount of tokens through the collection of gaming assets as well as other general non-fungible tokens such as art, music, PFPs, etc. Though wallets can store as many assets as a user needs, their existing user interfaces aren't designed to adequately organize large volumes of tokens. Consequently, they eventually become too cumbersome to maintain. Furthermore, wallet security is not very user-friendly. Storing valuable NFT's in a hot wallet without backup account access or locking features is not conducive to secure asset storage.

To solve this, an NFT Bank is necessary so that users may be able to securely store and unload assets currently not in use, thus keeping their main wallet manageable.

## Specification
The Bank contracts will be able to handle the storage of both ERC1155 and ERC721 assets.

Since this bank is a smart contract, which by its own nature is quite rigid, it utilizes measures to ensure that if one's private keys are lost, stored assets may still be retrievable. This is achieved by setting backup addresses that will have access to the accounts if the main wallet has not interacted with the contracts for a given length of time.

In pursuit of decentralization, each user will launch their own personal banking contracts which they will have full control over, thus avoiding a central point of failure.

### Proposed Technical Design

#### LibBank.sol-
Structure object:
-   AssetData
    -   address contract address
    -   uint256 tokenId
#### Bank.sol-
Contract Interfaces Used:
-   IERC721Upgradeable and IERC1155Upgradeable which allows the contract to make transfer calls.
-   ERC165CheckerUpgradeable which gives the contract the ability to check supported interfaces.

Inherits:
-   ERC1155HolderUpgradeable and ERC721HolderUpgradeable which gives the contract the ability to receive and store NFT's.
-   AccessControlUpgradeable which allows the contract to grant account access to multiple wallets, specifically backup and alternative wallets.
-   PausableUpgradeable which provides the user the functionality of disabling withdrawals on their smart contract but still maintain the ability to deposit items.
-   IBank and IBankStorage interfaces.

Global variables:
-   uint64 lastAccessed - blockNumber of the last time an account was accessed.
-   bytes32 - password for locking the account, cannot be queried.
-   uint64 maxSecurityTimeout- a blockNumber which represents the length of time required before a backup address can request permission to access the account.
-   uint64 lastPaused - blockNumber of when the contract's withdrawal function has been frozen.
-   uint256 amountTimeUnlocked- blockNumber which represents the amount of time before an account is locked.
-   address[] backupAddresses- backUp accounts in case the main account's private keys are lost.

Functions:
-   depositBatch function to deposit multiple assets. This function takes in an array of assetData structs, an array of uint256 deposit amounts and an address.
    -   calls the deposit function in BankStorage.sol to update the itemBalances mapping.
    -   calls a transfer function to send the item to Bank.sol.
-   withdrawBatch function to withdraw multiple assets. This function takes in an array of assetData structs, an array of uint256 withdraw amounts and an address.
    -   calls the withdraw function in BankStorage.sol to update the itemBalances mapping.
    -   calls a transfer function to retrieve the item from the bank to the user.
-   rescueToken function, which takes in a contract address and token id, allows the user to retrieve tokens which were transferred to the bank contract without using depositBatch. It reconciles the mapping in BankStorage.sol with the token amount in possession by the Bank contract.
-   addBackup function to add a back up address in case one loses their keys.
-   removeBackup function to remove an address from the backupAddresses array.
-   setPassword function which takes in strings of the old password and new password. If the old password is confirmed, it hashes the new password and sets it as the account password.
-   lock function, which requires a password, to set the value of the variable amountTimeUnlocked.
-   getAmount function to call the getAmount function in BankStorage.sol
-   hashPassword internal function which creates a hash of the given password string.

#### BankStorage.sol
Inherits:
-    OwnableUpgradeable which guarantees that only the Bank contract can call its functions.

Mappings:
-   mapping(address contractAddress => mapping(uint256 tokenId => uint256 tokenAmount)) itemBalances

Functions:
-   withdraw function which decreases the value of tokenAmount in the itemBalances mapping.
-   deposit function which increases the value of tokenAmount in the itemBalances mapping.
-   getAmount function to query the amount of an item stored in the Bank by the caller.

## Rationale