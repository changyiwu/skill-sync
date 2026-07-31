@agents.md

<!--
  本檔是「橋接檔」：Claude Code 只讀 CLAUDE.md，不讀 agents.md，
  所以用第一行的 @agents.md 把跨 Agent 專案藍圖 import 進來。
  專案內容一律寫進 agents.md，這裡只放 Claude Code 專屬規範，避免兩份分叉。
-->

## Claude Code 專屬

- 改技能一律改本資料夾的原始檔，改完在本專案說「同步技能」；**絕不可用 Write/Edit 重建 `~/.claude/skills/` 的安裝副本**
- 驗證橋接是否生效：跑 `/context`，看 **Memory files** 有沒有列到 `CLAUDE.md`
