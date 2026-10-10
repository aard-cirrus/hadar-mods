# Cirrus's MODs of Hadar's plugins

## Hadar-Mods: changes to SpellupRecast ([Hadar_Spellups.xml](https://github.com/aard-cirrus/hadar-mods/blob/master/SpellupRecast/Hadar_Spellups.xml))

This fork is based on [zzyzzyzzx/Hadar](https://github.com/zzyzzyzzx/Hadar) **v1.17**. All credit for the original plugin goes to Hadar. The modified version is **v1.22**.

Install: open the file above, click **Raw**, save it into your plugins folder, and add it in MUSHclient (or reinstall if you already have Hadar's version).

### Added

#### New commands

| Command | What it does |
|---|---|
| `hsp miniwin on\|off\|reset` | The Special Affects Tracker window (see below). `reset` puts its position, size and font back to the defaults. |
| `hsp diag` | Dumps the plugin's internal state in one go: what's blocking casts, settings, recast queue, active recoveries, sanctuary/aura state, and per-skill tracking with live timers. |
| `hsp refresh` | Sends the server `spellup` command, re-reads `slist` so recovery timers resync with the server, and checks sprite invis with `aflags`. Fixes cooldowns that got stuck. |
| `hsp <skill> debug` | Shows every decision the plugin makes about one tracked skill (e.g. `hsp spriteinvis debug`), whether or not `hsp debug` is on. Run it again to stop. |
| `hsp sanc` | Forces a sanctuary/aura restore attempt, clearing any backoff. |
| `hsp room` | Dumps GMCP `room.info`, including room flags. |
| `hsp death mana <amount>` | Resumes spell autocast at a fixed mana value instead of a percentage. `hsp death mana off` goes back to percentage mode. |
| `dese` | Toggles desecration tracking on/off (same as `hsp desecration on\|off`). |

#### Tracked skills the server `spellup` command skips

| Command | Skill | Requirement | Used in combat |
|---|---|---|---|
| `hsp quickstab on\|off` | Quickstab | Ninja | no |
| `hsp stealth on\|off` | Stealth | Ninja | no |
| `hsp stalk on\|off` | Stalk | Ninja | no |
| `hsp shadow on\|off` | Shadow form | Must be learned (no class check, so it works with psi as a secondary) | no |
| `hsp spriteinvis on\|off` (or `hsp sprite`) | Sprite racial invis | Sprite race; **switches on by itself** for Sprites | no |
| `hsp desecration on\|off` (or `dese`) | Desecration | Harmer | **yes** |

- Quickstab, stealth and stalk have cooldowns that last longer than the buff. When the buff drops, the plugin waits and recasts once the recovery clears.
- Shadow form, sprite invis and desecration have no such wait: they're reused as soon as they drop, to keep them up all the time.
- Checked again every 10 seconds while you can act. After each attempt the plugin waits up to 10 seconds for the game's answer before trying again, so nothing is spammed.
- Skill numbers and recovery groups are read from `slist`, not hardcoded.
- The toggle has the final say. A skill that's switched off won't be recast by any path.

#### Sprite invis
The racial ability sends **no affect tags**, so it's tracked from the game's own messages:

- **Up:** "You vanish out of sight!", "You fade out of existence." or "You are already invisible."
- **Down:** "You are no longer invisible.", "You fade into existence." (entering combat), typing `vis`, or dying. After a fight it re-invises you.
- At login, reload and `hsp refresh` the plugin checks with `aflags`, the only way to ask the game. That output is hidden; a manual `aflags` still shows normally.
- `vis` always marks it inactive, and pauses re-invising (and shadow form) until you're invisible again by any means, so the plugin doesn't undo your `vis`.
- On automatically when GMCP reports you're a Sprite, off for any other race. `hsp spriteinvis off` as a Sprite is remembered.
- With it off, invis falls back to the normal spellup path and is recast as the spell (if practised).

#### Desecration
- Never cast by the server `spellup`, whether its toggle is on or off. Every server spellup the plugin sends is wrapped in `forgetspell desecration`. The plugin reads the game's reply to learn whether you'd already ignored it, and puts your setting back afterwards. Those replies are hidden while it happens.
- Recast as soon as it drops, mid-combat included. Not while resting or sleeping.

#### Special Affects Tracker window: `hsp miniwin on`

| Column | Shows |
|---|---|
| Name | Each tracked skill, plus Sanctuary. Default list: skills that fit your current class and race. |
| Status | **active** (green; for sanctuary the aura counts), **inactive** (red), or the cooldown **time left** in yellow, e.g. `1h23m5s` |
| Mode | **auto** / **autoaura** (cyan) when hsp is casting it right now, **manual** (dark cyan) otherwise |

- Right-click anywhere: show/hide any skill, name colour per skill, font (picker, quick size, quick font), reset list, reset colours, hide.
- Drag the title bar to move, the bottom-right grip to resize. Respects the Aardwolf client's layout lock.
- Position, size, font, colours and list all survive reloads and restarts.

#### Sanctuary + aura management
Requires `hsp autoaura on` with an aura item set.

- The aura goes on **the moment sanctuary drops**, in or out of combat.
- When you can cast again, the plugin puts your normal float item back on and recasts sanctuary. If the recast fails for any reason, the aura goes straight back on.
- Handles nocast rooms: it waits for you to change rooms instead of retrying and briefly taking your cover off.
- Uses the sanctuary fall-off message as a backup to GMCP.
- Does nothing if sanctuary isn't learned.

#### Class checks
- Totem (Shaman), elemental focus/ward (Elementalist), Gaia's focus (Ranger), prophecy and eye (Oracle), wraith form (Sorcerer/Elementalist/Enchanter), quickstab/stealth/stalk (Ninja) and desecration (Harmer) can't be enabled on the wrong class.
- On login, the plugin turns off any class-specific setting left over from another class.

### Modified

| Area | Before (1.17) | Now (1.22) |
|---|---|---|
| Recasting after death | Plugin could be left thinking you were asleep, so nothing recast while you stood idle. | A short hold that ends as soon as GMCP reports your real state. |
| "Can't cast while resting" spam | Every queued spell fired into a rejection. | The first rejection pauses casting until your state catches up. |
| Mud-side batch spellup | Cleared the whole queue on the assumption the server cast everything, so skills like quickstab were silently lost. | Skills are sent one by one first, and the queue is only cleared as each spell actually lands. |
| Nocast room / resting / out of moves | The failed spell was dropped and never recast. | The spell stays queued and is recast when possible. |
| "Already affected" rejections | Ignored, so the plugin kept retrying buffs you already had (common after relogging). | Treated as confirmation that the buff is up. |
| Stuck affect/recovery tracking | A missed `{affoff}`/`{recoff}` could block a recast forever. | Entries expire on their own timers. Effects with no fixed duration are recorded and never expire. |
| Spells disabled after mana failures | The re-enable path couldn't be reached, so spells stayed off permanently, even across sessions. | Re-enables once mana is back above your threshold. |
| `vis` / `land` while hsp is disabled | The command was swallowed and never reached the game. | Sent as normal. |
| Loading before you're logged in | Setup could stop part-way when GMCP had no data yet. | Loads cleanly whenever it starts. |
| slist capture | Its output-hiding triggers were never reset, so an interrupted capture could hide everything on later starts. | Forced off on load, connect and disconnect. |
| `hsp disable check` | Only reported the main toggle. | Also reports when spell casting is suppressed, and why. |
| Debug output | `We are attempting to cast_spells` printed on every combat round. | `cast_spells called, N queued`. |

### Removed

No user-facing features were removed. The code taken out was 1.17 logic that the fixes above replaced:

- The aura swap-back that took the aura off without checking that sanctuary was back.
- The hardcoded `currentState = 11` on death.
- The queue wipe after a mud-side spellup.
- The unreachable `failed >= 5` / `died2` re-enable conditions.
