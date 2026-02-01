# 第六章：狀態管理 (XState)

## 學習目標

完成本章後,你將能夠:
- ✅ 理解有限狀態機 (FSM) 的概念
- ✅ 掌握 XState 的基本使用
- ✅ 運用狀態機管理遊戲流程
- ✅ 實現 Bet/AutoBet/ResumeBet 邏輯

---

## 6.1 為什麼需要狀態機?

### 6.1.1 傳統狀態管理的問題

使用布林值和條件判斷管理複雜狀態:

```typescript
let isIdle = true;
let isBetting = false;
let isAutoBetting = false;
let isPlaying = false;
let isWinning = false;

function startBet() {
  if (isIdle && !isBetting && !isPlaying) {
    isIdle = false;
    isBetting = true;
    // ...
  }
}

function startAutoBet() {
  if (isIdle && !isBetting && !isAutoBetting) {
    // 組合爆炸！
  }
}
```

**問題:**
- 🔴 狀態組合爆炸
- 🔴 容易出現不可能的狀態
- 🔴 條件判斷複雜且容易出錯
- 🔴 難以測試和維護

### 6.1.2 狀態機的解決方案

```typescript
// 明確定義所有可能的狀態
const gameStates = {
  IDLE: 'idle',
  BETTING: 'betting',
  AUTO_BETTING: 'autoBetting',
  PLAYING: 'playing',
} as const;

// 只能處於一個狀態
let currentState = gameStates.IDLE;

// 明確定義狀態轉換
const transitions = {
  [gameStates.IDLE]: {
    START_BET: gameStates.BETTING,
    START_AUTO_BET: gameStates.AUTO_BETTING,
  },
  [gameStates.BETTING]: {
    COMPLETE: gameStates.IDLE,
  },
};
```

**優勢:**
- ✅ 狀態明確且互斥
- ✅ 轉換清晰可預測
- ✅ 不可能出現非法狀態
- ✅ 易於測試和視覺化

---

## 6.2 有限狀態機 (FSM) 基礎

### 6.2.1 核心概念

**有限狀態機**由以下元素組成:

```
┌──────────────────────────────────┐
│  有限狀態機 (FSM)                │
├──────────────────────────────────┤
│  1. 狀態 (States)                │
│     - IDLE, BETTING, PLAYING     │
│                                  │
│  2. 事件 (Events)                │
│     - START_BET, STOP, COMPLETE  │
│                                  │
│  3. 轉換 (Transitions)           │
│     - IDLE + START_BET → BETTING │
│                                  │
│  4. 動作 (Actions)               │
│     - 進入/離開狀態時執行         │
└──────────────────────────────────┘
```

### 6.2.2 狀態圖範例

```
     START_BET
IDLE ─────────→ BETTING
 ↑                 │
 │                 │ COMPLETE
 └─────────────────┘

     START_AUTO_BET
IDLE ─────────────→ AUTO_BETTING
 ↑                     │
 │                     │ STOP
 └─────────────────────┘
```

### 6.2.3 交通燈範例

```typescript
// 狀態
const states = {
  RED: 'red',
  YELLOW: 'yellow',
  GREEN: 'green',
};

// 狀態機
let currentState = states.RED;

function transition(event: string) {
  if (currentState === states.RED && event === 'TIMER') {
    currentState = states.GREEN;
  } else if (currentState === states.GREEN && event === 'TIMER') {
    currentState = states.YELLOW;
  } else if (currentState === states.YELLOW && event === 'TIMER') {
    currentState = states.RED;
  }
}

// 使用
transition('TIMER'); // RED → GREEN
transition('TIMER'); // GREEN → YELLOW
transition('TIMER'); // YELLOW → RED
```

---

## 6.3 XState 入門

### 6.3.1 安裝和基本使用

```typescript
import { setup, createActor } from 'xstate';

// 定義狀態機
const toggleMachine = setup({
  types: {
    events: {} as { type: 'TOGGLE' },
  },
}).createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: {
        TOGGLE: 'active',
      },
    },
    active: {
      on: {
        TOGGLE: 'inactive',
      },
    },
  },
});

// 創建 actor (運行實例)
const toggleActor = createActor(toggleMachine);
toggleActor.start();

// 發送事件
console.log(toggleActor.getSnapshot().value); // 'inactive'
toggleActor.send({ type: 'TOGGLE' });
console.log(toggleActor.getSnapshot().value); // 'active'
```

