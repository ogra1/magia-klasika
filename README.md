# magia-klasika

A strict-confinement snap that provisions an [LXD](https://canonical.com/lxd)
container running the latest Ubuntu LTS and installs a **classic snap** of
your choice inside it.

The snap talks to the host LXD daemon over the `lxd` snap interface using
LXD's REST API (via the container's unix socket), so it works under strict
confinement without needing the `lxc` client binary.

Provisioning runs automatically as a **oneshot service** whenever the
configuration changes, and it is safe to re-run (idempotent).

---

## What it does

When triggered, `magia-klasika`:

1. Acquires a lock so only one instance runs at a time.
2. Verifies it can reach the host LXD socket.
3. Reads its configuration via `snapctl`.
4. Ensures LXD is initialized — if not, it performs the equivalent of
   `lxd init --auto` (default storage pool, `lxdbr0` bridge, default
   profile wiring) over the REST API.
5. Waits for `lxdbr0` to obtain an IPv4 address.
6. Creates (or reuses) an Ubuntu 24.04 LTS container with
   `security.nesting=true` so snaps can run inside it.
7. Starts the container and waits for `snapd` to be ready.
8. Installs `snapd` inside the container if it is missing.
9. Probes container-side connectivity to `api.snapcraft.io`.
10. Installs your configured classic snap (or refreshes it to the configured
    channel if already installed), retrying transient store errors with
    exponential backoff.

---

## How it runs

The snap ships two apps:

| App                     | Type              | Purpose                                                                 |
|-------------------------|-------------------|-------------------------------------------------------------------------|
| `magia-klasika.reconcile` | `daemon: oneshot` | Runs automatically when config changes and at boot; performs the work. |
| `magia-klasika.run`       | command           | On-demand manual trigger of the same logic.                            |

The `reconcile` service is **disabled at install time** (`install-mode:
disable`). The `configure` hook enables and starts it once a valid
`snap-name` is set, and stops/disables it if `snap-name` is cleared.

Because the work is idempotent, re-running (on config change, manual restart,
or reboot) is cheap: if the target snap is already installed, it is refreshed
to the configured channel instead of reinstalled.

---

## Prerequisites

- A Linux host with `snapd`.
- The **LXD snap** installed on the host:

  ```bash
  sudo snap install lxd
  ```

- Your user able to manage the snap (root or a member of the `lxd` group is
  recommended for LXD administration).

> You do **not** need to run `lxd init` yourself — this snap will
> auto-initialize LXD if required. Running `sudo lxd init --auto` beforehand
> is also fine; the snap detects an existing setup and won't disturb it.

---

## Installation

### From the store

```bash
sudo snap install magia-klasika
```

### From a locally built snap

```bash
# Build
snapcraft pack

# Install (unsigned local build)
sudo snap install ./magia-klasika_*.snap --dangerous
```

### Connect the LXD interface

The `lxd` interface is **not auto-connected** unless a store snap declaration
grants it. Connect it manually:

```bash
sudo snap connect magia-klasika:lxd lxd:lxd
```

Verify:

```bash
snap connections magia-klasika
# Should show:  lxd  magia-klasika:lxd  lxd:lxd
```

---

## Configuration

All configuration is done with `snap set`. Values are validated by a
`configure` hook — invalid values are **rejected** and the previous
configuration is preserved. Setting a valid `snap-name` automatically enables
and starts the `reconcile` service.

| Option                 | Required | Default   | Description                                                        |
|------------------------|----------|-----------|--------------------------------------------------------------------|
| `snap-name`            | **Yes**  | —         | Name of the classic snap to install inside the container.          |
| `channel`              | No       | `stable`  | Channel to install from: `[track/]risk[/branch]`.                  |
| `install-max-attempts` | No       | `5`       | Max install attempts on transient errors (positive integer).       |
| `install-base-delay`   | No       | `3`       | Initial backoff between retries, in seconds (positive integer).    |
| `install-max-delay`    | No       | `60`      | Maximum backoff per retry, in seconds (positive integer).          |

### Examples

```bash
# Minimal: install VS Code from the stable channel
sudo snap set magia-klasika snap-name=code

# Choose a channel
sudo snap set magia-klasika snap-name=code channel=latest/stable

# Tune retry behaviour
sudo snap set magia-klasika install-max-attempts=8 \
                            install-base-delay=5 \
                            install-max-delay=90

# Inspect current configuration
snap get magia-klasika

# Clear the target (stops and disables the reconcile service)
sudo snap unset magia-klasika snap-name
```

### Validation rules

