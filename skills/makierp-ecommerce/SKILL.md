---
name: makierp-ecommerce
description: Runs the MakiERP online shop and the product master data behind it through MCP — shop settings and go-live readiness, the order inbox, the home page and menu, content pages, discount codes, memberships, web addresses, and the products, prices, categories, groups and collections the shop sells from. Use when the user asks about their online shop, web orders, storefront content, coupons, shop members, a shop web address, or about creating and editing products, prices, categories, groups or collections.
---

# MakiERP E-Commerce and Catalog

## Quick Start

1. Use `makierp-mcp-session` first if tenant orientation or access is not established. Everything here writes to a live shop.
2. `channel_show` — inspect the shop and its configured behaviour without exposing stored secrets.
3. `channel_readiness` — the go-live checklist. Start here whenever the user says orders are not arriving or the shop is not selling.

## Reading vs. writing

**There are no list tools, and that is deliberate.** Every shop record is a schema, so read them with the analytics tools instead — they filter, group and total, which hand-written list tools could not:

- `channel_orders`, `channel_order_lines` — web orders and what was bought
- `channel_items` — products as the shop carries them
- `channel_customers`, `customer_users` — shop members and their logins
- `channel_coupons`, `channel_payments`, `channel_pages`, `channel_menu_items`

Use `schema_describe` for what a field means, then `analytics_query_preview`. The tools in this skill are for **acting**, plus the few reads the query language cannot serve: shop settings, the readiness checklist, the home page draft, and one order with its open decisions.

## Orders

- Always `channel_orders_show` before deciding. It returns an `actions` object saying which decisions are open — approve, reject, retry, settle a cancellation. **Act only on one that is true**; never infer a decision from the status.
- `channel_orders_approve` books a real sales order in the ERP at the prices captured at checkout. It never re-prices. Confirm with the user first.
- `channel_orders_reject` releases the stock and mails the buyer. The reason is shown to the buyer word for word — take it from the user, in the buyer's language. An order that already became a sales order cannot be rejected, only cancelled.
- `channel_orders_retry` is for a parked order — one whose transfer to the ERP failed. It keeps holding its stock because the buyer was already told the order was placed. Fix the cause first; retrying never creates a second document.
- `channel_orders_cancellation_decide` settles a cancellation the buyer asked for. If the ERP refuses the cancellation, the order stays as it was and the reason comes back on the order.
- Every decision returns `buyer_notified`. When it is false the decision still stands, but the buyer does not know — tell the user to phone them.

## Products

- `channel_items_sync` makes every ERP product manageable in the shop. It puts nothing on sale.
- `channel_items_publish` puts products on the shelf or takes them off. Products the ERP does not mark sellable online come back in `skipped`.
- `channel_items_update` sets shop wording, address, search-engine text, listing order, and the unit buyers order in. It never touches the ERP product.
- Pictures live in the catalog tools, not here: `product_images_list` to see what a product already has (main image, gallery, shop-only pictures, and the photo of each selling unit), `product_images_upload` to add them, `product_images_delete` to remove them. For a local file, get a link from `product_images_upload_ticket` first; a picture that already has a web address needs no ticket.
- A unit photo is per selling unit — `surface: unit` plus that unit's `unit_id` — because a case looks nothing like a single bottle. The primary and unit surfaces hold one picture each and uploading replaces what is there; gallery and ecommerce accumulate.
- `channel_item_gallery_arrange` sets the order the shop shows them in. The first picture is what listings and the cart use.

## Home page and menu

- `channel_layout_sections` first — it is the only source of valid block names, their settings, which are required, the **page rules**, and the ready-made layouts. A page saved against anything else is refused.
- **The showcase rule is not advisory.** A non-empty page must open with the showcase block, exactly one of them, first, and visible. Its slides split by placement: at least one on the wide rotating panel and exactly the number of tiles `constraints.hero_tiles` reports. Read `constraints` and build to it; do not discover it from a rejection.
- An empty list is legal and means "clear the page" — the shop then draws the theme home page.
- Starting from a preset is the surest route: every preset already satisfies the rules.
- `channel_layout_show` → change → `channel_layout_save`. **The list you send is the whole page**: order is page order, and anything you leave out is removed.
- `channel_layout_publish` puts it live; `channel_layout_discard` throws the working copy away for good.
- The menu works the same way: `channel_menu_show` → `channel_menu_save` with the **whole** tree. Nesting comes from `parent_key`.
- Preview before publishing with `channel_preview_link`.

## Content, coupons, members, addresses

- `channel_pages_seed_legal` creates the legal pages the checkout depends on; it leaves existing ones alone. The texts are drafts and should be reviewed.
- Legal pages cannot be deleted — unpublish them with `channel_pages_update` instead.
- A coupon carries **either** its own discount **or** a campaign, never both — the discount would come off twice. Prefer switching a coupon off over deleting it; deleting loses the record.
- `channel_customers_update` decides whether a member may buy without paying and whether their credit limit is enforced. Take that from the user, never assume.
- A new web address does not work until its DNS records are published and `channel_domains_verify` confirms them. Give the user the records verbatim, and expect the first check after publishing to fail — DNS takes time.

## Boundaries

- **Ask before anything buyers see**: taking the shop live, publishing the home page, putting products on sale, approving or rejecting an order, blocking a member, deleting an address.
- Permissions are enforced per tool. A refusal means the user lacks that permission in MakiERP — say so plainly and stop; do not try another tool to get around it.
- Shop identity, mail, payment and captcha settings are managed in MakiERP rather than through MCP so credentials never enter a model conversation.

## Catalog (the products behind the shop)

Shop tools change how a product **reads and sells online**; catalog tools change the **product itself** in the ERP. Reach for these when the shop side cannot fix the problem.

- `items_update` sets the trading switches — `active`, `usable_for_sales`, `usable_for_purchase`, `usable_at_ecommerce`, `delist`. This is what fixes a product `channel_items_publish` returned in `skipped`.
- `item_prices_create` gives a product a price. This is what fixes a product `channel_readiness` reports as priceless. Read `item_price_overview` first so you add a price rather than a duplicate that quietly outranks the old one. Get `vat_inc` right — wrong, and the product is mispriced by the VAT rate.
- `item_prices_update` / `_delete` — prefer switching a price off over deleting it; the record and what it explains stay.
- **Creating a product is two steps**: `items_preview` validates and reports what is missing without writing anything, then `items_create`. Never call create before preview comes back clean — a product is master data that stock, prices and documents point at. New products are born sellable and purchasable but **closed to the shop and the cash points**; opening them is a deliberate act.
- `items_delete` refuses whenever the product has stock movements, storage records or prices. In almost every real case the right move is `items_update` with `active: false` or `delist: true`.
- Categories, groups and collections have create/update/delete tools. Categories and collections are trees: nothing can be moved under itself or its own descendants, and neither deletes while children or members remain.
- **`product_collections_update` replaces the whole membership** when you send `item_ids`. Read the current members first and send them back with your changes, or you empty the collection — and any shop product row pointing at it goes blank.
- Resolve every id (category, group, brand, unit family, VAT rate) with the analytics tools before sending. Never invent one.
