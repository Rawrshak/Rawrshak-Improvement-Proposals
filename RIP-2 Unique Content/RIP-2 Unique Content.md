---
rip: 2
title: Unique Content Feature
author:  @butterfiolee
discussions-to: <URL>
status: Draft
created: 2021-11-23
---

## Abstract
This is a way for users to be able to tailor digital assets and merchandise them to consumers if they choose.

## Motivation
Players are always keen to individualize their characters. And collectors are always in the market for unique and rare items, especially ones related to content creators or brands they support, or items associated with special events and contests.

Unique content will be a way for anyone to customize digital assets to sell to consumers or simply for personal use. This opens the door to a vast amount of collectible assets not limited to what game developers are able to periodically generate.

However, minted unique assets will only always have the functionality of the item they replaced, and so no game-breaking unique assets are ever produced.

## Specification
Users will have the ability to create a unique asset template which specifies which original item it will need to receive and what kind of new item it will return during the minting process. 

They will also be able to add additional royalties to their unique asset on top of the royalty of the original asset which carries over. This allows the creators of the unique asset and the developer of the original asset to be paid in perpetuity every time the asset is resold. Therefore, the original content developers do not have to worry about losing on future royalties when their item is locked into the unique content contract.

Content creators can choose to lock assets such that only the creator of the unique asset can burn the item. This helps ensure that the content creator will not lose out on future royalties.

Content creators can choose to launch their own instance of the Unique Content contract to produce and sell their products, but this poses a financial barrier to entry for users who simply want to individualize their own assets. These users can instead access the global Rawrshak unique content contract for personal use.

### Proposed Technical Design

#### UniqueContent.sol-
-   Interface implementations:
    -	Implements ERC721 so that the contract will be compatible with dApps like Rawrshak’s own exchange and other marketplaces on Ethereum.
    -	Implements ERC1155Receiver which gives the contract the ability to receive the original assets required to mint.
    -	Implements ERC2981 so that other exchanges can query the original developer's receiver address and rate when sending payment transfers for royalties.
    -	Implements IMultipleRoyalties interface so that the Rawrshak exchange can properly pay out all royalty receivers.

-   Mappings
    -   Stores a mapping of (uint256 uniqueId => uniqueAssetInfo)
        -   UniqueAsset struct is made up of
            -   address creator
            -   address contentAddress
            -   uint256 tokenId (tokenId of original asset)
            -   uint256 version (can be omitted)
            -   string[] uniqueAssetUri (can be a singular string)
            -   bool creatorLocked

Functions in UniqueContent.sol include:
-	Mint function to mint an asset to a user. This function takes in uniqueAssetCreateData as a parameter.
    -   The uniqueAssetCreateData struct is made up of:
        -   address to
        -	address contentAddress
        -   uint256 tokenId (tokenId of original asset)
        -	string uniqueAssetUri
        -	address[] receiverAddresses
        -	uint256[] royaltyRates
        -   boolean creatorLocked
    -   Checks whether the sum of the royalty rates exceed 1e6
    -   Checks whether receiverAddresses.length equal royaltyRates.length
    -	Checks whether the caller has the item in their possession.
    -	Receives the original item.
    -   Mints unique item.
    -   Updates mappings.
-	Burn function to burn an asset.
    -   checks whether the person burning the asset is the owner of the unique item.
    -	Checks whether the asset is creator locked.
    -	Transfers the original item back to the user.
    -	Burns unique item.
    -   Deletes token information.
-	An originalAssetUri function to retrieve the uri of the original item.
-   A uri function to query the unique asset's uri.
-   A setUniqueUri function to update the unique asset's uri.
-   A royaltyInfo function which will point to the royaltyInfo function of the original item's content contract.
-	A multipleRoyaltyInfo function which returns an array of receiver addresses and an array of royalty amounts calculated from the given salesprice and royalty rates.
-   A setTokenRoyalties function which updates the royalties of a function.

For uniqueAssetUri, we can go about it two ways:
-   The uri is permanent, and we will have a function to query a single uniqueAssetUri
-   The uri is updateable, and there will be a function to update the uri and a function to query a specific version of the asset.

#### MultipleRoyalties-
This is an abstract contract inherited by UniqueContent.sol.
-	Stores a mapping of (uint256 uniqueId => LibRoyalty.Fee[])

Internal functions in MultipleRoyalties.sol include:
-	A helper function to update the royalties.
-   A verifyRoyalties function which checks whether royalties designated are valid.
-	A getMultipleRoyalties function to return the multiple royalties of a unique asset.

### Updating Subgraph
Todo: Discuss subgraph changes that must be made.

### Exchange Updates
Currently the Rawrshak exchange contracts are only compatible with content contracts which use the ERC1155 token standard and so some changes must be made so that it can handle the exchange of ERC721 unique assets and pay out their corresponding multiple royalties.

-   In RoyaltyManager.sol and its interface, a new function, similar to payableRoyaltyies() (such as payableMultipleRoyalties()), must be created which will return arrays of addresses and fees instead of just one of each. This so that the exchange can send transfer payments to multiple royalty receivers.
-   In Exchange.sol, and its interface, two new functions must be added, similar to fillBuyOrder() and fillSellOrder(), with the difference that they will specifically handle ERC721 NFTs. They will strip away irrelevant modifiers, discard amountToBuy/amounToSell as a parameter, and use payableMultipleRoyalties() to retreive royalties and run a loop of transferRoyalty() to send payments.
-   In NftEscrow.sol, the _transfer() internal function, which exclusively transfers ERC1155 tokens, must be modified to call the transfer function of an ERC721 contract in case the token exchanged is a unique asset (ie. the content contract supports the IMultipleRoyalties interface). This change affects all deposit and withdraws functions on this contract which is called whenever a buy or sell order is executed, or when someone cancels or claims an order.

Design Considerations:
-   Exchange.sol
    -   It is possible modify the original fill order functions to handle ERC721 tokens. It can check whether the content contract implements the IMultipleRoyalties interface. This will decide whether it will run the original loop function with payableRoyalties or simply run payableMultipleRoyalties() once and a loop of transferRoyalty().
    -   However, the benefit of creating new fill order functions is to open the possibility down the line of allowing users to buy multiple unique assets in one go. This idea is incompatible with the original fill order functions which require all order Ids to have the same token information.

## Rationale

## Backwards Compatibility

## Test Cases

## Reference Implementation

## Security Considerations

## Copyright
