# vigilant — Cloudflare auto Under Attack Mode

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PHP](https://img.shields.io/badge/PHP-%3E%3D8.0-777bb4.svg)](https://www.php.net/)
[![Release](https://img.shields.io/badge/release-1.0.0-brightgreen.svg)](CHANGELOG.md)

A dependency-free PHP CLI tool that enables [Cloudflare Under Attack Mode (UAM)](https://developers.cloudflare.com/waf/tools/under-attack-mode/)
when server load or request rate crosses a threshold, and turns it back off
when things calm down. Built for `cron` and friendly to **shared hosting (cPanel)**.

## Features

- **Three trigger modes:** `load` (load average per core), `requests`
  (requests/sec to your zone via GraphQL Analytics), or `both`.
- **Hysteresis + min dwell time** to avoid flapping.
- **Secure:** token lives in a `chmod 600` config file, never on the CLI; the
  script refuses to run on loose permissions.
- **Shared-hosting friendly:** no `shell_exec`/`exec`, cURL with a stream
  fallback, all files kept in your home directory.

## Requirements

- PHP 8.0+ (CLI) with `ext-json`
- A Cloudflare-proxied domain and a scoped API token
- `ext-curl` *or* `allow_url_fopen = On`

## Install

```bash
git clone https://github.com/ostap-mykhaylyak/vigilant.git
cd vigilant
cp config.php.example config.php
chmod 600 config.php
$EDITOR config.php   # add token, zone id and thresholds
```

No `composer install` needed.

## API token

Create a **scoped** token (never the Global API Key) at
**Dashboard → My Profile → API Tokens → Create Token**, restricted to the zone:

| Permission | Needed for |
| --- | --- |
| `Zone → Zone Settings → Edit` | read/write `security_level` (all modes) |
| `Zone → Analytics → Read` | `requests` / `both` modes only |

The **Zone ID** is on the domain's Overview page.

## Usage

```bash
php vigilant              # one evaluation cycle (cron entry point)
php vigilant --status     # show current metrics + security level
php vigilant --force-on   # force Under Attack Mode on
php vigilant --force-off  # restore the normal level
php vigilant --help
```

## Cron

```cron
# Run every minute. On cPanel use the versioned PHP binary (e.g.
# /opt/cpanel/ea-php82/root/usr/bin/php) — check with `which php`.
* * * * * /usr/bin/php /home/USER/vigilant/vigilant >/dev/null 2>&1
```

## Configuration

All settings live in `config.php` (see `config.php.example`). Main keys:

| Key | Default | Description |
| --- | --- | --- |
| `trigger_mode` | `load` | `load`, `requests`, or `both` |
| `threshold_on` / `threshold_off` | `1.50` / `0.70` | load-per-core on/off |
| `req_threshold_on` / `req_threshold_off` | `50` / `20` | requests/sec on/off |
| `req_window_minutes` | `5` | requests/sec averaging window |
| `normal_level` | `medium` | level restored when calm |
| `min_uam_seconds` | `600` | min time UAM stays on before standing down |
| `cpu_cores` | `null` | cores for `load` mode (`null` = auto) |

**Tuning:** keep each `off` threshold below its `on` threshold (that gap is the
hysteresis). For `requests` mode, set `req_threshold_on` to ~3–5× your normal
peak. On shared hosting prefer `requests`/`both`, since the OS load reflects the
whole machine, not just your site.

## Exit codes

`0` success · `1` runtime error · `2` usage/config error

## Troubleshooting

- **Insecure permissions** — run `chmod 600 config.php`.
- **API auth error (9109)** — token wrong or lacking the required scope.
- **`authz: zone does not have access to the path`** — add `Zone → Analytics →
  Read`; vigilant uses `httpRequestsAdaptiveGroups`, available on all plans.
- **Cannot read load average** — host restricts `/proc`; use `trigger_mode = 'requests'`.

## License

[MIT](LICENSE) © Ostap Mykhaylyak
