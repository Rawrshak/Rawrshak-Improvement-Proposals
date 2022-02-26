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

Optionally, the creator can add additional royalties to their unique asset on top of the royalty of the original asset which carries over. This allows the creators of the unique asset and the developer of the original asset to be paid in perpetuity every time the asset is resold on marketplaces. Therefore, the original content developers do not have to worry about losing on future royalties when their item is locked into the unique content contract.

Content creators can choose to lock assets such that only the creator of the unique asset can burn the item. This helps ensure that the content creator will not lose out on future royalties.

Content creators can choose to launch their own instance of the Unique Content contract to produce and sell their products, but this poses a financial barrier to entry for users who simply want to individualize their own assets. These users can instead access the global Rawrshak unique content contract for personal use.

### Proposed Technical Design

#### UniqueContent.sol-
-   Interface implementations:
    -	Implements ERC721 so that the contract will be compatible with dApps like Rawrshak’s own exchange and other marketplaces on Ethereum.
    -	Implements ERC1155HolderUpgradeable which gives the contract the ability to receive the original assets required to mint.
    -   Implements ERC721HolderUpgradeable for the same reason as the one above.
    -	Implements ERC2981 so that other exchanges can query the original developer's receiver address and rate when sending payment transfers for royalties.
    -	Implements IMultipleRoyalties interface so that the Rawrshak exchange can properly pay out all royalty receivers.
    -   Implements IUniqueContent interface so that it can update and query necessary information about its unique assets.

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
    -   Checks whether the original asset supports IERC2981Upgradeable and IERC1155Upgradeable or ERC721Upgradeable interfaces.
    -   Checks whether the sum of the royalty rates exceed 2e5.
    -   Checks whether receiverAddresses.length equal royaltyRates.length
    -	Receives the original item.
    -   Updates mappings.
    -   Mints unique item to specified address.
    -   Emits Mint Event.
-	Burn function to burn an asset.
    -   Checks whether the person burning the asset is the owner of the unique item.
    -	Checks whether the asset is creator locked or if the caller is the creator.
    -	Burns unique item.
    -   Deletes token information.
    -	Transfers the original item back to the user.
    -   Emits a Burn event.
-	An originalAssetUri function to retrieve the uri of the original item.
-   Two tokenURI functions to query the unique asset's latest uri version or a specific version.
-   A setUniqueUri function to update the unique asset's uri.
-   A royaltyInfo function which retrieves the royalty rate and royaltyAmount of the original asset.
-	A multipleRoyaltyInfo function which returns an array of receiver addresses and an array of royalty amounts calculated from the given salesprice and royalty rates.
-   A setTokenRoyalties function which updates the royalties of a function.

For uniqueAssetUri, we can go about it two ways:
-   The uri is permanent, and we will have a function to query a single uniqueAssetUri
-   The uri is updateable, and there will be a function to update the uri and a function to query a specific version of the asset.

#### UniqueContentStorage.sol-
This contract will store and retrieve the token information of a unique asset.
-   Mappings:
    -   Stores a mapping of (uint256 uniqueId => LibAsset.uniqueAsset) uniqueAssetInfo
        -   UniqueAsset struct is made up of
            -   address creator
            -   address contentAddress
            -   uint256 tokenId (tokenId of original asset)
            -   uint256 version (can be omitted)
            -   string[] uniqueAssetUri (can be a singular string)
            -   bool creatorLocked

Functions in UniqueContent.sol include:
-   A setUniqueAssetInfo function updates the uniqueAssetInfo mapping with the token info of the newly minted unique asset.
-   A burnUniqueAssetInfo function which deletes the unique asset's data from the uniqueAssetInfo mapping.
-   A tokenURI function to retrieve the specific version of a token's uri.
-   A setUniqueUri function pushes a new uri to the uniqueAssetUri string array and increments the token version.
-   A getRoyalty function retrieves the original receiver address and royalty amount.
-   A getMultipleRoyalties function retrieves all royalty receivers and royalty amounts for a unique asset.
-   A setTokenRoyalties function updates a unique asset's token royalties.
-   An isCreator function verifies whether an address matches the creator address of a unique asset.
-   An isLocked function checks whether the a unique asset is creator locked.
-   A getAssetData function retrives the corresponding original token id and content contract address of a unique asset.

