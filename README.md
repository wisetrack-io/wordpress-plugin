# WiseTrack for WordPress

> **Plugin version:** 1.0.1 · **Bundled SDK:** WiseTrack Web SDK 2.3.2 · **Requires:** WordPress 6.2+, PHP 7.4+ (WooCommerce 8.0+ for shop tracking) · **License:** GPL-2.0-or-later
>
> 🇮🇷 نسخهٔ فارسی: [`README.fa.md`](README.fa.md)

The official WordPress plugin for [WiseTrack](https://wisetrack.io/) analytics & attribution. It wraps the
**WiseTrack Web SDK** and adds automatic page views, no-code events (event rules, visual picker,
block editor and Elementor panels), WooCommerce revenue and form tracking — cache-safe,
consent-aware and fully translated to Persian.

This document is the complete product and technical reference: installation, setup, every screen
and setting, integrations, page builders, privacy, performance, the developer API (HTML
attributes, CSS classes, shortcodes, JavaScript, PHP, hooks, REST) and how to build, test and
release the plugin.

| I want to… | Go to |
|---|---|
| Install and connect the plugin | [§3 Installation](#3-installation) → [§4 Setup wizard](#4-setup-wizard) |
| Configure tracking | [§8 Settings reference](#8-settings-reference) |
| Track a button, form or section without code | [§9 Events and the visual picker](#9-events-rules-presets-and-the-visual-picker) |
| Track WooCommerce / forms | [§11 Integrations](#11-integrations) |
| Send events from my theme or plugin | [§16 Developer reference](#16-developer-reference) |
| Fix a problem | [§19 Troubleshooting](#19-troubleshooting) |
| Work on the plugin itself | [§21 Development](#21-development) |

---

## Contents

**Part I — Product guide**

1. [Introduction](#1-introduction)
2. [Before you start](#2-before-you-start)
3. [Installation](#3-installation)
4. [Setup wizard](#4-setup-wizard)
5. [Verify the installation](#5-verify-the-installation)
6. [Going live (Sandbox → Production)](#6-going-live-sandbox--production)
7. [Admin screens](#7-admin-screens)
8. [Settings reference](#8-settings-reference)
9. [Events: rules, presets and the visual picker](#9-events-rules-presets-and-the-visual-picker)
10. [Per-page settings](#10-per-page-settings)
11. [Integrations](#11-integrations)
12. [Page builders](#12-page-builders)
13. [Privacy and consent](#13-privacy-and-consent)
14. [Caching, performance and compatibility](#14-caching-performance-and-compatibility)
15. [Security and permissions](#15-security-and-permissions)

**Part II — Developer reference**

16. [Developer reference](#16-developer-reference) — attributes, classes, shortcodes, JavaScript API, tokens, PHP API, hooks, rule format, REST, picker links, custom integrations

**Part III — Operations**

17. [Tools: export, import, reset, system info](#17-tools-export-import-reset-system-info)
18. [Updates, migrations and uninstall](#18-updates-migrations-and-uninstall)
19. [Troubleshooting](#19-troubleshooting)
20. [FAQ](#20-faq)
21. [Development](#21-development)
22. [Appendices](#22-appendices)

---

# Part I — Product guide

## 1. Introduction

### 1.1 Key capabilities

| Area | What you get |
|---|---|
| **Setup** | 6-step wizard with a real connection test, live page-naming preview and plugin auto-detection |
| **Page views** | Automatic, with meaningful names (path / template / URL), WordPress context parameters, WPML/Polylang language, popups as dialog screens, regex-based custom names |
| **No-code events** | Event rules editor, **visual picker** on the live site, ready-made presets, smart events (outbound links, downloads, phone/e-mail/WhatsApp/Telegram clicks, scroll depth, time on page) |
| **Integrations** | WordPress core (login, sign-up, comments, search, 404), WooCommerce (full funnel + revenue), Contact Form 7, WPForms, Gravity Forms, Elementor Pro Forms, Fluent Forms, Elementor Pro & Popup Maker popups |
| **Page builders** | Native panels in the block editor (Gutenberg) and Elementor; attribute/class recipes for Bricks, Breakdance, Oxygen, Divi, WPBakery, Beaver Builder, Thrive, Brizy |
| **Developers** | `data-wt-*` attributes, `wt-ev-*` classes, shortcodes, JavaScript API with early queue, PHP API, 40+ filters/actions, REST API |
| **Operations** | Debug overlay, admin-bar status, Site Health tests, export/import, cache purge, settings migrations, self-updates (private build) |
| **Quality** | Cache-safe, consent-aware, no PII, < 12 KB gzipped visitor script, no jQuery, fully translated to Persian (RTL) |

### 1.2 How it works

```
Visitor's browser                               WordPress (PHP)
─────────────────                               ───────────────
WiseTrack SDK  (loaded async)                   Settings + Schema (validation)
wisetrack-runtime.js (< 12 KB gz, defer)        PageContext  ─► page name & params for this URL
  ├─ SDK bridge   (init, page views, events)    Rules        ─► only the rules matching this URL
  ├─ sanitizer    (mirror of SDK validation)    ServerEventQueue ◄─ WooCommerce / forms / login …
  ├─ rules engine (click, submit, visible …)    REST wisetrack/v1  (settings, rules, pending, woo)
  ├─ attributes   (data-wt-*, wt-ev-*)          Admin: wizard, dashboard, events, settings, tools
  ├─ outbox       (survives page navigation)    Editors: Gutenberg panel, Elementor controls
  ├─ pending      (pulls server-side events)
  └─ consent gate
```

**Page view flow**

1. PHP decides whether this request is tracked (admin screens, login, previews, feeds, REST, builder iframes, excluded roles and pages marked *Do not track* are skipped).
2. The page name, display name, page type, context parameters and the event rules **for this URL only** are printed into `window.WiseTrackWP.config`. This data depends only on the URL, so the page stays fully cacheable.
3. The runtime loads with `defer` in the footer, checks consent / DNT / bots and then injects the SDK asynchronously.
4. Every event passes through one pipeline: `beforeSend` hook → sanitizer → currency conversion → SDK-validation mirror → `trackEvent`.

**Server-side event flow** (purchase, login, form submission…)

1. PHP puts the event in a short-lived per-visitor queue (`ServerEventQueue`) and sets the `wisetrack_pending` cookie.
2. On the next page (or right after an AJAX success signal) the browser calls `POST /wp-json/wisetrack/v1/pending/claim`, sends the events through the SDK and confirms with `/pending/ack`.
3. Nothing visitor-specific is ever printed into HTML — that is what makes the plugin safe with any page cache.

---

## 2. Before you start

### 2.1 Requirements

| Component | Minimum | Notes |
|---|---|---|
| WordPress | 6.2 | Single site and multisite |
| PHP | 7.4 | 8.x recommended |
| WooCommerce | 8.0 | Only for shop tracking; HPOS and Cart/Checkout Blocks compatible |
| Visitors' browsers | Evergreen browsers | The SDK is not loaded for bots by default |
| Outbound HTTPS | `*.wisetrack.io` | Must be allowed by any CSP / firewall |

### 2.2 Checklist

- [ ] A **WiseTrack account** with an application created for this website.
- [ ] The application's **App Token** and **Client Secret** (WiseTrack panel → your app → SDK settings).
- [ ] The site's **domain registered** for the application in the WiseTrack panel. The SDK sends the page hostname as `package_name`; an unregistered host returns `Application not found`.
- [ ] A decision on the **naming method** for page views ([§8.2](#82-page-tracking)) and on the site's **consent policy** ([§13](#13-privacy-and-consent)).
- [ ] A **staging site** or the **Sandbox** environment for the first tests.

> **Tip:** always start in **Sandbox** and switch to **Production** only after you have seen correct data ([§6](#6-going-live-sandbox--production)).

---

## 3. Installation

### 3.1 From the WordPress dashboard (recommended)

1. Download `wisetrack-<version>-private.zip` from the [latest release](https://github.com/wisetrack-io/wordpress-plugin/releases/latest).
2. *(Recommended)* verify the checksum published next to the zip:
   ```bash
   shasum -a 256 -c wisetrack-<version>-private.zip.sha256
   ```
3. Go to **Plugins › Add New › Upload Plugin**, choose the zip, click **Install Now**, then **Activate**.
4. The **setup wizard** opens automatically (only for the user who activated the plugin, and only while the site is not connected).

### 3.2 With WP-CLI

```bash
wp plugin install /path/to/wisetrack-<version>-private.zip --activate
```

The wizard does not open after CLI, bulk or network activation — open **WiseTrack › Setup wizard** yourself.

### 3.3 Multisite

- Activate per site or network-wide. Each site keeps its own settings, rules and credentials.
- `manage_wisetrack` is added to the *administrator* role of every site on activation.

### 3.4 What activation does

- Creates the option `wisetrack_settings` with safe defaults (nothing is tracked until credentials are saved).
- Registers the private post type `wisetrack_rule` (event rules).
- Grants `manage_wisetrack` to administrators.
- Adds the **WiseTrack** admin menu and a **WiseTrack** admin-bar menu on the front end.

---

## 4. Setup wizard

Open it any time from **WiseTrack › Setup wizard**. Every step can be skipped and revisited.

| # | Step | What to do |
|---|---|---|
| 1 | **Welcome** | Overview and where to find the App Token and Client Secret. |
| 2 | **Connect** | Enter **App Token**, **Client Secret** and **Environment** (*Sandbox* first). **Test connection** initializes the real SDK with these credentials and reports the result, e.g. *“Connected. The SDK initialized with these credentials in 1.8 s.”* |
| 3 | **Page views** | Automatic page views on/off, the **naming method** (with real samples from your site: home, latest post, a page, a product), excluded roles and paths. |
| 4 | **Integrations** | Detected active plugins (WooCommerce, Elementor, CF7, WPForms…) with switches (recommended: on), plus smart-event switches. |
| 5 | **Privacy** | Consent mode. If the WP Consent API or a known cookie plugin (Complianz, CookieYes, Real Cookie Banner, Cookiebot) is active, the matching option is suggested. |
| 6 | **Done** | Checklist and shortcuts: visual picker on the home page, debug overlay, developer docs. |

**Connection-test errors**

| Message | Cause | Fix |
|---|---|---|
| `Application not found` | Wrong token, or the current domain is not registered for the app | Check the token; register the domain. Tests from `localhost` usually fail for this reason. |
| Timeout / network error | Outbound requests to `*.wisetrack.io` blocked | Allow the domain in the firewall / CSP `connect-src`. |
| Token / secret validation error | Empty or malformed value | Paste the values again without spaces. |

---

## 5. Verify the installation

1. **Log out** or open a **private window** — administrators are not tracked by default.
2. Browse a few pages.
3. In the WiseTrack panel (Sandbox) page views appear within minutes with names such as `/` and `/sample-page` and the parameters `page_type`, `post_type`, `post_id`.
4. To see what is sent **from your own browser**, turn on the **debug overlay**: admin bar › **WiseTrack › Debug overlay**, or add `?wisetrack_debug=1` to any URL ([§7.5](#75-debug-overlay)).
5. Check **Tools › Site Health**: the WiseTrack tests (credentials, SDK file, environment, optimization plugins, consent mode, shop currency) should pass.

**Admin-bar status dot**

| Colour | Meaning |
|---|---|
| 🟢 Green | The runtime is loaded on this page |
| ⚪ Grey / idle | Your role is excluded, or this is an admin page |
| ⚫ Off | Not configured, not loaded, or the page is marked *Do not track* |

---

## 6. Going live (Sandbox → Production)

1. Confirm in Sandbox that page views, key events and (for shops) a test purchase arrive correctly.
2. **WiseTrack › Settings › General** → **Environment: Production** → Save.
3. Supported page caches are purged automatically; purge your CDN yourself (or hook `wisetrack_page_cache_purged`).
4. Turn the debug overlay off in your browser.

> A production-looking site that still sends to Sandbox triggers a Site Health warning and an admin notice (`wisetrack_is_production_site` adjusts the detection).

---

## 7. Admin screens

The **WiseTrack** menu contains **Dashboard · Events · Settings · Tools · Setup wizard**. All screens require `manage_wisetrack`. Every screen has a **Help › Developers** tab with code shortcuts.

### 7.1 Dashboard

- **Status:** connected / not connected, Sandbox or Production, SDK version and source, SDK file integrity, last connection test.
- **Quick actions:** re-run the connection test, open the visual picker, debug overlay help.
- **What is tracked:** page tracking, integrations and rules at a glance.
- Not connected yet: **Start / Continue setup**.

### 7.2 Settings

Five tabs, deep-linkable: `#general`, `#pages`, `#privacy`, `#integrations`, `#advanced`. A sticky save bar shows unsaved changes, offers **Discard** and warns before leaving. Server-side validation errors appear under the field and on the tab label. Reference: [§8](#8-settings-reference).

### 7.3 Events

Event rules with search, filters (status, trigger type, source), pagination, a per-row switch and bulk actions (enable, disable, delete, duplicate, export). Below the list: read-only cards for **WooCommerce events** and **smart events**. See [§9](#9-events-rules-presets-and-the-visual-picker).

### 7.4 Tools

Export / import, reset, debug overlay toggle, system info. See [§17](#17-tools-export-import-reset-system-info).

### 7.5 Debug overlay

A floating panel on the front end, enabled **per browser** (nothing is printed into HTML):

- Enable: admin bar › **WiseTrack › Debug overlay**, `?wisetrack_debug=1`, or `localStorage.setItem( 'wisetrack_debug', '1' )`.
- Shows: SDK status, a timeline of page views and events with the **before/after sanitizer diff**, dropped parameters/events with the reason, rule matches, server events and SDK logs.
- Buttons: copy JSON, clear, highlight targets. Draggable and collapsible. (Its strings are intentionally English.)

---

## 8. Settings reference

### 8.1 General

| Field | Default | Description |
|---|---|---|
| App Token | — | From the WiseTrack panel. |
| Client Secret | — | Masked; **Show** reveals it. A browser SDK needs it in the page, so it is a public client credential, not a server secret. |
| Environment | `production` | `sandbox` for tests, `production` for real data. |
| SDK source | `local` | *Private build only:* `local` (bundled file), `jsdelivr` or `unpkg`. The wp.org build always uses the bundled file. |
| App version | `1.0.0` | Sent as the app version; change it to segment data by site release. |
| Deep links | on | Emits `wisetrack:deeplink` for direct / deferred deep links. |

### 8.2 Page tracking

| Field | Default | Description |
|---|---|---|
| Automatic page views | on | A page view on every tracked page. |
| Track dialogs | on | Popups (Elementor Pro, Popup Maker) are sent as dialog screens `popup:<id>`. |
| Naming method | `path` | `path` → `/shop/shoes`; `template` → `single_product`, `archive_category`…; `url` → full URL without sensitive parameters. |
| Excluded paths | — | Path prefixes that never load the SDK, e.g. `/my-account`. |
| Excluded patterns | — | Regular expressions; check them with **Test a path**. |
| Context parameters | `page_type, post_type, post_id` | WordPress context attached to page views. |
| Sensitive query parameters | WP/Woo defaults | Stripped from URLs and tokens: `key, order, order_key, pwd, password, token, email, phone, coupon…` |

| Naming method | Best for | Example |
|---|---|---|
| `path` | Most sites; readable, one name per URL | `/blog/hello-world` |
| `template` | Large catalogues, grouped reports | `single_post`, `single_product`, `archive_product_cat` |
| `url` | Sites that distinguish pages by query string | `https://example.com/?p=12` |

Per-page names ([§10](#10-per-page-settings)) and custom screen names ([§8.5](#85-advanced)) always win.

### 8.3 Visitors & privacy

| Field | Default | Description |
|---|---|---|
| Excluded roles | `administrator` | Logged-in users with these roles are not tracked. |
| Exclude bots | on | User-agent list + `navigator.webdriver`. |
| Respect DNT / GPC | off | Do not track browsers sending *Do Not Track* / *Global Privacy Control*. |
| Consent mode | `off` | `off`, `wp_consent_api` or `custom` — [§13](#13-privacy-and-consent). |
| Consent category | `statistics` | WP Consent API category to wait for. |

### 8.4 Integrations

- Supported plugins with an on/off switch each (only active plugins run).
- Each integration event can be switched off or **renamed** (the UI shows *Sent as `…`*).
- **Forms:** per-form on/off, custom event name and extra parameters ([§11.2](#112-forms)).
- **Smart events:** outbound links, downloads, `tel:`, `mailto:`, WhatsApp, Telegram, scroll depth, time on page.
- **Download extensions** (default `pdf,zip,rar,docx,xlsx,pptx,mp3,mp4,apk`).
- **Outbound link delay** 0–500 ms — gives the SDK time to send before the browser leaves.
- **Send the search term** (off — search terms can contain personal data).
- **HTML attributes and CSS classes** (`data-wt-*`, `wt-ev-*`) on/off (`attributes_enabled`).
- **WooCommerce:** purchase trigger, revenue basis, currency conversion with a live example ([§11.3](#113-woocommerce)).

### 8.5 Advanced

| Field | Default | Description |
|---|---|---|
| Load strategy | `defer` | `defer` (after parsing) or `idle` (when the browser is idle — lowest impact, slightly later start). |
| Tracking delay | `0` s | Delay before tracking starts. |
| Optimization plugin compatibility | on | Keeps WP Rocket, LiteSpeed, Autoptimize, SiteGround, W3TC, Perfmatters and FlyingPress from delaying/combining/minifying WiseTrack scripts; adds `data-cfasync="false"` for Cloudflare Rocket Loader. |
| SDK log level | `warn` | `debug`, `info`, `warn`, `error`, `none`. |
| Custom screen names | — | Regex rules on the path (see below). |
| Remove data on uninstall | off | Delete settings, rules and queues when the plugin is deleted. |

**Custom screen names** rename page views by a regular expression on the path (first match wins, tested on `location.pathname` by the SDK). `$1`…`$9` insert capture groups into the name and display name:

| Pattern | Name | Display name | `/courses/php-basics/lesson-3` → |
|---|---|---|---|
| `^/courses/([^/]+)/lesson-(\d+)` | `course_lesson` | `$1 — lesson $2` | name `course_lesson`, display `php-basics — lesson 3` |
| `^/blog/` | `blog_post` | *(empty)* | name `blog_post`, display name from the page |

Names are sanitized like every screen name (≤ 150 chars); patterns are validated (PCRE) on save.

---

## 9. Events: rules, presets and the visual picker

### 9.1 Rules that apply everywhere

| Item | Rule |
|---|---|
| Event name / param key | Normalized to `[a-z0-9_]`, starts with a letter, ≤ 50 chars: `"Create Account"` → `create_account`. Use Latin names; other languages belong in labels and values. |
| Param value | ≤ 100 chars; quotes, `%`, `#`, HTML removed; SQL words neutralized (`"Update cart"` → `update_cart`). |
| Params per event | ≤ 25 (`wisetrack_max_event_params`). |
| PII guard | Values that look like e-mail addresses or phone numbers (incl. Iranian mobiles `09xxxxxxxxx`) are dropped. Never send personal data, WordPress user ids, names or identifying form values. |
| Caching | Never print visitor-specific data into HTML — use the PHP API ([§16.6](#166-php-api)). |

The SDK silently drops invalid events — the plugin validates everything first, and the debug overlay shows what was changed or dropped.

### 9.2 Event editor

| Section | Content |
|---|---|
| **Event** | Label (any language, e.g. “Hero buy button”) and event name (Latin); the normalized name is shown live. |
| **Trigger** | Click · Form submit · Visible (with %) · Page load · Scroll depth · Time on page · DOM event |
| **Target** | CSS selector + **Pick on site**. *Include children* counts clicks on icons inside the element. |
| **Where** | All pages, or *Only* / *Except* with conditions: specific posts/pages/terms, post type, path starts with, regex. |
| **Data** | Parameters with token chips and a live preview passed through the shared sanitizer. |
| **Revenue** | Amount source — fixed, element attribute (e.g. `data-price`), element text (e.g. `.price .amount`) or token — and currency. |
| **How often** | Every time · Once per page view · Once per session · Once per visitor, plus a private note. |
| **Preview** | JSON payload and live validation (`/rules/validate`). |

### 9.3 Examples

**Pricing button click** — label *Gold plan buy button*, name `click_buy_gold`, trigger *Click* on `#pricing .plan-gold .btn`, *Only* path `/pricing`, params `button_text = {{element.text}}` and `plan = gold`, revenue fixed `490000` `IRT` (sent as `4900000 IRR`), *Once per page view*.

**Pricing section seen** — trigger *Visible* 50 % on `#pricing`, name `pricing_seen`, once per session.

**Theme custom event** — trigger *DOM event* `my-theme:signup`; the theme runs
`document.dispatchEvent( new CustomEvent( 'my-theme:signup', { detail: { plan: 'pro' } } ) )` and `detail` becomes parameters (jQuery events of that name are heard too).

### 9.4 Presets

One click creates a pre-filled rule: outbound link · file download · phone click · e-mail click · WhatsApp click (`wa.me`, `api.whatsapp.com`, `whatsapp://`) · Telegram click (`t.me`, `tg://`) · Instagram click · scroll 50 % / 90 % · 30 s / 60 s on page · 404 page · search results page · form submit (generic) · *Form submit — &lt;plugin&gt;* for every active form plugin.

### 9.5 Visual picker

**Open it from:** admin bar › **WiseTrack › Pick an element to track** (any front-end page) · row action **Track elements** in the Posts/Pages lists · **Open the visual picker** on the dashboard · **Pick on site** in the event editor.

**Workflow**

1. Hover to highlight elements (`tag#id.class (N matches)`).
2. Click to select (links and forms do not fire). **Browse** mode or holding `Alt` lets the page work normally (e.g. to open a menu).
3. Keyboard: `Esc` cancel, `↑` / `↓` parent / child, `Enter` jump to the name field; breadcrumb chips work too.
4. Side panel (left in RTL, bottom sheet on mobile):
   - editable selector with live match count and a **stability score** (high / medium / low, with reasons);
   - **Select similar** (e.g. every *Add to cart* button of a product grid);
   - suggested label and name (Persian text is transliterated: «خرید» → `click_kharid`);
   - trigger (click / submit / visible) — inside forms: *Track the form submit instead*;
   - scope: this page / all pages of this type / whole site / current conditions / advanced;
   - parameters with live values, revenue (parsed even from «۴۹۰٬۰۰۰ تومان» → 490000), frequency;
   - **payload preview**; **Save** · **Save & pick another** · **Open in full editor**.
5. **Show existing events** badges elements that already have active rules, lets you edit/disable them and lists rules *not found on this page*.
6. Open Events screens refresh automatically after a save.

> Elements inside iframes cannot be picked. For robust selectors give elements a `data-wt-event`, a readable `id` or a `data-testid`.

---

## 10. Per-page settings

Block editor: **WiseTrack** panel in the document sidebar. Classic editor: a meta box.

| Field | Effect |
|---|---|
| Page name / display name | Overrides the naming method for this page. |
| Do not track this page | The SDK is not loaded on this URL. |
| Send an event when the page loads | Event name + parameters — e.g. a thank-you page sends `lead_thank_you` with `source = landing_a`. |

Post types with these settings: filter `wisetrack_post_types`.

---

## 11. Integrations

*Settings › Integrations* lists every supported plugin that is active. An integration runs only when **its plugin is active and the integration is switched on** (the wizard suggests it). Each event can be switched off or renamed.

### 11.1 WordPress core

| Event | Default name | Params | Delivered |
|---|---|---|---|
| Login | `login` | `method: wp` | Next page view (server queue) |
| Sign-up | `sign_up` | `method: wp` | Next page view — only self sign-ups (registration form, checkout); not accounts created by a logged-in user, WP-CLI or cron |
| Comment | `comment_submit` | `post_id`, `post_type`, `comment_status` (`approved`/`pending`) | Next page view; never spam, content or author data |
| Search | `search` | `results_count` (+ `search_term` only with *Send the search term*) | Search results page |
| Page not found | `page_not_found` | `path` | 404 page |

Role exclusions use the user that logs in / signs up; anonymous REST requests (form plugins without a nonce) still recognize a logged-in visitor by the auth cookie.

### 11.2 Forms

| Plugin | Server hook (event source) | Client signal (immediate pull) |
|---|---|---|
| Contact Form 7 | `wpcf7_mail_sent` | `wpcf7mailsent` |
| WPForms | `wpforms_process_complete` | `wpformsAjaxSubmitSuccess` |
| Gravity Forms | `gform_after_submission` (spam skipped) | `gform_confirmation_loaded` |
| Elementor Pro Forms | `elementor_pro/forms/new_record` | `submit_success` |
| Fluent Forms | `fluentform/submission_inserted` (+ the pre-5.0 name) | `fluentform_submission_success` |

- Only **successful** submissions count; the event is `form_submit` with `form_plugin` (integration slug), `form_id`, `form_name`.
- The server hook queues the event; AJAX forms trigger an immediate pull, page-reload forms deliver it on the next page view. Each submission is queued once (`plugin:form_id:submission`), so the client signal never adds a second event.
- **Per form** (*Settings › Integrations › Forms*): on/off, a custom name (e.g. `quote_request`) and extra params — fixed text or `{{form.field:NAME}}` / `{{form.id}}`. Field names: CF7 tag name, WPForms field id or label, Gravity field id / admin label / parameter name, Elementor field id, Fluent input name.
- **Never sent:** values of password, e-mail, phone, hidden and file fields, of fields whose name looks personal (`pass|mail|phone|mobile|tel|card|cvv|national|melli|nonce|token|secret`) and values that look like an e-mail or phone number.
- Need an extra page-specific event? *Events › Presets › Form submit — &lt;plugin&gt;* creates a rule on the plugin's client success signal.

*Example:* CF7 form “Consultation request” (id 123) → name `consult_request`, param `service = {{form.field:service-type}}` (a non-personal select).

### 11.3 WooCommerce

Requires WooCommerce ≥ 8.0; HPOS and Cart/Checkout Blocks compatible (orders are only touched through WooCommerce CRUD).

| Event | Default name | Params | Source |
|---|---|---|---|
| Product viewed | `view_item` | `product_id`, `product_sku`, `product_cat` (Yoast/Rank Math primary term, else the first), `price`, `currency` | Product page (URL data printed with the page) |
| Added to cart | `add_to_cart` | `product_id`, `variation_id`?, `quantity`, `price` (unit), `currency` | `woocommerce_add_to_cart` (classic AJAX, product form, Store API/blocks) → server queue |
| Removed from cart | `remove_from_cart` | `product_id`, `variation_id`?, `quantity` | `woocommerce_cart_item_removed` → server queue |
| Cart viewed | `view_cart` | `item_count`, `cart_total`, `currency` | Cart page (configured page, Cart block or `[woocommerce_cart]`); totals from `GET woo/cart-summary` |
| Checkout started | `begin_checkout` | as `view_cart` | Checkout page (not order-received / order-pay) |
| Purchase | `purchase` (revenue) | `order_id`, `item_count`, `payment_method`, `shipping_method`?, `coupon_used` | See below |

- Money params (`price`, `cart_total`) are converted like revenue (Toman → Rial) and carry the resulting `currency`.
- Cart events are pulled right after WooCommerce's client events: jQuery `added_to_cart`, `removed_from_cart`, `updated_cart_totals`, DOM `wc-blocks_added_to_cart`, `wc-blocks_removed_from_cart` and (not on checkout) `wc-blocks_store_sync_required`. Otherwise they arrive with the next page view.
- The page script `build/integrations.js` loads on product, cart and checkout pages only.

**Purchase — exactly once per order**

1. **Order created** (`woocommerce_checkout_order_processed`, `woocommerce_store_api_checkout_order_processed`): the visitor id is stored on the order (`_wisetrack_visitor_sid`). Excluded roles get none.
2. **Ready** (`_wisetrack_purchase_ready`), per *Purchase trigger*:
   - `paid` *(default)* — the first of `woocommerce_payment_complete` / status `processing` / `completed` (on-hold, failed, cancelled → nothing);
   - `thankyou` — at creation, sent from the thank-you page only (failed orders never).
3. **Queued once** (`_wisetrack_purchase_queued`) for the stored visitor — also when a gateway webhook made it ready. The visitor gets it on the thank-you page or the next page view (the browser keeps asking while a payment is pending). The thank-you page validates the order key, adopts the visitor if the order has none and re-offers a queued but unacknowledged purchase; an order-received link opened in another browser sends nothing.
4. **Acknowledged** by the browser → `_wisetrack_purchase_sent` + private order note **“WiseTrack: purchase event sent.”** (filter `wisetrack_order_note`). The item uid derives from the order id, and the browser remembers uids it already handed to the SDK, so re-deliveries are never sent twice.

**Revenue and currency**

- **Revenue basis:** *Order total* or *Subtotal* (items after discounts, without tax, shipping and fees); filter `wisetrack_revenue_amount`; amounts in the **order** currency.
- **Iranian currencies → Rial (`IRR`)**, rounded to integers: `IRT` ×10, `IRHT` (thousand Toman) ×10 000, `IRHR` (thousand Rial) ×1 000. Codes used by Iranian currency plugins (`TOMAN`, `TMN`, `IRTOMAN`, `RIAL`, `HTOMAN`…) and Persian symbols (`تومان`, `ریال`, `هزار تومان`…) are recognized.
- **Other unsupported codes:** add a row in *Currency conversion*, e.g. `{"XYZ": {"to": "USD", "factor": 1}}`. Still unsupported → the purchase is sent without revenue, Site Health warns and *WooCommerce › Status › Logs* (source `wisetrack`) records it.
- The settings screen shows a live example: *1,250,000 Toman → 12,500,000 IRR*.

Queue storage on WooCommerce sites: each visitor's queue is its own row in WooCommerce's session table (`session_key` `wt…`), cleaned up by WooCommerce — never inside the customer's session data.

### 11.4 Popups

Elementor Pro popups (`elementor/popup/show`) and Popup Maker (`pumAfterOpen`): each opening is a dialog screen `popup:<id>` with the popup title as display name. Optional event `popup_open` (`popup_id`, `popup_name`), off by default. `build/integrations.js` (< 1 KB gz) loads only where the popup plugin loads its own script. Native `<dialog>` elements are tracked by the SDK itself.

---

## 12. Page builders

### 12.1 Block editor (Gutenberg)

Every block has a **WiseTrack** panel: event name (normalized live), *Send when* (clicked, visible, form inside submitted — form-like blocks only, page loads), parameters with token chips, *How often*, and *Screen on click* for tab-like blocks (tabs, accordions, details, steps). A chart icon in the block toolbar marks blocks with an event; *Pick on site* opens the picker on the published page.

Settings live in a block attribute `wisetrack` stored only in the block comment (`<!-- wp:button {"wisetrack":{"event":"buy"}} -->`). Saved HTML is untouched, so content never becomes invalid and nothing remains if WiseTrack is removed. On render, the first HTML tag of the block gets the matching `data-wt-*` attributes (static and dynamic blocks; attributes already in the markup win).

### 12.2 Elementor (free and Pro)

*Advanced* tab → **WiseTrack** on every widget, section, column and container: event name, *Send when*, parameters (repeater, tokens allowed), *How often*. Attributes are printed on the element wrapper; for the Button widget on the link itself. Switching the integration on/off clears Elementor's cache. In Elementor free you can also use the class convention; in Elementor Pro, *Attributes*.

Hooks used (Elementor 4.3): `elementor/element/common/_section_style/after_section_end` (widgets), `elementor/element/section/section_advanced/after_section_end`, `elementor/element/column/section_advanced/after_section_end`, `elementor/element/container/section_layout/after_section_end`, `elementor/frontend/before_render`.

### 12.3 Other builders

No WiseTrack controls — use custom attributes where the builder has them, otherwise the class convention ([§16.2](#162-css-class-convention)), or the visual picker on the published page.

| Builder | Steps |
|---|---|
| **Bricks** (attributes) | Element → *Style* → *Attributes* → *Add attribute* → name `data-wt-event`, value `signup_click` (optional: `data-wt-param-plan` = `pro`, `data-wt-trigger` = `visible`). |
| **Breakdance / Oxygen 6** (attributes) | Element → *Settings* → *Advanced* → *Attributes* → *Add* → `data-wt-event` = `signup_click`. |
| **Oxygen 4** (attributes) | Element → *Advanced* → *Attributes* → *Add attribute* → `data-wt-event` = `signup_click`. |
| **Elementor Pro** (attributes) | Widget → *Advanced* → *Attributes*, one per line: `data-wt-event\|signup_click`, `data-wt-param-plan\|pro`. |
| **Elementor free** (classes) | Widget → *Advanced* → *CSS Classes* → `wt-ev-signup_click wt-p-plan-pro`. |
| **WPBakery** (classes) | Element → *Design Options* / *General* → *Extra class name* → `wt-ev-signup_click` (+ `wt-p-plan-pro`). |
| **Divi** (classes) | Module → *Advanced* → *CSS ID & Classes* → *CSS Class* → `wt-ev-signup_click` (visible: `wt-ev-pricing_seen--visible`). |
| **Beaver Builder** (classes) | Module → *Advanced* → *HTML Element* → *Class* → `wt-ev-signup_click`. |
| **Thrive Architect / Brizy** (classes) | Element → *HTML attributes* / *Advanced* → *CSS class* → `wt-ev-signup_click`. |

Editor/preview URLs never load the SDK: `elementor-preview`, `fl_builder`, `et_fb`, `ct_builder`, `oxygen=builder`, `oxygen_iframe`, `bricks=run`, `vc_editable`, `vcv-editable`, `tve`, `brizy-edit`, `brizy-edit-iframe`, `breakdance=builder`, `breakdance_iframe`, `fb-edit` (filter `wisetrack_builder_preview_params`).

---

## 13. Privacy and consent

### 13.1 Consent modes

| Mode | Behaviour | Use when |
|---|---|---|
| `off` | Tracking starts immediately | No consent requirement for your audience |
| `wp_consent_api` | The SDK is **not loaded** until the visitor grants the configured category (default `statistics`), then loads at once | Complianz, CookieYes, Real Cookie Banner, Cookiebot or another WP Consent API plugin |
| `custom` | Waits for `WiseTrackWP.consent( true )`; the choice is remembered in `localStorage` | Your own banner |

- Revoking (`consent( false )`) calls the SDK's `flush()` and stops tracking.
- Before consent, browser events stay in memory; server-queued items wait. Consent and DNT are decided in the browser.

```js
// Custom banner (consent mode "custom")
document.querySelector( '#accept-cookies' ).addEventListener( 'click', () => WiseTrackWP.consent( true ) );
document.querySelector( '#reject-cookies' ).addEventListener( 'click', () => WiseTrackWP.consent( false ) );
```

### 13.2 Data protection by design

- No WordPress user ids, names, e-mails, phones, hashes or `customDeviceId` are sent.
- The PII guard drops e-mail- and phone-like values from every parameter (server side: `wisetrack_allow_param_value` can allow one deliberately).
- Sensitive query parameters are removed from URLs and tokens (`wisetrack_sensitive_query_params`).
- Search terms are sent only when explicitly enabled.
- Administrators excluded by default, bots filtered, DNT/GPC optional.
- A suggested text is added to WordPress' **Privacy Policy guide** — mention WiseTrack in your policy.

---

## 14. Caching, performance and compatibility

### 14.1 Page cache

- HTML contains only URL-dependent data → every page can be cached for everyone.
- Rules and settings are printed into the page. When they change, WiseTrack purges the page cache of WP Rocket, LiteSpeed Cache, W3 Total Cache, SiteGround Optimizer, WP Super Cache, FlyingPress, WP Fastest Cache, Hummingbird, Cache Enabler, Breeze and Nginx Helper once at the end of the request (`wisetrack_purge_page_cache` turns it off). Other caches and CDNs: hook into `wisetrack_page_cache_purged`.

### 14.2 Optimization plugins

With *Optimization plugin compatibility* (`optimization_compat`, on) optimizers are told not to delay, defer, combine or minify WiseTrack scripts; the tags also carry `data-cfasync="false"` (Cloudflare Rocket Loader), `data-no-optimize`, `data-no-defer` and `data-no-minify`.

### 14.3 Performance

| Item | Value |
|---|---|
| Visitor runtime | < 12 KB gzipped, `defer`, footer, no jQuery |
| SDK | ~50 KB gzipped, loaded async after render |
| LCP / CLS | Identical with and without the plugin (Lighthouse) |
| Mobile TBT | ~90 ms from SDK start-up on emulated slow phones (desktop: 0) |
| SDK preload | Off (it delayed FCP/LCP by ~150 ms); enable with `wisetrack_preload_sdk` |

On very performance-sensitive sites use **Load strategy: idle**.

### 14.4 Content Security Policy

```
connect-src  https://*.wisetrack.io;
script-src   https://cdn.jsdelivr.net https://unpkg.com;   # only when the SDK source is a CDN (private build)
```

### 14.5 PWAs / service workers

Never cache requests to `wisetrack.io`, `/wp-json/wisetrack/v1/pending/*` and `/wp-json/wisetrack/v1/woo/*` — use **NetworkOnly**.

### 14.6 Multilingual

With WPML or Polylang the page language is added to page views; names stay stable across languages with the `template` method or custom screen names.

---

## 15. Security and permissions

- Every admin screen and admin REST route requires **`manage_wisetrack`** and a `wp_rest` nonce. Grant it to another role with a role editor, or change it with `wisetrack_capability`.
- The visual picker needs its own nonce; without it the request is redirected to the same URL without picker parameters ([§16.10](#1610-visual-picker-links)).
- Public routes (`/pending/*`, `/woo/cart-summary`) use an HttpOnly visitor cookie, same-origin `Origin`/`Referer` checks, rate limits (per visitor 30/min, per IP 120/min; cart summary 60/min per IP), body-size caps and `Cache-Control: no-store`.
- Inputs go through a central schema, output is escaped late, inline JSON uses `JSON_HEX_*`, regexes are validated and bounded; no `eval` / `new Function`.
- Behind a reverse proxy pass the real client IP with `wisetrack_client_ip`; extra allowed origins: `wisetrack_pending_allowed_hosts`.

> **Note:** the **Client Secret** is necessarily present in the page for a browser SDK. It is masked in the UI and excluded from exports unless you opt in — treat it as a public client credential.

---

# Part II — Developer reference

## 16. Developer reference

Everything you can do **without** the Events screen. All of it follows the rules of [§9.1](#91-rules-that-apply-everywhere); turn on the debug overlay to see every event, what the sanitizer changed and why something was dropped.

### 16.1 HTML attributes

Work in theme templates, block "additional attributes", page-builder custom attribute fields (Bricks, Breakdance, Oxygen…) and anywhere you control the markup. Enabled by default (*Settings › Integrations › HTML attributes and CSS classes*, key `attributes_enabled`).

```html
<a href="/pricing" data-wt-event="pricing_click" data-wt-param-plan="pro" data-wt-param-position="hero">Pricing</a>

<form data-wt-event="newsletter_signup" data-wt-trigger="submit">…</form>

<section data-wt-event="pricing_seen" data-wt-trigger="visible" data-wt-once="page">…</section>

<button data-wt-event="buy_plan" data-wt-revenue="490000" data-wt-currency="IRT"
        data-wt-params='{"plan":"gold"}'>Buy</button>

<button data-wt-screen="settings_billing" data-wt-screen-display="Billing" data-wt-screen-type="other">Billing tab</button>

<div data-wt-no-track>…rules, attributes, classes and smart events inside are ignored…</div>
```

| Attribute | Meaning |
|---|---|
| `data-wt-event` | Event name. |
| `data-wt-trigger` | `click` (default) · `submit` (on a `<form>`) · `visible` (50 % in view) · `load` (on page load). |
| `data-wt-param-<key>` | One param; kebab-case keys become snake_case (`data-wt-param-button-text` → `button_text`). |
| `data-wt-params` | JSON object of params (merged; `data-wt-param-*` wins). |
| `data-wt-revenue` | Revenue amount (`490000`, `"1,250,000"`, Persian digits are fine). |
| `data-wt-currency` | Currency code; default: the site (WooCommerce) currency. `IRT` → `IRR` ×10, `IRHT` ×10 000, `IRHR` ×1 000. |
| `data-wt-once` | `page` · `session` · `visitor` — at most once per page view / browser session / visitor. |
| `data-wt-screen` | Manual screen name (tabs, steps); on click by default, `data-wt-trigger="load"` on load. |
| `data-wt-screen-display` | Screen display name. |
| `data-wt-screen-type` | `other` (default) · `page` · `dialog` · `bottom_sheet` · `webview`. |
| `data-wt-no-track` | Opt a whole area out of tracking. |

Clicks inside an element count for the nearest ancestor with `data-wt-event`. Param values may contain [tokens](#165-tokens), e.g. `data-wt-param-label="{{element.text}}"`.

### 16.2 CSS-class convention

For builders that only let you add classes (WPBakery, Divi, Beaver Builder… — [§12.3](#123-other-builders)):

```html
<a class="button wt-ev-signup_click wt-p-plan-pro wt-p-position-hero">Sign up</a>
<section class="wt-ev-pricing_seen--visible">…</section>
```

- `wt-ev-<event_name>` → event on click.
- `wt-ev-<event_name>--visible` → event when the element becomes visible.
- `wt-p-<key>-<value>` → static param (`key` is everything up to the first `-`; simple values only).

### 16.3 Shortcodes

```text
[wisetrack_event name="lead" plan="pro"]
```
Fires `lead` with `plan=pro` **on page load** (prints an empty hidden `<span>`).

```text
[wisetrack_event name="cta_click" plan="pro"]Click me[/wisetrack_event]
```
Wraps the content in a `<span>`; fires on click (or `trigger="visible"`).

```text
[wisetrack_screen name="checkout_step_2" display="Shipping" type="other"]
```
Manual screen on load.

Reserved attributes: `name`, `trigger`, `once` (`page|session|visitor`), `revenue`, `currency`, `params` (JSON), `display`, `type`. Every other attribute becomes a param.

### 16.4 JavaScript API

The runtime is loaded deferred in the footer. Calls made before it is ready go into a queue — add this stub to your theme's `<head>` (or any earlier script) and use it anywhere:

```html
<script>
	window.wisetrackQueue = window.wisetrackQueue || [];
	wisetrackQueue.push( [ 'track', 'video_play', { video_id: 'intro' } ] );
</script>
```

Queue commands: `track`, `trackRevenue`, `trackScreen`, `consent`, `ready`, `on`, `off`, `pullPending`; a function pushed to the queue runs on `ready`.

```js
WiseTrackWP.ready( ( api ) => { /* SDK initialized and enabled */ } );

WiseTrackWP.track( 'signup', { method: 'google' } );
WiseTrackWP.trackRevenue( 'purchase', 1250000, 'IRT', { order_id: 'A-1' } ); // currency optional → 12500000 IRR
WiseTrackWP.trackScreen( { name: 'checkout_step_2', displayName: 'Shipping', type: 'other', params: {} } );

WiseTrackWP.on( 'beforeSend', ( payload ) => {
	if ( payload.name === 'internal_test' ) {
		return false; // cancel
	}
	return { ...payload, params: { ...payload.params, theme: 'dark' } }; // modify
} );
WiseTrackWP.on( 'sent', ( event ) => console.log( 'sent', event ) );
WiseTrackWP.on( 'dropped', ( info ) => console.warn( info.reason, info ) );

WiseTrackWP.consent( true );   // consent mode "custom": load/allow; false revokes (flush + stop)
WiseTrackWP.status();          // idle | waiting_consent | loading | ready | blocked | failed | revoked
WiseTrackWP.pullPending();     // fetch events queued by PHP for this visitor now (e.g. after your AJAX call)
WiseTrackWP.sdk;               // raw WiseTrackSDK namespace once ready (null before)
WiseTrackWP.config;            // the page config printed by PHP (rules, page context…)

// Without the API object:
document.dispatchEvent( new CustomEvent( 'wisetrack:track', { detail: { name: 'quiz_done', params: { score: 7 } } } ) );

// Emitted by the runtime:
document.addEventListener( 'wisetrack:ready', ( e ) => {} );     // detail: { version }
document.addEventListener( 'wisetrack:deeplink', ( e ) => {} );  // detail: { uri, isDeferred }
```

Events fired before the SDK is enabled (or from a click that navigates away) are kept in `sessionStorage.wisetrack_outbox` (max 50, 10 minutes) and delivered on the next page.

### 16.5 Tokens

Param values of event rules, attributes and the event editor can contain dynamic tokens, resolved in the visitor's browser and then sanitized:

| Token | Value |
|---|---|
| `{{element.text}}` | Text of the element (trimmed, whitespace collapsed; buttons: their value). |
| `{{element.attr:NAME}}` | Attribute value (`value` of form fields is refused). |
| `{{element.data:NAME}}` | `data-NAME` attribute. |
| `{{element.href}}` | Link URL without query string and hash (`mailto:`/`tel:` refused). |
| `{{element.id}}` | Element id. |
| `{{page.path}}` · `{{page.title}}` · `{{page.post_id}}` · `{{page.post_type}}` · `{{page.type}}` | Page data (`page.type` = template slug such as `single_product`, `404`). |
| `{{url.query:NAME}}` | Query param (refused for sensitive params: `token`, `password`, `email`, the *Sensitive query parameters* setting…). |
| `{{form.id}}` · `{{form.field:NAME}}` | Form id/name and a field value. **Refused** for `password`, `email`, `tel`, `hidden` and `file` fields and names matching `pass|mail|phone|mobile|tel|card|cvv|national|melli|nonce|token|secret`. |
| `{{scroll.percent}}` · `{{time.seconds}}` | Scroll depth / visible seconds on the page. |

A value that is exactly one token keeps its type (`{{page.post_id}}` stays a number). A refused or missing token drops that param (visible in the debug overlay).

### 16.6 PHP API

For things that happen on the server (logins, sign-ups, form handlers, webhooks in the visitor's request). Events are queued for the current visitor (cookie `wisetrack_sid`) and delivered by the browser on their next page view — or immediately after an AJAX success event registered with `wisetrack_pending_triggers`, or when your JS calls `WiseTrackWP.pullPending()`.

```php
wisetrack_track_event( string $name, array $params = [], array $args = [] ): bool;
// $args: 'revenue' => number, 'currency' => 'IRT', 'once_key' => string (queued once per visitor)
// Lower level (ServerEventQueue::track_event()): 'uid' => stable id [a-z0-9]{1,40} (the browser
// never sends two items with the same uid), 'ref' => internal reference passed to
// `wisetrack_server_events_acked`, 'sid' => queue for another visitor (e.g. from a webhook).

wisetrack_track_screen( string $name, array $args = [] ): bool;
// $args: 'display_name', 'type' (other|page|dialog|bottom_sheet|webview), 'params', 'once_key'

wisetrack_is_tracking_enabled(): bool; // configured and the current user's role is not excluded
```

```php
add_action( 'user_register', function ( $user_id ) {
	wisetrack_track_event( 'sign_up', [ 'method' => 'form' ], [ 'once_key' => 'signup:' . $user_id ] );
} );

add_action( 'my_plugin_order_paid', function ( $order ) {
	wisetrack_track_event( 'purchase', [ 'items' => $order->count ], [
		'revenue'  => $order->total,
		'currency' => 'IRT',
		'once_key' => 'order:' . $order->id,
	] );
} );
```

The functions return `false` when nothing was queued: tracking disabled, invalid name, duplicate `once_key`, or no visitor cookie could be set (headers already sent — call them before output). Consent and Do Not Track are decided in the browser: queued items wait until the SDK may run. Queue limits: 50 items and 24 h per visitor; unacknowledged items are re-delivered after 2 minutes.

### 16.7 Hooks

#### Filters

| Filter | Arguments | Use |
|---|---|---|
| `wisetrack_should_load` | `bool $load, ?string $reason` | Load the runtime on this request. Keep it URL/role dependent. |
| `wisetrack_sdk_config` | `array $config` | Runtime config printed into the page (no visitor data!). |
| `wisetrack_sdk_src` | `string $src, string $source, string $version` | SDK URL (local/jsDelivr/unpkg). |
| `wisetrack_app_framework` | `string $framework` | `appFrameWork` sent to the SDK (default `wordpress`). |
| `wisetrack_wrapper_version` | `string $version` | `ResourceWrapper.sdkVersion()` value (default: plugin version `WISETRACK_VERSION`). |
| `wisetrack_page_context` | `array $context` | Page context (name, displayName, params, type…). |
| `wisetrack_screen_name` | `?string $name, string $page_type, string $naming` | Screen name before sanitizing (`null` = SDK default URL). |
| `wisetrack_rules_for_request` | `array $rules, array $context` | Event rules printed for this URL (`$context`: `post_id`, `post_type`, `page_type`, `path`, `is_front_page`). |
| `wisetrack_event_params` | `array $params, string $name, string $source` | Params of server events (`$source` = `server`). |
| `wisetrack_allow_param_value` | `bool $allow, string $value, string $key` | Allow a value that looks like an e-mail/phone (server side). |
| `wisetrack_max_event_params` | `int $max` | Max params per event (default 25). |
| `wisetrack_sensitive_query_params` | `string[] $params` | Query params never sent with page views or read by tokens. |
| `wisetrack_currency_map` | `array $map` | `{CODE: {to, factor}}` for currencies the SDK does not support. |
| `wisetrack_revenue_amount` | `mixed $amount, string $name, array $args` | Revenue of server events (WooCommerce purchase: amount in the order currency; `$args` has `order`, `order_id`, `basis`). |
| `wisetrack_order_note` | `bool $add, WC_Order $order` | Add the private order note “WiseTrack: purchase event sent.” (default true). |
| `wisetrack_is_tracking_enabled` | `bool $enabled` | Server-side tracking decision for the current visitor. |
| `wisetrack_queue_store` | `?QueueStore $store` | Replace the queue storage (`WiseTrack\WP\Tracking\Queue\QueueStore`; WooCommerce sites use its session table). |
| `wisetrack_pending_triggers` | `string[] $events` | DOM/jQuery events after which the runtime pulls queued server events (contains the success signals of enabled form integrations; add e.g. `added_to_cart`). |
| `wisetrack_pending_rate_limits` | `array{sid:int, ip:int} $limits` | Requests per minute for `/pending/*`. |
| `wisetrack_pending_allowed_hosts` | `string[] $hosts` | Hosts (`host[:port]`) allowed as Origin/Referer for `/pending/*` and `/woo/cart-summary`. |
| `wisetrack_client_ip` | `string $ip` | Client IP for rate limiting (behind a trusted proxy). |
| `wisetrack_cart_summary_rate_limit` | `int $limit` | Requests per minute and IP for `/woo/cart-summary` (default 60). |
| `wisetrack_builder_preview_params` | `array $params` | Query params of page-builder editor iframes (no SDK there). |
| `wisetrack_integrations` | `IntegrationInterface[] $list` | Add/remove integrations ([§16.12](#1612-writing-an-integration)). |
| `wisetrack_integration_events` | `array $events, string $slug` | Default events of an integration (`key => {name, label, type?, enabled?}`). |
| `wisetrack_post_types` | `string[] $types` | Post types with the per-post WiseTrack settings. |
| `wisetrack_capability` | `string $cap` | Capability for all WiseTrack admin screens/REST (default `manage_wisetrack`). |
| `wisetrack_show_admin_bar` | `bool $show` | WiseTrack admin bar menu. |
| `wisetrack_docs_urls` | `array $urls` | Documentation links in the admin. |
| `wisetrack_consent_plugins` | `array $plugins` | Consent plugins detected by the setup wizard. |
| `wisetrack_activation_redirect` | `bool $redirect` | Redirect to the setup wizard after activation. |
| `wisetrack_is_production_site` | `bool $production` | Production detection for the Sandbox reminder. |
| `wisetrack_naming_preview_samples` | `array $samples` | Sample pages of the naming preview. |
| `wisetrack_migrate_settings` | `array $settings, int $from` | Runs after the built-in migrations, for the stored option and imported files ([§18.2](#182-settings-migrations)). |
| `wisetrack_purge_page_cache` | `bool $purge` | Purge full-page caches after rules/settings changed (default true). |
| `wisetrack_preload_sdk` | `bool $preload` | Print `<link rel="preload">` for the SDK (default false). |
| `wisetrack_updater_enabled` | `bool $enabled` | Private build: check GitHub Releases for updates. |
| `wisetrack_update_repository` | `string $url` | Private build: GitHub repository with the releases. |
| `wisetrack_updater_library` | `string $path` | Private build: path of Plugin Update Checker's loader. |

#### Actions

| Action | Arguments | When |
|---|---|---|
| `wisetrack_loaded` | `Plugin $plugin` | Services registered (`plugins_loaded`). |
| `wisetrack_integrations_registered` | `array $integrations, Registry $registry` | The integration list is known. |
| `wisetrack_server_event_queued` | `array $item, string $sid` | A server event/screen was queued for a visitor. |
| `wisetrack_server_events_acked` | `array $items, string $sid` | The browser confirmed items (handed to the SDK); items include their internal `ref`. |
| `wisetrack_rules_changed` | — | Event rules were created, updated, deleted or imported. |
| `wisetrack_page_cache_purged` | — | WiseTrack asked the known caching plugins to purge — purge other caches or your CDN here. |
| `wisetrack_upgraded` | `int $from, int $to` | Stored settings were migrated to a new schema version. |

#### Examples

```php
// Add a param to every server event.
add_filter( 'wisetrack_event_params', function ( $params, $name ) {
	$params['site_section'] = 'shop';
	return $params;
}, 10, 2 );

// A rule only for logged-out-cacheable landing pages, defined in code.
add_filter( 'wisetrack_rules_for_request', function ( $rules, $context ) {
	if ( 0 === strpos( $context['path'], '/landing' ) ) {
		$rules[] = [
			'id'      => 'landing_cta',
			'name'    => 'landing_cta_click',
			'trigger' => [ 'type' => 'click', 'selector' => '.landing .cta' ],
			'params'  => [ 'text' => '{{element.text}}' ],
		];
	}
	return $rules;
}, 10, 2 );

// Pull queued server events after your own AJAX form.
add_filter( 'wisetrack_pending_triggers', function ( $events ) {
	$events[] = 'my-form:success'; // document.dispatchEvent( new Event( 'my-form:success' ) )
	return $events;
} );
```

### 16.8 Runtime rule format

Rules printed in `WiseTrackWP.config.rules` (and accepted from `wisetrack_rules_for_request`):

```jsonc
{
	"id": "r12",                       // unique; also the frequency storage key
	"name": "click_buy_now",
	"trigger": {
		"type": "click",                // click | submit | visible | page_load | scroll_depth | time_on_page | dom_event
		"selector": "#hero .btn",       // click / submit / visible
		"exact": true,                  // optional: no match from descendants
		"threshold": 50,                // visible % | scroll % | seconds
		"event": "my-theme:signup"      // dom_event: its `detail` object is merged into params
	},
	"params": { "button_text": "{{element.text}}" },
	"revenue": { "source": "fixed|attr|text|token", "value": "490000", "currency": "IRT" },
	"frequency": "once_per_session"    // always | once_per_page | once_per_session | once_per_visitor
}
```

Invalid selectors are ignored (shown in the debug overlay); selectors are only used with `closest()` / `querySelectorAll()`.

### 16.9 REST API

Namespace `wisetrack/v1`. Admin routes need `manage_wisetrack` and a `wp_rest` nonce.

| Route | Access | Purpose |
|---|---|---|
| `GET/POST settings` | Admin | Read / save settings (`?reveal=1` shows secrets) |
| `GET/POST rules`, `GET/PUT/PATCH/DELETE rules/<id>` | Admin | Rule CRUD |
| `POST rules/bulk` | Admin | `enable` · `disable` · `delete` · `duplicate` · `export` |
| `POST rules/validate` | Admin | Normalized rule, field errors, warnings — nothing saved |
| `GET integrations`, `GET integrations/<slug>/forms` | Admin | Integrations and their forms |
| `GET preview/page-names` | Admin | Naming preview |
| `tools/export`, `tools/import`, `tools/reset`, `tools/system-info` | Admin | Tools ([§17](#17-tools-export-import-reset-system-info)) |
| `POST notices/dismiss` | Admin | Dismiss a notice |
| `POST pending/claim`, `POST pending/ack` | Public by design | Server-queue delivery from cached pages: HttpOnly visitor cookie, same-origin check, rate limits, size caps |
| `GET woo/cart-summary` | Public, same-origin | `{item_count, cart_total, currency}` of the visitor's own cart, `no-store`; keeps the cookie user (like the Store API) so a logged-in customer's session is never reset |

### 16.10 Visual picker links

The picker opens on any front-end URL with `?wisetrack_picker=1&_wpnonce=<nonce>` (nonce action `wisetrack_picker`, bound to the logged-in user) for users with `manage_wisetrack`; any other request is redirected to the same URL without these params. Build links with `WiseTrack\WP\Admin\Picker::url( $url, $extra )`:

| Param | Meaning |
|---|---|
| `wisetrack_rule=<id>` | Open the rule for editing (its selector is replaced by the next pick). |
| `wisetrack_mode=existing` | Start with *Show existing events* on. |
| `wisetrack_origin=<token>` | Token of the admin tab that opened the picker (echoed in messages). |

Picker pages load neither the runtime nor the SDK, send `nocache_headers()` and `X-Robots-Tag: noindex`, define `DONOTCACHEPAGE` and hide the admin bar. After a save (or disable) the picker posts `{ type: 'wisetrack:rule-saved', rule, origin }` on `BroadcastChannel( 'wisetrack' )` and to `window.opener` (target origin = admin origin), so open Events screens refresh.

### 16.11 wp-config constants (private build)

```php
define( 'WISETRACK_DISABLE_UPDATER', true );                 // no update checks
define( 'WISETRACK_UPDATE_REPOSITORY', 'https://github.com/acme/wisetrack-wordpress/' ); // a fork
define( 'WISETRACK_UPDATER_TOKEN', 'github_pat_…' );         // private repository (read-only token)
```

### 16.12 Writing an integration

```php
use WiseTrack\WP\Integrations\AbstractIntegration;

class My_Booking_Integration extends AbstractIntegration {
	public function slug(): string { return 'my_booking'; }
	public function label(): string { return 'My Booking'; }
	public function category(): string { return 'forms'; }
	public function description(): string { return __( 'Confirmed bookings.', 'my-plugin' ); }
	public function is_available(): bool { return defined( 'MY_BOOKING_VERSION' ); }

	public function default_events(): array {
		return array( 'booked' => array( 'name' => 'booking_confirmed', 'label' => 'Booking confirmed' ) );
	}

	public function register(): void { // Only called when active and switched on.
		add_action( 'my_booking_confirmed', function ( $booking ) {
			$this->queue_event( 'booked', array( 'service' => $booking->service_slug ), array( 'once_key' => 'booking:' . $booking->id ) );
		} );
	}

	public function frontend_payload(): array { // Never visitor data (pages are cached); `pending` = client success signals.
		return array( 'pending' => array( 'my_booking_done' ) );
	}
}

add_filter( 'wisetrack_integrations', function ( $list ) {
	$list[] = new My_Booking_Integration();
	return $list;
} );
```

`frontend_payload()` ends up in `WiseTrackWP.config.integrations[slug]` (its `pending` list in `config.pending`); it may depend on the URL, never on the visitor. Override `needs_frontend_script()` when the page needs `build/integrations.js`. Form plugins can extend `WiseTrack\WP\Integrations\Forms\FormIntegration` and call `$this->submitted( $form_id, $form_name, $fields, $submission_id )`.

---

# Part III — Operations

## 17. Tools: export, import, reset, system info

| Tool | Description |
|---|---|
| **Export** | JSON file with settings and rules; credentials only if you tick the box. |
| **Import** | Preview of differences (secrets masked), count of valid/invalid rules, then apply. Max 2 MB and 1 000 rules. Files from older versions are migrated; files from newer versions are refused. |
| **Reset** | Restore default settings (credentials kept by default). |
| **Debug overlay** | Toggle for this browser. |
| **System info** | Copy versions, related plugins and settings (without secrets) for support. |

**Staging → production:** export on staging (without credentials) → import on production → enter production credentials → verify in Sandbox → switch to Production.

---

## 18. Updates, migrations and uninstall

### 18.1 Updates

- **Private build** (`wisetrack-<version>-private.zip` from GitHub Releases) updates itself through [Plugin Update Checker](https://github.com/YahnisElsts/plugin-update-checker) (MIT, bundled in `lib/`, version-namespaced) and shows updates in *Dashboard › Updates* like any plugin. Constants: [§16.11](#1611-wp-config-constants-private-build).
- **wp.org build:** updated by WordPress.org; contains no updater and no CDN option.

### 18.2 Settings migrations

Settings carry a schema version (`Schema::VERSION`, currently **3**). The installed version of each site is stored in the option `wisetrack_db_version`; on the first request after an update the stored settings are migrated step by step (also for updates that never run the activation hook) and `wisetrack_upgraded` fires. Imported files run the same steps: files from older versions import, files from a newer version are refused.

| Version | Change |
|---|---|
| 1 → 2 | Site search and 404 moved from `smart_events.search` / `page_not_found` (0.1 builds) to the `wordpress` integration. |
| 2 → 3 | Currency map target `EHT` → `ETH` (1.0.1). |

```php
// Your own migration on top of the built-in ones.
add_filter( 'wisetrack_migrate_settings', function ( $settings, $from ) {
	if ( $from < 2 && isset( $settings['naming'] ) && 'url' === $settings['naming'] ) {
		$settings['naming'] = 'path';
	}
	return $settings;
}, 10, 2 );
```

### 18.3 Deactivate / uninstall

- **Deactivate:** tracking stops; settings and rules are kept.
- **Delete:** data is removed only when *Advanced › Remove data on uninstall* is on (settings, rules, queues, capability).

---

## 19. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No data in the panel | Browsing as an administrator (excluded) | Test logged out / in a private window |
| | Environment mismatch | Panel and plugin both on Sandbox, or both on Production |
| | Waiting for consent | Accept the banner; `WiseTrackWP.status()` returns `waiting_consent` |
| | Domain not registered | Register it for the app (`Application not found`) |
| `WiseTrackWP` is undefined | Excluded path/role, *Do not track*, builder preview URL, or an optimizer removed the script | Check exclusions; enable *Optimization plugin compatibility*; clear caches |
| Status `blocked` | Ad blocker / privacy extension or CSP | Test without extensions; add `*.wisetrack.io` to `connect-src` |
| Old rules still run | Page or CDN cache | Purge the CDN (supported page caches are purged automatically) |
| Event missing or renamed | Sanitizer normalized or dropped it | Debug overlay shows before/after and the drop reason |
| Param missing | Looks like PII, > 25 params, refused/empty token | Rename the param / change the value |
| Purchase missing | Buyer never returned after a delayed payment | Delivered on the next visit; check the order note |
| Purchase without revenue | Non-ISO currency without mapping | Add a *Currency conversion* row ([§11.3](#113-woocommerce)) |
| Form event missing | Failed validation / spam, or the form is switched off | Test a valid submission; check *Settings › Integrations › Forms* |
| PHP API returns `false` | Called after output, duplicate `once_key`, tracking disabled | Call before output; check `wisetrack_is_tracking_enabled()` |
| Element not pickable | Inside an iframe | Use attributes in the iframe's own page |
| Low selector stability | Generated / dynamic classes | Add `data-wt-event` or a stable `id` |
| Site Health: Sandbox on production | Environment left on Sandbox | Switch to Production ([§6](#6-going-live-sandbox--production)) |

Still stuck? Send WiseTrack support the output of **Tools › System info** and the debug overlay JSON.

---

## 20. FAQ

**Is it GDPR-friendly?** The plugin gives you the tools — consent mode (the SDK is not even loaded before consent), no PII, DNT/GPC, sensitive-parameter stripping. Whether your setup is compliant depends on your site and policy.

**Does it work with caching plugins?** Yes. Pages contain only URL-dependent data; server events reach the browser through a cookie-flagged REST pull.

**Are administrators tracked?** No, by default (*Visitors & privacy › Excluded roles*).

**My shop uses Toman. Is revenue correct?** Yes — Toman is converted to Rial (×10) before sending; *Settings › Integrations › WooCommerce* shows an example with your own currency.

**Purchases are counted twice / missing?** Purchases are sent once per order from the buyer's browser after payment (or on the thank-you page). If the buyer never returns after a gateway that confirms later, it is delivered on their next visit. The order note “WiseTrack: purchase event sent.” confirms delivery.

**Can I track without the Events screen?** Yes — attributes, classes, shortcodes, the JavaScript or PHP API ([§16](#16-developer-reference)).

**Does it slow my site down?** No measurable LCP/CLS impact: the runtime is < 12 KB and the SDK loads asynchronously after render.

---

## 21. Development

> This section is for working on the plugin's source code.

### 21.1 Environment

Node 24 (`.nvmrc`), PHP 7.4+ with Composer, Docker (for `@wordpress/env`).

```bash
composer install && npm install
npm run build            # webpack (runtime, picker, debug, admin, editor, integrations) + size budget
npm start                # watch mode
npm run env:start        # wp-env: development site :8888, tests site :8889 (admin / password)
```

The development site activates WooCommerce, Elementor, Contact Form 7 and WPForms Lite. The tests site loads `tests/e2e/mu-plugins/wisetrack-test.php`, which replaces the SDK with `tests/fixtures/mock-sdk.js` (records calls on `window.__wtCalls`).

### 21.2 Source layout

| Path | Content |
|---|---|
| `wisetrack.php` | Bootstrap, own PSR-4 autoloader for `WiseTrack\WP\` → `includes/` |
| `includes/` | PHP (settings, tracking, rules, integrations, admin, REST) |
| `src/` | TypeScript: `runtime/` (visitors), `picker/`, `debug/`, `admin/`, `editor/`, `integrations/`, `shared/` (used by PHP/JS parity tests) |
| `languages/` | POT, Persian PO/MO/JSON, glossary |
| `assets/vendor/wisetrack-sdk/` | SDK bundle from npm (`bin/sync-sdk.sh <version>`) |
| `docs/` | Spec, SDK notes, progress log, release process (internal) |

### 21.3 Tests and quality

```bash
npm run lint             # ESLint, stylelint, tsc, PHPCS (WordPress-Extra/Docs/Security, PHPCompatibility)
composer phpstan         # PHPStan level 6
composer test            # PHPUnit unit tests (no WordPress)
npm run test:unit        # Jest
npm run test:php         # PHPUnit integration tests inside wp-env (WooCommerce, CF7, WPForms, Elementor loaded)
npm run test:e2e         # Playwright against the wp-env tests site
```

The PHP examples of [§16](#16-developer-reference) and [§18.2](#182-settings-migrations) are executed by an integration test (`DeveloperDocsTest`) — keep every `php` block in this file runnable (signature lists and `define()` blocks are skipped).

Other environments: `node bin/wp-env-matrix.mjs 'WordPress/WordPress#6.2' 7.4 none` writes `.wp-env.override.json`, then `npm run env:start`. `bin/multisite-smoke.sh` converts the development site into a network (throwaway environments only).

### 21.4 Translations

`bin/make-pot.sh` regenerates `languages/wisetrack.pot`, updates the `.po` files and builds `.mo` and script JSON files. Persian terminology: `languages/GLOSSARY.md`.

### 21.5 Versions and release

```bash
bin/sync-sdk.sh 2.3.2                              # refresh the bundled SDK (published npm package only)
bin/bump-version.sh --plugin 1.0.2 --sdk 2.3.3     # bump plugin and/or SDK version everywhere
bin/build-zip.sh private                           # or: wporg → dist/wisetrack-<version>-<flavor>.zip
bin/release.sh                                     # checks, both zips, publish to GitHub Releases
```

| Build | Distribution | Updates | SDK source |
|---|---|---|---|
| `private` | GitHub Releases ([`wisetrack-io/wordpress-plugin`](https://github.com/wisetrack-io/wordpress-plugin)) | Built-in updater | Local (default), jsDelivr or unpkg |
| `wporg` | WordPress.org | WordPress.org | Local only |

`bin/build-zip.sh` checks version consistency, the SDK checksum, PHP syntax and that no dev files (or, for `wporg`, no updater/CDN code) are included. The release steps, Plugin Check and the public repository setup are described in `docs/RELEASE.md`.

---

## 22. Appendices

### 22.1 Settings keys (`wisetrack_settings`)

| Key | Default | Values |
|---|---|---|
| `app_token`, `client_secret` | `''` | |
| `environment` | `production` | `sandbox` / `production` |
| `source` | `local` | `local` / `jsdelivr` / `unpkg` (CDN: private build only) |
| `app_version` | `1.0.0` | |
| `log_level` | `warn` | `debug` / `info` / `warn` / `error` / `none` |
| `tracking_delay` | `0` | seconds |
| `deeplink_enabled` | `true` | |
| `load_strategy` | `defer` | `defer` / `idle` |
| `auto_track`, `track_dialogs` | `true` | |
| `naming` | `path` | `path` / `template` / `url` |
| `excluded_paths`, `excluded_patterns` | `[]` | prefixes / regex |
| `sensitive_params` | WP/Woo defaults | |
| `context_params` | `page_type, post_type, post_id` | |
| `excluded_roles` | `['administrator']` | |
| `respect_dnt` | `false` | |
| `exclude_bots` | `true` | |
| `smart_events` | all `false` | `outbound, download, tel, mailto, whatsapp, telegram, scroll_depth, time_on_page` |
| `download_extensions` | `pdf,zip,rar,docx,xlsx,pptx,mp3,mp4,apk` | |
| `outbound_delay_ms` | `0` | 0–500 |
| `attributes_enabled` | `true` | |
| `include_search_term` | `false` | |
| `integrations` | `{}` | `{slug: {enabled, events, forms}}` |
| `purchase_trigger` | `paid` | `paid` / `thankyou` |
| `revenue_basis` | `total` | `total` / `subtotal` |
| `currency_map` | `{}` | `{"XYZ": {"to": "USD", "factor": 1}}` |
| `consent_mode` | `off` | `off` / `wp_consent_api` / `custom` |
| `consent_category` | `statistics` | |
| `optimization_compat` | `true` | |
| `remove_data_on_uninstall` | `false` | |
| `custom_screen_rules` | `[]` | `[{pattern, name, display_name}]` |

### 22.2 Cookies and browser storage

| Name | Type | Purpose |
|---|---|---|
| `wisetrack_sid` | HttpOnly cookie, 1 day | Key of the visitor's server queue |
| `wisetrack_pending` | Cookie | “Queued items exist” flag |
| `wisetrack_outbox` | sessionStorage | Events awaiting delivery across navigation (max 50, 10 min) |
| `wisetrack_freq_<id>` | session/localStorage | Frequency limits |
| `wisetrack_srv` | localStorage | Last 50 server-event uids handed to the SDK |
| `wisetrack_consent` | localStorage | Consent choice (`custom` mode) |
| `wisetrack_debug` | localStorage | Debug overlay flag |
| SDK keys | localStorage | The WiseTrack SDK's own storage |

### 22.3 Glossary

| Term | Meaning |
|---|---|
| Screen / page view | A page (or dialog) view sent to WiseTrack |
| Rule | A no-code event definition (trigger + target + where + data) |
| Smart events | Built-in automatic events (outbound, downloads, contact links, scroll, time) |
| Server event queue | Per-visitor queue for events that happen in PHP |
| Sandbox / Production | WiseTrack environments for test and real data |
| Sanitizer | Validation layer that mirrors the SDK's rules so no event is silently dropped |

### 22.4 More

- [`CHANGELOG.md`](CHANGELOG.md) — version history
- [`readme.txt`](readme.txt) — WordPress.org readme

## License

GPL-2.0-or-later (see [`LICENSE`](LICENSE)). The bundled WiseTrack Web SDK is licensed to this plugin under GPL-2.0-or-later by WiseTrack (`assets/vendor/wisetrack-sdk/LICENSE`). Questions and bug reports: WiseTrack support.
