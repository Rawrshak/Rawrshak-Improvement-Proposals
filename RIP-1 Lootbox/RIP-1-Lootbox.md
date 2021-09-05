---
rip: 1
title: Lootbox System Feature
author: (@livingaftermidnight)
discussions-to: https://discord.gg/BeEjqs9Qtx
status: Draft 
created: 2021-08-23
version: 0.1
---

## Abstract
We want to give the players the ability to burn in-game items for credits that they can then redeem for loot boxes. Loot boxes will be transferable or can be burned in order to mint the items they contain (whether guaranteed or random).

## Motivation
Lootboxes are a way to create demand for the ever-increasing supply of assets in the gaming ecosystem. Over time, gamers will continuously play games, unlock and mint assets. Aside from assets used as crafting material, there will eventually be an oversupply of common, even uncommon, assets. The Lootbox mechanism allows players to burn unwanted common assets and turn them into credits that can be redeemed for lootboxes. 

Ideally, this creates an equilibrium price for common assets on the marketplace. If assets become too cheap, people can buy these assets up, turn them into credits and then use those to open lootboxes for a chance to acquire interesting, exclusive, and rarer assets. 

Lootbox credits are per-developer and therefore do not interfere with other developer's ecosystem. If developer 1's ecosystem produces alot of Dev1 credit, the effect on Dev2's ecosystem is limited as Dev1's credits aren't redeemable elsewhere. Likewise, if Dev2 decides not to create lootboxes, only their assets' prices will converge to 0 over time due to the problem of oversupply.

## Specification
### Background
This document will explain how the loot box contract (and associated systems/contracts) will work within the existing Rawrshak ecosystem.

There are two major stakeholder perspectives we need to take into account to build this system out:

1. Developer Perspective
2. Player/User Perspective

### Developer Perspective

A content developer (games, art, etc) that wants to enable loot boxes will need to be able to interact with the Content Creator dApp GUI that will then talk to Rawrshak contracts on the blockchain. Through the dApp they will be able to execute the following loot box related actions:

- Add content and register content contracts. The loot box contract will need permission to go and mint (and burn) on those content contracts. 
- Set how much loot box credit will be given when the user burns said content/asset
- Create individual loot boxes.
	- Specify a list of asset tokens to be minted when the loot box is burned and set probabilities on each.
	- Enable certain asset tokens to be guaranteed to be minted or randomly minted based on probabilities.
	- Register the credit price to buy the loot box.

### Player/User Perspective

A user will have the following loot box related abilities:
- Burn assets for loot box credits
- Purchase (i.e. Mint) loot boxes with the credits they’ve earned (i.e. burn credits)
- Send (i.e. Gift) loot boxes to someone else
- Redeem (i.e. Burn) loot boxes to roll for the assets within.

### Proposed Technical Design
#### LootboxCredit.sol
- Users burn this in exchange for loot boxes.
- SalvageableAssets (see LibCraft.sol) will be setup by the developer to include LootboxCredit as a given SalvageReward
	- Developers fill out LibCraft.AssetData (need content address and LootboxCredit tokenId)
	- Players can earn/mint LootboxCredit from burning SalvageableAssets.
- Implements ERC20
	- This will allow the token to interface with the vast majority of existing ERC20-based dApps/systems.
 
#### LibLootbox.sol
- Contains library level code for Lootbox functions/operations.
- Contains dice roll functionality.
	- random()
		- Returns a single dice roll.
	- randomBatch()
		- May need to batch dice rolls together. So if there are 6 items that need randomization we return an array of 6 randomly seeded (non-unique) values.
- Upgradeable (will likely want to upgrade dice roll functionality after launch).
- struct LootboxReward (similar to LibCraft.sol SalvageReward)<br>
{<br>
	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AssetData asset; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;// asset to be minted when some Lootbox is burned.<br>
	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;uint256 probability; &nbsp;&nbsp;// stored in ETH<br>
	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;uint256 amount; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;// amount of asset minted when dice roll probability is met.<br>
}<br>

#### ILootbox.sol
- Interface that all Lootbox contracts share/implement
- Functions for getting/setting credit cost
- Functions for getting/setting content and probabilities that could be given when lootbox is burned.
- Functions for the minting of content assets and transfer of said assets to the lootbox owner.

#### LootboxManager.sol
- Developer talks to this (through the Content Creator dApp) to register ILootbox.sol instances
	- Developer only usage, no player/user will ever need to interface with this contract.
- Requires read/write permissions to set data into the LootboxStorage contract and a given Lootbox contract itself.
- Developer will fill out a structure (i.e. LibLootbox.Recipe) and passes it to this manager contract which will create the lootbox contract and fill its internal data with the struct.

#### LootboxStorage.sol
- Storage class for storing important lootbox specific data that can be used across multiple lootbox types for a given Rawrshak project.
- Stores lootbox "recipes" (i.e. ILootbox.sol instances)
- Stores credit cost to buy each lootbox recipe
	- uint256 cost;
- Stores per-game rarity classes (in array form) (i.e. Common, Uncommon, Rare, Ultra-Rare, etc)
- Individual ILootbox-derived Lootbox contracts will store a pointer to the storage contract for the reading of data in a read-only fashion.
- Potentially partially upgradeable. Will need further investigation.
 
