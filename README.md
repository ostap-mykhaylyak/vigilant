# vigilant — Cloudflare auto Under Attack Mode

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PHP](https://img.shields.io/badge/PHP-%3E%3D8.0-777bb4.svg)](https://www.php.net/)
[![Release](https://img.shields.io/badge/release-1.0.0-brightgreen.svg)](CHANGELOG.md)

A small, dependency-free PHP CLI tool that watches your server and **automatically
enables [Cloudflare Under Attack Mode (UAM)](https://developers.cloudflare.com/waf/tools/under-attack-mode/)**
when load or traffic crosses a threshold — then turns it back off when things
calm down. Designed to be run from `cron`, and to behave well on **shared
hosting (cPanel & friends)**.

> **Why?** Under Attack Mode is a great emergency brake during a DDoS or a
> traffic spike, but leaving it always-on hurts UX (everyone gets a JS
> challenge). This tool toggles it only when you actually need it.

---

## Table of contents

- [Features](#features)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Cloudflare API token](#cloudflare-api-token)
- [Configuration](#configuration)
- [Usage](#usage)
- [Scheduling with cron](#scheduling-with-cron)
- [Security model](#security-model)
- [Shared hosting notes](#shared-hosting-notes)
- [Exit codes](#exit-codes)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- **Three trigger modes**
  - `load` — server load average, normalized per CPU core.
  - `requests` — requests/sec to your zone, via the Cloudflare GraphQL
    Analytics API (ideal on shared hosting, where the system load is not yours).
  - `both` — activates if *either* metric is over its threshold; only stands
    down when *both* are back under.
- **Hysteresis + anti-flapping** — separate on/off thresholds plus a minimum
  dwell time keep UAM from toggling every minute.
- **Secure by default** — credentials live in a `chmod 600` config file, never
  on the command line; the script refuses to run on loose permissions.
- **Zero dependencies** — pure PHP, no Composer packages. Uses cURL when
  available and transparently falls back to stream wrappers.
- **Shared-hosting friendly** — no `shell_exec`/`exec`, no writes outside your
  home directory, tolerant of `open_basedir` and `disable_functions`.

## How it works

On each run the script:

1. Loads and validates its config (and checks the config file permissions).
2. Computes one or more metrics depending on `trigger_mode`.
3. Compares them against the on/off thresholds, taking the persisted state and
   the minimum-dwell timer into account.
4. Calls the Cloudflare API to set the zone `security_level` to `under_attack`
   or back to your normal level — only when a change is actually needed.
5. Persists state to a small JSON file and logs the decision.

## Requirements

- PHP **8.0+** (CLI), with the `json` extension.
- A domain proxied through Cloudflare.
- A scoped Cloudflare **API Token** (see below).
- `ext-curl` *or* `allow_url_fopen = On` for outbound HTTPS.
- For `load` mode: a Linux host exposing `/proc/loadavg` (or
  `sys_getloadavg()`).

## Installation

```bash
git clone https://github.com/ostap-mykhaylyak/vigilant.git
cd vigilant
cp config.php.example config.php
chmod 600 config.php          # required — see Security model
$EDITOR config.php            # add your token, zone id and thresholds
```

No `composer install` is needed — the tool has no runtime dependencies.

## Cloudflare API token

Create a **scoped** token (never use the Global API Key) at
**Cloudflare Dashboard → My Profile → API Tokens → Create Token → Create Custom Token**.

Grant the minimum permissions, restricted to the single zone:

| Permission                | Needed for           |
| ------------------------- | -------------------- |
| `Zone → Zone Settings → Edit` | reading/writing `security_level` (all modes) |
| `Zone → Analytics → Read`     | `requests` / `both` modes only |

Copy the generated token into `config.php`. The **Zone ID** is on your domain's
**Overview** page in the dashboard.

## Configuration

All settings live in `config.php` (start from `config.php.example`). Key options:

| Key                  | Default        | Description |
| -------------------- | -------------- | ----------- |
| `api_token`          | —              | Scoped Cloudflare API token. **Required.** |
| `zone_id`            | —              | Cloudflare Zone ID. **Required.** |
| `trigger_mode`       | `load`         | `load`, `requests`, or `both`. |
| `normal_level`       | `medium`       | Level to restore: `off`/`essentially_off`/`low`/`medium`/`high`. |
| `min_uam_seconds`    | `600`          | Minimum time UAM stays on before it may stand down. |
| `cpu_cores`          | `null`         | CPU cores for `load` mode; `null` = auto-detect. |
| `load_index`         | `0`            | Which load average: `0`=1 min, `1`=5 min, `2`=15 min. |
| `threshold_on`       | `1.50`         | Load-per-core that switches UAM **on**. |
| `threshold_off`      | `0.70`         | Load-per-core that switches UAM **off**. |
| `req_threshold_on`   | `50.0`         | Requests/sec that switches UAM **on**. |
| `req_threshold_off`  | `20.0`         | Requests/sec that switches UAM **off**. |
| `req_window_minutes` | `5`            | Averaging window for requests/sec. |
| `state_file`         | `./vigilant-state.json` | Anti-flapping state (must be writable). |
| `log_file`           | `./vigilant.log` | Log file; `null` = stdout/stderr. |
| `http_timeout`       | `15`           | HTTP timeout in seconds. |

### Tuning thresholds

- **`load` mode** thresholds are *per core*, so `1.50` means 150 % of nominal
  capacity regardless of how many cores the box has.
- **`requests` mode**: check your normal peak traffic in Cloudflare Analytics
  and set `req_threshold_on` to roughly **3–5×** that peak so legitimate spikes
  don't trip it.
- Always keep the `off` threshold meaningfully **lower** than the `on`
  threshold — that gap is the hysteresis that prevents flapping.

## Usage

```bash
php vigilant                 # evaluate once and act (cron entry point)
php vigilant --status        # print current metrics + security level
php vigilant --force-on      # manually switch Under Attack Mode on
php vigilant --force-off     # manually restore the normal level
php vigilant --config=/path/to/config.php
php vigilant --help
php vigilant --version
```

Example `--status` output:

```text
Trigger mode              : both
Security level Cloudflare : medium
Load average (1/5/15)     : 0.42 / 0.55 / 0.60
Core rilevati             : 4
Load per core (1-min)     : 0.11  (on>=1.50 / off<=0.70)
Richieste/sec (5 min)     : 12.4  (on>=50.0 / off<=20.0)
```

## Scheduling with cron

Run it every minute:

```cron
# Generic Linux / VPS
* * * * * /usr/bin/php /home/youruser/vigilant/vigilant >/dev/null 2>&1
```

On **cPanel** (Cron Jobs UI), the PHP binary is often versioned — find it with
`which php` or use e.g. `/opt/cpanel/ea-php82/root/usr/bin/php`:

```cron
* * * * * /opt/cpanel/ea-php82/root/usr/bin/php /home/youruser/vigilant/vigilant >/dev/null 2>&1
```

Set a `MAILTO=you@example.com` line above the job if you want cron to email you
on errors (the script writes errors to stderr / the log file).

## Security model

- **No secrets on the command line.** Arguments are world-visible via
  `ps aux` and `/proc/<pid>/cmdline`; the token is only ever read from
  `config.php`.
- **Enforced file permissions.** The script aborts if `config.php` is
  group/other readable or writable — it must be `0600` (or stricter).
- **Least privilege.** Use a per-zone scoped token, not the Global API Key.
- **Keep `config.php` out of git.** It is listed in `.gitignore`; only
  `config.php.example` (with placeholders) is committed.
- **Place it outside the web root** (e.g. above `public_html`) so it can never
  be served, even misconfigured.

## Shared hosting notes

On shared hosting the OS **load average reflects the whole physical machine**,
shared by many tenants — not just your site. Consequences:

- `load` mode can trip because of *other* customers' traffic.
- For per-site DDoS protection, prefer **`requests`** (or `both`), which counts
  only requests to *your* zone.

The tool already avoids `shell_exec`/`exec`, writes only inside your home
directory, and degrades gracefully when `/proc` or `sys_getloadavg()` is
restricted by `open_basedir` / `disable_functions`.

## Exit codes

| Code | Meaning |
| ---- | ------- |
| `0`  | Success. |
| `1`  | Runtime error (e.g. API or network failure). |
| `2`  | Usage or configuration error (bad flag, missing/insecure config). |

## Troubleshooting

- **`permessi insicuri su config.php`** — run `chmod 600 config.php`.
- **`API Cloudflare fallita … 9109`/`authentication`** — token is wrong or
  lacks the required permission/zone scope.
- **`GraphQL … not authorized`** — add `Zone → Analytics → Read` to the token.
- **`Impossibile leggere il load average`** — your host restricts `/proc` and
  `sys_getloadavg()`; switch to `trigger_mode = 'requests'`.
- **`Ne cURL ne allow_url_fopen…`** — ask your host to enable one of them.

## Contributing

Issues and pull requests are welcome. Please keep the tool dependency-free and
compatible with restrictive shared-hosting environments. Run `php -l vigilant`
before submitting.

## License

[MIT](LICENSE) © Ostap Mykhaylyak
