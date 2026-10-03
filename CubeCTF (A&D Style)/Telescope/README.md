# Defending "telescope" (TelescopeNet) - A&D CTF Writeup

A beginner-friendly writeup of how I defended the **telescope** service in an Attack & Defense (A&D) CTF. It was the most-attacked service, so I focused on it.

## The situation

- **Mode:** Attack & Defense. My job was to **patch** the service so attackers could not steal my flag.
- **Service:** `telescope`, a small text-based server written in C++ (`src/main.cpp`, about 17 KB). It listens on port `7419`.
- **Where the flag lives:** inside a user's **note** (the `STORE_NOTE` command). So the goal was simple: *stop one user from reading another user's notes.*
- **Limits of the console:**
  - Only a web console. No copy-paste, no file transfer, no scrolling, no inspect.
  - I had to read code a few lines at a time with `sed -n 'START,ENDp'`.
  - Everything I read was passed through OCR, so some characters were misread (for example `lsqllite3` instead of `lsqlite3`). I always double-checked important lines.

## My method (simple and repeatable)

1. **Back up first.** `cp -r /services/telescope /services/telescope_backup`
2. **Read the manual** (`README.md`) to see which commands exist and guess which ones touch other users' data.
3. **Search, don't scroll.** Use `grep -n` to jump to the interesting lines instead of reading 500 lines.
4. **Read small pieces** of code with `sed -n`.
5. **Patch one thing at a time** with `sed -i`, then **print the line again** to confirm the patch worked.
6. **Rebuild only once** at the end (the data lives in `tmpfs`, so every restart wipes users and notes).
7. **Test** with two fake users: one stores a note, the other tries to steal it.

Useful commands:

```bash
grep -n "CAL_PREFETCH\|JOB_RESULT\|QUEUE_AUDIT" src/main.cpp   # find where commands live
grep -n 'cmd == "' src/main.cpp                                 # list EVERY command the server knows
sed -n '381,394p' src/main.cpp                                  # read a small part
```

> **Lesson:** the second `grep` found two commands (`RESET_PREP`, `RESET_FINISH`) that were **not in the README**. Never trust the manual alone. Always list the real commands from the code.

## The vulnerabilities

The commands I checked and what I found:

| # | Command(s) | Problem | Result for the attacker |
|---|-----------|---------|-------------------------|
| 1 | `CAL_OPEN` + `CAL_PREFETCH` + `CAL_COMMIT` | `CAL_PREFETCH <user>` copies **any** user's latest note | Flag shows up in `OK committed <memo>` |
| 2 | `QUEUE_AUDIT <target>` | Puts **any** user's latest note into the job result | Flag shows up in the job result |
| 3 | `JOB_RESULT <id>` | Returns a job by ID only, never checks who owns it | Anyone who gets or guesses a job ID can read it |
| 4 | `RESET_PREP` + `RESET_FINISH` | Password reset needs **no login**, and the token can be calculated by anyone | Attacker takes over the victim's account, logs in, reads the flag |
| 5 | `LOGIN` (`password_acceptable`) | Also accepts **old** password hashes, not just the current one | Hardening fix (was not exploitable at that moment) |

### 1. Calibration chain (`CAL_PREFETCH`)

In simple words, the attack was:

1. Log in as any user.
2. `CAL_OPEN x` (creates a buffer).
3. `CAL_PREFETCH <victim>` (the code copies the victim's note into your buffer, without asking if you are the victim).
4. `CAL_COMMIT` (the server prints the buffer text back to you).

The bad line (line 387):

```cpp
std::string body = first_note_for(args[1]);
```

**Patch:** only use the note if the name is **you**. For any other name, use an empty text. The command still answers `OK prefetched`, so a health checker that calls it will still see the service as "up".

```cpp
std::string body = (args[1] == sess.user) ? first_note_for(args[1]) : std::string();
```

Patch command:

```bash
sed -i '387s/first_note_for(args\[1\]);/(args[1] == sess.user) ? first_note_for(args[1]) : std::string();/' src/main.cpp
```

### 2. `QUEUE_AUDIT` (line 349)

Same mistake. It builds the job result from `first_note_for(args[1])`, where `args[1]` is any target the attacker types.

```cpp
// before
std::string note = first_note_for(args[1]);
// after
std::string note = (args[1] == sess.user) ? first_note_for(args[1]) : std::string();
```

```bash
sed -i '349s/first_note_for(args\[1\]);/(args[1] == sess.user) ? first_note_for(args[1]) : std::string();/' src/main.cpp
```

### 3. `JOB_RESULT` (line 358)

The jobs table already saves who asked for each job (`requester`), but the query never used it.

```cpp
// before
scalar("SELECT result FROM jobs WHERE id=?", {args[1]});
// after
scalar("SELECT result FROM jobs WHERE id=? AND requester=?", {args[1], sess.user});
```

```bash
sed -i '358s/WHERE id=?", {args\[1\]}/WHERE id=? AND requester=?", {args[1], sess.user}/' src/main.cpp
```

Before using two values, I checked that `scalar()` takes a `std::vector<std::string>` (a list of any length), so the patch compiles.

### 4. Password reset (`RESET_PREP`, `RESET_FINISH`)

This was the sneaky one, because it **skips all the other protections**.

- Neither command needs a login.
- `RESET_PREP <user>` even replies with the victim's `home` value and the time bucket.
- The reset "token" is `fnv1a(user:home:time:seed)`. FNV-1a is a plain hash with **no secret key**. Every ingredient is public: the username, the leaked `home` and time, and the `seed` (printed by `BANNER`). So the attacker can calculate the token, set a new password, log in as the victim and just use `GET_NOTE`.

**Patch:** switch both commands off by returning an error as the very first thing.

```bash
sed -i -e '225s/$/ return "ERR reset";/' -e '235s/$/ return "ERR reset";/' src/main.cpp
```

Result:

```cpp
if (cmd == "RESET_PREP") { return "ERR reset";
if (cmd == "RESET_FINISH") { return "ERR reset";
```

The old code below still exists but never runs. C++ allows that.

**Trade-off:** these commands are not in the README, so I assumed the checker does not use them. If the checker did, the service would start failing checks. In that case, the fix is to restore the commands and make the token depend on a **secret** the attacker cannot know (see "Better fixes").

### 5. Old passwords still work (hardening)

`password_acceptable()` accepted the current hash **and** every hash in the `old` list. This was dormant (the list is only filled by `RESET_FINISH`, which I had switched off), but I closed it anyway. I matched the exact text instead of a line number, because OCR had made the line numbers look off by one.

```bash
sed -i 's/if (part == hash) return true;/if (false) return true;/' src/main.cpp
```

## Commands I checked and found safe

| Command | Why it is safe |
|---------|----------------|
| `GET_NOTE` | Query uses `{sess.user, title}`, so it only reads **your** notes |
| `LIST_NOTES` | Binds `sess.user` |
| `LIST_SCOPES` | Binds `sess.user`, and escapes output |
| `ADD_SCOPE`, `STORE_NOTE` | Use `?` placeholders (no SQL injection) and save under `sess.user` |
| `REGISTER` | Uses placeholders and `valid_name()` |
| `CONSTELLATIONS`, `EPHEMERIS` | Only do math or use fixed lists. No database or files |

## Rebuild and restart

The Dockerfile compiles the code, so the patch only takes effect after a rebuild. I **built first** (`docker compose build`) so that if it failed, the old service would keep running. Then I restarted.

```bash
docker compose build 2>&1 | tail -n 8     # look for: Image telescope-telescope Built
docker compose up -d 2>&1 | tail -n 5
docker compose ps                          # status should say Up
```

Note: `/data` is `tmpfs`, so a restart wipes all users and notes. That is fine in this game because fresh flags are planted each round, but it is why I rebuilt only once.

## Testing the fix

I used two fake users. (`FLAG_TEST` is a fake flag.) On this machine `nc -q` did not work (it is `ncat`), so I used `-w2`.

```bash
# Victim stores a note
printf 'REGISTER vic1 pw1\nLOGIN vic1 pw1\nADD_SCOPE s1 1 1 100 dob\nSTORE_NOTE s1 secret FLAG_TEST\nQUIT\n' | nc -w2 localhost 7419

# Attacker tries the attacks
printf 'REGISTER att1 pw2\nLOGIN att1 pw2\nQUEUE_AUDIT vic1\nCAL_OPEN x\nCAL_PREFETCH vic1\nCAL_COMMIT\nRESET_PREP vic1\nQUIT\n' | nc -w2 localhost 7419
```

What I got back from the attacker session:

```
OK job 8394a79a5e97af23
OK cal
OK prefetched
OK committed        <- nothing after it, no FLAG_TEST leaked
ERR reset           <- password reset is closed
```

Normal commands (`PING`, `REGISTER`, `LOGIN`, `ADD_SCOPE`, `STORE_NOTE`) still worked, so the checker should still see the service as healthy.

## Honest notes: what I did not verify

- I did **not** test `JOB_RESULT` against a live attacker. I only confirmed that the code compiled.
- The old-password patch (#5) was applied to the source after the last rebuild test, and I did not get to rebuild and test it before the console closed.
- I did **not** read the connection-handling code (`client_thread`, from about line 410). A hidden command or weak input check could still be there.
- I did not check the helper functions `valid_name()` and `tail_after()`.

## Mistakes along the way (so you can avoid them)

- I once patched the wrong line number (388 instead of 387). Nothing changed because the pattern did not match. **Always print the line again after patching** (`sed -n '387p' src/main.cpp`).
- OCR garbled a few characters. When a command depends on exact characters, compare carefully or re-run a small `grep`.

## Better fixes (if I had more time)

1. **Reset tokens:** use a **secret key** the attacker cannot know (for example an HMAC with a random server secret), or require the user to be logged in. Never build tokens from public values.
2. **Password hashing:** passwords are hashed with FNV-1a, which is fast and not meant for security. A real password hash (bcrypt, scrypt, argon2) with a salt would be much better.
3. **Job IDs:** use random IDs instead of a hash of `user:target:counter`, so they cannot be guessed.
4. **One access rule:** make a single helper such as `note_for(requesting_user, owner)` that always checks the owner, so no command can forget the check.

## Key takeaways

- The bugs all had the **same root cause**: the server trusted a username typed by the user and never checked "is this me?".
- Read the code, not just the manual. The most dangerous commands were not in the README.
- Back up first, patch one thing at a time, confirm each patch, and test with two users.
- Prefer patches that keep the command answering normally (like returning an empty result) so the checker does not mark your service as down.