### 6.3.2 在 Svelte 中使用

```svelte
<script lang="ts">
  import { setup, createActor } from 'xstate';
  
  const toggleMachine = setup({
    types: {
      events: {} as { type: 'TOGGLE' },
    },
  }).createMachine({
    initial: 'inactive',
    states: {
      inactive: { on: { TOGGLE: 'active' } },
      active: { on: { TOGGLE: 'inactive' } },
    },
  });
  
  const actor = createActor(toggleMachine);
  actor.start();
  
  let state = $state(actor.getSnapshot().value);
  
  actor.subscribe((snapshot) => {
    state = snapshot.value;
  });
</script>

<div>
  <p>當前狀態: {state}</p>
  <button onclick={() => actor.send({ type: 'TOGGLE' })}>
    切換
  </button>
</div>
```

### 🎯 實作練習 6.1

創建一個燈泡狀態機:

```svelte
<script lang="ts">
  import { setup, createActor } from 'xstate';
  
  const lightMachine = setup({
    types: {
      events: {} as 
        | { type: 'TURN_ON' }
        | { type: 'TURN_OFF' }
        | { type: 'BREAK' },
    },
  }).createMachine({
    initial: 'off',
    states: {
      off: {
        on: {
          TURN_ON: 'on',
          BREAK: 'broken',
        },
      },
      on: {
        on: {
          TURN_OFF: 'off',
          BREAK: 'broken',
        },
      },
      broken: {
        // 無法轉換到其他狀態
        type: 'final',
      },
    },
  });
  
  const actor = createActor(lightMachine);
  actor.start();
  
  let state = $state(actor.getSnapshot().value);
  
  actor.subscribe((snapshot) => {
    state = snapshot.value;
  });
</script>

<div>
  <p>燈泡狀態: {state}</p>
  <button onclick={() => actor.send({ type: 'TURN_ON' })}>開燈</button>
  <button onclick={() => actor.send({ type: 'TURN_OFF' })}>關燈</button>
  <button onclick={() => actor.send({ type: 'BREAK' })}>打破</button>
</div>
```

---

## 6.4 專案中的狀態機架構

### 6.4.1 主狀態機

查看 [packages/utils-xstate/src/createPrimaryMachines.ts](../packages/utils-xstate/src/createPrimaryMachines.ts):

```typescript
const gameMachine = setup({
  actors: {
    bet: intermediateMachines.bet,
    autoBet: intermediateMachines.autoBet,
    resumeBet: intermediateMachines.resumeBet,
  },
}).createMachine({
  initial: 'rendering',
  states: {
    [STATE_RENDERING]: stateRendering,
    [STATE_IDLE]: stateIdle,
    [STATE_BET]: stateBet,
    [STATE_AUTOBET]: stateAutoBet,
    [STATE_RESUME_BET]: stateResumeBet,
  },
});
```

**狀態層級:**
```
gameMachine
├─ rendering (渲染中)
├─ idle (閒置)
├─ bet (單次下注)
│   └─ 子狀態機
├─ autoBet (自動下注)
│   └─ 子狀態機
└─ resumeBet (恢復下注)
    └─ 子狀態機
```

### 6.4.2 狀態圖

```
                START_GAME
RENDERING ─────────────→ IDLE
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    START_BET      START_AUTO_BET   RESUME_SESSION
           │               │               │
           ↓               ↓               ↓
         BET          AUTO_BET        RESUME_BET
           │               │               │
           └───────────────┴───────────────┘
                           │
                           ↓
                         IDLE
```

---

## 6.5 遊戲狀態詳解

### 6.5.1 RENDERING 狀態

初始狀態,載入資源:

```typescript
const stateRendering = {
  on: {
    START_GAME: {
      target: STATE_IDLE,
    },
  },
};
```

**用途:**
- 載入遊戲資源
- 初始化遊戲數據
- 顯示載入畫面

### 6.5.2 IDLE 狀態

閒置狀態,等待用戶操作:

