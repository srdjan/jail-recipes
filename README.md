# JailRun Recipes

This repository is organized as a package-first monorepo for [Jailrun](https://github.com/hyphatech/jailrun) playbooks.

## How Jailrun Works

Jailrun boots a FreeBSD virtual machine on your Mac or Linux host and manages lightweight isolated environments called jails inside it. Each jail gets its own filesystem, network stack, and processes - nothing inside a jail can see or touch anything outside of it.

Under the hood, Jailrun orchestrates several tools together:

- **QEMU** runs FreeBSD with hardware acceleration (HVF on macOS, KVM on Linux)
- **Bastille** manages jail lifecycles
- **Ansible** provisions software inside jails via playbooks
- **ZFS** enables instant jail clones through snapshots
- **9p filesystem** shares host directories into jails in real time
- **pf** (FreeBSD packet filter) handles port forwarding from host to jail
- **Monit** supervises processes, runs healthchecks, and auto-restarts on failure

You describe your application in a `.ucl` recipe file. A recipe defines one or more jails, each with its own setup (what software to install), mounts (which host directories to share), port forwards, and supervised processes. When you run `jrun up recipe.ucl`, Jailrun reads the recipe, creates the jails, runs the Ansible playbooks to provision them, mounts your project directories, sets up port forwarding, and starts your processes under supervision.

### Key concepts

**Recipes** are `.ucl` configuration files that define jails. UCL is a human-friendly, JSON-like format. A single recipe can define multiple jails with dependencies between them.

**Playbooks** are Ansible YAML files that install software inside a jail. A playbook runs once during jail creation - for example, `pkg install deno` and configuring environment variables. Playbooks can be local files or remote URLs from jailrun-hub.

**Base jails** let you clone an already-provisioned jail instead of building from scratch. You define a jail with a `setup` block (which runs Ansible), then other jails reference it via `base { type = "jail"; name = "..."; }` to get an instant ZFS clone with all the software already installed.

**Jail blocks** in a recipe support these sections:

| Section | Purpose |
|---------|---------|
| `setup` | Ansible playbooks to run during jail creation |
| `base` | Clone from an existing provisioned jail |
| `mount` | Share host directories into the jail |
| `forward` | Expose jail ports on the host |
| `exec` | Supervised processes with healthchecks |
| `depends` | Other jails that must start first |

Jails on the same VM automatically discover each other by name on the internal network, so a Deno app can call `fetch("http://zig-compute:3000/compute")` without any extra configuration.

## Getting Started: Deno on FreeBSD

This walkthrough sets up a Deno HTTP server running inside a FreeBSD jail using the playbook and example recipe from this repo.

### Prerequisites

Install Jailrun:

```sh
# macOS
brew tap hyphatech/jailrun
brew install jailrun

# Linux (requires qemu-system, mkisofs, ansible)
pipx install jailrun
```

### Step 1: Start the VM

```sh
jrun start
```

On first run this downloads a FreeBSD image and boots the VM. Subsequent starts reuse the existing image.

### Step 2: Write your application

Create a project directory and add a `main.ts`:

```ts
Deno.serve({ port: 8000 }, (_req) => new Response("Hello from Deno on FreeBSD!"));
```

### Step 3: Write a recipe

Create a `recipe.ucl` in the same directory. This recipe defines two jails - a base jail that installs Deno, and an application jail that runs your server:

```ucl
jail "hypha-deno" {
  setup {
    deno {
      type = "ansible";
      url  = "https://github.com/hyphatech/jailrun-hub/blob/main/packages/playbooks/deno/latest/playbook.yml";
    }
  }
}

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
      cmd = "deno run --allow-net --allow-read --allow-env --watch /srv/app/main.ts";
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

Here is what each part does:

1. The `hypha-deno` jail has a `setup` block that runs the Deno Ansible playbook. This installs Deno from FreeBSD packages and configures the cache directory. This jail exists only to be cloned.

2. The `deno-app` jail uses `base` to clone `hypha-deno`, so it starts with Deno already installed. It forwards port 8000 from host to jail, mounts the current directory at `/srv/app`, and runs the Deno server under supervision with a healthcheck.

### Step 4: Deploy

```sh
jrun up recipe.ucl
```

Jailrun creates `hypha-deno` first (running the Ansible playbook), then clones it to create `deno-app`, mounts your project directory, sets up port forwarding, and starts the server.

### Step 5: Verify

```sh
curl localhost:8000
# Hello from Deno on FreeBSD!
```

You can also SSH into the jail directly:

```sh
jrun ssh deno-app
```

### Step 6: Develop

Because the project directory is mounted via 9p, edits to `main.ts` on your host are visible inside the jail immediately. The `--watch` flag on the Deno command triggers a reload on every save.

### Managing jails

```sh
jrun status            # Show all jails and their state
jrun status --tree     # Hierarchical view with dependencies
jrun pause recipe.ucl  # Stop processes but preserve state
jrun down recipe.ucl   # Destroy all jails in the recipe
jrun stop              # Shut down the VM gracefully
jrun purge             # Destroy the VM and all jails
```

### Writing your own playbook

If you need a runtime or tool that is not in this repo, create a new Ansible playbook. The Deno playbook at `packages/playbooks/deno/latest/playbook.yml` is a good template - it installs a package, creates a cache directory, sets an environment variable, and verifies the installation. See the "Adding a playbook" section below for the repo conventions.

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
