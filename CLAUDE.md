# CLAUDE.md

Astonia 3 game server, Community Edition. C code from 1997–2008 (Daniel Brockhaus / Intent Software), built as **32-bit** (`-m32`) Linux binaries against MySQL/MariaDB. `readme.md` has the Ubuntu setup steps, `server.md` covers config options and `DOCKER.md` covers the container setup.

## Build

- `make` (or `make -j4`) builds everything. Needs `gcc-multilib`, `lib32z-dev`, `libargon2-dev:i386`, `libmysqlclient-dev:i386`.
- `CFLAGS` include `-Wall -Wshadow -Werror`, so **any warning breaks the build**. `-DSTAFF` is always set (see "STAFF mode" below).
- Objects go to `.obj/`. `make clean` removes the objects, binaries and `runtime/*/*`.
- `make pretty` / `make pretty-check` run clang-format **21** over `*.c *.h` (see `.clang-format`). CI (`.github/workflows/build-and-style.yml`) runs `pretty-check`, then `make`, on Ubuntu 22.04. Run `make pretty` before committing.
- The Makefile lists header dependencies by hand. When you add an `#include`, also add it to that object's dependency line.

What `make` produces:
| Output | Source |
|---|---|
| `server` | core objects in `OBJS` (linked with `-rdynamic` so DLLs can call back into it) |
| `runtime/generic/*.dll`, `runtime/<area>/*.dll` | one `.c` per DLL, built `-fPIC -shared` (the `.dll` name is just convention; they are ELF shared objects) |
| `chatserver` | `chatserver.c` |
| `create_account`, `create_character` | admin tools (link `config.o`, plus `argon.o` for accounts) |
| `create_weapons`, `create_armor` | generators, run by make to write `zones/generic/weapons.itm` and `armor.itm` (both gitignored) |

`dungeon_tab.c` and `ice_shared.c` are not compiled on their own. They are `#include`d (by `dungeon.c`, and by `ice.c` + `palace.c`).

## Runtime architecture