```typescript
const stateIdle = {
  on: {
    START_BET: {
      target: STATE_BET,
      guard: 'canStartBet', // 檢查是否可以下注
    },
    START_AUTO_BET: {
      target: STATE_AUTOBET,
      guard: 'canStartAutoBet',
    },
    RESUME_SESSION: {
      target: STATE_RESUME_BET,
    },
  },
};
```

**用途:**
- 等待用戶點擊下注按鈕
- 顯示當前餘額和投注額
- UI 完全可互動

### 6.5.3 BET 狀態

單次下注流程:

```typescript
const stateBet = {
  invoke: {
    src: 'bet', // 調用 bet 子狀態機
    onDone: {
      target: STATE_IDLE,
    },
    onError: {
      target: STATE_IDLE,
      actions: 'handleError',
    },
  },
};
```

**流程:**
1. 發送下注請求到 RGS
2. 接收並播放 book
3. 更新餘額
4. 返回 IDLE

### 6.5.4 AUTO_BET 狀態

自動下注循環:

```typescript
const stateAutoBet = {
  invoke: {
    src: 'autoBet', // 調用 autoBet 子狀態機
    onDone: {
      target: STATE_IDLE,
    },
  },
  on: {
    STOP_AUTO_BET: {
      target: STATE_IDLE,
      actions: 'stopAutoBet',
    },
  },
};
```

**流程:**
1. 循環執行下注
2. 檢查餘額是否足夠
3. 檢查是否達到次數限制
4. 用戶可隨時停止

### 6.5.5 RESUME_BET 狀態

恢復未完成的遊戲:

```typescript
const stateResumeBet = {
  invoke: {
    src: 'resumeBet',
    onDone: {
      target: STATE_IDLE,
    },
  },
};
```

**用途:**
- 從認證數據中恢復遊戲狀態
- 播放未完成的 book
- 確保遊戲一致性

---

## 6.6 子狀態機

### 6.6.1 Bet 子狀態機

```typescript
const betMachine = setup({}).createMachine({
  initial: 'newGame',
  states: {
    newGame: {
      invoke: {
        src: 'requestNewGame',
        onDone: {
          target: 'playing',
          actions: 'saveBook',
        },
        onError: {
          target: 'error',
        },
      },
    },
    playing: {
      invoke: {
        src: 'playBook',
        onDone: {
          target: 'endGame',
        },
      },
    },
    endGame: {
      invoke: {
        src: 'handleEndGame',
        onDone: {
          type: 'final', // 完成,返回父狀態機
        },
      },
    },
    error: {
      type: 'final',
    },
  },
});
```

**狀態流程:**
```
newGame → playing → endGame → (完成)
   ↓
 error → (完成)
```

### 6.6.2 AutoBet 子狀態機

```typescript
const autoBetMachine = setup({}).createMachine({
  initial: 'checkCondition',
  states: {
    checkCondition: {
      always: [
        {
          target: 'betting',
          guard: 'canContinue', // 檢查餘額和次數
        },
        {
          target: 'complete',
        },
      ],
    },
    betting: {
      invoke: {
        src: 'bet', // 重用 bet 子狀態機
        onDone: {
          target: 'checkCondition', // 循環
        },
      },
    },
    complete: {
      type: 'final',
    },
  },
});
```

**循環流程:**
```
checkCondition → betting → checkCondition → ...
       ↓
   complete
```

### 🎯 實作練習 6.2

創建一個計時器狀態機:

```typescript
const timerMachine = setup({
  types: {
    context: {} as { elapsed: number; duration: number },
    events: {} as 
      | { type: 'START'; duration: number }
      | { type: 'TICK' }
      | { type: 'RESET' },
  },
  actions: {
    tick: ({ context }) => {
      context.elapsed++;
    },
    reset: ({ context }) => {
      context.elapsed = 0;
    },
    setDuration: ({ context, event }) => {
      if (event.type === 'START') {
        context.duration = event.duration;
      }
    },
  },
  guards: {
    isComplete: ({ context }) => {
      return context.elapsed >= context.duration;
    },
  },
}).createMachine({
  initial: 'idle',
  context: { elapsed: 0, duration: 10 },
  states: {
    idle: {
      on: {
        START: {
          target: 'running',
          actions: ['setDuration', 'reset'],
        },
      },
    },
    running: {
      on: {
        TICK: [
          {
            target: 'complete',
            guard: 'isComplete',
          },
          {
            actions: 'tick',
          },
        ],
        RESET: 'idle',
      },
    },
    complete: {
      on: {
        RESET: 'idle',
      },
    },
  },
});
```

