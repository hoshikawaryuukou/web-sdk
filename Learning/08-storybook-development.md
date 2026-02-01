# 第八章：Storybook 測試與開發

## 學習目標

完成本章後,你將能夠:
- ✅ 理解 Storybook 的核心價值
- ✅ 掌握組件隔離開發技巧
- ✅ 創建和組織 Stories
- ✅ 測試不同場景和狀態

---

## 8.1 什麼是 Storybook?

### 8.1.1 核心概念

**Storybook** 是一個組件開發環境:

```
傳統開發流程           Storybook 開發流程
┌─────────────┐        ┌─────────────┐
│ 開發組件    │        │ 開發組件    │
│      ↓      │        │      ↓      │
│ 整合到遊戲  │   vs   │ 在 Story 測試│
│      ↓      │        │      ↓      │
│ 運行遊戲    │        │ 快速迭代    │
│      ↓      │        │      ↓      │
│ 測試和除錯  │        │ 整合到遊戲  │
└─────────────┘        └─────────────┘
```

**優勢:**
- ✅ 組件隔離開發
- ✅ 快速視覺化測試
- ✅ 不需要完整遊戲環境
- ✅ 方便測試邊緣情況

### 8.1.2 專案中的應用

查看 [apps/lines/src/stories/](../apps/lines/src/stories/):

```
stories/
├── ComponentsBoard.stories.svelte
├── ComponentsGame.stories.svelte
├── ComponentsSymbol.stories.svelte
├── ModeBaseBook.stories.svelte
├── ModeBaseBookEvent.stories.svelte
├── ModeBonusBook.stories.svelte
└── ModeBonusBookEvent.stories.svelte
```

---

## 8.2 Story 的結構

### 8.2.1 基本 Story

```svelte
<!-- SimpleButton.stories.svelte -->
<script lang="ts">
  import { Meta, Story } from '@storybook/addon-svelte-csf';
  import SimpleButton from './SimpleButton.svelte';
</script>

<Meta
  title="Components/SimpleButton"
  component={SimpleButton}
/>

<Story name="Default">
  <SimpleButton label="Click Me" />
</Story>

<Story name="Disabled">
  <SimpleButton label="Disabled" disabled={true} />
</Story>

<Story name="Large">
  <SimpleButton label="Large Button" size="large" />
</Story>
```

### 8.2.2 帶參數的 Story

```svelte
<script lang="ts">
  import { Meta, Story, Template } from '@storybook/addon-svelte-csf';
  import Symbol from './Symbol.svelte';
</script>

<Meta
  title="Components/Symbol"
  component={Symbol}
  argTypes={{
    state: {
      control: { type: 'select' },
      options: ['default', 'dim', 'bright', 'win'],
    },
    symbolName: {
      control: { type: 'select' },
      options: ['H1', 'H2', 'H3', 'H4', 'L1', 'L2'],
    },
  }}
/>

<Template let:args>
  <Symbol {...args} />
</Template>

<Story
  name="Interactive"
  args={{
    state: 'default',
    symbolName: 'H1',
    x: 100,
    y: 100,
  }}
/>
```

---

## 8.3 專案 Story 類型

### 8.3.1 COMPONENTS Stories

測試單個組件:

```svelte
<!-- ComponentsSymbol.stories.svelte -->
<script lang="ts">
  import { Meta, Story } from '@storybook/addon-svelte-csf';
  import Game from '../components/Game.svelte';
  import { setContext } from '../game/context';
  
  setContext();
</script>

<Meta title="COMPONENTS/Symbol" />

<!-- 測試單個符號 -->
<Story name="component">
  <Game>
    <Symbol rawSymbol={{ name: 'H1' }} state="default" />
  </Game>
</Story>

<!-- 測試所有符號 -->
<Story name="symbols">
  <Game>
    {#each ['H1', 'H2', 'H3', 'H4', 'L1', 'L2'] as name, i}
      <Symbol
        rawSymbol={{ name }}
        x={i * 150}
        y={0}
      />
    {/each}
  </Game>
</Story>

<!-- 測試所有狀態 -->
<Story name="states">
  <Game>
    {#each ['default', 'dim', 'bright'] as state, i}
      <Symbol
        rawSymbol={{ name: 'H1' }}
        {state}
        x={i * 150}
        y={0}
      />
    {/each}
  </Game>
</Story>
```

### 8.3.2 MODE Stories

