---
title: Paima Achievement Interface
description: Interface for sharing in-game achievements.
author: seba@dcspark.io, edward@dcspark.io
status: open
created: 2023-11-08
---

## Specification

`/achievements/public/list`
```
interface AchievementPublicList {
    achievements: {
      name: string;         // Unique String
      displayName: string;  // Display Name
      details: string       // Display Information
      spoiler: boolean;     // Hide to not spoiler player
      iconURI: string;
      iconGreyURI?: string; // Optional
    }[];
}
```

`/achievements/wallet/:wallet`
or 
`/achievements/nft/:nft_address`

```
interface PlayerAchievements {
  wallet: string;
  completed: number;    // number of completed achievements for game
  achievements: {
    name: string;        // Unique String
    completed: boolean;  // Is achievement completed
    completedDate?: Date // If completed, when
    completedRate: { progress: number, total: number } // If partially completed;
  }[];
}
```

