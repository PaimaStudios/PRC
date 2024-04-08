---
title: Paima Orderbook DEX
description: Interface for facilitating trading of a game asset (living on a game chain) on different base chain.
author: sebastiengllmt, matejos (@matejos)
status: Draft
created: 2024-03-13
---

## Abstract

Allowing tradability of game assets directly on popular networks helps achieve a lot more composability and liquidity than would otherwise be possible. This standard helps define how to define an orderbook-like DEX smart contract for trading game assets on different chains without introducing centralization, lowered security or wait times for finality. The game asset is represented by an IERC1155 compliant token, utilizing the PRC-5 standard (Inverse Projection Interface for ERC1155).

## Motivation

Many games, due to being data and computation heavy applications, run on sidechains, L2s and appchains as opposed to popular L1 blockchains. This is problematic because liquidity for trading assets live primarily on the L1s (different environments). A common solution to this problem is building a bridge, but bridges have a bad reputation, often require a long delay (especially for optimistic bridges which often require 1 week), and bridging also makes upgrading the game harder as any update to the game state may now also require you to update the data associated with all the bridged state.

Instead of bridging, this standard allows trading the game assets on base chain via creating and filling sell orders in an orderbook DEX smart contract.

## Specification

Every PRC-4 compliant contract must implement the `IOrderbookDex` interface and events:

```solidity
/// @notice Facilitates base-chain trading of an asset that is living on a different app-chain.
/// @dev Orders are identified by a unique incremental `orderId`.
interface IOrderbookDex is IERC1155Receiver {
    struct Order {
        /// @dev The asset's unique token identifier.
        uint256 assetId;
        /// @dev The amount of the asset that is available to be sold.
        uint256 assetAmount;
        /// @dev The price per one unit of asset.
        uint256 pricePerAsset;
        /// @dev The seller's address.
        address payable seller;
    }

    /// @param seller The seller's address.
    /// @param orderId The order's unique identifier.
    /// @param assetId The asset's unique token identifier.
    /// @param assetAmount The amount of the asset that has been put for sale.
    /// @param pricePerAsset The requested price per one unit of asset.
    event OrderCreated(
        address indexed seller,
        uint256 indexed orderId,
        uint256 indexed assetId,
        uint256 assetAmount,
        uint256 pricePerAsset
    );

    /// @param seller The seller's address.
    /// @param orderId The order's unique identifier.
    /// @param buyer The buyer's address.
    /// @param assetAmount The amount of the asset that was traded.
    /// @param pricePerAsset The price per one unit of asset that was paid.
    event OrderFilled(
        address indexed seller,
        uint256 indexed orderId,
        address indexed buyer,
        uint256 assetAmount,
        uint256 pricePerAsset
    );

    /// @param seller The seller's address.
    /// @param id The order's unique identifier.
    event OrderCancelled(address indexed seller, uint256 indexed id);

    /// @notice Returns the address of the asset that is being traded in this DEX contract.
    function getAsset() external view returns (address);

    /// @notice Returns the `orderId` of the next sell order.
    function getCurrentOrderId() external view returns (uint256);

    /// @notice Returns the Order struct information about an order identified by the `orderId`.
    function getOrder(uint256 orderId) external view returns (Order memory);

    /// @notice Creates a sell order for the `assetAmount` of `assetId` at `pricePerAsset`.
    /// @dev The order information is saved in a mapping `orderId -> Order`, with `orderId` being a unique incremental identifier.
    /// MUST transfer the `assetAmount` of `assetId` from the seller to the contract.
    /// MUST emit `OrderCreated` event.
    /// @return The unique identifier of the created order.
    function createSellOrder(
        uint256 assetId,
        uint256 assetAmount,
        uint256 pricePerAsset
    ) external returns (uint256);

    /// @notice Creates a batch of sell orders for the `assetAmount` of `assetId` at `pricePerAsset`.
    /// @dev This is a batched version of `createSellOrder` that simply iterates through the arrays to call said function.
    /// @return The unique identifiers of the created orders.
    function createBatchSellOrder(
        uint256[] memory assetIds,
        uint256[] memory assetAmounts,
        uint256[] memory pricesPerAssets
    ) external returns (uint256[] memory);

    /// @notice Consecutively fills an array of orders identified by the `orderId` of each order,
    /// by providing an exact amount of ETH and requesting a specific minimum amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be at least `minimumAsset`.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the excess back to `msg.sender`.
    /// MUST decrease the `assetAmount` parameter for the specified order according to how much of it was filled,
    /// and transfer that amount of the order's `assetId` to the buyer.
    /// MUST emit `OrderFilled` event for each order accordingly.
    function fillOrdersExactEth(uint256 minimumAsset, uint256[] memory orderIds) external payable;

    /// @notice Consecutively fills an array of orders identified by the `orderId` of each order,
    /// by providing a possibly surplus amount of ETH and requesting an exact amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be exactly `assetAmount`. Excess ETH MUST be returned back to `msg.sender`.
    /// MUST decrease the `assetAmount` parameter for the specified order according to how much of it was filled,
    /// and transfer that amount of the order's `assetId` to the buyer.
    /// MUST emit `OrderFilled` event for each order accordingly.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the difference back to `msg.sender`.
    function fillOrdersExactAsset(uint256 assetAmount, uint256[] memory orderIds) external payable;

    /// @notice Cancels the sell order identified by the `orderId`, transferring the order's assets back to the seller.
    /// @dev MUST revert if the order's seller is not `msg.sender`.
    /// MUST change the `assetAmount` parameter for the specified order to `0`.
    /// MUST emit `OrderCancelled` event.
    /// MUST transfer the `assetAmount` of `assetId` back to the seller.
    function cancelSellOrder(uint256 orderId) external;

    /// @notice Cancels a batch of sell orders identified by the `orderIds`, transferring the orders' assets back to the seller.
    /// @dev This is a batched version of `cancelSellOrder` that simply iterates through the array to call said function.
    function cancelBatchSellOrder(uint256[] memory orderIds) external;
}
```

