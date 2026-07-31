---
name: sync-skills
description: 技能副本同步助手。當使用者說「同步技能」、「同步技能副本」、「技能同步」、「把技能複製到四個 agent」、「更新技能安裝副本」、「技能改完了幫我同步」等任何要把專案裡的技能原始檔覆蓋到 Claude Code／Codex／OpenCode／Antigravity 全域技能目錄的請求時，請一定要使用此技能。本技能會先用 git 驗證原始檔可信（防雲端硬碟餵出過期內容）、只用 Copy-Item 從磁碟複製、最後逐檔 hash 比對證明四份副本真的一致。
---

# 技能副本同步助手

把「專案裡的技能原始檔」覆蓋到本機各 Agent 工具的全域技能目錄，並**證明**覆蓋成功。

適用於任何「原始檔放在專案資料夾、安裝副本散在各工具技能目錄」的技能包，不限定特定專案。

## 鐵則（違反就等於沒同步）

1. **只能用 `Copy-Item` 從磁碟複製**。絕不可用 Write／Edit／heredoc「重建」副本——那會把 Agent context 裡記得的**舊內容**寫成「新版」，而且事後看起來跟正常同步一模一樣，只有逐檔比對抓得到。在「之前已開過、context 很長的舊對話」裡要求同步時風險最高。
2. **同步前先問 git，不要只看檔案內容**。雲端硬碟（Google Drive／OneDrive）會餵出過期的檔案 bytes，而 git 的 HEAD 已經是新版。拿那種源檔同步＝**把所有副本一起降版**，看檔案內容或時間戳都看不出來。
3. **同步後一定要驗**。沒跑逐檔 hash 比對就回報「已同步」，等於沒同步。

## 步驟 1：確認來源與範圍

來源＝使用者當下的專案根目錄（沒講就用當前工作目錄；不確定就問）。技能資料夾＝根目錄下**含 `SKILL.md` 的子資料夾**：

```powershell
$src = (Get-Location).Path
Get-ChildItem $src -Directory |
  Where-Object { Test-Path (Join-Path $_.FullName 'SKILL.md') } |
  Select-Object -ExpandProperty Name
```

把偵測到的清單念給使用者確認，再往下走。有子技能或範本資料夾（例如 `templates/`）不用另外處理——步驟 4 是整個資料夾遞迴複製。

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

## 步驟 4：複製

把步驟 1 確認的清單填進 `$names`：

```powershell
$src   = (Get-Location).Path
$names = @('技能A','技能B')
$dests = [ordered]@{
  'Claude Code' = "$HOME\.claude\skills"
  'Codex'       = "$HOME\.agents\skills"
  'OpenCode'    = "$HOME\.config\opencode\skills"
  'Antigravity' = "$HOME\.gemini\config\skills"
}

foreach ($d in $dests.GetEnumerator()) {
  if (-not (Test-Path $d.Value)) { "{0,-12} 目錄不存在，略過（這台沒裝）" -f $d.Key; continue }
  foreach ($n in $names) { Copy-Item "$src\$n" "$($d.Value)\" -Recurse -Force }
}
```

## 步驟 5：驗證（逐檔 hash，不可省略）

```powershell
foreach ($d in $dests.GetEnumerator()) {
  if (-not (Test-Path $d.Value)) { continue }
  foreach ($n in $names) {
    $sRoot = Join-Path $src $n
    $tRoot = Join-Path $d.Value $n
    $sMap = @{}; $tMap = @{}
    Get-ChildItem $sRoot -Recurse -File | ForEach-Object {
      $sMap[$_.FullName.Substring($sRoot.Length + 1)] = (Get-FileHash $_.FullName).Hash
    }
    if (Test-Path $tRoot) {
      Get-ChildItem $tRoot -Recurse -File | ForEach-Object {
        $tMap[$_.FullName.Substring($tRoot.Length + 1)] = (Get-FileHash $_.FullName).Hash
      }
    }
    $bad   = @($sMap.Keys | Where-Object { $tMap[$_] -ne $sMap[$_] })
    $extra = @($tMap.Keys | Where-Object { -not $sMap.ContainsKey($_) })
    $msg = if ($bad.Count -eq 0 -and $extra.Count -eq 0) { "OK（$($sMap.Count) 檔）" }
           else { "不一致 內容差異=[$($bad -join ', ')] 副本多出=[$($extra -join ', ')]" }
    "{0,-14} {1,-16} {2}" -f $d.Key, $n, $msg
  }
}
```

兩種不一致要分開看：

- **內容差異**：複製失敗，或檔案被鎖住。重跑步驟 4，還是不行就檢查是不是有 Agent／編輯器開著那個檔
- **副本多出**：原始檔已經**刪掉**的檔案還留在副本裡。`Copy-Item -Force` 只覆蓋、不刪除，這種殘留要手動 `Remove-Item` 清掉（清之前先念給使用者確認）

## 步驟 6：回報

```
📦 來源：<專案資料夾名>（<N> 個技能：<清單>）
🔍 源檔可信度：<git 乾淨且與遠端同步｜有 N 個未 push 的 commit｜非 git 專案>
📋 同步結果：
   Claude Code    技能A  OK（3 檔）
   Claude Code    技能B  OK（1 檔）
   Codex          …
   OpenCode       目錄不存在，略過（這台沒裝）
   Antigravity    …
✅ 全部一致 ／ ⚠️ <N> 項不一致：<明細與建議>
```

## 步驟 7：善後提醒

- 有用 **chezmoi** 管 dotfile 的話：`Copy-Item` 是**繞過 chezmoi 直接寫 target**，所以同步完 `chezmoi status` 第一欄**必然**出現這些技能。這是正常的，正解是 `chezmoi add --recursive` 收進來源再 commit + push，**不是 `apply`**（那會拿舊的 source 蓋掉剛同步好的新版）
- 源檔這次有改動但還沒 commit／push 的話，提醒使用者收工前補上

## 不該做的事

- ❌ 用 Write／Edit／`Set-Content` 產生副本內容（鐵則 1）
- ❌ 跳過步驟 2 直接複製（可能把所有副本一起降版）
- ❌ 跳過步驟 5 就說「已同步」
- ❌ 目標目錄不存在時自己 `New-Item` 建起來（那台沒裝那個工具，建了只是留垃圾）
- ❌ 擅自 `git pull`／`git commit`／`git push`（本技能只讀 git 狀態，不動 git）
- ❌ 順手「修正」原始檔的內容（同步就只是同步）

## 注意事項

- 所有訊息使用**繁體中文**
- **占位符要填在原始檔**。若做法是「原始檔留 `<你的帳號>` 占位符、裝好之後才在副本裡替換」，步驟 5 的 hash 比對會**每次都報不一致**。建議原始檔就填好本機的值，副本純粹是複製品
- `SKILL.md` 不可存成含 BOM 的 UTF-8，否則 frontmatter 解析失敗、技能觸發不了。`Copy-Item` 是位元組複製，不會改變編碼——BOM 問題只會在**編輯原始檔**時發生
- 本 skill 的**原始檔**在 `skill-sync/sync-skills/`（`skill-sync` 專案資料夾本身放哪由你決定，建議放雲端硬碟資料夾以便跨電腦同步）。這個技能自己也適用自己：改完在該專案說「同步技能」即可
