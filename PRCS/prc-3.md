---
title: Paima Inverse Projection Interface
description: Interface for inverse projection of game state into NFTs in other chains
author: sebastiengllmt, matejos (@matejos)
status: Draft
created: 2024-03-03
---

## Abstract

Allows tradability of game state directly on popular networks helps achieve a lot more composability and liquidity than would otherwise be possible. This standard helps define how to define NFTs in different chains without introducing centralization, lowered security or wait times for finality.

## Motivation

Many games, due to being data and computation heavy applications, run on sidechains, L2s and appchains as opposed to popular L1 blockchains. This is problematic because liquidity for trading assets live primarily on the L1s (different environments). A common solution to this problem is building an NFT bridge, but bridges have a bad reputation, often require a long delay (especially for optimistic bridges which often require 1 week), and bridging also makes upgrading the game harder as any update to the game state may now also require you to update the data associated with all the bridged state (ex: adding a new field for monsters in the game would require you to introduce this new field to all bridged assets).

Instead of bridging NFTs, this standard allows minting NFTs on more popular chains that acts as a pointing to game state. This allows keeping the game as the source of truth for game state.

## Specification

Every PRC-3 compliant contract must implement the `IInverseProjectedNft` interface:

```solidity
/// @dev A standard ERC721 that accepts calldata in the mint function for any initialization data needed in a Paima dApp.
interface IInverseProjectedNft is IERC4906 {
    /// @dev Emitted when `baseExtension` is updated from `oldBaseExtension` to `newBaseExtension`.
    event SetBaseExtension(string oldBaseExtension, string newBaseExtension);

    /// @dev Emitted when `baseUri` is updated from `oldUri` to `newUri`.
    event SetBaseURI(string oldUri, string newUri);

    /// @dev Burns token of ID `_tokenId`. Callable only by the owner of the specified token.
    /// Reverts if `_tokenId` is not existing.
    function burn(uint256 _tokenId) external;

    /// @dev Sets `_URI` as the `baseURI` of the NFT.
    /// Callable only by the contract owner.
    /// Emits the `SetBaseURI` event.
    function setBaseURI(string memory _URI) external;

    /// @dev Sets `_newBaseExtension` as the `baseExtension` of the NFT.
    /// Callable only by the contract owner.
    function setBaseExtension(string memory _newBaseExtension) external;

    /// @dev Returns the token URI of specified `tokenId` using a custom base URI.
    function tokenURI(
        uint256 tokenId,
        string memory customBaseUri
    ) external view returns (string memory);
}
```

We recommend the following baseURI:

```bash
https://${rpcBase}/inverseProjection/${purpose}/${chainIdentifier}/
```

Where
- `rpcBase` is the URI for the RPC
- `purpose` is a app-dependent string to describe what the NFT is for (ex: `monsters`)
- `chainIdentifier` is a unique ID for the chain following [caip-2](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md)

An example of such a `baseURI` is `https://rpc.mygame.com/inverseProjection/monsters/eip155:1/`

### Token Identifier

There are two possible ways to define the token identifier with different tradeoffs.

#### 1) App Initiated

In this case, the user first initiates the projection on the game layer by specifying the chain ID they want to project data to as well as the address they will mint with. The game then provides the user with a unique `userTokenId`, and the identifier will be `${address}/${userTokenId}.json`.

It will be up to the smart contract on the base layer to ensure the combination of `<address, userTokenId>` is unique across all mints. We RECOMMEND setting `userTokenId` to be an address-specific counter increasing in value starting from 1 to implement this.


```mermaid
sequenceDiagram
    actor Buyer
    participant L1
    actor Seller
    participant Game
    Seller->>Game: Request to mint NFT on <chainId, address>
    Game->>Seller: Unique userTokenId
    destroy Seller
    Seller->>L1: Mint NFT to chainId <br> using address<br> passing in userTokenId
    activate L1
    L1->>Game: Paima Primitive detects NFT creation
    Game->>Game: check address, chainId and userTokenId match
    L1->>Game: tokenURI
    Game->>L1: Asset state in the game
    Buyer->>L1: Buys NFT on market
    Buyer->>L1: Burn NFT
    deactivate L1
    L1->>Game: Paima Primitive detects burn
    Game->>Game: Give asset to buyer in-game
```

This case uses the following extension to the base interface

