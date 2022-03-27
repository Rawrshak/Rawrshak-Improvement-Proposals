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
Players over time will accumulate a large amount of tokens through the collection of gaming assets as well as other general non-fungible tokens such as art. Though wallets can store as many assets as a user wants, their existing user interfaces aren't designed to adequately organize large volumes of tokens. Consequently, they eventually become too cumbersome to maintain. Furthermore, wallet security is not very user-friendly. Storing valuable NFT's in a hot wallet without backup account access or locking features is not conducive to secure asset storage.

To solve this, an NFT Bank is necessary so that users may be able to securely store and unload assets currently not in use, thus keeping their main wallet manageable.

## Specification
The Bank contracts will be able to handle the storage of both ERC1155 and ERC721 assets.

Since this bank is a smart contract, which by its own nature is quite rigid, it utilizes measures to ensure that if one's private keys are lost, stored assets may still be retrievable. This is achieved by setting backup addresses that will have access to the accounts if the main wallet has not interacted with the contracts for a given length of time.

To pursue decentralization, each user will launch their own personal banking contracts which they will have full control over, thus avoiding a central point of failure.

### Proposed Technical Design

#### LibBank.sol-
Structure object:
-   AssetData
    -   address contract address
    -   uint256 tokenId
-   SecurityInfo
    -   uint64 lastAccessed - blockNumber of the last time an account was accessed.
    -   bool accountLocked - whether an account is locked or not
    -   bytes32 - password for locking the account
    -   address[] backupAddresses- backUp accounts in case the main account's private keys are lost
#### Bank.sol-
Imports:
-   IERC721Upgradeable and IERC1155Upgradeable which allows the contract to make transfer calls.
-   ERC165CheckerUpgradeable which gives the contract the ability to check supported interfaces.

Inherits:
-   ERC1155HolderUpgradeable and ERC721HolderUpgradeable which gives the contract the ability to receive and store NFT's.
-   AccessControlUpgradeable which allows the contract to grant account access to backup addresses.
-   PausableUpgradeable which provides the functionality of locking functions in case a bug is discovered.
-   IBank and IBankStorage interfaces.

Global variables:
-   (address => LibBank.SecurityInfo) accountInfo
-   uint64 maxSecurityTimeout- a blockNumber which represents the length of time required before a backup address will have access to an account
-   uint64 lastPaused - blockNumber of when the contracts have been frozen

Functions:

Note: All mutative functions will have a modifier that requires that either msg.sender == the affected account's address or that the current blockNumber - lastAccessed variable is over a certain time frame and that msg.sender == one of the backup addresses. And all mutative functions will reset the lastAccessed blockNumber of an address to the current blockNumber, but not if the functions were accessed by the backup addresses.

-   depositBatch function to deposit multiple assets. This function takes in an array of assetData structs, an array of uint256 deposit amounts and an address.
    -   checks if the user possesses the amount of tokens they want to deposit.
    -   calls the deposit function in BankStorage.sol to update the storedAmounts mapping.
    -   calls a transfer function to send the item to Bank.sol.
-   withdrawBatch function to withdraw multiple assets. This function takes in an array of assetData structs, an array of uint256 withdraw amounts and an address.
    -   requires that stored amount of the item is greater than or equal to the amount to withdraw.
    -   calls the withdraw function in BankStorage.sol to update the storedAmounts mapping.
    -   calls a transfer function to retrieve the item from the bank.
-   addBackup function to add a back up address in case one loses their keys.
-   removeBackup function to remove an address from the backupAddresses array.
-   setPassword function which sets an account password required to lock and unlock an account.
-   lock function which requires a password and signature to set the accountLocked boolean to true or false.
-   tokenAmount function to call the tokenAmount function in BankStorage.sol
-   hashAssetId internal function which creates a hash of the item's contract address and token Id.

#### BankStorage.sol
Mappings:
-   (bytes4 assetIdHash => uint256 tokenAmount) storedAmounts 
-   (address user => storedAmounts) assetInfo

Functions:
-   withdraw function which decreases the value of tokenAmount in the storedAmounts mapping.
-   deposit function which increases the value of tokenAmount in the storedAmounts mapping.
-   tokenAmount function to query the amount of an item stored in the Bank by the caller.

## Rationale
Design Considerations:
-   In BankStorage.sol, because storedAmounts, which basically is the bank account, is tied to an account address, backup addresses will not be able to obtain true admin priveleges. To remedy this, it is possible that instead of mapping addresses to storedAmounts, we can map uint256 accountNumbers to the storedAmounts mapping. And in Bank.sol, we can add a createAccount function, which initializes the SecurityInfo struct of an accountNumber with an adminAddress variable. So if a user's private keys are lost, instead of checking whether msg.sender is equal to one of the backup addresses, we can set one of the backup addresses as the new admin address of the account.