測試完整遊戲流程:

**Book Stories** - 測試完整的 book:

```svelte
<!-- ModeBaseBook.stories.svelte -->
<script lang="ts">
  import { Story } from '@storybook/addon-svelte-csf';
  import Game from '../components/Game.svelte';
  import { setContext } from '../game/context';
  import { playBookEvents } from 'utils-book';
  import books from './data/base_books';
  
  setContext();
  
  const templateArgs = (args) => ({
    skipLoadingScreen: true,
    ...args,
  });
</script>

<Story
  name="random"
  args={templateArgs({
    data: books[Math.floor(Math.random() * books.length)],
    action: async (data) => {
      await playBookEvents(data.events, { bookEvents: data.events });
    },
  })}
/>
```

**BookEvent Stories** - 測試單個 bookEvent:

```svelte
<!-- ModeBaseBookEvent.stories.svelte -->
<script lang="ts">
  import { Story } from '@storybook/addon-svelte-csf';
  import Game from '../components/Game.svelte';
  import { playBookEvent } from 'utils-book';
  import events from './data/base_events';
  
  setContext();
</script>

<Story
  name="reveal"
  args={{
    data: events.reveal,
    action: async (data) => {
      await playBookEvent(data, { bookEvents: [] });
    },
  }}
/>

<Story
  name="win"
  args={{
    data: events.win,
    action: async (data) => {
      await playBookEvent(data, { bookEvents: [] });
    },
  }}
/>
```

### 8.3.3 EmitterEvent Stories

測試單個 emitterEvent:

```svelte
<Story
  name="boardSpin"
  args={{
    action: async () => {
      await context.eventEmitter.broadcastAsync({
        type: 'boardSpin',
        duration: 2000,
      });
    },
  }}
/>
```

---

## 8.4 Action 按鈕

### 8.4.1 StoryAction 組件

查看 [packages/components-storybook/src/StoryAction.svelte](../packages/components-storybook/src/StoryAction.svelte):

```svelte
<script lang="ts">
  let { action, children }: Props = $props();
  
  let isRunning = $state(false);
  let isResolved = $state(false);
  
  async function handleClick() {
    if (isRunning) return;
    
    isRunning = true;
    isResolved = false;
    
    try {
      await action?.();
      isResolved = true;
    } catch (error) {
      console.error('Story action error:', error);
    } finally {
      isRunning = false;
    }
  }
</script>

<button onclick={handleClick} disabled={isRunning}>
  {#if isRunning}
    ⏳ Running...
  {:else if isResolved}
    ✅ Action is resolved
  {:else}
    ▶ Action
  {/if}
</button>

{@render children?.()}
```

### 8.4.2 使用 Action

```svelte
<Story
  name="spinReel"
  args={{
    action: async () => {
      // 設置初始狀態
      context.eventEmitter.broadcast({
        type: 'boardSetSymbols',
        board: initialBoard,
      });
      
      // 執行動畫
      await context.eventEmitter.broadcastAsync({
        type: 'reelSpin',
        duration: 2000,
      });
      
      // 完成
      context.eventEmitter.broadcast({
        type: 'reelStop',
      });
    },
  }}
/>
```

---

## 8.5 測試數據管理

### 8.5.1 Books 數據

查看 [apps/lines/src/stories/data/base_books.ts](../apps/lines/src/stories/data/base_books.ts):

```typescript
export default [
  {
    id: 1,
    payoutMultiplier: 0.0,
    events: [
      {
        index: 0,
        type: 'reveal',
        board: [
          [{ name: 'L2' }, { name: 'L1' }, { name: 'L4' }],
          [{ name: 'H1' }, { name: 'L5' }, { name: 'L2' }],
          [{ name: 'L3' }, { name: 'L5' }, { name: 'L3' }],
          [{ name: 'H4' }, { name: 'H3' }, { name: 'L4' }],
          [{ name: 'H3' }, { name: 'L3' }, { name: 'L3' }],
        ],
      },
      { index: 1, type: 'setTotalWin', amount: 0 },
      { index: 2, type: 'finalWin', amount: 0 },
    ],
  },
  // ... 更多 books
];
```

### 8.5.2 Events 數據

查看 [apps/lines/src/stories/data/base_events.ts](../apps/lines/src/stories/data/base_events.ts):

