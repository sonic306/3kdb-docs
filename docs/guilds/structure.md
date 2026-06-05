---
id: structure
title: Guild File Structure
sidebar_position: 2
---

# Guild File Structure

Each guild lives in `common/guilds/<guild>/` and provides a standardized set of files. All guild code runs in named TinTin++ classes so it can be cleanly swapped at runtime with `.gswap`.

## How the template works

The `common/guilds/guild template/` folder is a complete, runnable skeleton. Copy it, rename the folder, and fill in each file. The files are designed to be independent — you can implement one at a time and the guild will still load.

### Step-by-step

1. Copy `common/guilds/guild template/` to `common/guilds/myguild/`
2. In `index.tin`, replace `$guild` (it resolves at load time) — no change needed unless you add optional files
3. Fill in each file below, starting with `aliases.tin` and `strategy.tin`
4. Register aliases in the library with `.add_library` calls inside `#if {"$guild" == "myguild"}` guards
5. Set `#var guild myguild` in your character file and reload

---

## `index.tin` — the entry point

`common/index.tin` reads exactly one file per guild:

```tintin
#read common/guilds/$guild/index.tin
```

The `index.tin` in the template then reads all the other files:

```tintin
#read common/guilds/$guild/aliases.tin;
#read common/guilds/$guild/actions.tin;
#read common/guilds/$guild/gags.tin;
#read common/guilds/$guild/heartbeat.tin;
#read common/guilds/$guild/miphp.tin;
#read common/guilds/$guild/tickers.tin;
```

**What `strategy.tin` is not listed here:** strategy.tin is read separately by the common strategy system, not by index.tin. If your guild has a strategy file, add it to `index.tin` yourself:

```tintin
#read common/guilds/$guild/strategy.tin;
```

The order matters — `aliases.tin` before `heartbeat.tin` so the heartbeat can call aliases you define.

---

## `aliases.tin` — class: `guild_aliases`

Guild-specific commands. The class wrapper is mandatory:

```tintin
#class {guild_aliases} {kill}
#class {guild_aliases} {open}

#alias {myspell} {cast fireball};
#alias {mybuff %1} {cast giant strength on %1};

#class {guild_aliases} {close}
```

**Why `kill` before `open`:** The `kill` line destroys any previously-loaded `guild_aliases` class before loading the new one. This is what makes `.gswap` work — the old guild's aliases are killed first. Never skip the `kill` line.

**Registering in the library:** Wrap your `.add_library` calls in a guild check so they only show up for your guild:

```tintin
#if {"$guild" == "myguild"} {
    .add_library {myspell} {guild} {cast fireball} {myspell} {common/guilds/myguild/aliases.tin};
};
```

---

## `actions.tin` — class: `guild_actions`

Trigger patterns that match MUD output. Typical uses:
- Detect when a guild proc fires (damage, heal, effect)
- Parse HP/resource output for guild-specific resources
- Respond to mob-specific messages

```tintin
#class {guild_actions} {kill}
#class {guild_actions} {open}

#action {^Your fireball scorches %* for %d damage!} {
    #math stats[fireball_dmg] {$stats[fireball_dmg] + %2};
};

#class {guild_actions} {close}
```

Actions defined here are killed and reloaded by `.gswap`, so they won't bleed into other guilds.

---

## `gags.tin` — class: `guild_gags`

Suppress guild-specific spam. Each `#gag` pattern matches one line of MUD output and suppresses it:

```tintin
#class {guild_gags} {kill}
#class {guild_gags} {open}

#gag {^Your spell fizzles harmlessly.$}
#gag {^You begin to chant...$}

#class {guild_gags} {close}
```

Keep gags as specific as possible — overly broad patterns will suppress output you want to see.

---

## `heartbeat.tin` — class: `guild_heartbeat`

The per-round hook. The common strategy system calls `_guild_heartbeat` every round. This is where you maintain buffs, manage resources, and make decisions that don't fit neatly into the action queue.

The template gives you the minimum viable heartbeat:

```tintin
#class {guild_heartbeat} {kill}
#class {guild_heartbeat} {open}

#alias {_guild_heartbeat} {
    #if !$idle_flag {
        #NOP;
    };
};

#class {guild_heartbeat} {close}
```

**`$idle_flag`:** Set to 1 when the bot is paused. Always guard your heartbeat logic with `#if !$idle_flag` so you don't take actions when the bot is stopped.

**Per-round vs. any-tick logic:** The heartbeat fires every tick, not just once per round. To only do something once per round, gate on `$action_round < $mip[round]`:

```tintin
#alias {_guild_heartbeat} {
    #if !$idle_flag {
        #NOP -- runs every tick;
        #if {$defs[blink] && !$my[prots][blink]} {cast blink};

        #NOP -- runs once per round;
        #if {$action_round < $mip[round] && $mip[round] > 0} {
            add_action {{{name} {my_nuke} {action} {cast fireball} {priority} {5}}};
        };
    };
};
```

**Common patterns in production guilds:**

| Pattern | Example |
|---|---|
| Auto-recast a defence | `#if {$defs[mb] && !$my[prots][MB]} {cast mindblank}` |
| Use ability once per fight | Set a flag on round 1 reset, check before using |
| Manage a cooldown | Set a toggle var, use `#delay` to reset it |
| Pause bot on resource | Call `.pause`, start a `#ticker` to monitor recovery, call `.unpause` when safe |

---

