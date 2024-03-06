---
title: Paima Hololocker Interface
description: Interface for projecting ERC721 tokens on EVM networks for usage in Paima
author: matejos (@matejos)
status: Draft
created: 2023-11-08
---

## Abstract

The following standard allows for the implementation of a standard API for projecting ERC721 tokens on EVM networks for usage in Paima applications. This standard provides basic functionality to lock tokens, request to unlock locked tokens, and withdraw tokens after certain time has passed since the unlock request.

## Motivation

Many games, due to being data and computation heavy applications, run on sidechains, L2s and appchains as opposed to popular L1 blockchains. This is problematic because popular NFT collections (which people generally want to use in-game) live on the L1 (a different environment). A common solution to this problem is building an NFT bridge, but bridges not only have a bad reputation for fungible tokens which limits usage, the problem is even worse for NFTs where there is also a philosophical disconnect (if a bridge gets hacked, which is the canonical NFT? The one the hacker stole, or the bridged asset?)

Instead of bridging NFTs, this standard encourages users to project their NFT directly into the game, allowing them to access their asset in-game without having to bridge it to the game chain. Although the main use-case is projecting a single NFT, it supports projecting multiple NFTs at once as well.

## Specification

Every PRC-2 compliant contract must implement the `HololockerInterface` interface:

```solidity
interface HololockerInterface is IERC721Receiver {
    /// @dev Data structure that must exist for each locked NFT
    struct LockInfo {
        /// Timestamp when NFT will be withdrawable, 0 if unlock hasn't been requested
        uint256 unlockTime;
        /// Rightful owner of the NFT
        address owner;
        /// Account that initiated the lock
        address operator;
    }

    /// @dev This emits when NFT is locked, either via lock function or via NFT being sent to this contract
    /// @param token NFT address
    /// @param owner Rightful owner of the NFT
    /// @param tokenId NFT token identifier
    /// @param operator Address initiating the lock
    event Lock(address indexed token, address indexed owner, uint256 tokenId, address operator);

    /// @dev This emits when NFT is requested to unlock.
    /// @param token NFT address
    /// @param owner Rightful owner of the NFT
    /// @param tokenId NFT token identifier
    /// @param operator Address initiating the unlock request
    /// @param unlockTime Timestamp when NFT will be withdrawable.
    event Unlock(address indexed token, address indexed owner, uint256 tokenId, address operator, uint256 unlockTime);

    /// @dev This emits when NFT is withdrawn.
    /// @param token NFT address
    /// @param owner Rightful owner of the NFT
    /// @param tokenId NFT token identifier
    /// @param operator Address initiating the withdraw
    event Withdraw(address indexed token, address indexed owner, uint256 tokenId, address operator);

    /// @dev This emits when lockTime value changes.
    /// @param newValue New lockTime value
    event LockTimeUpdate(uint256 newValue);

    /// @notice Returns `LockInfo` for specified `token => tokenId`
    /// @param token NFT tokens contract address
    /// @param tokenId NFT tokens identifier
    /// @return The `LockInfo` struct information
    function getLockInfo(address token, uint256 tokenId) external view returns (LockInfo memory);

    /// @notice Initiates a lock for one or more NFTs
    /// @dev Reverts if `tokens` length is not equal to `tokenIds` length.
    /// Stores a `LockInfo` struct `{owner: owner, operator: msg.sender, unlockTime: 0}` for each `token => tokenId`
    /// Emits `Lock` event.
    /// Transfers each token:tokenId to this contract.
    /// @param tokens NFT tokens contract addresses
    /// @param tokenIds NFT tokens identifiers
    /// @param owner NFT tokens owner
    function lock(address[] memory tokens, uint256[] memory tokenIds, address owner) external;

    /// @notice Requests unlock for one or more NFTs
    /// @dev Reverts if `tokens` length is not equal to `tokenIds` length.
    /// Reverts if msg.sender is neither `owner` nor `operator` of LockInfo struct for
    /// any of the input tokens.
    /// Reverts if `unlockTime` of LockInfo struct for any of the input tokens is not 0.
    /// Modifies a `LockInfo` struct `{unlockTime: block.timestamp + lockTime}` for each `token => tokenId`
    /// Emits `Unlock` event.
    /// @param tokens NFT tokens contract addresses
    /// @param tokenIds NFT tokens identifiers
    function requestUnlock(address[] memory tokens, uint256[] memory tokenIds) external;

    /// @notice Withdraws one or more NFTs to their rightful owner
    /// @dev Reverts if `tokens` length is not equal to `tokenIds` length.
    /// Reverts if msg.sender is neither `owner` nor `operator` of LockInfo struct for
    /// any of the input tokens.
    /// Reverts if `unlockTime` of LockInfo struct for any of the input tokens is
    /// either 0 or greater than block.timestamp.
    /// Modifies a `LockInfo` struct `{unlockTime: block.timestamp + lockTime}` for each `token => tokenId`
    /// Emits `Unlock` event.
    /// @param tokens NFT tokens contract addresses
    /// @param tokenIds NFT tokens identifiers
    function withdraw(address[] memory tokens, uint256[] memory tokenIds) external;

    /// @notice Returns `lockTime`, which is the value that gets added to block.timestamp and saved as unlockTime
    /// in the requestUnlock function.
    /// @return The `lockTime` variable
    function getLockTime() external view returns (uint256);

    /// @notice Changes the `lockTime` variable.
    /// @dev This function should be protected with appropriate access control mechanisms.
    /// The new value should be checked against a sane upper limit constant, which if exceeded,
    /// should cause a revert.
    /// Emits `LockTimeUpdate` event.
    /// @param newLockTime New lockTime value
    function setLockTime(uint256 newLockTime) external;
}
```

