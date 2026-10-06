<?php
/**
 * Plugin Name: GiftPilot – Auto Gift for WooCommerce
 * Description: Automatically adds a selected free product to the cart for every qualifying order.
 * Version: 1.0.7
 * Author: Syed Husnain
 * Text Domain: giftpilot-auto-gift-for-woocommerce
 * Requires at least: 5.0
 * Requires PHP: 7.0
 * Requires Plugins: woocommerce
 * WC requires at least: 4.0
 * WC tested up to: 8.0
 * License: GPLv2 or later
 * License URI: https://www.gnu.org/licenses/gpl-2.0.html
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

/* =========================================================
 * 0. TOP-LEVEL: HPOS DECLARATION + ADMIN LINKS
 * ========================================================= */

add_action( 'before_woocommerce_init', function() {
    if ( class_exists( '\Automattic\WooCommerce\Utilities\FeaturesUtil' ) ) {
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'cart_checkout_blocks', __FILE__, true );
    }
} );

// "Settings" link next to Deactivate.
add_filter( 'plugin_action_links_' . plugin_basename( __FILE__ ), 'giftpilot_action_links' );
function giftpilot_action_links( $links ) {
    $settings_url  = admin_url( 'admin.php?page=wc-settings&tab=free_gift' );
    $settings_link = '<a href="' . esc_url( $settings_url ) . '">' . esc_html__( 'Settings', 'giftpilot-auto-gift-for-woocommerce' ) . '</a>';
    array_unshift( $links, $settings_link );
    return $links;
}

// Support link under the Description column.
add_filter( 'plugin_row_meta', 'giftpilot_row_meta', 10, 2 );
function giftpilot_row_meta( $links, $file ) {
    if ( plugin_basename( __FILE__ ) !== $file ) {
        return $links;
    }

    $support_url = 'https://wordpress.org/support/plugin/giftpilot-auto-gift-for-woocommerce/';

    $custom_links = array(
        '<a href="' . esc_url( $support_url ) . '" target="_blank" rel="noopener noreferrer">' . esc_html__( 'Community Support', 'giftpilot-auto-gift-for-woocommerce' ) . '</a>',
    );

    return array_merge( $links, $custom_links );
}

/* =========================================================
 * 1. INITIALIZE PLUGIN
 * ========================================================= */

add_action( 'plugins_loaded', 'giftpilot_init' );
function giftpilot_init() {
    if ( ! class_exists( 'WooCommerce' ) ) {
        return;
    }

    add_filter( 'woocommerce_settings_tabs_array', 'giftpilot_add_settings_tab', 99 );
    add_action( 'woocommerce_settings_free_gift', 'giftpilot_settings_page' );
    add_action( 'woocommerce_update_options_free_gift', 'giftpilot_update_settings' );

    add_action( 'woocommerce_cart_loaded_from_session', 'giftpilot_ensure_gift_in_cart', 20 );
    add_action( 'woocommerce_add_to_cart', 'giftpilot_ensure_gift_in_cart', 20 );
    add_action( 'woocommerce_cart_item_removed', 'giftpilot_ensure_gift_in_cart', 20 );
    add_action( 'woocommerce_before_calculate_totals', 'giftpilot_validate_gift', 15 ); // NEW: Validate before zeroing price
    add_action( 'woocommerce_before_calculate_totals', 'giftpilot_zero_price', 20 );
}

/* =========================================================
 * 2. SETTINGS TAB
 * ========================================================= */

function giftpilot_add_settings_tab( $tabs ) {
    $tabs['free_gift'] = __( 'Free Gift', 'giftpilot-auto-gift-for-woocommerce' );
    return $tabs;
}

function giftpilot_settings_page() {
    woocommerce_admin_fields( giftpilot_get_settings() );
}

function giftpilot_update_settings() {
    woocommerce_update_options( giftpilot_get_settings() );
}

function giftpilot_get_settings() {
    return array(
        array(
            'name' => __( 'Free Gift Settings', 'giftpilot-auto-gift-for-woocommerce' ),
            'type' => 'title',
            'desc' => __( 'Configure the free gift product that is automatically added to qualifying orders.', 'giftpilot-auto-gift-for-woocommerce' ),
            'id'   => 'giftpilot_section_title',
        ),
        array(
            'name'     => __( 'Free Gift Product', 'giftpilot-auto-gift-for-woocommerce' ),
            'type'     => 'select',
            'desc'     => __( 'Select the product to give away for free.', 'giftpilot-auto-gift-for-woocommerce' ),
            'id'       => 'giftpilot_product_id',
            'options'  => giftpilot_get_product_options(),
            'css'      => 'min-width:300px;',
            'desc_tip' => true,
        ),
        array(
            'name'              => __( 'Minimum Order Amount', 'giftpilot-auto-gift-for-woocommerce' ),
            'type'              => 'number',
            'desc'              => __( 'Only add the free gift when the cart subtotal is at least this amount. Leave empty or 0 to always add.', 'giftpilot-auto-gift-for-woocommerce' ),
            'id'                => 'giftpilot_minimum_amount',
            'default'           => '',
            'custom_attributes' => array( 'min' => '0', 'step' => '0.01' ),
            'desc_tip'          => true,
        ),
        array( 'type' => 'sectionend', 'id' => 'giftpilot_section_end' ),
    );
}

