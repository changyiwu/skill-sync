# 技能副本同步（skill-sync）

> 一個口令：「**同步技能**」。把專案裡的技能原始檔安全地覆蓋到四個 Agent 工具的全域技能目錄，並**證明**覆蓋成功。

## 這包裡有什麼

| 技能 | 口令 | 做什麼 |
|------|------|--------|
| `sync-skills` | 「同步技能」 | 偵測專案裡的技能資料夾 → 用 git 驗證源檔可信 → `Copy-Item` 覆蓋四家安裝副本 → 逐檔 hash 比對回報 |

適用於任何「原始檔放在專案資料夾、安裝副本散在各工具技能目錄」的技能包。

## 為什麼需要一個技能，而不是一行 `Copy-Item`

三個坑，光是複製只擋得住第一個：

1. **Agent 用 Write/Edit「重建」副本** — 它會把 context 裡記得的舊內容寫成「新版」，事後看起來跟正常同步一模一樣，只有逐檔比對抓得到。在已開很久的舊對話裡要求同步時風險最高
2. **雲端硬碟餵出過期的檔案 bytes** — 磁碟讀到舊內容、git 的 HEAD 已是新版。照樣同步＝把所有副本**一起降版**，看檔案內容或時間戳都看不出來，一定要問 git
3. **同步完沒驗** — 複製失敗、檔案被鎖、原始檔已刪的檔案殘留在副本裡，都不會有人發現

所以技能有三條鐵則：只能 `Copy-Item` 從磁碟複製、複製前先問 git、複製後一定跑 hash 比對。

## 支援的技能目錄

| 工具 | 全域技能目錄 |
|------|-------------|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` |
| Antigravity | `~/.gemini/config/skills/` |

四家的技能格式相同（`SKILL.md` + YAML frontmatter），所以**同一份檔案直接共用**。目錄不存在＝這台沒裝那個工具，技能會略過、不會建目錄。

## 安裝

把 `sync-skills/` 複製到你的全域技能目錄（第一次只能手動）。在這個 repo 的根目錄跑：

```powershell
$src = Join-Path (Get-Location) 'sync-skills'
foreach ($d in "$HOME\.claude\skills","$HOME\.agents\skills","$HOME\.config\opencode\skills","$HOME\.gemini\config\skills") {
  if (Test-Path $d) { Copy-Item $src "$d\" -Recurse -Force }
}
```

裝好之後，這個技能就能同步自己了——在本專案說「同步技能」即可。

## 怎麼用

在**技能原始檔所在的專案**裡說：

> 「同步技能」

技能會：

1. 列出這個專案裡含 `SKILL.md` 的資料夾，念給你確認
2. 跑 `git diff HEAD --stat` 與 `git rev-list --left-right --count 'HEAD...@{u}'` 判斷源檔可不可信（落後遠端就停下來）
3. 對存在的技能目錄跑 `Copy-Item -Recurse -Force`
4. 逐檔遞迴 hash 比對，分開回報「內容差異」與「副本多出的殘留檔」
5. 提醒 chezmoi 善後（`Copy-Item` 繞過 chezmoi 直接寫 target，同步完 `chezmoi status` 必然報出這些技能，正解是 `chezmoi add --recursive` 不是 `apply`）

## 注意事項

- **占位符要填在原始檔**。若做法是「原始檔留 `<你的帳號>`、裝好才在副本裡替換」，hash 比對會每次都報不一致
- 編輯 `SKILL.md` 不可存成含 BOM 的 UTF-8，否則 frontmatter 解析失敗、技能觸發不了。`Copy-Item` 是位元組複製，不會改變編碼——BOM 只會在編輯原始檔時出問題
- `'@{u}'` 在 PowerShell 一定要單引號包起來，裸的 `@{` 會被當成 hashtable 語法直接噴解析錯誤

## 相關專案

- `cross-device-agent-skills` — 三技能（初始化專案／開工／收工）。本技能原本是它 README 的一段
- `chezmoi-setup` — `chezmoi-sync` 技能，管 dotfile 三方一致。與本技能是兩條不同路徑
