# Jail Recipes

This repository is organized as a package-first Jailrun monorepo.

## Layout

- `packages/playbooks/<name>/<version>/` is the canonical home for each install playbook package. Keep that package's `README.md`, `playbook.yml`, examples, templates, and support files together.
- `packages/stacks/<name>/<version>/` holds composite recipes that compose multiple playbook packages.
- `shared/` is reserved for assets reused by more than one package.
- The repo root stays minimal: index docs plus shared assets only.

## Packages

- `packages/playbooks/deno/latest/` installs Deno and includes the `deno-app` example recipe.
- `packages/playbooks/zig/latest/` installs Zig and includes the `zig-app` example recipe.
- `packages/stacks/deno-zig/latest/` provides a Deno API plus Zig compute stack recipe.

## Adding a playbook

1. Create `packages/playbooks/<name>/<version>/`.
2. Keep the package's docs, playbook, examples, templates, and support files in that directory.
3. Move assets into `shared/` only after more than one package needs them.
4. Add the new package to this index.
