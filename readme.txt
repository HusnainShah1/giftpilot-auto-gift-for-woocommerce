=== GiftPilot – Auto Gift for WooCommerce ===
Contributors: HusnainShah1
Tags: woocommerce, free gift, auto gift, gift, ecommerce
Requires at least: 5.0
Tested up to: 6.5
Requires PHP: 7.0
WC requires at least: 4.0
WC tested up to: 8.0
Stable tag: 1.0.7
License: GPLv2 or later
License URI: https://www.gnu.org/licenses/gpl-2.0.html

Automatically adds a selected free product to the cart for every qualifying order.

== Description ==

GiftPilot lets you offer a free gift product to your customers. When a customer adds a product to their cart, the gift product you selected is automatically added with a price of zero. If the customer removes the gift manually, it is re-added on the next cart update.

You can optionally set a minimum cart subtotal. If set, the gift is only added when the cart subtotal reaches that amount; otherwise the gift is removed.

== Installation ==

1. Upload the `giftpilot-auto-gift-for-woocommerce` folder to the `/wp-content/plugins/` directory, or install the plugin through the WordPress Plugins screen directly.
2. Activate the plugin through the 'Plugins' screen in WordPress.
3. Go to WooCommerce > Settings > Free Gift to select your free gift product.
4. (Optional) Enter a minimum order amount. Leave it empty to always add the gift.

== Frequently Asked Questions ==

= Can I use a variable product as the free gift? =
No. The free gift must be a simple product with its price set to 0.

= How does the minimum order amount work? =
If you set a minimum amount in the plugin settings, the gift is only added when the cart subtotal is greater than or equal to that amount. If you leave the field empty or set it to 0, the gift is added to every order.

= Can the customer remove the free gift? =
The customer can remove it, but the plugin will re-add it automatically while the cart meets the minimum amount condition.

== Changelog ==

= 1.0.7 =
* Renamed plugin to GiftPilot to comply with WordPress.org naming guidelines.
* Clarified description and FAQ regarding minimum order amount.
* Added Requires Plugins header.
* Removed external WooCommerce documentation link.

= 1.0.6 =
* Initial public release.