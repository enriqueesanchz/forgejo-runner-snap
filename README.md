# forgejo-runner-snap

Snap packaging for [Forgejo Runner](https://code.forgejo.org/forgejo/runner): a daemon that connects to a Forgejo instance and executes Forgejo Actions jobs.

This snap runs the runner as a system daemon and integrates with the Docker snap for job execution.

## What’s in this repo

- `snap/snapcraft.yaml`: Snap build recipe (builds the upstream runner from source).
- `snap/hooks/install`: Writes a default config to `$SNAP_COMMON/forgejo-config.yaml` on first install.
- `snap/hooks/configure`: Validates required snap settings, ensures Docker is present, registers the runner, and restarts the daemon.
- `snap/local/default-config.yaml`: Default runner configuration copied into `$SNAP_COMMON`.
- `Makefile`: Convenience targets for building/packing (and an example deploy target).

## Requirements

- `snapcraft` (for building locally)
- A Forgejo instance with Actions enabled
- The Docker snap (the snap will attempt to install it if missing)

## Build

From the repo root:

```bash
make clean   # optional
make build
make pack
```

This produces a local `.snap` named like `forgejo-runner_<version>_amd64.snap`.

## Install (local build)

Install the packed snap locally:

```bash
sudo snap install ./forgejo-runner_*.snap --dangerous
```

If you are iterating on confinement/interfaces you may temporarily need `--devmode`, but the intended mode is strict confinement.

## Connect interfaces

The runner needs network access and Docker access.

Check connections:

```bash
snap connections forgejo-runner
```

If they are not auto-connected, connect them manually:

```bash
sudo snap connect forgejo-runner:network
sudo snap connect forgejo-runner:docker docker:docker-daemon
sudo snap connect forgejo-runner:docker-executables docker:docker-executables
```

(Exact plug/slot names may vary by Docker snap version; `snap connections docker` will show available slots.)

## Configure

Two snap settings are required:

- `host`: Forgejo instance URL (e.g. `https://forgejo.example.com`)
- `secret`: runner registration secret from Forgejo

NB! The secret is NOT the forgejo runner registraion token, but obtained by running the following commands on the Forgejo server:
```
SECRET=$(forgejo forgejo-cli actions generate-secret)
forgejo forgejo-cli actions register --secret $SECRET --labels "docker" --labels "runner" # etc.
```

Set them like this:

```bash
sudo snap set forgejo-runner host="https://forgejo.example.com" secret="<RUNNER_SECRET>"
```

What happens next:

- The `configure` hook verifies both values are set.
- It ensures Docker is installed (`snap install docker || true`).
- It runs `forgejo-runner create-runner-file ... -c $SNAP_COMMON/forgejo-config.yaml`.
- It restarts the daemon.

You should then see the runner appear in the Forgejo UI (named like `$(hostname)-runner`).

## Configuration file

On install, the snap writes a default config file (only if it doesn’t already exist):

- `/var/snap/forgejo-runner/common/forgejo-config.yaml`

You can edit this file to tune runner behavior (capacity, labels, container options, etc.), then restart the service:

```bash
sudo snap restart forgejo-runner
```

The default template comes from `snap/local/default-config.yaml`.

## Proxy support

If your environment requires HTTP/HTTPS proxies, set them via snap settings:

```bash
sudo snap set forgejo-runner \
  proxy.http="http://proxy.example.com:3128" \
  proxy.https="http://proxy.example.com:3128" \
  proxy.no-proxy="localhost,127.0.0.1,.internal"
```

The proxy variables are applied in two places:

1. **Runner daemon**: the daemon process itself inherits `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` (and their lowercase equivalents), so outbound connections to Forgejo and container registries go through the proxy.
2. **CI job containers**: the same variables are written to the runner's `env_file` so that jobs spawned by Docker also get the proxy settings.

## Service management & logs

```bash
sudo snap services forgejo-runner
sudo snap restart forgejo-runner
sudo snap logs -f forgejo-runner
```

## Development notes

- The runner is built from upstream using `source-tag: v$SNAPCRAFT_PROJECT_VERSION` as set in `snap/snapcraft.yaml`.
- The `Makefile` includes a `deploy` target that SCPs to a hard-coded host; treat it as an example and adjust/remove for your environment.

## Troubleshooting

- If the runner doesn’t register, confirm `host` and `secret`:
  ```bash
  snap get forgejo-runner
  ```
- Check interface connections:
  ```bash
  snap connections forgejo-runner
  ```
- Inspect logs:
  ```bash
  sudo snap logs -n 200 forgejo-runner
  ```
- Confirm Docker is installed and running:
  ```bash
  snap services docker
  ```

## References

- Forgejo Runner upstream: https://code.forgejo.org/forgejo/runner
- Forgejo docs (runner/admin): https://forgejo.org/docs/next/admin/actions/#forgejo-runner
