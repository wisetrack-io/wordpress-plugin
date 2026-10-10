# WiseTrack for WordPress

WordPress plugin for [WiseTrack](https://wisetrack.io/) analytics & attribution: automatic page
views, no-code events (visual picker, event rules, block editor and Elementor panels), WooCommerce
revenue and form tracking — cache-safe, consent-aware and fully translated to Persian.

This repository hosts the **releases** of the plugin. Features, requirements and the changelog are
in [`readme.txt`](readme.txt) and [`CHANGELOG.md`](CHANGELOG.md); attributes, shortcodes, the
JavaScript and PHP APIs and all hooks are in the [developer guide](docs/DEVELOPER.md).

Requirements: WordPress 6.2+, PHP 7.4+ (WooCommerce integration: WooCommerce 8.0+).

## Install

1. Download `wisetrack-<version>-private.zip` from the
   [latest release](https://github.com/wisetrack-io/wordpress-plugin/releases/latest)
   (the `.sha256` file next to it is its checksum).
2. In WordPress go to *Plugins › Add New › Upload Plugin*, choose the zip and activate it.
3. The setup wizard opens: enter the App Token and Client Secret from the WiseTrack panel.

## Updates

The plugin checks this repository for new releases and updates like any other plugin
(*Dashboard › Updates*). To turn that off, add to `wp-config.php`:

```php
define( 'WISETRACK_DISABLE_UPDATER', true );
```

## نصب (فارسی)

فایل `wisetrack-<version>-private.zip` را از
[آخرین نسخه](https://github.com/wisetrack-io/wordpress-plugin/releases/latest) دانلود کنید، در
پیشخوان وردپرس از *افزونه‌ها › افزودن › بارگذاری افزونه* نصب و فعال کنید و در راه‌اندازی، App Token و
Client Secret پنل WiseTrack را وارد کنید. نسخه‌های بعدی از همین مخزن، مثل هر افزونهٔ دیگری از بخش
*به‌روزرسانی‌ها* نصب می‌شوند.

## License

GPL-2.0-or-later (see [`LICENSE`](LICENSE)). The source code of any release is available on request
from [WiseTrack](https://wisetrack.io/). Questions and bug reports: WiseTrack support.