## Rationale

It's not an AMM. In a typical AMM dex, there are liquidity pools and you trade against these pools. This is unsuitable for this use-case because a contract on base chain has no means of knowing the state of the game chain so the contract cannot automate market making. Instead, buyers have to be sending tokens directly to the sellers.

### The idea:

1. User withdraws the game asset to the base chain by using the PRC-5 Inverse Projection system - they lock the appropriate amount of the asset and the game chain assigns it the address-specific incremental `orderId`, for which they mint InverseProjected1155 tokens on the base chain.
2. Smart contract on base chain facilitates trading game asset. This contract allows the following:
   - Function to create a sell order of specified amount of asset and price per one unit of said asset. A sell order persists and has the following properties:
     - `uint256 assetId` - token ID of the ERC1155 asset being sold,
     - `uint256 assetAmount` - amount of the game asset the seller is selling,
     - `uint256 pricePerAsset` - the price in native gas tokens of the base chain the seller is requesting,
     - `address seller` - the seller's address.
   - Function to cancel a sell order of specified `orderId`
   - Batch functions `createBatchSellOrder` and `cancelBatchSellOrder` of the abovementioned functions
   - Function to fill sell orders (in other words - buy). There is no "buy order" that persists, and rather the buy function directly transfers the value specified by price defined in the orders to the sellers. There are 2 variations of fill function:
     - `fillOrdersExactEth` - consecutively fills the array of specified orders until all provided ETH is spent and a specified minimum amount of asset has been achieved. Example: You want to buy as much asset as possible for 1 ETH.
     - `fillOrdersExactAsset` - consecutively fills the array of specified orders until specified exact amount of asset has been achieved, and returns the excess ETH back to the sender. Example: You want to buy 1000 units of asset as cheaply as possible.

The contract on the base chain has no way of knowing if somebody who makes a sell order really has that amount of game assets in their account. Rather,

1. When somebody creates a sell order for a specific asset ID on the game chain, its validity can be checked via the API in the token's `uri` function.
2. When somebody wants to buy, they specify the orders they wish to purchase by their unique ID.
   - The responsibility of querying the API of game chain to check the validity of the sell order is left up to the front-end providers, as well as the presentation of the possible sell orders.
3. The user then makes a smart contract call that fulfills the orders and transfers the right amounts of assets to the corresponding accounts in a single transaction (atomic).

## Reference Implementation

You can find all the contracts and interfaces in the Paima Engine codebase [here](https://github.com/PaimaStudios/paima-engine/blob/master/packages/contracts/evm-contracts/contracts/orderbook/).

## Considerations

### Avoiding many failed txs in this model

The problem with this model is people will naturally all try to buy the order with the most favorable sell price. The chosen base chain should have fast blocks so that collisions are not too likely if the UI updates fast enough, but if bots start trading good orders as soon as they appear it may cause a rush with many failed transactions.

**Option 1) A L3 for all games**

The key observation is that solving this concurrency issue does not need any knowledge of the game itself (it doesn't matter if orders are valid or not from the dApp perspective. It just assumes buyers are not making bad purchases).

This is an important note because it means we could have a decentralized L3 on top of the base chain whose goal it is to match orders properly. Notably, there is a model that provides exactly what we want with no tx fee if concurrent actions are attempted like this: the UTxO model. That is to say, implementing this system on top of Cardano (Aiken) or Fuel is actually much easier than the account model of EVM. However, e.g. Arbitrum users of course cannot be using Cardano, so this would most likely require running a Fuel L3 for Arbitrum (or maybe some ZK-ified UTxO platform) where the validators are decided by the Paima ecosystem token.

However, this falls short on a few key points:

- It would require adding Fuel support to Paima Engine to properly monitor it.
- It would require the Paima ecosystem token to be released (not released yet).
- It means we lose some composability with other Arbitrum dApps (unless Fuel adds a wrapped smart contract system like Milkomeda).
- Users may be hesitant to bridge to this L3 just to trade, so we would have to abstract this away from them (again, wrapped smart contracts may help in the optimistic case where there isn't a conflicting order so they can buy game assets right away).

**Option 2) Frontend-driven concurrency management**

Since there are 2 different buy functions - each with clear one-way intent and slippage mechanism - it is possible to submit a fill order specifying surplus of suboptimal orders. This surplus would automatically get used the best orders are quickly filled in the periods of high activity, and would result in slightly suboptimal trade for the user, but non-reverting transaction nonetheless. However, it might be tricky to find the right balance of providing enough surplus orders to the transaction and not providing needlessly too much.

## Security Considerations

**Honest RPC**: This standard relies on the default RPC being honestly operated. This is, however, not really a new trust assumption because this is a required assumption in nearly all dApps at the moment (including those like OpenSea where you have to trust them to be operating their website honestly). Just like somebody can run their own Ethereum fullnode to verify the data they see in an NFT marketplace, they can also sync fullnode for a Paima app and use their own RPC to fetch the state.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