#### LootboxByItem.sol
- Basic implementation of a Lootbox. All rewards are predetermined up front. Probabilities are stored by tokenId not by rarity/class.
- Implements ERC1155Upgradeable
- Implements ILootbox
- Stores pointer to the LootboxStorage contract for grabbing any necessary data in a read-only fashion.
- Uses LibLootbox.sol for dice roll (and other) functionality.
- Stores content and probabilities that could be given when lootbox is burned.
	- LibLootbox.LootboxReward[] rewardAssets;    // see LibLootbox.sol
	- Some content can be guaranteed to be minted in singles or multiples.
		- uint256 amount; // amount to be minted if probability passes.
	- Some content will be minted randomly based on given probabilities.
		- uint256 probability;
- Probabilities stored in ETH (uint256).
	- 1 ETH is 100%, 0.5 ETH is 50%, etc etc.
	- Probability cannot equal zero ETH/Wei
- uint256 numRewardAssetsGiven;			  // number of items the lootbox will randomly pick from the assets list when burned. Set by Developer when created.
- Minted by a given player/user from them calling a function which allows the lootbox to be minted if the required amount of LootboxCredit is available.
- Contains burn()
	- Each lootbox might want to handle this logic differently so we should store this logic in each lootbox type.
	- First checks if numRewardAssetsGiven < rewardAssets.length. If not, we skip ahead to checking individual item probabilities. If so, then we do the following:
	- First kicks off a dice roll randomBatch() with numRewardAssetsGiven number of seeds
		- The response determines which of the rewardAssets the user will be minted.
		- If this doesn't handle providing unique random numbers then we can just allow duplicate random numbers to mean that the user mints more than one of a given asset.
	- Once the list of assets to be minted is available, we can run the following logic to determine exactly what reward items the user will get.
- Loops through all assets in rewardAssets and checks probability.
	- If probability is equal to 1 ETH, then the asset is minted without a dice roll. (i.e. guaranteed items)
	- If probability is not equal to 1 ETH, then a dice roll is performed and compared against the probability assigned for that asset.
		- If a dice roll is won, the asset is minted and sent to the lootbox owner.
	- If randomBatch() is a possible function, then we should add all assets requiring a dice roll into a data structure and pass that along to the randomBatch() function to avoid a high gas cost for the overall burning transaction.
- Requests asset mint/burn permissions from each content contract that we need to interact with.
	- IContent(tokenId).approvalAllSystems(true);
 
#### LootboxByClass.sol
- Implementation of a Lootbox where probabilities are set by class type and not by individual items. Users will receive a random assortment of items from any class where the probability succeeds in the dice roll. Works a lot like OpenSea's MyLootBox.sol
- This class is very similar to LootboxByItem.sol except the following points:
- Stores a mapping of LootboxOption (uint256) to OptionSettings (struct)
	- OptionSettings contains the following:
		- uint256 maxQuantityPerOpen;  // Number of items to send per open. Set to zero to disable this option.
		- uint256[MAX_CLASSES] classProbabilities;  // Probability in ETH for receiving an item of each class. Descending.
		- bool hasGuaranteedClasses;  // Whether or not to enable guaranteed items below
		- uint16[MAX_CLASSES] guarantees;   // Number of items you are guaranteed to get for each class.
- Stores a mapping of tokenIds to rarity classes.
	- i.e. which tokenIds are assigned to a given rarity. This allows lootboxes to randomly pick tokenIds based on a given class when minting reward items.
	- Obviously contains function methods for adding tokenIds to this mapping which will need filled out by the developer through the Content Creator dApp.
- Stores a mapping of rarity class to a bool for whether or not the class is preminted.
 
### For Players/Users:
- Talks to LootboxManager which talks to LootboxStorage to handle burning players credits and minting a given Lootbox.
- Different Lootbox options will cost different credit amounts.
- Players burn assets for LootboxCredit which they mint through the LootboxManager (which forwards the request to LootboxStorage).
- Talks to a given/owned ERC1155 ILootbox.sol contract to burn it in exchange for assets it contains.

### Extensibility

Our two provided Lootbox contracts may work for the vast majority of developers but some developers we expect will want to add functionality and therefore mint their own unique lootbox contract. The dApp UI will need to allow devs to input their own lootbox contract address (follows ILootbox.sol).

### Rarity / Classes

Developers will likely want to customize the different rarities that their items have. For instance, one project might want their rarities to be like Common, Uncommon, Rare, Ultra Rare while
another project would want something like Bronze, Silver, Gold, Platinum, etc. The Content Creator dApp will need to allow the developer to input whatever rarity titles they want to use when
they setup a project. We can store this into a data structure (likely an array) and access it throughout the contracts whenever we need to deal with item rarities.
 
### The Graph support

All mutative functions will emit events that allow the relational database stored within the Graph node to be updated.

## Rationale
Todo:

## Backwards Compatibility
None

## Test Cases

All major functionality will be tested with unit tests that ensure everything is working properly.		
		
## Reference Implementation
[OpenSea's Lootbox Implementation](https://github.com/ProjectOpenSea/opensea-erc1155/blob/master/contracts/MyLootBox.sol)

## Security Considerations

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