## `miphp.tin` — class: `guild_miphp`

Configures the MIP HP bar display for this guild's resources. The template exposes all four bars (HP, SP, GP1, GP2) and the enemy health:

```tintin
#class {guild_miphp} kill;
#class {guild_miphp} open;

#var guild_miphp 1;

#alias {_guild_miphp} {
    #list temp clear;
    #list temp add {HP:$hp/$hp_max SP:$sp/$sp_max GP1:$gp1/$gp1_max GP2:$gp2/$gp2_max E:$ehealth};
    #list temp add {H:$my[hp][graph] S:$my[sp][graph] GP1:$my[gp1][graph] GP2:$my[gp2][graph] E:$enemy[graph]};
    #var hpbar[prompt] $temp;
};

#class {guild_miphp} close;
```

Most guilds only use two or three of the four resource bars. Remove the ones that don't apply. For guilds with a custom hpbar command (like gentech's `genset hpbar`), implement that in a `hpbar.tin` file and call `setup_3kdb_hpbar` from your character file.

---

## `strategy.tin` — class: `guild_strategy`

Defines what combat actions the guild takes each round. The strategy system (common) calls `_strategy_guild_start` on round 1 and `_strategy_guild` every subsequent round.

The template pattern:

```tintin
#class {guild_strategy} {kill};
#class {guild_strategy} {open};

#NOP -- Register your strategies in strategyList;
#list strategyList add {{{guild} {myguild} {name} {fireball} {active} {0} {description} {MyGuild: Cast fireball every round}}};
#list strategyList indexate name;

#NOP -- Actions to perform at any time in the fight;
#alias {_strategy_guild} {
    #if {$enemy[hp] > 0} {
        checkStrategy {fireball} {
            #if {$my[sp][current] > 200} {cast fireball};
        };
    };
};

#NOP -- Actions to perform the first rounds of combat;
#alias {_strategy_guild_start} {
    checkStrategy {debuff} {cast slow};
};

#class {guild_strategy} {close};
```

**`checkStrategy {name} {code}`:** Runs `{code}` only if the strategy named `{name}` is currently active. The player toggles strategies with `strategy <name>`. This lets the same script handle multiple play styles without code changes.

**Registering a strategy:** The `#list strategyList add` line makes the strategy visible in `strategy list` and toggleable. Fields:
- `guild` — which guild owns it (informational)
- `name` — the key used in `checkStrategy`
- `active` — 0 = off by default, 1 = on by default
- `description` — shown in `strategy list`

**`_strategy_guild_start`:** For openers that should only happen on round 1 — debuffs, first-round offensive cooldowns, opening moves. The common system calls this once at the start of each fight.

---

## `tickers.tin` — class: `guild_tickers`

Repeating timers using `#ticker`. Empty in the template:

```tintin
#class {guild_tickers} {kill}
#class {guild_tickers} {open}

#class {guild_tickers} {close}
```

Common uses:

```tintin
#ticker {mana_check} {
    #if {$my[sp][current] < 200} {
        #echo {<dcc>Warning: SP low<088>};
    };
} {10};
```

Tickers run whether or not you are in combat. Use `$idle_flag` and `$incombat` guards if you only want them active in certain states.

---

## Optional files

| File | Class | Purpose |
|---|---|---|
| `vars.tin` | `guild_vars` | Default variables, data tables, costs. Loaded once at guild load. |
| `eval.tin` | `guild_eval` | Parse `evaluate` / `appraise` output and build tables. Used by bard for resistance grids. |
| `hpbar.tin` | — | Send hpbar setup commands for guilds with their own hpbar system (e.g. `setup_3kdb_hpbar`). |
| `data.tin` | `guild_data` | Static reference data — spell lists, power tables, cost tables. |
| `summoner.tin` | `guild_summoner` | Summon management logic (mage). |
| `pstats.tin` | — | Performance stat tracking. |

---

## Class names and `.gswap`

Every guild class follows the naming convention `guild_*`. The `.gswap <guild>` command kills all these classes before loading the new guild, which is how guild swapping works cleanly at runtime:

| Class | Description |
|---|---|
| `guild_actions` | Trigger patterns |
| `guild_aliases` | Commands |
| `guild_data` | Static data |
| `guild_eval` | Evaluate parsers |
| `guild_gags` | Spam filters |
| `guild_heartbeat` | Per-round logic |
| `guild_hpbar` | HP bar setup |
| `guild_mage_defense_tracker` | Mage-specific defense tracker |
| `guild_miphp` | MIP bar configuration |
| `guild_summoner` | Summon management |
| `guild_tickers` | Repeating timers |
| `guild_vars` | Default variables |
| `guild_strategy` | Combat strategy |

---

## Creating a new guild: checklist

1. Copy `common/guilds/guild template/` → `common/guilds/myguild/`
2. Add `#read common/guilds/$guild/strategy.tin;` to `index.tin` if you want strategies
3. Add optional files (`vars.tin`, `eval.tin`, etc.) to `index.tin` as needed
4. Write `aliases.tin` — start here, it's the most immediately useful
5. Write `strategy.tin` — register at least one strategy even if empty
6. Write `heartbeat.tin` — add resource maintenance and per-round logic
7. Write `miphp.tin` — trim the template to show only the resource bars your guild uses
8. Register aliases in `library3kdb` with `.add_library` inside guild guards
9. Set `#var guild myguild` in your character file and reload
