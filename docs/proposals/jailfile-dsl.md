# Proposal: Jailfile - a high-level DSL for jailrun

## Problem

Today, running a Deno server in a jail requires writing 45 lines of UCL across two jail blocks, knowing the Ansible playbook URL, understanding the base-jail cloning pattern, and configuring mount/forward/exec/healthcheck manually. Adding a second tool (say, claude-codex) roughly doubles the setup block boilerplate. None of this complexity is inherent to the user's intent, which is: "give me a jail with Deno and Codex, mount my code, expose port 8000."

The raw UCL recipe format is powerful and should remain the low-level primitive. But most users need a happy-path configuration that expresses what they want, not how to provision it.

## Design

Introduce a `jailfile.json` (working name) that sits in a project root, analogous to `deno.json` or `package.json`. It declares dependencies on jailrun-hub packages using an import map, and the `jrun` CLI resolves those imports into the full UCL + playbook graph at deploy time.

### Minimal example

A Deno server with Claude Codex installed:

```json
{
  "name": "my-app",
  "imports": {
    "deno": "jailrun:deno@latest",
    "claude-codex": "jailrun:claude-codex@latest"
  }
}
```

That is the entire file. Running `jrun up` in that directory would:

1. Resolve each import to a jailrun-hub package (e.g. `packages/playbooks/deno/latest/`)
2. Generate a base jail that composes all imported playbooks
3. Clone it into an app jail with sensible defaults: mount `.` at `/srv/app`, no port forwards, no exec

The result is a jail you can SSH into (`jrun ssh my-app`) with Deno and Codex ready to use.

### Adding behavior

Most projects need port forwards and a start command. These are declared at the top level:

```json
{
  "name": "my-api",
  "imports": {
    "deno": "jailrun:deno@latest"
  },
  "mount": ".",
  "forward": [8000],
  "exec": "deno run --allow-net --allow-read --allow-env --watch /srv/app/main.ts"
}
```

`mount`, `forward`, and `exec` are optional shorthand:

| Field | Type | Expands to |
|-------|------|------------|
| `mount` | `string` | `mount { src { host = "<value>"; jail = "/srv/app"; } }` |
| `mount` | `object` | `mount { <key> { host = "<host>"; jail = "<jail>"; } }` for each entry |
| `forward` | `number[]` | `forward { p<N> { host = N; jail = N; } }` for each port |
| `forward` | `object` | `forward { <key> { host = <host>; jail = <jail>; } }` for each entry |
| `exec` | `string` | Single supervised process with a default healthcheck |
| `exec` | `object` | Named processes, each with cmd/dir/env/healthcheck |

When `exec` is a bare string, the runtime is inferred from imports. If the Jailfile imports `deno` and the command starts with `deno`, the healthcheck defaults to an HTTP fetch against the port in `forward`. If no port is forwarded, no healthcheck is generated.

### Full form

For cases that need more control without dropping to raw UCL:

```json
{
  "name": "my-stack",
  "imports": {
    "deno": "jailrun:deno@latest",
    "postgres": "jailrun:postgres@16"
  },
  "jails": {
    "db": {
      "uses": ["postgres"],
      "forward": [5432],
      "vars": {
        "POSTGRES_DB": "myapp",
        "POSTGRES_USER": "dev"
      }
    },
    "api": {
      "uses": ["deno"],
      "depends": ["db"],
      "mount": ".",
      "forward": [8000],
      "exec": "deno run --allow-all --watch /srv/app/main.ts",
      "env": {
        "DATABASE_URL": "postgresql://dev@db/myapp"
      }
    }
  }
}
```

When `jails` is present, each key becomes a separate jail. The `uses` array selects which imports apply to that jail. Dependencies, env vars, and per-jail overrides all live here. Jails still discover each other by name on the internal network - `db` in the DATABASE_URL above resolves automatically.

## Import resolution

The import specifier format is `jailrun:<package>@<version>`:

```
jailrun:deno@latest       -> packages/playbooks/deno/latest/
jailrun:deno@1.45         -> packages/playbooks/deno/1.45/
jailrun:postgres@16       -> packages/playbooks/postgres/16/
jailrun:claude-codex@latest -> packages/playbooks/claude-codex/latest/
```

