---
name: ah-meal-bridge
description: "Connect meal_manager (recipes, fridge, what to cook) with Albert Heijn (receipts, shopping list, bonus deals)."
version: 0.1.0
author: motionbug
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [meal planning, groceries, albert heijn, ah, shopping list, fridge, receipts, bonus, kassabon]
    category: productivity
---

# AH ↔ Meal Manager Bridge

Two systems, one job each:

- **meal_manager** (plugin tools: `list_fridge`, `update_fridge_inventory`, `get_meal_suggestions`, `get_quick_shopping_list`, `get_missing_for_dish`, `list_dishes`, …) owns recipes, fridge and cooking history. Load the `meal_manager:assistant` skill for its own rules and follow them.
- **Albert Heijn** (MCP tools `mcp__ah__*`) owns products, prices, bonus deals, receipts, orders and the AH shopping list.

meal_manager decides **what** is needed; AH answers **which product** and **what was bought**. Never let one system's data silently overwrite the other: always show the user the translation and let them correct it.

## Product mapping file

Keep the user's ingredient ↔ product choices in `~/.hermes/data/ah-meal-bridge/products.md` (one table row per ingredient). Read it before any search; only search AH for ingredients that have no row. When the user picks or confirms a product, add or update the row. When a receipt line is matched to an ingredient, add a row for that too. Do not store anything else there.

Columns: meal_manager ingredient (exactly as meal_manager spells it) | Dutch search term | preferred AH product_id | product title | portions per pack.

## Workflow 1 — Restock the fridge from AH

Triggers: "I just got back from AH", "my delivery arrived", "update my fridge from my receipt".

1. Find the purchase: in-store → `ah_get_receipts` then `ah_get_receipt_details`; delivery → `ah_get_past_orders` then `ah_get_order_details`. Take the most recent unless the user says otherwise.
2. Drop non-food lines (bags, deposit/statiegeld, household, drugstore, discounts).
3. Map each food line to a meal_manager ingredient name: use the mapping file first, then `list_dishes` / `list_fridge` spellings, then a plain lowercase English generic ("AH Scharreleieren 10 stuks" → `eggs`). Use the user's own word if they have one.
4. Convert packs to portions (dishes' worth, per meal_manager's rules): mapping file first, otherwise estimate.
5. Show the user one compact list — `ingredient: portions (from: receipt line)` plus skipped lines — and wait for confirmation or corrections.
6. Call `update_fridge_inventory` with action `add` and the object form, once. Save new mappings to the file.

## Workflow 2 — Put what's missing on the AH list

Triggers: "what should I buy, put it on my AH list", "I want to make X on Thursday, get what's missing".

1. Get the need from meal_manager: a named dish → `get_missing_for_dish`; otherwise → `get_quick_shopping_list` (follow its `still_missing` rules exactly, and retry with `max_missing: 2` if empty).
2. Resolve each ingredient: mapping file → known product_id. Unknown ones → one `ah_search_products_bulk` call with Dutch search terms (max 10 per call).
3. Show the user the proposed products with price and bonus status; flag anything ambiguous. Wait for confirmation.
4. Call `ah_add_to_shopping_list` once with all confirmed product_ids. Use `ah_add_free_text_to_shopping_list` only for items with no good match.
5. Save new mappings. Tell the user the items are on their AH **shopping list**. Moving them into the online order is not available to you; the user does that in the AH app.

## Workflow 3 — Bonus-aware dinner planning

Triggers: "what can I cook this week that's on bonus", "use the bonus deals".

1. `get_meal_suggestions` (cookable now) and `get_quick_shopping_list` with `max_missing: 2` (close to cookable).
2. For the missing ingredients of those dishes, check bonus status: mapped product_ids via `ah_get_products_bulk`, others via `ah_search_products_filtered` with `bonus: "true"`, or filter `ah_get_bonus_offers` by keyword.
3. Present dishes whose missing items are on bonus first, with the deal text. Rankings within each group come from meal_manager — do not re-rank by your own taste.
4. Offer Workflow 2 for the chosen dish(es).

## Rules

- Confirm before every write: `update_fridge_inventory` and `ah_add_to_shopping_list` / `ah_remove_from_shopping_list`.
- Batch: one bulk search, one shopping-list add, one fridge update per workflow run.
- Suggestions and rankings come only from meal_manager tool output; product facts (price, bonus, title) only from AH tool output.
- If an AH tool reports `not_authenticated`, call `ah_login` once with no arguments. If the browser login ends on a blank page, ask the user for the `appie://login-exit?code=...` line from the browser console and pass it as `code`.
- Not available on purpose: emptying the cart or list, editing the online order. If asked, say the user can do it in the AH app.
