# kannabi-plugin-gs1

GS1 identifiers for [Kannabi](https://github.com/iomz/kannabi), as a plugin.

Kannabi core records identifiers and issues none.
This plugin is what gives a Kannabi instance GS1: without it there is no GS1 scheme, route, tool, setting or address, and nothing is missing.

## What it contributes

- Identifier schemes: `gtin`, `sgtin`, `grai`, `giai`.
- GS1 Company Prefix namespaces, managed class keys and key issuance, with an issuance ledger.
- GS1 Digital Link addresses for individual-level identifiers.
- The GTIN consistency policy and its conflict set.
- A section of the Asset representation, `extensions.gs1`.
- HTTP routes under `/api/gs1/`.
- MCP tools: `list_gs1_namespaces`, `list_managed_class_keys`, `list_gs1_issuances`.

## Layout

- `toolkit/` is GS1 as Kannabi needs it: syntax policy, check digits, key composition and Digital Link encoding. It imports nothing of Kannabi.
- `adapter/` makes the toolkit a Kannabi capability: schemes, persistence, authority checks, routes, tools and addresses.
- `web/` holds every GS1 screen.
- `index.ts` is the plugin definition Kannabi composes.

The plugin's rules are in [AGENTS.md](AGENTS.md).

## Requirements

- Kannabi plugin contract 1.
- No dependency beyond those Kannabi already has.

## Using it

The plugin is source that lives at `plugins/gs1/` inside a Kannabi checkout and is built with it; it imports Kannabi's modules by relative path and is not a standalone package.
Its tests run through Kannabi's own `pnpm test` and `pnpm test:integration`.

This repository was split from Kannabi v0.11.0, where the same files are at `plugins/gs1/`.
The history of the code before it became a plugin is in the Kannabi repository.

## License

MIT. See [LICENSE](LICENSE).