| Option                 | Accepted examples                              | Rejected examples                          |
|------------------------|------------------------------------------------|--------------------------------------------|
| `snap-name`            | `code`, `hello-world`, `node-red`              | `Bad_Name`, `-lead`, `trail-`, `a--b`, `123` |
| `channel`              | `stable`, `latest/stable`, `22/beta`, `3.4`    | `dooper`, `a/b/c/d`, `track/notarisk`      |
| `install-max-attempts` | `1`, `5`, `10`                                 | `0`, `-1`, `abc`, `3.5`                    |
| `install-base-delay`   | `3`, `10`                                      | `0`, `1.5`, `x`                            |
| `install-max-delay`    | `60`, `120` (and ≥ effective base delay)       | `0`, `abc`, or a value below the base delay|

> Snap names follow snapd's rules: lowercase letters, digits and single
> hyphens; at least one letter; no leading/trailing hyphen; max 40 chars.

---

## Running and monitoring

### Automatic (recommended)

```bash
sudo snap set magia-klasika snap-name=code channel=latest/stable
```

### Manual trigger

```bash
# Re-run the reconcile service on demand
sudo snap restart magia-klasika.reconcile

# Or run the on-demand app directly (foreground)
magia-klasika.run
```

### Watching progress

```bash
# Follow the service logs
sudo journalctl -u snap.magia-klasika.reconcile -f

# Service status
snap services magia-klasika
```

> A `oneshot` service shows `inactive` after it completes successfully —
> that is normal; it ran to completion.

The container is named `magia-klasika-container` and is **reused** on
subsequent runs.

---

## Working with the container

```bash
lxc list
lxc exec magia-klasika-container -- bash
lxc exec magia-klasika-container -- snap list
```

Remove the container (a fresh one is created on the next reconcile):

```bash
lxc delete --force magia-klasika-container
```

---

## Troubleshooting

Start by reading the service logs:

```bash
sudo journalctl -u snap.magia-klasika.reconcile -e
```

### `LXD socket not found ...`

```bash
sudo snap install lxd
sudo snap connect magia-klasika:lxd lxd:lxd
snap connections magia-klasika
```

### `No snap configured.`

```bash
sudo snap set magia-klasika snap-name=<name>
```

### `Another lxd-wrapper instance is running; exiting.`

A run is already in progress. This exits cleanly with success; the other
instance is handling the work. Wait and check the logs.

### Configuration change rejected

The `configure` hook rejected an invalid value; the previous config is kept.
Read the error message — it names the offending option and the rule.

### `Failed to create storage pool` / `Failed to create network`

Usually a permissions problem: the snap needs full LXD admin access. Confirm
the `lxd` interface is connected. If a `zfs`/`btrfs` backend fails, the snap
falls back to `dir` automatically.

### `Network 'lxdbr0' did not obtain an IPv4 address ...`

```bash
lxc network show lxdbr0
lxc network get lxdbr0 ipv4.address   # should be an address, not "none"
```

### `Container could not resolve api.snapcraft.io ...`

```bash
lxc network get lxdbr0 ipv4.nat       # expect "true"
lxc exec magia-klasika-container -- getent hosts api.snapcraft.io
lxc exec magia-klasika-container -- ping -c1 1.1.1.1
```

### `snap install failed after retries`

```bash
sudo snap set magia-klasika install-max-attempts=10 install-max-delay=120
sudo snap restart magia-klasika.reconcile
```

### `Permanent error installing ...; not retrying.`

```bash
snap info <snap-name>   # confirm it exists and is classic-confined
```

### Service keeps restarting

```bash
sudo snap stop --disable magia-klasika.reconcile
# fix the root cause, then:
sudo snap start --enable magia-klasika.reconcile
```

### Nested snaps fail to install inside the container

```bash
lxc delete --force magia-klasika-container
sudo snap restart magia-klasika.reconcile
```

---

## Uninstalling

```bash
sudo snap stop --disable magia-klasika.reconcile
lxc delete --force magia-klasika-container
sudo snap remove magia-klasika
```

---

## Project layout

```
magia-klasika/
├── snap/
│   ├── snapcraft.yaml        # metadata, reconcile (oneshot) + run apps, hook
│   └── hooks/
│       └── configure         # validates config + controls reconcile service
├── bin/
│   └── lxd-wrapper           # main wrapper (lock, init, install, retry)
└── README.md
```

---

## Notes & limitations

- Confinement is **strict**; LXD access is granted solely through the
  `lxd` interface (manually connected unless a store declaration auto-connects
  it).
- The `reconcile` service is a `oneshot` daemon: it runs to completion rather
  than staying resident.
- Runs are guarded by a lock file in `$SNAP_COMMON`, so concurrent
  invocations are serialized; a contending run exits successfully as a no-op.
- Operation is idempotent: an already-installed target is refreshed to the
  configured channel rather than reinstalled.
- Individual LXD API calls use a curl timeout so a hung daemon cannot hang
  the service indefinitely.
- The target Ubuntu LTS is pinned to `24.04` in the wrapper
  (`UBUNTU_LTS_ALIAS`).
- The container is long-lived and reused between runs; the snap does not
  delete it automatically.

---

## License

GPL-3.0
