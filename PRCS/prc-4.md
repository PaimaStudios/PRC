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
import {IERC1155Receiver} from "@openzeppelin/contracts/token/ERC1155/IERC1155Receiver.sol";

/// @notice Facilitates base-chain trading of assets that are living on a different app-chain.
/// @dev Orders are identified by an asset-specific unique incremental `orderId`.
interface IOrderbookDex is IERC1155Receiver {
    struct FeeInfo {
        /// @dev The maker fee collected from the seller when a sell order is filled. Expressed in basis points.
        uint256 makerFee;
        /// @dev The taker fee collected from the buyer when a sell order is filled. Expressed in basis points.
        uint256 takerFee;
        /// @dev Flag indicating whether the fees are set.
        bool set;
    }

    struct Order {
        /// @dev The asset's unique token identifier.
        uint256 assetId;
        /// @dev The amount of the asset that is available to be sold.
        uint256 assetAmount;
        /// @dev The price per one unit of asset.
        uint256 pricePerAsset;
        /// @dev The seller's address.
        address payable seller;
        /// @dev The maker fee in basis points, set when order is created, defined by the asset's fee info.
        uint256 makerFee;
        /// @dev The taker fee in basis points, set when order is created, defined by the asset's fee info.
        uint256 takerFee;
        /// @dev The order creation fee paid by the seller when creating the order, refunded if sell order is cancelled.
        uint256 creationFeePaid;
    }

    /// @param user The address that claimed the balance.
    /// @param amount The amount that was claimed.
    event BalanceClaimed(address indexed user, uint256 amount);

    /// @param asset The asset's address (zero address if changing default fees).
    /// @param makerFee The new maker fee in basis points.
    /// @param takerFee The new taker fee in basis points.
    event FeeInfoChanged(address indexed asset, uint256 makerFee, uint256 takerFee);

    /// @param asset The asset's address.
    /// @param assetId The asset's unique token identifier.
    /// @param orderId The order's asset-specific unique identifier.
    /// @param seller The seller's address.
    /// @param assetAmount The amount of the asset that has been put for sale.
    /// @param pricePerAsset The requested price per one unit of asset.
    /// @param makerFee The maker fee in basis points.
    /// @param takerFee The taker fee in basis points.
    event OrderCreated(
        address indexed asset,
        uint256 indexed assetId,
        uint256 indexed orderId,
        address seller,
        uint256 assetAmount,
        uint256 pricePerAsset,
        uint256 makerFee,
        uint256 takerFee
    );

    /// @param asset The asset's address.
    /// @param assetId The asset's unique token identifier.
    /// @param orderId The order's asset-specific unique identifier.
    /// @param seller The seller's address.
    /// @param buyer The buyer's address.
    /// @param assetAmount The amount of the asset that was traded.
    /// @param pricePerAsset The price per one unit of asset that was paid.
    /// @param makerFeeCollected The maker fee in native tokens that was collected.
    /// @param takerFeeCollected The taker fee in native tokens that was collected.
    event OrderFilled(
        address indexed asset,
        uint256 indexed assetId,
        uint256 indexed orderId,
        address seller,
        address buyer,
        uint256 assetAmount,
        uint256 pricePerAsset,
        uint256 makerFeeCollected,
        uint256 takerFeeCollected
    );

    /// @param asset The asset's address.
    /// @param assetId The asset's unique token identifier.
    /// @param orderId The order's asset-specific unique identifier.
    event OrderCancelled(address indexed asset, uint256 indexed assetId, uint256 indexed orderId);

    /// @param oldFee The old fee value.
    /// @param newFee The new fee value.
    event OrderCreationFeeChanged(uint256 oldFee, uint256 newFee);

    /// @param receiver The address that received the fees.
    /// @param amount The amount of fees that were withdrawn.
    event FeesWithdrawn(address indexed receiver, uint256 amount);

    /// @notice The balance of `user` that's claimable by `claim` function.
    function balances(address user) external view returns (uint256);

    /// @notice Withdraw the claimable balance of the caller.
    function claim() external;

    /// @notice The `orderId` of the next sell order for specific `asset`.
    function currentOrderId(address asset) external view returns (uint256);

    /// @notice The total amount of fees collected by the contract.
    function collectedFees() external view returns (uint256);

    /// @notice The default maker fee, used if fee information for asset is not set.
    function defaultMakerFee() external view returns (uint256);

    /// @notice The default taker fee, used if fee information for asset is not set.
    function defaultTakerFee() external view returns (uint256);

    /// @notice The flat fee paid by the seller when creating a sell order, to prevent spam.
    function orderCreationFee() external view returns (uint256);

    /// @notice The maximum fee, maker/taker fees cannot be set to exceed this amount.
    function maxFee() external view returns (uint256);

