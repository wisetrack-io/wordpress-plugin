# WiseTrack for WordPress — Developer guide

Everything you can do **without** the Events screen: HTML attributes, CSS classes, shortcodes, a
JavaScript API with an early-call queue, a PHP API and WordPress hooks — plus what the
integrations (WordPress core, forms, popups) and page builders send ([§12](#12-integrations),
[§13](#13-page-builders)).

> **Rules that apply everywhere**
>
> - Event names and param keys are normalized to `[a-z0-9_]` (≤ 50 chars, starting with a letter):
>   `"Create Account"` → `create_account`. Use Latin names; put other languages in labels only.
> - Param values are sanitized so the SDK never drops them (≤ 100 chars, quotes/`%`/`#`/HTML removed,
>   SQL words neutralized: `"Update cart"` → `update_cart`). Max 25 params per event
>   (`wisetrack_max_event_params`).
> - **Values that look like emails or phone numbers are dropped.** Never send personal data,
>   WordPress user ids, names or form field values that identify a person.
> - Pages are often cached: never print visitor-specific data into HTML. Use the PHP API instead
>   (it delivers events through a per-visitor queue).
> - Turn on the **debug overlay** to see every event, what the sanitizer changed and why something
>   was dropped: admin bar → *WiseTrack → Debug overlay: on*, or `?wisetrack_debug=1`, or
>   `localStorage.setItem( 'wisetrack_debug', '1' )`.

---

## 1. HTML attributes

Work in theme templates, block "HTML anchor/additional attributes", page-builder "custom attributes"
fields (Bricks, Breakdance, Oxygen…) and anywhere else you control the markup. Enabled by default
(*Settings → Integrations → Automatic events → HTML attributes and CSS classes*, key
`attributes_enabled`).

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
| `data-wt-once` | `page` · `session` · `visitor` — fire at most once per page view / browser session / visitor. |
| `data-wt-screen` | Manual screen name (e.g. tabs, steps); `click` by default, `data-wt-trigger="load"` on load. |
| `data-wt-screen-display` | Screen display name. |
| `data-wt-screen-type` | `other` (default) · `page` · `dialog` · `bottom_sheet` · `webview`. |
| `data-wt-no-track` | Opt a whole area out of tracking. |

Clicks inside an element count for the nearest ancestor with `data-wt-event`. Param values may
contain [tokens](#5-tokens), e.g. `data-wt-param-label="{{element.text}}"`.

## 2. CSS-class convention

For builders that only let you add classes (WPBakery, Divi, Beaver Builder…; step lists in
[§13](#13-page-builders)):

```html
<a class="button wt-ev-signup_click wt-p-plan-pro wt-p-position-hero">Sign up</a>
<section class="wt-ev-pricing_seen--visible">…</section>
```

- `wt-ev-<event_name>` → event on click.
- `wt-ev-<event_name>--visible` → event when the element becomes visible.
- `wt-p-<key>-<value>` → static param (`key` is everything up to the first `-`; simple values only).

## 3. Shortcodes

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

Reserved shortcode attributes: `name`, `trigger`, `once` (`page|session|visitor`), `revenue`,
`currency`, `params` (JSON), `display`, `type`. Every other attribute becomes a param.

## 4. JavaScript API

The runtime is loaded deferred in the footer. Calls made before it is ready go into a queue —
add this stub to your theme's `<head>` (or any earlier script) and use it anywhere:

```html
<script>
	window.wisetrackQueue = window.wisetrackQueue || [];
	wisetrackQueue.push( [ 'track', 'video_play', { video_id: 'intro' } ] );
</script>
```

Queue commands: `track`, `trackRevenue`, `trackScreen`, `consent`, `ready`, `on`, `off`,
`pullPending`; a function pushed to the queue runs on `ready`.

```js
WiseTrackWP.ready( ( api ) => { /* SDK initialized and enabled */ } );

WiseTrackWP.track( 'signup', { method: 'google' } );
WiseTrackWP.trackRevenue( 'purchase', 1250000, 'IRT', { order_id: 'A-1' } ); // currency optional
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

Events fired before the SDK is enabled (or from a click that navigates away) are kept in
`sessionStorage.wisetrack_outbox` (max 50, 10 minutes) and delivered on the next page.

## 5. Tokens

Param values of event rules, attributes and the Events editor can contain dynamic tokens, resolved
in the visitor's browser and then sanitized:

| Token | Value |
|---|---|
| `{{element.text}}` | Text of the element (trimmed, whitespace collapsed; buttons: their value). |
| `{{element.attr:NAME}}` | Attribute value (`value` of form fields is refused). |
| `{{element.data:NAME}}` | `data-NAME` attribute. |
| `{{element.href}}` | Link URL without query string and hash (`mailto:`/`tel:` refused). |
| `{{element.id}}` | Element id. |
| `{{page.path}}` · `{{page.title}}` · `{{page.post_id}}` · `{{page.post_type}}` · `{{page.type}}` | Page data (`page.type` = template slug such as `single_product`, `404`). |
| `{{url.query:NAME}}` | Query param (refused for sensitive params: `token`, `password`, `email`, the *Sensitive query params* setting…). |
| `{{form.id}}` · `{{form.field:NAME}}` | Form id/name and a field value. **Refused** for `password`, `email`, `tel`, `hidden` and `file` fields and for names matching `pass|mail|phone|mobile|tel|card|cvv|national|melli|nonce|token|secret`. |
| `{{scroll.percent}}` · `{{time.seconds}}` | Scroll depth / visible seconds on the page. |

A value that is exactly one token keeps its type (`{{page.post_id}}` stays a number). A refused or
missing token drops that param (visible in the debug overlay).

## 6. PHP API

For things that happen on the server (logins, sign-ups, form handlers, webhooks in the visitor's
request). Events are queued for the current visitor (cookie `wisetrack_sid`) and delivered by the
browser on their next page view — or immediately after an AJAX success event registered with
`wisetrack_pending_triggers`, or when your JS calls `WiseTrackWP.pullPending()`.

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

The functions return `false` when nothing was queued: tracking disabled, invalid name, duplicate
`once_key`, or no visitor cookie could be set (headers already sent — call them before output).
Consent and Do Not Track are decided in the browser: queued items wait until the SDK may run.
Queue limits: 50 items and 24 h per visitor; unacknowledged items are re-delivered after 2 minutes.

## 7. Hooks

### Filters

| Filter | Arguments | Use |
|---|---|---|
| `wisetrack_should_load` | `bool $load, ?string $reason` | Load the runtime on this request. Keep it URL/role dependent. |
| `wisetrack_sdk_config` | `array $config` | Runtime config printed into the page (no visitor data!). |
| `wisetrack_sdk_src` | `string $src, string $source, string $version` | SDK URL (local/jsDelivr/unpkg). |
| `wisetrack_app_framework` | `string $framework` | `appFrameWork` sent to the SDK. |
| `wisetrack_wrapper_version` | `string $version` | `ResourceWrapper.sdkVersion()` value (default: plugin version `WISETRACK_VERSION`). |
| `wisetrack_page_context` | `array $context` | Page context (name, displayName, params, type…). |
| `wisetrack_screen_name` | `?string $name, string $page_type, string $naming` | Screen name before sanitizing (`null` = SDK default URL). |
| `wisetrack_rules_for_request` | `array $rules, array $context` | Event rules printed for this URL (`$context`: `post_id`, `post_type`, `page_type`, `path`, `is_front_page`). |
| `wisetrack_event_params` | `array $params, string $name, string $source` | Params of server events (`$source` = `server`). |
| `wisetrack_allow_param_value` | `bool $allow, string $value, string $key` | Allow a value that looks like an email/phone (server side). |
| `wisetrack_max_event_params` | `int $max` | Max params per event (default 25). |
| `wisetrack_sensitive_query_params` | `string[] $params` | Query params never sent with page views or read by tokens. |
| `wisetrack_currency_map` | `array $map` | `{CODE: {to, factor}}` for currencies the SDK does not support. |
| `wisetrack_revenue_amount` | `mixed $amount, string $name, array $args` | Revenue of server events (WooCommerce purchase: amount in the order currency, `$args` has `order`, `order_id`, `basis`). |
| `wisetrack_order_note` | `bool $add, WC_Order $order` | Add the private order note "WiseTrack: purchase event sent." (default true). |
| `wisetrack_is_tracking_enabled` | `bool $enabled` | Server-side tracking decision for the current visitor. |
| `wisetrack_queue_store` | `?QueueStore $store` | Replace the queue storage (`WiseTrack\WP\Tracking\Queue\QueueStore`; WooCommerce sites use its session table). |
| `wisetrack_pending_triggers` | `string[] $events` | DOM/jQuery events after which the runtime pulls queued server events (contains the success signals of enabled form integrations; add e.g. `added_to_cart`). |
| `wisetrack_pending_rate_limits` | `array{sid:int, ip:int} $limits` | Requests per minute for `/pending/*`. |
| `wisetrack_pending_allowed_hosts` | `string[] $hosts` | Hosts (`host[:port]`) allowed as Origin/Referer for `/pending/*` and `/woo/cart-summary`. |
| `wisetrack_client_ip` | `string $ip` | Client IP for rate limiting (behind a trusted proxy). |
| `wisetrack_builder_preview_params` | `array $params` | Query params of page-builder editor iframes (no SDK there). |
| `wisetrack_integrations` | `IntegrationInterface[] $list` | Add/remove integrations ([§12](#12-integrations)). |
| `wisetrack_integration_events` | `array $events, string $slug` | Default events of an integration (`key => {name, label, type?, enabled?}`). |
| `wisetrack_post_types` | `string[] $types` | Post types with the per-post WiseTrack settings. |
| `wisetrack_capability` | `string $cap` | Capability for all WiseTrack admin screens/REST (default `manage_wisetrack`). |
| `wisetrack_show_admin_bar` | `bool $show` | WiseTrack admin bar menu. |
| `wisetrack_docs_urls` | `array $urls` | Documentation links in the admin. |
| `wisetrack_consent_plugins` | `array $plugins` | Consent plugins detected by the setup wizard. |
| `wisetrack_activation_redirect` | `bool $redirect` | Redirect to the setup wizard after activation. |
| `wisetrack_is_production_site` | `bool $production` | Production detection for the sandbox reminder. |
| `wisetrack_naming_preview_samples` | `array $samples` | Sample pages of the naming preview. |
| `wisetrack_migrate_settings` | `array $settings, int $from` | Runs after the built-in migrations, for the stored option and for imported files ([§15](#15-upgrades-and-settings-migrations)). |
| `wisetrack_purge_page_cache` | `bool $purge` | Purge full-page caches after rules/settings changed (default true, [§10](#10-caching-csp-pwas)). |
| `wisetrack_preload_sdk` | `bool $preload` | Print `<link rel="preload">` for the SDK (default false: it delayed FCP/LCP by ~150 ms in Lighthouse). |
| `wisetrack_cart_summary_rate_limit` | `int $limit` | Requests per minute and IP for `/woo/cart-summary` (default 60). |
| `wisetrack_updater_enabled` | `bool $enabled` | Private build: check GitHub Releases for updates ([§14](#14-updates-private-build)). |
| `wisetrack_update_repository` | `string $url` | Private build: GitHub repository with the releases. |
| `wisetrack_updater_library` | `string $path` | Private build: path of Plugin Update Checker's loader. |

### Actions

| Action | Arguments | When |
|---|---|---|
| `wisetrack_loaded` | `Plugin $plugin` | Services registered (`plugins_loaded`). |
| `wisetrack_integrations_registered` | `array $integrations, Registry $registry` | The integration list is known. |
| `wisetrack_server_event_queued` | `array $item, string $sid` | A server event/screen was queued for a visitor. |
| `wisetrack_server_events_acked` | `array $items, string $sid` | The browser confirmed items (handed to the SDK); items include their internal `ref`. |
| `wisetrack_rules_changed` | — | Event rules were created, updated, deleted or imported. |
| `wisetrack_page_cache_purged` | — | WiseTrack asked the known caching plugins to purge (end of a request that changed rules/settings) — purge other caches or your CDN here. |
| `wisetrack_upgraded` | `int $from, int $to` | Stored settings were migrated to a new schema version. |

### Examples

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

## 8. Runtime rule format

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

Invalid selectors are ignored (shown in the debug overlay); selectors are only used with
`closest()`/`querySelectorAll()`.

## 9. REST API (admin)

All routes need the `manage_wisetrack` capability and a `wp_rest` nonce:
`GET/POST wisetrack/v1/rules`, `GET/PUT/DELETE wisetrack/v1/rules/<id>`,
`POST wisetrack/v1/rules/bulk` (`enable|disable|delete|duplicate|export`),
`POST wisetrack/v1/rules/validate` (normalized rule, field errors, warnings — nothing saved).

`POST wisetrack/v1/pending/claim` and `/pending/ack` are public by design (called from cached
pages): protected by the HttpOnly visitor cookie, a same-origin check, rate limits and size caps.
`GET wisetrack/v1/woo/cart-summary` (WooCommerce on) returns `{item_count, cart_total, currency}`
of the visitor's own cart, `no-store`, same-origin only; it keeps the cookie user (like the Store
API) so loading a logged-in customer's cart never resets their session.

## 10. Caching, CSP, PWAs

- Rules and settings are printed into the page. When they change, WiseTrack purges the page cache
  of WP Rocket, LiteSpeed Cache, W3 Total Cache, SiteGround Optimizer, WP Super Cache,
  FlyingPress, WP Fastest Cache, Hummingbird, Cache Enabler, Breeze and Nginx Helper once at the
  end of the request (`wisetrack_purge_page_cache` turns it off). Other caches and CDNs: hook into
  `wisetrack_page_cache_purged`.
- Optimization plugins (`optimization_compat`, on by default) are told not to delay, defer,
  combine or minify our scripts; the tags also carry `data-cfasync="false"` (Cloudflare Rocket
  Loader), `data-no-optimize`, `data-no-defer` and `data-no-minify`.
- CSP: allow `*.wisetrack.io` in `connect-src` (and `script-src` when loading the SDK from
  jsDelivr/unpkg).
- Service workers must not cache requests to `wisetrack.io` (use NetworkOnly) nor
  `/wp-json/wisetrack/v1/pending/*` and `/wp-json/wisetrack/v1/woo/*`.

## 11. Visual picker links

The picker opens on any front-end URL with `?wisetrack_picker=1&_wpnonce=<nonce>` (nonce action
`wisetrack_picker`, bound to the logged-in user) for users with `manage_wisetrack`; any other request
is redirected to the same URL without these params. Build links with
`WiseTrack\WP\Admin\Picker::url( $url, $extra )`:

| Param | Meaning |
|---|---|
| `wisetrack_rule=<id>` | Open the rule for editing (its selector is replaced by the next pick). |
| `wisetrack_mode=existing` | Start with "Show existing events" on. |
| `wisetrack_origin=<token>` | Token of the admin tab that opened the picker (echoed in messages). |

Picker pages load neither the runtime nor the SDK, send `nocache_headers()`, `X-Robots-Tag: noindex`,
define `DONOTCACHEPAGE` and hide the admin bar. After a save (or disable) the picker posts
`{ type: 'wisetrack:rule-saved', rule, origin }` on `BroadcastChannel( 'wisetrack' )` and to
`window.opener` (target origin = admin origin), so open Events screens refresh.

Selectors are generated in the browser (`src/picker/selector.ts`); to make an element easy to pick
and robust, give it a `data-wt-event`, a readable `id` or a `data-testid` attribute.

## 12. Integrations

Settings › Integrations lists every supported plugin that is active. An integration runs only when
its plugin is active **and** it is switched on (the setup wizard suggests it). Each event can be
switched off or renamed there; form plugins also have per-form settings.

### WordPress (core)

| Event | Default name | Params | Delivered |
|---|---|---|---|
| Login | `login` | `method: wp` | next page view (server queue) |
| Sign-up | `sign_up` | `method: wp` | next page view — only self sign-ups (registration form, checkout); not accounts created by a logged-in user, WP-CLI or cron |
| Comment | `comment_submit` | `post_id`, `post_type`, `comment_status` (`approved`/`pending`) | next page view; never spam, never content or author data |
| Search | `search` | `results_count` (+ `search_term` only with *Send the search term*) | search results page |
| Page not found | `page_not_found` | `path` | 404 page |

Role exclusions use the user that logs in / signs up, and requests that run as anonymous on REST
(no nonce, e.g. form plugins) still recognize a logged-in visitor by the auth cookie.

### Forms

| Plugin | Server hook (source of the event) | Client signal (pulls the queue at once) |
|---|---|---|
| Contact Form 7 | `wpcf7_mail_sent` | `wpcf7mailsent` |
| WPForms | `wpforms_process_complete` | `wpformsAjaxSubmitSuccess` |
| Gravity Forms | `gform_after_submission` (spam skipped) | `gform_confirmation_loaded` |
| Elementor Pro forms | `elementor_pro/forms/new_record` | `submit_success` |
| Fluent Forms | `fluentform/submission_inserted` (+ the pre-5.0 name) | `fluentform_submission_success` |

- Only **successful** submissions count; the event is `form_submit` with `form_plugin`
  (integration slug), `form_id`, `form_name`.
- The server hook queues the event for the visitor; AJAX forms trigger an immediate pull, page
  reload forms deliver it on the next page view. Each submission is queued once
  (`plugin:form_id:submission`), so the client signal never adds a second event.
- **Per form** (Settings › Integrations › *Forms*): on/off, a custom event name (e.g.
  `quote_request`) and extra params. A param value can be fixed text or contain
  `{{form.field:NAME}}` / `{{form.id}}`. Field names: CF7 tag name, WPForms field id or label,
  Gravity field id / admin label / parameter name, Elementor field id, Fluent input name. Values
  of password, email, phone, hidden and file fields, of fields whose name looks personal
  (`pass|mail|phone|mobile|tel|card|cvv|national|melli|nonce|token|secret`) and values that
  look like an email or phone number are never sent.
- Want an extra, page-specific event? Events › Presets › *Form submit — <plugin>* creates a rule
  that listens to the plugin's client success signal (`dom_event` triggers also listen to jQuery
  events of that name).

### WooCommerce

Requires WooCommerce ≥ 8.0; HPOS and Cart/Checkout Blocks compatible (orders are only touched
through WooCommerce CRUD).

| Event | Default name | Params | Source |
|---|---|---|---|
| Product viewed | `view_item` | `product_id`, `product_sku`, `product_cat` (primary: Yoast/Rank Math primary term, else the first), `price`, `currency` | product page (data printed with the page — URL data only) |
| Added to cart | `add_to_cart` | `product_id`, `variation_id`?, `quantity`, `price` (unit), `currency` | `woocommerce_add_to_cart` (classic AJAX, product form, Store API/blocks) → server queue |
| Removed from cart | `remove_from_cart` | `product_id`, `variation_id`?, `quantity` | `woocommerce_cart_item_removed` → server queue |
| Cart viewed | `view_cart` | `item_count`, `cart_total`, `currency` | cart page (configured page, Cart block or `[woocommerce_cart]`); totals from `GET woo/cart-summary` |
| Checkout started | `begin_checkout` | same as `view_cart` | checkout page (configured, Checkout block or shortcode; not order-received/order-pay) |
| Purchase | `purchase` (revenue) | `order_id`, `item_count`, `payment_method`, `shipping_method`?, `coupon_used` | see below |

- Money params (`price`, `cart_total`) are converted like revenue (Toman → Rial) and carry the
  resulting `currency`.
- Cart events are pulled right after WooCommerce's client events: jQuery `added_to_cart`,
  `removed_from_cart`, `updated_cart_totals`, DOM `wc-blocks_added_to_cart`,
  `wc-blocks_removed_from_cart` and (not on checkout pages) `wc-blocks_store_sync_required`,
  the only signal of Cart block / mini-cart removals in WooCommerce 11. Otherwise they arrive with
  the next page view.
- The page script (`build/integrations.js`) is loaded on product, cart and checkout pages only.

**Purchase — exactly once per order.**

1. Order created (`woocommerce_checkout_order_processed`, `woocommerce_store_api_checkout_order_processed`):
   the visitor id is stored on the order (`_wisetrack_visitor_sid`). Excluded roles get none.
2. Ready (`_wisetrack_purchase_ready`): *When the order is paid* (default) — the first of
   `woocommerce_payment_complete` / status `processing` / `completed` (on-hold, failed,
   cancelled → nothing); *When the thank-you page is shown* — at creation, sent from the
   thank-you page only (failed orders never).
3. Queued once (`_wisetrack_purchase_queued`) for the stored visitor — also when a payment
   gateway webhook made it ready; the visitor gets it on the thank-you page or the next page
   view (the browser keeps asking while a payment is pending). The thank-you page validates the
   order key, adopts the visitor if the order has none and re-offers a queued but unacked
   purchase; an order-received link opened in another browser sends nothing.
4. Acked by the browser → `_wisetrack_purchase_sent` + private order note (filter
   `wisetrack_order_note`). The item uid is derived from the order id (`wco<blog>o<order>`), and
   the browser remembers uids it already handed to the SDK, so re-deliveries are acked but never
   sent twice.

Revenue: *Order total* or *Subtotal* (items after discounts, without tax, shipping and fees),
filter `wisetrack_revenue_amount`, in the **order** currency, converted in the browser: IRT ×10,
IRHT ×10 000, IRHR ×1 000 → IRR (rounded to integers); codes used by Iranian currency plugins
(`TOMAN`, `TMN`, `IRTOMAN`, `RIAL`, `HTOMAN`…) and Persian symbols (`تومان`, `ریال`, `هزار تومان`…) are
recognized; other unsupported codes need a row in *Currency conversion*. Still unsupported → the
purchase is sent as a default event without revenue, Site Health shows a warning and WooCommerce
→ Status → Logs (source `wisetrack`) records it.

Queue storage: each visitor's queue is a row of its own in WooCommerce's session table
(`session_key` `wt…`, expires with the queue and is removed by WooCommerce's session cleanup) —
not inside the customer's session data, which WooCommerce rewrites at the end of every request.

### Popups

Elementor Pro popups (`elementor/popup/show`) and Popup Maker (`pumAfterOpen`): each opening is
tracked as a dialog screen `popup:<id>` with the popup title as display name. Optional event
`popup_open` (`popup_id`, `popup_name`), off by default. A small extra script
(`build/integrations.js`, < 1 KB gz) is loaded only on pages where the popup plugin loads its own
script (or on WooCommerce product/cart/checkout pages). Native `<dialog>` elements are tracked by the SDK itself.

### Writing an integration

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

`frontend_payload()` ends up in `WiseTrackWP.config.integrations[slug]` (its `pending` list in
`config.pending`); it may depend on the URL, never on the visitor. Override
`needs_frontend_script()` when the page needs `build/integrations.js`. Form plugins can extend `WiseTrack\WP\Integrations\Forms\FormIntegration` and
call `$this->submitted( $form_id, $form_name, $fields, $submission_id )`.

## 13. Page builders

### Block editor (Gutenberg)

Every block has a **WiseTrack** panel in its settings sidebar: event name (the normalized name is
shown live), *Send when* (clicked, visible, form inside submitted — form-like blocks only, page
loads), parameters with token chips, *How often*, and *Screen on click* for tab-like blocks
(tabs, accordions, details, steps). A chart icon in the block toolbar marks blocks with an event.
*Pick on site* opens the visual picker on the published page.

The settings are stored in a block attribute `wisetrack` that lives only in the block comment
(`<!-- wp:button {"wisetrack":{"event":"buy"}} -->`); the saved HTML is untouched, so existing
content never becomes invalid and the attribute disappears cleanly if WiseTrack is removed. When
the page is rendered, the first HTML tag of the block gets the matching `data-wt-*` attributes
(static and dynamic blocks; attributes already in the markup win).

### Elementor (free and Pro)

Advanced tab → **WiseTrack** on every widget, section, column and container: event name,
*Send when*, parameters (repeater, tokens allowed), *How often*. The attributes are printed on
the element wrapper; for the Button widget on the link itself. Hooks used (Elementor 4.3):
`elementor/element/common/_section_style/after_section_end` (widgets),
`elementor/element/section/section_advanced/after_section_end`,
`elementor/element/column/section_advanced/after_section_end`,
`elementor/element/container/section_layout/after_section_end`,
`elementor/frontend/before_render`. Switching the integration on/off clears Elementor's cache.
In Elementor free you can also use the class convention; in Elementor Pro, *Attributes*.

### Other builders

No WiseTrack controls; their editors never load the SDK (`wisetrack_builder_preview_params`).
Use custom attributes where the builder has them, otherwise the class convention — or the
visual picker on the published page.

**Bricks** (attributes)
1. Select the element → *Style* tab → *Attributes* → *Add attribute*.
2. Name `data-wt-event`, value `signup_click`.
3. Optional: more attributes such as `data-wt-param-plan` = `pro`, `data-wt-trigger` = `visible`.

**Breakdance / Oxygen 6** (attributes)
1. Select the element → *Settings* → *Advanced* → *Attributes* → *Add*.
2. Name `data-wt-event`, value `signup_click` (+ `data-wt-param-…` as needed).

**Oxygen 4** (attributes)
1. Select the element → *Advanced* → *Attributes* → *Add attribute*.
2. Name `data-wt-event`, value `signup_click`.

**Elementor Pro** (attributes, besides the WiseTrack section)
1. Widget → *Advanced* → *Attributes*.
2. One per line: `data-wt-event|signup_click`, `data-wt-param-plan|pro`.

**WPBakery** (classes)
1. Edit the element → *Design Options* or *General* → *Extra class name*.
2. Enter `wt-ev-signup_click` (add `wt-p-plan-pro` for a fixed param).

**Divi** (classes)
1. Module settings → *Advanced* → *CSS ID & Classes* → *CSS Class*.
2. Enter `wt-ev-signup_click` (visible: `wt-ev-pricing_seen--visible`).

**Beaver Builder** (classes)
1. Module settings → *Advanced* → *HTML Element* → *Class*.
2. Enter `wt-ev-signup_click`.

**Thrive Architect / Brizy** (classes)
1. Select the element → *HTML attributes* / *Advanced* → *CSS class*.
2. Enter `wt-ev-signup_click`.

**Elementor free** (classes, if you prefer them to the WiseTrack section)
1. Widget → *Advanced* → *CSS Classes*.
2. Enter `wt-ev-signup_click wt-p-plan-pro`.

Editor/preview URLs that never load the SDK: `elementor-preview`, `fl_builder`, `et_fb`,
`ct_builder`, `oxygen=builder`, `oxygen_iframe`, `bricks=run`, `vc_editable`, `vcv-editable`,
`tve`, `brizy-edit`, `brizy-edit-iframe`, `breakdance=builder`, `breakdance_iframe`, `fb-edit`.

## 14. Updates (private build)

The private build (`wisetrack-<version>-private.zip` from GitHub Releases) updates itself through
[Plugin Update Checker](https://github.com/YahnisElsts/plugin-update-checker) (MIT, bundled in
`lib/`, version-namespaced so other plugins bundling it do not conflict). The wordpress.org build
contains no updater. Constants for `wp-config.php`:

```php
define( 'WISETRACK_DISABLE_UPDATER', true );                 // no update checks
define( 'WISETRACK_UPDATE_REPOSITORY', 'https://github.com/acme/wisetrack-wordpress/' ); // a fork
define( 'WISETRACK_UPDATER_TOKEN', 'github_pat_…' );         // private repository (read-only token)
```

## 15. Upgrades and settings migrations

Settings carry a schema version (`Schema::VERSION`, currently 2). The installed version of each
site is stored in the option `wisetrack_db_version`; on the first request after an update the
stored settings are migrated step by step (also for updates that never run the activation hook)
and `wisetrack_upgraded` fires. Imported export files run the same steps, so files from older
versions can be imported; files from a newer version are refused.

| Version | Change |
|---|---|
| 1 → 2 | Site search and 404 moved from `smart_events.search` / `page_not_found` (0.1 builds) to the `wordpress` integration (only those events on). |

```php
// Your own migration on top of the built-in ones.
add_filter( 'wisetrack_migrate_settings', function ( $settings, $from ) {
	if ( $from < 2 && isset( $settings['naming'] ) && 'url' === $settings['naming'] ) {
		$settings['naming'] = 'path';
	}
	return $settings;
}, 10, 2 );
```

## 16. Custom screen names (Settings › Advanced)

Screen rules rename page views by regular expression on the path (first match wins, tested on
`location.pathname` by the SDK). `$1`…`$9` insert capture groups into the name and display
name:

| Pattern | Name | Display name | `/courses/php-basics/lesson-3` → |
|---|---|---|---|
| `^/courses/([^/]+)/lesson-(\d+)` | `course_lesson` | `$1 — lesson $2` | name `course_lesson`, display name `php-basics — lesson 3` |
| `^/blog/` | `blog_post` | | name `blog_post`, display name from the page |

Names are sanitized like every screen name (≤ 150 chars); patterns are validated (PCRE) when saved.
