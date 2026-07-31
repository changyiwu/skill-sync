---
name: sync-skills
description: 技能副本同步助手。當使用者說「同步技能」、「同步技能副本」、「技能同步」、「把技能複製到四個 agent」、「更新技能安裝副本」、「技能改完了幫我同步」等任何要把專案裡的技能原始檔覆蓋到 Claude Code／Codex／OpenCode／Antigravity 全域技能目錄的請求時，請一定要使用此技能。本技能會遞迴找出專案裡的技能、以 frontmatter 的 name 決定安裝名（改名安裝的技能也涵蓋）、先用 git 驗證原始檔可信（防雲端硬碟餵出過期內容）、只用 Copy-Item 從磁碟複製、最後逐檔 hash 比對證明副本真的一致。
---

# 技能副本同步助手

把「專案裡的技能原始檔」覆蓋到本機各 Agent 工具的全域技能目錄，並**證明**覆蓋成功。

適用於任何「原始檔放在專案資料夾、安裝副本散在各工具技能目錄」的技能包，不限定特定專案。來源資料夾名與安裝名**不一樣也沒關係**（例：`skills/08-draw` → `claude-draw`）。

## 鐵則（違反就等於沒同步）

1. **只能用 `Copy-Item` 從磁碟複製**。絕不可用 Write／Edit／heredoc「重建」副本——那會把 Agent context 裡記得的**舊內容**寫成「新版」，而且事後看起來跟正常同步一模一樣，只有逐檔比對抓得到。在「之前已開過、context 很長的舊對話」裡要求同步時風險最高。
2. **同步前先問 git，不要只看檔案內容**。雲端硬碟（Google Drive／OneDrive）會餵出過期的檔案 bytes，而 git 的 HEAD 已經是新版。拿那種源檔同步＝**把所有副本一起降版**，看檔案內容或時間戳都看不出來。
3. **同步後一定要驗**。沒跑逐檔 hash 比對就回報「已同步」，等於沒同步。
4. **安裝名取自 `SKILL.md` 的 `name:`，不是資料夾名**。兩者不同的技能（改名安裝）若照資料夾名同步，會在目標目錄旁邊生出一份沒人載入的孤兒副本，而真正被載入的那份繼續留在舊版——比不同步更難發現。

## 步驟 1：偵測來源技能

來源＝使用者當下的專案根目錄（沒講就用當前工作目錄；不確定就問）。技能可能在**任意深度**（很多技能包放在 `skills/` 子資料夾），所以要遞迴找 `SKILL.md`：

```powershell
$src  = (Get-Location).Path
$skip = '[\\/](\.git|node_modules|\.venv|venv|site-packages|generated|dist|build|__pycache__)[\\/]'
$all  = @(Get-ChildItem $src -Recurse -Filter 'SKILL.md' -File -ErrorAction SilentlyContinue |
          Where-Object { $_.FullName -notmatch $skip })
$sub  = @($all | Where-Object { $_.Directory.FullName -ne $src } | Sort-Object { $_.FullName.Length })
$pick = if ($sub.Count -gt 0) { $sub } else { @($all) }

$roots = @(); $skills = @()
foreach ($f in $pick) {
  $dir = $f.Directory.FullName
  if ($roots | Where-Object { $dir.StartsWith($_ + '\') }) { continue }   # 被外層技能包住，不算獨立技能
  $roots += $dir
  $name = ((Get-Content $f.FullName -TotalCount 12 | Where-Object { $_ -match '^\s*name:' } |
            Select-Object -First 1) -replace '^\s*name:\s*','').Trim().Trim('"',"'")
  $skills += [pscustomobject]@{
    Path        = $dir
    Folder      = if ($dir -eq $src) { '.' } else { $dir.Substring($src.Length).TrimStart('\') }
    InstallName = $name
    Valid       = ($name -match '^[A-Za-z0-9._-]+$')
  }
}
$skills | Format-Table Folder, InstallName, Valid -AutoSize
```

三個設計要點，改動前先看懂：

- **安裝名 ＝ frontmatter 的 `name:`**。這是各家工具的規則（資料夾名要跟 `name:` 一致），所以「來源資料夾 → 安裝名」的對照關係**本來就寫在源檔裡**，不需要另外手維護一張會過期的對照表。
- **根目錄的 `SKILL.md` 是整包標記，不是技能**。技能包常在根目錄放一份介紹整包的 `SKILL.md`（例：`name: claude-code-lazy-packs`），它跟 `skills/` 是兄弟關係。只要子資料夾裡找得到技能，根目錄那份就略過；**只有**全專案找不到子技能時，才把根目錄自己當成單一技能（`rdq-skill/SKILL.md` 那種寫法）。
- **`$skip` 不是可有可無**。實測會撈到安裝器產生的測試副本（`generated/`）與 Python 套件內建的技能（`site-packages/`），照同步等於拿垃圾覆蓋。

