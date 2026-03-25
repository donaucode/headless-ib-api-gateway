# IBKR API Gateway Docker

Minimal Docker setup for running Interactive Brokers Gateway on a server or VPS.

This repository does one thing:

- starts a single IB Gateway container
- exposes paper and/or live IB API socket ports on the host
- optionally exposes VNC for GUI access
- persists IB Gateway settings in a Docker volume

It is intentionally small. This repo is not your trading app, not a frontend, and not a strategy engine. It is only the gateway runtime.

## Restart and rebuild 

```
From the gateway repo directory:

  cd /home/forstner/fackatrader/headless-ib-api-gateway
  docker compose down
  docker compose up -d --force-recreate

If you also changed the pinned image and want Docker to fetch it first:

  cd /home/forstner/fackatrader/headless-ib-api-gateway
  docker compose down
  docker compose pull
  docker compose up -d --force-recreate
```


## What This Gives You

The container runs IB Gateway in Docker and makes it reachable from other processes on the same machine.

Typical setup:

```text
Your trading app -> 127.0.0.1:4002 or 127.0.0.1:4001 -> IB Gateway container -> IBKR
```

Use:

- `4002` for paper trading
- `4001` for live trading
- `5900` for VNC access to the IB Gateway GUI

By default the ports are bound to `127.0.0.1`, so they are reachable from the host itself but not from the public network.

## How It Works

This repo uses the community-maintained `gnzsnz/ib-gateway-docker` image, pinned to an immutable digest in [docker-compose.yml](/home/forstner/fackatrader/headless-ib-api-gateway/docker-compose.yml).

The compose file starts one service:

- `ib-gateway`

The container:

- logs in to IB Gateway with the credentials from `.env`
- starts in `paper`, `live`, or `both` mode
- exposes IB Gateway's internal ports through Docker port mappings
- stores settings in the named volume `ib-gateway-settings`

The image is pinned so upgrades do not happen implicitly when upstream changes `stable`.

Upstream project:

- https://github.com/gnzsnz/ib-gateway-docker

## Repository Files

- [docker-compose.yml](/home/forstner/fackatrader/headless-ib-api-gateway/docker-compose.yml): container definition, port mappings, volume
- [.env.example](/home/forstner/fackatrader/headless-ib-api-gateway/.env.example): example runtime configuration
- [.env](/home/forstner/fackatrader/headless-ib-api-gateway/.env): your local runtime configuration, not committed

## Requirements

- Docker
- Docker Compose
- valid IBKR credentials

If you want to use both paper and live at the same time, you need separate paper and live credentials.

## Setup

1. Create your local env file:

```bash
cp .env.example .env
```

2. Edit `.env` and set your credentials and preferred mode.

Credential rules in this repo:

- `TRADING_MODE=paper`: set `TWS_USERID_PAPER` and `TWS_PASSWORD_PAPER`
- `TRADING_MODE=live`: set `TWS_USERID` and `TWS_PASSWORD`
- `TRADING_MODE=both`: set all four

Important:

- upstream `gnzsnz/ib-gateway-docker` expects `TWS_USERID` / `TWS_PASSWORD` for normal startup
- this repo adds a small compatibility layer in `docker-compose.yml` so `paper` mode reliably uses `TWS_USERID_PAPER` / `TWS_PASSWORD_PAPER`
- this avoids the common failure mode where the GUI opens but the login fields are effectively blank and the API socket on `4002` never comes up

3. Start the gateway:

```bash
docker compose up -d
```

4. Check container status:

```bash
docker compose ps
```

5. Follow logs when needed:

```bash
docker compose logs -f ib-gateway
```

6. Wait for a successful login before connecting your app.

Healthy login signals in the logs:

- `Click button: Paper Log In` or `Click button: Log In`
- `Login has completed`

If you need to verify the API is actually up, check that your client can connect to:

- `127.0.0.1:4002` for paper
- `127.0.0.1:4001` for live

## Trading Modes

`TRADING_MODE` controls which IB session(s) the container starts:

- `paper`: use your paper credentials and paper session; live credentials are optional
- `live`: use your live credentials and live session; paper credentials are optional
- `both`: start both sessions in parallel

This repo's exact behavior:

- in `paper` mode, the startup wrapper validates `TWS_USERID_PAPER` / `TWS_PASSWORD_PAPER` and maps them into the upstream image's expected login fields before launch
- in `live` mode, it uses `TWS_USERID` / `TWS_PASSWORD`
- in `both` mode, it requires both credential sets and leaves the upstream dual-mode behavior intact

