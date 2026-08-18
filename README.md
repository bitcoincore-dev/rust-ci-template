# rust-ci

Shared **GitHub Actions composite actions** for Rust repositories in the **BTCDecoded** org. The goal is one place to define how we install Rust, normalize CI for crates.io builds, and operate safely on **self-hosted** runners (disk and Cargo caches).

This repository has **no crates**—only reusable workflow building blocks. Workflows reference actions with:

`uses: BTCDecoded/rust-ci/<action-name>@main`

(or a pinned SHA for reproducibility).

## Starter workflows (GitHub template)

The directory [`.github/workflow-templates/`](.github/workflow-templates/) contains **GitHub org-level starter workflows** that appear under *Actions → New workflow* for any repository in the org.  
Each `.yml` file is paired with a `.properties.json` metadata file.

### Template variables

The workflow files use two kinds of placeholder that must be replaced before the workflow runs:

| Variable | Kind | Description |
| --- | --- | --- |
| `$default-branch` | GitHub built-in | Substituted automatically by GitHub with the repo's default branch when a user instantiates the workflow. Do **not** replace manually. |
| `{{ actions_org }}` | Handlebar | GitHub org or user that owns the rust-ci actions repo (e.g. `bitcoincore-dev`). |
| `{{ actions_repo }}` | Handlebar | Name of the actions repository (e.g. `rust-ci-template`). |
| `{{ actions_ref }}` | Handlebar | Ref or SHA to pin the actions (e.g. `main`, or a commit SHA for reproducibility). |
| `{{ runner_label }}` | Handlebar | Self-hosted runner label(s) — `rust-ci-self-hosted.yml` only. |
| `{{ cache_root }}` | Handlebar | Persistent cache directory on the runner host — `rust-ci-self-hosted.yml` only. |

**Example substitution** (GitHub-hosted workflow):

```yaml
# before
- uses: {{ actions_org }}/{{ actions_repo }}/install-rust-toolchain@{{ actions_ref }}

# after
- uses: bitcoincore-dev/rust-ci-template/install-rust-toolchain@main
```

Replace all `{{ … }}` placeholders with `sed` or your editor's find-and-replace before committing the workflow to your repository.

## Actions

### `install-rust-toolchain`

