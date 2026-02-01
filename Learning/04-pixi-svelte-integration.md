# 第四章：pixi-svelte 整合應用

## 學習目標

完成本章後,你將能夠:
- ✅ 理解 pixi-svelte 的設計理念
- ✅ 掌握聲明式 vs 命令式的差異
- ✅ 使用 pixi-svelte 組件系統
- ✅ 在 Svelte 中優雅地操作 PixiJS

---

## 4.1 為什麼需要 pixi-svelte?

### 4.1.1 傳統 PixiJS 寫法

```typescript
// 命令式：手動管理所有狀態
const sprite = new PIXI.Sprite(texture);
sprite.x = 100;
sprite.y = 100;
sprite.anchor.set(0.5);
container.addChild(sprite);

// 更新位置
sprite.x = 200;

// 移除
container.removeChild(sprite);
sprite.destroy();
```

**問題:**
- 🔴 需要手動管理生命週期
- 🔴 狀態更新需要手動同步
- 🔴 容易產生記憶體洩漏
- 🔴 代碼冗長且重複

### 4.1.2 pixi-svelte 寫法

```svelte
<script lang="ts">
  let x = $state(100);
  let y = $state(100);
</script>

<Container>
  <Sprite {texture} {x} {y} anchor={{ x: 0.5, y: 0.5 }} />
</Container>
```

**優勢:**
- ✅ 聲明式：只描述「是什麼」
- ✅ 自動管理生命週期
- ✅ 響應式：狀態改變自動更新
- ✅ 代碼簡潔清晰

---

## 4.2 聲明式 vs 命令式

### 4.2.1 概念對比

**命令式 (Imperative):**
```typescript
// 告訴電腦「怎麼做」
const sprite = new PIXI.Sprite(texture);
sprite.x = 100;
if (condition) {
  sprite.alpha = 0.5;
} else {
  sprite.alpha = 1;
}
container.addChild(sprite);
```

**聲明式 (Declarative):**
```svelte
<!-- 告訴電腦「是什麼」-->
<Sprite 
  {texture} 
  x={100} 
  alpha={condition ? 0.5 : 1} 
/>
```

### 4.2.2 實際範例

創建一個會閃爍的精靈:

**命令式寫法:**
```typescript
const sprite = new PIXI.Sprite(texture);
let visible = true;

setInterval(() => {
  visible = !visible;
  sprite.visible = visible; // 手動同步
}, 1000);
```

**聲明式寫法:**
```svelte
<script lang="ts">
  let visible = $state(true);
  
  setInterval(() => {
    visible = !visible; // 自動同步到 UI
  }, 1000);
</script>

<Sprite {texture} {visible} />
```

---

## 4.3 pixi-svelte 核心組件

### 4.3.1 App 組件

`<App>` 是 pixi-svelte 的根組件,對應 `PIXI.Application`:

```svelte
<script lang="ts">
  import { App } from 'pixi-svelte';
</script>

<App
  width={800}
  height={600}
  backgroundColor={0x1099bb}
  resizeTo={window}
>
  <!-- 所有其他組件放這裡 -->
</App>
```

### 專案實例

查看 [apps/lines/src/components/Game.svelte](../apps/lines/src/components/Game.svelte):

```svelte
<script lang="ts">
  import { App } from 'pixi-svelte';
  import { innerWidth, innerHeight } from 'svelte/reactivity/window';
  
  $effect(() => {
    context.stateApp.pixiApplication = untrack(() => app);
  });
</script>

<App
  bind:this={app}
  width={innerWidth()}
  height={innerHeight()}
  resizeTo={window}
  antialias={true}
  autoDensity={true}
  backgroundColor={0x000000}
>
  <Board />
  <UI />
</App>
```

### 4.3.2 Container 組件

`<Container>` 對應 `PIXI.Container`,用於組織元素:

