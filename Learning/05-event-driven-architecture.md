# 第五章：事件驅動架構 (Event Emitter)

## 學習目標

完成本章後,你將能夠:
- ✅ 理解事件驅動架構的設計模式
- ✅ 掌握 EventEmitter 的實作原理
- ✅ 運用 bookEvent 和 emitterEvent 的關係
- ✅ 實現組件間的解耦通信

---

## 5.1 為什麼需要事件驅動架構?

### 5.1.1 傳統方式的問題

**Props 傳遞 (Props Drilling):**

```
App
 ├─ Board
 │   ├─ Reel
 │   │   └─ Symbol (需要 gameState)
 │   └─ WinLine (需要 gameState)
 └─ UI
     └─ BetButton (需要 gameState, 需要更新 Board)
```

**問題:**
- 🔴 Props 需要層層傳遞
- 🔴 組件高度耦合
- 🔴 難以維護和擴展
- 🔴 父組件需要知道所有子組件的需求

### 5.1.2 事件驅動的解決方案

```
EventEmitter (事件總線)
    ↓ broadcast
 ┌──┴──┬──────┬──────┐
 │     │      │      │
Board  UI  Symbol  WinLine
 │     │      │      │
 └─ subscribe ──────┘
```

**優勢:**
- ✅ 組件解耦
- ✅ 靈活的通信
- ✅ 易於擴展
- ✅ 清晰的數據流

---

## 5.2 EventEmitter 核心概念

### 5.2.1 發布-訂閱模式

EventEmitter 基於**發布-訂閱模式 (Pub-Sub Pattern)**:

```typescript
// 發布者 (Publisher)
eventEmitter.broadcast({ 
  type: 'symbolShow', 
  index: [0, 0] 
});

// 訂閱者 (Subscriber)
eventEmitter.subscribe({
  symbolShow: (event) => {
    console.log('Symbol 顯示:', event.index);
  }
});
```

### 5.2.2 事件類型

專案中有兩種主要事件類型:

**同步事件:**
```typescript
eventEmitter.broadcast({ type: 'boardShow' });
// 立即執行,不等待
```

**異步事件:**
```typescript
await eventEmitter.broadcastAsync({ 
  type: 'reelSpin',
  duration: 1000 
});
// 等待所有處理器完成
```

---

## 5.3 EventEmitter 實作

### 5.3.1 基本結構

查看 [packages/utils-event-emitter/src/createEventEmitter.ts](../packages/utils-event-emitter/src/createEventEmitter.ts):

```typescript
export const createEventEmitter = <EmitterEvent extends { type: string }>() => {
  const handlers = new Map<string, Set<EmitterEventHandler>>();
  
  // 訂閱事件
  const subscribe = <T extends Record<string, EmitterEventHandler>>(
    handlerMap: T
  ) => {
    Object.entries(handlerMap).forEach(([type, handler]) => {
      if (!handlers.has(type)) {
        handlers.set(type, new Set());
      }
      handlers.get(type)!.add(handler);
    });
    
    // 返回取消訂閱函數
    return () => {
      Object.entries(handlerMap).forEach(([type, handler]) => {
        handlers.get(type)?.delete(handler);
      });
    };
  };
  
  // 廣播事件 (同步)
  const broadcast = (event: EmitterEvent) => {
    const eventHandlers = handlers.get(event.type);
    if (eventHandlers) {
      eventHandlers.forEach(handler => handler(event));
    }
  };
  
  // 廣播事件 (異步)
  const broadcastAsync = async (event: EmitterEvent) => {
    const eventHandlers = handlers.get(event.type);
    if (eventHandlers) {
      await Promise.all(
        Array.from(eventHandlers).map(handler => handler(event))
      );
    }
  };
  
  return { subscribe, broadcast, broadcastAsync };
};
```

### 5.3.2 關鍵設計

**使用 Map 和 Set:**
```typescript
Map<事件類型, Set<處理器>>

{
  'symbolShow': Set([handler1, handler2]),
  'symbolHide': Set([handler3]),
  'boardSpin': Set([handler4, handler5, handler6]),
}
```

**為什麼用 Set?**
- 自動去重,避免重複訂閱
- 快速添加/刪除
- 保持訂閱順序

---

## 5.4 在組件中使用 EventEmitter

