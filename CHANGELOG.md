# Changelog

All notable changes to WiseTrack for WordPress. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow
[Semantic Versioning](https://semver.org/).

## [1.0.1] — 2026-10-11

Bundles WiseTrack Web SDK 2.3.2.

### Changed

- WiseTrack Web SDK updated from 2.3.0 to 2.3.2.
- Redesign dashboard UI/UX.
- Persian text in the admin screens, setup wizard, editor panels and visual picker uses the
  bundled Vazirmatn font (Arabic-script glyphs only; never loaded for visitors).


## [1.0.0] — 2026-10-08

First stable release (bundles WiseTrack Web SDK 2.3.0).

### Added

- **Setup wizard** with a real connection test, page naming preview, integration detection and
  privacy step; dashboard, settings (General, Page tracking, Visitors & Privacy, Integrations,
  Advanced), tools (export/import with preview, reset, system info), admin bar status and debug
  toggle, Site Health tests, per-post settings (block editor panel and classic meta box).
- **Automatic page views** with path/template/URL naming, page types, context params,
  WPML/Polylang language, custom screen rules with `$1…$9` substitution, popups as dialog screens.
- **Event engine**: rules stored as a private post type, compiled per URL; triggers click, submit,
  visible, page load, scroll depth, time on page and DOM events; tokens, revenue, frequency limits;
  presets; `data-wt-*` attributes, `wt-ev-*` classes, `[wisetrack_event]`/`[wisetrack_screen]`
  shortcodes; smart events (outbound, downloads, tel/mailto/WhatsApp/Telegram, scroll, time).
- **Visual picker** on the live site with robust selector generation, "select similar", existing
  events view and live payload preview.
- **Server event queue** delivered through cache-safe REST pulls (`/pending/claim|ack`) for PHP
  API calls, integrations and WooCommerce.
- **Integrations**: WordPress core (login, sign-up, comments, search, 404), Contact Form 7,
  WPForms, Gravity Forms, Elementor Pro Forms, Fluent Forms, Elementor Pro and Popup Maker popups,
  WooCommerce funnel with exactly-once purchases, HPOS and block checkout support, IRT → IRR.
- **Builders**: block editor panel on every block, Elementor section on widgets/containers, step
  guides for Bricks, Breakdance, Oxygen, Divi, WPBakery, Beaver, Thrive and Brizy.
- **Debug overlay** (shadow DOM) with payloads, sanitizer changes, drops and SDK logs.
- **Privacy**: consent mode (WP Consent API or custom), DNT/GPC, bot filter, excluded roles,
  sensitive query params, PII guard; suggested privacy policy text.
- **Compatibility**: exclusions for WP Rocket, LiteSpeed, Autoptimize, SiteGround Optimizer,
  W3 Total Cache, Perfmatters, FlyingPress, Cloudflare Rocket Loader; automatic page-cache purge
  after rule/settings changes; multisite (network activation, new sites).
- **Upgrades**: settings schema version 2 with a migration runner (`wisetrack_db_version`) shared
  by the stored option and imported files.
- **Release**: private build with automatic updates from GitHub Releases (Plugin Update Checker,
  `WISETRACK_DISABLE_UPDATER`), wordpress.org build without updater, CI (lint, PHPStan, PHPUnit,
  Jest, Playwright on WP 6.2 and latest, PHP 7.4 and 8.3, multisite smoke test, Plugin Check)
  and a release workflow.
- Persian (fa_IR) translation of the whole UI (admin, picker, editor panels) and a glossary.

### Changed (compared to the 0.x test builds)

- Site search and 404 events belong to the WordPress integration (migrated automatically).
- The SDK `<link rel="preload">` is off by default (`wisetrack_preload_sdk`): it delayed FCP/LCP
  by ~150 ms in Lighthouse.

### Fixed (found during release hardening)

- Admin screens failed to render on WordPress 6.2–6.4 (`wp.data.createSelector` is 6.5+).
- The `defer` fallback for WordPress < 6.3 was skipped because of the `data-no-defer` attribute.
- Integration event names stored without a custom name overrode the default name.
- Color contrast of help texts, success messages and (WordPress < 6.3) buttons; the setup wizard
  has a page heading and a labelled progress bar.
- `/woo/cart-summary` is rate limited; settings imports are capped at 2 MB and 1000 rules.

[Unreleased]: https://github.com/wisetrack-io/wordpress-plugin/releases
[1.0.1]: https://github.com/wisetrack-io/wordpress-plugin/releases/tag/v1.0.1
[1.0.0]: https://github.com/wisetrack-io/wordpress-plugin/releases/tag/v1.0.0
