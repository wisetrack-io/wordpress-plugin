=== WiseTrack ===
Contributors: wisetrack
Tags: analytics, attribution, woocommerce, events, tracking
Requires at least: 6.2
Tested up to: 7.1
Requires PHP: 7.4
Stable tag: 1.0.0
License: GPLv2 or later
License URI: https://www.gnu.org/licenses/gpl-2.0.html

WiseTrack analytics and attribution for WordPress: automatic page views, no-code events, WooCommerce revenue and form tracking.

== Description ==

WiseTrack connects your WordPress site to [WiseTrack](https://wisetrack.io/), a mobile and web analytics and attribution platform. The plugin loads the official WiseTrack Web SDK and sends page views and events from your visitors' browsers to your WiseTrack app.

Everything works without code — and developers get attributes, shortcodes, a JavaScript and a PHP API when they need more.

= Features =

* **Setup wizard** — connect your app with the App Token and Client Secret from the WiseTrack panel, test the connection, choose how pages are named and which plugins to track. Ready in minutes.
* **Automatic page views** with clean names (by path, by template or by full URL), page type and post context, multilingual sites (WPML, Polylang) and popups as dialog screens.
* **Visual event picker** — open any page of your site, click a button, a link or a form, name the event and save. No selectors, no code.
* **Event rules** — clicks, form submits, elements becoming visible, scroll depth, time on page, page load and custom DOM events, with parameters from the page (element text, attributes, URL, page context), revenue and frequency limits (once per page, session or visitor).
* **Smart events** — outbound links, downloads, phone, email, WhatsApp and Telegram links, scroll depth and time on page with one switch each.
* **WooCommerce** — `view_item`, `add_to_cart`, `remove_from_cart`, `view_cart`, `begin_checkout` and `purchase` with revenue, exactly once per order, also when the payment is confirmed later by a gateway webhook. Classic and block cart/checkout, HPOS compatible. Toman (IRT) prices are converted to Rial (IRR) automatically.
* **Forms** — Contact Form 7, WPForms, Gravity Forms, Elementor Pro Forms and Fluent Forms submissions, with per-form names and parameters.
* **WordPress events** — login, sign-up, comments, site search and 404 pages.
* **Block editor and Elementor panels** — attach an event to any block or widget in the editor you already use.
* **Built for real sites** — works with full-page caches (WP Rocket, LiteSpeed Cache, W3 Total Cache, SiteGround Optimizer, Cloudflare…) and optimization plugins, loads asynchronously, adds less than 12 KB of JavaScript (gzipped) besides the SDK, no jQuery.
* **Privacy controls** — consent mode (WP Consent API or your own banner), Do Not Track / Global Privacy Control, bot filtering, excluded roles, pages and query parameters, a PII guard that drops e-mail addresses and phone numbers from event data.
* **Debug overlay** — see every page view and event with its final payload, what was changed or dropped and why, right on your site.
* **Fully translated to Persian** (fa_IR) with a right-to-left interface.

= For developers =

`data-wt-*` attributes and CSS classes, `[wisetrack_event]` and `[wisetrack_screen]` shortcodes, `window.WiseTrackWP` / `wisetrackQueue` in JavaScript, `wisetrack_track_event()` in PHP and more than 40 filters and actions. See the [developer guide](https://github.com/wisetrack-io/wordpress-plugin/blob/main/docs/DEVELOPER.md).

== External services ==

This plugin connects your site to the **WiseTrack** analytics service. It cannot work without it: tracking data is the purpose of the plugin.

* **What is sent and when:** once you enter your App Token and Client Secret, every visitor's browser loads the WiseTrack Web SDK and sends page views (page name, path, page type and the context parameters you chose), the events you configured (with their parameters and revenue), and technical data the SDK collects for attribution (a random device/session id, browser and operating system, screen size, language, referrer and campaign parameters). Nothing is sent while consent mode is on and the visitor has not consented, when Do Not Track / Global Privacy Control is respected, for bots and for excluded roles. Your site's server does not send visitor data to WiseTrack.
* **Where:** WiseTrack servers (`*.wisetrack.io`).
* **Connection test:** the setup wizard sends your credentials to WiseTrack from your browser to check them.
* WiseTrack [Terms of Service](https://wisetrack.io/terms/) and [Privacy Policy](https://wisetrack.io/privacy/).

The WiseTrack Web SDK is bundled with the plugin and always loaded from your own site; the plugin loads no scripts from third-party CDNs.

== Installation ==

1. Install and activate the plugin (*Plugins › Add New* or upload the zip).
2. The setup wizard opens. Paste the **App Token** and **Client Secret** of your app from the WiseTrack panel and click **Test connection**.
3. Choose how pages are named, which plugins to track and your privacy settings.
4. Open any page of your site and look at the WiseTrack sandbox — page views arrive within a few minutes.
5. Create events in *WiseTrack › Events* or with the visual picker (admin bar › *WiseTrack › Pick an element to track*).

== Frequently Asked Questions ==

= Is it GDPR-friendly? =

WiseTrack gives you the tools; whether your setup is compliant depends on your site. Turn on **consent mode** in *Settings › Visitors & Privacy*: with the WP Consent API (Complianz, CookieYes, Real Cookie Banner, Cookiebot…) or with your own banner calling `WiseTrackWP.consent( true )`, the SDK is not even loaded before the visitor agrees. The plugin never sends WordPress user ids, names, e-mail addresses or phone numbers, drops values that look like personal data from event parameters, ignores sensitive query parameters (order keys, tokens, e-mail…) and can respect Do Not Track / Global Privacy Control. Mention WiseTrack in your privacy policy (WordPress' privacy policy guide includes a suggested text).

= Does it work with caching plugins? =

Yes. Pages contain only data that depends on the URL — never visitor ids, nonces or queued events — so they can be cached for everybody. Events that happen on the server (purchases, form submissions, logins) wait in a small per-visitor queue and are picked up by the browser on the next page view. The plugin keeps WP Rocket, LiteSpeed Cache, Autoptimize, SiteGround Optimizer, W3 Total Cache, Perfmatters, FlyingPress and Cloudflare Rocket Loader from delaying or combining its scripts, and purges their page cache when you change events or settings.

= My shop uses Toman. Is revenue correct? =

Yes. WiseTrack expects ISO currency codes; Toman is not one. The plugin recognizes Toman shops (code `IRT`, `TOMAN` or the symbol «تومان») and converts amounts to Rial (`IRR`, × 10) before sending. *Settings › Integrations › WooCommerce* shows an example with your own currency. Other custom codes can be mapped in the same section (*Currency map*).

= Purchases are counted twice / missing =

Purchases are sent once per order, from the buyer's browser, after the payment is confirmed (or on the thank-you page, if you choose that). If the buyer never returns to your site after paying through a gateway that confirms later, the purchase is delivered on their next visit. Check the order notes: "WiseTrack: purchase event sent." appears when the event left the browser.

= How do I see what is sent? =

Click *WiseTrack › Debug overlay: off* in the admin bar to switch the overlay on for your browser, then open any page. The overlay lists every page view and event with its payload, and what was changed or dropped by validation.

= Are administrators tracked? =

No, administrators are excluded by default (*Settings › Visitors & Privacy › Excluded roles*).

= Does it slow my site down? =

The plugin's own script is under 12 KB gzipped, loads with `defer` in the footer and adds no blocking time. It then loads the WiseTrack SDK (about 50 KB gzipped) asynchronously, after the page has been rendered, so loading speed and layout stay the same (Lighthouse: identical LCP and CLS with and without the plugin). On slow phones the SDK's start-up uses about 0.1 s of processor time once the page is visible.

== Screenshots ==

1. Setup wizard — connect your WiseTrack app and test the connection.
2. Dashboard — connection status, SDK and what is tracked.
3. Events — rules list with filters, presets and WooCommerce events.
4. Event editor — trigger, target, conditions, parameters with live preview, revenue and frequency.
5. Visual picker — click an element on your site to track it.
6. Settings — page naming, privacy and integrations.
7. Debug overlay — every event with its final payload.
8. The admin in Persian (right-to-left).

== Changelog ==

= 1.0.0 =
* First stable release.
* Automatic page views, event rules, visual picker, smart events, debug overlay.
* WooCommerce funnel with revenue (IRT → IRR), forms (Contact Form 7, WPForms, Gravity Forms, Elementor Pro, Fluent Forms), popups, WordPress core events.
* Block editor and Elementor panels, attributes, shortcodes, JavaScript and PHP API.
* Consent mode, caching and optimization plugin compatibility, multisite support, Persian translation.

== Upgrade Notice ==

= 1.0.0 =
First stable release. Settings from 0.x test builds are migrated automatically.