function giftpilot_get_product_options() {
    $options = array( '' => __( 'Select a product…', 'giftpilot-auto-gift-for-woocommerce' ) );

    $products = wc_get_products(
        array(
            'limit'   => -1,
            'status'  => 'publish',
            'orderby' => 'title',
            'order'   => 'ASC',
        )
    );

    foreach ( $products as $product ) {
        $options[ $product->get_id() ] = $product->get_name();
    }

    return $options;
}

/* =========================================================
 * 3. CORE LOGIC
 * ========================================================= */

/**
 * Calculate cart subtotal excluding the gift product.
 */
function giftpilot_get_subtotal_excluding_gift() {
    if ( ! function_exists( 'WC' ) || ! WC()->cart ) {
        return 0;
    }

    $gift_id  = absint( get_option( 'giftpilot_product_id', 0 ) );
    $subtotal = 0;

    foreach ( WC()->cart->get_cart() as $cart_item ) {
        if ( absint( $cart_item['product_id'] ) !== $gift_id ) {
            $subtotal += $cart_item['line_subtotal'];
        }
    }
    return $subtotal;
}

function giftpilot_ensure_gift_in_cart() {
    if ( ! function_exists( 'WC' ) || ! WC()->cart ) {
        return;
    }

    $cart = WC()->cart;
    if ( ! $cart || $cart->is_empty() ) {
        return;
    }

    $gift_id = absint( get_option( 'giftpilot_product_id', 0 ) );
    if ( empty( $gift_id ) ) {
        return;
    }

    if ( $cart->find_product_in_cart( $cart->generate_cart_id( $gift_id ) ) ) {
        return;
    }

    $min = get_option( 'giftpilot_minimum_amount', '' );
    $subtotal = giftpilot_get_subtotal_excluding_gift();

    if ( '' !== $min && (float) $min > 0 && $subtotal < (float) $min ) {
        return;
    }

    $product = wc_get_product( $gift_id );
    if ( ! $product || ! $product->is_purchasable() || ! $product->is_in_stock() ) {
        return;
    }

    $cart->add_to_cart( $gift_id, 1 );
}

/**
 * NEW: Remove gift if cart subtotal drops below minimum amount.
 */
function giftpilot_validate_gift( $cart ) {
    if ( ! is_a( $cart, 'WC_Cart' ) ) {
        return;
    }

    if ( is_admin() && ! defined( 'DOING_AJAX' ) ) {
        return;
    }

    $gift_id = absint( get_option( 'giftpilot_product_id', 0 ) );
    if ( empty( $gift_id ) ) {
        return;
    }

    $cart_id = $cart->generate_cart_id( $gift_id );
    if ( ! $cart->find_product_in_cart( $cart_id ) ) {
        return;
    }

    $subtotal = giftpilot_get_subtotal_excluding_gift();
    $min      = get_option( 'giftpilot_minimum_amount', '' );

    // Remove gift if subtotal drops below minimum, or if cart is empty of other products.
    if ( ( '' !== $min && (float) $min > 0 && $subtotal < (float) $min ) || $subtotal == 0 ) {
        $cart->remove_cart_item( $cart_id );
    }
}

/* =========================================================
 * 4. FORCE GIFT PRICE TO ZERO
 * ========================================================= */

function giftpilot_zero_price( $cart ) {
    if ( ! is_a( $cart, 'WC_Cart' ) ) {
        return;
    }

    if ( is_admin() && ! defined( 'DOING_AJAX' ) ) {
        return;
    }

    $gift_id = absint( get_option( 'giftpilot_product_id', 0 ) );
    if ( empty( $gift_id ) ) {
        return;
    }

    foreach ( $cart->get_cart() as $cart_item ) {
        if ( absint( $cart_item['product_id'] ) === $gift_id ) {
            if ( isset( $cart_item['data'] ) && is_object( $cart_item['data'] ) ) {
                $cart_item['data']->set_price( 0 );
            }
        }
    }
}
