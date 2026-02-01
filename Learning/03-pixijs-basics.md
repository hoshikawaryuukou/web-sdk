# 第三章：PixiJS 基礎與 Canvas 渲染

## 學習目標

完成本章後,你將能夠:
- ✅ 理解 PixiJS 的核心概念 (Application, Container, Sprite)
- ✅ 掌握 Canvas 渲染原理和渲染循環
- ✅ 運用座標系統和定位技巧
- ✅ 處理紋理、圖片和資源載入

---

## 3.1 什麼是 PixiJS?

### Canvas vs DOM

傳統的網頁使用 **DOM** (Document Object Model) 來渲染元素:
- ✅ 適合：文字、表單、少量元素
- ❌ 不適合：大量動畫、遊戲、複雜圖形

**PixiJS** 使用 **Canvas** (WebGL/Canvas 2D) 來渲染:
- ✅ 適合：遊戲、大量精靈、粒子效果
- ✅ 高效能：GPU 加速
- ✅ 跨平台：支援各種瀏覽器

### PixiJS 的優勢

```
傳統 DOM 渲染           PixiJS Canvas 渲染
┌────────────┐         ┌────────────┐
│ 1000 個    │  vs     │ 10000 個   │
│ <div>      │         │ Sprites    │
│ 卡頓...    │         │ 流暢 60fps │
└────────────┘         └────────────┘
```

---

## 3.2 PixiJS 核心概念

### 3.2.1 Application (應用程式)

`PIXI.Application` 是 PixiJS 的入口點:

```typescript
import * as PIXI from 'pixi.js';

const app = new PIXI.Application({
  width: 800,
  height: 600,
  backgroundColor: 0x1099bb,
  resolution: window.devicePixelRatio || 1,
});

// 將 canvas 添加到 DOM
document.body.appendChild(app.canvas);
```

### 專案實例：App 組件

查看 [packages/pixi-svelte/src/lib/components/App.svelte](../packages/pixi-svelte/src/lib/components/App.svelte):

```svelte
<script lang="ts">
  import * as PIXI from 'pixi.js';
  import { innerWidth, innerHeight } from 'svelte/reactivity/window';
  
  // 創建 Application
  const app = new PIXI.Application();
  
  await app.init({
    resizeTo: window,  // 自動調整大小
    resolution: window.devicePixelRatio,
    autoDensity: true,
    antialias: true,
  });
</script>

<canvas bind:this={app.canvas}></canvas>
```

### 3.2.2 Stage (舞台)

`app.stage` 是所有可視元素的根容器:

```typescript
// Stage 就像是 DOM 中的 <body>
app.stage
  └─ Container (遊戲場景)
      ├─ Container (背景層)
      ├─ Container (遊戲層)
      └─ Container (UI 層)
```

### 3.2.3 Container (容器)

`Container` 用於組織和分組顯示對象:

```typescript
const container = new PIXI.Container();
container.x = 100;
container.y = 100;
container.rotation = Math.PI / 4; // 45 度

// 所有子元素會繼承容器的變換
app.stage.addChild(container);
```

**重要特性:**
- 可以包含其他 Container 或 Sprite
- 變換會影響所有子元素
- 用於實現層級結構

### 3.2.4 Sprite (精靈)

`Sprite` 是顯示圖片的基本單位:

```typescript
// 從紋理創建 Sprite
const sprite = new PIXI.Sprite(texture);
sprite.x = 200;
sprite.y = 150;
sprite.anchor.set(0.5); // 設置錨點為中心

container.addChild(sprite);
```

---

## 3.3 座標系統

### 3.3.1 基本座標

PixiJS 使用標準的 2D 座標系統:

```
(0,0) ───────────────────> X
  │
  │     (100, 100)
  │        ●
  │
  │
  │
  v
  Y
```

- 原點 (0, 0) 在左上角
- X 軸向右增加
- Y 軸向下增加

### 3.3.2 錨點 (Anchor)

錨點決定物體的「中心點」:

