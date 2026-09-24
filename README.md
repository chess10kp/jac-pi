# jac-pi

Dedicated pi workspace for Jac editing. Running `pi` from this directory gives
you the Jac AST-editing toolchain on top of your normal global config.

## What's in the config

| File | Purpose |
|---|---|
| `.pi/settings.json` | Loads [`pi-jac-ast-edit`](../pi-jac-ast-edit) — the `jac_ast_edit` tool (tree-sitter-backed AST editing for `.jac`, harvested from Empryo's ast_edit) |
| `.pi/mcp.json` | Enables the `jac` MCP server (`jac mcp`) — `jac_check_syntax`, `jac_validate_jac`, `jac_lint_jac`, `jac_format_jac`, `jac_run_jac`, docs tools |

## Usage

```bash
cd ~/repos/jac-pi
pi
```

Typical loop: `jac_ast_edit` for symbol surgery → `jac_check_syntax` to verify
→ `jac_format_jac` to canonicalize.

Local-path packages are referenced in place — edits to
`~/repos/pi-jac-ast-edit` take effect after a `/reload` (no reinstall).

**Clone layout:** the package reference `../../pi-jac-ast-edit` is relative to
`.pi/settings.json`, i.e. it expects the sibling checkout
`~/repos/pi-jac-ast-edit`. Clone both repos side by side:

```bash
git clone git@github.com:chess10kp/jac-pi.git ~/repos/jac-pi
git clone git@github.com:chess10kp/pi-jac-ast-edit.git ~/repos/pi-jac-ast-edit
```

Project settings merge with your global `~/.pi/agent` config; start pi with
`--no-plugins` variants or trim global packages if you want this dir fully
minimal.
