# 第二章：Svelte 5 基礎概念

## 學習目標

完成本章後,你將能夠:
- ✅ 理解 Svelte 5 的響應式系統 ($state, $derived, $effect)
- ✅ 掌握組件的生命週期與掛載機制
- ✅ 使用 Props 傳遞數據和 Snippets 傳遞內容
- ✅ 運用 Context API 在組件樹中共享狀態

---

## 2.1 Svelte 5 的革新

### Svelte 5 vs Svelte 4

Svelte 5 引入了全新的響應式系統,稱為 **Runes (符文)**:

| Svelte 4 | Svelte 5 | 用途 |
|----------|----------|------|
| `let count = 0` | `let count = $state(0)` | 響應式變數 |
| `$: doubled = count * 2` | `let doubled = $derived(count * 2)` | 衍生值 |
| `$: { console.log(count) }` | `$effect(() => { console.log(count) })` | 副作用 |
| `export let value` | `let { value } = $props()` | 組件 Props |

### 為什麼要改變?

- 🎯 **更明確**: 響應式變數一眼就能識別
- 🚀 **更高效**: 編譯器能更好地優化
- 🔧 **更靈活**: 可以在 `.svelte.ts` 檔案中使用

---

## 2.2 響應式系統：$state

### 基本用法

`$state()` 用來創建響應式變數,當值改變時會自動更新 UI:

```svelte
<script lang="ts">
  let count = $state(0);
  
  function increment() {
    count++; // UI 會自動更新
  }
</script>

<button onclick={increment}>
  點擊次數: {count}
</button>
```

### 專案實例：Symbol 組件

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts">
  let show = $state(true);
  let alpha = $state(1);
  let state = $state<'default' | 'dim' | 'bright'>('default');
  
  // 當 eventEmitter 發送事件時,這些值會改變並觸發重新渲染
</script>
```

### 🎯 實作練習 2.1

1. 打開 Storybook: `pnpm run storybook --filter=lines`
2. 找到 `COMPONENTS/Symbol/component`
3. 使用右側的 Controls 改變 `state` (default/dim/bright)
4. 觀察符號的視覺變化

### 物件和陣列的響應式

```svelte
<script lang="ts">
  // ❌ 錯誤：淺層響應式
  let user = $state({ name: 'Alice', age: 25 });
  user.age++; // 不會觸發更新
  
  // ✅ 正確：深層響應式
  let user = $state({ name: 'Alice', age: 25 });
  user = { ...user, age: user.age + 1 }; // 會觸發更新
  
  // 或使用 $state.raw 處理大型不可變數據
  let bigData = $state.raw({ /* 大量數據 */ });
</script>
```

---

## 2.3 衍生狀態：$derived

### 基本概念

`$derived()` 創建一個依賴於其他響應式變數的計算值:

```svelte
<script lang="ts">
  let count = $state(0);
  let doubled = $derived(count * 2);
  let message = $derived(count > 10 ? '很多' : '很少');
</script>

<p>Count: {count}</p>
<p>Doubled: {doubled}</p>
<p>Message: {message}</p>
```

### 專案實例：Layout 系統

查看 [packages/utils-layout/src/createLayout.svelte.ts](../packages/utils-layout/src/createLayout.svelte.ts):

```typescript
const stateLayoutDerived = {
  canvasSizes: () => ({ 
    width: innerWidth(), 
    height: innerHeight() 
  }),
  
  canvasRatio: () => {
    const sizes = canvasSizes();
    return sizes.width / sizes.height;
  },
  
  layoutType: () => {
    const ratio = canvasRatio();
    if (ratio > 1.5) return 'landscape';
    if (ratio < 0.75) return 'portrait';
    return 'square';
  },
};
```

**解析:**
- `canvasSizes` 依賴 `innerWidth` 和 `innerHeight`
- `canvasRatio` 依賴 `canvasSizes`
- `layoutType` 依賴 `canvasRatio`
- 當視窗大小改變時,整個鏈條自動更新

### 🎯 實作練習 2.2

1. 運行 `pnpm run storybook --filter=lines`
2. 開啟瀏覽器的開發者工具
3. 調整瀏覽器視窗大小
4. 觀察遊戲如何響應不同的螢幕尺寸

---

## 2.4 副作用：$effect

### 基本用法

`$effect()` 用於執行副作用,當依賴的響應式變數改變時自動重新執行:

```svelte
<script lang="ts">
  let count = $state(0);
  
  $effect(() => {
    console.log(`Count 改變為: ${count}`);
    
    // 清理函數 (可選)
    return () => {
      console.log('清理上一次的 effect');
    };
  });
