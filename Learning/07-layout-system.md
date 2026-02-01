# 第七章：佈局系統 (Layout System)

## 學習目標

完成本章後,你將能夠:
- ✅ 理解響應式設計在 Canvas 中的應用
- ✅ 掌握佈局系統的實作原理
- ✅ 適配不同螢幕尺寸和方向
- ✅ 實現 Portrait/Landscape 佈局切換

---

## 7.1 為什麼需要佈局系統?

### 7.1.1 Canvas vs HTML 的佈局差異

**HTML (自動佈局):**
```html
<div style="display: flex; justify-content: center;">
  <button>按鈕</button> <!-- 自動居中 -->
</div>
```

**Canvas (手動佈局):**
```typescript
// 需要手動計算位置
const button = new PIXI.Sprite(texture);
button.x = canvas.width / 2; // 手動居中
button.y = canvas.height / 2;
button.anchor.set(0.5);
```

**問題:**
- 🔴 不同螢幕尺寸需要不同佈局
- 🔴 手機直向/橫向需要調整
- 🔴 手動計算容易出錯
- 🔴 維護困難

### 7.1.2 佈局系統的解決方案

```
視窗尺寸改變
    ↓
Layout System 監聽
    ↓
計算佈局類型
    ↓
組件響應式更新
    ↓
UI 自動調整
```

---

## 7.2 佈局系統架構

### 7.2.1 核心概念

查看 [packages/utils-layout/src/createLayout.svelte.ts](../packages/utils-layout/src/createLayout.svelte.ts):

```typescript
import { innerWidth, innerHeight } from 'svelte/reactivity/window';

const stateLayout = $state({
  showLoadingScreen: true,
});

const stateLayoutDerived = {
  canvasSizes,      // 畫布尺寸
  canvasRatio,      // 畫布比例
  canvasRatioType,  // 比例類型 (ultra-wide, wide, standard...)
  canvasSizeType,   // 尺寸類型 (small, medium, large...)
  layoutType,       // 佈局類型 (portrait, landscape, square)
  isStacked,        // 是否堆疊佈局
  mainLayout,       // 主佈局配置
  normalBackgroundLayout,   // 橫向背景佈局
  portraitBackgroundLayout, // 直向背景佈局
};
```

### 7.2.2 響應式尺寸

```typescript
const canvasSizes = () => ({
  width: innerWidth(),
  height: innerHeight(),
});

// innerWidth/innerHeight 是 Svelte 的響應式變數
// 當視窗大小改變時自動更新
```

---

## 7.3 畫布比例計算

### 7.3.1 計算比例

```typescript
const canvasRatio = () => {
  const sizes = canvasSizes();
  return sizes.width / sizes.height;
};

// 範例
// 1920x1080 → 1.78 (16:9)
// 1024x768  → 1.33 (4:3)
// 768x1024  → 0.75 (3:4)
```

### 7.3.2 比例類型

```typescript
const canvasRatioType = () => {
  const ratio = canvasRatio();
  
  if (ratio >= 2.2) return 'ultra-wide';  // 21:9
  if (ratio >= 1.7) return 'wide';        // 16:9
  if (ratio >= 1.5) return 'standard';    // 3:2
  if (ratio >= 1.2) return 'moderate';    // 5:4
  if (ratio >= 0.9) return 'square';      // 1:1
  if (ratio >= 0.6) return 'tall';        // 4:5
  return 'ultra-tall';                     // 9:16
};
```

**視覺化:**
```
ultra-wide  ▓▓▓▓▓▓▓▓▓▓▓▓  (21:9)
wide        ▓▓▓▓▓▓▓▓      (16:9)
standard    ▓▓▓▓▓▓        (3:2)
moderate    ▓▓▓▓          (5:4)
square      ▓▓▓           (1:1)
tall        ▓▓            (4:5)
ultra-tall  ▓             (9:16)
```

### 🎯 實作練習 7.1

創建一個比例檢測器:

```svelte
<script lang="ts">
  import { innerWidth, innerHeight } from 'svelte/reactivity/window';
  
  let ratio = $derived(innerWidth() / innerHeight());
  let ratioType = $derived(
    ratio >= 2.2 ? 'ultra-wide' :
    ratio >= 1.7 ? 'wide' :
    ratio >= 1.5 ? 'standard' :
    ratio >= 1.2 ? 'moderate' :
    ratio >= 0.9 ? 'square' :
    ratio >= 0.6 ? 'tall' : 'ultra-tall'
  );
</script>

<div>
  <p>視窗尺寸: {innerWidth()} x {innerHeight()}</p>
  <p>比例: {ratio.toFixed(2)}</p>
  <p>比例類型: {ratioType}</p>
</div>
```

---

## 7.4 佈局類型

### 7.4.1 三種主要佈局

```typescript
const layoutType = () => {
  const ratio = canvasRatio();
  
  if (ratio > 1.2) return 'landscape'; // 橫向
  if (ratio < 0.8) return 'portrait';  // 直向
  return 'square';                      // 方形
};
```

**視覺化:**
```
Landscape (橫向)          Portrait (直向)
┌───────────────┐         ┌──────┐
│  Game Board   │         │ Logo │
│               │         │      │
│   UI Below    │         │Board │
└───────────────┘         │      │
                          │  UI  │
                          └──────┘
```

### 7.4.2 堆疊佈局判斷

```typescript
const isStacked = () => {
  return layoutType() === 'portrait' || canvasSizeType() === 'small';
};
```

**用途:**
- Portrait: UI 上下堆疊
- Small screen: 簡化佈局
- Landscape: UI 左右並排

---

## 7.5 尺寸類型

### 7.5.1 計算尺寸類型

```typescript
const canvasSizeType = () => {
  const sizes = canvasSizes();
  const minDimension = Math.min(sizes.width, sizes.height);
  
  if (minDimension < 480) return 'tiny';    // 手機小螢幕
  if (minDimension < 768) return 'small';   // 手機
  if (minDimension < 1024) return 'medium'; // 平板
  if (minDimension < 1440) return 'large';  // 桌機
  return 'xlarge';                           // 大螢幕
};
```

### 7.5.2 響應式字體大小

```typescript
const REM = $derived(() => {
  const sizeType = context.stateLayoutDerived.canvasSizeType();
  
  switch (sizeType) {
    case 'tiny': return 12;
    case 'small': return 14;
    case 'medium': return 16;
    case 'large': return 18;
    case 'xlarge': return 20;
    default: return 16;
  }
});
```

---

## 7.6 主佈局配置

### 7.6.1 佈局參數

查看 [packages/utils-layout/src/createLayout.svelte.ts](../packages/utils-layout/src/createLayout.svelte.ts):

```typescript
const mainLayout = () => {
  const sizes = canvasSizes();
  const ratio = canvasRatio();
  const type = layoutType();
  
  // 基礎配置
  const config = {
    width: sizes.width,
    height: sizes.height,
    ratio: ratio,
    type: type,
    
    // 安全區域 (避開瀏覽器 UI)
    safeAreaTop: 0,
    safeAreaBottom: 0,
    safeAreaLeft: 0,
    safeAreaRight: 0,
    
    // 內容區域
    contentWidth: sizes.width,
    contentHeight: sizes.height,
  };
  
  // Portrait 調整
  if (type === 'portrait') {
    config.safeAreaTop = 60;
    config.safeAreaBottom = 80;
    config.contentHeight = sizes.height - 140;
  }
  
  return config;
};
```

### 7.6.2 使用佈局配置

```svelte
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext();
  const layout = context.stateLayoutDerived.mainLayout();
  
  // 計算遊戲面板位置
  let boardX = $derived(layout.contentWidth / 2);
  let boardY = $derived(
    layout.safeAreaTop + (layout.contentHeight / 2)
  );
</script>

<Container x={boardX} y={boardY}>
  <Board />
</Container>
```

---

## 7.7 背景佈局

### 7.7.1 橫向背景

