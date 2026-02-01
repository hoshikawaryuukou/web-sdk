# 第十章：進階主題與最佳實踐

## 學習目標

完成本章後,你將能夠:
- ✅ 實作國際化 (i18n)
- ✅ 管理音效系統
- ✅ 打包和部署遊戲
- ✅ 運用最佳實踐

---

## 10.1 國際化 (i18n)

### 10.1.1 為什麼需要 i18n?

遊戲需要支援多語言:
- 🌍 觸及更廣泛的用戶
- 💰 增加收益潛力
- 🎯 滿足不同市場需求

### 10.1.2 Lingui 基礎

專案使用 **Lingui** 處理國際化:

```typescript
import { i18n } from '@lingui/core';
import { msg } from '@lingui/macro';

// 定義訊息
const greeting = msg`Hello, World!`;

// 使用訊息
const text = i18n._(greeting);
```

### 10.1.3 設定語言

查看 [packages/components-shared/src/components/Authenticate.svelte](../packages/components-shared/src/components/Authenticate.svelte):

```typescript
import { i18n } from '@lingui/core';
import { messages as enMessages } from '../i18n/en/messages';
import { messages as zhMessages } from '../i18n/zh/messages';

// 載入訊息
i18n.load({
  en: enMessages,
  zh: zhMessages,
});

// 設定語言
i18n.activate('en');
```

### 10.1.4 在組件中使用

```svelte
<script lang="ts">
  import { i18n } from '@lingui/core';
  import { msg } from '@lingui/macro';
  
  let greeting = $derived(i18n._(msg`Welcome`));
  let betLabel = $derived(i18n._(msg`Bet`));
</script>

<Text text={greeting} />
<Button label={betLabel} />
```

### 10.1.5 帶變數的訊息

```typescript
import { t } from '@lingui/macro';

const winMessage = t`You won ${amount} coins!`;

// 或使用 plural
const spinsLeft = plural(count, {
  one: '# spin left',
  other: '# spins left',
});
```

### 10.1.6 數字和貨幣格式化

查看 [packages/utils-shared/amount.ts](../packages/utils-shared/amount.ts):

```typescript
export const numberToCurrencyString = (
  amount: number,
  currency: string,
  locale: string
): string => {
  // 特殊貨幣處理
  if (NO_LOCALISATION_CURRENCY_MAP[currency]) {
    return formatSpecialCurrency(amount, currency);
  }
  
  // 使用 Intl.NumberFormat
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency: currency,
  }).format(amount);
};
```

### 🎯 實作練習 10.1

添加多語言支援:

```typescript
// i18n/en/messages.ts
export const messages = {
  'game.bet': 'Bet',
  'game.spin': 'Spin',
  'game.autoplay': 'Auto Play',
  'game.win': 'Win',
  'game.balance': 'Balance',
};

// i18n/zh/messages.ts
export const messages = {
  'game.bet': '下注',
  'game.spin': '旋轉',
  'game.autoplay': '自動遊戲',
  'game.win': '獲勝',
  'game.balance': '餘額',
};
```

---

## 10.2 音效管理

### 10.2.1 Howler.js 基礎

專案使用 **Howler.js** 管理音效:

```typescript
import { Howl } from 'howler';

// 創建音效
const spinSound = new Howl({
  src: ['/sounds/spin.mp3'],
  volume: 0.5,
});

// 播放
spinSound.play();

// 停止
spinSound.stop();

// 淡入淡出
spinSound.fade(0, 1, 1000); // 從 0 淡入到 1,1秒
```

### 10.2.2 音效系統架構

查看 [packages/utils-sound/](../packages/utils-sound/):

```typescript
class SoundManager {
  private sounds: Map<string, Howl> = new Map();
  private masterVolume: number = 1;
  private musicVolume: number = 1;
  private sfxVolume: number = 1;
  
  // 載入音效
  load(id: string, src: string, options?: HowlOptions) {
    const sound = new Howl({ src: [src], ...options });
    this.sounds.set(id, sound);
  }
  
  // 播放音效
  play(id: string) {
    const sound = this.sounds.get(id);
    if (sound) {
      sound.volume(this.sfxVolume * this.masterVolume);
      sound.play();
    }
  }
  
  // 播放音樂
  playMusic(id: string) {
    const sound = this.sounds.get(id);
    if (sound) {
      sound.volume(this.musicVolume * this.masterVolume);
      sound.loop(true);
      sound.play();
    }
  }
  
  // 設定音量
  setMasterVolume(volume: number) {
    this.masterVolume = volume;
    this.updateAllVolumes();
  }
  
  private updateAllVolumes() {
    this.sounds.forEach((sound, id) => {
      const isMuic = id.startsWith('music_');
      const volume = isMusic 
        ? this.musicVolume * this.masterVolume
        : this.sfxVolume * this.masterVolume;
      sound.volume(volume);
    });
  }
}

export const soundManager = new SoundManager();
```