---

## 6.7 Context (上下文)

### 6.7.1 什麼是 Context?

Context 是狀態機的記憶體,儲存數據:

```typescript
const counterMachine = setup({
  types: {
    context: {} as { count: number },
    events: {} as 
      | { type: 'INCREMENT' }
      | { type: 'DECREMENT' },
  },
  actions: {
    increment: ({ context }) => {
      context.count++;
    },
    decrement: ({ context }) => {
      context.count--;
    },
  },
}).createMachine({
  initial: 'active',
  context: { count: 0 }, // 初始 context
  states: {
    active: {
      on: {
        INCREMENT: {
          actions: 'increment',
        },
        DECREMENT: {
          actions: 'decrement',
        },
      },
    },
  },
});
```

### 6.7.2 專案中的 Context

```typescript
// 遊戲的 context
type GameContext = {
  betAmount: number;
  balance: number;
  autoBetCount: number;
  maxAutoBetCount: number;
  book: Book | null;
  error: Error | null;
};

const gameMachine = setup({
  types: {
    context: {} as GameContext,
  },
}).createMachine({
  context: {
    betAmount: 100,
    balance: 10000,
    autoBetCount: 0,
    maxAutoBetCount: 10,
    book: null,
    error: null,
  },
  // ...
});
```

---

## 6.8 Guards (守衛)

### 6.8.1 條件轉換

Guards 決定是否允許狀態轉換:

```typescript
const vendingMachine = setup({
  types: {
    context: {} as { balance: number; price: number },
  },
  guards: {
    hasEnoughMoney: ({ context }) => {
      return context.balance >= context.price;
    },
  },
}).createMachine({
  initial: 'idle',
  context: { balance: 0, price: 100 },
  states: {
    idle: {
      on: {
        INSERT_COIN: {
          actions: ({ context, event }) => {
            context.balance += event.amount;
          },
        },
        BUY: [
          {
            target: 'vending',
            guard: 'hasEnoughMoney', // 必須滿足條件
          },
          {
            target: 'insufficientFunds',
          },
        ],
      },
    },
    vending: {
      // ...
    },
    insufficientFunds: {
      // ...
    },
  },
});
```

### 6.8.2 專案中的 Guards

```typescript
const guards = {
  canStartBet: ({ context }) => {
    return (
      context.balance >= context.betAmount &&
      !context.isPlaying
    );
  },
  canContinueAutoBet: ({ context }) => {
    return (
      context.balance >= context.betAmount &&
      context.autoBetCount < context.maxAutoBetCount
    );
  },
};
```

---

## 6.9 Actions (動作)

### 6.9.1 進入/離開動作

```typescript
const doorMachine = setup({
  actions: {
    openDoor: () => console.log('Door opened'),
    closeDoor: () => console.log('Door closed'),
    lockDoor: () => console.log('Door locked'),
  },
}).createMachine({
  initial: 'closed',
  states: {
    closed: {
      entry: 'closeDoor', // 進入時執行
      on: {
        OPEN: 'opened',
      },
    },
    opened: {
      entry: 'openDoor',
      exit: 'closeDoor', // 離開時執行
      on: {
        CLOSE: 'closed',
      },
    },
  },
});
```

### 6.9.2 專案中的 Actions

```typescript
const actions = {
  saveBook: ({ context, event }) => {
    context.book = event.output;
  },
  updateBalance: ({ context, event }) => {
    context.balance = event.balance;
  },
  incrementAutoBetCount: ({ context }) => {
    context.autoBetCount++;
  },
  resetAutoBetCount: ({ context }) => {
    context.autoBetCount = 0;
  },
  handleError: ({ context, event }) => {
    context.error = event.error;
    console.error('Game error:', event.error);
  },
};
```

---

## 6.10 使用狀態機控制 UI

### 6.10.1 獲取當前狀態

查看 [packages/utils-xstate/src/createXstateUtils.svelte.ts](../packages/utils-xstate/src/createXstateUtils.svelte.ts):