```typescript
const normalBackgroundLayout = () => {
  const sizes = canvasSizes();
  
  return {
    x: 0,
    y: 0,
    width: sizes.width,
    height: sizes.height,
    scale: calculateBackgroundScale(sizes, 'landscape'),
  };
};

const calculateBackgroundScale = (sizes, orientation) => {
  const bgWidth = orientation === 'landscape' ? 1920 : 1080;
  const bgHeight = orientation === 'landscape' ? 1080 : 1920;
  
  const scaleX = sizes.width / bgWidth;
  const scaleY = sizes.height / bgHeight;
  
  // 使用較大的縮放比例,確保覆蓋整個畫布
  return Math.max(scaleX, scaleY);
};
```

### 7.7.2 直向背景

```typescript
const portraitBackgroundLayout = () => {
  const sizes = canvasSizes();
  
  return {
    x: 0,
    y: 0,
    width: sizes.width,
    height: sizes.height,
    scale: calculateBackgroundScale(sizes, 'portrait'),
  };
};
```

### 專案實例：背景組件

```svelte
<script lang="ts">
  const context = getContext();
  
  let isPortrait = $derived(
    context.stateLayoutDerived.layoutType() === 'portrait'
  );
  
  let bgLayout = $derived(
    isPortrait 
      ? context.stateLayoutDerived.portraitBackgroundLayout()
      : context.stateLayoutDerived.normalBackgroundLayout()
  );
  
  let bgTexture = $derived(
    isPortrait 
      ? context.stateApp.loadedAssets['bg-portrait']
      : context.stateApp.loadedAssets['bg-landscape']
  );
</script>

<Sprite
  texture={bgTexture}
  x={bgLayout.x}
  y={bgLayout.y}
  scale={{ x: bgLayout.scale, y: bgLayout.scale }}
/>
```

---

## 7.8 組件佈局適配

### 7.8.1 條件渲染

```svelte
<script lang="ts">
  const context = getContext();
  
  let isPortrait = $derived(
    context.stateLayoutDerived.layoutType() === 'portrait'
  );
</script>

<Container>
  {#if isPortrait}
    <!-- 直向佈局 -->
    <Container x={centerX} y={topArea}>
      <Logo />
    </Container>
    <Container x={centerX} y={middleArea}>
      <Board />
    </Container>
    <Container x={centerX} y={bottomArea}>
      <UI />
    </Container>
  {:else}
    <!-- 橫向佈局 -->
    <Container x={leftArea} y={centerY}>
      <Logo />
    </Container>
    <Container x={centerX} y={centerY}>
      <Board />
    </Container>
    <Container x={rightArea} y={centerY}>
      <UI />
    </Container>
  {/if}
</Container>
```

### 7.8.2 響應式尺寸

```svelte
<script lang="ts">
  const context = getContext();
  
  let symbolSize = $derived(() => {
    const sizeType = context.stateLayoutDerived.canvasSizeType();
    
    switch (sizeType) {
      case 'tiny': return 60;
      case 'small': return 80;
      case 'medium': return 100;
      case 'large': return 120;
      case 'xlarge': return 140;
      default: return 100;
    }
  });
</script>

<Sprite
  texture={symbolTexture}
  width={symbolSize()}
  height={symbolSize()}
/>
```

### 專案實例：Board 佈局

查看 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte):

```svelte
<script lang="ts">
  const SYMBOL_SIZE = 150;
  const REEL_COUNT = 5;
  const ROW_COUNT = 3;
  
  const context = getContext();
  
  let boardWidth = $derived(SYMBOL_SIZE * REEL_COUNT);
  let boardHeight = $derived(SYMBOL_SIZE * ROW_COUNT);
  
  let boardX = $derived(context.stateLayoutDerived.canvasSizes().width / 2);
  let boardY = $derived(context.stateLayoutDerived.canvasSizes().height / 2);
</script>

<Container
  x={boardX}
  y={boardY}
  anchor={{ x: 0.5, y: 0.5 }}
>
  {#each board as reel, reelIndex}
    {#each reel as symbol, symbolIndex}
      <Symbol
        x={(reelIndex - 2) * SYMBOL_SIZE}
        y={(symbolIndex - 1) * SYMBOL_SIZE}
        {rawSymbol}
      />
    {/each}
  {/each}
</Container>
```