```typescript
sprite.anchor.set(0, 0);    // 左上角 (預設)
sprite.anchor.set(0.5, 0.5); // 中心
sprite.anchor.set(1, 0);     // 右上角
sprite.anchor.set(1, 1);     // 右下角
```

視覺化:

```
anchor(0, 0)        anchor(0.5, 0.5)      anchor(1, 1)
┌────────┐         ┌────────┐            ┌────────┐
●(x,y)   │            │   ●   │            │     (x,y)●
│        │            │   │   │            │        │
└────────┘         └────────┘            └────────┘
```

### 專案實例：Symbol 定位

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<Sprite
  texture={context.stateApp.loadedAssets[`symbol-${symbol.name}`]}
  anchor={{ x: 0.5, y: 0.5 }}
  x={x}
  y={y}
  width={SYMBOL_SIZE}
  height={SYMBOL_SIZE}
/>
```

**為什麼用 anchor 0.5?**
- 符號從中心點旋轉
- 方便對齊到格子中心
- 縮放時保持中心位置

### 3.3.3 相對座標 vs 絕對座標

```typescript
// 相對座標：相對於父容器
const container = new PIXI.Container();
container.x = 100;
container.y = 100;

const sprite = new PIXI.Sprite(texture);
sprite.x = 50;  // 相對於 container
sprite.y = 50;

// sprite 的絕對座標是 (150, 150)
```

### 🎯 實作練習 3.1

1. 運行 `pnpm run storybook --filter=lines`
2. 打開 `COMPONENTS/Symbol/component`
3. 觀察符號如何定位在格子中
4. 思考：如果 anchor 改為 (0, 0) 會發生什麼?

---

## 3.4 紋理與資源載入

### 3.4.1 什麼是紋理?

**紋理 (Texture)** 是 GPU 可以使用的圖片數據:

```
圖片檔案 (PNG/JPG)
    ↓ 載入
PIXI.Texture
    ↓ 創建
PIXI.Sprite (可視元素)
```

### 3.4.2 載入資源

使用 `PIXI.Assets` 載入資源:

```typescript
// 單個資源
const texture = await PIXI.Assets.load('path/to/image.png');

// 多個資源
await PIXI.Assets.load([
  { alias: 'hero', src: 'hero.png' },
  { alias: 'enemy', src: 'enemy.png' },
  { alias: 'background', src: 'bg.jpg' },
]);

// 使用載入的紋理
const sprite = new PIXI.Sprite(PIXI.Assets.get('hero'));
```

### 專案實例：資源管理

查看 [packages/pixi-svelte/src/lib/createApp.svelte.ts](../packages/pixi-svelte/src/lib/createApp.svelte.ts):

```typescript
const stateApp = $state({
  assets: [] as Asset[],
  loaded: false,
  loadingProgress: 0,
  loadedAssets: {} as LoadedAssets,
});

// 載入所有資源
const loadedAssets = (await PIXI.Assets.load(
  stateApp.assets.map((asset) => asset.src),
  (progress) => {
    stateApp.loadingProgress = progress; // 更新進度
  }
)) as LoadedAssets;

stateApp.loadedAssets = loadedAssets;
stateApp.loaded = true;
```

### 3.4.3 資源類型

專案支援多種資源類型:

```typescript
type Asset = 
  | { alias: string; src: string }           // 圖片
  | { alias: string; src: string; data: SpineData } // Spine 動畫
  | { alias: string; src: string[]; data: SpriteSheetData } // 精靈圖