- **One `server` process per area (and mirror).** `./server -a <areaID> [-m <mirror>] [-i <serverID>] [-d] [-c]`; area, mirror and serverID all default to 1. `-d` daemonizes and `-c` disables concurrent DB access. Startup order is in `server.c:main`, and per-area `maxchars` limits are hardcoded in a switch there.
- Each area server binds the **first free TCP port in 5556–5599** (`io.c:init_io`), so the port depends on start order. It writes its address and port to the `area` table (`database.c:area_alive`). When a character moves to another area, the client is redirected to the server and port read from that table (`player.c`, `database.c:get_area`/`change_area`).
- **chatserver** listens on **5554** (hardcoded in `chatserver.c` and `chat.c`). It relays chat/tell traffic between area servers. Area servers connect to `chathost` (default `localhost`). Start it before the area servers.
- **Area content** comes from:
  - `zones/generic/` plus `zones/<areaID>/` (`.map`, `.chr`, `.itm`, `.pre` files, loaded by `create.c`)
  - `runtime/generic/*.dll` plus `runtime/<areaID>/*.dll` (NPC/item "drivers", loaded by `libload.c`, which `dlsym`s each one's `driver` function)
- `libload.c:load_lib` re-checks each DLL's mtime on every driver call and reloads it if it changed. That means **rebuilding a DLL hot-swaps it into running servers**. The `.tmp` + `mv` steps in the Makefile rules exist for this, so keep that pattern.
- Logs go to the working directory as `YY_MM_DD.log` / `.ilog` / `.elog` (`log.c`).

### start / stop
- `./start` kills any running `chatserver`/`server`, starts `chatserver`, then runs `./server -a N -d` for **only areas 1, 2, 3, 14, 33**. The comment says to add the others yourself for a full server.
- `./stop` runs `pkill chatserver; pkill server`. Note that `pkill server` also matches any other process whose name contains "server".
- Docker (`docker-entrypoint.sh`) starts areas `1 2 3 5 6 8 10 11 13 14 15 16 17 18 19 20 22 23 24 25 26 28 29 31 32 33 34 35 36 37` in the foreground with `-e`.

### Area IDs → DLLs (from Makefile) and names (from `area` rows in `merc.sql`)
1 Cameron (gwendylon, shrike) · 2 Deep Below Aston (area2) · 3 Aston (area3, arena, gatekeeper, military) · 4 Pentagram Quest · 5 Sewers (sewers) · 6 Earth Underground (edemon) · 7/9 Fire/Ice Pents · 8 Fire Underground (fdemon) · 10 Ice Underground (ice) · 11 Ice Palace (palace) · 12 Mines · 13 Catacombs (dungeon) · 14 Random Dungeon (random) · 15 Swamp (swamp) · 16 Forest (forest) · 17 Exkordon (two) · 18 Bones and Towers (bones) · 19 Nomad Plains (nomad, saltmine) · 20 Live Quest (lq) · 21 Test Pents · 22 Labyrinth (lab1–lab5) · 23 Ice Army Caves (strategy) · 24 More Ice Army Caves · 25 Warped World (warped) · 26 Below Aston 2 (staffer) · 28 Brannington Forest (staffer3) · 29 Brannington (staffer2) · 30 Clan Spawners · 31 Grimroot (warrmines) · 32 Jobbington (missions) · 33 Long Tunnel (tunnel) · 34 Teufelheim (teufel) · 36 Caligar (caligar) · 37 Arkhata (arkhata).
Generic DLLs that every area loads: base, lostcon, merchant, simple_baddy, bank, book, transport, clanmaster, clubmaster, pents, alchemy, mine, professor, sidestory.

## Source map (core `server` binary)

| Area | Files |
|---|---|
| Entry, main loop | `server.c` (+ `server.h`, the big shared header with `ch[]`/`it[]`/map structs and flags) |
| Config | `config.c` (defaults, `-s name=value`, `-f file`, `-e` env vars `AS3_DBHOST/DBUSER/DBPASS/DBNAME/CHATHOST/SVRKEY`). The order of options matters: later ones override earlier ones. |
| Networking, client protocol | `io.c` (sockets), `player.c` (login packet, `decrypt()`, client commands, sends the server key), `player_driver.c`, `client.h` |
| Database | `database.c` (all game-server SQL), `lookup.c` (name↔ID cache), `badip.c`, `argon.c` (argon2id hash/verify) |
| World, simulation | `create.c` (zone file loading, char/item creation), `map.c`, `sector.c`, `los.c`, `light.c`, `path.c`, `see.c`, `notify.c`, `timer.c`, `effect.c`, `expire.c`, `container.c`, `mem.c` |
| Actions, combat | `do.c` (action requests), `act.c` (action execution), `death.c`, `skill.c`, `poison.c`, `respawn.c` |
| Driver support | `libload.c` (DLL loading/dispatch), `drvlib.c` (helper lib for NPC drivers), `drdata.c` (per-char persistent data blobs, the "ppd"), `tool.c` |
| Chat, social | `chat.c` (link to chatserver), `talk.c`, `tell.c`, `ignore.c`, `swear.c`, `mail.c`, `motd.c` |
| Commands | `command.c` (`/` commands, including god/staff ones) |
| Clans, clubs, economy | `clan.c`, `clanlog.c`, `club.c`, `store.c`, `depot.c`, `lab.c`, `prof.c`, `questlog.c`, `teufel_pk.c` |
| Admin, misc | `punish.c`, `task.c` (cross-server task queue via `task` table), `area.c`, `consistency.c`, `statistics.c`, `date.c`, `btrace.c`, `error.c`, `log.c`, `sleep.c`, `rdtsc.S` |

Every other `.c` is an NPC/area driver compiled into a DLL (see the table above). The `*_ppd.h` headers define per-player data blocks stored in `chars.ppd`.

## Database

- **Schema:** `create_tables.sql`. It runs `CREATE DATABASE IF NOT EXISTS merc; USE merc;`, so the database name `merc` is hardcoded in it. It is an old MySQL 3.23 dump.
- **Seed data:**
  - `merc.sql`: 43 `area` rows, 1 `constants` row (`unique`), and **subscriber ID 1 + character ID 1 "Ishtar"** (a god char)
  - `storage.sql`: 38 `storage` blobs (clan data is storage ID 1, plus arena toplists, etc.)
- **Main tables:**
  - `subscriber` (accounts; `password` holds an argon2id hash)
  - `chars` (character state as binary blobs `chr`, `item`, `ppd`, plus `current_area`/`current_mirror` to lock a char to one server)
  - `charinfo` (a denormalized copy for lookups)
  - `area` (live server registry)
  - `storage`, `task`, `notes`, `clanlog`, `iplog`/`ipban`/`badip`, `constants`
- **Threading rule** (header of `database.c`): DB queries run on a separate DB thread through a query queue (`add_query`). Never make mysql calls from the main server thread.
- On startup each server resets `chars.current_area` for its own area/mirror and zeroes its `area` row.
- All SQL is built with `sprintf`. Strings are escaped with `mysql_real_escape_string`, and blobs are zlib-compressed (`compress_escape_string`).

## Accounts and characters

- `./create_account [-s k=v|-f file|-e] <email> <password>` inserts into `subscriber` with an argon2id hash (no pepper) and prints the new account ID. Passwords are at most **15 chars**, because the client sends only `MAXPASSWORD-1` bytes.
- `./create_character [-s k=v|-f file|-e] <accountID> <name> <GC[G]>` inserts into `chars` and `charinfo`. Arguments:
  - name: letters only, 2–38 chars
  - `G`: `M`/`F` (gender)
  - `C`: `W`/`M` (warrior/mage)
  - optional third letter `G`: god flag
- Login (`player.c:read_login` → `database.c:find_login`/`load_char`/`load_char_pwd`):
  1. The client password is XOR-"decrypted" with a static table in `player.c:decrypt`.
  2. It is checked against the argon2 hash.
  3. Then the server checks locked/banned/IP bans and paid time.

## Hardcoded credentials and secrets

- **`config.c`**: default `dbuser=root`, `dbpass="tgbdwf3h"`, `svrkey="4241"`. Every binary uses these unless overridden by `-s`/`-f`/`-e`.
- **`my`** (shell helper): `mysql -hlocalhost -uroot -ptgbdwf3h ...`. The same root password ends up in the process list. `readme.md` calls this script a security risk.
- **`MYSQLPASSWD`**: contains the same password. It is deleted in the working tree (staged), but it is still in git history, and `readme.md` still tells users to `cat` it.
- **`docker-compose.yml` / `docker-entrypoint.sh`**: default DB root password `astonia` (`MYSQL_ROOT_PASSWORD`, `AS3_DBPASS`) unless the env var is set. The server connects as `root`.
- **`player.c:decrypt`**: static XOR key table for the login password. It is obfuscation, not encryption: passwords are effectively cleartext on the wire.
- **`.serverkey`** (gitignored, optional): read by `server.c` with the config parser, so its format is presumably `svrkey=<n>`. It overrides the default key 4241. The key is sent to clients (`player.c`).
- **`merc.sql`**: seeds subscriber 1 with an argon2 hash of an unknown password and **real-looking personal data** (name/address). Treat this as PII.
- `server.md` example uses `dbpass="top secret"`. This is documentation only.

## STAFF mode (`-DSTAFF`, always on in the Makefile)
- `load_char_pwd` treats every account as paid.
- `-d` no longer sets `nologin`.
- Area 27 blocks at startup until a file `zones/27/start` or `zones/27/start.txt` exists, and deletes it (`motd.c:check_staff_start`).

## Things I'm unsure about / inconsistencies noticed

- **Advertised server IP is hardcoded**: `io.c` sets `server_addr = 212.202.240.67 + serverID - 1` (the `get_interface2` call is commented out). That value is stored in `area.server` and used for cross-area redirects. I don't know how the community client handles this. It may ignore the IP and reuse the host it connected to. Check this before relying on area transfers on another host.
- **Firewall ports in `readme.md`** (8080–8090, 27584–27777) don't match the ports the code binds (5556–5599 for areas, 5554 for chat). The Docker setup uses the code's ports.
- **The readme quick-start may collide with the seed data**: `merc.sql` already creates subscriber 1 and character "Ishtar" (sID 1). The readme's `./create_account …` would then get ID 2, and `./create_character 1 Ishtar MWG` should fail with "name already taken". I have not run this to confirm.
- `create_character` writes the 8-byte class flag into both the `chr` and `item` blobs, plus a junk ppd block. I assume the server builds the real character on first login, but I didn't trace that path.
- The Docker entrypoint loads `create_tables.sql` + `merc.sql` but **not `storage.sql`**, while the readme loads all three. I don't know if the missing storage rows matter. The server can create storage entries itself (`create_storage`).
- The Docker area list includes 24 and 35. Neither has DLLs in the Makefile, and 35 has no `area` row in `merc.sql`. Areas 4, 7, 9, 12, 21, 27, 30 have zone dirs and/or area rows but are not started by Docker or `start`. Which areas are meant to run in a "full" server is unclear.
- `.gitignore` mentions `update_server` and `.dockerignore` mentions `accman/`. Neither exists in the repo.
- I did not verify the exact zone file formats (`.chr`, `.itm`, `.map`, `.pre`) or the client protocol beyond the login path.