```typescript
export default {
  reveal: {
    type: 'reveal',
    board: [
      [{ name: 'H1' }, { name: 'H2' }, { name: 'H3' }],
      [{ name: 'H4' }, { name: 'L1' }, { name: 'L2' }],
      [{ name: 'L3' }, { name: 'L4' }, { name: 'L5' }],
      [{ name: 'H1' }, { name: 'H2' }, { name: 'H3' }],
      [{ name: 'H4' }, { name: 'L1' }, { name: 'L2' }],
    ],
  },
  win: {
    type: 'win',
    lines: [
      {
        lineIndex: 0,
        positions: [[0, 0], [1, 0], [2, 0], [3, 0], [4, 0]],
        multiplier: 10,
      },
    ],
  },
  // ... 更多 events
};
```

---

## 8.6 組件開發工作流

### 8.6.1 TDD 風格開發

1. **創建 Story**
```svelte
<Story name="WinLine">
  <WinLine
    positions={[[0,0], [1,0], [2,0]]}
    multiplier={5}
  />
</Story>
```

2. **實作組件**
```svelte
<!-- WinLine.svelte -->
<script lang="ts">
  let { positions, multiplier }: Props = $props();
</script>

<Container>
  {#each positions as [x, y]}
    <Graphics ... />
  {/each}
</Container>
```

3. **在 Storybook 中測試**
- 調整參數
- 測試邊緣情況
- 確認視覺效果

4. **整合到遊戲**

### 🎯 實作練習 8.1

創建一個 Coin 組件的完整開發流程:

```svelte
<!-- Coin.stories.svelte -->
<script lang="ts">
  import { Meta, Story } from '@storybook/addon-svelte-csf';
  import Coin from './Coin.svelte';
</script>

<Meta title="Components/Coin" component={Coin} />

<Story name="static">
  <Coin x={100} y={100} />
</Story>

<Story name="spinning">
  <Coin x={100} y={100} spinning={true} />
</Story>

<Story name="collected"
  args={{
    action: async () => {
      // 測試收集動畫
    },
  }}
/>
```

---

## 8.7 測試不同場景

### 8.7.1 狀態測試

```svelte
<Story name="allStates">
  <Container>
    {#each ['idle', 'hover', 'pressed', 'disabled'] as state, i}
      <Button
        x={i * 150}
        y={0}
        {state}
        label={state}
      />
    {/each}
  </Container>
</Story>
```

### 8.7.2 尺寸測試

```svelte
<Story name="allSizes">
  <Container>
    {#each ['small', 'medium', 'large'] as size, i}
      <Symbol
        x={i * 200}
        y={0}
        {size}
      />
    {/each}
  </Container>
</Story>
```

### 8.7.3 邊緣情況測試

```svelte
<Story name="edgeCases">
  <Container>
    <!-- 空數據 -->
    <WinLine positions={[]} />
    
    <!-- 單個位置 -->
    <WinLine positions={[[0,0]]} />
    
    <!-- 大量位置 -->
    <WinLine positions={generateManyPositions(100)} />
  </Container>
</Story>
```

---

## 8.8 Storybook Controls

### 8.8.1 配置 Controls

```svelte
<Meta
  title="Components/Symbol"
  component={Symbol}
  argTypes={{
    symbolName: {
      control: { type: 'select' },
      options: ['H1', 'H2', 'H3', 'H4', 'L1', 'L2', 'L3', 'L4', 'L5'],
      description: '符號名稱',
    },
    state: {
      control: { type: 'radio' },
      options: ['default', 'dim', 'bright', 'win'],
      description: '符號狀態',
    },
    x: {
      control: { type: 'range', min: 0, max: 800, step: 10 },
      description: 'X 座標',
    },
    y: {
      control: { type: 'range', min: 0, max: 600, step: 10 },
      description: 'Y 座標',
    },
    alpha: {
      control: { type: 'range', min: 0, max: 1, step: 0.1 },
      description: '透明度',
    },
    visible: {
      control: { type: 'boolean' },
      description: '是否可見',
    },
  }}
/>
```

### 8.8.2 Control 類型

| 類型 | 用途 | 範例 |
|------|------|------|
| text | 文字輸入 | `{ type: 'text' }` |
| number | 數字輸入 | `{ type: 'number' }` |
| range | 滑桿 | `{ type: 'range', min: 0, max: 100 }` |
| boolean | 開關 | `{ type: 'boolean' }` |
| select | 下拉選單 | `{ type: 'select', options: [...] }` |
| radio | 單選按鈕 | `{ type: 'radio', options: [...] }` |
| color | 顏色選擇器 | `{ type: 'color' }` |