```

查看 [apps/lines/src/game/assets.ts](../apps/lines/src/game/assets.ts):

```typescript
export const assets: Asset[] = [
  // 圖片紋理
  { alias: 'symbol-H1', src: '/symbols/H1.png' },
  { alias: 'symbol-H2', src: '/symbols/H2.png' },
  
  // Spine 動畫
  {
    alias: 'button-spin',
    src: '/spines/button-spin.json',
    data: { spineAtlasFile: '/spines/button-spin.atlas' },
  },
];
```

### 🎯 實作練習 3.2

1. 打開 [apps/lines/src/game/assets.ts](../apps/lines/src/game/assets.ts)
2. 找到所有 symbol 相關的資源
3. 計算總共有多少個符號紋理
4. 思考：為什麼要預先載入而不是用到時才載入?

---

## 3.5 渲染循環

### 3.5.1 Ticker (計時器)

PixiJS 使用 `Ticker` 來管理渲染循環:

```typescript
app.ticker.add((delta) => {
  // 每幀執行
  // delta: 自上一幀經過的時間 (1 = 60fps)
  
  sprite.rotation += 0.01 * delta;
  sprite.x += 1 * delta;
});
```

### 3.5.2 FPS (每秒幀數)

```
60 FPS (理想)
  ↓
每幀 16.67ms
  ↓
流暢動畫
```

### 專案實例：動畫循環

雖然專案主要使用 Spine 和 Tween 動畫,但理解 Ticker 很重要:

```typescript
// 簡單的旋轉動畫
let rotation = 0;

app.ticker.add((delta) => {
  rotation += 0.05 * delta;
  sprite.rotation = rotation;
});
```

### 3.5.3 性能優化

```typescript
// ❌ 差勁：每幀創建新對象
app.ticker.add(() => {
  const newSprite = new PIXI.Sprite(texture); // 記憶體洩漏!
});

// ✅ 良好：重用對象
const sprite = new PIXI.Sprite(texture);
app.ticker.add(() => {
  sprite.x += 1;
});

// ✅ 更好：移除不需要的 ticker
const tickerFn = () => sprite.x += 1;
app.ticker.add(tickerFn);

// 完成後移除
app.ticker.remove(tickerFn);
```

---

## 3.6 變換屬性

### 3.6.1 位置 (Position)

```typescript
sprite.x = 100;
sprite.y = 200;

// 或使用 position
sprite.position.set(100, 200);
```

### 3.6.2 縮放 (Scale)

```typescript
sprite.scale.x = 2;    // 寬度放大 2 倍
sprite.scale.y = 0.5;  // 高度縮小一半

// 統一縮放
sprite.scale.set(1.5);
```

### 3.6.3 旋轉 (Rotation)

```typescript
sprite.rotation = Math.PI;      // 180 度 (弧度)
sprite.angle = 180;              // 180 度 (角度)

// 動畫旋轉
app.ticker.add(() => {
  sprite.rotation += 0.01;
});
```

### 3.6.4 透明度 (Alpha)

```typescript
sprite.alpha = 0.5;   // 半透明
sprite.alpha = 0;     // 完全透明
sprite.alpha = 1;     // 完全不透明
```

### 3.6.5 可見性 (Visibility)

```typescript
sprite.visible = false; // 隱藏 (但仍佔用記憶體)
sprite.visible = true;  // 顯示
```

### 專案實例：Symbol 狀態

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts">
  let show = $state(true);
  let alpha = $state(1);
  let state = $state<'default' | 'dim' | 'bright'>('default');
  
  // 根據狀態調整透明度
  $effect(() => {
    if (state === 'dim') alpha = 0.5;
    else if (state === 'bright') alpha = 1.2;
    else alpha = 1;
  });
</script>

<Container {alpha} visible={show}>
  <Sprite ... />
</Container>
```

### 🎯 實作練習 3.3

1. 打開 `COMPONENTS/Symbol/component` 在 Storybook
2. 使用 Controls 改變 `state` (default/dim/bright)
3. 觀察透明度變化
4. 嘗試在代碼中添加旋轉效果

---

## 3.7 圖形繪製 (Graphics)

### 3.7.1 基本形狀

PixiJS 可以繪製矢量圖形:

```typescript
const graphics = new PIXI.Graphics();

// 矩形
graphics.rect(0, 0, 100, 100);
graphics.fill(0xff0000);

// 圓形
graphics.circle(50, 50, 30);
graphics.fill(0x00ff00);

// 線條
graphics.moveTo(0, 0);
graphics.lineTo(100, 100);
graphics.stroke({ width: 2, color: 0x0000ff });

app.stage.addChild(graphics);
```

