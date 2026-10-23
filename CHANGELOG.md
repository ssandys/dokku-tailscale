# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]
### Security
- The auth key no longer needs to live in Dokku global config, which Dokku
  injects into every app container's environment (letting any app read a
  reusable key that adds `tag:dokku` machines to the tailnet). It is now read
  from `${DOKKU_LIB_ROOT}/data/tailscale/.authkey` (mode 600, dokku user only).
  Global `TS_AUTHKEY` is still honoured as a fallback with a deprecation
  warning. Migrate with `tailscale:set-key`, then
  `dokku config:unset --global TS_AUTHKEY`, then rotate the old key.

### Added
- `tailscale:set-key` command: reads the auth key from stdin, validates the
  `tskey-` prefix and writes the key file atomically with mode 600.
- `tailscale:key-status` command: shows whether the key comes from the key
  file, the deprecated global config, or is not set (never prints the key).

### Fixed
- Compatibility with current Dokku (verified on 0.38.31): stop sourcing the
  removed `docker-options/functions` and `storage/functions` core libraries.
  Previously every hook that sourced `functions` failed, so `ps:restart` and
  deploys exited with status 1 for *all* apps on the host.
- `tailscale:up` sets `--hostname <app>` via `dokku docker-options:add` and no
  longer stacks duplicate `--hostname` options when run more than once.

### Changed
- `tailscale:down` removes the `--hostname <app>` docker option it added.

## [0.1.1] - 2024-05-01
### Added
- MIT License

### Changed
- `install` attempts to change the ownership of the tailscale dir to dokku:dokku

## [0.1.0] - 2024-04-21
### Added
- `tailscale_attach`

### Changed
- Add ts container to app container network, not the other way around
- Rename `tailscale_down` to `tailscale_detach`
- Data storage goes in a plugin data dir, not the storage dir
- Print tailscale serve url first

### Fixed
- `tailscale_serve_url` does not request TTY during exec

## [0.0.4] - 2024-04-19
### Changed
- Make `tailscale:up` idempotent

### Fixed
- Changelog tag links
- Shorter help text when not calling tailscale:help

## [0.0.3] - 2024-04-13
### Fixed
- `tailscale_serve_url` correction to the conditional

## [0.0.2] - 2024-04-13
### Added
- This changelog
- `tailscale_is_running_for_app` function
- `tailscale_serve_url` function
- `app-urls` trigger

## [0.0.1] - 2024-04-13
### Added
- `tailscale:up` command
- `tailscale:down` command
- `tailscale:serve` command
- `tailscale_up` function
- `tailscale_down` function
- `tailscale_serve_start` function
- `tailscale_serve_stop` function
- `post-destroy` trigger

[unreleased]: https://github.com/andrew-womeldorf/dokku-tailscale/compare/0.1.0...HEAD
[0.1.0]: https://github.com/andrew-womeldorf/dokku-tailscale/compare/0.0.4...0.1.0
[0.0.4]: https://github.com/andrew-womeldorf/dokku-tailscale/compare/0.0.3...0.0.4
[0.0.3]: https://github.com/andrew-womeldorf/dokku-tailscale/compare/0.0.2...0.0.3
[0.0.2]: https://github.com/andrew-womeldorf/dokku-tailscale/compare/0.0.1...0.0.2
[0.0.1]: https://github.com/andrew-womeldorf/dokku-tailscale/releases/tag/0.0.1