```svelte
<Container
  x={100}
  y={100}
  rotation={Math.PI / 4}
  alpha={0.8}
  scale={{ x: 1.5, y: 1.5 }}
>
  <!-- 子元素 -->
  <Sprite ... />
  <Text ... />
</Container>
```

### 專案實例：Symbol 組件

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<Container
  {x}
  {y}
  {alpha}
  visible={show}
  oncreate={(container) => {
    // 創建時的回調
    containerHolder = container;
  }}
>
  <Sprite ... />
  {#if state === 'bright'}
    <Sprite texture={glowTexture} />
  {/if}
</Container>
```

### 4.3.3 Sprite 組件

`<Sprite>` 對應 `PIXI.Sprite`,用於顯示圖片:

```svelte
<Sprite
  texture={myTexture}
  x={100}
  y={100}
  anchor={{ x: 0.5, y: 0.5 }}
  width={200}
  height={200}
  rotation={0}
  alpha={1}
  tint={0xffffff}
/>
```

### 專案實例：符號顯示

```svelte
<script lang="ts">
  const symbolTexture = context.stateApp.loadedAssets[`symbol-${symbol.name}`];
</script>

<Sprite
  texture={symbolTexture}
  anchor={{ x: 0.5, y: 0.5 }}
  x={x}
  y={y}
  width={SYMBOL_SIZE}
  height={SYMBOL_SIZE}
/>
```

### 4.3.4 Text 組件

`<Text>` 對應 `PIXI.Text`,用於顯示文字:

```svelte
<Text
  text="Hello PixiJS!"
  x={400}
  y={300}
  anchor={{ x: 0.5, y: 0.5 }}
  style={{
    fontFamily: 'Arial',
    fontSize: 24,
    fill: 0xffffff,
    align: 'center',
  }}
/>
```

### 專案實例：UI 文字

查看 [packages/components-ui-pixi/src/UiGameName.svelte](../packages/components-ui-pixi/src/UiGameName.svelte):

```svelte
<Text
  text={name}
  anchor={{ x: 0.5, y: 0 }}
  style={{
    fontFamily: 'proxima-nova',
    fontSize: REM * 1.5,
    fontWeight: '600',
    fill: 0xffffff,
  }}
/>
```

### 4.3.5 Graphics 組件

`<Graphics>` 用於繪製矢量圖形:

```svelte
<script lang="ts">
  import { Graphics } from 'pixi-svelte';
  
  function draw(graphics: PIXI.Graphics) {
    graphics.clear();
    graphics.rect(0, 0, 100, 100);
    graphics.fill(0xff0000);
  }
</script>

<Graphics oncreate={draw} />
```

### 🎯 實作練習 4.1

創建一個動態矩形:

```svelte
<script lang="ts">
  import { Graphics } from 'pixi-svelte';
  let size = $state(100);
  let color = $state(0xff0000);
  
  function draw(graphics: PIXI.Graphics) {
    graphics.clear();
    graphics.rect(0, 0, size, size);
    graphics.fill(color);
  }
  
  $effect(() => {
    // 當 size 或 color 改變時重新繪製
    draw(graphicsRef);
  });
</script>

<Graphics oncreate={(g) => (graphicsRef = g)} />

<button onclick={() => size += 10}>放大</button>
<button onclick={() => color = Math.random() * 0xffffff}>換色</button>
```

---

## 4.4 生命週期回調

### 4.4.1 oncreate

在 PixiJS 對象創建時調用:

```svelte
<Sprite
  {texture}
  oncreate={(sprite) => {
    console.log('Sprite 已創建:', sprite);
    // 可以保存引用或做初始化
    spriteRef = sprite;
  }}
/>
```

### 4.4.2 ondestroy

在 PixiJS 對象銷毀前調用:

```svelte
<Sprite
  {texture}
  ondestroy={(sprite) => {
    console.log('Sprite 即將銷毀:', sprite);
    // 清理工作
  }}
/>
```

### 專案實例：保存引用

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts">
  let containerHolder: PIXI.Container;
  let spriteHolder: PIXI.Sprite;
</script>

<Container
  oncreate={(container) => {
    containerHolder = container;
  }}
>
  <Sprite
    oncreate={(sprite) => {
      spriteHolder = sprite;
    }}
  />
</Container>
```

**為什麼需要保存引用?**
- 用於直接操作 PixiJS 對象
- 執行複雜動畫
- 調用 PixiJS 特定方法

---

## 4.5 響應式屬性

### 4.5.1 自動更新

所有屬性都是響應式的:

```svelte
<script lang="ts">
  let x = $state(0);
  let alpha = $state(1);
  
  // x 或 alpha 改變時,Sprite 自動更新
  setInterval(() => {
    x += 1;
    if (x > 800) x = 0;
  }, 16);
</script>

<Sprite {texture} {x} {alpha} />
```

### 4.5.2 衍生屬性

使用 `$derived` 創建計算屬性:

```svelte
<script lang="ts">
  let progress = $state(0); // 0 到 1
  
  let x = $derived(progress * 800);
  let alpha = $derived(1 - progress);
  let scale = $derived(0.5 + progress * 0.5);
</script>

<Sprite 
  {texture} 
  {x} 
  {alpha} 
  scale={{ x: scale, y: scale }} 
/>
```

### 專案實例：Symbol 狀態

```svelte
<script lang="ts">
  let state = $state<'default' | 'dim' | 'bright'>('default');
  
  let alpha = $derived(
    state === 'dim' ? 0.5 :
    state === 'bright' ? 1 :
    0.8
  );
  
  let scale = $derived(
    state === 'bright' ? 1.2 : 1
  );
</script>

<Sprite 
  {texture} 
  {alpha} 
  scale={{ x: scale, y: scale }} 
/>
```

---

## 4.6 條件渲染

### 4.6.1 使用 {#if}

```svelte
<script lang="ts">
  let showSprite = $state(true);
</script>

<Container>
  {#if showSprite}
    <Sprite {texture} />
  {/if}
</Container>
```

### 4.6.2 使用 visible 屬性

```svelte
<script lang="ts">
  let isVisible = $state(true);
</script>

<Sprite {texture} visible={isVisible} />
```

**差異:**
- `{#if}`: 完全移除/創建對象 (適合不常切換)
- `visible`: 只改變可見性 (適合頻繁切換)

### 專案實例：Symbol 顯示

查看 [apps/lines/src/components/Symbol.svelte](../apps/lines/src/components/Symbol.svelte):

```svelte
<script lang="ts">
  let show = $state(true);
  
  context.eventEmitter.subscribeOnMount({
    symbolShow: (event) => {
      if (matchIndex(event.index)) show = true;
    },
    symbolHide: (event) => {
      if (matchIndex(event.index)) show = false;
    },
  });
</script>

<Container visible={show}>
  <Sprite ... />
</Container>
```

---

## 4.7 列表渲染

### 4.7.1 使用 {#each}

```svelte
<script lang="ts">
  let items = $state([
    { id: 1, x: 100, y: 100 },
    { id: 2, x: 200, y: 200 },
    { id: 3, x: 300, y: 300 },
  ]);
</script>

<Container>
  {#each items as item (item.id)}
    <Sprite {texture} x={item.x} y={item.y} />
  {/each}
</Container>
```

**注意 key:**
- 使用 `(item.id)` 作為 key
- 幫助 Svelte 追蹤元素
- 避免不必要的重新創建

### 專案實例：Board 組件

查看 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte):

