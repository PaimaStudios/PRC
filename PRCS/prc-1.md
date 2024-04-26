---
title: Paima Achievement Interface
description: Interface for sharing in-game achievements.
author: seba@paimastudios.com, edward@paimastudios.com
status: open
created: 2023-11-08
---

## Abstract

A open cross-game standard achievement specification to gamify onchain participation. 
This specification allows to interact through standard http methods.  

## Motivation

Most games have a achievement system, but there are not compatible: vendors and tools have adapt to each game. By implementing this standard, developer spend no or less time implementing the format itself, vendors can use the format for gamification apps, explorers, prizes, etc.

## Format

### HTTP
* Network calls are done to any game node via HTTP to `BASE_URL`
* All network requests are `Method GET`
* All responses ContentType is `application/json`
* `Standard HTTP codes` are used for status. E.g., 200 OK, 500 Internal Server Error, 404 Not Found, etc.  

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
  chainId: number;       // Data ChainId
  time: string;          // Optional Date ISO8601 YYYY-MM-DDTHH:mm:ss.sssZ
}
```

Player Info
``` 
interface Player = {     
  wallet:                // e.g., addr1234... or 0x1234..,
  walletType?: 'cardano' | 'evm' | 'polkadot' | 'algorand' | string  // Optional wallet-type
  userId?: string;       // If data for specific user: e.g., "1", "player-1", "unique-name", etc
  userName?: string;     // Player Display Name
}
```

## Specification

### Get all available achievements:  
`{BASE_URL}/achievements/public/list`
```
interface AchievementPublicList extends Game, Validity {
    achievements: {
      name: string;                     // Unique Achievement String
      score?: number;                   // Optional: Relative Value of the Achievement
      category?: string;                // Optional: 'Gold' | 'Diamond' | 'Beginner' | 'Advanced' | 'Vendor'
      percentCompleted?: number         // Percent of players that have unlocked achievement 
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

### Get completed Achievements  
`{BASE_URL}/achievements/wallet/:wallet`  
`{BASE_URL}/achievements/nft/:nft_address`

Optional subset of achievements by name  
`{BASE_URL}/achievements/wallet/:wallet?name=start_game,end_game,defeat_red_dragon`  
```
interface PlayerAchievements extends Validity, Player {
  completed: number;                    // Total number of completed achievements for game
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