### 3.7.2 複雜形狀

```typescript
const graphics = new PIXI.Graphics();

// 多邊形
graphics.poly([
  0, 0,
  100, 50,
  50, 100,
  0, 100
]);
graphics.fill(0xffff00);

// 圓角矩形
graphics.roundRect(0, 0, 100, 100, 10);
graphics.fill(0xff00ff);
```

### 專案實例：除錯工具

Graphics 常用於除錯和原型設計:

```typescript
// 顯示碰撞邊界
const debugGraphics = new PIXI.Graphics();
debugGraphics.rect(sprite.x, sprite.y, sprite.width, sprite.height);
debugGraphics.stroke({ width: 2, color: 0xff0000 });
```

---

## 3.8 文字渲染

### 3.8.1 Text (基本文字)

```typescript
const text = new PIXI.Text({
  text: 'Hello PixiJS!',
  style: {
    fontFamily: 'Arial',
    fontSize: 24,
    fill: 0xffffff,
    align: 'center',
  }
});

text.x = 100;
text.y = 100;
app.stage.addChild(text);
```

### 3.8.2 BitmapText (點陣文字)

效能更好,適合遊戲:

```typescript
// 先註冊字體
PIXI.BitmapFont.from('MyFont', {
  fontFamily: 'Arial',
  fontSize: 32,
  fill: 0xffffff,
});

// 使用
const bitmapText = new PIXI.BitmapText({
  text: 'Score: 1000',
  style: { fontFamily: 'MyFont', fontSize: 32 }
});
```

### 專案實例：UI 文字

查看 [packages/pixi-svelte/src/lib/components/Text.svelte](../packages/pixi-svelte/src/lib/components/Text.svelte):

```svelte
<script lang="ts">
  import * as PIXI from 'pixi.js';
  
  let {
    text = '',
    style = {},
    anchor,
    x = 0,
    y = 0,
  }: Props = $props();
  
  const pixiText = new PIXI.Text({ text, style });
  
  $effect(() => {
    pixiText.text = text;
  });
</script>
```

---

## 3.9 互動性 (Interactivity)

### 3.9.1 啟用互動

```typescript
sprite.eventMode = 'static';  // 啟用互動
sprite.cursor = 'pointer';     // 改變游標

sprite.on('pointerdown', (event) => {
  console.log('點擊位置:', event.global.x, event.global.y);
});

sprite.on('pointerover', () => {
  sprite.tint = 0xff0000; // 滑鼠懸停變紅
});

sprite.on('pointerout', () => {
  sprite.tint = 0xffffff; // 恢復原色
});
```

### 3.9.2 事件類型

| 事件 | 觸發時機 |
|------|---------|
| `pointerdown` | 按下滑鼠/觸碰 |
| `pointerup` | 放開滑鼠/觸碰 |
| `pointermove` | 移動游標 |
| `pointerover` | 游標進入 |
| `pointerout` | 游標離開 |
| `tap` | 快速點擊 |

### 專案實例：按鈕互動

查看 [packages/components-ui-pixi/src/SimpleUiButton.svelte](../packages/components-ui-pixi/src/SimpleUiButton.svelte):

```svelte
<Container
  eventMode={disabled ? 'none' : 'static'}
  cursor="pointer"
  onpointerdown={() => {
    if (!disabled) onclick?.();
  }}
>
  <Sprite texture={buttonTexture} />
  <Text text={label} />
</Container>
```

### 🎯 實作練習 3.4

創建一個互動式精靈:

```typescript
const sprite = new PIXI.Sprite(texture);
sprite.eventMode = 'static';
sprite.anchor.set(0.5);
sprite.x = 400;
sprite.y = 300;

let isDragging = false;
let dragOffset = { x: 0, y: 0 };

sprite.on('pointerdown', (event) => {
  isDragging = true;
  dragOffset.x = event.global.x - sprite.x;
  dragOffset.y = event.global.y - sprite.y;
});

app.stage.on('pointermove', (event) => {
  if (isDragging) {
    sprite.x = event.global.x - dragOffset.x;
    sprite.y = event.global.y - dragOffset.y;
  }
});

app.stage.on('pointerup', () => {
  isDragging = false;
});

app.stage.addChild(sprite);
```

---

## 3.10 濾鏡效果 (Filters)

### 3.10.1 內建濾鏡

```typescript
import { BlurFilter, ColorMatrixFilter } from 'pixi.js';

// 模糊
const blurFilter = new BlurFilter();
blurFilter.blur = 5;
sprite.filters = [blurFilter];

// 灰階
const colorMatrix = new ColorMatrixFilter();
colorMatrix.greyscale(0.5, false);
sprite.filters = [colorMatrix];

// 組合濾鏡
sprite.filters = [blurFilter, colorMatrix];
```

### 3.10.2 常用濾鏡

| 濾鏡 | 效果 |
|------|------|
| `BlurFilter` | 模糊 |
| `ColorMatrixFilter` | 顏色調整 |
| `DisplacementFilter` | 扭曲 |
| `NoiseFilter` | 雜訊 |
| `AlphaFilter` | 透明度遮罩 |

### 專案實例：勝利特效

```typescript
// 勝利時添加發光效果
const glowFilter = new ColorMatrixFilter();
glowFilter.brightness(1.5, true);

winningSymbols.forEach(symbol => {
  symbol.filters = [glowFilter];
});
```

---

## 3.11 性能優化技巧

### 3.11.1 物件池 (Object Pool)

```typescript
// ❌ 差勁：不斷創建和銷毀
function spawnParticle() {
  const particle = new PIXI.Sprite(texture);
  // ... 使用後銷毀
  particle.destroy();
}

// ✅ 良好：重用物件
const particlePool: PIXI.Sprite[] = [];

function getParticle() {
  return particlePool.pop() || new PIXI.Sprite(texture);
}

function recycleParticle(particle: PIXI.Sprite) {
  particle.visible = false;
  particlePool.push(particle);
}
```

### 3.11.2 Culling (剔除)

```typescript
// 只渲染可見的物件
app.ticker.add(() => {
  sprites.forEach(sprite => {
    const bounds = sprite.getBounds();
    sprite.renderable = (
      bounds.x + bounds.width > 0 &&
      bounds.x < app.screen.width &&
      bounds.y + bounds.height > 0 &&
      bounds.y < app.screen.height
    );
  });
});
```

### 3.11.3 批次渲染

```typescript
// ✅ 良好：使用 ParticleContainer
const particles = new PIXI.ParticleContainer(10000, {
  scale: true,
  position: true,
  rotation: true,
  alpha: true,
});

// 可以高效渲染大量相似的精靈
for (let i = 0; i < 10000; i++) {
  const particle = new PIXI.Sprite(texture);
  particles.addChild(particle);
}
```

---

## 3.12 實戰：創建一個簡單場景

### 📝 任務說明

創建一個包含背景、角色和互動的簡單場景:

