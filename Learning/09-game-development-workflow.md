# 第九章：遊戲開發流程 (Book Events)

## 學習目標

完成本章後,你將能夠:
- ✅ 深入理解 Book 和 BookEvent 的設計
- ✅ 掌握開發新 BookEvent 的完整流程
- ✅ 運用測試驅動開發模式
- ✅ 實現常見遊戲機制

---

## 9.1 Book 和 BookEvent 深入

### 9.1.1 什麼是 Book?

**Book** 是 RGS (Remote Game Server) 返回的遊戲結果:

```typescript
interface Book {
  id: number;                    // Book ID
  payoutMultiplier: number;      // 賠付倍數
  events: BookEvent[];           // 事件序列
  baseGameWins: number;          // 基礎遊戲獲勝
  freeGameWins: number;          // 免費遊戲獲勝
  criteria?: string;             // 其他標準
}
```

**Book 的本質:**
- 📜 完整的遊戲劇本
- 🎬 按順序執行的事件
- 🎲 由數學模型生成
- 🔒 不可修改,確保公平

### 9.1.2 BookEvent 的類型

```typescript
type BookEvent = 
  | BookEventReveal      // 顯示結果
  | BookEventWin         // 獲勝動畫
  | BookEventBonus       // 觸發 Bonus
  | BookEventFreeSpins   // 免費旋轉
  | BookEventMultiplier  // 倍數變化
  | ...;
```

**設計原則:**
- 每個 BookEvent 代表一個原子操作
- BookEvent 之間不應有依賴
- 順序決定遊戲流程

---

## 9.2 開發新 BookEvent 的完整流程

### 9.2.1 第一步：定義 BookEvent 類型

假設我們要添加一個 `cascadeWin` BookEvent:

```typescript
// apps/lines/src/game/typesBookEvent.ts

type BookEventCascadeWin = {
  index: number;
  type: 'cascadeWin';
  explodingSymbols: [number, number][];  // 爆炸的符號位置
  newSymbols: RawSymbol[][];             // 新掉落的符號
  cascadeMultiplier: number;             // 連鎖倍數
  winAmount: number;                     // 獲勝金額
};

export type BookEvent =
  | BookEventReveal
  | BookEventWin
  | BookEventCascadeWin  // ← 添加新類型
  | ...;
```

### 9.2.2 第二步：準備測試數據

```typescript
// apps/lines/src/stories/data/base_events.ts

export default {
  // ... 其他 events
  cascadeWin: {
    index: 5,
    type: 'cascadeWin',
    explodingSymbols: [
      [0, 0], [0, 1], [0, 2],
      [1, 0], [1, 1],
    ],
    newSymbols: [
      [{ name: 'H1' }, { name: 'H2' }, { name: 'H3' }],
      [{ name: 'L1' }, { name: 'L2' }],
    ],
    cascadeMultiplier: 2,
    winAmount: 500,
  },
};
```

### 9.2.3 第三步：創建 Storybook Story

```svelte
<!-- apps/lines/src/stories/ModeBaseBookEvent.stories.svelte -->

<script lang="ts">
  import { Story } from '@storybook/addon-svelte-csf';
  import Game from '../components/Game.svelte';
  import { setContext } from '../game/context';
  import { playBookEvent } from 'utils-book';
  import events from './data/base_events';
  
  setContext();
</script>

<Story
  name="cascadeWin"
  args={{
    skipLoadingScreen: true,
    data: events.cascadeWin,
    action: async (data) => {
      await playBookEvent(data, { bookEvents: [] });
    },
  }}
/>
```

### 9.2.4 第四步：定義 EmitterEvents

思考需要哪些 emitterEvents 來實現這個功能:

```typescript
// apps/lines/src/components/Board.svelte

export type EmitterEventBoard =
  | ...
  | { type: 'cascadeInit'; newSymbols: RawSymbol[][] }
  | { type: 'cascadeExplode'; positions: [number, number][] }
  | { type: 'cascadeRemove' }
  | { type: 'cascadeDrop' }
  | { type: 'cascadeSettle' };
```