### 10.2.3 在遊戲中使用

```typescript
// 載入音效
soundManager.load('spin', '/sounds/spin.mp3');
soundManager.load('win', '/sounds/win.mp3');
soundManager.load('music_main', '/sounds/main-theme.mp3');

// 在 bookEventHandler 中播放
export const bookEventHandlerMap = {
  reveal: async (bookEvent) => {
    soundManager.play('spin');
    await eventEmitter.broadcastAsync({ type: 'reelSpin' });
  },
  
  win: async (bookEvent) => {
    soundManager.play('win');
    await eventEmitter.broadcastAsync({ type: 'winAnimation' });
  },
};

// 播放背景音樂
soundManager.playMusic('music_main');
```

### 10.2.4 音效設定 UI

```svelte
<script lang="ts">
  import { soundManager } from 'utils-sound';
  
  let masterVolume = $state(1);
  let musicVolume = $state(1);
  let sfxVolume = $state(1);
  
  $effect(() => {
    soundManager.setMasterVolume(masterVolume);
  });
</script>

<div class="sound-settings">
  <label>
    主音量: {Math.round(masterVolume * 100)}%
    <input
      type="range"
      min="0"
      max="1"
      step="0.1"
      bind:value={masterVolume}
    />
  </label>
  
  <label>
    音樂: {Math.round(musicVolume * 100)}%
    <input
      type="range"
      min="0"
      max="1"
      step="0.1"
      bind:value={musicVolume}
    />
  </label>
  
  <label>
    音效: {Math.round(sfxVolume * 100)}%
    <input
      type="range"
      min="0"
      max="1"
      step="0.1"
      bind:value={sfxVolume}
    />
  </label>
</div>
```

---

## 10.3 打包和部署

### 10.3.1 建構遊戲

```bash
# 建構單個遊戲
pnpm run build --filter=lines

# 建構所有遊戲
pnpm run build
```

### 10.3.2 輸出結構

```
apps/lines/.svelte-kit/output/
├── client/              # 客戶端檔案
│   ├── _app/           # 應用程式代碼
│   ├── assets/         # 資源檔案
│   └── favicon.svg
└── prerendered/
    └── pages/
        └── index.html  # 預渲染的 HTML
```

### 10.3.3 準備部署

```bash
# 創建部署資料夾
mkdir -p build/lines

# 複製 index.html
cp apps/lines/.svelte-kit/output/prerendered/pages/index.html build/lines/

# 複製所有客戶端檔案
cp -r apps/lines/.svelte-kit/output/client/* build/lines/

# 結構
build/lines/
├── index.html
├── _app/
├── assets/
└── favicon.svg
```

### 10.3.4 上傳到 Stake Engine

