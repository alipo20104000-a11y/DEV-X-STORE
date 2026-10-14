# DEV-X STORE — Backend Integration Notes

This document describes the currently connected Supabase project and the frontend contract. It does not claim that the current static HTML page is already connected to Supabase.

## Supabase project

- Project ref: `ktviowcmqlkgpmzvtluq`
- API URL: `https://ktviowcmqlkgpmzvtluq.supabase.co`
- Region: `eu-west-1`
- Use the project's **publishable** key (or legacy anon key) in browser code. Never expose a `service_role` key, database password, or other secret in frontend code or GitHub.
- Configure the URL and publishable key in the frontend's environment/configuration. For a static single-file app, use a small dedicated config module and load Supabase JS from a pinned CDN version; if the project is migrated to a build tool, use environment variables instead.

## Database areas

Core tables currently present include:

- Catalog: `categories`, `products`, `product_images`
- Customer: `profiles`, `addresses`, `cart_items`, `wishlist_items`
- Orders: `orders`, `order_items`
- Authorization/audit: `roles`, `user_roles`, `admin_permissions`, `user_admin_permissions`, `user_activity`

Row Level Security is enabled on the application's tables. Public catalog reads should only return active products/categories and images attached to active products. Do not bypass RLS from the browser.

## Roles and permissions

The owner role is reserved for the project owner. Admin access is permission-based; available permission keys are:

- `activity.view`
- `admins.manage`
- `catalog.manage`
- `customers.view`
- `orders.manage`
- `orders.view`
- `store.settings`

Do not let the client decide its own role or permissions. Read role/permission state from the database and let RLS/RPC checks enforce access.

## Supported RPC calls

### Record the current user's login

Call `record_my_login()` after a successful authenticated sign-in. The function derives the user from the authenticated JWT.

### Create an order from the cart

Call `create_order_from_cart` only for an authenticated customer, with:

```js
supabase.rpc("create_order_from_cart", {
  p_shipping_address: {
    recipient_name: "Customer name",
    phone: "Customer phone",
    address_line: "Street and building",
    city: "City",
    // Optional: additional address fields supported by the UI
  },
  p_payment_method: "cash_on_delivery",
  p_customer_note: null
});
```

Allowed payment method values are currently `cash_on_delivery` and `online`. Prices and totals are calculated on the server; the RPC validates cart contents/stock, creates order items, reduces stock, clears the cart, and records activity. The shipping fee is currently configured as zero. The `online` option does **not** mean a payment gateway is integrated; a real provider and verified payment webhook are still required before accepting online payments.

### Owner-only admin management

- `set_store_admin_permissions(p_user_id, p_permissions)`
- `remove_store_admin(p_user_id)`

These calls are intended only for the owner. The database functions perform authorization checks. Do not expose admin assignment to ordinary customers.

## Current repository state and remaining integration work

The repository's default branch currently contains `index.html` and `README.md`; no `package.json` or Supabase client configuration was found at the repository root. The HTML is a large standalone page. The frontend still needs to be connected to Supabase and tested in a browser.

Before launch:

1. Configure email confirmation and allowed redirect URLs in Supabase Auth.
2. Enable leaked-password protection in Supabase Auth settings.
3. Configure Google OAuth only after adding the OAuth credentials in the Supabase dashboard.
4. Add and test sign-up/sign-in/sign-out, session persistence, catalog reads, cart/wishlist, addresses, order history, and owner/admin views.
5. Test checkout with a dedicated test account and test products. Do not create fake customer orders in production.
6. Integrate a real payment provider before treating `online` as a live payment option.
7. Keep migrations under version control and apply changes through reviewed migrations.

## Migration history currently recorded in Supabase

- `20261014215718` — `store_admin_permissions_activity_and_owner`
- `20261014215732` — `tighten_store_helper_function_execute_grants`
- `20261014215754` — `enforce_granular_store_admin_permissions`
- `20261014215801` — `preserve_public_catalog_reads_with_rls`
- `20261014220034` — `secure_checkout_rpc_and_order_write_paths`
- `20261014220044` — `owner_managed_admin_assignment_rpc`
- `20261014220104` — `index_store_foreign_keys_and_rls_lookups`
- Latest additional migration — `split_store_admin_write_policies`

The migrations listed here are those recorded by the live Supabase project; the SQL migration files still need to be committed into this repository for a reproducible local/dev setup.