### 5.4.1 訂閱事件

使用 `subscribeOnMount` 在組件掛載時訂閱:

```svelte
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext();
  
  let show = $state(false);
  
  context.eventEmitter.subscribeOnMount({
    symbolShow: (event) => {
      show = true;
    },
    symbolHide: (event) => {
      show = false;
    },
  });
</script>

<Container visible={show}>
  <Sprite ... />
</Container>
```

### 5.4.2 subscribeOnMount 的實作

查看 [packages/utils-event-emitter/src/createEventEmitter.ts](../packages/utils-event-emitter/src/createEventEmitter.ts):

```typescript
const subscribeOnMount = <T extends Record<string, EmitterEventHandler>>(
  handlerMap: T
) => {
  onMount(() => {
    const unsubscribeHandlers = subscribe(handlerMap);
    
    // 組件銷毀時自動取消訂閱
    return () => {
      unsubscribeHandlers.forEach((unsubscribe) => unsubscribe());
    };
  });
};
```

**關鍵點:**
- 在 `onMount` 中訂閱
- 返回清理函數自動取消訂閱
- 避免記憶體洩漏

### 🎯 實作練習 5.1

創建一個簡單的事件系統:

```svelte
<!-- EventDemo.svelte -->
<script lang="ts">
  import { createEventEmitter } from 'utils-event-emitter';
  import { onMount } from 'svelte';
  
  type MyEvent = 
    | { type: 'increment' }
    | { type: 'decrement' }
    | { type: 'reset' };
  
  const { eventEmitter } = createEventEmitter<MyEvent>();
  
  let count = $state(0);
  
  onMount(() => {
    return eventEmitter.subscribe({
      increment: () => count++,
      decrement: () => count--,
      reset: () => count = 0,
    });
  });
</script>

<div>
  <p>Count: {count}</p>
  <button onclick={() => eventEmitter.broadcast({ type: 'increment' })}>
    +1
  </button>
  <button onclick={() => eventEmitter.broadcast({ type: 'decrement' })}>
    -1
  </button>
  <button onclick={() => eventEmitter.broadcast({ type: 'reset' })}>
    重置
  </button>
</div>
```

---

## 5.5 EmitterEvent 類型設計

### 5.5.1 Union Types

專案使用 TypeScript Union Types 定義所有事件:

```typescript
// Symbol 組件的事件
export type EmitterEventSymbol =
  | { type: 'symbolShow'; index: [number, number] }
  | { type: 'symbolHide'; index: [number, number] }
  | { type: 'symbolDim'; index: [number, number] }
  | { type: 'symbolBright'; index: [number, number] };

// Board 組件的事件
export type EmitterEventBoard =
  | { type: 'boardShow' }
  | { type: 'boardHide' }
  | { type: 'boardSpin'; duration: number }
  | { type: 'boardSettle'; board: RawSymbol[][] };

// 遊戲的所有事件
export type EmitterEventGame =
  | EmitterEventSymbol
  | EmitterEventBoard
  | EmitterEventUI
  | ...;
```

### 5.5.2 類型安全

TypeScript 會自動推斷事件類型:

```typescript
context.eventEmitter.subscribeOnMount({
  symbolShow: (event) => {
    // TypeScript 知道 event 的類型是:
    // { type: 'symbolShow'; index: [number, number] }
    console.log(event.index); // ✅ 有類型提示
  },
  boardSpin: (event) => {
    // { type: 'boardSpin'; duration: number }
    console.log(event.duration); // ✅ 有類型提示
  },
});
```

### 專案實例：Symbol 事件

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts" module>
  export type EmitterEventSymbol =
    | { type: 'symbolShow'; index: [number, number] }
    | { type: 'symbolHide'; index: [number, number] }
    | { type: 'symbolDim'; index: [number, number] }
    | { type: 'symbolBright'; index: [number, number] }
    | { type: 'symbolWin'; index: [number, number]; multiplier: number }
    | { type: 'symbolIdle'; index: [number, number] };
</script>

