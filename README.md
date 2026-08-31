# CNCjs Deploy for NU31 Infra

This repository deploys [`cncjs`](https://github.com/cncjs/cncjs), a web interface for
CNC controllers running Grbl, Marlin, Smoothieware, TinyG or g2core, to the existing
[`infra`](https://github.com/nu31hackerspace/infra) Docker Swarm.

It follows the same deployment model as `gancio-deploy` and `matterbridge-deploy`:

- one Swarm stack file
- one GitHub Actions workflow
- deployment over SSH
- no local image build, because cncjs already publishes `cncjs/cncjs` to Docker Hub

## Image version

The tag is written out in `docker-stack.yml`:

```yaml
image: cncjs/cncjs:v1.11.3
```

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