---

## 7.9 Layout Components

### 7.9.1 MainContainer

查看 [packages/components-layout/src/MainContainer.svelte](../packages/components-layout/src/MainContainer.svelte):

```svelte
<script lang="ts">
  import { Container } from 'pixi-svelte';
  import { getContextLayout } from 'utils-layout';
  
  const context = getContextLayout();
  
  let x = $derived(context.stateLayoutDerived.canvasSizes().width / 2);
  let y = $derived(context.stateLayoutDerived.canvasSizes().height / 2);
</script>

<Container {x} {y} anchor={{ x: 0.5, y: 0.5 }}>
  {@render children?.()}
</Container>
```

### 7.9.2 StackedContainer

堆疊佈局容器:

```svelte
<script lang="ts">
  interface Props {
    gap?: number;
    direction?: 'vertical' | 'horizontal';
    children?: Snippet;
  }
  
  let { gap = 10, direction = 'vertical' }: Props = $props();
  
  let childPositions = $derived(() => {
    // 計算每個子元素的位置
    return children.map((child, index) => {
      if (direction === 'vertical') {
        return { x: 0, y: index * gap };
      } else {
        return { x: index * gap, y: 0 };
      }
    });
  });
</script>

<Container>
  {#each children as child, index}
    <Container 
      x={childPositions[index].x} 
      y={childPositions[index].y}
    >
      {@render child()}
    </Container>
  {/each}
</Container>
```

---

## 7.10 REM 單位系統

### 7.10.1 什麼是 REM?

REM (Root Em) 是相對於根字體大小的單位:

```typescript
const REM = 16; // 基礎單位

// 使用 REM 定義尺寸
const buttonWidth = REM * 10;   // 160px
const padding = REM * 0.5;      // 8px
const fontSize = REM * 1.5;     // 24px
```

### 7.10.2 響應式 REM

```typescript
const REM = $derived(() => {
  const sizes = context.stateLayoutDerived.canvasSizes();
  const minDimension = Math.min(sizes.width, sizes.height);
  
  // 根據最小尺寸動態調整
  if (minDimension < 480) return 12;
  if (minDimension < 768) return 14;
  if (minDimension < 1024) return 16;
  if (minDimension < 1440) return 18;
  return 20;
});
```

### 7.10.3 專案實例

查看 [packages/components-ui-pixi/src/constants.ts](../packages/components-ui-pixi/src/constants.ts):

```typescript
export const REM = 16;

export const SIZES = {
  buttonHeight: REM * 3,      // 48px
  iconSize: REM * 2,          // 32px
  padding: REM * 1,           // 16px
  margin: REM * 0.5,          // 8px
  fontSize: {
    small: REM * 0.875,       // 14px
    medium: REM * 1,          // 16px
    large: REM * 1.25,        // 20px
  },
};
```

---

## 7.11 安全區域 (Safe Area)

### 7.11.1 為什麼需要安全區域?

手機有瀏海、Home Bar 等 UI 元素:

```
┌─────────────────┐
│  ▓▓▓ 瀏海 ▓▓▓   │ ← Safe Area Top
├─────────────────┤
│                 │
│  Content Area   │ ← 內容區域
│                 │
├─────────────────┤
│  ═════════════  │ ← Safe Area Bottom (Home Bar)
└─────────────────┘
```

### 7.11.2 計算安全區域

```typescript
const safeArea = () => {
  const isPortrait = layoutType() === 'portrait';
  const isSmall = canvasSizeType() === 'small' || canvasSizeType() === 'tiny';
  
  if (isPortrait && isSmall) {
    return {
      top: 60,    // 避開狀態列
      bottom: 80, // 避開 Home Bar
      left: 20,
      right: 20,
    };
  }
  
  return {
    top: 20,
    bottom: 20,
    left: 20,
    right: 20,
  };
};
```

