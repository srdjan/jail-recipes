# deno/latest

Installs the [Deno](https://deno.land/) JavaScript/TypeScript runtime on a FreeBSD jail.

Deno is installed from the FreeBSD package repository (`www/deno`), which provides the latest stable version available in the ports tree.

## Package contents

- `playbook.yml` is the canonical install playbook.
- `examples/deno-app/recipe.ucl` is the package-local example recipe.

## Variables

| Variable | Default | Description |
|---|---|---|
| `DENO_INSTALL_DIR` | `/usr/local/bin` | Directory where the `deno` binary is available |
| `DENO_DIR` | `/var/cache/deno` | Cache directory for downloaded modules and compiled artifacts |

## Usage

### From this repo (local)

```sh
jrun up playbooks/deno/latest/examples/deno-app/recipe.ucl
```

Compatibility alias:

```sh
jrun up deno-app.ucl
```

### From jailrun-hub (remote)

```ucl
jail "hypha-deno" {
  setup {
    deno {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/playbooks/deno/latest/playbook.yml";
    }
  }
}
```

### With custom cache directory

```ucl
jail "hypha-deno" {
  setup {
    deno {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/playbooks/deno/latest/playbook.yml";
      vars { DENO_DIR = "/srv/app/.deno"; }
    }
  }
}
```

### Full example — Deno HTTP server

```ucl
jail "deno-app" {
  base { type = "jail"; name = "hypha-deno"; }

  forward {
    http { host = 8000; jail = 8000; }
  }
  mount {
    src { host = "."; jail = "/srv/app"; }
  }
  exec {
    server {
      cmd = "deno run --allow-net --allow-read /srv/app/main.ts";
      dir = "/srv/app";
      healthcheck {
        test = "fetch -qo /dev/null http://127.0.0.1:8000";
        interval = "30s";
        timeout = "10s";
        retries = 5;
      }
    }
  }
}
```

## What gets installed

- `deno` binary via FreeBSD `pkg` (port: `www/deno`)
- A module cache directory at `DENO_DIR`
- System-wide `DENO_DIR` environment variable in `/etc/profile`

## Notes

- The official Deno install script (`curl -fsSL https://deno.land/install.sh | sh`) downloads a Linux ELF binary that **will not work** on FreeBSD. This playbook uses the native FreeBSD port instead.
- The version installed depends on the FreeBSD package repository branch (quarterly or latest). To use the latest packages, configure your jail's pkg repo to point to the `latest` branch.