#### MultipleRoyalties.sol-
This is an abstract contract inherited by UniqueContentStorage.sol.
-   Mappings:
    -	Stores a mapping of (uint256 uniqueId => address[]) royaltyReceivers
    -   Stores a mapping of (uint256 uniqueId => uint24[]) royaltyRates

Internal functions in MultipleRoyalties.sol include:
-	A setTokenRoyalties helper function to update the royalties.
-	A getMultipleRoyalties helper function to return the multiple royalties of a unique asset.
-   A deleteTokenRoyalties helper function to delete the token royalties of a unique asset.
-   A getTokenRoyaltiesLength helper function to ascertain the size of an array of receivers.

#### LibRoyalty.sol
-   Updates:
    -   A verifyRoyalties function must be added which verifies whether the given royalties are valid to use.

#### UniqueContentFactory.sol-
This contract will create clones of the unique content contracts
-   State Variables:
    -   uniqueContentImplementation
    -   uniqueContentStorageImplementation
    -   contractVersion

Functions in UniqueContentFactory.sol include:
-   An updateContracts function to update the uniqueContent and uniqueContentStorage implementation addresses.
-   A contentExists function to query whether the given proxy contract exists.
-   A createContracts function which creates and initializes clones of the unique content contracts.


### Updating Subgraph
Todo: Discuss subgraph changes that must be made.

### Exchange Updates
Currently the Rawrshak exchange contracts are only compatible with content contracts which use the ERC1155 token standard and so some changes must be made so that it can handle the exchange of ERC721 unique assets and pay out their corresponding multiple royalties.

-   Erc20Escrow.sol:
    -   Two new functions must be added, both called TransferMultipleRoyalties(), each with different sets of parameters. Which one is called depends on whether the order which calls this function is a buy order or sell order. Instead of calling the transferRoyalty() function in Exchange.sol multiple times per unique asset, TransferMultipleRoyalties() will be called once and will handle paying out the receivers.
-   RoyaltyManager.sol:
    -   A new function, called payableMultipleRoyalties()), must be created which will return arrays of addresses and fees instead of just one of each.
    -   This contract will also contain TransferMultipleRoyalties() functions which will call the ones in Erc20Escrow.
-   Exchange.sol:
    -   A new function must be added, similar to fillBuyOrder() and fillSellOrder(), with the difference that they will specifically handle ERC721 NFTs. It will strip away irrelevant modifiers, discard amountToBuy/amountToSell and maxSpend as parameters, and use payableMultipleRoyalties() to retrieve royalties and call TransferMultipleRoyalties() to send payments. 
    -   A new event called UniqueOrdersFilled will also have to be added.
    -   The functions fillBuyOrder() and fillSellOrder() must subsequently be combined into a function called fillOrder() to remain within the contract code size limit in light of necessary additions to exchange contract.
-   ExecutionManager.sol:
    -   A new function called executeUniqueBuyOrder() must be added that can handle depositing more than one type of asset to escrow.
-   In NftEscrow.sol:
    -   the internal _transfer() function, which exclusively transfers ERC1155 tokens, must be modified to call the transfer function of an ERC721 contract in case the token exchanged is an ERC721 NFT. This change affects all deposit and withdraws functions on this contract which is called whenever a buy or sell order is executed, or when someone cancels or claims an order.
- In Orderbook.sol:
    -   A new function called verifyTokenPayment must be added. It would fill a similar role to verifyAllOrdersData() for the fillUniqueOrder() function in Exchange.sol but it will not require that that the orderIds be the same type of asset.
    -   To combine fillBuyOrder() and fillSellOrder() in the Exchange.sol contract, verifyAllOrdersData() must check whether all the orderIds' _isBuyOrder boolean simply match each other rather than match the boolean parameter that was designated originally by fillBuyOrder() and fillSellOrder().

Design Considerations:
-   Exchange.sol
    -   It is possible modify the original fill order functions to handle ERC721 tokens. It can check whether the content contract implements the IMultipleRoyalties interface. This will decide whether it will run the original loop function with payableRoyalties or simply run payableMultipleRoyalties() once and a loop of transferRoyalty().
    -   However, the benefit of creating new fill order functions is to open the possibility down the line of allowing users to buy multiple unique assets in one go. This idea is incompatible with the original fill order functions which require all order Ids to have the same token information.
    -   It is under consideration whether an entirely new UniqueExchange.sol contract can be created in the future to specifically handle the exchange of ERC721 NFTs.

## Rationale

## Backwards Compatibility

## Test Cases

## Reference Implementation

## Security Considerations

## Copyright