```typescript
const stateXstate = $state({
  value: '' as StateValue,
});

const matchesXstate = (state: string) => 
  matchesState(state, stateXstate.value);

const stateXstateDerived = {
  matchesXstate,
  isRendering: () => matchesXstate(STATE_RENDERING),
  isIdle: () => matchesXstate(STATE_IDLE),
  isBetting: () => matchesXstate(STATE_BET),
  isAutoBetting: () => matchesXstate(STATE_AUTOBET),
  isResumingBet: () => matchesXstate(STATE_RESUME_BET),
  isPlaying: () => 
    !matchesXstate(STATE_RENDERING) && 
    !matchesXstate(STATE_IDLE),
};
```

### 6.10.2 在組件中使用

```svelte
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext();
  
  // 根據狀態決定按鈕是否可用
  let canBet = $derived(
    context.stateXstateDerived.isIdle() &&
    balance >= betAmount
  );
  
  let isPlaying = $derived(
    context.stateXstateDerived.isPlaying()
  );
</script>

<button 
  disabled={!canBet}
  onclick={() => gameActor.send({ type: 'START_BET' })}
>
  {#if isPlaying}
    遊戲中...
  {:else}
    下注
  {/if}
</button>
```

### 專案實例：BetButton

查看 [packages/components-ui-pixi/src/BetButton.svelte](../packages/components-ui-pixi/src/BetButton.svelte):

```svelte
<script lang="ts">
  const context = getContext();
  
  let disabled = $derived(
    context.stateXstateDerived.isPlaying() ||
    context.stateBet.balance < context.stateBet.betAmount
  );
  
  function handleClick() {
    if (context.stateXstateDerived.isIdle()) {
      context.gameActor.send({ type: 'START_BET' });
    }
  }
</script>

<SimpleUiButton
  {disabled}
  onclick={handleClick}
  label="下注"
/>
```

---

## 6.11 下注類型處理

### 6.11.1 三種獲勝類型

專案定義了三種獲勝類型:

```typescript
const BET_TYPE_METHODS_MAP = {
  // 無獲勝
  noWin: {
    newGame: async () => undefined,
    endGame: async () => undefined,
  },
  
  // 單輪獲勝 (在 newGame 時結算)
  singleRoundWin: {
    newGame: async () => {
      const endRoundData = await handleRequestEndRound();
      if (endRoundData?.balance) {
        balanceAmountFromApiHolder = endRoundData.balance.amount;
      }
    },
    endGame: async () => {
      if (balanceAmountFromApiHolder !== null) {
        handleUpdateBalance({ 
          balanceAmountFromApi: balanceAmountFromApiHolder 
        });
        balanceAmountFromApiHolder = null;
      }
    },
  },
  
  // Bonus 獲勝 (在 endGame 時結算)
  bonusWin: {
    newGame: async () => undefined,
    endGame: async () => {
      const data = await handleRequestEndRound();
      if (data?.balance) {
        handleUpdateBalance({ 
          balanceAmountFromApi: data.balance.amount 
        });
      }
    },
  },
};
```

### 6.11.2 為什麼區分?

**noWin:**
- 沒有獲勝
- 不需要調用 end-round API

**singleRoundWin:**
- 基礎遊戲獲勝
- 在 newGame 時調用 end-round
- 可以立即開始下一輪

**bonusWin:**
- Bonus 遊戲獲勝
- 在 endGame 時調用 end-round
- 確保 bonus 完全播放完畢

---

## 6.12 實戰：創建投幣機狀態機

### 📝 任務說明

創建一個完整的投幣機狀態機:

```typescript
type VendingContext = {
  balance: number;
  inventory: Record<string, number>;
  prices: Record<string, number>;
};

type VendingEvent =
  | { type: 'INSERT_COIN'; amount: number }
  | { type: 'SELECT_ITEM'; item: string }
  | { type: 'CANCEL' }
  | { type: 'DISPENSE_COMPLETE' };

const vendingMachine = setup({
  types: {
    context: {} as VendingContext,
    events: {} as VendingEvent,
  },
  guards: {
    hasEnoughMoney: ({ context, event }) => {
      if (event.type === 'SELECT_ITEM') {
        const price = context.prices[event.item];
        return context.balance >= price;
      }
      return false;
    },
    hasInventory: ({ context, event }) => {
      if (event.type === 'SELECT_ITEM') {
        return (context.inventory[event.item] || 0) > 0;
      }
      return false;
    },
  },
  actions: {
    addBalance: ({ context, event }) => {
      if (event.type === 'INSERT_COIN') {
        context.balance += event.amount;
      }
    },
    deductBalance: ({ context, event }) => {
      if (event.type === 'SELECT_ITEM') {
        context.balance -= context.prices[event.item];
      }
    },
    decrementInventory: ({ context, event }) => {
      if (event.type === 'SELECT_ITEM') {
        context.inventory[event.item]--;
      }
    },
    returnChange: ({ context }) => {
      console.log(`Returning change: $${context.balance}`);
      context.balance = 0;
    },
  },
}).createMachine({
  id: 'vending',
  initial: 'idle',
  context: {
    balance: 0,
    inventory: { cola: 10, chips: 5, candy: 8 },
    prices: { cola: 150, chips: 100, candy: 50 },
  },
  states: {
    idle: {
      on: {
        INSERT_COIN: {
          actions: 'addBalance',
        },
        SELECT_ITEM: [
          {
            target: 'dispensing',
            guard: { type: 'and', guards: ['hasEnoughMoney', 'hasInventory'] },
            actions: ['deductBalance', 'decrementInventory'],
          },
          {
            target: 'error',
          },
        ],
        CANCEL: {
          actions: 'returnChange',
        },
      },
    },
    dispensing: {
      after: {
        2000: 'idle', // 2秒後自動返回 idle
      },
      on: {
        DISPENSE_COMPLETE: 'idle',
      },
    },
    error: {
      after: {
        3000: 'idle',
      },
    },
  },
});
```

### 🎯 實作練習 6.3

1. 實作上述投幣機
2. 在 Svelte 中創建 UI
3. 添加視覺化狀態顯示
4. 測試各種場景 (餘額不足、庫存不足等)

---

## 6.13 本章小結

### XState 核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| States | 定義狀態 | `{ idle: {}, betting: {} }` |
| Events | 觸發轉換 | `{ type: 'START_BET' }` |
| Transitions | 狀態轉換 | `on: { START: 'active' }` |
| Guards | 條件檢查 | `guard: 'canBet'` |
| Actions | 執行邏輯 | `actions: 'updateBalance'` |
| Context | 儲存數據 | `context: { count: 0 }` |

### 你已經學會:
- ✅ 有限狀態機的概念和優勢
- ✅ XState 的基本使用
- ✅ 專案中的狀態機架構
- ✅ 子狀態機和狀態組合
- ✅ Context、Guards、Actions
- ✅ 使用狀態機控制 UI
- ✅ 不同獲勝類型的處理

### 🎯 作業

1. **分析遊戲狀態機**: 閱讀 [packages/utils-xstate/src/createPrimaryMachines.ts](../packages/utils-xstate/src/createPrimaryMachines.ts)
   - 畫出完整的狀態圖
   - 找出所有可能的狀態轉換
   - 思考為什麼這樣設計

2. **創建自訂狀態機**: 創建一個 Boss 戰狀態機
   - 階段: idle → phase1 → phase2 → phase3 → defeated
   - 每個階段有不同的攻擊模式
   - HP 降低時自動進入下一階段

3. **探索 UI 控制**: 查看 [packages/components-ui-pixi/src/](../packages/components-ui-pixi/src/)
   - 找出所有使用狀態機的組件
   - 理解如何根據狀態禁用按鈕
   - 思考如何優化用戶體驗

### 下一章預告

**第七章: 佈局系統 (Layout System)**
- 響應式設計原理
- Canvas 佈局系統
- 適配不同螢幕尺寸
- Portrait vs Landscape 佈局

---

## 📚 延伸閱讀

- [XState 官方文檔](https://stately.ai/docs/xstate)
- [有限狀態機](https://zh.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA)
- [狀態圖視覺化工具](https://stately.ai/viz)

---

[⬅️ 上一章: 事件驅動架構](./05-event-driven-architecture.md) | [返回目錄](./README.md) | [下一章: 佈局系統 ➡️](./07-layout-system.md)