根據 [README.md](../README.md#launchAGame):

1. 登入 [Stake Engine](https://engine.stake.com/)
2. 進入遊戲的 Files 頁面
3. 選擇整個 `build/lines` 資料夾上傳
4. 點擊 "Publish Game" → "Front End"
5. 在 Developer 頁面測試

### 10.3.5 環境變數

```typescript
// 從 URL 參數獲取配置
const params = new URLSearchParams(window.location.search);

const config = {
  apiUrl: params.get('apiUrl'),
  gameId: params.get('gameId'),
  sessionToken: params.get('sessionToken'),
  currency: params.get('currency'),
  language: params.get('language'),
};
```

---

## 10.4 性能優化

### 10.4.1 資源優化

**圖片優化:**
```bash
# 使用 tinypng 壓縮圖片
# 使用 WebP 格式
# 使用紋理圖集 (Texture Atlas)
```

**音效優化:**
```bash
# 使用 MP3 (壓縮率好)
# 使用較低的位元率 (64-128 kbps)
# 只載入需要的音效
```

### 10.4.2 代碼分割

```typescript
// 動態導入
const BonusGame = lazy(() => import('./BonusGame.svelte'));

// 使用時才載入
{#if showBonus}
  <Suspense fallback={<Loading />}>
    <BonusGame />
  </Suspense>
{/if}
```

### 10.4.3 記憶體管理

```typescript
// 銷毀不需要的紋理
texture.destroy();

// 清理 PixiJS 對象
sprite.destroy({ children: true, texture: false });

// 取消事件訂閱
onMount(() => {
  const unsubscribe = eventEmitter.subscribe({ ... });
  return () => unsubscribe();
});
```

### 10.4.4 性能監控

```typescript
// FPS 監控
let fps = $state(60);
let frameCount = 0;
let lastTime = performance.now();

app.ticker.add(() => {
  frameCount++;
  const now = performance.now();
  
  if (now - lastTime >= 1000) {
    fps = frameCount;
    frameCount = 0;
    lastTime = now;
  }
});

// 記憶體監控
if (performance.memory) {
  console.log('Used:', performance.memory.usedJSHeapSize);
  console.log('Total:', performance.memory.totalJSHeapSize);
}
```

---

## 10.5 除錯技巧

### 10.5.1 Chrome DevTools

**Performance 面板:**
- 錄製遊戲運行
- 找出性能瓶頸
- 分析幀率下降

**Memory 面板:**
- 檢測記憶體洩漏
- 分析記憶體使用
- Heap Snapshot

### 10.5.2 PixiJS 除錯

```typescript
// 啟用除錯模式
PIXI.settings.RENDER_OPTIONS.hello = true;

// 顯示邊界框
sprite.getBounds();

// 檢查渲染樹
console.log(app.stage.children);
```

### 10.5.3 EventEmitter 除錯

```typescript
const originalBroadcast = eventEmitter.broadcast;

eventEmitter.broadcast = (event) => {
  console.log('📡 Broadcast:', event.type, event);
  return originalBroadcast(event);
};

const originalSubscribe = eventEmitter.subscribe;

eventEmitter.subscribe = (handlerMap) => {
  console.log('👂 Subscribe:', Object.keys(handlerMap));
  return originalSubscribe(handlerMap);
};
```

---

## 10.6 測試策略

### 10.6.1 單元測試

```typescript
// utils.test.ts
import { describe, it, expect } from 'vitest';
import { calculateWin } from './utils';

describe('calculateWin', () => {
  it('should calculate correct win amount', () => {
    const result = calculateWin({
      betAmount: 100,
      multiplier: 5,
      lines: 3,
    });
    
    expect(result).toBe(1500);
  });
});
```

### 10.6.2 組件測試 (Storybook)

已在第八章詳細介紹。

### 10.6.3 整合測試

```typescript
// game.test.ts
describe('Game Flow', () => {
  it('should complete a bet cycle', async () => {
    const game = new Game();
    
    // 開始下注
    await game.startBet();
    expect(game.state).toBe('betting');
    
    // 播放 book
    await game.playBook(testBook);
    expect(game.state).toBe('idle');
    
    // 檢查餘額
    expect(game.balance).toBe(expectedBalance);
  });
});
```

---

## 10.7 最佳實踐總結

### 10.7.1 代碼組織

```
✅ 良好的組織:
├── components/      # UI 組件
├── game/           # 遊戲邏輯
│   ├── context.ts
│   ├── eventEmitter.ts
│   ├── bookEventHandlerMap.ts
│   └── stateGame.ts
├── stories/        # Storybook
└── assets/         # 資源

❌ 不好的組織:
├── index.svelte
├── game.ts         # 所有邏輯在一個檔案
└── stuff/          # 混亂的資源
```

### 10.7.2 命名規範

```typescript
// ✅ 良好：清晰描述
const calculateWinAmount = () => { ... };
const isValidBet = () => { ... };
const SYMBOL_SIZE = 150;

// ❌ 不好：不清楚
const calc = () => { ... };
const check = () => { ... };
const size = 150;
```

### 10.7.3 註釋規範

```typescript
// ✅ 良好：解釋為什麼
// 使用 setTimeout 而非 requestAnimationFrame
// 因為需要在背景執行
setTimeout(() => { ... }, 1000);

// ❌ 不好：重複代碼
// 設定 x 為 100
const x = 100;
```

### 10.7.4 錯誤處理

```typescript
// ✅ 良好：完整處理
try {
  await playBook(book);
} catch (error) {
  console.error('Play book error:', error);
  
  // 通知用戶
  showErrorMessage('遊戲發生錯誤,請重新整理');
  
  // 恢復狀態
  resetGameState();
  
  // 上報錯誤
  reportError(error);
}

// ❌ 不好：忽略錯誤
await playBook(book).catch(() => {});
```

### 10.7.5 性能優先

```typescript
// ✅ 良好：避免不必要的計算
let cachedValue = null;

function expensiveCalculation() {
  if (cachedValue !== null) {
    return cachedValue;
  }
  
  cachedValue = doExpensiveWork();
  return cachedValue;
}

// ❌ 不好：每次都計算
function expensiveCalculation() {
  return doExpensiveWork();
}
```

---

## 10.8 常見問題

### 10.8.1 Storybook 載入慢

**問題:** Windows 上 Storybook 首次載入很慢

**解決:**
- 首次載入可能需要 10-15 分鐘
- 之後的熱重載很快
- 保持 Storybook 運行

### 10.8.2 記憶體洩漏

**問題:** 長時間運行後記憶體持續增長

**解決:**
```typescript
// 確保取消事件訂閱
onMount(() => {
  const unsubscribe = subscribe();
  return unsubscribe; // ← 重要!
});

// 銷毀 PixiJS 對象
sprite.destroy();

// 清理 Ticker
app.ticker.remove(tickerFn);
```

### 10.8.3 打包後無法運行

**問題:** 開發環境正常,打包後錯誤

**檢查:**
- 資源路徑是否正確
- 環境變數是否設定
- Console 錯誤訊息
- 網路請求是否成功

---

## 10.9 學習資源

### 10.9.1 官方文檔

- [Svelte 5](https://svelte.dev/docs/svelte/overview)
- [PixiJS](https://pixijs.download/release/docs/index.html)
- [XState](https://stately.ai/docs/xstate)
- [Storybook](https://storybook.js.org/docs)
- [Lingui](https://lingui.dev/)

### 10.9.2 社群資源

- [Svelte Discord](https://svelte.dev/chat)
- [PixiJS Forums](https://github.com/pixijs/pixijs/discussions)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/svelte)

### 10.9.3 進階主題

- WebGL 渲染原理
- 遊戲數學和機率
- 動畫插值算法
- 狀態機設計模式

---

## 10.10 專案未來發展

### 10.10.1 可能的改進

- **更多遊戲類型**
  - Megaways
  - Crash 遊戲
  - Plinko

- **更強大的工具**
  - 視覺化編輯器
  - 自動化測試
  - 性能分析工具

- **更好的開發體驗**
  - 熱模組替換 (HMR)
  - TypeScript 嚴格模式
  - 更好的錯誤訊息

### 10.10.2 貢獻指南

如果你想貢獻到這個專案:

1. Fork 專案
2. 創建功能分支
3. 提交 Pull Request
4. 遵循代碼規範
5. 添加測試

---

## 10.11 結語

### 你已經完成了:

- ✅ 10 章完整的學習內容
- ✅ 從基礎到進階的全面知識
- ✅ 實戰項目和練習
- ✅ 最佳實踐和技巧

### 下一步:

1. **實作自己的遊戲**
   - 選擇一個遊戲類型
   - 設計遊戲機制
   - 實作和測試

2. **深入研究**
   - WebGL 和著色器
   - 複雜動畫系統
   - 多人遊戲同步

3. **分享和交流**
   - 加入社群
   - 分享你的作品
   - 幫助其他開發者

### 💡 最後的建議

- 持續學習和實踐
- 多看優秀的開源項目
- 不要害怕犯錯
- 享受創造的過程

**祝你在遊戲開發的旅程中一切順利! 🎮🚀**

---

## 📚 完整學習路徑回顧

1. [環境設置與專案架構](./01-setup-and-architecture.md)
2. [Svelte 5 基礎概念](./02-svelte5-basics.md)
3. [PixiJS 基礎與 Canvas 渲染](./03-pixijs-basics.md)
4. [pixi-svelte 整合應用](./04-pixi-svelte-integration.md)
5. [事件驅動架構](./05-event-driven-architecture.md)
6. [狀態管理 (XState)](./06-state-management-xstate.md)
7. [佈局系統](./07-layout-system.md)
8. [Storybook 測試與開發](./08-storybook-development.md)
9. [遊戲開發流程](./09-game-development-workflow.md)
10. [進階主題與最佳實踐](./10-advanced-topics.md) ← 你在這裡

---

[⬅️ 上一章: 遊戲開發流程](./09-game-development-workflow.md) | [返回目錄](./README.md)