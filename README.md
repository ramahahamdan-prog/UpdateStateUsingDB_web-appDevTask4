# UpdateStateUsingDB_web-appDevTask4
## Robot Control Panel (PHP + MySQL, hosted on InfinityFree)

## Overview

A minimal web-based remote control for a robot/device. A simple HTML page
with buttons lets a user send movement commands; each button press updates
a single row in a MySQL database, and the robot (or any client polling the
database) reads that row to know the current command to execute. Hosted
entirely on free PHP/MySQL hosting (InfinityFree), no Node.js or persistent
server process required.

## Features

- Single-row "current state" design (`robot_state`, `id = 1`) — always
  reflects the latest command only, no growing history table to manage
- Lightweight PHP endpoints: one to update the command, one to read it
- Simple, dependency-free HTML/JS frontend with on-screen confirmation
  after each button press
- Customizable command encoding via a `$map` array in `update_command.php`
  (e.g. direction names mapped to single-letter codes)
- Runs on standard shared hosting (InfinityFree) with just File Manager/FTP
  upload — no special server access needed

## Technologies Used

| Layer | Technology |
|---|---|
| Frontend | HTML5 + vanilla JavaScript (`fetch`) |
| Backend | PHP (MySQLi/PDO) |
| Database | MySQL (InfinityFree-hosted) |
| Hosting | InfinityFree shared hosting (File Manager / FTP, phpMyAdmin) |

## How It Works

1. **`setup.sql`** creates the `robot_state` table with a single row
   (`id = 1`), initialized to the stop command (`"S"`). This row is never
   inserted into again — only updated — so the table always holds exactly
   one current command.
2. **`db.php`** holds the MySQL connection details (`$host`, `$user`,
   `$pass`, `$dbname`) and opens the database connection used by the other
   two PHP scripts.
3. **`index.html`** renders the control buttons. Each button click sends a
   request (via `fetch`) to `update_command.php` with the chosen command,
   and shows a confirmation message once the server responds.
4. **`update_command.php`** receives the button's command label, looks it
   up in the `$map` array to translate it into the stored code (e.g.
   `"Forward" → "F"`), and runs an `UPDATE robot_state SET command = ... WHERE id = 1`.
5. **`get_state.php`** reads the current value from `robot_state` (`id = 1`)
   and returns it as the response — this is what the robot/device (or any
   external client) polls to find out which command to execute next.

```
Browser (button click) → fetch(POST) → update_command.php → $map lookup → MySQL UPDATE (robot_state, id=1)
                                                                                    ↓
Robot / external client ← current command ← get_state.php ← MySQL SELECT (robot_state, id=1)
```

## What Was Configured / Fixed

Deploying a generic template like this to a specific InfinityFree account
always requires a few concrete adjustments — these are the steps that were
completed to make it functional on your account:

1. **Database created on InfinityFree** via MySQL Databases in the control
   panel, and the generated `Hostname`, `Username`, `Password`, and
   `Database name` were recorded for use in `db.php`.
2. **Table created** by running `setup.sql` through phpMyAdmin's SQL tab —
   this provisions `robot_state` with its single seed row (`id = 1`,
   `command = "S"`).
3. **`db.php` connection values replaced** with the real InfinityFree
   credentials (`sqlXXX.infinityfree.com`, `epiz_XXXXXXXX`, the real
   password, and `epiz_XXXXXXXX_control_db`) instead of the placeholder
   values.
4. **All five files uploaded** (`db.php`, `update_command.php`,
   `get_state.php`, `index.html`, and implicitly `setup.sql` for the DB
   setup step) into the same folder inside `htdocs` via File Manager/FTP,
   since `db.php` is included using a relative path and must sit alongside
   the other PHP scripts.
5. **Verified end-to-end** by opening the live site, pressing a button,
   confirming the on-screen confirmation message, and cross-checking the
   change directly in phpMyAdmin with `SELECT * FROM robot_state;`.

## Improvements

- The command encoding is centralized in a single `$map` array in
  `update_command.php`, so the letter/number scheme (e.g. directions vs.
  capital-letter codes) can be changed in one place without touching the
  frontend or the database schema.
- Using a fixed single-row table (`id = 1`, update-only) instead of an
  append-only log keeps reads trivial for the robot/device — always exactly
  one row to fetch, no need to query "the latest" entry.
- Credentials are isolated in `db.php`, so `index.html` and the frontend
  never see database details.

## Challenges

- **Shared-hosting constraints.** InfinityFree offers no shell/SSH access
  and no long-running processes, so the design had to rely on a
  polling-based database row rather than a persistent socket or webhook —
  the robot/client must poll `get_state.php` on an interval.
- **Credential handling on free hosting.** Unlike a VPS, there's no
  environment-variable support, so connection details must live directly in
  `db.php`; care is needed not to expose this file publicly or commit real
  credentials to a public GitHub repo.
- **Verifying encoding changes safely.** Since `$map` in
  `update_command.php` is the single source of truth for command codes, any
  change to it must stay in sync with whatever the robot/device expects on
  its end — a mismatch would silently send the wrong command with no error.

---

## Deployment Steps (for reference)

1. **Create the database** — InfinityFree control panel → MySQL Databases →
   create a new database, and record the Hostname, Username, Password, and
   Database name.
2. **Create the table** — phpMyAdmin → SQL tab → paste and run the contents
   of `setup.sql` (creates `robot_state` with one row, `id = 1`, initial
   value `"S"`).
3. **Configure the connection** — edit `db.php` with the real values:
   ```php
   $host = "sqlXXX.infinityfree.com";
   $user = "epiz_XXXXXXXX";
   $pass = "your_password_here";
   $dbname = "epiz_XXXXXXXX_control_db";
   ```
4. **Upload the files** — `db.php`, `update_command.php`, `get_state.php`,
   and `index.html` into the same folder inside `htdocs` via File Manager
   or FTP.
5. **Test it** — open the site, click a button, and check for the
   confirmation message. Verify the update landed by running
   `SELECT * FROM robot_state;` in phpMyAdmin.

> To change the command encoding (e.g. all-capital letters, or numbers
> instead of letters), update the `$map` array in `update_command.php`
> accordingly.