    /// @notice The fee information of `asset`.
    function getAssetFeeInfo(address asset) external view returns (FeeInfo memory);

    /// @notice Returns the asset fees if set, otherwise returns the default fees.
    function getAssetAppliedFees(
        address asset
    ) external view returns (uint256 makerFee, uint256 takerFee);

    /// @notice Set the fee information of `asset`. Executable only by the owner.
    /// @dev MUST revert if `makerFee` or `takerFee` exceeds `maxFee`.
    /// MUST revert if called by unauthorized account.
    function setAssetFeeInfo(address asset, uint256 makerFee, uint256 takerFee) external;

    /// @notice Set the default fee information that is used if fee information for asset is not set. Executable only by the owner.
    /// @dev MUST revert if `makerFee` or `takerFee` exceeds `maxFee`.
    /// MUST revert if called by unauthorized account.
    function setDefaultFeeInfo(uint256 makerFee, uint256 takerFee) external;

    /// @notice Set the flat fee paid by the seller when creating a sell order, to prevent spam. Executable only by the owner.
    /// @dev MUST revert if called by unauthorized account.
    function setOrderCreationFee(uint256 fee) external;

    /// @notice Returns the Order struct information about an order identified by the `orderId` for specific `asset`.
    function getOrder(address asset, uint256 orderId) external view returns (Order memory);

    /// @notice Creates a sell order for the `assetAmount` of `asset` with ID `assetId` at `pricePerAsset`. Requires payment of `orderCreationFee`.
    /// @dev The order information is saved in a mapping `asset -> orderId -> Order`, with `orderId` being an asset-specific unique incremental identifier.
    /// MUST transfer the `assetAmount` of `asset` with ID `assetId` from the seller to the contract.
    /// MUST emit `OrderCreated` event.
    /// MUST revert if `msg.value` is less than `orderCreationFee`.
    /// @return The asset-specific unique identifier of the created order.
    function createSellOrder(
        address asset,
        uint256 assetId,
        uint256 assetAmount,
        uint256 pricePerAsset
    ) external payable returns (uint256);

    /// @notice Creates a batch of sell orders for the `assetAmount` of `asset` with ID `assetId` at `pricePerAsset`. Requires payment of `orderCreationFee` times the amount of orders.
    /// @dev This is a batched version of `createSellOrder` that simply iterates through the arrays to call said function.
    /// MUST revert if `msg.value` is less than `orderCreationFee * assetIds.length`.
    /// @return The asset-specific unique identifiers of the created orders.
    function createBatchSellOrder(
        address asset,
        uint256[] memory assetIds,
        uint256[] memory assetAmounts,
        uint256[] memory pricesPerAssets
    ) external payable returns (uint256[] memory);

    /// @notice Consecutively fills an array of orders of `asset` identified by the asset-specific `orderId` of each order,
    /// by providing an exact amount of ETH and requesting a specific minimum amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be at least `minimumAsset`.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the excess back to `msg.sender`.
    /// MUST decrease the `assetAmount` parameter for the specified order according to how much of it was filled,
    /// and transfer that amount of the order's `asset` with ID `assetId` to the buyer.
    /// MUST emit `OrderFilled` event for each order accordingly.
    function fillOrdersExactEth(
        address asset,
        uint256 minimumAsset,
        uint256[] memory orderIds
    ) external payable;

    /// @notice Consecutively fills an array of orders identified by the asset-specific `orderId` of each order,
    /// by providing a possibly surplus amount of ETH and requesting an exact amount of asset to receive.
    /// @dev Transfers portions of msg.value to the orders' sellers according to the price.
    /// The sum of asset amounts of filled orders MUST be exactly `assetAmount`. Excess ETH MUST be returned back to `msg.sender`.
    /// MUST decrease the `assetAmount` parameter for the specified order according to how much of it was filled,
    /// and transfer that amount of the order's `asset` with ID `assetId` to the buyer.
    /// MUST emit `OrderFilled` event for each order accordingly.
    /// If msg.value is more than the sum of orders' prices, it MUST refund the difference back to `msg.sender`.
    function fillOrdersExactAsset(
        address asset,
        uint256 assetAmount,
        uint256[] memory orderIds
    ) external payable;

    /// @notice Cancels the sell order of `asset` with asset-specific `orderId`, transferring the order's assets back to the seller.
    /// @dev MUST revert if the order's seller is not `msg.sender`.
    /// MUST change the `assetAmount` parameter for the specified order to `0`.
    /// MUST emit `OrderCancelled` event.
    /// MUST transfer the order's `assetAmount` of `asset` with `assetId` back to the seller.
    function cancelSellOrder(address asset, uint256 orderId) external;

    /// @notice Cancels a batch of sell orders of `asset` with asset-specific `orderIds`, transferring the orders' assets back to the seller.
    /// @dev This is a batched version of `cancelSellOrder` that simply iterates through the array to call said function.
    function cancelBatchSellOrder(address asset, uint256[] memory orderIds) external;

