# Flow2API Codebase Guide

Navigation and non-obvious constraints for people and agents changing this
repository. User-facing setup, model list, and API examples live in the root
[`README.md`](../README.md); this guide does not repeat them.

## What this project is

An OpenAI- and Gemini-compatible HTTP facade over Google Flow (image and video
generation). It holds a pool of Google account sessions ("tokens"), solves the
reCAPTCHA Enterprise challenge Flow requires per generation, and translates
Flow's internal RPC protocol into `/v1/chat/completions` and
`/v1beta/models/{model}:generateContent`.

Because Flow's private API changes without notice, most commits are protocol
adaptations (RPC argument shapes, cookie scopes, media/generation id handling).
Expect upstream breakage to be the dominant source of bugs.

## Repository relationship

This repository is a fork. `upstream` is `TheSmallHanCat/flow2api`; `origin` is
the fork. Keep fork-only material (like this `docs/` folder) in new files so
upstream merges stay conflict-free, and avoid rewriting the upstream-owned
`README.md`.

## Where things live

| Concern | Owner |
|---|---|
| Process entry, startup order, captcha runtime warmup | `src/main.py` (`lifespan`) |
| Public generation and model-list endpoints, captcha extension WebSocket | `src/api/routes.py` |
| Admin console API (tokens, config, stats) | `src/api/admin.py` |
| Request orchestration: model → token → captcha → Flow call → polling → stream | `src/services/generation_handler.py` |
| Flow RPC protocol and captcha method dispatch | `src/services/flow_client.py` |
| Model id and alias resolution | `src/core/model_resolver.py` |
| Token selection, concurrency, bans | `src/services/load_balancer.py`, `src/services/concurrency_manager.py`, `src/services/token_manager.py` |
| Cookie auth, refresh, and snapshots | `src/services/flow_cookies.py`, `src/services/browser_cookie_utils.py`, `src/services/protocol_login.py` |
| Persistent state, schema migration, runtime config | `src/core/database.py` |
| Static config defaults | `config/setting_example.toml`, read by `src/core/config.py` |
| Admin, login, and model test pages | `static/` |
| UI and API translations | `static/locales/`, `extension/locales.json`, `src/core/i18n.py` |
| Browser captcha worker extension | `extension/` |
| Image builds (standard and headed) | `Dockerfile`, `Dockerfile.headed`, `.github/workflows/` |

## Constraints the code does not state loudly

- **The database owns configuration after first boot.** `setting.toml` (or the
  example file when absent) seeds `data/flow.db` only on first startup. Later
  startups only add missing tables, columns, and rows; they never overwrite
  existing config. Changes made in the admin console win over the TOML. To
  re-seed from TOML, the database must be removed. See
  `Database.init_config_from_toml` and `Database.check_and_migrate_db`.
- **Config is mirrored in memory.** Code reads `config` from
  `src/core/config.py`; after writing config to the database, call
  `Database.reload_config_to_memory` or the change will not take effect.
- **Captcha method decides the runtime footprint.** `browser` and `personal`
  launch real browsers (Playwright and nodriver respectively) and need a display;
  in Docker that means `docker-compose.headed.yml`. API solvers (yescaptcha,
  capsolver, captcharun, twocaptcha, ezcaptcha, flow2api_team) need no browser.
  `remote_browser` delegates to an external headed service; `extension` relies
  on the in-repo extension connected over `/captcha_ws`. Normalize method
  names through `normalize_captcha_method` before comparing.
- **Generated media is cached on disk** under `tmp/` when the cache is enabled.
  A timeout of `0` means never expire, so the `tmp` mount must persist.
- **Runtime state is git-ignored**: `data/`, `tmp/`, `browser_data*`, and
  `config/setting.toml`. Never commit them; they hold account cookies.
- **Translations are keyed by the Chinese source string.** Both `zh-CN` and `en`
  entries must exist with identical keys and placeholders; `tests/test_i18n.py`
  enforces this. Generation output and SSE streams are never translated.

## Tests

Tests live in `tests/` and use pytest, which is not listed in
`requirements.txt`; install it alongside the requirements. Browser-driven tests
(`tests/test_i18n_browser.py`) skip silently unless Playwright and its
browsers are installed. CI does
not run the suite: `.github/workflows/` only builds and publishes images.