```svelte
<script lang="ts">
  let board = $state<RawSymbol[][]>([]);
</script>

<Container>
  {#each board as reel, reelIndex (reelIndex)}
    {#each reel as symbol, symbolIndex (symbolIndex)}
      <Symbol
        x={reelIndex * SYMBOL_SIZE}
        y={symbolIndex * SYMBOL_SIZE}
        index={[reelIndex, symbolIndex]}
        rawSymbol={symbol}
      />
    {/each}
  {/each}
</Container>
```

### 🎯 實作練習 4.2

創建一個動態粒子系統:

```svelte
<script lang="ts">
  interface Particle {
    id: number;
    x: number;
    y: number;
    vx: number;
    vy: number;
  }
  
  let particles = $state<Particle[]>([]);
  let nextId = 0;
  
  function addParticle() {
    particles.push({
      id: nextId++,
      x: 400,
      y: 300,
      vx: (Math.random() - 0.5) * 10,
      vy: (Math.random() - 0.5) * 10,
    });
  }
  
  // 更新粒子位置
  setInterval(() => {
    particles = particles.map(p => ({
      ...p,
      x: p.x + p.vx,
      y: p.y + p.vy,
    })).filter(p => 
      p.x > 0 && p.x < 800 && 
      p.y > 0 && p.y < 600
    );
  }, 16);
</script>

<Container>
  {#each particles as particle (particle.id)}
    <Sprite
      {texture}
      x={particle.x}
      y={particle.y}
      anchor={{ x: 0.5, y: 0.5 }}
    />
  {/each}
</Container>

<button onclick={addParticle}>添加粒子</button>
```

