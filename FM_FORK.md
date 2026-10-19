# FM fork of CoD2rev_Server — `fm-sniper` branch

This branch runs existing **zk_libcod mods unchanged** on CoD2rev. It is the
exact source of the `<FM>Bots` sniper server (bots, MySQL accounts and stats,
JSON stats menu), which moved from CoD2 1.3 + zk_libcod to CoD2rev without
editing a single line of GSC.

Base: [theCrow007/CoD2rev_Server](https://github.com/theCrow007/CoD2rev_Server),
branch `test` (Crow's zk_libcod port plus his `SV_UserinfoChanged` experiment).
Ten commits on top, each independent.

## What this branch changes

| Commit | Problem on rev | Fix |
|---|---|---|
| MySQL voron: integer result handles | `mysql_initialize()` / `mysql_store_result()` return a raw pointer type. Mods written for zk test `handle != 0` / `result == 0`, which is a script error ("unmatching types") on rev, so MySQL never initializes. | Results are kept in a C-side table and scripts get a small integer id, as on zk. Works the same on x86 and x64. An invalid handle is a readable script error instead of a crash. |
| JSON module | No `json_*` builtins on rev. | `gsc_json.cpp` ported from zk_libcod with `yyjson` (vendored in `src/libcod/yyjson/`). Adds `json_parse`, `json_stringify`, `json_load`, `json_save` and their async variants, plus the `scr_json_*` dvars. Behind `LIBCOD_COMPILE_JSON`. |
| Makefile: link Speex | `gsc_zk_sound.cpp` is built by default (`COMPILE_CUSTOM_VOICE`) but `-lspeex` was missing, so the link failed. | `-lspeex` added to the Linux libraries. |
| `getEntByNum` outside `/# #/` | rev keeps the stock developer-only flag; zk_libcod removes it on 1.3, so mods use the return value directly (compile error on rev). | Registered as a normal builtin in libcod builds. Stock behavior kept without `LIBCOD`. |
| Weapons: 1-based index | `isValidWeaponId()` assumed 0-based ids: the **last** loaded weapon was rejected and index 0 (`none`, also what an unknown name resolves to) was accepted. | Range is `1..BG_GetNumWeapons()`, as in the engine and zk. An unknown weapon returns `undefined` with a developer print instead of a fatal script error, as on zk. |
| `getStance` registered once | `getstance` (`"duck"`/`"lie"`) and `getStance` (`"crouch"`/`"prone"`) were both registered; lookup is case-insensitive and stops at the first match, so the zk-compatible values were unreachable. | Only the zk / CoD2x variant is registered. |
| Bot input reset and `switchToWeapon` | Bot buttons, movement and requested weapon were never cleared, so a bot taking over a slot inherited a held reload or fire button and could not raise its scope (bots replaced for a dead scope). `switchToWeapon()` sent a server command that a bot never reads. | Bot input is cleared on connect with the rest of the per-client state. For a bot, `switchToWeapon()` sets the weapon its usercmd asks for, as on zk. |
| Keep every queued server command | The engine replaced a server command not yet sent by a newer one of the same kind. Several `executeClientCommand()` calls in one frame all queue a `v activeAction` command, so only the last reached the client: login fields declared with `setu` never became userinfo and the login menu read empty values. | `SV_CanReplaceServerCommand()` returns -1 in libcod builds, as zk_libcod does (an exploit fix it took from IW4x). |
| Send message fragments one per frame | The libcod fast download loop drained every fragment of a large message in one burst, for every client, fast download or not. A gamestate (about 11 KB, 9 packets) left all at once; lines with small buffers (ADSL) lost the tail of the burst every time and the client looped on `dropped gamestate, resending`, never entering the game. | Clients in a fast (non-www) download keep the original drain, so fast download speed is unchanged. Every other client gets the stock pacing: one fragment per frame, timed by its rate. |
| Native stack on a fatal signal | A crash such as `free(): invalid pointer` left a single line in the console, with nothing to say where it happened. | On SIGSEGV, SIGABRT, SIGBUS or SIGILL the server writes its call stack to stderr (the console log), then dies as before (same exit code, core dumps unchanged). Resolve the addresses with `addr2line -f -e <binary> 0x...`: the binary is built `-no-pie` and not stripped. |

## Build

```bash
./build.sh mysql2              # x86, VoroN MySQL variant
./build.sh mysql2 ARCH=x64     # x64
```

Ubuntu packages (x64): `libmysqlclient-dev libspeex-dev libssl-dev`.
For x86 add the `:i386` versions and `g++-multilib`. The two
`libmysqlclient-dev` architectures conflict in apt; rev ships its own MySQL
headers, so for the second architecture the runtime library plus a
`libmysqlclient.so` link is enough.

## Run

```bash
./cod2rev_lnxded_x64 \
    +set dedicated 2 \
    +set fs_homepath "/path/to/serverfiles" \
    +set fs_basepath "/path/to/serverfiles" \
    +set fs_library "/path/to/Library" \
    +set fs_game "yourmod" \
    +set net_port 28960 \
    +exec server.cfg \
    +set protocol 118 \
    +set sv_protocolLegacyMode 0 \
    +map_rotate
```

Notes from the migration:

- `fs_library` must be set on the command line and **not** reset in
  `server.cfg`: manymaps uses it on the first `map` command.
- Relative paths read by scripts (`json_load`, `fopen`...) resolve under
  `fs_homepath/fs_game`. If the 1.3 server ran with the default
  `~/.callofduty2`, link those directories into the new `fs_homepath`.
- `sv_botReconnectMode` is applied on `map_restart` only. To drop bots on a
  map change, as zk does with `sv_botReconnectMode 1`, also set
  `sv_kickbots 1`; otherwise bots stay connected without entering the new map
  (spectators with ping -1).
- Process managers that match `cod2_lnxded` do not see `cod2rev_lnxded_x64`;
  match on `_lnxded` plus the port instead. libcod is compiled in, so there is
  no `libcod2.so` in `/proc/<pid>/maps`.

## Known open point

Crow's `test` branch moved `SV_UserinfoChanged()` out of `SV_UpdateUserinfo_f()`
into the `clientUserinfoChanged()` script method, which zk mods do not call.
Rate, snaps and the server-side name may then not update when a client changes
its userinfo during a game. Not changed here: it is Crow's experiment, to be
discussed with him upstream.