Resolution follows this order:

1. Check local `packages/playbooks/<name>/<version>/` if inside a jailrun-hub checkout
2. Fetch from the configured jailrun-hub registry (default: GitHub)
3. Fail with a clear error listing available packages

Each package must contain a `manifest.json` that declares its playbook path and default configuration:

```json
{
  "name": "deno",
  "version": "latest",
  "playbook": "playbook.yml",
  "defaults": {
    "vars": {
      "DENO_DIR": "/var/cache/deno"
    }
  }
}
```

The `defaults` block lets packages provide sensible values that users can override via `vars` in the Jailfile.

## Composing multiple imports into a single jail

When a Jailfile imports multiple packages into one jail (the flat format, or a `jails` entry with multiple `uses`), the resolver must produce a single base jail whose setup block chains all the playbooks in declaration order. This is straightforward since Ansible playbooks are idempotent and additive - installing Deno then installing Codex produces a jail with both.

The generated UCL for the minimal example would look like:

```ucl
jail "my-app-base" {
  setup {
    deno {
      type = "ansible";
      url  = "jailrun-hub:.../deno/latest/playbook.yml";
    }
    claude-codex {
      type = "ansible";
      url  = "jailrun-hub:.../claude-codex/latest/playbook.yml";
    }
  }
}

jail "my-app" {
  base { type = "jail"; name = "my-app-base"; }
  mount {
    src { host = "."; jail = "/srv/app"; }
  }
}
```

## Escape hatch

Some configurations need raw UCL. Rather than trying to cover every UCL feature in JSON, the Jailfile supports a `recipe` field that points to a UCL file:

```json
{
  "name": "custom",
  "imports": {
    "deno": "jailrun:deno@latest"
  },
  "recipe": "./custom.ucl"
}
```

When `recipe` is present, the Jailfile handles only import resolution and base jail generation. The referenced UCL file defines the app jails and can reference the generated base jails by the conventional name `<import-name>-base` (e.g. `deno-base`).

This keeps the Jailfile as the single entry point while allowing power users to drop to UCL for advanced topologies.

## CLI changes

The `jrun` CLI would gain Jailfile awareness:

```sh
jrun up                   # Auto-detects jailfile.json in current directory
jrun up jailfile.json     # Explicit path
jrun up recipe.ucl        # Still works, no change to existing behavior
```

When given a Jailfile, `jrun` resolves imports, generates UCL in a `.jailrun/` cache directory, and deploys from there. The generated UCL is inspectable for debugging:

```sh
jrun resolve              # Print the generated UCL without deploying
jrun resolve --write      # Write it to .jailrun/recipe.ucl
```

## Migration path

This is purely additive. Existing UCL recipes continue to work unchanged. The Jailfile is syntactic sugar that compiles down to UCL. Users can start with a Jailfile and eject to raw UCL at any time by running `jrun resolve --write` and switching to `jrun up .jailrun/recipe.ucl`.

## Open questions

1. **File format** - JSON is familiar and tooling-friendly, but lacks comments. JSONC (JSON with comments, used by deno.json and tsconfig.json) would be a better fit. Alternatively, UCL itself could serve as the Jailfile format since it already supports comments and is less verbose than JSON, but that may defeat the purpose of providing a simpler abstraction.

2. **Version pinning and lock files** - Should `jrun` generate a `jailfile.lock` that pins resolved playbook SHAs? This would make builds reproducible but adds complexity. The `@latest` convention suggests users opt into floating versions deliberately.

3. **Package composition conflicts** - Two packages might install conflicting system packages or write to the same config file. The current design runs playbooks sequentially and relies on Ansible's idempotency, but a manifest-level conflict declaration (e.g. `"conflicts": ["node"]`) could catch issues earlier.

4. **Inline playbooks** - Should the Jailfile support declaring simple setup steps inline (e.g. `"run": ["pkg install -y curl", "pip install httpie"]`) for one-off needs that do not warrant a full package? This is convenient but blurs the line between the Jailfile and raw UCL.

5. **Stack packages** - The current repo has `packages/stacks/` for multi-jail compositions. Should stacks be importable too (e.g. `jailrun:stacks/deno-zig@latest`), or should multi-jail setups always be expressed in the `jails` block?