</script>
```

### 專案實例：Game 組件

查看 [apps/lines/src/components/Game.svelte](../apps/lines/src/components/Game.svelte):

```svelte
<script lang="ts">
  import { untrack } from 'svelte';
  
  $effect(() => {
    context.stateApp.pixiApplication = untrack(() => app);
  });
  
  $effect(() => {
    context.stateXstate.value = snapshot.value;
  });
</script>
```

**注意 `untrack()`:**
- 防止 `app` 成為 effect 的依賴
- 只有 `context.stateApp.pixiApplication` 改變時才觸發

### 常見陷阱

```svelte
<script lang="ts">
  let count = $state(0);
  
  // ❌ 錯誤：無限循環
  $effect(() => {
    count++; // effect 改變依賴,導致再次觸發
  });
  
  // ✅ 正確：只讀取不修改
  $effect(() => {
    console.log(count);
  });
</script>
```

---

## 2.5 組件生命週期

### Svelte 5 的生命週期

在 Svelte 5 中,生命週期概念簡化了:

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  
  // 組件掛載時執行
  onMount(() => {
    console.log('組件已掛載');
    
    // 返回清理函數 (組件銷毀時執行)
    return () => {
      console.log('組件即將銷毀');
    };
  });
  
  // 使用 $effect 替代其他生命週期
  $effect(() => {
    // 每次響應式變數改變時執行
  });
</script>
```

### 專案實例：EventEmitter 訂閱

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

**這個模式很重要:**
- `onMount` 時訂閱事件
- 返回清理函數取消訂閱
- 避免記憶體洩漏

### 🎯 實作練習 2.3

查看 [apps/lines/src/components/FreeSpinCounter.svelte](../apps/lines/src/components/FreeSpinCounter.svelte):

```svelte
<script lang="ts">
  // 在組件掛載時訂閱事件
  context.eventEmitter.subscribeOnMount({
    freeSpinCounterShow: () => (show = true),
    freeSpinCounterHide: () => (show = false),
    freeSpinCounterUpdate: (emitterEvent) => {
      if (emitterEvent.current !== undefined) current = emitterEvent.current;
      if (emitterEvent.total !== undefined) total = emitterEvent.total;
    },
  });
</script>
```

**思考:** 為什麼不需要手動調用清理函數?

---

## 2.6 Props：組件通信

### Svelte 5 的 Props 語法

```svelte
<!-- ParentComponent.svelte -->
<script lang="ts">
  import ChildComponent from './ChildComponent.svelte';
</script>

<ChildComponent name="Alice" age={25} />

<!-- ChildComponent.svelte -->
<script lang="ts">
  // Svelte 5 新語法
  let { name, age = 18 } = $props();
  
  // TypeScript 類型
  interface Props {
    name: string;
    age?: number;
  }
  let { name, age = 18 }: Props = $props();
</script>

<p>{name} 今年 {age} 歲</p>
```

### 專案實例：Symbol 組件

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts" module>
  export interface Props {
    x?: number;
    y?: number;
    index?: [number, number];
    rawSymbol?: RawSymbol;
    state?: SymbolStateType;
  }
</script>

<script lang="ts">
  let {
    x = 0,
    y = 0,
    index = [0, 0],
    rawSymbol = { name: 'H1' },
    state: stateProps = 'default',
  }: Props = $props();
</script>
```

**注意:**
- `export interface Props` 在 `<script lang="ts" module>` 中
- 使用解構語法獲取 props
- 可以設定預設值

### 🎯 實作練習 2.4

1. 打開 `COMPONENTS/Symbol/component` 在 Storybook
2. 使用 Controls 改變不同的 props
3. 觀察組件如何響應 props 的變化

---

## 2.7 Snippets：內容傳遞

### 什麼是 Snippets?

Snippets 是 Svelte 5 中替代 slots 的新機制:

```svelte
<!-- ParentComponent.svelte -->
<script lang="ts">
  import Card from './Card.svelte';
</script>