### 9.2.5 第五步：實作 BookEventHandler

```typescript
// apps/lines/src/game/bookEventHandlerMap.ts

export const bookEventHandlerMap = {
  // ... 其他 handlers
  
  cascadeWin: async (bookEvent: BookEventOfType<'cascadeWin'>) => {
    // 1. 初始化新符號
    eventEmitter.broadcast({
      type: 'cascadeInit',
      newSymbols: bookEvent.newSymbols,
    });
    
    // 2. 播放爆炸動畫 (異步)
    await eventEmitter.broadcastAsync({
      type: 'cascadeExplode',
      positions: bookEvent.explodingSymbols,
    });
    
    // 3. 移除爆炸的符號
    eventEmitter.broadcast({
      type: 'cascadeRemove',
    });
    
    // 4. 播放掉落動畫 (異步)
    await eventEmitter.broadcastAsync({
      type: 'cascadeDrop',
    });
    
    // 5. 穩定面板
    eventEmitter.broadcast({
      type: 'cascadeSettle',
    });
    
    // 6. 更新倍數和金額
    eventEmitter.broadcast({
      type: 'updateMultiplier',
      multiplier: bookEvent.cascadeMultiplier,
    });
    
    eventEmitter.broadcast({
      type: 'updateWin',
      amount: bookEvent.winAmount,
    });
  },
};
```

### 9.2.6 第六步：在組件中處理 EmitterEvents

```svelte
<!-- apps/lines/src/components/Board.svelte -->

<script lang="ts">
  let cascadeSymbols = $state<RawSymbol[][]>([]);
  let explodingPositions = $state<[number, number][]>([]);
  
  context.eventEmitter.subscribeOnMount({
    cascadeInit: (event) => {
      cascadeSymbols = event.newSymbols;
    },
    
    cascadeExplode: async (event) => {
      explodingPositions = event.positions;
      
      // 播放爆炸動畫
      await Promise.all(
        event.positions.map(async ([x, y]) => {
          await playExplosionAnimation(x, y);
        })
      );
    },
    
    cascadeRemove: () => {
      // 移除爆炸的符號
      explodingPositions.forEach(([x, y]) => {
        board[x][y] = null;
      });
      explodingPositions = [];
    },
    
    cascadeDrop: async () => {
      // 播放掉落動畫
      await animateSymbolsDrop(cascadeSymbols);
    },
    
    cascadeSettle: () => {
      // 更新面板
      board = mergeCascadeSymbols(board, cascadeSymbols);
      cascadeSymbols = [];
    },
  });
</script>
```

### 9.2.7 第七步：測試

1. **在 Storybook 中測試**
   - 運行 `pnpm run storybook --filter=lines`
   - 找到 `MODE_BASE/bookEvent/cascadeWin`
   - 點擊 Action 按鈕
   - 確認動畫正確

2. **測試邊緣情況**
   - 空的 explodingSymbols
   - 單個符號爆炸
   - 全部符號爆炸

3. **整合測試**
   - 添加到完整的 book 中
   - 測試與其他 bookEvents 的配合

### 🎯 實作練習 9.1

按照上述流程,實作一個 `multiplyWin` BookEvent:

```typescript
type BookEventMultiplyWin = {
  index: number;
  type: 'multiplyWin';
  originalWin: number;
  multiplier: number;
  finalWin: number;
};
```

要求:
- 顯示原始獲勝金額
- 播放倍數增加動畫
- 顯示最終金額
- 在 Storybook 中測試

---

## 9.3 常見 BookEvent 模式

### 9.3.1 Reveal 模式

顯示遊戲結果:

```typescript
reveal: async (bookEvent: BookEventOfType<'reveal'>) => {
  // 設置面板
  eventEmitter.broadcast({
    type: 'boardSetSymbols',
    board: bookEvent.board,
  });
  
  // 播放揭示動畫
  await eventEmitter.broadcastAsync({
    type: 'reelSpin',
  });
  
  // 停止
  eventEmitter.broadcast({
    type: 'reelStop',
  });
},
```