    /// @notice Withdraws the contract balance (containing collected fees) to the owner. Executable only by the owner.
    /// @dev MUST transfer the entire contract balance to the owner.
    /// MUST revert if called by unauthorized account.
    /// MUST emit `FeesWithdrawn` event.
    function withdrawFees() external;
}
```

## Rationale

It's not an AMM. In a typical AMM dex, there are liquidity pools and you trade against these pools. This is unsuitable for this use-case because a contract on base chain has no means of knowing the state of the game chain so the contract cannot automate market making. Instead, buyers have to be explicitely buying only those tokens that they deem to be valid.

### The idea:

1. User withdraws the game asset to the base chain by using the PRC-5 Inverse Projection system - they lock the appropriate amount of the asset and the game chain assigns it the address-specific incremental `orderId`, for which they mint InverseProjected1155 tokens on the base chain.
2. Smart contract on base chain facilitates trading game asset. This contract allows the following:
   - Function to create a sell order of specified amount of asset and price per one unit of said asset. A sell order persists and has the following properties:
     - `uint256 assetId` - token ID of the ERC1155 asset being sold,
     - `uint256 assetAmount` - amount of the game asset the seller is selling,
     - `uint256 pricePerAsset` - the price in native gas tokens of the base chain the seller is requesting,
     - `address seller` - the seller's address.
     - `uint256 makerFee` - maker fee expressed in basis points, depicting the ratio of payment tokens that will be deducted from the payment to the seller
     - `uint256 takerFee` - taker fee expressed in basis points, depicting the ratio of payment tokens that are added to the purchase cost of the order
     - `uint256 creationFeePaid` - fee paid by the seller when creating the order, refunded if sell order is cancelled
       - This fee exists to deter from an attack of creating thousands of extremely small sell orders, which would cost large amounts of gas to fill.
   - Function to cancel a sell order of specified `orderId`
   - Batch functions `createBatchSellOrder` and `cancelBatchSellOrder` of the abovementioned functions
   - Function to fill sell orders (in other words - buy). There is no "buy order" that persists, and rather the buy function directly executes the sell order fill and transfers the value specified by price defined in the orders to the contract and attributing it to the sellers balance mapping. There are 2 variations of fill function:
     - `fillOrdersExactEth` - consecutively fills the array of specified orders until all provided ETH is spent and a specified minimum amount of asset has been achieved. Example: You want to buy as much asset as possible for 1 ETH.
     - `fillOrdersExactAsset` - consecutively fills the array of specified orders until specified exact amount of asset has been achieved, and returns the excess ETH back to the sender. Example: You want to buy 1000 units of asset as cheaply as possible.

The DEX contract on the base chain only facilitates the trading (transferring) of existing Inverse Projected ERC1155 assets, it does not make any assurances about validity of such assets. That aspect is handled by the feature set of the Inverse Projected ERC1155 standard itself. The responsibility of querying the API of game chain to check the validity of the sell order is left up to the front-end providers, as well as the presentation of the valid sell orders.

### Claim-based system of payments

When sell orders are filled, payments attributed to the sellers are **not** transferred directly in the fill transaction, but rather a balance value in the contract is updated for each seller. A seller can then claim their balance via the `claim` function and get their balance transferred to them.

This was done because if there were direct transfers to the sellers in the fill transaction, there would be two problems with that:

1. When transferring eth to the seller, all gas is forwarded. A malicious smart contract seller could be consuming large amounts of gas for arbitrary execution when receiving eth.
2. Any one of those transfers can fail, and that would revert the transaction. Why the transfer would fail - for example somebody might make a malicious smart contract that always fails on receiving eth and create sell orders with that contract.

There are some other options on how to fix the:

- first issue:
  - Set gas limit to some value, possibly calculated to result in the sell order creation fee
    - Not great because there might be a legit reason why seller should consume gas over the limit (smart contract wallet, any smart contracts built on top of the DEX)
- second issue:
  - Don't revert on transfer fail, ignore sell order, return payment to buyer
    - Not great because an always-reverting (but good priced) sell order would end up being tried to fill over and over by multiple fill requests, unnecessarily wasting gas of buyers
  - Don't revert on transfer fail, cancel the sell order, return payment to buyer
    - Might potentially be annoying to have sell order cancelled because of a transfer fail

These fixes are suboptimal in comparison with the fix of adopting a claim-based system. Admittedly, it results in a slightly worse UX for the sellers because they are forced to do one more transaction to get their payment, but the pros dramatically outweight this one con.

## Game Node Dex API (OPTIONAL)

These endpoints are provided by the game node to allow external sites generate a frontend for the DEX.

1. Get Game Assets and Metadata.

   `GET /dex/`

   RESPONSE

   ```js
   {
       assets: {
           asset: string;           // Asset Code
           name?: string;           // OPTIONAL Asset Name
           description?: string;    // OPTIONAL Asset Description
           fromSym: string;         // Name of base Asset
           toSym: string;           // Name of unit to convert
           contractAsset: string;   // Contract Address for Asset (IERC1155)
           contractDex: string;     // Contract Address for Dex (OrderbookDex)
           contractChain: string;   // CAIP2 Chain Identifier
           image?: string;          // OPTIONAL Asset URL Image (1:1 200px Image)
       }[],
       game: {
           id: string;              // Game ID
           name?: string;           // Optional Game Name
           version?: string;        // Optional Game Version
       }
   }
   ```

   RESPONSE Example

   ```js
   {
       assets: [{
           asset: 'gold',
           name: 'Game Gold',
           description: 'Purchase Items with GG',
           fromSym: 'GG',
           toSym: 'ETH',
           contractAsset: '0x1111',
           contractDex: '0xaaaa',
           contractChain: 'eip155:1',
           image: 'https://game-assets/gg.png'
       }, {
           asset: 'silver',
           name 'Game Silver',
           description: 'Purchase Magic with GS',
           fromSym: 'GS',
           toSym: 'ETH',
           contractAsset: '0x2222',
           contractDex: '0xbbbb',
           contractChain: 'eip155:42161',
           image: 'https://game-assets/gs.png'
       }],
       game: {
           id: 'my-game',
           name: 'My Game',
           version: '1.0.0'
       }
   }

   ```

2. Get Asset information.

   `GET /dex/{asset}`

   - asset: valid name for specific game asset token.

   RESPONSE

   ```js
   {
       asset: string;           // Asset Code
       name?: string;           // OPTIONAL Asset Name
       description?: string;    // OPTIONAL Asset Description
       fromSym: string;         // Name of base Asset
       toSym: string;           // Name of unit to convert
       contractAsset: string;   // Contract Address for Asset (IERC1155)
       contractDex: string;     // Contract Address for Dex (OrderbookDex)
       contractChain: string;   // CAIP2 Chain Identifier
       image?: string;          // OPTIONAL Asset URL Image (1:1 200px Image)
       totalSupply: number;     // Total number of assets
   }
   ```

3. Get ERC1155 tokens of `asset` for the specified `wallet` that have been minted or owned in a valid way. This is used by the DEX to get the list of valid assets that user is able to create sell orders for.

   `GET /dex/{asset}/wallet/{wallet}`

   - asset: valid name for specific game asset token.
   - wallet: wallet to query for asset

   RESPONSE

   ```js
   {
     total: number; // Total number of assets owned
     stats: {
       tokenId: number; // ERC1155 Token ID
       amount: number; // Number of assets owned
     }
     [];
   }
   ```

4. Gets valid created Sell Orders. This is used by the DEX to get the list of valid Sell Orders to display to users wanting to buy. Ordered by lowest price.

   `GET /dex/{asset}/orders?seller=wallet&page=number&limit=number`

   - asset: valid name for specific game asset token.
   - seller (OPTIONAL): fetch where wallet address matches wallet
   - page (OPTIONAL): results page number, default = 1
   - limit (OPTIONAL): 10, 25, 50, 100, default = 25

   RESPONSE

   ```js
   {
     stats: {
       orderId: number; // Order unique ID
       seller: string; // Seller wallet
       tokenId: number; // ERC1155 TokenID
       amount: number; // Number of assets for sale
       price: string; // Price per asset
       asset: string; // Asset address
       makerFee: number; // Maker Fee
       takerFee: number; // Taker Fee
     }
     [];
   }
   ```

5. Get Asset Historical data. Allows the UI to draw a chart with historical values.

   `GET /dex/{asset}/historical_price?freq=string&start=number&end=number`

   - asset: valid name for specific game asset token.
   - freq (OPTIONAL): hour | day | month - range for specific (default: hour)
   - start (OPTIONAL): start range unix time
   - end (OPTIONAL): end range unix time

   If start is not defined, 5, 30, 365 days ago are used as defaults.  
   If end is not defined, now is used.  
   NOTES:  
   Data is limited to 170 data points per query (1 week of data per hour)  
   If data points miss then previous data point is still valid (no changes)

   RESPONSE

   ```js
   {
     timeFrom: number; // First data point date
     timeTo: number; // Last data point date
     data: {
       time: number; // Time start date for data point
       high: number; // Max price for range
       low: number; // Min price for range
       open: number; // Start price for range
       close: number; // End price for range
       volumeFrom: number; // Total Supply of Assets (at `time`) in fromSym Units
       volumeTo: number; // Total Supply of Assets (at `time`) in toSym Units
     }
     [];
   }
   ```

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