<Card>
  {#snippet header()}
    <h1>標題</h1>
  {/snippet}
  
  {#snippet content()}
    <p>內容</p>
  {/snippet}
</Card>

<!-- Card.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  
  let { header, content }: { 
    header?: Snippet; 
    content?: Snippet 
  } = $props();
</script>

<div class="card">
  {#if header}
    {@render header()}
  {/if}
  
  {#if content}
    {@render content()}
  {/if}
</div>
```

### 專案實例：UI 組件

查看 [packages/components-ui-pixi/src/UI.svelte](../packages/components-ui-pixi/src/UI.svelte):

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  
  let { 
    gameName, 
    logo 
  }: { 
    gameName?: Snippet; 
    logo?: Snippet 
  } = $props();
</script>

<Container>
  {#if gameName}
    {@render gameName()}
  {/if}
  
  {#if logo}
    {@render logo()}
  {/if}
</Container>
```

使用方式:

```svelte
<UI>
  {#snippet gameName()}
    <UiGameName name="我的遊戲" />
  {/snippet}
  
  {#snippet logo()}
    <Text text="LOGO" />
  {/snippet}
</UI>
```

---

## 2.8 Context API：跨組件狀態共享

### 為什麼需要 Context?

當多層組件需要共享狀態時,逐層傳遞 props 很麻煩:

```
App → Board → Reel → Symbol
  └─→ Props → Props → Props (繁瑣)
```

使用 Context:

```
App (setContext)
  └─→ Symbol (getContext) 直接獲取
```

### Context 的基本用法

```svelte
<!-- App.svelte -->
<script lang="ts">
  import { setContext } from 'svelte';
  import Child from './Child.svelte';
  
  const myContext = { message: 'Hello' };
  setContext('myKey', myContext);
</script>

<Child />

<!-- Child.svelte (或任何子孫組件) -->
<script lang="ts">
  import { getContext } from 'svelte';
  
  const context = getContext<{ message: string }>('myKey');
</script>

<p>{context.message}</p>
```

### 專案實例：遊戲 Context

查看 [apps/lines/src/game/context.ts](../apps/lines/src/game/context.ts):

```typescript
import { setContext as setSvelteContext, getContext as getSvelteContext } from 'svelte';

// 設置所有需要的 Context
export const setContext = () => {
  setContextEventEmitter<EmitterEvent>({ eventEmitter });
  setContextXstate({ stateXstate, stateXstateDerived });
  setContextLayout({ stateLayout, stateLayoutDerived });
  setContextApp({ stateApp });
};

// 獲取所有 Context
export const getContext = () => {
  return {
    eventEmitter: getContextEventEmitter<EmitterEvent>(),
    ...getContextXstate(),
    ...getContextLayout(),
    ...getContextApp(),
  };
};
```

在入口處設置:

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { setContext } from '../game/context';
  import Game from '../components/Game.svelte';
  
  setContext(); // 設置一次
</script>

<Game />
```

在任何子組件中使用:

```svelte
<!-- Symbol.svelte -->
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext(); // 獲取所有 context
  
  // 使用 eventEmitter
  context.eventEmitter.subscribeOnMount({ /* ... */ });
  
  // 使用 layout
  const isPortrait = context.stateLayoutDerived.layoutType() === 'portrait';
</script>
```

### 🎯 實作練習 2.5

1. 打開 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte)
2. 找到 `const context = getContext();`
3. 查看它如何使用 `context.eventEmitter`
4. 思考: 如果不用 Context,需要多少層 props 傳遞?

---

## 2.9 模組級別的 Script

### `<script lang="ts" module>`

Svelte 5 允許在組件中定義模組級別的代碼:

```svelte
<!-- MyComponent.svelte -->
<script lang="ts" module>
  // 這些代碼在模組載入時執行一次
  export interface Props {
    name: string;
  }
  
  export type EmitterEvent = 
    | { type: 'show' }
    | { type: 'hide' };
  
  const CONSTANT = 100; // 所有實例共享
</script>

<script lang="ts">
  // 這些代碼在每個組件實例創建時執行
  let { name }: Props = $props();
  let count = $state(0);
</script>
```

### 專案實例：類型定義

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts" module>
  // 導出類型供其他文件使用
  export interface Props {
    x?: number;
    y?: number;
    state?: SymbolStateType;
  }
  
  export type EmitterEventSymbol =
    | { type: 'symbolShow'; index: [number, number] }
    | { type: 'symbolHide'; index: [number, number] };
</script>

<script lang="ts">
  // 組件邏輯
  let { x, y, state }: Props = $props();
</script>
```

**優點:**
- 類型定義可以被其他文件導入
- 常數在所有實例間共享,節省記憶體
- 清晰分離模組級別和實例級別的代碼

---

## 2.10 實戰：創建一個簡單組件

### 📝 任務說明

創建一個計數器組件,整合本章所學的概念:

```svelte
<!-- Counter.svelte -->
<script lang="ts" module>
  export interface Props {
    initialCount?: number;
    step?: number;
  }
</script>

<script lang="ts">
  import { onMount } from 'svelte';
  
  // Props
  let { initialCount = 0, step = 1 }: Props = $props();
  
  // State
  let count = $state(initialCount);
  let clicks = $state(0);
  
  // Derived
  let total = $derived(count * clicks);
  let message = $derived(count > 10 ? '高' : '低');
  
  // Effect
  $effect(() => {
    console.log(`Count: ${count}, Message: ${message}`);
  });
  
  // Lifecycle
  onMount(() => {
    console.log('Counter 已掛載');
    return () => console.log('Counter 即將銷毀');
  });
  
  // Methods
  function increment() {
    count += step;
    clicks++;
  }
  
  function reset() {
    count = initialCount;
    clicks = 0;
  }
</script>

<div class="counter">
  <p>計數: {count}</p>
  <p>點擊次數: {clicks}</p>
  <p>總和: {total}</p>
  <p>狀態: {message}</p>
  
  <button onclick={increment}>增加 {step}</button>
  <button onclick={reset}>重置</button>
</div>

<style>
  .counter {
    padding: 20px;
    border: 2px solid #333;
    border-radius: 8px;
  }
  
  button {
    margin: 5px;
    padding: 10px 20px;
  }
</style>
```

### 🎯 實作練習 2.6

1. 在 `apps/lines/src/components/` 創建 `Counter.svelte`
2. 複製上述代碼
3. 在 Storybook 中創建測試故事
4. 實驗不同的 props 值

---

## 2.11 本章小結

### 核心概念回顧

| 概念 | 用途 | 範例 |
|------|------|------|
| `$state` | 響應式變數 | `let count = $state(0)` |
| `$derived` | 計算值 | `let doubled = $derived(count * 2)` |
| `$effect` | 副作用 | `$effect(() => console.log(count))` |
| `$props` | 接收 props | `let { name } = $props()` |
| `onMount` | 生命週期 | `onMount(() => { /* ... */ })` |
| `setContext/getContext` | 跨組件共享 | `setContext('key', value)` |
| Snippets | 內容傳遞 | `{#snippet name()}{/snippet}` |

### 你已經學會:
- ✅ Svelte 5 的響應式系統 (Runes)
- ✅ 組件的生命週期管理
- ✅ Props 和 Snippets 的使用
- ✅ Context API 的應用場景
- ✅ 在專案中找到實際範例

### 🎯 作業

1. **分析組件**: 打開 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte)
   - 找出所有 `$state` 變數
   - 找出所有 `$derived` 值
   - 找出它如何使用 Context

2. **修改計數器**: 擴展 Counter.svelte
   - 添加一個 `max` prop,限制最大值
   - 添加一個 `onCountChange` callback prop
   - 使用 `$effect` 在達到 max 時顯示警告

3. **探索 EventEmitter**: 閱讀 [packages/utils-event-emitter/src/createEventEmitter.ts](../packages/utils-event-emitter/src/createEventEmitter.ts)
   - 理解 `subscribeOnMount` 的實作
   - 為什麼要在 `onMount` 中訂閱?

### 下一章預告

**第三章: PixiJS 基礎與 Canvas 渲染**
- PixiJS 的核心概念 (Application, Container, Sprite)
- 渲染循環和動畫
- 座標系統和定位
- 紋理和圖片處理

---

## 📚 延伸閱讀

- [Svelte 5 官方文檔 - Runes](https://svelte.dev/docs/svelte/$state)
- [Svelte 5 官方文檔 - Context](https://svelte.dev/docs/svelte/context)
- [Svelte 5 官方文檔 - Snippets](https://svelte.dev/docs/svelte/snippet)

---

[⬅️ 上一章: 環境設置與專案架構](./01-setup-and-architecture.md) | [返回目錄](./README.md) | [下一章: PixiJS 基礎 ➡️](./03-pixijs-basics.md)