<script lang="ts">
  const matchIndex = (targetIndex: [number, number]) => {
    return index[0] === targetIndex[0] && index[1] === targetIndex[1];
  };
  
  context.eventEmitter.subscribeOnMount({
    symbolShow: (event) => {
      if (matchIndex(event.index)) show = true;
    },
    symbolHide: (event) => {
      if (matchIndex(event.index)) show = false;
    },
    symbolDim: (event) => {
      if (matchIndex(event.index)) state = 'dim';
    },
    symbolBright: (event) => {
      if (matchIndex(event.index)) state = 'bright';
    },
  });
</script>
```

---

## 5.6 同步 vs 異步事件

### 5.6.1 同步事件 (broadcast)

用於不需要等待的操作:

```typescript
// 發送
eventEmitter.broadcast({ type: 'boardShow' });
eventEmitter.broadcast({ type: 'updateScore', score: 100 });

// 接收
context.eventEmitter.subscribeOnMount({
  boardShow: () => {
    show = true; // 立即執行
  },
  updateScore: (event) => {
    score = event.score; // 立即更新
  },
});
```

### 5.6.2 異步事件 (broadcastAsync)

用於需要等待動畫完成的操作:

```typescript
// 發送 (等待所有處理器完成)
await eventEmitter.broadcastAsync({ 
  type: 'reelSpin',
  duration: 2000 
});
console.log('轉軸旋轉完成');

// 接收 (返回 Promise)
context.eventEmitter.subscribeOnMount({
  reelSpin: async (event) => {
    await animateReel(event.duration);
    // 動畫完成後才返回
  },
});
```

### 專案實例：轉軸旋轉

查看 [apps/lines/src/game/bookEventHandlerMap.ts](../apps/lines/src/game/bookEventHandlerMap.ts):

```typescript
export const bookEventHandlerMap = {
  reveal: async (bookEvent: BookEventOfType<'reveal'>) => {
    // 1. 設置面板數據 (同步)
    eventEmitter.broadcast({
      type: 'boardSetSymbols',
      board: bookEvent.board,
    });
    
    // 2. 播放旋轉動畫 (異步,等待完成)
    await eventEmitter.broadcastAsync({
      type: 'reelSpin',
      duration: 2000,
    });
    
    // 3. 顯示獲勝線 (同步)
    if (bookEvent.winLines) {
      eventEmitter.broadcast({
        type: 'winLinesShow',
        lines: bookEvent.winLines,
      });
    }
  },
};
```

---

## 5.7 BookEvent 和 EmitterEvent 的關係

### 5.7.1 概念圖

```
RGS Server
    ↓
Book (書籍)
    ↓
BookEvents (書籍事件)
    ↓
bookEventHandlerMap (處理器映射)
    ↓
EmitterEvents (發射器事件)
    ↓
Component Handlers (組件處理器)
    ↓
UI Update (UI 更新)
```

### 5.7.2 一對多關係

一個 BookEvent 可以產生多個 EmitterEvents:

```typescript
// BookEvent: reveal
{
  type: 'reveal',
  board: [...],
  winLines: [...],
  anticipation: [...]
}

// 轉換為多個 EmitterEvents:
bookEventHandlerMap.reveal = async (bookEvent) => {
  // EmitterEvent 1: 設置面板
  eventEmitter.broadcast({ 
    type: 'boardSetSymbols', 
    board: bookEvent.board 
  });
  
  // EmitterEvent 2: 播放預期動畫
  if (bookEvent.anticipation.some(a => a > 0)) {
    await eventEmitter.broadcastAsync({ 
      type: 'anticipationPlay' 
    });
  }
  
  // EmitterEvent 3: 旋轉轉軸
  await eventEmitter.broadcastAsync({ 
    type: 'reelSpin' 
  });
  
  // EmitterEvent 4: 停止轉軸
  eventEmitter.broadcast({ 
    type: 'reelStop' 
  });
  
  // EmitterEvent 5: 顯示獲勝線
  if (bookEvent.winLines) {
    eventEmitter.broadcast({ 
      type: 'winLinesShow', 
      lines: bookEvent.winLines 
    });
  }
};
```

### 5.7.3 跨組件協作

不同組件訂閱不同的 EmitterEvents:

```typescript
// Board.svelte
context.eventEmitter.subscribeOnMount({
  boardSetSymbols: (event) => { /* 設置符號 */ },
  reelSpin: async (event) => { /* 播放旋轉 */ },
  reelStop: (event) => { /* 停止旋轉 */ },
});