把 `Folder → InstallName` 對照念給使用者確認，再往下走。有 `Valid` 為 `False` 的（`name:` 有空白或中文，不能當資料夾名）→ **停下來**問使用者那一項該裝成什麼名字，或是否根本不該全域安裝。

有子技能或範本資料夾（例如 `templates/`）不用另外處理——步驟 5 是整個資料夾遞迴複製。

## 步驟 2：源檔可信度檢查（git）

```powershell
git diff HEAD --stat
git rev-list --left-right --count 'HEAD...@{u}'
```

> `'@{u}'` 在 PowerShell **一定要單引號包起來**，裸的 `@{` 會被當成 hashtable 語法、直接噴解析錯誤。

輸出是 `<領先>	<落後>`（tab 分隔）。判讀：

| 情況 | 怎麼做 |
|------|--------|
| worktree 乾淨、`0	0` | 源檔可信，往下跑 |
| 有未 commit 的改動，且是使用者這次剛改的 | 正常，往下跑（同步的就是這些改動） |
| 有**非預期**的改動（不是這次改的） | **停下來**問使用者：這些是什麼？要一起同步嗎？ |
| **落後 > 0** | **停下來**。磁碟很可能是雲端硬碟餵出的舊 bytes，先 `git pull` 再回來 |
| 領先 > 0 | 可往下跑，但回報時提醒「這些改動還沒 push」 |
| 不是 git repo | 註明「**無法用 git 驗證來源**，只能相信磁碟內容」，問使用者要不要照跑 |

## 步驟 3：偵測目標目錄

| 工具 | 全域技能目錄 |
|------|-------------|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` |
| Antigravity | `~/.gemini/config/skills/` |

**目錄不存在＝這台沒裝那個工具**，略過即可，不要建目錄、不要當成錯誤。

## 步驟 4：規劃同步矩陣（只覆蓋已存在的副本）

```powershell
$dests = [ordered]@{
  'Claude Code' = "$HOME\.claude\skills"
  'Codex'       = "$HOME\.agents\skills"
  'OpenCode'    = "$HOME\.config\opencode\skills"
  'Antigravity' = "$HOME\.gemini\config\skills"
}
$plan = foreach ($s in $skills) {
  foreach ($d in $dests.GetEnumerator()) {
    if (-not (Test-Path $d.Value)) { continue }
    $t = Join-Path $d.Value $s.InstallName
    if (Test-Path $t) { [pscustomobject]@{ Tool = $d.Key; Name = $s.InstallName; Source = $s.Path; Target = $t } }
  }
}
$plan | Format-Table Tool, Name -AutoSize
$skills | Where-Object { $n = $_.InstallName; -not ($plan | Where-Object { $_.Name -eq $n }) } |
  ForEach-Object { "未安裝：{0} -> {1}" -f $_.Folder, $_.InstallName }
