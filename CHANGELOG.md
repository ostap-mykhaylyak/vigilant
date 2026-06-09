# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-06-09

### Added
- Initial release.
- Automatic activation/deactivation of Cloudflare Under Attack Mode (UAM).
- Three trigger modes: `load` (server load average), `requests` (zone
  requests/sec via the GraphQL Analytics API), and `both`.
- Hysteresis (separate on/off thresholds) plus a minimum-dwell timer to
  prevent flapping.
- Secure credential handling: config file is mandated to be `chmod 600`;
  the script refuses to run otherwise.
- Shared-hosting friendly: no `shell_exec`/`exec` dependency, cURL with an
  `allow_url_fopen` stream fallback, all state/log files kept in the home dir.
- CLI flags: `--status`, `--force-on`, `--force-off`, `--config=`,
  `--help`, `--version`.

[1.0.0]: https://github.com/ostap-mykhaylyak/vigilant/releases/tag/v1.0.0
