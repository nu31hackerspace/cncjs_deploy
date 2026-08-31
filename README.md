# CNCjs Deploy for NU31 Infra

This repository deploys [`cncjs`](https://github.com/cncjs/cncjs), a web interface for
CNC controllers running Grbl, Marlin, Smoothieware, TinyG or g2core, to the existing
[`infra`](https://github.com/nu31hackerspace/infra) Docker Swarm.

## Notes

- `update_config.order` is `stop-first`, two instances would fight over the `.cncrc`
  file and, with a controller attached, over the serial port.
- The app publishes no port of its own, the only way in is caddy.
- The stack uses no environment variables, so the workflow deploys without an env file.