### 9.3.2 Win 模式

顯示獲勝:

```typescript
win: async (bookEvent: BookEventOfType<'win'>) => {
  // 顯示獲勝線
  eventEmitter.broadcast({
    type: 'winLinesShow',
    lines: bookEvent.lines,
  });
  
  // 播放獲勝動畫
  await eventEmitter.broadcastAsync({
    type: 'winAnimation',
  });
  
  // 更新金額
  eventEmitter.broadcast({
    type: 'updateWin',
    amount: bookEvent.amount,
  });
},
```

### 9.3.3 Bonus 觸發模式

```typescript
triggerBonus: async (bookEvent: BookEventOfType<'triggerBonus'>) => {
  // 顯示觸發符號
  eventEmitter.broadcast({
    type: 'highlightBonusSymbols',
    positions: bookEvent.triggerPositions,
  });
  
  // 播放觸發動畫
  await eventEmitter.broadcastAsync({
    type: 'bonusTriggerAnimation',
  });
  
  // 切換到 Bonus 模式
  eventEmitter.broadcast({
    type: 'switchToBonus',
    bonusType: bookEvent.bonusType,
  });
},
```

---

## 9.4 狀態管理

### 9.4.1 遊戲狀態

使用 `$state` 管理組件狀態:

```svelte
<script lang="ts">
  // Board 狀態
  let board = $state<RawSymbol[][]>([]);
  let isSpinning = $state(false);
  let winningPositions = $state<[number, number][]>([]);
  
  // 動畫狀態
  let animationQueue = $state<Animation[]>([]);
  let currentAnimation = $state<Animation | null>(null);
</script>
```

### 9.4.2 全局遊戲狀態

查看 [apps/lines/src/game/stateGame.ts](../apps/lines/src/game/stateGame.ts):

```typescript
export const stateGame = $state({
  mode: 'base' as 'base' | 'bonus',
  currentBook: null as Book | null,
  currentEventIndex: 0,
  
  // Base game state
  baseBoard: [] as RawSymbol[][],
  baseWinLines: [] as WinLine[],
  
  // Bonus game state
  bonusBoard: [] as RawSymbol[][],
  freeSpinsRemaining: 0,
  freeSpinsTotal: 0,
});

export const stateGameDerived = {
  currentBoard: () => {
    return stateGame.mode === 'base' 
      ? stateGame.baseBoard 
      : stateGame.bonusBoard;
  },
  
  isInBonus: () => {
    return stateGame.mode === 'bonus';
  },
};
```

---

## 9.5 動畫協調

### 9.5.1 順序動畫

```typescript
await sequence([
  () => animateSymbolIn(symbol1),
  () => animateSymbolIn(symbol2),
  () => animateSymbolIn(symbol3),
]);
```

### 9.5.2 並行動畫

```typescript
await Promise.all([
  animateSymbolIn(symbol1),
  animateSymbolIn(symbol2),
  animateSymbolIn(symbol3),
]);
```

### 9.5.3 延遲動畫

```typescript
for (let i = 0; i < symbols.length; i++) {
  setTimeout(() => {
    animateSymbolIn(symbols[i]);
  }, i * 100); // 每個延遲 100ms
}
```

### 專案實例：Reel 動畫

```typescript
// 轉軸依次停止
for (let i = 0; i < REEL_COUNT; i++) {
  await eventEmitter.broadcastAsync({
    type: 'reelStop',
    reelIndex: i,
  });
  
  // 等待一小段時間
  await new Promise(resolve => setTimeout(resolve, 100));
}
```

---

## 9.6 錯誤處理

### 9.6.1 BookEvent 錯誤

