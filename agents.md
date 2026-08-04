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
- [x] 階段三：把 `sync-skills` 安裝到四家技能目錄（三台都已裝）
- [x] 階段四：git init ＋ GitHub repo（L2，公開）
- [x] 階段五：建立 L3 Obsidian 專案筆記
- [x] 階段六：跨電腦實測（`<電腦C>` 首裝後跑「同步技能」，並用清單補齊落差的 11 個技能）
- [x] 階段七：讓改名安裝的技能納入同步（安裝名改為讀 frontmatter `name:`，不再靠資料夾名）
- [x] 階段八：跨電腦安裝清單（`.skill-install/<電腦名>.json`），分辨「只有這台沒裝」與「都沒裝」
- [x] 階段九：修掉階段八盤點抓到的三個複製缺陷（衍生物排除改在技能內；另兩個從**來源端**根治）
- [ ] （選作）階段十：回報加上反向指標「已安裝但四家都查無來源」，把目前混在「都沒裝」裡的「安裝名對不上」拆出來
- [x] 階段十一：從**來源端**封住 CRLF 無聲改寫路徑（28 個 repo 補 `.gitattributes`，並寫進 `project-init` 讓新 repo 一建立就有）
- [x] 階段十二：全專案審查，補掉三個「檢查看起來有效、實際失效」的洞（步驟 1 的排除規則比錯對象、步驟 2 沒 `fetch`、CRLF 知識只在 `agents.md` 而執行時讀不到）

## 資料夾結構

```
skill-sync/
├─ .gitignore
├─ .gitattributes     # `* text=auto eol=lf`（階段十一，見〈已知限制〉的 CRLF 那條）
├─ README.md          # 技能說明與安裝方式
├─ agents.md          # 本檔：專案藍圖
├─ CLAUDE.md          # 橋接檔（@agents.md）
├─ handoff.md         # 交接檔（每次收工必更新）；已 gitignore，只走 L1
└─ sync-skills/
   └─ SKILL.md        # 「同步技能」技能
```

## 已知限制

技能守不住的邊界。完整成因、實測數據與當初怎麼取捨都在 L3 筆記，這裡只留結論：

- **首裝要人工確認**。四家都沒裝過的技能只列出來問、不自動裝——刻意的：同步工具的預設行為應該是**收斂到現狀**，不是擴張
- **不相容的安裝模式技能偵測不到，只能靠來源端不要那樣做**。三種已從來源端根治：占位符替換型（`html-slide-builder`）、安裝時改寫 `name:`（`agents-lazy-guide`）、專案根目錄即技能（`rdq-skill`）。**同樣模式再出現時不會有任何警告**
- **「都沒裝」這個分類會安靜吃掉「安裝名對不上」的技能**，兩者在畫面上一模一樣。反向指標（已安裝但四家查無來源）尚未實作＝路線圖階段十（成因與判讀方式詳見 L3 筆記〈踩坑筆記〉）
- **安裝時注入使用者金鑰仍不相容**：`html-slide-builder` 的 `references/firebase-config.md` 一旦設定 Firebase 就不等於源檔（三台目前都沒設定，所以是條件式限制而非固定狀態）
- **CRLF 有兩條路進來，技能擋不住、但（階段十二起）會出聲**：步驟 2 會查 `autocrlf` ＋ `.gitattributes` 並警告，步驟 6 會把「僅換行／BOM 差異」跟真落後分開。**根治仍只能靠來源端的 `.gitattributes`**（`agents/` 底下 30 個 repo 已全數有，且已寫進 `project-init`）——技能刻意不做正規化比對，那會讓它對「副本被安裝器改寫」失明
- **只保證副本追得上源檔，不保證源檔是對的**。驗的是 bytes 一致；內容本身壞掉（例如指到已廢路徑）不在守備範圍，那要靠人讀

## 專案專屬規則