### 7.11.3 使用安全區域

```svelte
<script lang="ts">
  const context = getContext();
  const layout = context.stateLayoutDerived.mainLayout();
  
  let contentTop = $derived(layout.safeAreaTop);
  let contentBottom = $derived(
    layout.height - layout.safeAreaBottom
  );
</script>

<Container y={contentTop}>
  <Header />
</Container>

<Container y={contentBottom} anchor={{ x: 0, y: 1 }}>
  <Footer />
</Container>
```

---

## 7.12 實戰：響應式遊戲介面

### 📝 任務說明

創建一個完全響應式的遊戲介面:

```svelte
<!-- ResponsiveGame.svelte -->
<script lang="ts">
  import { App, Container, Sprite, Text } from 'pixi-svelte';
  import { innerWidth, innerHeight } from 'svelte/reactivity/window';
  
  // 計算佈局
  let canvasWidth = $derived(innerWidth());
  let canvasHeight = $derived(innerHeight());
  let ratio = $derived(canvasWidth / canvasHeight);
  let isPortrait = $derived(ratio < 0.8);
  let isLandscape = $derived(ratio > 1.2);
  
  // REM 單位
  let rem = $derived(() => {
    const minDim = Math.min(canvasWidth, canvasHeight);
    if (minDim < 480) return 12;
    if (minDim < 768) return 14;
    if (minDim < 1024) return 16;
    return 18;
  });
  
  // 遊戲面板尺寸
  let boardSize = $derived(() => {
    const minDim = Math.min(canvasWidth, canvasHeight);
    return Math.min(minDim * 0.8, 600);
  });
  
  // 佈局位置
  let positions = $derived(() => {
    const centerX = canvasWidth / 2;
    const centerY = canvasHeight / 2;
    
    if (isPortrait) {
      return {
        logo: { x: centerX, y: rem() * 3 },
        board: { x: centerX, y: centerY },
        ui: { x: centerX, y: canvasHeight - rem() * 5 },
      };
    } else {
      return {
        logo: { x: rem() * 3, y: rem() * 3 },
        board: { x: centerX, y: centerY },
        ui: { x: canvasWidth - rem() * 3, y: centerY },
      };
    }
  });
</script>

<App width={canvasWidth} height={canvasHeight} resizeTo={window}>
  <!-- 背景 -->
  <Sprite
    texture={isPortrait ? bgPortrait : bgLandscape}
    width={canvasWidth}
    height={canvasHeight}
  />
  
  <!-- Logo -->
  <Container
    x={positions().logo.x}
    y={positions().logo.y}
    anchor={{ x: isPortrait ? 0.5 : 0, y: 0 }}
  >
    <Text
      text="GAME"
      style={{
        fontSize: rem() * 2,
        fill: 0xffffff,
      }}
    />
  </Container>
  
  <!-- 遊戲面板 -->
  <Container
    x={positions().board.x}
    y={positions().board.y}
    anchor={{ x: 0.5, y: 0.5 }}
  >
    <Sprite
      texture={boardTexture}
      width={boardSize()}
      height={boardSize()}
    />
  </Container>
  
  <!-- UI -->
  <Container
    x={positions().ui.x}
    y={positions().ui.y}
    anchor={{ x: isPortrait ? 0.5 : 1, y: isPortrait ? 1 : 0.5 }}
  >
    <Text
      text="Score: 1000"
      style={{
        fontSize: rem() * 1.5,
        fill: 0xffffff,
      }}
    />
  </Container>
</App>
```

### 🎯 實作練習 7.2

1. 實作上述響應式介面
2. 添加平板橫向特殊佈局
3. 添加桌面超寬螢幕佈局
4. 測試所有裝置尺寸

---

## 7.13 除錯工具

### 7.13.1 佈局資訊顯示

