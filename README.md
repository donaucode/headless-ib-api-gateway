# IBKR API Gateway Docker

Minimal repository for running Interactive Brokers Gateway in Docker for headless environments.

What remains in this repo:
- `docker-compose.yml`
- `.env`
- `.env.example`
- this `README.md`

The gateway container is based on the `gnzsnz/ib-gateway-docker` image, pinned to an immutable digest in [docker-compose.yml](/home/forstner/ft-tui/docker-compose.yml):
- upstream image docs: https://github.com/gnzsnz/ib-gateway-docker
- `@stoqey/ib` client docs: https://github.com/stoqey/ib

## Purpose

This repo is now only a reusable IBKR Gateway runtime. It does not include:
- Nuxt
- Supabase
- any frontend
- any custom backend proxy

It exposes the native IB Gateway socket ports so another service can connect to them.

## Important Integration Constraint

IB Gateway speaks the native IB socket protocol over raw TCP.

That means:
- a browser frontend cannot connect to it directly
- a server, worker, Electron main process, or API/BFF layer can connect to it
- your frontend should talk to your own backend, and that backend should talk to IB Gateway

This matches `@stoqey/ib`, which uses Node's socket APIs rather than browser APIs.

## Files

`docker-compose.yml`
- starts one `ib-gateway` container
- exposes live, paper, and optional VNC ports
- persists gateway settings in the named Docker volume `ib-gateway-settings`

`.env` / `.env.example`
- contain the runtime configuration for credentials, ports, and safety settings

## Configuration

1. Copy the example file if you do not already have a populated `.env`.

```bash
cp .env.example .env
```

2. Set the required credentials.

Required:
- `TWS_USERID`
- `TWS_PASSWORD`

Required when `TRADING_MODE=both`:
- `TWS_USERID_PAPER`
- `TWS_PASSWORD_PAPER`

Key operational variables:
- `TRADING_MODE`: `paper`, `live`, or `both`
- `READ_ONLY_API`: keep `yes` unless you intentionally want trading access
- `TWS_ACCEPT_INCOMING`: usually `accept`
- `TWOFA_TIMEOUT_ACTION`: usually `restart`
- `RELOGIN_AFTER_TWOFA_TIMEOUT`: usually `yes`
- `TIME_ZONE`: container time zone
- `AUTO_RESTART_TIME`: optional daily restart handled by IBC

Host exposure variables:
- `IB_GATEWAY_BIND_ADDRESS`: keep `127.0.0.1` by default
- `IB_GATEWAY_LIVE_HOST_PORT`: default `4001`
- `IB_GATEWAY_PAPER_HOST_PORT`: default `4002`
- `IB_GATEWAY_VNC_HOST_PORT`: default `5900`

## Run

Start the gateway:

```bash
docker compose up -d
```

Follow logs:

```bash
docker compose logs -f ib-gateway
```

Stop it:

```bash
docker compose down
```

Remove it together with the persisted gateway settings volume:

```bash
docker compose down -v
```

## Ports

By default the host exposes:
- live socket API on `127.0.0.1:4001`
- paper socket API on `127.0.0.1:4002`
- VNC on `127.0.0.1:5900` when `VNC_SERVER_PASSWORD` is set

Inside the container the image publishes:
- live on `4003`
- paper on `4004`

This mirrors the upstream image design, which relays the internal IB ports to externally reachable container ports.

## Using It From Your Own App

Use a backend service on the same host or network boundary and connect it to the exposed socket port.

Typical mapping:
- paper mode: connect your client to `127.0.0.1:${IB_GATEWAY_PAPER_HOST_PORT}`
- live mode: connect your client to `127.0.0.1:${IB_GATEWAY_LIVE_HOST_PORT}`

Example with `@stoqey/ib` in Node.js:

```ts
import { IBApi } from "@stoqey/ib";

const ib = new IBApi({
  host: "127.0.0.1",
  port: 4002,
  clientId: 123,
});

ib.connect();
```

If your frontend is a browser app, place a backend between the browser and IB Gateway:

```text
Browser UI -> your API/BFF -> IB Gateway socket
```

## Security

The IB API socket is not designed to be exposed openly on the network.

Default recommendation:
- keep `IB_GATEWAY_BIND_ADDRESS=127.0.0.1`
- keep `READ_ONLY_API=yes` unless order placement is intentional
- only expose the ports remotely behind your own SSH tunnel, VPN, reverse proxy, or other access control layer

## Notes

- `TRADING_MODE=both` is useful when one environment needs both live and paper sessions.
- The Docker volume `ib-gateway-settings` keeps gateway settings across container restarts.
- The image is pinned to the digest that was behind upstream `stable` when this repo was updated, corresponding to upstream image version `10.37`.
- To upgrade later, inspect the upstream package page for a newer tested digest and update the `image:` line in [docker-compose.yml](/home/forstner/ft-tui/docker-compose.yml).
