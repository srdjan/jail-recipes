# zig/latest

Installs the [Zig](https://ziglang.org/) programming language toolchain on a FreeBSD jail.

Zig is installed from the FreeBSD package repository (`lang/zig`), which provides the latest stable version available in the ports tree.

## Package contents

- `playbook.yml` is the canonical install playbook.
- `examples/zig-app/recipe.ucl` is the package-local example recipe.

## Variables

| Variable | Default | Description |
|---|---|---|
| `ZIG_CACHE_DIR` | `/var/cache/zig` | Global cache directory for compiled artifacts |
| `ZIG_INSTALL_CTOOLS` | `no` | Install complementary C/C++ build tools (`cmake`, `ninja`, `pkgconf`, `gmake`) |

## Usage

### From this repo (local)

```sh
jrun up packages/playbooks/zig/latest/examples/zig-app/recipe.ucl
```

### From jailrun-hub (remote)

```ucl
jail "hypha-zig" {
  setup {
    zig {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/packages/playbooks/zig/latest/playbook.yml";
    }
  }
}
```

### With C/C++ build tools

Useful when Zig is used as a cross-compiler or when linking against C libraries:

```ucl
jail "hypha-zig" {
  setup {
    zig {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/packages/playbooks/zig/latest/playbook.yml";
      vars { ZIG_INSTALL_CTOOLS = "yes"; }
    }
  }
}
```

### Full example — Zig HTTP server with live reload

```ucl
jail "zig-app" {
  base { type = "jail"; name = "hypha-zig"; }

  forward {
    http { host = 3000; jail = 3000; }
  }
  mount {
    src { host = "."; jail = "/srv/app"; }
  }
  exec {
    server {
      cmd = "zig build run -- --port 3000";
      dir = "/srv/app";
      healthcheck {
        test = "fetch -qo /dev/null http://127.0.0.1:3000";
        interval = "30s";
        timeout = "10s";
        retries = 5;
      }
    }
  }
}
```

### Build-only jail (compile artifacts for the host)

```ucl
jail "hypha-zig" {
  setup {
    zig {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/packages/playbooks/zig/latest/playbook.yml";
    }
  }
  mount {
    src { host = "."; jail = "/srv/project"; }
  }
}
```

Then build from the host:

```sh
jrun ssh -j hypha-zig -- "cd /srv/project && zig build -Doptimize=ReleaseSafe"
```

## What gets installed

- `zig` compiler and toolchain via FreeBSD `pkg` (port: `lang/zig`)
- A global cache directory at `ZIG_CACHE_DIR`
- System-wide `ZIG_GLOBAL_CACHE_DIR` environment variable in `/etc/profile`
- Optionally: `cmake`, `ninja`, `pkgconf`, `gmake` (when `ZIG_INSTALL_CTOOLS=yes`)

## Notes

- Zig includes a built-in C/C++ compiler (`zig cc` / `zig c++`) and linker, which can be used as a drop-in replacement for `cc` in many build systems.
- Zig supports cross-compilation out of the box. A FreeBSD jail running Zig can target Linux, macOS, and other platforms without additional toolchains.
- The version installed depends on the FreeBSD package repository branch (quarterly or latest). To use the latest packages, configure your jail's pkg repo to point to the `latest` branch.