```typescript
export const bookEventHandlerMap = {
  reveal: async (bookEvent: BookEventOfType<'reveal'>) => {
    try {
      // 驗證數據
      if (!bookEvent.board || bookEvent.board.length === 0) {
        throw new Error('Invalid board data');
      }
      
      // 執行邏輯
      await playRevealAnimation(bookEvent.board);
      
    } catch (error) {
      console.error('BookEvent reveal error:', error);
      
      // 廣播錯誤事件
      eventEmitter.broadcast({
        type: 'gameError',
        error: error.message,
      });
      
      // 恢復到安全狀態
      eventEmitter.broadcast({
        type: 'resetToIdle',
      });
    }
  },
};
```

### 9.6.2 動畫錯誤

```svelte
<script lang="ts">
  context.eventEmitter.subscribeOnMount({
    reelSpin: async () => {
      try {
        await animateReelSpin();
      } catch (error) {
        console.error('Reel spin animation error:', error);
        // 強制停止動畫
        stopAllAnimations();
      }
    },
  });
</script>
```

---

## 9.7 性能優化

### 9.7.1 避免不必要的重新渲染

```svelte
<script lang="ts">
  // ❌ 不好：每次都創建新對象
  let position = $derived({ x: symbolX, y: symbolY });
  
  // ✅ 良好：只在值改變時更新
  let x = $state(0);
  let y = $state(0);
</script>

<Sprite {x} {y} />
```

### 9.7.2 批次更新

```typescript
// ❌ 不好：多次廣播
eventEmitter.broadcast({ type: 'updateSymbol', index: [0, 0] });
eventEmitter.broadcast({ type: 'updateSymbol', index: [0, 1] });
eventEmitter.broadcast({ type: 'updateSymbol', index: [0, 2] });

// ✅ 良好：批次更新
eventEmitter.broadcast({
  type: 'updateSymbols',
  indices: [[0, 0], [0, 1], [0, 2]],
});
```

### 9.7.3 動畫性能

```typescript
// 使用 PIXI.Ticker 而非 setTimeout
const ticker = (delta) => {
  sprite.rotation += 0.01 * delta;
};

app.ticker.add(ticker);

// 完成後移除
app.ticker.remove(ticker);
```

---

## 9.8 測試策略

### 9.8.1 單元測試 (Storybook)

```svelte
<!-- 測試單個 emitterEvent -->
<Story name="cascadeExplode"
  args={{
    action: async () => {
      await context.eventEmitter.broadcastAsync({
        type: 'cascadeExplode',
        positions: [[0, 0], [1, 1]],
      });
    },
  }}
/>
```

### 9.8.2 整合測試

```svelte
<!-- 測試完整 bookEvent -->
<Story name="cascadeWin"
  args={{
    action: async () => {
      await playBookEvent(events.cascadeWin, { bookEvents: [] });
    },
  }}
/>
```

### 9.8.3 端到端測試

```svelte
<!-- 測試完整 book -->
<Story name="cascadeWinBook"
  args={{
    action: async () => {
      await playBookEvents(books.cascadeWinBook.events, {
        bookEvents: books.cascadeWinBook.events,
      });
    },
  }}
/>
```

---

## 9.9 實戰：實作瀑布式獲勝

### 📝 完整範例

讓我們從頭到尾實作一個完整的瀑布式獲勝功能:

**第一步：定義數據結構**

```typescript
// typesBookEvent.ts
type BookEventCascade = {
  index: number;
  type: 'cascade';
  cascades: Array<{
    exploding: [number, number][];
    dropping: RawSymbol[][];
    winLines: WinLine[];
    winAmount: number;
  }>;
  totalWin: number;
  cascadeMultipliers: number[];
};
```

**第二步：準備測試數據**

```typescript
// base_events.ts
cascade: {
  index: 6,
  type: 'cascade',
  cascades: [
    {
      exploding: [[0, 0], [0, 1], [1, 0]],
      dropping: [[{ name: 'H1' }], [{ name: 'H2' }]],
      winLines: [{ lineIndex: 0, positions: [...], multiplier: 5 }],
      winAmount: 100,
    },
    {
      exploding: [[0, 0], [1, 0]],
      dropping: [[{ name: 'H3' }], [{ name: 'H4' }]],
      winLines: [{ lineIndex: 1, positions: [...], multiplier: 3 }],
      winAmount: 150,
    },
  ],
  totalWin: 250,
  cascadeMultipliers: [1, 2],
},
```