// Symbol.svelte
context.eventEmitter.subscribeOnMount({
  symbolShow: (event) => { /* 顯示符號 */ },
  symbolWin: (event) => { /* 播放獲勝動畫 */ },
});

// WinLine.svelte
context.eventEmitter.subscribeOnMount({
  winLinesShow: (event) => { /* 顯示獲勝線 */ },
  winLinesHide: (event) => { /* 隱藏獲勝線 */ },
});

// UI.svelte
context.eventEmitter.subscribeOnMount({
  updateBalance: (event) => { /* 更新餘額 */ },
  updateWin: (event) => { /* 更新獲勝金額 */ },
});
```

### 🎯 實作練習 5.2

模擬一個簡單的 BookEvent 處理:

```typescript
// bookEvent.ts
type BookEvent = {
  type: 'spin';
  result: string[];
  win: number;
};

type EmitterEvent =
  | { type: 'reelStart' }
  | { type: 'reelStop'; symbols: string[] }
  | { type: 'showWin'; amount: number };

// bookEventHandler.ts
const handleSpin = async (bookEvent: BookEvent) => {
  // 1. 開始旋轉
  eventEmitter.broadcast({ type: 'reelStart' });
  
  // 2. 等待 2 秒
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  // 3. 停止旋轉
  eventEmitter.broadcast({ 
    type: 'reelStop', 
    symbols: bookEvent.result 
  });
  
  // 4. 顯示獲勝
  if (bookEvent.win > 0) {
    eventEmitter.broadcast({ 
      type: 'showWin', 
      amount: bookEvent.win 
    });
  }
};
```

---

## 5.8 任務分解 (Task Breakdown)

### 5.8.1 設計原則

專案遵循**單一職責原則 (Single Responsibility Principle)**:

```typescript
// ❌ 錯誤：一個事件做太多事
eventEmitter.subscribeOnMount({
  boardSpin: async (event) => {
    // 播放音效
    playSound('spin');
    // 旋轉動畫
    await animate();
    // 停止動畫
    stopAnimation();
    // 更新 UI
    updateUI();
    // 太多職責！
  },
});

// ✅ 正確：分解為多個小事件
eventEmitter.subscribeOnMount({
  boardSpinStart: () => {
    playSound('spin');
  },
  boardSpinAnimate: async () => {
    await animate();
  },
  boardSpinStop: () => {
    stopAnimation();
  },
  boardSpinComplete: () => {
    updateUI();
  },
});
```

### 5.8.2 實際範例：tumbleBoard

查看 [apps/cluster/src/game/bookEventHandlerMap.ts](../apps/cluster/src/game/bookEventHandlerMap.ts):

```typescript
tumbleBoard: async (bookEvent: BookEventOfType<'tumbleBoard'>) => {
  // 1. 顯示 tumble 面板
  eventEmitter.broadcast({ type: 'tumbleBoardShow' });
  
  // 2. 初始化新符號
  eventEmitter.broadcast({ 
    type: 'tumbleBoardInit', 
    addingBoard: bookEvent.newSymbols 
  });
  
  // 3. 爆炸動畫
  await eventEmitter.broadcastAsync({
    type: 'tumbleBoardExplode',
    explodingPositions: bookEvent.explodingSymbols,
  });
  
  // 4. 移除已爆炸的符號
  eventEmitter.broadcast({ type: 'tumbleBoardRemoveExploded' });
  
  // 5. 下落動畫
  await eventEmitter.broadcastAsync({ type: 'tumbleBoardSlideDown' });
  
  // 6. 穩定面板
  eventEmitter.broadcast({
    type: 'boardSettle',
    board: finalBoard,
  });
  
  // 7. 重置 tumble 狀態
  eventEmitter.broadcast({ type: 'tumbleBoardReset' });
  
  // 8. 隱藏 tumble 面板
  eventEmitter.broadcast({ type: 'tumbleBoardHide' });
},
```

**每個事件都可以在 Storybook 中獨立測試!**

---

## 5.9 在 Storybook 中測試事件

### 5.9.1 測試單個事件

```svelte
<!-- ModeBonusBookEvent.stories.svelte -->
<Story
  name="tumbleBoardExplode"
  args={templateArgs({
    skipLoadingScreen: true,
    data: events.tumbleBoardExplode,
    action: async (data) => {
      // 只觸發這個事件
      await context.eventEmitter.broadcastAsync({
        type: 'tumbleBoardExplode',
        explodingPositions: data.explodingPositions,
      });
    },
  })}