- **CRLF 三條規則的正本在 `sync-skills/SKILL.md`，不要在本檔另立一份**：`autocrlf=true` 讓步驟 2 失效（步驟 2 的警告）、判斷「只有換行差異」要去 BOM ＋ CRLF→LF 重比（步驟 6 的 `Get-FileFacts`）、量換行不要用 PowerShell 管線（步驟 6）。**理由：技能是在別的專案裡跑的，那時讀不到本檔**——凡是執行時才需要的知識，只寫在這裡等於沒寫
- **但「工作區 CRLF 的兩種成因」只在本檔**（診斷 repo 用，不在同步流程裡）：用 `git cat-file blob HEAD:<路徑>` 對工作區逐檔比 CRLF 數，兩邊相同＝committed CRLF（每台 clone 都一樣、不會漂移，不用管），只有工作區有＝checkout 轉出來的（危險）。**`git status` 沉默證明不了工作區是 LF**——有 `.gitattributes` 時 git 比對本來就會正規化，本 repo 自己就是「工作區與 blob 都是 CRLF 卻一直 `clean`」的實例
- **同一檔案前後讀到不同內容時，先查 commit 時間再下結論**。「來源被人改了」比「Drive 餵舊 bytes」常見得多；`git update-index --really-refresh` 的建議已評估後撤回，**不要照著加進技能**
- **三個 PowerShell 實作坑的正本也在 `sync-skills/SKILL.md`，本檔只留結論**：`Copy-Item <資料夾> <資料夾>` 在目標已存在時會塞成巢狀、原本那份一個 byte 都沒動且不報錯 → 一律「列舉再逐檔複製」（步驟 5）；排除規則比 `FullName` 會讓含 `\build\` 路徑的專案整包被跳過且不報錯 → 一律比相對路徑（步驟 1、5、6）；`TryParseExact` 在 PowerShell 綁不上 5 參數多載，會讓**所有**電腦被誤標成「時間格式無法解讀」→ 用 `try/catch` ＋ `ParseExact`（步驟 4）。**理由與 CRLF 那條不同：這三條 `SKILL.md` 本來就有完整正本，兩邊各存一份只會分歧**——要改成因或實測細節，改那邊
- **沙箱會攔下含 `Remove-Item` 的整段 PowerShell 指令**（訊息 `system path '/' is blocked`），清暫存檔改用 Bash 的 `rm`
- **讀 `handoff.md` 先看「更新者 @ 哪台」**：技能副本每台各一份，交接檔描述的永遠是單機狀態，「88/88 全 `OK`」對另一台沒有任何意義

## 同步層級（本專案初始化至第 3 層級）

| 層級 | 平台 | 位置 | 讀取時機 |
|------|------|------|---------|
| L1 | 本地（GDrive） | `agents.md`＋`handoff.md`＋`CLAUDE.md`（橋接）；另有專案上層的 `.skill-install/<電腦名>.json` | 每個 session |
| L2 | GitHub | https://github.com/changyiwu/skill-sync （公開） | 指定時 |
| L3 | Obsidian | `skill-sync/專案工作流程.md` | 有需要時 |

## 三個檔案的職責（依「時效性」分家，不是依「詳細程度」）

| 檔案 | 時效 | 寫入方式 | 放什麼 |
|------|------|---------|--------|
| `handoff.md` | **只對下一個 session 有效**，過期即丟 | 每次收工整份重寫 | 做到哪、下一步、**這次**的暫時 workaround |
| `agents.md`（本檔） | **長期有效**，每個 session 都適用 | 只有規則本身變了才改 | 目標、路線圖、常設規則、結構 |
| Obsidian／`git log` | **歷史**：發生過什麼、為什麼 | 只增不刪 | 決策紀錄、踩坑完整版、逐次進度 |

驗收標準：**`handoff.md` 整份刪掉，不應損失任何長期資訊**——會的話代表該升級進本檔卻沒升級。

**本檔不要出現的東西**：❌ `## 最近進度`／逐次工作紀錄、❌ 完整的決策論述。2026-08-03 移除了 `## 最近進度`（13 條）與兩節「設計決策」（階段七、階段八），內容在 L3 筆記的〈🗓️ 最近更動紀錄〉與〈🧠 決策紀錄〉都有更完整的版本——**是主動移除，不是遺漏，不要補回來**。

## 工作約定

- 任何 Agent、任何電腦：**開工先讀 `handoff.md`，收工必更新 `handoff.md`**
- `handoff.md` **不進 git**（含真實電腦名與本機絕對路徑），跨電腦靠雲端硬碟同步——不要把它加回版控
- 修改共用檔案前先讀最新內容，避免覆蓋其他 Agent 的變更
- 修改前先確認計畫，優先保留原有資料結構
- 所有回應與文件使用繁體中文
- **本資料夾是技能原始檔**。改動一律改這裡，改完在本專案說「同步技能」覆蓋四份安裝副本
- 編輯 `SKILL.md` 時不可存成含 BOM 的 UTF-8，否則 frontmatter 解析失敗、技能觸發不了
- 這個技能的鐵則同樣約束改它的人：**副本一律 `Copy-Item` 從磁碟複製**，絕不可用 Write/Edit 重建

## 相關專案

- `cross-device-agent-skills`：三技能（初始化專案／開工／收工）。本技能原本是它 README 的一段，抽出後那邊改為指向這裡
- `chezmoi-setup`：`chezmoi-sync` 技能。管的是 dotfile 的 target↔source↔GitHub 三方一致，與本技能是**兩條不同路徑**——`Copy-Item` 繞過 chezmoi 直接寫 target，所以同步完 `chezmoi status` 必然報出這些技能，正解是 `chezmoi add` 不是 `apply`
