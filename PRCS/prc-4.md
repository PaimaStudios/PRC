---
title: Paima Orderbook DEX
description: Interface for facilitating trading of a game asset (living on a game chain) on different base chain.
author: sebastiengllmt, matejos (@matejos)
status: Draft
created: 2024-03-13
---

## Abstract

Allowing tradability of game assets directly on popular networks helps achieve a lot more composability and liquidity than would otherwise be possible. This standard helps define how to define an orderbook-like DEX smart contract for trading game assets on different chains without introducing centralization, lowered security or wait times for finality.

## Motivation

Many games, due to being data and computation heavy applications, run on sidechains, L2s and appchains as opposed to popular L1 blockchains. This is problematic because liquidity for trading assets live primarily on the L1s (different environments). A common solution to this problem is building a bridge, but bridges have a bad reputation, often require a long delay (especially for optimistic bridges which often require 1 week), and bridging also makes upgrading the game harder as any update to the game state may now also require you to update the data associated with all the bridged state.

Instead of bridging, this standard allows trading the game assets on base chain via creating and filling sell orders in an orderbook DEX smart contract.

## Specification

Every PRC-4 compliant contract must implement the `IOrderbookDex` interface and events:

```solidity
/// @notice Facilitates base-chain trading of an asset that is living on a different app-chain.
/// @dev The contract should never hold any ETH itself.
interface IOrderbookDex is IERC165 {
    struct Order {
        uint256 assetAmount;
        uint256 pricePerAsset;
        bool cancelled;
    }

    event OrderCreated(address indexed seller, uint256 indexed orderId, uint256 assetAmount, uint256 pricePerAsset);
    event OrderFilled(
        address indexed seller,
        uint256 indexed orderId,
        address indexed buyer,
        uint256 assetAmount,
        uint256 pricePerAsset
    );
    event OrderCancelled(address indexed seller, uint256 indexed orderId);

    /// @notice Returns the seller's current `orderId` (index that their new sell order will be mapped to).
    function getSellerOrderId(address seller) external view returns (uint256);

    /// @notice Returns the Order struct information about an order identified by the combination `<seller, orderId>`.
    function getOrder(address seller, uint256 orderId) external view returns (Order memory);

    /// @notice Creates a sell order with incremental seller-specific `orderId` for the specified `assetAmount` at specified `pricePerAsset`.
    /// @dev The order information is saved in a nested mapping `seller address -> orderId -> Order`.
    /// MUST emit `OrderCreated` event.
    function createSellOrder(uint256 assetAmount, uint256 pricePerAsset) external;

    /// @notice Consecutively fills an array of orders identified by the combination `<seller, orderId>`,
    /// by providing an exact amount of ETH and requesting a specific minimum amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be at least `minimumAsset`.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the excess back to msg.sender.
    /// An order whose `cancelled` parameter has value `true` MUST NOT be filled.
    /// MUST change the `assetAmount` parameter for the specified order according to how much of it was filled.
    /// MUST emit `OrderFilled` event for each order accordingly.
    function fillOrdersExactEth(uint256 minimumAsset, address[] memory sellers, uint256[] memory orderIds)
        external
        payable;

    /// @notice Consecutively fills an array of orders identified by the combination `<seller, orderId>`,
    /// by providing a possibly surplus amount of ETH and requesting an exact amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be exactly `assetAmount`. Excess ETH MUST be returned back to `msg.sender`.
    /// An order whose `cancelled` parameter has value `true` MUST NOT be filled.
    /// MUST change the `assetAmount` parameter for the specified order according to how much of it was filled.
    /// MUST emit `OrderFilled` event for each order accordingly.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the difference back to msg.sender.
    function fillOrdersExactAsset(uint256 assetAmount, address[] memory sellers, uint256[] memory orderIds)
        external
        payable;

    /// @notice Cancels the sell order identified by combination `<msg.sender, orderId>`, making it unfillable.
    /// @dev MUST change the `cancelled` parameter for the specified order to `true`.
    /// MUST emit `OrderCancelled` event.
    function cancelSellOrder(uint256 orderId) external;
}
```

## Rationale

It's not an AMM. In a typical AMM dex, there are liquidity pools and you trade against these pools. This is unsuitable for this use-case because a contract on base chain has no means of knowing the state of the game chain so the contract cannot automate market making. Instead, buyers have to be sending tokens directly to the sellers.

### The idea:

1. User initiates the sell order on the game chain - they lock the appropriate amount of the asset and the game chain assigns it the address-specific incremental `orderId`.
2. Smart contract on base chain facilitates trading game asset. This contract allows the following:
    * Function to create a sell order of specified amount of asset and price per one unit of said asset. A sell order persists and has the following properties: 
      * `uint256 assetAmount` - amount of the game asset the seller is selling,
      * `uint256 pricePerAsset` - the price in native gas tokens of the base chain the seller is requesting,
      * `bool cancelled` - signals whether or not the sell order has been cancelled.
    * Function to cancel a sell order of specified `orderId`.
    * Function to fill a sell order (in other words - buy). There is no "buy order" that persists, and rather the buy function directly transfers the value specified by price defined in the orders to the sellers. There are 2 variations of fill function:
      * `fillOrdersExactEth` - consecutively fills the array of specified orders until all provided ETH is spent and a specified minimum amount of asset has been achieved. Example: You want to buy as much asset for 1 ETH.
      * `fillOrdersExactAsset` - consecutively fills the array of specified orders until specified exact amount of asset has been achieved, and returns the excess ETH back to the sender. Example: You want to buy 1000 units of asset as cheaply as possible.
3. The game chain monitors this contract using Paima Primitives and exposes an API to query its state.

The contract on the base chain has no way of knowing if somebody who makes a sell order really has that amount of game assets in their account. Rather,

1. When somebody creates a sell order on the game chain and on the base chain, they will have the same address-specific ID that symbolically links them together.
2. When somebody wants to buy, they specify the orders they wish to purchase by combination of seller address and address-specific ID. 
    * The responsibility of querying the API of game chain to check the validity of the sell order is left up to the front-end providers, as well as the presentation of the possible sell orders.
3. The user then makes a smart contract call that fulfills the orders and sends the right amount to the corresponding accounts in a single transaction (atomic).
4. The game chain monitors the successfully fulfilled orders on the base chain and transfers the assets between accounts.

#### Cancelling an asset lock on game chain:

To maintain base chain as the source of truth for transacting, it is important for the game chain to allow the user to unlock their game asset (cancel the sell order lock) ONLY IF a corresponding sell order on base chain side exists AND it has been cancelled. That means if a user initiates a sell order on game chain and changes their mind, they must proceed to the base chain to create and cancel a sell order with the corresponding ID.

### **Note:**
All actions (sell, buy, cancel) are done on the base chain (and not on the game chain). This allows the base chain to be the source of truth for the state of which orders are valid which avoids double-spends. That is to say, if you try and perform a buy order, you are guaranteed by the base chain itself that the order hasn't been fulfilled by anybody else yet.

## Reference Implementation

You can find all the contracts and interfaces in the Paima Engine codebase [here](https://github.com/PaimaStudios/paima-engine/blob/master/packages/contracts/evm-contracts/contracts/orderbook/).

## Considerations

### Avoiding many failed txs in this model

The problem with this model is people will naturally all try to buy the order with the most favorable sell price. The chosen base chain should have fast blocks so that collisions are not too likely if the UI updates fast enough, but if bots start trading good orders as soon as they appear it may cause a rush with many failed transactions.

**Option 1) A L3 for all games**

The key observation is that solving this concurrency issue does not need any knowledge of the game itself (it doesn't matter if orders are valid or not from the dApp perspective. It just assumes buyers are not making bad purchases).

This is an important note because it means we could have a decentralized L3 on top of the base chain whose goal it is to match orders properly. Notably, there is a model that provides exactly what we want with no tx fee if concurrent actions are attempted like this: the UTxO model. That is to say, implementing this system on top of Cardano (Aiken) or Fuel is actually much easier than the account model of EVM. However, e.g. Arbitrum users of course cannot be using Cardano, so this would most likely require running a Fuel L3 for Arbitrum (or maybe some ZK-ified UTxO platform) where the validators are decided by the Paima ecosystem token.

However, this falls short on a few key points:

* It would require adding Fuel support to Paima Engine to properly monitor it.
* It would require the Paima ecosystem token to be released (not released yet).
* It means we lose some composability with other Arbitrum dApps (unless Fuel adds a wrapped smart contract system like Milkomeda).
* Users may be hesitant to bridge to this L3 just to trade, so we would have to abstract this away from them (again, wrapped smart contracts may help in the optimistic case where there isn't a conflicting order so they can buy game assets right away).

**Option 2) Frontend-driven concurrency management**

Since there are 2 different buy functions - each with clear one-way intent and slippage mechanism - it is possible to submit a fill order specifying surplus of suboptimal orders. This surplus would automatically get used the best orders are quickly filled in the periods of high activity, and would result in slightly suboptimal trade for the user, but non-reverting transaction nonetheless. However, it might be tricky to find the right balance of providing enough surplus orders to the transaction and not providing needlessly too much.

## Security Considerations

**Honest RPC**: This standard relies on the default RPC being honestly operated. This is, however, not really a new trust assumption because this is a required assumption in nearly all dApps at the moment (including those like OpenSea where you have to trust them to be operating their website honestly). Just like somebody can run their own Ethereum fullnode to verify the data they see in an NFT marketplace, they can also sync fullnode for a Paima app and use their own RPC to fetch the state.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
