! Please share this post with people who are hosting a CS:GO server and want to fix this issue !

If you have a CS:GO (NOT CS2) server, users can't join your server IF they are using the archived version of CS:GO (app/4465480/CounterStrike_Global_Offensive/).
If you want to allow those users to join, follow this guide.

!THIS ONLY WORKS IF YOU ARE THE SERVER OWNER WITH ACCESS TO THE SERVER FILES!

---

STORY & TUTORIAL

Valve archived CS:GO and made it available as a separate Steam app. The problem is that
clients using the archived build **can't connect to community servers**.

The auth ticket comes in with a mismatched appid and the server rejects it:

    S3: Client connected with ticket for the wrong game
    RejectConnection: STEAM validation rejected

**Root cause**

The engine has a jump table dispatch that handles appid validation.
Case 4 (archived build) hits a rejection path instead of the default OK path.

---

**Fix: SourceMod Extension (recommended)**

Download the extension and drop these files into your server:

    addons/sourcemod/extensions/csgo_steamfix.ext.so       <- Linux
    addons/sourcemod/extensions/csgo_steamfix.ext.dll      <- Windows
    
    addons/sourcemod/extensions/csgo_steamfix.autoload     <- auto-load trigger (empty file) [no clue if it requires .ext or no XD]
    addons/sourcemod/extensions/csgo_steamfix.ext.autoload <- auto-load trigger (empty file)

The extension loads automatically on server start, patches the engine, and
**restores the original state on unload**. (Works on both Linux and Windows)

If it works, you'll see this in the console / steamfix.log (Windows):

    [steamfix] loading..
    [steamfix] engine patched!

After this, **everyone can connect**, archived build clients included.

**You can download the extension from the repo:**
https://github.com/eonexdev/csgo-sv-fix-engine

---

**Manual fix** *(only if you know reverse engineering):*

Find the jmp dispatch via IDA (sigmaker):

    Linux: FF 24 85 ? ? ? ? 8D B4 26 ? ? ? ? 31 F6
    Windows: FF 24 85 ? ? ? ? FF 75 ? 68

That's `jmp ds:jpt[eax*4]`. Then locate the jump table:

**Linux** (`.rodata:jpt_18A2BA`):
- `[0]` → default (status OK)
- `[4]` → loc_18A3D8 ← rejection path

Solution: copy `jt[0]` into `jt[4]` at runtime.

**Windows** (`.text:jpt_1BF138`):
- The compiler emits `dec eax` before the dispatch, so case 4 maps to index 3
- `[3]` → loc_1BF189 ← rejection path
- The success path (`def_1BF138`) is NOT in the table, it's reached via a `ja` instruction
- Compute its address from the `ja rel32` sitting 6 bytes before the `jmp ds:jpt`
- Write that address into `jt[3]`

The table address is embedded in the instruction itself (`FF 24 85 [addr]`),
so no separate signature is needed for the table.

> Should work on any engine version, just find the jmp dispatch and identify
> which table index maps to the wrong-game rejection case.

---

**Note on the old method**

The original fix was a Linux-only shared lib loaded via LD_PRELOAD:

    LD_PRELOAD=/home/container/csgo_steamfix.so ./srcds_run -game csgo

That still works if you prefer it, but the SourceMod extension is easier to use,
works on Windows too, and (properly) restores the engine state on unload.