```typescript
import * as PIXI from 'pixi.js';

// 初始化應用
const app = new PIXI.Application();
await app.init({ width: 800, height: 600 });
document.body.appendChild(app.canvas);

// 載入資源
await PIXI.Assets.load([
  { alias: 'background', src: 'background.jpg' },
  { alias: 'hero', src: 'hero.png' },
  { alias: 'coin', src: 'coin.png' },
]);

// 背景
const background = new PIXI.Sprite(PIXI.Assets.get('background'));
background.width = app.screen.width;
background.height = app.screen.height;
app.stage.addChild(background);

// 角色
const hero = new PIXI.Sprite(PIXI.Assets.get('hero'));
hero.anchor.set(0.5);
hero.x = 400;
hero.y = 300;
hero.eventMode = 'static';
hero.cursor = 'pointer';
app.stage.addChild(hero);

// 點擊角色跳躍
hero.on('pointerdown', () => {
  let jumpHeight = 0;
  const jumpSpeed = 5;
  
  const ticker = () => {
    jumpHeight += jumpSpeed;
    hero.y -= Math.sin(jumpHeight * 0.1) * 2;
    
    if (jumpHeight >= 31.4) { // 完成一次跳躍
      app.ticker.remove(ticker);
    }
  };
  
  app.ticker.add(ticker);
});

// 金幣容器
const coins = new PIXI.Container();
app.stage.addChild(coins);

// 生成金幣
for (let i = 0; i < 10; i++) {
  const coin = new PIXI.Sprite(PIXI.Assets.get('coin'));
  coin.anchor.set(0.5);
  coin.x = Math.random() * app.screen.width;
  coin.y = Math.random() * app.screen.height;
  coin.scale.set(0.5);
  coins.addChild(coin);
  
  // 旋轉動畫
  app.ticker.add(() => {
    coin.rotation += 0.05;
  });
}

// 分數文字
const scoreText = new PIXI.Text({
  text: 'Score: 0',
  style: {
    fontFamily: 'Arial',
    fontSize: 32,
    fill: 0xffffff,
  }
});
scoreText.x = 10;
scoreText.y = 10;
app.stage.addChild(scoreText);

// 碰撞檢測
let score = 0;
app.ticker.add(() => {
  coins.children.forEach((coin, index) => {
    const dx = hero.x - coin.x;
    const dy = hero.y - coin.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    
    if (distance < 50) {
      coins.removeChildAt(index);
      score++;
      scoreText.text = `Score: ${score}`;
    }
  });
});
```

### 🎯 實作練習 3.5

1. 在專案中創建這個場景
2. 添加鍵盤控制 (WASD 移動角色)
3. 添加音效 (收集金幣時)
4. 添加粒子效果 (收集金幣時)

---

## 3.13 本章小結

### PixiJS 核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| Application | 應用入口 | `new PIXI.Application()` |
| Stage | 根容器 | `app.stage` |
| Container | 組織元素 | `new PIXI.Container()` |
| Sprite | 顯示圖片 | `new PIXI.Sprite(texture)` |
| Texture | 圖片數據 | `PIXI.Assets.load()` |
| Graphics | 繪製形狀 | `new PIXI.Graphics()` |
| Text | 顯示文字 | `new PIXI.Text()` |

### 你已經學會:
- ✅ PixiJS 的核心架構
- ✅ 座標系統和定位
- ✅ 資源載入和紋理管理
- ✅ 渲染循環和動畫
- ✅ 互動事件處理
- ✅ 性能優化技巧

### 🎯 作業

1. **分析 Symbol 組件**: 打開 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte)
   - 找出所有 PixiJS 相關的屬性
   - 理解 anchor 的設置原因
   - 思考如何優化渲染

2. **創建粒子系統**: 
   - 使用 `ParticleContainer` 創建 1000 個粒子
   - 讓它們隨機移動和旋轉
   - 超出邊界時回收到物件池

3. **探索 Board 組件**: 閱讀 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte)
   - 理解格子的佈局方式
   - 計算每個 Symbol 的座標
   - 思考如何處理不同螢幕尺寸

### 下一章預告

**第四章: pixi-svelte 整合應用**
- pixi-svelte 的設計理念
- 聲明式 vs 命令式
- pixi-svelte 組件系統
- 在 Svelte 中使用 PixiJS

---

## 📚 延伸閱讀

- [PixiJS 官方教學](https://pixijs.io/guides/basics/getting-started.html)
- [PixiJS API 文檔](https://pixijs.download/release/docs/index.html)
- [PixiJS Examples](https://pixijs.io/examples/)

---

[⬅️ 上一章: Svelte 5 基礎概念](./02-svelte5-basics.md) | [返回目錄](./README.md) | [下一章: pixi-svelte 整合 ➡️](./04-pixi-svelte-integration.md)
