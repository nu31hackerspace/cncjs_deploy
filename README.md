# CNCjs Deploy for NU31 Infra

This repository deploys [`cncjs`](https://github.com/cncjs/cncjs), a web interface for
CNC controllers running Grbl, Marlin, Smoothieware, TinyG or g2core, to the existing
[`infra`](https://github.com/nu31hackerspace/infra) Docker Swarm.

It follows the same deployment model as `gancio-deploy` and `matterbridge-deploy`:

- one Swarm stack file
- one GitHub Actions workflow
- deployment over SSH
- no local image build, because cncjs already publishes `cncjs/cncjs` to Docker Hub

## Files

- [`docker-stack.yml`](./docker-stack.yml) Swarm stack for cncjs
- [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml) validate and deploy workflow

## Runtime details

- image: `cncjs/cncjs:v1.11.3` (hardcoded, see [Image version](#image-version))
- stack name: `cncjs`
- service: `cncjs_app`, listening on port `8000` inside the `infra_reverse-proxy` network
- persistent volume: `cncjs-data`, mounted at `/var/lib/cncjs`

cncjs keeps all of its state in a single `.cncrc` file: the session secret, user
accounts, macros, watch directory and UI settings. `$HOME` in the image is `/root`,
which also holds the node runtime and therefore cannot be replaced by a volume, so the
stack passes `--config /var/lib/cncjs/.cncrc` and mounts the volume there instead.

## Image version

The tag is written out in `docker-stack.yml`:

```yaml
image: cncjs/cncjs:v1.11.3
```

`v1.11.3` is the current cncjs release. Upstream builds the image in its own
`ci-docker-hub` workflow on every `v*` git tag and pushes `cncjs/cncjs:v<version>` and
`cncjs/cncjs:latest` for `linux/amd64` and `linux/arm64`.

`latest` is deliberately not used: with `latest` a redeploy of an unchanged stack file
can silently change the running version. To upgrade, bump the tag in `docker-stack.yml`
in a pull request and merge it, that is the whole procedure. Available tags are listed
on [Docker Hub](https://hub.docker.com/r/cncjs/cncjs/tags).

## Infra dependencies

Before deploying, [`infra`](https://github.com/nu31hackerspace/infra) must already be
running.

Required external network:

- `infra_reverse-proxy`

Required service:

- `caddy`

No database is needed, cncjs stores everything in its own volume.

## How to deploy CNCjs on infra

### 1. Deploy infra

`infra` must already be deployed.

### 2. Choose a domain

This setup expects `cnc.nu31.space`. DNS must point to the server running `infra`.

### 3. Add a Caddy route

Add this block to `infra/caddy/CaddyFile`:

```caddyfile
cnc.nu31.space {
    reverse_proxy * cncjs_app:8000
}
```

Then redeploy `infra`.

cncjs allows requests from private address ranges without authentication, and caddy
reaches the app over the `10.0.0.0/8` overlay network, so every request that arrives
through caddy is treated as local. Until a user account exists, anyone who can open the
domain can drive the machine. Either create an account right after the first deploy
(see step 6) or put basic auth in front of it in the same caddy block:

```caddyfile
cnc.nu31.space {
    basic_auth {
        nu31 <bcrypt-hash-from-"caddy hash-password">
    }
    reverse_proxy * cncjs_app:8000
}
```

### 4. Configure GitHub Variables

- `HOST`: SSH host of the machine running `infra`, for example `65.109.28.227`
- `USERNAME`: SSH user with access to Docker Swarm, usually `root`

### 5. Configure GitHub Secrets

- `ROOT_SSH_PRIVATE_KEY`: private key used by Actions to deploy

### 6. Run the deployment

Push to `main` or run the `Deploy CNCjs` workflow manually. Then open
`https://cnc.nu31.space` and create a user account under
**Settings -> User Accounts**. cncjs enforces a login as soon as one enabled account
exists.

### 7. Verify the result

- GitHub Actions job status
- `docker service ls`
- `docker service ps cncjs_app`
- `docker service logs -f cncjs_app`
- `https://cnc.nu31.space`

## Talking to a controller

cncjs connects to a controller over a **serial port** only, it has no network or telnet
transport. The web UI is useful on its own for browsing and previewing G-code, but to
actually run a machine the container needs the USB serial device of that machine.

Swarm mode has no equivalent of `docker run --device` and does not support `privileged`
services, so a serial adapter cannot be handed to a service the way it can to a plain
container. Two workable options:

- **Run cncjs next to the machine.** On the shop floor computer that is wired to the
  controller, run the same pinned image directly:

  ```sh
  docker run -d --name cncjs \
    --device /dev/ttyUSB0 \
    -p 8000:8000 \
    -v cncjs-data:/var/lib/cncjs \
    cncjs/cncjs:v1.11.3 --config /var/lib/cncjs/.cncrc
  ```

- **Join that computer to the swarm** as an extra node, label it
  (`docker node update --label-add cncjs=true <node>`), and uncomment the serial blocks
  at the bottom of `docker-stack.yml`. This pins the service to that node and bind
  mounts `/dev/ttyUSB0`. Whether the bind mount is enough depends on the device cgroup
  policy of that node, so verify with `docker service logs cncjs_app` and the port list
  in the UI before relying on it.

Device names are not stable across reboots, `/dev/serial/by-id/...` is the safer path
when the node has more than one adapter.

## Notes

- `update_config.order` is `stop-first`, two instances would fight over the `.cncrc`
  file and, with a controller attached, over the serial port.
- The app publishes no port of its own, the only way in is caddy.
- The stack uses no environment variables, so the workflow deploys without an env file.