```

**規則：某個技能在哪幾家已經裝過，就只覆蓋那幾家。**理由：

- 很多技能包是**綁單一工具**的（`codex-*` 只該進 Codex 目錄）。無條件四家全裝會把 `codex-draw` 塞進 Claude Code，比落後更糟。
- 來源裡有、但四家都沒裝的技能，通常是使用者**刻意沒裝**。自動補裝等於推翻使用者的選擇。

所以「未安裝」的那幾項要**列出來問**使用者要不要首裝，不要自己決定。使用者說要，才建目錄再複製：

```powershell
$t = Join-Path $dests['Claude Code'] '<安裝名>'
New-Item -ItemType Directory $t -Force | Out-Null
Copy-Item -Path (Join-Path '<來源絕對路徑>' '*') -Destination $t -Recurse -Force
```

## 步驟 5：複製

```powershell
foreach ($p in $plan) {
  Copy-Item -Path (Join-Path $p.Source '*') -Destination $p.Target -Recurse -Force
}
```

⚠️ **一定要用「複製內容」的 `\*` 寫法，不可以複製資料夾本身。**目標目錄已存在時，`Copy-Item <來源資料夾> <目標資料夾>` 會把來源**塞進目標裡面**變成巢狀（`claude-draw\08-draw\`），而目標原本那份 `SKILL.md` 完全沒被更新——實測證實過。`\*` 寫法才是把內容覆蓋上去，且 `-Force` 會一併複製隱藏檔。

## 步驟 6：驗證（逐檔 hash，不可省略）

```powershell
foreach ($p in $plan) {
  $sMap = @{}; $tMap = @{}
  Get-ChildItem $p.Source -Recurse -File -Force | ForEach-Object {
    $sMap[$_.FullName.Substring($p.Source.Length + 1)] = (Get-FileHash $_.FullName).Hash
  }
  Get-ChildItem $p.Target -Recurse -File -Force | ForEach-Object {
    $tMap[$_.FullName.Substring($p.Target.Length + 1)] = (Get-FileHash $_.FullName).Hash
  }
  $bad   = @($sMap.Keys | Where-Object { $tMap[$_] -ne $sMap[$_] })
  $extra = @($tMap.Keys | Where-Object { -not $sMap.ContainsKey($_) })
  $msg = if ($bad.Count -eq 0 -and $extra.Count -eq 0) { "OK（$($sMap.Count) 檔）" }
         else { "不一致 內容差異=[$($bad -join ', ')] 副本多出=[$($extra -join ', ')]" }
  "{0,-14} {1,-24} {2}" -f $p.Tool, $p.Name, $msg
}
```

`Get-ChildItem` 兩邊都要加 `-Force`，否則隱藏檔複製了卻不在比對範圍內。兩種不一致要分開看：

- **內容差異**：複製失敗，或檔案被鎖住。重跑步驟 5，還是不行就檢查是不是有 Agent／編輯器開著那個檔
- **副本多出**：原始檔已經**刪掉**的檔案還留在副本裡；若出現一個「跟來源資料夾同名的子資料夾」（例 `claude-draw\08-draw`），那就是踩到步驟 5 的巢狀陷阱。`Copy-Item -Force` 只覆蓋、不刪除，這種殘留要手動 `Remove-Item` 清掉（清之前先念給使用者確認）

## 步驟 7：回報

```
📦 來源：<專案資料夾名>（<N> 個技能）
🔍 源檔可信度：<git 乾淨且與遠端同步｜有 N 個未 push 的 commit｜非 git 專案>
📋 同步結果：
   Claude Code    claude-draw      OK（4 檔）      ← 來源 skills\08-draw
   Codex          codex-draw       OK（4 檔）
   OpenCode       目錄不存在，略過（這台沒裝）
⏭️ 未安裝（四家都沒有，要首裝再說）：claude-supabase、claude-ollama
✅ 全部一致 ／ ⚠️ <N> 項不一致：<明細與建議>
```

來源資料夾名與安裝名不同的，回報時**兩個都要寫**——不然使用者看不出哪個源檔對到哪個副本。

## 步驟 8：善後提醒

- 有用 **chezmoi** 管 dotfile 的話：`Copy-Item` 是**繞過 chezmoi 直接寫 target**，所以同步完 `chezmoi status` 第一欄**必然**出現這些技能。這是正常的，正解是 `chezmoi add --recursive` 收進來源再 commit + push，**不是 `apply`**（那會拿舊的 source 蓋掉剛同步好的新版）
- 源檔這次有改動但還沒 commit／push 的話，提醒使用者收工前補上

## 不該做的事

- ❌ 用 Write／Edit／`Set-Content` 產生副本內容（鐵則 1）
- ❌ 跳過步驟 2 直接複製（可能把所有副本一起降版）
- ❌ 跳過步驟 6 就說「已同步」
- ❌ 拿**資料夾名**當安裝名（鐵則 4）
- ❌ 把來源有、但四家都沒裝的技能自動補裝（步驟 4）
- ❌ 目標目錄不存在時自己 `New-Item` 建起來（那台沒裝那個工具，建了只是留垃圾）
- ❌ 擅自 `git pull`／`git commit`／`git push`（本技能只讀 git 狀態，不動 git）
- ❌ 順手「修正」原始檔的內容（同步就只是同步）

## 注意事項

- 所有訊息使用**繁體中文**
- **占位符要填在原始檔**。若做法是「原始檔留 `<你的帳號>` 占位符、裝好之後才在副本裡替換」，步驟 6 的 hash 比對會**每次都報不一致**。建議原始檔就填好本機的值，副本純粹是複製品
- `SKILL.md` 不可存成含 BOM 的 UTF-8，否則 frontmatter 解析失敗、技能觸發不了。連帶地步驟 1 也會讀不到 `name:`、安裝名變空字串。`Copy-Item` 是位元組複製，不會改變編碼——BOM 問題只會在**編輯原始檔**時發生
- 某些工具的安裝器（如 `npx skills add`）會把檔案寫成 CRLF，於是 hash 跟來源的 LF 版不同、步驟 6 報「內容差異」但內容其實一樣。確認只有換行差異的話可以不處理，但要在回報裡講清楚，別讓它混在真正的不一致裡
- 本 skill 的**原始檔**在 `skill-sync/sync-skills/`（`skill-sync` 專案資料夾本身放哪由你決定，建議放雲端硬碟資料夾以便跨電腦同步）。這個技能自己也適用自己：改完在該專案說「同步技能」即可