---

## 8.9 最佳實踐

### 8.9.1 Story 組織

```
stories/
├── Components/          # 原子組件
│   ├── Button.stories.svelte
│   ├── Symbol.stories.svelte
│   └── Text.stories.svelte
├── Composite/           # 組合組件
│   ├── Board.stories.svelte
│   ├── ReelSet.stories.svelte
│   └── WinLines.stories.svelte
├── Features/            # 功能測試
│   ├── SpinFeature.stories.svelte
│   ├── WinFeature.stories.svelte
│   └── BonusFeature.stories.svelte
└── Flows/              # 完整流程
    ├── BaseGame.stories.svelte
    └── BonusGame.stories.svelte
```

### 8.9.2 命名規範

```svelte
<!-- ✅ 良好：清晰描述 -->
<Story name="default" />
<Story name="withWinAnimation" />
<Story name="multipleLinesWin" />

<!-- ❌ 不好：不清楚 -->
<Story name="test1" />
<Story name="story2" />
<Story name="example" />
```

### 8.9.3 獨立性

```svelte
<!-- ✅ 良好：完全獨立 -->
<Story name="spinAnimation">
  <Game initialState="idle">
    <Board board={testBoard} />
  </Game>
</Story>

<!-- ❌ 不好：依賴其他 story -->
<Story name="afterSpin">
  <!-- 假設已經執行過 spinAnimation -->
  <Board />
</Story>
```

---

## 8.10 除錯技巧

### 8.10.1 Console 輸出

```svelte
<Story
  name="debugSymbol"
  args={{
    action: async () => {
      console.log('Before animation');
      await animateSymbol();
      console.log('After animation');
    },
  }}
/>
```

### 8.10.2 視覺化輔助

```svelte
<Story name="withDebugGrid">
  <Container>
    <!-- 繪製網格 -->
    <Graphics oncreate={(g) => {
      for (let x = 0; x < 800; x += 100) {
        g.moveTo(x, 0);
        g.lineTo(x, 600);
        g.stroke({ width: 1, color: 0x333333 });
      }
      for (let y = 0; y < 600; y += 100) {
        g.moveTo(0, y);
        g.lineTo(800, y);
        g.stroke({ width: 1, color: 0x333333 });
      }
    }} />
    
    <Symbol x={100} y={100} />
  </Container>
</Story>
```

---

## 8.11 本章小結

### Storybook 核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| Story | 測試單一場景 | `<Story name="default">` |
| Args | 參數化 Story | `args={{ value: 100 }}` |
| Controls | 互動式調整 | `argTypes={{ ... }}` |
| Action | 測試互動 | `action: async () => {}` |
| Template | 重用結構 | `<Template let:args>` |

### 你已經學會:
- ✅ Storybook 的價值和應用
- ✅ Story 的結構和類型
- ✅ 組件隔離開發流程
- ✅ 測試數據管理
- ✅ Controls 的使用
- ✅ 最佳實踐和除錯技巧

### 🎯 作業

1. **探索現有 Stories**: 運行 `pnpm run storybook --filter=lines`
   - 瀏覽所有 Story 類別
   - 測試不同的 Controls
   - 理解每個 Story 的目的

2. **創建新 Story**: 為 WinLine 組件創建完整的 Stories
   - 單條線
   - 多條線
   - 不同倍數
   - 動畫效果

3. **TDD 開發**: 使用 Storybook 開發一個新組件
   - 先寫 Story
   - 再實作組件
   - 測試所有場景

### 下一章預告

**第九章: 遊戲開發流程 (Book Events)**
- Book 和 BookEvent 深入
- 開發新 BookEvent
- 測試和整合流程
- 常見模式和技巧

---

## 📚 延伸閱讀

- [Storybook 官方文檔](https://storybook.js.org/docs)
- [Storybook for Svelte](https://github.com/storybookjs/storybook/tree/next/code/frameworks/svelte-vite)
- [Component-Driven Development](https://www.componentdriven.org/)

---

[⬅️ 上一章: 佈局系統](./07-layout-system.md) | [返回目錄](./README.md) | [下一章: 遊戲開發流程 ➡️](./09-game-development-workflow.md)