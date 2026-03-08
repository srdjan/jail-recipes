# Jail Recipes

This repository is organized around self-contained Jailrun packages.

## Layout

- `playbooks/<name>/<version>/` is the canonical home for each install playbook package. Keep that package's `README.md`, `playbook.yml`, examples, templates, and support files together.
- `stacks/<name>/<version>/` holds composite recipes that compose multiple playbook packages.
- `shared/` is reserved for assets reused by more than one package.
- `mnt/user-data/outputs/jailrun-hub/playbooks/<name>/<version>/` mirrors the canonical playbook packages for compatibility.
- The root `playbook.yml` and root `*.ucl` files are compatibility symlinks for existing local commands.

## Packages

- `playbooks/deno/latest/` installs Deno and includes the `deno-app` example recipe.
- `playbooks/zig/latest/` installs Zig and includes the `zig-app` example recipe.
- `stacks/deno-zig/latest/` provides a Deno API plus Zig compute stack recipe.

## Adding a playbook

1. Create `playbooks/<name>/<version>/`.
2. Keep the package's docs, playbook, examples, templates, and support files in that directory.
3. Move assets into `shared/` only after more than one package needs them.
4. Add the new package to this index.

## Assumptions

- Existing root entrypoints stay available as symlinks so `jrun up deno-app.ucl`, `jrun up zig-app.ucl`, `jrun up deno-zig-stack.ucl`, and direct references to the root `playbook.yml` continue to work.
- The old `mnt/.../playbooks/*/latest` paths are compatibility mirrors and no longer own the source files.
