# 第一章：環境設置與專案架構

## 學習目標

完成本章後,你將能夠:
- ✅ 設置完整的開發環境
- ✅ 理解 TurboRepo Monorepo 架構
- ✅ 了解專案的目錄結構與組織方式
- ✅ 成功運行第一個範例遊戲

---

## 1.1 開發環境設置

### 必要工具安裝

根據 [README.md](../README.md#installation) 的說明:

1. **安裝 Node.js 22.16.0**
```bash
# 使用 nvm 安裝
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22.16.0
node -v  # 驗證版本
```

2. **安裝 pnpm 10.5.0**
```bash
npm install pnpm@10.5.0 -g
pnpm -v  # 驗證版本
```

3. **克隆專案並安裝依賴**
```bash
git clone https://github.com/StakeEngine/web-sdk.git
cd web-sdk
pnpm install
```

### 🎯 實作練習 1.1
執行以上指令,確認沒有錯誤訊息。

---

## 1.2 專案架構概覽

### Monorepo 架構

這個專案使用 **TurboRepo** 來管理 monorepo,主要分為兩大區域:

```
web-sdk/
├── apps/           # 應用程式 (遊戲範例)
│   ├── lines/      # 線型老虎機遊戲
│   ├── cluster/    # 集群型遊戲
│   ├── scatter/    # 散點型遊戲
│   └── ...
│
└── packages/       # 共用套件
    ├── pixi-svelte/           # PixiJS + Svelte 整合
    ├── components-ui-pixi/    # Pixi UI 組件
    ├── utils-event-emitter/   # 事件系統
    └── ...
```

### 📚 關鍵概念

**Monorepo 的優勢:**
- 程式碼重用: 多個遊戲共用相同的組件
- 統一版本管理: 所有套件版本一致
- 開發效率: 修改套件後立即生效

---

## 1.3 目錄結構詳解

### Apps 目錄
每個 `apps/` 下的子目錄都是一個獨立的遊戲:

```
apps/lines/
├── src/
│   ├── routes/          # SvelteKit 路由
│   │   └── +page.svelte # 遊戲入口
│   ├── components/      # 遊戲組件
│   │   ├── Game.svelte
│   │   ├── Board.svelte
│   │   └── Symbol.svelte
│   ├── game/            # 遊戲邏輯
│   │   ├── context.ts
│   │   ├── eventEmitter.ts
│   │   └── bookEventHandlerMap.ts
│   └── stories/         # Storybook 故事
└── package.json
```

### Packages 目錄
共用套件按功能分類:

| 套件類型 | 說明 | 範例 |
|---------|------|------|
| `config-*` | 配置檔案 | config-ts, config-vite |
| `utils-*` | 工具函數 | utils-shared, utils-slots |
| `components-*` | UI 組件 | components-ui-pixi |
| `pixi-*` | PixiJS 相關 | pixi-svelte |

---

## 1.4 運行第一個範例

### 啟動 Storybook

Storybook 是開發和測試組件的最佳工具:

```bash
pnpm run storybook --filter=lines
```

這個指令會:
1. 啟動 lines 遊戲的 Storybook
2. 自動打開瀏覽器 (通常是 http://localhost:6006)
3. 顯示所有可用的故事 (stories)

### 🎯 實作練習 1.2

1. 運行上述指令
2. 在左側邊欄找到 `MODE_BASE/book/random`
3. 點擊右上角的 `Action` 按鈕
4. 觀察遊戲的運行過程

**預期結果:**
- 轉軸開始旋轉
- 符號停止在隨機位置
- 如果中獎會顯示獲勝動畫

---

## 1.5 理解 package.json 的 filter

在 TurboRepo 中,`--filter` 參數用來指定要操作的套件:

```bash
# 格式
pnpm run <script> --filter=<package-name>

# 範例
pnpm run storybook --filter=lines
pnpm run dev --filter=cluster
pnpm run build --filter=scatter
```

**package-name 從哪裡來?**
查看 [apps/lines/package.json](../apps/lines/package.json):
```json
{
  "name": "lines",  // ← 這就是 package-name
  ...
}
```

### 🎯 實作練習 1.3

嘗試運行不同遊戲的 storybook:
```bash
pnpm run storybook --filter=cluster
pnpm run storybook --filter=scatter
```

---

## 1.6 開發模式 vs Storybook 模式

這個專案提供兩種開發方式:

### Storybook 模式 (推薦用於開發)
```bash
pnpm run storybook --filter=lines
```
- ✅ 可以測試獨立組件
- ✅ 可以使用模擬數據
- ✅ 不需要連接 RGS (遊戲伺服器)
- ✅ 熱重載速度快

### 開發模式 (用於整合測試)
```bash
pnpm run dev --filter=lines
```
- ⚠️ 需要完整的認證流程
- ⚠️ 需要 RGS 連線參數
- ✅ 完整的遊戲體驗

---

## 1.7 專案的核心依賴

理解這些核心依賴對學習很重要:

| 依賴 | 版本 | 用途 |
|------|------|------|
| [Svelte](https://svelte.dev) | 5.x | UI 框架 |
| [PixiJS](https://pixijs.com) | 8.x | Canvas 渲染引擎 |
| [pixi-svelte](https://www.npmjs.com/package/pixi-svelte) | - | Pixi + Svelte 整合 |
| [SvelteKit](https://kit.svelte.dev) | - | 應用框架 |
| [Storybook](https://storybook.js.org) | - | 組件開發工具 |
| [XState](https://stately.ai/docs) | - | 狀態機 |
| [TurboRepo](https://turbo.build/repo) | - | Monorepo 工具 |

### 📚 延伸閱讀

在進入下一章前,建議先瀏覽:
- [Svelte 5 官方教學](https://svelte.dev/tutorial/svelte/welcome-to-svelte)
- [PixiJS 基礎教學](https://pixijs.download/release/docs/index.html)

---

## 1.8 本章小結

### 你已經學會:
- ✅ 設置完整的開發環境
- ✅ 理解 Monorepo 架構的優勢
- ✅ 了解 apps 和 packages 的區別
- ✅ 運行並測試範例遊戲
- ✅ 使用 TurboRepo 的 filter 參數

### 🎯 作業

1. **探索 Storybook**: 在 lines 遊戲的 Storybook 中,找到並測試:
   - `COMPONENTS/Symbol/symbols` - 查看所有符號
   - `COMPONENTS/Board/component` - 查看遊戲面板
   - `MODE_BASE/bookEvent/reveal` - 觀察轉軸旋轉

2. **比較不同遊戲**: 分別運行 lines, cluster, scatter 三個遊戲的 storybook,觀察它們的差異

3. **閱讀原始碼**: 打開 [apps/lines/src/routes/+page.svelte](../apps/lines/src/routes/+page.svelte),理解遊戲的入口點

### 下一章預告

**第二章: Svelte 5 基礎概念**
- Svelte 5 的響應式系統 ($state, $derived)
- 組件的生命週期
- Props 和 Events
- Context API 的使用

---

## 💡 常見問題

**Q: Windows 上 Storybook 載入很慢怎麼辦?**
A: 這是已知問題。首次載入可能需要 10-15 分鐘,但之後的熱重載會很快。建議開著 Storybook 持續開發。

**Q: 為什麼要用 pnpm 而不是 npm?**
A: pnpm 在 monorepo 中效能更好,且能更好地處理依賴關係。

**Q: 可以用其他 IDE 嗎?**
A: 可以,但 VS Code 對 Svelte 的支援最好。建議安裝 "Svelte for VS Code" 擴充套件。

---

[⬅️ 返回目錄](./README.md) | [下一章: Svelte 5 基礎概念 ➡️](./02-svelte5-basics.md)