```svelte
<script lang="ts">
  import { Graphics } from 'pixi-svelte';
  
  const context = getContext();
  
  let showDebug = $state(true);
  
  function drawDebugInfo(g: PIXI.Graphics) {
    if (!showDebug) return;
    
    const sizes = context.stateLayoutDerived.canvasSizes();
    const layout = context.stateLayoutDerived.mainLayout();
    
    g.clear();
    
    // 繪製安全區域
    g.rect(
      layout.safeAreaLeft,
      layout.safeAreaTop,
      sizes.width - layout.safeAreaLeft - layout.safeAreaRight,
      sizes.height - layout.safeAreaTop - layout.safeAreaBottom
    );
    g.stroke({ width: 2, color: 0xff0000 });
    
    // 繪製中心線
    g.moveTo(sizes.width / 2, 0);
    g.lineTo(sizes.width / 2, sizes.height);
    g.stroke({ width: 1, color: 0x00ff00 });
    
    g.moveTo(0, sizes.height / 2);
    g.lineTo(sizes.width, sizes.height / 2);
    g.stroke({ width: 1, color: 0x00ff00 });
  }
</script>

<Graphics oncreate={drawDebugInfo} />
```

### 7.13.2 佈局資訊面板

```svelte
<div style="position: fixed; top: 10px; left: 10px; background: rgba(0,0,0,0.8); color: white; padding: 10px;">
  <p>尺寸: {innerWidth()} x {innerHeight()}</p>
  <p>比例: {(innerWidth() / innerHeight()).toFixed(2)}</p>
  <p>佈局類型: {context.stateLayoutDerived.layoutType()}</p>
  <p>尺寸類型: {context.stateLayoutDerived.canvasSizeType()}</p>
  <p>比例類型: {context.stateLayoutDerived.canvasRatioType()}</p>
</div>
```

---

## 7.14 本章小結

### 佈局系統核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| canvasSizes | 畫布尺寸 | `{ width, height }` |
| canvasRatio | 畫布比例 | `1.78 (16:9)` |
| layoutType | 佈局類型 | `portrait/landscape` |
| canvasSizeType | 尺寸類型 | `small/medium/large` |
| safeArea | 安全區域 | `{ top, bottom, left, right }` |
| REM | 響應式單位 | `16px` |

### 你已經學會:
- ✅ Canvas 佈局的特殊性
- ✅ 佈局系統的架構設計
- ✅ 響應式尺寸計算
- ✅ Portrait/Landscape 適配
- ✅ 安全區域處理
- ✅ REM 單位系統
- ✅ 響應式組件實作

### 🎯 作業

1. **分析專案佈局**: 打開 [packages/utils-layout/src/createLayout.svelte.ts](../packages/utils-layout/src/createLayout.svelte.ts)
   - 理解所有衍生值的計算邏輯
   - 找出可以優化的地方
   - 思考如何支援更多裝置

2. **創建響應式組件**: 創建一個 `ResponsivePanel` 組件
   - 自動適配不同螢幕尺寸
   - 支援 Portrait/Landscape 切換
   - 包含安全區域處理

3. **探索 UI 組件**: 查看 [packages/components-ui-pixi/src/UI.svelte](../packages/components-ui-pixi/src/UI.svelte)
   - 理解 UI 如何使用佈局系統
   - 找出所有佈局相關的計算
   - 思考如何改進用戶體驗

### 下一章預告

**第八章: Storybook 測試與開發**
- Storybook 基礎
- 組件隔離開發
- 測試不同場景
- 建立組件庫

---

## 📚 延伸閱讀

- [響應式設計原則](https://developer.mozilla.org/zh-TW/docs/Learn/CSS/CSS_layout/Responsive_Design)
- [安全區域 API](https://developer.mozilla.org/en-US/docs/Web/CSS/env)
- [REM vs PX](https://www.24a11y.com/2019/pixels-vs-relative-units-in-css-why-its-still-a-big-deal/)

---

[⬅️ 上一章: 狀態管理](./06-state-management-xstate.md) | [返回目錄](./README.md) | [下一章: Storybook 測試 ➡️](./08-storybook-development.md)
