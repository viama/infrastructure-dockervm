# Docker Infrastructure Config

Git repository for Docker Compose stacks running on this host.

## Repo structure

```
/var/data/config/
├── .gitignore
├── README.md
├── homeassistant/
│   ├── homeassistant.yml       # Home Assistant + Cloudflare Tunnel
│   ├── .env.example            # → copy to .env and fill in secrets
│   ├── grafana.env.example     # → copy to grafana.env
│   └── grafana.env             # (gitignored)
├── komodo/
│   ├── komodo.yml              # Komodo Periphery agent
│   ├── komodo.env.example      # → copy to komodo.env and fill in secrets
│   └── komodo.env              # (gitignored)
├── matter-server/
│   └── matter-server.yml       # python-matter-server
├── matterbridge/
│   └── matterbridge.yml        # Matterbridge Matter bridge
├── mqtt/
│   └── mqtt.yml                # Eclipse Mosquitto MQTT broker
├── openwakeword/
│   ├── config.yaml             # (gitignored — contains MQTT credentials)
│   └── config.yaml.example     # → copy to config.yaml and fill in secrets
├── traefik/
│   ├── traefik_config.yml      # Traefik reverse proxy
│   ├── traefik.toml            # Traefik static config
│   ├── .env.example            # → copy to .env and fill in secrets
│   ├── .env                    # (gitignored)
│   ├── acme.json               # (gitignored — TLS cert store)
│   └── traefik.log             # (gitignored — runtime log)
└── zigbee2mqtt/
    └── zigbee2mqtt.yml         # Zigbee2MQTT bridge
```

## Secrets management

Each stack that requires secrets follows this convention:

- **`.env`** — real values, **never committed** (gitignored by `*.env` / `.env` rules)
- **`.env.example`** — placeholder template, committed to the repo
- For non-compose config files with secrets (e.g. `openwakeword/config.yaml`): the real file is gitignored and a `*.example` template is committed instead

To set up on a new host:

```bash
cp homeassistant/.env.example homeassistant/.env
# edit homeassistant/.env with real values
cp traefik/.env.example traefik/.env
# edit traefik/.env with real values
cp komodo/komodo.env.example komodo/komodo.env
# edit komodo/komodo.env with real values
cp openwakeword/config.yaml.example openwakeword/config.yaml
# edit openwakeword/config.yaml with real credentials
```

Docker Compose auto-loads `.env` from the compose file's directory — no `env_file:` directive needed for `${VAR}` substitution in the YAML.

For stacks using a named env file (e.g. komodo):
```bash
docker compose --env-file komodo/komodo.env -f komodo/komodo.yml up -d
```

## Services and their secrets

| Stack | Secret file | Secrets |
|-------|-------------|---------|
| homeassistant | `homeassistant/.env` | `TUNNEL_TOKEN` — Cloudflare Tunnel token |
| traefik | `traefik/.env` | `CF_DNS_API_TOKEN` — DNS challenge for Let's Encrypt wildcard certs |
| komodo | `komodo/komodo.env` | `KOMODO_DB_PASSWORD`, `KOMODO_PASSKEY`, `KOMODO_WEBHOOK_SECRET`, `KOMODO_JWT_SECRET` |
| openwakeword | `openwakeword/config.yaml` | MQTT broker password |

## Services overview

### Home automation
| Stack | Container | Image | Purpose |
|-------|-----------|-------|---------|
| homeassistant | `homeassistant` | `homeassistant/home-assistant` | Home Assistant core |
| homeassistant | `hass_cloudflare_tunnel` | `cloudflare/cloudflared` | Cloudflare Tunnel (remote access) |
| zigbee2mqtt | `zigbee2mqtt` | `koenkk/zigbee2mqtt` | Zigbee coordinator bridge |
| mqtt | — | `eclipse-mosquitto` | MQTT broker |
| matter-server | `matter-server` | `python-matter-server:stable` | Matter/Thread controller |
| matterbridge | `matterbridge` | `luligu/matterbridge` | Matter bridge for legacy devices |
| openwakeword | — | `rhasspy/wyoming-openwakeword` | Wake word detection (currently disabled) |

### Infrastructure
| Stack | Container | Image | Purpose |
|-------|-----------|-------|---------|
| traefik | `traefik` | `traefik:latest` | Reverse proxy + TLS (wildcard cert via Cloudflare DNS) |
| komodo | `periphery` | `komodo-periphery:latest` | Komodo agent for container management |

## Running a stack

```bash
# From repo root
docker compose -f <appname>/<appname>.yml up -d

# Example
docker compose -f homeassistant/homeassistant.yml up -d

# For komodo (named env file)
docker compose --env-file komodo/komodo.env -f komodo/komodo.yml up -d
```

## Network architecture

All containers attach to the `traefik_public` external Docker network (created separately). Traefik listens on ports 80/443 and routes by hostname using wildcard TLS cert for `*.viama.co.uk` obtained via Cloudflare DNS-01 challenge.

Home Assistant runs in `network_mode: host` for mDNS/device discovery. The Cloudflare Tunnel container provides remote HTTPS access without exposing ports externally.

## Related repositories

### Home Assistant config
The HA application config (automations, scenes, blueprints, etc.) lives separately at:

**[viama/homeassistant-data-config](https://github.com/viama/homeassistant-data-config)** → `/var/data/homeassistant/`

The compose stack that runs HA is defined here in [`homeassistant/homeassistant.yml`](homeassistant/homeassistant.yml), with secrets in `homeassistant/.env` (see [`homeassistant/.env.example`](homeassistant/.env.example)).

## Excluded app repos

Some apps manage their own git repos separately (e.g. Komodo-managed stacks). If any subdirectory gains its own `.git` folder in future, add it to `.gitignore` to prevent it being tracked as part of this infra repo.
