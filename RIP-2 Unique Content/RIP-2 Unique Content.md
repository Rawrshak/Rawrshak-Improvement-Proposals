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

Content creators can choose to launch their own instance of the Unique Content contract to sell their products, but this poses a financial barrier to entry for users who simply want to individualize their own assets. These users can instead access the global Rawrshak unique content contract for personal use.

### Proposed Technical Design

#### UniqueContent.sol-
-   Interface implementations:
    -	Implements ERC721 so that the contract will be compatible with dApps like Rawrshak’s own exchange and other marketplaces on Ethereum.
    -	Implements ERC1155Receiver which gives the contract the ability to receive the original assets required to mint.
    -	Implements ERC2981 so that other exchanges can query the original developer's receiver address and rate when sending payment transfers for royalties.
    -	Implements IMultipleRoyalties interface so that the Rawrshak exchange can properly pay out all royalty receivers.

-   Mappings
    -	Stores a mapping of (uint tokenId => uint uniqueAssetId)
    -	Stores a mapping of (uint tokenId => address)

    -   Stores a mapping of (uint uniqueAssetId => UniqueAsset)
        -   UniqueAsset struct is made up of
            -   address contentAddress
            -   mapping (uint version => string uri) uniqueAssetUri
                -   or this could be a string, if the uri is permanent
            -   bool idExists
            -   bool creatorLocked

Functions in UniqueContent.sol include:
-	Mint function to mint an asset to a user. This function takes in uniqueAssetData as a parameter.
    -   The uniqueAssetData struct is made up of:
        -	address contentAddress
        -   uint tokenId (tokenId of the original item)
        -	uint uniqueAssetId
        -	string uniqueAssetUri
        -	address[] receiverAddresses
        -	uint[] royaltyRates
        -   boolean creatorLocked
    -	Checks whether the caller has the item in their possession.
    -	Receives the original item.
    -   Mints unique item.
    -	Checks whether the uniqueAssetId exists.
        -   If the uniqueAssetId already exists, then there's no need to update certain mappings.
    -   Updates mappings
-	Burn function to burn an asset.
    -	Checks whether the asset is creator locked.
    -	Checks whether the user has the unique item.
    -	Transfers the original item back to the user.
    -	Burns unique item.
-	A function to retrieve the uri of the original item.
-   A royaltyInfo function which will point to the royaltyInfo function of the original item's content contract.
-	A getRoyalties function which returns an array of receiver addresses and an array of royalty amounts calculated from the given salesprice and royalty rates.
-   For uniqueAssetUri, we can go about it two ways:
    -   The uri is permanent, and we will have a function to query a single uniqueAssetUri
    -   The uri is updateable, and there will be a function to update the uri and a function to query a specific version of the asset.

#### MultipleRoyalties-
This is an abstract contract inherited by UniqueContent.sol.
-	Stores a mapping of (uniqueAssetId =>  LibRoyalty.Fee[]) tokenRoyalties
Functions in MultipleRoyalties.sol include:
-	A function to update the additional royalties.
    -	Checks whether the sum of all royalties exceed 1e6. 
-	A function to return the royalties of a uniqueAssetId.

### Updating Subgraph
Todo: Discuss subgraph changes that must be made.

### Exchange Updates
-   In RoyaltyManager.sol and its interface, a new function, similar to payableRoyaltyies() (such as payableMultipleRoyalties()), must be created which will return arrays of addresses and fees instead of just one of each. This so that the exchange can send transfer payments to more than one royalty receiver.
-   In Exchange.sol, a conditional must be added in functions fillBuyOrder() and fillSellOrder() which checks whether the content contract implements the IMultipleRoyalties interface. This will decide whether it will run the original loop function with payableRoyalties or the new loop that must be added, which uses payableMultipleRoyalties() instead.

## Rationale

## Backwards Compatibility

## Test Cases

## Reference Implementation

## Security Considerations

## Copyright