/>
```

### 5.9.2 測試事件序列

```svelte
<Story
  name="tumbleBoard"
  args={templateArgs({
    skipLoadingScreen: true,
    data: events.tumbleBoard,
    action: async (data) => {
      // 觸發整個 bookEvent 處理器
      await playBookEvent(data, { bookEvents: [] });
    },
  })}
/>
```

### 🎯 實作練習 5.3

1. 運行 `pnpm run storybook --filter=cluster`
2. 找到 `MODE_BASE/bookEvent/tumbleBoard`
3. 觀察每個 EmitterEvent 的執行順序
4. 思考：如果缺少某個事件會發生什麼?

---

## 5.10 組件間通信模式

### 5.10.1 父子通信

**傳統方式 (Props):**
```svelte
<!-- Parent.svelte -->
<Child value={parentValue} onchange={(v) => parentValue = v} />

<!-- Child.svelte -->
<button onclick={() => onchange(newValue)}>更新</button>
```

**事件驅動方式:**
```svelte
<!-- Parent.svelte -->
<Child />

<!-- Child.svelte -->
<button onclick={() => eventEmitter.broadcast({ type: 'valueChange', value: newValue })}>
  更新
</button>
```

### 5.10.2 兄弟組件通信

```svelte
<!-- ComponentA.svelte -->
<script lang="ts">
  context.eventEmitter.subscribeOnMount({
    dataUpdate: (event) => {
      // 接收來自 ComponentB 的數據
      localData = event.data;
    },
  });
</script>

<!-- ComponentB.svelte -->
<script lang="ts">
  function sendData() {
    context.eventEmitter.broadcast({
      type: 'dataUpdate',
      data: myData,
    });
  }
</script>
```

### 5.10.3 跨層級通信

```
App
 ├─ Board
 │   └─ Symbol (發送事件)
 └─ UI
     └─ ScoreDisplay (接收事件)
```

無需通過中間層:

```svelte
<!-- Symbol.svelte (深層組件) -->
<script lang="ts">
  function onWin() {
    eventEmitter.broadcast({ 
      type: 'updateScore', 
      score: 100 
    });
  }
</script>

<!-- ScoreDisplay.svelte (另一個分支) -->
<script lang="ts">
  let score = $state(0);
  
  context.eventEmitter.subscribeOnMount({
    updateScore: (event) => {
      score += event.score;
    },
  });
</script>
```

---

## 5.11 最佳實踐

### 5.11.1 命名規範

```typescript
// ✅ 良好：動詞 + 名詞
{ type: 'symbolShow' }
{ type: 'boardHide' }
{ type: 'reelSpin' }
{ type: 'winLineAnimate' }

// ❌ 不好：不清楚
{ type: 'symbol' }
{ type: 'update' }
{ type: 'action' }
```

### 5.11.2 事件粒度

```typescript
// ✅ 良好：細粒度事件
{ type: 'fadeIn' }
{ type: 'fadeOut' }
{ type: 'slideUp' }

// ❌ 不好：粗粒度事件
{ type: 'animate', action: 'fadeIn' }
```

### 5.11.3 避免循環依賴

```typescript
// ❌ 錯誤：A 觸發 B,B 觸發 A
// ComponentA
eventEmitter.subscribeOnMount({
  eventB: () => {
    eventEmitter.broadcast({ type: 'eventA' }); // 危險!
  },
});

// ComponentB
eventEmitter.subscribeOnMount({
  eventA: () => {
    eventEmitter.broadcast({ type: 'eventB' }); // 危險!
  },
});
```

### 5.11.4 使用 TypeScript

```typescript
// ✅ 良好：嚴格的類型定義
type EmitterEvent =
  | { type: 'increment'; amount: number }
  | { type: 'decrement'; amount: number };

