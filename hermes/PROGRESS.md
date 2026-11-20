# Hermes integration: progress log

Working notes for getting ah-mcp running in Hermes Agent and connected to the [meal-manager](https://github.com/sergiparpal/meal-manager) plugin. Newest entries at the top of each section. Update this file whenever something is decided, found, or done.

## Status

| Area | State |
|---|---|
| ah-mcp in Hermes (stdio) | ✅ working, 22 tools exposed |
| AH login | ✅ logged in (via console-code fallback); redirect fix not yet proven by a full browser login |
| ah-meal-bridge skill | v0.2.0 installed |
| Workflow 1: AH purchase → fridge | ✅ tested once on a delivery order |
| Workflow 2: missing items → AH list | ⏳ not tested (needs recipes) |
| Workflow 3: bonus-aware planning | ⏳ not tested (needs recipes) |
| meal-manager recipes | ❌ none yet (blocker for workflows 2 and 3) |

## To do

- [ ] Apply restock corrections in Hermes: add `salami: 3` (AH Pizza salami is 125 g topping salami), set `chicken breast` to 3 (800 g pack); update both rows in the mapping file.
- [ ] Set up recipes in meal-manager (from fridge + frequent AH items), using the fridge's English ingredient names.
- [ ] Test workflow 2 (missing items → AH shopping list).
- [ ] Test workflow 3 (bonus-aware planning).
- [ ] Test workflow 1 on an in-store receipt (unknown whether receipt lines have readable names and product ids).
- [ ] Fix `ah_get_product` / `ah_get_products_bulk` returning 404 for valid product ids (search finds the same ids).
- [ ] Prove the RSC redirect fix with a real browser login (`ah_logout` → `ah_login`).
- [ ] After about a week of use: decide whether the connector plugin (mapping + pack→portion rules as tools) is needed.
- [ ] Decide on upstream PR to mrserzhan/ah-mcp (login fix only, or login fix + `hermes/`).
- [ ] If the skill URL moves off the `hermes` branch, update the install URL in `hermes/README.md`.

## Decisions

- **2026-11-20: Ingredient names stay English** in meal-manager, matching the existing fridge. Dutch AH search terms live in the mapping file.
- **2026-11-20: AH tools exposed to Hermes are read-only plus shopping-list adds/removes.** Cart edits, order edits, `ah_shopping_list_to_order` and the clear tools are excluded; moving the list into the order stays manual in the AH app.
- **2026-11-20: Integration is a skill first, plugin later.** Neither ah-mcp nor meal-manager is changed for the integration; build a connector plugin only if the skill + mapping file isn't enough.
- **2026-11-20: Work happens on the `hermes` branch of the fork** (`origin` = motionbug/ah-mcp, `upstream` = mrserzhan/ah-mcp).

## Findings

- **2026-11-20: Delivery order lines are clean:** product_id, full product name, quantity, price. No pack size.
- **2026-11-20: Pack sizes come from search:** `ah_search_products_bulk` with the exact title returns `unit` (`1 kg`, `800 g`, `2 stuks`) under the same id.
- **2026-11-20: `ah_get_product` / `ah_get_products_bulk` return 404** for valid ids (e.g. 613305, 531825, 400699).
- **2026-11-20: Names mislead:** "AH Pizza salami" is 125 g sliced salami, not a pizza. Judge by size and price, not name alone.
- **2026-11-20: One bonus category errors on AH's side** ("Failed dependency" for Vegetarisch, vegan en plantaardig); the server falls back to spotlight deals.
- **2026-11-20: Login failure root cause:** AH's login site is now Next.js and delivers the final `appie://login-exit?code=…` redirect inside the RSC payload (`text/x-component`), which the proxy didn't rewrite. Fixed in `auth.go`; `ah_login` also accepts `code` as a fallback.
- **2026-11-20: Hermes keeps the old MCP process after a binary update** until `/reload-mcp` or a gateway restart (`hermes gateway restart`). Earlier failures in a chat keep steering the model; start a new chat after fixes.
- **2026-11-20: macOS kills a binary copied over in place** (exit 137); `rm` then `cp`.

## Local setup (this machine)

| What | Where |
|---|---|
| ah-mcp binary | `~/.local/bin/ah-mcp` (backup of v1.0.12: `ah-mcp.v1.0.12.bak`) |
| Hermes config | `~/.hermes/config.yaml` → `mcp_servers.ah` (backups: `config.yaml.bak-*`) |
| Installed skill | `~/.hermes/skills/productivity/ah-meal-bridge/SKILL.md` (copy of `hermes/skills/ah-meal-bridge/SKILL.md`) |
| Product mapping | `~/.hermes/data/ah-meal-bridge/products.md` |
| meal-manager data | `~/.hermes/plugins/meal_manager/data/` |
| AH tokens | `~/Library/Application Support/ah-mcp/tokens.json` |
| Hermes logs | `~/.hermes/logs/agent.log`, `mcp-stderr.log` |
| Restart | `/reload-mcp` (MCP servers only) or `hermes gateway restart` (everything) |

## Log

- **2026-11-20:** Found and fixed the login failure; logged in with the console code. Forked to motionbug/ah-mcp, `hermes` branch. Widened the Hermes AH tool list, wrote the ah-meal-bridge skill, added `hermes/` docs. First restock test from a delivery order: fridge 22 → 39 items, verified; mapping file seeded. Skill updated to 0.2.0 from what the test showed.
