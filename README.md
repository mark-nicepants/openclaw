x# openclaw-deploy

Dockerized [OpenClaw](https://docs.openclaw.ai/install/docker) gateway running 24/7 on **shady** (`49.13.138.137`), reachable at `https://claw.mooibroek.dev/` through Nginx Proxy Manager. Updates are pushed via GitHub Actions on `main`.

## Topology

- Host: `shady` (`mark@49.13.138.137`)
- Compose dir: `/var/docker_projects/openclaw/`
- Image: `ghcr.io/openclaw/openclaw:latest` (pre-built)
- Sandbox: **enabled** (`OPENCLAW_SANDBOX=1`, host `docker.sock` mounted)
- Reverse proxy: NPM on the same host, forwarding `claw.mooibroek.dev → openclaw-gateway:18789`
- Public access: restricted at NPM to Tailscale IP + home WAN IP
- Backups: handled by Kopia on the VPS (point it at `/var/docker_projects/openclaw/data/` and the `openclaw_home` Docker volume)

## One-time VPS bootstrap

Do this once on shady before the workflow can deploy.

### 1. Fix SSH access if your home IP changed

The current `~/.ssh/authorized_keys` or any `sshd` / firewall rule on shady may still reference your old WAN IP. From a machine that can still reach shady (or via your VPS provider's web console), update:

```bash
# Hetzner Cloud Firewall (if used): web console → Firewalls → edit SSH rule
# Or host-level (ufw/iptables):
sudo ufw status numbered
sudo ufw allow from <new-home-ip> to any port 22 proto tcp
sudo ufw delete <old-rule-number>
```

### 2. Prepare host directory and clone

```bash
ssh mark@49.13.138.137
sudo mkdir -p /var/docker_projects/openclaw
sudo chown $USER:$USER /var/docker_projects/openclaw
cd /var/docker_projects/openclaw
git clone git@github.com:<YOUR_GH_USER>/openclaw-deploy.git .
```

### 3. Create `.env`

```bash
cp .env.example .env
# Generate the gateway token
sed -i "s|^OPENCLAW_GATEWAY_TOKEN=.*|OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)|" .env

# Find the NPM Docker network name and put it in .env
docker network ls
$EDITOR .env   # set PROXY_NETWORK=<name>
```

### 4. Prepare bind-mount ownership

The container runs as uid 1000 (`node`). Host directories must match.

```bash
mkdir -p data/config data/workspace data/secrets
sudo chown -R 1000:1000 data
```

### 5. First start

```bash
docker compose pull
docker compose up -d
docker compose logs -f openclaw-gateway   # watch for "ready" / healthy
```

### 6. Wire up Nginx Proxy Manager

In the NPM UI:

- **Proxy Host**: `claw.mooibroek.dev`
- **Forward hostname / IP**: `openclaw-gateway` (container DNS name, since NPM is on the same Docker network)
- **Forward port**: `18789`
- **Websockets support**: **on** (Control UI uses WS)
- **Block Common Exploits**: on
- **SSL**: request a Let's Encrypt cert, force SSL, HTTP/2

Access list:

- Create an NPM **Access List** with your Tailscale IP and home WAN IP.
- Attach it to the proxy host.

### 7. Configure the Kimi 2.6 thinking model

OpenClaw treats models as authenticated profiles. Easiest is the Control UI:

1. Open `https://claw.mooibroek.dev/`, paste `OPENCLAW_GATEWAY_TOKEN` from `.env` into Settings.
2. Add a provider → custom OpenAI-compatible endpoint:
   - Provider: Moonshot (or OpenRouter, whichever you have an API key for)
   - Base URL: `https://api.moonshot.ai/v1` (Moonshot direct) **or** `https://openrouter.ai/api/v1` (OpenRouter)
   - Model id: the exact slug for Kimi 2.6 thinking on the provider you chose
   - API key: paste yours
3. Set it as the default agent model.

If you prefer the CLI:

```bash
docker compose run --rm openclaw-cli models auth login --provider <provider> --method api-key --set-default
```

## GitHub Actions secrets

Set these in the repo settings → Secrets and variables → Actions:

| Secret           | Value                                             |
| ---------------- | ------------------------------------------------- |
| `DEPLOY_HOST`    | `49.13.138.137` (or `shady.mooibroek.dev` if DNS) |
| `DEPLOY_USER`    | `mark`                                            |
| `DEPLOY_SSH_KEY` | Private key for a dedicated deploy keypair        |
| `DEPLOY_PORT`    | (optional) custom SSH port                        |

Generate a dedicated deploy key:

```bash
ssh-keygen -t ed25519 -C "openclaw-deploy@github" -f ~/.ssh/openclaw_deploy -N ""
ssh-copy-id -i ~/.ssh/openclaw_deploy.pub mark@49.13.138.137
# Paste the contents of ~/.ssh/openclaw_deploy into DEPLOY_SSH_KEY (private key, full file)
```

Then test from GitHub: Actions → "Deploy OpenClaw to shady" → Run workflow.

## Day-to-day

- **Update**: push to `main` (changes touching `docker-compose.yml` or the workflow auto-deploy). For a forced redeploy with the latest image, run the workflow manually.
- **Tail logs**: `docker compose logs -f openclaw-gateway`
- **CLI**: `docker compose run --rm openclaw-cli <command>` — e.g. `dashboard --no-open`, `devices list`, `models list`
- **Health**: `curl -fsS http://127.0.0.1:18789/healthz` (from inside shady) or check NPM proxy host status

## Sandbox setup (follow-up)

`OPENCLAW_SANDBOX=1` is on, but for the agent to actually spawn sandbox containers OpenClaw needs the sandbox image present on the host Docker daemon. With a pre-built image install (no source checkout), follow the inline `docker build` command from [Sandboxing § Images and setup](https://docs.openclaw.ai/gateway/sandboxing#images-and-setup) on shady. Until then the gateway runs fine but agents that try to use sandboxed shells will fail.

## Adding integrations later

These are deliberately not configured in v1. Add them after the gateway is up:

- **GitHub PRs**: add a fine-grained PAT or dedicated GitHub App via the Control UI's plugin/auth config.
- **Email**: add Gmail/IMAP through a messaging channel or skill plugin (`docker compose run --rm openclaw-cli plugins install …`).
- **Channels** (WhatsApp/Telegram/Discord/Synology): `docker compose run --rm openclaw-cli channels …` per the [docs](https://docs.openclaw.ai/install/docker#configure-channels-optional).

## Troubleshooting

- **`PROXY_NETWORK` empty / network not found**: set it in `.env` to the NPM Docker network name (`docker network ls`).
- **Permission errors on `/home/node/.openclaw`**: `sudo chown -R 1000:1000 /var/docker_projects/openclaw/data`.
- **Control UI 502 from NPM**: confirm NPM container is on the same network as `openclaw-gateway` and the forward host is the container name, not `127.0.0.1`.
- **Unhealthy container**: `docker compose logs --tail=200 openclaw-gateway`.
- **`Missing config. Run openclaw setup or set gateway.mode=local`**: `OPENCLAW_SKIP_ONBOARDING=1` skips the wizard but never writes `gateway.mode`, and there is no env var for it. The gateway refuses to start without it. Write the config once (persists in `./data/config`); the deploy workflow does this automatically, or run it by hand on shady:

  ```bash
  cd /var/docker_projects/openclaw
  docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
    dist/index.js config set --batch-json \
    '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
  docker compose up -d
  ```
- **Control UI: "Browser origin not allowed"**: the gateway only allows `localhost` origins by default and rejects the browser served via `https://claw.mooibroek.dev`. The deploy workflow writes the domain into `gateway.controlUi.allowedOrigins`; to add another origin by hand:

  ```bash
  docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
    dist/index.js config set --batch-json \
    '[{"path":"gateway.controlUi.allowedOrigins","value":["https://claw.mooibroek.dev","http://localhost:18789","http://127.0.0.1:18789"]}]'
  docker compose up -d openclaw-gateway   # restart to apply
  ```
- **Control UI: "pairing required: device is not approved"**: each new browser must be approved once. List and approve the pending request:

  ```bash
  docker compose run --rm --entrypoint node openclaw-cli dist/index.js devices list
  docker compose run --rm --entrypoint node openclaw-cli dist/index.js devices approve <requestId>
  ```
- **NPM 525 / "Internal Server Error" over HTTPS**: Cloudflare (Full/strict) couldn't TLS-handshake the origin because the NPM proxy host had no SSL. Request a Let's Encrypt cert on the `claw` proxy host (or install a Cloudflare Origin cert) and enable Force SSL.
- **Control UI loads but won't connect (WebSocket)**: enable **Websockets Support** on the `claw` proxy host in NPM — without it nginx drops the `Upgrade` headers and `wss://` fails.