// ❌ 不好：寬鬆的類型
type EmitterEvent = {
  type: string;
  [key: string]: any;
};
```

---

## 5.12 實戰：創建一個計分系統

### 📝 任務說明

使用 EventEmitter 創建一個完整的計分系統:

```typescript
// events.ts
export type ScoreEvent =
  | { type: 'scoreAdd'; points: number; source: string }
  | { type: 'scoreSubtract'; points: number }
  | { type: 'scoreReset' }
  | { type: 'scoreMultiply'; multiplier: number }
  | { type: 'comboStart'; level: number }
  | { type: 'comboEnd' };

// ScoreManager.svelte
<script lang="ts">
  let score = $state(0);
  let combo = $state(1);
  let comboLevel = $state(0);
  
  context.eventEmitter.subscribeOnMount({
    scoreAdd: (event) => {
      score += event.points * combo;
      console.log(`+${event.points} from ${event.source}`);
    },
    scoreSubtract: (event) => {
      score = Math.max(0, score - event.points);
    },
    scoreReset: () => {
      score = 0;
      combo = 1;
      comboLevel = 0;
    },
    scoreMultiply: (event) => {
      score *= event.multiplier;
    },
    comboStart: (event) => {
      comboLevel = event.level;
      combo = 1 + (event.level * 0.5);
    },
    comboEnd: () => {
      combo = 1;
      comboLevel = 0;
    },
  });
</script>

<Container>
  <Text text={`Score: ${score}`} />
  {#if comboLevel > 0}
    <Text text={`Combo x${combo}!`} />
  {/if}
</Container>

// Game.svelte
<script lang="ts">
  function onEnemyKilled() {
    eventEmitter.broadcast({ 
      type: 'scoreAdd', 
      points: 100, 
      source: 'enemy' 
    });
    
    if (consecutiveKills > 3) {
      eventEmitter.broadcast({ 
        type: 'comboStart', 
        level: Math.floor(consecutiveKills / 3) 
      });
    }
  }
  
  function onLevelComplete() {
    eventEmitter.broadcast({ 
      type: 'scoreMultiply', 
      multiplier: 1.5 
    });
  }
</script>
```

### 🎯 實作練習 5.4

1. 實作上述計分系統
2. 添加計分歷史記錄
3. 添加最高分記錄
4. 在 Storybook 中測試各種事件組合

---

## 5.13 本章小結

### EventEmitter 核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| broadcast | 同步發送事件 | `broadcast({ type: 'show' })` |
| broadcastAsync | 異步發送事件 | `await broadcastAsync(...)` |
| subscribe | 訂閱事件 | `subscribe({ show: () => {} })` |
| subscribeOnMount | 組件訂閱 | `subscribeOnMount({ ... })` |
| Union Types | 類型安全 | `type Event = A \| B \| C` |

### 你已經學會:
- ✅ 事件驅動架構的優勢
- ✅ EventEmitter 的實作原理
- ✅ 同步和異步事件的使用
- ✅ BookEvent 和 EmitterEvent 的關係
- ✅ 任務分解的設計原則
- ✅ 組件間通信的最佳實踐

### 🎯 作業

1. **分析 bookEventHandlerMap**: 打開 [apps/lines/src/game/bookEventHandlerMap.ts](../apps/lines/src/game/bookEventHandlerMap.ts)
   - 找出 `reveal` bookEvent 產生的所有 emitterEvents
   - 畫出事件流程圖
   - 思考為什麼要這樣設計

2. **創建自訂 bookEvent**: 創建一個 `bonusWin` bookEvent
   - 播放勝利音效
   - 顯示勝利動畫
   - 更新分數
   - 在 Storybook 中測試

3. **探索 Board 組件**: 閱讀 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte)
   - 找出所有訂閱的事件
   - 理解每個事件的職責
   - 思考如何添加新功能

### 下一章預告

**第六章: 狀態管理 (XState)**
- 有限狀態機概念
- XState 基礎
- 遊戲狀態管理
- Bet/AutoBet/ResumeBet 流程

---

## 📚 延伸閱讀

- [發布訂閱模式](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)
- [事件驅動架構](https://en.wikipedia.org/wiki/Event-driven_architecture)
- [單一職責原則](https://en.wikipedia.org/wiki/Single-responsibility_principle)

---

[⬅️ 上一章: pixi-svelte 整合](./04-pixi-svelte-integration.md) | [返回目錄](./README.md) | [下一章: 狀態管理 (XState) ➡️](./06-state-management-xstate.md)
