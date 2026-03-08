# deno-zig/latest

Composite Jailrun stack package for a Deno API gateway backed by a Zig compute service.

## Package contents

- `recipe.ucl` is the canonical stack recipe.

## Usage

### From this repo (local)

```sh
jrun up stacks/deno-zig/latest/recipe.ucl
```

Compatibility alias:

```sh
jrun up deno-zig-stack.ucl
```

## Expected project layout

- `./deno-api` is mounted into the `deno-api` jail at `/srv/app`.
- `./zig-service` is mounted into the `zig-compute` jail at `/srv/app`.
