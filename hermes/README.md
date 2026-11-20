# ah-mcp with Hermes Agent

Setup notes for running ah-mcp in [Hermes Agent](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes), plus an optional skill that connects it to the [meal-manager](https://github.com/sergiparpal/meal-manager) plugin.

| File | What it is |
|---|---|
| [`config.example.yaml`](config.example.yaml) | `mcp_servers` block for `~/.hermes/config.yaml` |
| [`skills/ah-meal-bridge/SKILL.md`](skills/ah-meal-bridge/SKILL.md) | Hermes skill: AH receipts → fridge, missing ingredients → AH list, bonus-aware planning |
| [`skills/ah-meal-bridge/products.template.md`](skills/ah-meal-bridge/products.template.md) | Empty ingredient ↔ AH product table the skill maintains |

## 1. Install the server

Build or download the binary (see the [main README](../README.md#quick-start-pre-built-binary)) and put it somewhere stable, e.g. `~/.local/bin/ah-mcp`.

> **macOS:** when replacing an existing binary, delete it first and then copy the new one (`rm ~/.local/bin/ah-mcp && cp ah-mcp ~/.local/bin/`). Copying over the old file in place can leave macOS with a stale code signature, and the binary gets killed on launch (exit code 137, no output).

## 2. Configure Hermes

Merge [`config.example.yaml`](config.example.yaml) into `~/.hermes/config.yaml` and fix the `command` path. Then check it:

```bash
hermes mcp test ah
```

In a running session, run `/reload-mcp` to pick up changes.

Notes:

- Hermes registers the tools as `mcp__ah__<tool>`, e.g. `mcp__ah__ah_login`.
- Hermes passes only the `env` you configure, plus a small safe baseline (`PATH`, `HOME`, …), to stdio servers. Put any `AH_*` variables under `env:` in the server block.
- `timeout: 330` matters: in local mode `ah_login` waits for you to finish logging in in the browser (up to 5 minutes). The Hermes default of 300 s is too close.
- **After updating the binary, restart the server** (`/reload-mcp` or restart Hermes). Hermes keeps the old process, and with it the old code, running until then.

## 3. Log in

Ask Hermes to *"log in to Albert Heijn"*. Your browser opens on a local login page (`http://localhost:9876/login…`). Log in there, including 2FA, and the tool returns *"Login successful! Connected as …"*. Tokens are saved and refreshed automatically, so this is a one-time step.

**Blank page after logging in?** Open the browser's developer console. If it shows

```
Failed to launch 'appie://login-exit?code=…' because the scheme does not have a registered handler.
```

copy that `appie://login-exit?code=…` URL and tell Hermes: *"call ah_login with code `<paste>`"*. The code is exchanged directly and you're logged in. Codes are short-lived, so do it right away.

**Hermes says login is still timing out after you fixed it?** Earlier failures in the same chat steer the model. Start a new chat, or tell it to call `ah_login` once with no arguments. If you're already logged in, it returns *"Already connected as …"* immediately.

## 4. Optional: meal-manager integration

[meal-manager](https://github.com/sergiparpal/meal-manager) is a Hermes plugin that keeps your recipes, fridge, and cooking history and decides what to cook. The **ah-meal-bridge** skill tells the agent how to combine it with AH:

1. **Restock from AH:** reads your latest receipt or delivery order, skips non-food lines, converts packs to fridge portions, and updates the fridge after you confirm.
2. **Missing items → AH shopping list:** takes what meal-manager says you need (`get_missing_for_dish` / `get_quick_shopping_list`), finds the products in one bulk search, and adds them to your AH shopping list after you confirm.
3. **Bonus-aware planning:** among dishes that are cookable or close, puts those whose missing items are on bonus first.

The skill stages items on your AH **shopping list**. Moving them into your online order, or emptying the cart or list, stays a manual step in the AH app; the example config leaves those tools out.

### Install

```bash
hermes plugins install sergiparpal/meal-manager --enable

hermes skills install --category productivity \
  https://raw.githubusercontent.com/motionbug/ah-mcp/hermes/hermes/skills/ah-meal-bridge/SKILL.md

mkdir -p ~/.hermes/data/ah-meal-bridge
cp hermes/skills/ah-meal-bridge/products.template.md ~/.hermes/data/ah-meal-bridge/products.md
```

Or copy the skill folder by hand to `~/.hermes/skills/productivity/ah-meal-bridge/`. Check with `hermes skills list | grep ah-meal-bridge`.

### Try it

meal-manager starts with no recipes, so suggestions are empty at first. A good order:

1. *"Look at what I buy most often at AH, suggest which dishes I usually cook, and set them up as recipes."*
2. *"I just got back from AH, update my fridge from my latest receipt."*
3. *"What can I cook this week that's mostly on bonus? Put what's missing on my AH list."*

The skill saves your product choices in `~/.hermes/data/ah-meal-bridge/products.md` (ingredient, Dutch search term, product ID, portions per pack), so it stops re-searching the same things. You can edit that file directly.