A Hololocker implementation MUST implement the IERC721Receiver interface to be able to receive ERC721 assets via `IERC721.safeTransferFrom`.
It MUST initialize a lock similarly as in the `lock` function, and it MUST emit the `Lock` event

```solidity
interface IERC721Receiver {
    /// @dev Whenever an {IERC721} `tokenId` token is transferred to this contract via {IERC721-safeTransferFrom}
    ///  by `operator` from `from`, this function is called.
    ///  It must return its Solidity selector to confirm the token transfer.
    ///  If any other value is returned or the interface is not implemented by the recipient, the transfer will be
    ///  reverted.
    ///  The selector can be obtained in Solidity with `IERC721Receiver.onERC721Received.selector`.
    ///  Note: the contract address is always the message sender.
    /// @param _operator The address which called `safeTransferFrom` function
    /// @param _from The address which previously owned the token
    /// @param _tokenId The NFT identifier which is being transferred
    /// @param _data Additional data with no specified format
    /// @return `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`
    ///  unless throwing
    function onERC721Received(address _operator, address _from, uint256 _tokenId, bytes _data) external returns(bytes4);
}
```

## Rationale

The Hololocker contract is in its core a NFT staking contract. While there have been attempts at EIPs on this matter, such as [EIP-4987](https://eips.ethereum.org/EIPS/eip-4987), none gained enough traction and approval to be finalized.
This proposal therefore proposes a standard for light-weight implementation of such mechanism.

The main functions accept an array of token addresses and token IDs to be able to accommodate manipulations with multiple NFTs in one transaction, to save on transaction costs.

The standard user interaction flow is: Lock -> Request Unlock -> (after `lockTime` has passed from the unlock request) Withdraw

### Why Unlocking phase / lockTime ?

If we did not have a special "Unlocking" state that requires users to wait for finality on the L1 before withdrawing their NFT, it could cause a situation where the same NFT is used in two places at once.
Therefore, there must be a period of Unlocking state, initiated by the request for unlock and lasting the amount of seconds specified in the `lockTime` variable. This `lockTime` delay should reflect the finality period of the chain. Only after this delay has passed, the NFT can be withdrawn.

### Consistent contract address

To facilitate a more reliable experience for users, the Hololocker contract should be deployed to the same address across all EVM chains. This can be done by using a deployment proxy. Our reference implementation does this by specifying a salt `bytes32(uint256(1))` in the [Foundry deployment script](https://github.com/dcSpark/projected-nft-whirlpool/blob/8b0d0367139eb9a43be94edff34a656258e25793/evm/script/Deploy.s.sol).  
Hololocker is currently deployed at `0x963ba25745aEE135EdCFC2d992D5A939d42738B6`.

### NFT locking UX

Upon locking an NFT in an application that integrates the Hololocker, it might happen that users will get confused as to where did their NFTs go. An option to mint a sort of a "receipt NFT" in exchange for the locked NFT was thought of, but it does not sufficiently solve the inconvenience and would greatly increase the complexity and transactional costs. It would roughly double the gas cost of `lock` operation (85k gas units -> 161k gas units) and increase the cost of `withdraw` operation by 60% (7564 gas units -> 12125 gas units). Therefore, this approach is not recommended.

### Projecting other standards (ERC20, ERC1155...)

Should there be a need for supporting other token standards than ERC721, this interface can be appropriately modified to accommodate for the differences between the standards' methods of token identification. For example: To support ERC20, instead of dealing with token IDs, you'd be dealing with token amounts. That would mean changing the `LockInfo` struct, the events, and the function parameters slightly.

## Reference Implementation

https://github.com/dcSpark/projected-nft-whirlpool/blob/8b0d0367139eb9a43be94edff34a656258e25793/evm/src/Hololocker.sol

## Security Considerations

It is useful to note, that since NFTs can be locked also via transferring them to Hololocker with `ERC721.safeTransferFrom` function, it is imperative to instruct potential developers integrating Hololocker in their applications that they MUST use the `safe` variant of the transfer function. Using basic `ERC721.transferFrom` function to transfer NFT to the Hololocker contract will lead to that NFT being permanently stuck in the contract, since in that case there is no mechanism to be able to store the previous owner of the NFT. This is a caveat of the ERC721 standard.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