---

## 4.8 事件處理

### 4.8.1 基本事件

```svelte
<Sprite
  {texture}
  eventMode="static"
  cursor="pointer"
  onpointerdown={(event) => {
    console.log('點擊位置:', event.global.x, event.global.y);
  }}
  onpointerover={() => {
    console.log('滑鼠懸停');
  }}
  onpointerout={() => {
    console.log('滑鼠離開');
  }}
/>
```

### 4.8.2 自訂事件處理

```svelte
<script lang="ts">
  let isHovered = $state(false);
  let isPressed = $state(false);
  
  let tint = $derived(
    isPressed ? 0xff0000 :
    isHovered ? 0x00ff00 :
    0xffffff
  );
</script>

<Sprite
  {texture}
  {tint}
  eventMode="static"
  cursor="pointer"
  onpointerover={() => isHovered = true}
  onpointerout={() => isHovered = false}
  onpointerdown={() => isPressed = true}
  onpointerup={() => isPressed = false}
/>
```

### 專案實例：按鈕互動

查看 [packages/components-ui-pixi/src/SimpleUiButton.svelte](../packages/components-ui-pixi/src/SimpleUiButton.svelte):

```svelte
<script lang="ts">
  let { disabled = false, onclick }: Props = $props();
  
  let isPressed = $state(false);
  let scale = $derived(isPressed ? 0.95 : 1);
</script>

<Container
  eventMode={disabled ? 'none' : 'static'}
  cursor={disabled ? 'default' : 'pointer'}
  scale={{ x: scale, y: scale }}
  onpointerdown={() => !disabled && (isPressed = true)}
  onpointerup={() => {
    if (!disabled && isPressed) {
      onclick?.();
    }
    isPressed = false;
  }}
>
  <Sprite ... />
  <Text ... />
</Container>
```

---

## 4.9 高級組件

### 4.9.1 Spine 動畫

查看 [packages/pixi-svelte/src/lib/components/Spine.svelte](../packages/pixi-svelte/src/lib/components/Spine.svelte):

```svelte
<script lang="ts">
  import { Spine, SpineTrack } from 'pixi-svelte';
</script>

<Spine
  resource="button-spin"
  x={400}
  y={300}
  oncreate={(spine) => {
    spineRef = spine;
  }}
>
  <SpineTrack
    trackIndex={0}
    animationName="idle"
    loop={true}
  />
</Spine>
```

### 專案實例：按鈕動畫

```svelte
<script lang="ts">
  let animationName = $state('idle');
</script>

<Spine resource="button-spin">
  <SpineTrack
    trackIndex={0}
    {animationName}
    loop={animationName === 'idle'}
    oncomplete={() => {
      if (animationName === 'press') {
        animationName = 'idle';
      }
    }}
  />
</Spine>
```

