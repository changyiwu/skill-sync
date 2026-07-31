# 技能副本同步（skill-sync）（專案藍圖）

> 本檔為跨 Agent 通用的專案藍圖（AGENTS.md 開放標準）。任何 Agent 的每個 session 都應先讀本檔＋`handoff.md`。
> Claude Code 不讀 `agents.md`，改由 `CLAUDE.md` 的 `@agents.md` import 本檔；Claude 專屬規範寫在 `CLAUDE.md`。

## 專案簡介

維護 `sync-skills` 技能——把「專案裡的技能原始檔」安全地覆蓋到本機四個 Agent 工具（Claude Code／Codex／OpenCode／Antigravity）的全域技能目錄。口令：「**同步技能**」。

從 `cross-device-agent-skills` 的 README 抽出來獨立成技能，因為這套流程對**任何**有技能原始檔的專案都適用，不該綁在那個 repo 裡。

## 這個技能解決什麼

三個看不出來的坑，README 裡的一段 `Copy-Item` 只擋得住第一個：

1. **Agent 用 Write/Edit「重建」副本** → 把 context 裡記得的舊內容寫成新版，事後長得跟正常同步一模一樣
2. **雲端硬碟餵出過期的檔案 bytes** → 磁碟是舊的、git HEAD 是新的，照樣同步等於把所有副本一起降版
3. **同步完沒驗** → 複製失敗、檔案被鎖、原始檔刪掉的檔案殘留在副本裡，都不會有人發現

對應到技能的三條鐵則與步驟 2／步驟 5。

## 關鍵時程

（無固定期限，依需要推進）

## 目標與路線圖

- [x] 階段一：`sync-skills/SKILL.md` 成形（含 git 前置檢查、逐檔 hash 驗證、殘留檔偵測）
- [x] 階段二：本專案自身初始化（agents.md ＋ CLAUDE.md ＋ handoff.md ＋ README.md）
- [x] 階段三：把 `sync-skills` 安裝到四家技能目錄（`<電腦A>` 已裝，另兩台待補）
- [x] 階段四：git init ＋ GitHub repo（L2，公開）
- [x] 階段五：建立 L3 Obsidian 專案筆記
- [ ] 階段六：跨電腦實測（另一台電腦跑「同步技能」）

## 資料夾結構

```
skill-sync/
├─ .gitignore
├─ README.md          # 技能說明與安裝方式
├─ agents.md          # 本檔：專案藍圖
├─ CLAUDE.md          # 橋接檔（@agents.md）
├─ handoff.md         # 交接檔（每次收工必更新）；已 gitignore，只走 L1
└─ sync-skills/
   └─ SKILL.md        # 「同步技能」技能
```

## 同步層級（本專案初始化至第 3 層級）

| 層級 | 平台 | 位置 | 讀取時機 |
|------|------|------|---------|
| L1 | 本地（GDrive） | `agents.md`＋`handoff.md`＋`CLAUDE.md`（橋接） | 每個 session |
| L2 | GitHub | https://github.com/changyiwu/skill-sync （公開） | 指定時 |
| L3 | Obsidian | `skill-sync/專案工作流程.md` | 有需要時 |

## 工作約定

- 任何 Agent、任何電腦：**開工先讀 `handoff.md`，收工必更新 `handoff.md`**
- 修改共用檔案前先讀最新內容，避免覆蓋其他 Agent 的變更
- 修改前先確認計畫，優先保留原有資料結構
- 所有回應與文件使用繁體中文
- **本資料夾是技能原始檔**。改動一律改這裡，改完在本專案說「同步技能」覆蓋四份安裝副本
- 編輯 `SKILL.md` 時不可存成含 BOM 的 UTF-8，否則 frontmatter 解析失敗、技能觸發不了
- **這是公開 repo**：進 git 的檔案（`README.md`／`agents.md`／`CLAUDE.md`／`SKILL.md`）一律不寫本機絕對路徑，電腦名一律用 `<電腦A>`／`<電腦B>`／`<電腦C>` 占位符
- `handoff.md` 已 gitignore，是唯一保留真實電腦名的檔案——因為「上次在哪台收工」是它的核心功能（`startup` 技能會拿它跟 `$env:COMPUTERNAME` 比對），寫占位符就失效。跨電腦靠 L1 雲端硬碟同步，不靠 git
- 這個技能的鐵則同樣約束改它的人：**副本一律 `Copy-Item` 從磁碟複製**，絕不可用 Write/Edit 重建

## 相關專案

- `cross-device-agent-skills`：三技能（初始化專案／開工／收工）。本技能原本是它 README 的一段，抽出後那邊改為指向這裡
- `chezmoi-setup`：`chezmoi-sync` 技能。管的是 dotfile 的 target↔source↔GitHub 三方一致，與本技能是**兩條不同路徑**——`Copy-Item` 繞過 chezmoi 直接寫 target，所以同步完 `chezmoi status` 必然報出這些技能，正解是 `chezmoi add` 不是 `apply`

## 最近進度

- 2026-07-31（`<電腦A>`）：專案建立。`sync-skills/SKILL.md` 從 `cross-device-agent-skills/README.md` 的 `Copy-Item` 段抽出並擴充：新增源檔可信度的 git 判讀表、逐檔遞迴 hash 驗證、副本殘留檔偵測、chezmoi 善後提醒。同日裝進本機四家技能目錄並實跑一次（含三技能共 16 項 hash 全 `OK`），`git init` ＋ 建立公開 repo `changyiwu/skill-sync` 並推送。
- 2026-07-31（`<電腦A>`）：重跑 `project-init`，L1 三個檔依最新範本重建（補上「關鍵時程」與範本版工作約定條目），並補建 L3 Obsidian 專案筆記，三個層級到齊。