```solidity
/// @dev A Paima Inverse Projection NFT where initialization is handled by the app-layer.
interface IAppInverseProjectedNft is IInverseProjectedNft {
    /// @dev Emitted when the globally-enforced tokenId as well as the unique <minter, userTokenId> pair, and `initialData` provided in the `mint` function parameters.
    event Minted(uint256 indexed tokenId, address indexed minter, uint256 indexed userTokenId);

    /// @dev Mints a new token to address `_to`
    /// Increases the `totalSupply` and `currentTokenId`.
    /// Reverts if `totalSupply` is not less than `maxSupply` or if `_to` is a zero address.
    /// Emits the `Minted` event.
    function mint(address _to) external returns (uint256);
}
```

There are 2 error-cases to handle:
1. Querying a `tokenID` that has not yet been seen by the game node. This should not happen under normal use, but may happen if a user mints more times on the base layer without making any equivalent transaction in the app layer. This should return a `404 error` (to avoid NFT marketplaces caching dummy data)
2. Querying a `tokenID` 

#### 2) Base Layer Initiated

In this case, the user first initiates the projection on the base layer by simply minting the NFT specifying data as needed in the `initialData`. The `tokenId` from the smart contract will act as the `identifier`.

There are 2 error-cases to handle:
1. Querying a `tokenID` that has not yet been seen by the game node. This will happen because there is always a delay between something happening on the base layer and the Paima node detecting it. This should return a `404 error` instead of dummy data (to avoid NFT marketplaces caching dummy data)
2. Invalid `initialData` provided (the definition of invalid is app-specific)

This case uses the following extension to the base interface

```solidity
/// @dev A standard ERC721 that accepts calldata in the mint function for any initialization data needed in a Paima dApp.
interface IInverseBaseProjectedNft is IInverseProjectedNft {
    /// @dev Emitted when the globally-enforced tokenId, and `initialData` provided in the `mint` function parameters.
    event Minted(uint256 indexed tokenId, string initialData);

    /// @dev Mints a new token to address `_to`, passing `initialData` to be emitted in the event.
    /// Increases the `totalSupply` and `currentTokenId`.
    /// Reverts if `totalSupply` is not less than `maxSupply` or if `_to` is a zero address.
    /// Emits the `Minted` event.
    function mint(address _to, string calldata initialData) external returns (uint256);
}
```

### Invalid mint response

TODO: define the format for malicious mints

## Rationale

Instead of holding the data for the NFT in IPFS or other immutable storage, the NFT instead encodes which RPC call needs to be made to the game node to fetch the data this NFT encodes. Note that for this standard to be secure, you cannot mint these NFTs on arbitrary chains - rather, it has to be on a chain that the game is either actively monitoring (or occasionally receives updates about through a bridge or other mechanism).

Key differences from ERC721:
- `mint` can be called by anybody at anytime (infinite supply) and includes an `initialData` to pass in any other necessary data for the game, as well as possibly an `userTokenId` depending on which layer initiates the projection.
- `tokenURI` from `IERC721` will lookup from default RPC for the game to ensure data is properly visible from standard marketplaces like OpenSea. To avoid this being a point of decentralization, there is an additional `tokenURI` function that accepts a `customBaseUri` for marketplaces / users to provide their own RPC if they wish.
- The contract uses [ERC-4906](https://eips.ethereum.org/EIPS/eip-4906) to force marketplaces to invalidate their cache. This function is callable by anybody (not just the admin) so that if ever the game updates with new features (either user-initiated or by the original authors of the game), marketplaces will properly refetch the data.

### Rationale App-layer

By having `userTokenId` be a deterministic increasing value, it not only avoids double-mints (creating 2 NFTs pointing to the same app data), it also avoids any issues with failed transactions (if a tx on the base layer fails, just create a new tx)

Upside:
- Parallelize
Downside:
- Requires a transaction on the layer where the app is deployed (although usually this is a place where tx fees are cheap)

Edge case:
- Mint an NFT on the base layer that doesn't exist in the game yet, but exists later. This isn't a security vulnerability, but marketplaces may be slow to update their cache

### Rationale Base-layer

- Need to avoid double mint
    - Minter minting twice
    - Other people minting your NFT

Upside:
-
Downside:
- Extra `calldata` cost for `initialData`

## Reference Implementation

https://github.com/PaimaStudios/paima-engine/blob/master/packages/contracts/evm-contracts/contracts/token/InverseProjectedNft.sol

## Security Considerations

TODO

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
