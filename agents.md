# 技能副本同步（skill-sync）（專案藍圖）

> 本檔為跨 Agent 通用的專案藍圖（AGENTS.md 開放標準）。任何 Agent 的每個 session 都應先讀本檔＋`handoff.md`。
> Claude Code 不讀 `agents.md`，改由 `CLAUDE.md` 的 `@agents.md` import 本檔；Claude 專屬規範寫在 `CLAUDE.md`。

## 專案簡介

維護 `sync-skills` 技能——把「專案裡的技能原始檔」安全地覆蓋到本機四個 Agent 工具（Claude Code／Codex／OpenCode／Antigravity）的全域技能目錄。口令：「**同步技能**」。

從 `cross-device-agent-skills` 的 README 抽出來獨立成技能，因為這套流程對**任何**有技能原始檔的專案都適用，不該綁在那個 repo 裡。

## 這個技能解決什麼

四個看不出來的坑，README 裡的一段 `Copy-Item` 只擋得住第一個：

1. **Agent 用 Write/Edit「重建」副本** → 把 context 裡記得的舊內容寫成新版，事後長得跟正常同步一模一樣
2. **雲端硬碟餵出過期的檔案 bytes** → 磁碟是舊的、git HEAD 是新的，照樣同步等於把所有副本一起降版
3. **同步完沒驗** → 複製失敗、檔案被鎖、原始檔刪掉的檔案殘留在副本裡，都不會有人發現
4. **安裝時會改名的技能被漏掉** → 資料夾叫 `skills/08-draw`、裝進去叫 `claude-draw`，只認資料夾名的同步連找都找不到它們，於是它們靜靜落後、沒有任何警告

對應到技能的四條鐵則與步驟 1／步驟 2／步驟 6。

## 關鍵時程

（無固定期限，依需要推進）

## 目標與路線圖

- [x] 階段一：`sync-skills/SKILL.md` 成形（含 git 前置檢查、逐檔 hash 驗證、殘留檔偵測）
- [x] 階段二：本專案自身初始化（agents.md ＋ CLAUDE.md ＋ handoff.md ＋ README.md）
- [x] 階段三：把 `sync-skills` 安裝到四家技能目錄（`<電腦A>` 已裝，另兩台待補）
- [x] 階段四：git init ＋ GitHub repo（L2，公開）
- [x] 階段五：建立 L3 Obsidian 專案筆記
- [ ] 階段六：跨電腦實測（另一台電腦跑「同步技能」）
- [x] 階段七：讓改名安裝的技能納入同步（安裝名改為讀 frontmatter `name:`，不再靠資料夾名）

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

## 設計決策：安裝名怎麼決定（階段七）

原本規劃是讓技能吃一張手寫的「來源資料夾 → 安裝名」對照表。實測後改掉了——**對照表的資料本來就寫在源檔裡**。

各家工具都要求「安裝資料夾名 ＝ `SKILL.md` frontmatter 的 `name:`」，所以安裝名可以直接從源檔推導，不需要另外維護一份會過期的設定檔（那正是這次要根治的病）。本機 42 項安裝副本實測 **42/42 成立**，包含所有改名安裝的例子：

| 來源 | `name:` | 安裝成 |
|------|---------|--------|
| `claude-code-lazy-packs/skills/08-draw` | `claude-draw` | `claude-draw` |
| `codex-lazy-packs/skills/02-essentials` | `codex-essentials` | `codex-essentials` |
| `opencode-lazy-packs/skills/07-install-all` | `opencode-install-all` | `opencode-install-all` |

連帶修掉的三件事：

1. **遞迴偵測**。lazy-pack 的技能在 `skills/` 子資料夾，舊版只掃頂層目錄，所以連找都找不到。同時要排除 `generated/`（安裝器的測試產物）與 `site-packages/`（Python 套件內建的技能）
2. **根目錄的 `SKILL.md` 是整包標記、不是技能**。它跟 `skills/` 是兄弟關係，若當成技能會把底下所有技能吃掉（實測 `claude-code-lazy-packs` 就是這樣，10 個技能只剩 1 個）
3. **只覆蓋已存在的副本**。技能包常綁單一工具（`codex-*` 只該進 Codex），四家全裝會把 `codex-draw` 塞進 Claude Code，比落後更糟；而且來源有、四家都沒裝的通常是刻意沒裝，自動補裝等於推翻使用者的選擇

## 已知限制

- **首裝仍要人工確認**。四家都沒裝過的技能不會自動裝，只會列出來問——這是刻意的（見上面第 3 點），但也表示「新技能第一次上線」還是需要一次手動決定。
- **只保證副本追得上源檔，不保證源檔是對的**。技能驗的是 bytes 一致，內容本身有沒有壞（例如指到已廢的路徑）不在守備範圍。2026-07-31 抓到的三個實際壞掉的技能，是靠人工讀內容發現的。

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
- 2026-07-31（`<電腦A>`）：全盤比對四家全域技能共 42 項。7 個來源 repo 皆 `clean` 且與 origin `0/0`（來源可信這關先過）；同名的專案類 16 項全部 byte-identical；改名安裝的 lazy-pack 類有 7 項落後，已逐項 `Copy-Item` 修復並驗到 byte-identical。由此確立〈已知限制〉與階段七。
- 2026-07-31（`<電腦A>`）：**階段七完成**。改名安裝的技能納入覆蓋範圍——做法沒照原訂的「手寫對照表」，改為從 frontmatter `name:` 推導安裝名（實測 42/42 成立，零維護）。連帶改了遞迴偵測（含雜訊排除）、根目錄 `SKILL.md` 視為整包標記、只覆蓋已存在的副本、`Copy-Item` 改用 `\*` 內容寫法、hash 比對兩邊加 `-Force`。動手前實測抓到三個真 bug（`Copy-Item` 巢狀陷阱、根目錄標記吃掉全部技能、`generated/` 假技能，細節見 L3 筆記）。驗證：`sync-skills` 自同步四家全 `OK`；唯讀掃四個 lazy-pack repo，34 個改名技能全部進入比對範圍且自動只比對正確那一家。
