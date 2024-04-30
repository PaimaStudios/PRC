---
title: Paima Achievement Interface
description: Interface for sharing in-game achievements.
author: seba@paimastudios.com, edward@paimastudios.com, tad@paimastudios.com
status: open
created: 2023-11-08
---

## Abstract

An open cross-game standard achievement specification to gamify on-chain participation. 
This specification allows interaction through standard HTTP methods.  

## Motivation

Most games have an achievement system, but they are not compatible: vendors and tools have to be adapted to each game. By implementing this standard, developers spend no or less time implementing the format itself, vendors can use the format for gamification apps, explorers, prizes, etc.  This open-specification does not depend on any specific platform game host, technology, or language, and can be completely self-hosted. 

The achievement content is easily indexable for games and API consumers, in a recognizable format, allowing caching and generating useful tools for end users. 

This achievement system can be used by the target game itself for unlocking functionalities such as opening new areas, triggering game progress, or giving away prizes. Third parties may also consume API to expand on its actions, which are compatible with on-chain games.

## Format

### HTTP
* Network calls are done to any game node via HTTP to `BASE_URL`
* All network requests are `Method GET`
* All responses ContentType is `application/json`
* `Standard HTTP codes` are used for status. E.g., 200 OK, 500 Internal Server Error, 404 Not Found, etc.  
* Request Accept-Language header with RFC 7231 content may be used to request the content in a specific language.
* Response Content-Language header with RFC 7231 shall be used to inform the client of the language of the content. 

### Response Interfaces

General Game Info
```
interface Game = {
  id: string             // Game ID
  name?: string          // Optional Game Name
  version?: string       // Optional Game Version
}
```

Data Validity
```
interface Validity = {
  block: number;         // Data Block height (0 always valid)
  caip2: string;         // CAIP-2 blockchain identifier 
  time: string;          // Optional Date ISO8601 YYYY-MM-DDTHH:mm:ss.sssZ
}
```

Player Info
``` 
interface Player = {     
  wallet:                // e.g., addr1234... or 0x1234..,
  walletType?: 'cardano' | 'evm' | 'polkadot' | 'algorand' | string  // (Optional) Wallet-type
  userId?: string;       // (Optional) User ID for a specific player account.
                         // This value should be immutable and define a specific account,
                         // as the wallet might be migrated or updated.
  userName?: string;     // (Optional) Player Display Name
}
```

## Specification

### Get All Available Achievements
`{BASE_URL}/achievements/public/list`

Optional: Subset of achievements by category
`{BASE_URL}/achievements/public/list?category=Silver`

Optional: Subset of active achievements
`{BASE_URL}/achievements/public/list?isActive=true`


```
interface AchievementPublicList extends Game, Validity {
    achievements: {
      name: string;                     // Unique Achievement String
      score?: number;                   // Optional: Relative Value of the Achievement
      category?: string;                // Optional: 'Gold' | 'Diamond' | 'Beginner' | 'Advanced' | 'Vendor'
      percentCompleted?: number         // Percent of players that have unlocked the achievement 
      isActive: boolean                 // If achievement can be unlocked at the time. 
      displayName: string;              // Achievement Display Name
      description: string               // Achievement Description
      spoiler?: 'all' | 'description';  // Hide entire achievement or description if not completed
      iconURI?: string;                 // Optional Icon for Achievement
      iconGreyURI?: string;             // Optional Icon for locked Achievement
      startDate?: string                // Optional Date ISO8601 YYYY-MM-DDTHH:mm:ss.sssZ
      endDate?: string                  // Optional Date ISO8601 YYYY-MM-DDTHH:mm:ss.sssZ
    }[];
}
```

### Get completed Achievements for Wallet or Token
`{BASE_URL}/achievements/wallet/:wallet`
* wallet: Wallet address

`{BASE_URL}/achievements/erc/:erc/:cde/:token_id`  
`{BASE_URL}/achievements/erc/:erc/:cde/:token_id/:wallet_id`  

e.g., /achievements/erc/erc1155/0/10/
or /achievements/erc/erc721/1/20/0x1234
* erc: Any supported ERC standard by the game, such as erc721, erc6551, or erc1155.
* cde: Chain Data Extension identifier, might be used by the game to track identify the contract
* token_id: Unique token id defined by the ERC standard
* wallet: Optional Query param for ERCs that supports pairs as (token, wallet) as erc1155 or filter by ownership. 

Optional subset of achievements by name  
`{BASE_URL}/achievements/wallet/:wallet?name=start_game,end_game,defeat_red_dragon`  

```
interface PlayerAchievements extends Validity, Player {
  completed: number;                    // Total number of completed achievements for the game
  achievements: {
    name: string;                       // Unique Achievement String
    completed: boolean;                 // Is Achievement completed
    completedDate?: Date;               // Completed Date ISO8601 YYYY-MM-DDTHH:mm:ss.sssZ
    completedRate?: {                   // If achievement has incremental progress
      progress: number,                 // Current Progress
      total: number                     // Total Progress
    }
  }[];
}
```

### Reference implementation


### Copyright