Wraps [`dtolnay/rust-toolchain`](https://github.com/dtolnay/rust-toolchain) with an org **default pinned** toolchain when you do not use a repo `rust-toolchain.toml`.

| Input | Role |
| --- | --- |
| `toolchain` | Version string (default **1.88.0**). Ignored if `toolchain-file` is set. |
| `toolchain-file` | Path to `rust-toolchain.toml` or `rust-toolchain`; channel comes from that file. |
| `components` | Extra components (`rustfmt`, `clippy`, …). With `toolchain-file`, comma-separated for dtolnay; otherwise passed to `rustup component add` after the pinned install. |

Use `toolchain-file` when the repo owns the channel; use the default pin when you want org-wide consistency without per-repo files.

**Self-hosted runners (shared disk, concurrent jobs):** If you see `rustc: Text file busy` (exit 126 / `ETXTBSY`) during `install-rust-toolchain`, rustup is usually replacing the `~/.cargo/bin` proxies while another job is executing them. This action exports `RUSTUP_PERMIT_COPY_RENAME` and `RUSTUP_NO_SELF_UPDATE` for all steps so file updates use a safer pattern and rustup does not self-update mid-job.

**Multiple listeners, one Unix user (e.g. Zeus `zeus-workstation` + `zeus-workstation-2`):** Both services often run as **`github-runner`** with the same **`HOME=/var/lib/github-runner`**, so concurrent jobs on different listeners still collide on **`~/.cargo/bin/cargo`**. Fix at the host:

1. **`packaging/github-runner/runner-toolchain-isolation.conf`** — systemd drop-in per extra listener (`HOME` / `CARGO_HOME` / `RUSTUP_HOME` under e.g. `/var/lib/github-runner-2`).
2. **`packaging/github-runner/bootstrap-isolated-runner-toolchain.sh`** — one-time seed of rustup + **`cargo-audit`** for that home.
3. Workflows: **`isolate-cargo-env`** per job + invoke **`cargo-audit`** (not **`cargo audit`**) when auditing; pre-install audit on the runner so jobs skip **`cargo install`**.

If flakes persist after host isolation, reduce parallelism (workflow **`concurrency`**) or use fully job-local **`RUSTUP_HOME`** (slower: reinstalls toolchain per job).

### `strip-patch-crates-io`

Removes **`[patch.crates-io]`** sections from `Cargo.toml` and `.cargo/config.toml` under a chosen directory so **CI resolves dependencies from crates.io** instead of local path overrides (typical monorepo / path-dev setup).

| Input | Role |
| --- | --- |
| `working-directory` | Root to search (default `.`). |

Run this **before** `cargo fetch` / `cargo build` in workflows that must behave like downstream crates.io consumers.

### `runner-disk-guard` (self-hosted)

If **root filesystem (`/`)** *or* the **filesystem containing `cache-root`** is above a threshold, prunes old **`cache-root`** subtrees (default age **7 days**, maxdepth **2**) so long-lived runners do not fill the disk. That includes the common case where **`/tmp` is a small tmpfs**, **`cache-root` defaults to `/tmp/runner-cache`**, and **`/` still has free space**—previously the guard only looked at `/` and would never prune.

| Input | Role |
| --- | --- |
| `cache-root` | Directory to prune when over threshold. |
| `threshold-percent` | Trigger cleanup when **either** measured use exceeds this percent (default **80**). |
| `show-df` | If true, log `df -h` before/after when cleanup runs. |

### `isolate-cargo-env` (self-hosted)

Sets **`CARGO_HOME`** to **`${RUNNER_TEMP}/<subdir>`** (default `cargo`) and prepends **`$CARGO_HOME/bin`** to **`PATH`**. Symlinks **`registry`** and **`git`** to the runner user's shared store (default **`$HOME/.cargo/...`**) so per-job metadata stays isolated without duplicating downloaded crates.

Use **early in the job**, before **`install-rust-toolchain`** / **`cargo install`**, on workflows that run on shared self-hosted users. Pairs with **`runner-cargo-cache`** (restore still populates the shared registry; isolated **`CARGO_HOME`** picks it up via symlink).

| Input | Role |
| --- | --- |
| `cargo-home-subdir` | Subdir under **`RUNNER_TEMP`** (default **`cargo`**). |
| `shared-registry` | Override registry symlink target (default **`$HOME/.cargo/registry`**). |
| `shared-git` | Override git symlink target (default **`$HOME/.cargo/git`**). |

### `runner-cargo-cache` (self-hosted)

Manages **persistent Cargo/registry/git and optional `target`** caches on the runner: bind environment variables from a cache key, **restore** before builds, **save** after builds, or **prune** old cache directories.

| Input | Role |
| --- | --- |
| `operation` | **`bind-env`** \| **`restore`** \| **`save`** \| **`prune`** (required). |
| `cache-key` | Segment used to isolate cache trees (required for `bind-env`). |
| `cache-root` | Root for cache dirs (default `/tmp/runner-cache`). |
| `include-target` | For `bind-env`, whether to set `TARGET_CACHE_DIR`. |
| `save-target-only` | For `save`, only sync `./target` (no registry/git). |

Typical sequence: `bind-env` → `restore` → build → `save` (and occasional `prune` in maintenance jobs).

### `setup-blvm-spec`

Clones the [blvm-spec](https://github.com/BTCDecoded/blvm-spec) (Orange Paper) repository for **spec-lock** verification, mdBook includes, or any job that needs spec markdown on disk.

| Input | Role |
| --- | --- |
| `working-directory` | Repo root for path resolution (default `.`). |
| `target` | Empty (default): clone to **`../blvm-spec`** (sibling of the checked-out repo), matching `cargo-spec-lock verify --spec-path ../blvm-spec/...`. Set to a path **relative to the repo** (e.g. `modules/blvm-spec`) for in-tree checkouts. |
| `repository` | Git URL (default official `blvm-spec` repo). |
| `depth` | Shallow clone depth (default `1`). |

## Typical usage

**GitHub-hosted:** `install-rust-toolchain` and often `strip-patch-crates-io` are enough.

**Self-hosted:** add `runner-disk-guard` early in the job, and use `runner-cargo-cache` around the compile steps so repeated builds stay fast without unbounded disk growth.

## Self-hosted runners: cache location (avoid filling tmpfs `/tmp`)

Default **`cache-root`** is **`/tmp/runner-cache`**. On many Linux setups **`/tmp` is tmpfs (RAM)**. Cargo/registry + `target` caches can grow to tens of GiB and **fill `/tmp` entirely**, breaking anything that writes there (e.g. rustup’s `rustup-init` download). **`/` may still have plenty of space.**

**Do one of the following (recommended: A).**

### A. Symlink the default path to real disk (no workflow edits)

Pick a directory on a **large persistent volume** (example: `/mnt/data/github-runner-cache`). Workflows that keep the default **`/tmp/runner-cache`** will follow the symlink and use disk.

1. **Find and stop the runner service(s)** (units look like `actions.runner.<org>-<repo>.<hostname>.service`). Stop every **enabled** runner on this machine:

   ```bash
   systemctl list-unit-files --no-legend \
     | awk '/^actions\.runner\..*\.service/ && $2 == "enabled" {print $1}'
   ```

   ```bash
   systemctl list-unit-files --no-legend \
     | awk '/^actions\.runner\..*\.service/ && $2 == "enabled" {print $1}' \
     | xargs -r sudo systemctl stop
   ```

   Wait until jobs drain; optionally confirm nothing is writing the cache:  
   `sudo lsof +D /tmp/runner-cache 2>/dev/null | head` (empty is ideal).

2. **Find the Unix user (and group) the runner runs as** (needed for `chown` and the write test). Use any **enabled** `actions.runner` unit (after stop, `list-units` may omit idle services, so use unit files):

   ```bash
   UNIT=$(systemctl list-unit-files --no-legend \
     | awk '/^actions\.runner\..*\.service/ && $2 == "enabled" {print $1; exit}')
   systemctl cat "$UNIT" | grep -E '^User=|^Group='
   ```

   If **`Group=`** is missing, use the same name as **`User=`** for `RUNNER_GROUP`, or `id -gn RUNNER_USER`. Put those in place of **`RUNNER_USER`** / **`RUNNER_GROUP`** in step 3 and 6.

3. **Create the disk directory, migrate cache, replace `/tmp/runner-cache` with a symlink**

   If **`/tmp/runner-cache` is already a symlink**, `rm -rf /tmp/runner-cache` only removes the link (not the target). If it is a real directory, this deletes/recreates the path under `/tmp`.

   ```bash
   DISK_CACHE=/mnt/data/github-runner-cache   # change to your volume
   sudo mkdir -p "$DISK_CACHE"

   if [ -d /tmp/runner-cache ] && [ ! -L /tmp/runner-cache ]; then
     sudo rsync -a /tmp/runner-cache/ "$DISK_CACHE/"
   elif [ -L /tmp/runner-cache ]; then
     echo "Existing symlink: $(readlink -f /tmp/runner-cache) — copy from there if you still want migration."
   fi

   sudo rm -rf /tmp/runner-cache
   sudo ln -s "$DISK_CACHE" /tmp/runner-cache

   sudo chown -R RUNNER_USER:RUNNER_GROUP "$DISK_CACHE"
   ```

   Replace **`RUNNER_USER:RUNNER_GROUP`** with the values from step 2 (e.g. `josh:josh` or `runner:runner`).

4. **Optional — SELinux (enforcing):** if the new path is not on a typical home/data label, you may need a context the runner can write, e.g. after policy review:

   ```bash
   sudo semanage fcontext -a -t var_lib_t "/mnt/data/github-runner-cache(/.*)?" 2>/dev/null || true
   sudo restorecon -RFv /mnt/data/github-runner-cache
   ```

   Use the **same path** as **`DISK_CACHE`** in step 3. Adjust the SELinux **type** to match your distribution if the journal still shows denials.

5. **Start the runner(s) again**

   Prefer starting the **same unit names you stopped** in step 1. If you did not record them, start every **enabled** runner service:

   ```bash
   systemctl list-unit-files --no-legend \
     | awk '/^actions\.runner\..*\.service/ && $2 == "enabled" {print $1}' \
     | xargs -r sudo systemctl start
   ```

6. **Verify** (replace **`RUNNER_USER`** — same as step 2)

   ```bash
   readlink -f /tmp/runner-cache          # should print your DISK_CACHE path
   df -P /tmp/runner-cache                # should show the disk filesystem, not tmpfs
   sudo -u RUNNER_USER sh -c 'touch /tmp/runner-cache/.write-test && rm /tmp/runner-cache/.write-test'
   ```

**Notes**

- **several runners on one host** — they usually share the same default **`/tmp/runner-cache`**; one symlink updates all of them.
- **No systemd** (you only run `./run.sh` / `runsvc.sh`) — stop those processes, perform steps 3–4 as the **same user** that runs the runner (skip `systemctl`; use `chown -R` only if you created the tree as root), then start the listener again.

### B. Set `cache-root` (and disk-guard’s `cache-root`) to a disk path

Use the same directory in **`runner-cargo-cache`** and **`runner-disk-guard`** inputs (org **Variables** help). No symlink; every workflow must pass the path (or inherit from a reusable workflow).

### C. Host maintenance

- Cron or a nightly job can run **`runner-cargo-cache`** with **`operation: prune`** (same **`cache-root`**).
- **`TMPDIR` on disk:** for systemd runners, add e.g. `Environment=TMPDIR=/var/tmp` via `systemctl edit <actions.runner.unit>` so rustup and other tools avoid a full tmpfs. Create **`/var/tmp`** (or your choice) with normal permissions if needed.

## Contributing

Changes here affect **every workflow** that pins `@main` (or your tag). Prefer small, backward-compatible input defaults; document new inputs in each action’s `action.yml` and update this README when behavior or defaults change.
