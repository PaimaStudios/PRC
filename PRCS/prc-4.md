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
        uint256 price;
        address payable seller;
        bool active;
    }

    event OrderCreated(
        uint256 indexed orderId,
        address indexed seller,
        uint256 assetAmount,
        uint256 price
    );
    event OrderFilled(
        uint256 indexed orderId,
        address indexed seller,
        address indexed buyer,
        uint256 assetAmount,
        uint256 price
    );
    event OrderCancelled(uint256 indexed orderId);

    /// @notice Returns the current index of orders (index that a new sell order will be mapped to).
    function getOrdersIndex() external view returns (uint256);

    /// @notice Returns the Order struct information about order of specified `orderId`.
    function getOrder(uint256 orderId) external view returns (Order memory);

    /// @notice Creates a sell order for the specified `assetAmount` at specified `price`.
    /// @dev The order is saved in a mapping from incremental ID to Order struct.
    /// MUST emit `OrderCreated` event.
    function createSellOrder(uint256 assetAmount, uint256 price) external;

    /// @notice Fills an array of orders specified by `orderIds`, transferring portion of msg.value
    /// to the orders' sellers according to the price.
    /// @dev MUST revert if `active` parameter is `false` for any of the orders.
    /// MUST change the `active` parameter for the specified order to `false`.
    /// MUST emit `OrderFilled` event for each order.
    /// If msg.value is more than the sum of orders' prices, it SHOULD refund the difference back to msg.sender.
    function fillSellOrders(uint256[] memory orderIds) external payable;

    /// @notice Cancels the sell order specified by `orderId`, making it unfillable.
    /// @dev Reverts if the msg.sender is not the order's seller.
    /// MUST change the `active` parameter for the specified order to `false`.
    /// MUST emit `OrderCancelled` event.
    function cancelSellOrder(uint256 orderId) external;
}
```

## Rationale

It's not an AMM. In a typical AMM dex, there are liquidity pools and you trade against these pools. This is unsuitable for this use-case because a contract on base chain has no means of knowing the state of the game chain so the contract cannot automate market making. Instead, buyers have to be sending tokens directly to the sellers.

### The idea:

1. Smart contract on base chain is created to facilitate trading game asset. This contract allows the following:
    * Function to create a sell order. A sell order persists and has the following properties: 
      * `assetAmount` - amount of the game asset the seller is selling,
      * `price` - the price in native gas tokens of the base chain the seller is requesting,
      * `seller` - the address of the seller,
      * `active` - signalizes if the sell order is active or not (meaning it has been cancelled or filled)
    * Function to cancel a sell order.
    * Function to fill a sell order (in other words - buy). There is no "buy order" that persists, and rather the buy function directly transfers the value specified by price defined in the orders to the sellers. Buying marks all the sell orders it fulfills as inactive by changing the value of `active` flag to `false`.
2. The game chain monitors this contract using CDEs and exposes an API to query its state.

That is to say, the contract on the base chain has no way of knowing if somebody who makes a sell order really has that amount of game assets in their account. Rather,

1. When somebody creates a sell order: The game chain sees it and checks if the user has enough of the asset in their account.
    * If yes, the amount is locked so they cannot spend it in-game (unless they cancel the order).
    * If not, the order is marked as invalid (to differentiate it from the case where it doesn't know the order exists yet).
2. When somebody wants to buy, they specify the orders they wish to purchase by ID. 
    * The responsibility of querying the API of game chain to check the validity of the sell order is left up to the front-end providers, as well as the presentation of the possible sell orders.
3. The user then makes a smart contract call that fulfills all the orders and sends the right amount to the corresponding accounts in a single transaction (atomic).
4. The game chain monitors the successfully fulfilled orders on the base chain and transfers the assets between accounts.

**Note:**
All actions (sell, buy, cancel) are done on the base chain (and not on the game chain). This allows the base chain to be the source of truth for the state of which orders are valid which avoids double-spends. That is to say, if you try and perform a buy order, you are guaranteed by the base chain itself that the order hasn't been fulfilled by anybody else yet.

## Reference Implementation

You can find all the contracts and interfaces in the Paima Engine codebase [here](https://github.com/PaimaStudios/paima-engine/blob/master/packages/contracts/evm-contracts/contracts/orderbook/).

## Considerations

### Avoiding many failed txs in this model

The problem with this model is people will naturally all try to buy the order with the most favorable sell price. Chosen base chain should have fast blocks so collisions are not so likely if the UI updates fast enough, but if bots start trading good orders as soon as they appear it may cause a rush with many failed transactions.

**Option 1) A L3 for all games**

However, the key observation is that solving this concurrency issue does not need any knowledge of the game itself (it doesn't matter if orders are valid or not from the dApp perspective. It just assumes buyers are not making bad purchases).

This is an important note because it means we could have a decentralized L3 on top of the base chain whose goal it is to match orders properly. Notably, there is a model that provides exactly what we want with no tx fee if concurrent actions are attempted like this: the UTxO model. That is to say, implementing this system on top of Cardano (Aiken) or Fuel is actually much easier than the account model of EVM. However, e.g. Arbitrum users of course cannot be using Cardano, so this would most likely require running a Fuel L3 for Arbitrum (or maybe some ZK-ified UTxO platform if one exists) where the validators are decided by the Paima ecosystem token.

It falls short on a few key points:

* It would require adding Fuel support to Paima Engine to properly monitor it.
* It would require the Paima ecosystem to be released (not released yet).
* It means we lose some composability with other Arbitrum dApps (unless Fuel adds a wrapped smart contract system like Milkomeda).
* Users may be hesitant to bridge to this L3 just to trade, so we would have to abstract this away from them (again, wrapped smart contracts may help in the optimistic case where there isn't a conflicting order so they can buy game assets right away).

**Option 2) Stylus**

This option assumes the base chain being Arbitrum.  
Arbitrum recently introduced a new programming language called Stylus which is much faster & cheaper than EVM. Additionally, it's composable with EVM contracts so you do not run into the same composability tradeoffs as with L3s. However, it does not appear to be able to solve the concurrent issue entirely like the UTxO model. Rather, it might just make the gas cost of failing cheaper. Additionally, it's not live on mainnet at the moment.

**Option 3) Frontend-driven concurrency management**

This option is perhaps the easiest if there is only a single website for the DEX, because the website itself can keep track of orders people are attempting to make and thus avoid conflicting orders being placed. However, this can quickly fall apart if another website appears or if people start making trades by directly interacting with the contract.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
