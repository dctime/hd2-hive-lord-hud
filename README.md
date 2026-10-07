# How Helldivers 2 mods work — told through building Hive Lord HUD

**[Read the tutorial (English)](https://dctime.github.io/hd2-hive-lord-hud/)** · **[閱讀教學（繁體中文）](https://dctime.github.io/hd2-hive-lord-hud/zh.html)**

How can code that Arrowhead never wrote run inside Helldivers 2? How does a mod find the game's data, and how do the tools around modding (the loader, mod managers, FileDiver, a recon mod, a disassembler) actually work? This tutorial answers that from first principles, one idea per chapter, following the real development of **Hive Lord HUD** from a one-line idea to its third release. Every chapter starts with the question it answers, introduces at most a handful of new concepts, and ends with key points and self-check questions. It includes interactive demos (a murmur64 hash calculator, a patch-file byte viewer, a memory-record explorer, and more).

**[Source code, linked to the tutorial](https://dctime.github.io/hd2-hive-lord-hud/source.html)**: the full source of both mods as shipped, with every block that a chapter explains highlighted and linked back to that chapter. Each chapter also has an "In the source" box pointing to the exact lines.

## The mods: Hive Lord HUD and Dead Plate Fix

**Download:** [Releases](https://github.com/dctime/hd2-hive-lord-hud/releases/latest) → `HiveLordHUDv3.zip`. The full Lua source is inside the zip (`Source/`).

**Hive Lord HUD**
- Boss HP bar (out of 150,000) with a damage trail and a combo counter.
- Armour panel: a silhouette of the Hive Lord with the armour % of every plate, fin and head part. Poly (default) or Pixel art style; Monochrome (default) or Colour palette.
- On-screen tag with distance, in the style of the game's squad markers; an edge arrow when off-screen.
- Emerge warning when the Hive Lord stops underground or rears up within 100 m.
- All settings in the escape menu > MODS > Hive Lord HUD, including sizes, colour correction and a 15-second preview.

**Dead Plate Fix** (optional, works without the HUD)
- Sometimes a plate is destroyed (often by a teammate's Eagle) but stays drawn on the body: it looks intact while your shots pass straight through it. This fix hides those plates the same way the game normally would.

**Install**
1. Requires **Bingus Shared Loader**. Optional: **Mod Options Menu** (part of Vanilla Plus Megapack) for the in-game settings.
2. Import `HiveLordHUDv3.zip` into Arsenal or HD2 Mod Manager.
3. In Options, pick **Hive Lord HUD**, **Dead Plate Fix**, or both, then deploy.

**Notes**
- Made for game build 25480438. A game update may move what the mods read; the HUD then falls back to a slower search and the plate fix stops acting.
- Client-side and visual only. The HUD only reads memory; Dead Plate Fix writes one "hide this plate" bit, the same one the game itself sets. Damage, collision, network traffic and the game's code are never touched.
- Unofficial modification. Use at your own risk.

### Why the HP can be off when you're not the host

Short version: **the HUD shows the numbers your own game has, and when you're a client those are not always the host's numbers.** The mod can't fix that, because the host never sends the true values. Here is why.

**Every player's game keeps its own copy of the Hive Lord's health.** Helldivers 2 doesn't stream HP from the host to everyone. Each machine simulates the shots and explosions it sees and subtracts the damage locally. The host only sends *results*: "this plate cracked", "this plate is destroyed", "the Hive Lord died". The main HP is never re-synced mid-fight. (The game only broadcasts a fresh main HP when something is healed or revived, which never happens to a Hive Lord.)

So your copy drifts from the host's in three situations:

| Situation | What happens on your machine | How big it gets |
|---|---|---|
| **Damage your game never simulated** (which sources exactly is not confirmed; likely some damage-over-time effects and things only the host simulates) | That damage is missing from your main HP, so it reads **higher** than the truth | Usually 0–2% of the 150,000. In a few fights we logged 15–20%, and once almost 90% |
| **The same hit lands on a different zone** on your machine than on the host's | Main HP stays about right, but individual plates fall behind. When the host says a plate cracked or was destroyed, your copy snaps to it | Plates up to tens of thousands behind |
| **You joined mid-fight** | You receive the main HP as a snapshot (sometimes stale), plates that were already cracked, and dead fins. Damage dealt to a plate before you joined is never sent, so it starts from full | The snapshot was up to ~51,000 off in one test |

**What you see in the HUD:**
- When the mod knows a number can only be an upper bound, it puts a **`<`** in front of it (for example `<40%` on a plate, and in front of the main HP). That happens after a mid-fight join, and for plates after the host has corrected one of them.
- When the Hive Lord dies, the host's death event snaps your main HP to 0, so you may see a sudden drop at the end.
- If you were there from the start and you're close to the fight, the bar is normally within a couple of percent.
- As host, your game *is* the authority, so the numbers should be exact (the author always plays as a client, so this hasn't been tested).

**Why not just ask the host?** A client mod can only read its own game's memory. The game has no message for "send me the true health", and making the mod send network requests would mean affecting other players' games, which these mods deliberately never do.

For the full story, with the evidence from the logs, see [Chapter 15 of the tutorial](https://dctime.github.io/hd2-hive-lord-hud/#m15).

---

# Helldivers 2 模組是怎麼運作的：以製作 Hive Lord HUD 為例

**[閱讀教學（繁體中文）](https://dctime.github.io/hd2-hive-lord-hud/zh.html)** · **[Read the tutorial (English)](https://dctime.github.io/hd2-hive-lord-hud/)**

一個不是開發商寫的程式，怎麼會在 Helldivers 2 裡執行？模組怎麼找到遊戲的資料？做模組用的工具（Loader、模組管理器、FileDiver、偵察模組、反組譯器）又各自是什麼原理？這份教學從最基礎講起，一章講一個原理，用 **Hive Lord HUD** 從一句話的想法做到第三版發佈的真實過程當主線。每章開頭是這一章要回答的問題，最多介紹幾個新概念，結尾有重點和自我檢查，並附互動示範（murmur64 雜湊計算、patch 檔 bytes 檢視器、記憶體紀錄結構圖等）。

**[原始碼導讀](https://dctime.github.io/hd2-hive-lord-hud/source-zh.html)**：兩個模組發佈時的完整原始碼，教學文說明過的段落都有標色，並連回講它的那一章；每一章也有「在原始碼裡」區塊，指到確切的行號。

## 模組：Hive Lord HUD 與 Dead Plate Fix

**下載：** [Releases](https://github.com/dctime/hd2-hive-lord-hud/releases/latest) → `HiveLordHUDv3.zip`。完整的 Lua 原始碼在 zip 裡的 `Source/` 資料夾。

- **Hive Lord HUD**：Boss 血條、裝甲面板（每塊板子、鰭、頭部零件的剩餘 %）、HL 畫面標記、出土警告。所有設定在 Esc 選單 > MODS > Hive Lord HUD。
- **Dead Plate Fix**（選配，可單獨使用）：板子已經被打掉（常見於隊友的 Eagle）、子彈已經能穿過去，模型卻還畫在身上時，用遊戲自己的方式把它藏起來。
- 需要 **Bingus Shared Loader**；選配 **Mod Options Menu**（Vanilla Plus Megapack 內附）。用 Arsenal 或 HD2 Mod Manager 匯入 zip，在 Options 勾選要的功能。
- 對應遊戲 build 25480438。只在用戶端執行、只改變你自己看到的畫面。非官方修改，風險自負。