**第三步：實作 BookEventHandler**

```typescript
// bookEventHandlerMap.ts
cascade: async (bookEvent: BookEventOfType<'cascade'>) => {
  for (let i = 0; i < bookEvent.cascades.length; i++) {
    const cascade = bookEvent.cascades[i];
    const multiplier = bookEvent.cascadeMultipliers[i];
    
    // 1. 顯示獲勝線
    eventEmitter.broadcast({
      type: 'winLinesShow',
      lines: cascade.winLines,
    });
    
    await delay(500);
    
    // 2. 爆炸動畫
    await eventEmitter.broadcastAsync({
      type: 'symbolsExplode',
      positions: cascade.exploding,
    });
    
    // 3. 移除符號
    eventEmitter.broadcast({
      type: 'symbolsRemove',
      positions: cascade.exploding,
    });
    
    // 4. 掉落新符號
    await eventEmitter.broadcastAsync({
      type: 'symbolsDrop',
      symbols: cascade.dropping,
    });
    
    // 5. 更新倍數
    eventEmitter.broadcast({
      type: 'updateMultiplier',
      multiplier: multiplier,
    });
    
    // 6. 更新獲勝金額
    eventEmitter.broadcast({
      type: 'updateWin',
      amount: cascade.winAmount * multiplier,
    });
    
    await delay(300);
  }
  
  // 顯示總獲勝
  eventEmitter.broadcast({
    type: 'showTotalWin',
    amount: bookEvent.totalWin,
  });
},
```

**第四步：創建 Story**

```svelte
<Story
  name="cascade"
  args={{
    skipLoadingScreen: true,
    data: events.cascade,
    action: async (data) => {
      await playBookEvent(data, { bookEvents: [] });
    },
  }}
/>
```

### 🎯 實作練習 9.2

實作上述瀑布式獲勝功能的所有 emitterEvent handlers。

---

## 9.10 本章小結

### 開發流程

1. ✅ 定義 BookEvent 類型
2. ✅ 準備測試數據
3. ✅ 創建 Storybook Story
4. ✅ 定義 EmitterEvents
5. ✅ 實作 BookEventHandler
6. ✅ 實作組件處理器
7. ✅ 測試和除錯

### 你已經學會:
- ✅ Book 和 BookEvent 的深入理解
- ✅ 完整的開發流程
- ✅ 常見模式和技巧
- ✅ 動畫協調
- ✅ 錯誤處理
- ✅ 性能優化
- ✅ 測試策略

### 🎯 作業

1. **分析現有 BookEvents**: 打開 [apps/lines/src/game/bookEventHandlerMap.ts](../apps/lines/src/game/bookEventHandlerMap.ts)
   - 找出所有 BookEvent 類型
   - 分析它們的實作模式
   - 繪製流程圖

2. **實作新功能**: 實作一個 `expandingWild` BookEvent
   - Wild 符號展開到整個轉軸
   - 播放展開動畫
   - 重新計算獲勝
   - 在 Storybook 中測試

3. **優化性能**: 找出一個性能瓶頸並優化
   - 使用 Chrome DevTools 分析
   - 實作優化
   - 測量改進效果

### 下一章預告

**第十章: 進階主題與最佳實踐**
- 國際化 (i18n)
- 音效管理
- 打包和部署
- 性能監控
- 最佳實踐總結

---

## 📚 延伸閱讀

- [遊戲設計模式](https://gameprogrammingpatterns.com/)
- [動畫原理](https://en.wikipedia.org/wiki/12_basic_principles_of_animation)
- [測試驅動開發](https://en.wikipedia.org/wiki/Test-driven_development)

---

[⬅️ 上一章: Storybook 測試](./08-storybook-development.md) | [返回目錄](./README.md) | [下一章: 進階主題 ➡️](./10-advanced-topics.md)