### 4.9.2 Tilemap (瓦片地圖)

用於創建大型地圖:

```svelte
<script lang="ts">
  import { TilemapProvider, Tilemap, TilemapLayer } from 'pixi-svelte';
</script>

<TilemapProvider>
  <Tilemap>
    <TilemapLayer
      data={mapData}
      tileSize={32}
    />
  </Tilemap>
</TilemapProvider>
```

### 4.9.3 ParticleContainer

用於大量精靈:

```svelte
<ParticleContainer
  maxSize={10000}
  properties={{
    scale: true,
    position: true,
    rotation: true,
    alpha: true,
  }}
>
  {#each particles as particle (particle.id)}
    <Sprite ... />
  {/each}
</ParticleContainer>
```

---

## 4.10 Context 整合

### 4.10.1 使用 Context

pixi-svelte 與 Svelte Context 完美整合:

```svelte
<!-- Parent.svelte -->
<script lang="ts">
  import { setContext } from '../game/context';
  import { App } from 'pixi-svelte';
  
  setContext(); // 設置 Context
</script>

<App>
  <Child />
</App>

<!-- Child.svelte -->
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext();
  const texture = context.stateApp.loadedAssets['myTexture'];
</script>

<Sprite {texture} />
```

### 專案實例：Game 組件

查看 [apps/lines/src/components/Game.svelte](../apps/lines/src/components/Game.svelte):

```svelte
<script lang="ts">
  import { getContext } from '../game/context';
  
  const context = getContext();
  
  // 將 app 實例存入 context
  $effect(() => {
    context.stateApp.pixiApplication = untrack(() => app);
  });
</script>

<App bind:this={app}>
  <!-- 所有子組件都能通過 context 訪問 app -->
  <Board />
  <UI />
</App>
```

---

## 4.11 性能優化

### 4.11.1 避免不必要的重新渲染

```svelte
<script lang="ts">
  // ❌ 差勁：每次都創建新對象
  let anchor = $derived({ x: 0.5, y: 0.5 });
  
  // ✅ 良好：重用對象
  const ANCHOR_CENTER = { x: 0.5, y: 0.5 };
  let anchor = ANCHOR_CENTER;
</script>

<Sprite {texture} {anchor} />
```

### 4.11.2 使用 untrack

```svelte
<script lang="ts">
  import { untrack } from 'svelte';
  
  let count = $state(0);
  let expensive = $state(0);
  
  $effect(() => {
    // count 改變時執行,但不追蹤 expensive
    console.log(count);
    const value = untrack(() => expensive);
  });
</script>
```

### 4.11.3 條件載入資源

```svelte
<script lang="ts">
  let showHeavyComponent = $state(false);
</script>

{#if showHeavyComponent}
  <HeavySpineAnimation />
{/if}
```

---

## 4.12 實戰：創建一個轉軸遊戲

### 📝 任務說明

使用 pixi-svelte 創建一個簡單的老虎機轉軸:

```svelte
<!-- Reel.svelte -->
<script lang="ts">
  import { Container, Sprite } from 'pixi-svelte';
  
  interface Props {
    x: number;
    symbols: string[];
    textures: Record<string, PIXI.Texture>;
  }
  
  let { x, symbols, textures }: Props = $props();
  
  let offset = $state(0);
  let isSpinning = $state(false);
  
  const SYMBOL_HEIGHT = 150;
  
  function spin() {
    if (isSpinning) return;
    
    isSpinning = true;
    let speed = 20;
    
    const ticker = () => {
      offset += speed;
      
      if (offset >= SYMBOL_HEIGHT * symbols.length) {
        offset = 0;
      }
      
      speed *= 0.98; // 減速
      
      if (speed < 0.5) {
        isSpinning = false;
        offset = Math.round(offset / SYMBOL_HEIGHT) * SYMBOL_HEIGHT;
        // 移除 ticker
      }
    };
    
    // 添加 ticker (實際應使用 PIXI ticker)
  }
  
  defineExpose({ spin });
</script>

<Container {x}>
  {#each symbols as symbol, i (i)}
    <Sprite
      texture={textures[symbol]}
      x={0}
      y={i * SYMBOL_HEIGHT - offset}
      anchor={{ x: 0.5, y: 0.5 }}
      width={120}
      height={120}
    />
  {/each}
</Container>

<!-- SlotMachine.svelte -->
<script lang="ts">
  import { App, Container } from 'pixi-svelte';
  import Reel from './Reel.svelte';
  
  const symbols = ['cherry', 'lemon', 'orange', 'plum', 'bell', 'bar', 'seven'];
  
  let reels = $state([
    ['cherry', 'lemon', 'orange'],
    ['lemon', 'orange', 'plum'],
    ['orange', 'plum', 'bell'],
  ]);
  
  let reelRefs: any[] = [];
  
  async function spinAll() {
    for (let i = 0; i < reels.length; i++) {
      reelRefs[i]?.spin();
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }
</script>

<App width={800} height={600}>
  <Container x={400} y={300}>
    {#each reels as reelSymbols, i (i)}
      <Reel
        bind:this={reelRefs[i]}
        x={i * 150 - 150}
        symbols={reelSymbols}
        {textures}
      />
    {/each}
  </Container>
  
  <button onclick={spinAll}>SPIN</button>
</App>
```

### 🎯 實作練習 4.3

1. 完成上述轉軸組件
2. 添加音效
3. 添加勝利檢測
4. 添加分數系統

---

## 4.13 本章小結

### pixi-svelte 核心概念

| 概念 | 用途 | 範例 |
|------|------|------|
| 聲明式渲染 | 描述 UI 狀態 | `<Sprite x={100} />` |
| 響應式屬性 | 自動更新 | `let x = $state(0)` |
| 生命週期 | 管理創建/銷毀 | `oncreate={(s) => {}}` |
| 條件渲染 | 動態顯示 | `{#if show}<Sprite />{/if}` |
| 列表渲染 | 批量創建 | `{#each items as item}` |
| 事件處理 | 用戶互動 | `onpointerdown={...}` |

### 你已經學會:
- ✅ pixi-svelte 的設計理念
- ✅ 聲明式 vs 命令式開發
- ✅ 所有核心組件的使用
- ✅ 響應式屬性和衍生狀態
- ✅ 條件和列表渲染
- ✅ 事件處理和互動
- ✅ Context 整合

### 🎯 作業

1. **分析 Board 組件**: 打開 [apps/lines/src/components/Board.svelte](../apps/lines/src/components/Board.svelte)
   - 理解轉軸是如何佈局的
   - 找出所有響應式變數
   - 思考為什麼要用二維陣列

2. **創建自訂組件**: 創建一個 `AnimatedSprite` 組件
   - 接收 `textures: PIXI.Texture[]` prop
   - 使用 ticker 循環播放紋理
   - 支援 `fps` 和 `loop` props

3. **探索 UI 組件**: 閱讀 [packages/components-ui-pixi/src/UI.svelte](../packages/components-ui-pixi/src/UI.svelte)
   - 理解佈局系統
   - 找出如何使用 Context
   - 思考如何優化性能

### 下一章預告

**第五章: 事件驅動架構 (Event Emitter)**
- 事件驅動的設計模式
- EventEmitter 實作原理
- bookEvent 和 emitterEvent 的關係
- 組件間通信最佳實踐

---

## 📚 延伸閱讀

- [pixi-svelte GitHub](https://github.com/qk0106/pixi-svelte-storybook)
- [pixi-svelte NPM](https://www.npmjs.com/package/pixi-svelte)
- [Svelte 聲明式 UI](https://svelte.dev/docs/svelte/overview)

---

[⬅️ 上一章: PixiJS 基礎](./03-pixijs-basics.md) | [返回目錄](./README.md) | [下一章: 事件驅動架構 ➡️](./05-event-driven-architecture.md)