When `TRADING_MODE=both`, you must provide:

- `TWS_USERID`
- `TWS_PASSWORD`
- `TWS_USERID_PAPER`
- `TWS_PASSWORD_PAPER`

Important:

- `TRADING_MODE=both` does not mean the gateway automatically switches for you
- it means both sessions are available
- your trading app decides which one to use by connecting to the appropriate port

In practice:

- connect to `127.0.0.1:4002` for paper
- connect to `127.0.0.1:4001` for live

## Environment Variables

The main settings live in [.env.example](/home/forstner/fackatrader/headless-ib-api-gateway/.env.example).

Important variables:

- `TRADING_MODE`: `paper`, `live`, or `both`
- `READ_ONLY_API`: `yes` prevents order placement through the API
- `TWS_USERID` / `TWS_PASSWORD`: live credentials
- `TWS_USERID_PAPER` / `TWS_PASSWORD_PAPER`: paper credentials
- `TWS_ACCEPT_INCOMING`: usually `accept`
- `TWOFA_TIMEOUT_ACTION`: usually `restart`
- `RELOGIN_AFTER_TWOFA_TIMEOUT`: usually `yes`
- `VNC_SERVER_PASSWORD`: set this if you want VNC access
- `TIME_ZONE`: container time zone
- `AUTO_RESTART_TIME`: optional daily restart time
- `IB_GATEWAY_BIND_ADDRESS`: keep `127.0.0.1` unless you know exactly why you want wider exposure
- `IB_GATEWAY_LIVE_HOST_PORT`: host port for live, default `4001`
- `IB_GATEWAY_PAPER_HOST_PORT`: host port for paper, default `4002`
- `IB_GATEWAY_VNC_HOST_PORT`: host port for VNC, default `5900`

## Ports

The compose file maps host ports to the ports used inside the container:

- host `4001` -> container `4003` for live
- host `4002` -> container `4004` for paper
- host `5900` -> container `5900` for VNC

The internal ports `4003` and `4004` come from the upstream image. Your own app should connect to the host ports, not the internal container ports.

Note:

- seeing Docker publish `4002` does not guarantee that IB Gateway finished login
- the socket becomes usable only after the GUI login completes and IB Gateway starts listening internally
- if your client gets disconnected immediately, check the container logs first

## Connecting From Your Trading App

Your trading app should run separately from this repo and connect to the gateway over TCP.

Typical mapping:

- paper environment -> `127.0.0.1:4002`
- live environment -> `127.0.0.1:4001`

This repo does not decide whether your strategy is in paper or live mode. Your trading app owns that decision.

Recommended separation:

- gateway repo: credentials, session runtime, port exposure
- trading app: client IDs, risk limits, live-trading kill switch, strategy selection

If you have a browser frontend, the frontend should talk to your own backend, and the backend should talk to IB Gateway.

## VNC Access

VNC is only for maintenance and troubleshooting.

Use cases:

- checking whether IB Gateway is logged in
- handling login issues or prompts
- verifying the GUI state during setup

If you do not need VNC, leave it disabled operationally or avoid exposing the port where possible.

## Persistence

The named Docker volume `ib-gateway-settings` stores IB Gateway settings across container restarts.

That means:

- restarting or recreating the container does not wipe settings
- `docker compose down` keeps the volume
- `docker compose down -v` removes the container and the persisted settings volume

## Common Commands

Start:

```bash
docker compose up -d
```

View status:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f ib-gateway
```

Stop:

```bash
docker compose down
```

Stop and remove persisted settings:

```bash
docker compose down -v
```

Pull the pinned image and recreate:

```bash
docker compose pull
docker compose up -d
```

## Security Notes

IB Gateway exposes a raw TCP API. It is not something you should place openly on the public internet.

Keep these defaults unless you have a deliberate security design:

- `IB_GATEWAY_BIND_ADDRESS=127.0.0.1`
- `READ_ONLY_API=yes` while testing

If you ever need remote access, put your own security boundary in front of it, for example:

- SSH tunnel
- VPN
- tightly controlled private network

Do not expose the IB API socket directly to the public internet.

## Upgrading

The image in [docker-compose.yml](/home/forstner/fackatrader/headless-ib-api-gateway/docker-compose.yml) is pinned by digest for stability.

To upgrade:

1. choose a newer tested upstream image version or digest
2. update the `image:` line in [docker-compose.yml](/home/forstner/fackatrader/headless-ib-api-gateway/docker-compose.yml)
3. run `docker compose pull`
4. run `docker compose up -d`

This keeps upgrades explicit and predictable.
