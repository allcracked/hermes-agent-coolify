# hermes-agent-coolify

Minimal Coolify deployment of [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
using the prebuilt Docker Hub image (`nousresearch/hermes-agent:latest`). No build on the server.

## One-time setup

Before deploying, make sure you have:

- **Swap on the VM** (2 GB VM — do this once over SSH):
  ```bash
  sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && \
    sudo mkswap /swapfile && sudo swapon /swapfile && \
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```
- **Telegram bot** created via `@BotFather` (`/newbot`). If the bot ever had a webhook, clear it first:
  ```bash
  curl "https://api.telegram.org/bot<TOKEN>/deleteWebhook"
  ```
- **Your Telegram user ID** — message `@userinfobot` on Telegram.
- **At least one LLM key** — OpenRouter, Anthropic, or OpenAI.
- **Coolify GitHub App** installed on this private repo (Coolify → *Sources* → *GitHub Apps*).

## Coolify wiring

1. **+ New Resource** → **Docker Compose** → select this repo, branch `main`.
2. Build Pack: **Docker Compose**. Compose file path: `docker-compose.yaml`.
3. **Environment Variables** tab → paste and fill in the values from `.env.example`.
4. **Webhooks** → enable auto-deploy on push to `main`.
5. **Deploy.** Coolify pulls the image and starts the `gateway` service. First pull takes 1–3 min.

## Use it

Message your Telegram bot. A reply confirms the gateway is live.

## Day-to-day operations

| Task | How |
|---|---|
| View logs | Coolify *Logs* tab, or `docker logs -f $(docker ps -qf name=gateway)` |
| Shell into the container | `docker exec -it $(docker ps -qf name=gateway) bash` (`hermes` CLI is on PATH) |
| Update to latest upstream | Push a redeploy in Coolify — `pull_policy: always` re-pulls `:latest` |
| Pin a version | Change `image: nousresearch/hermes-agent:latest` to a specific tag and commit |
| Back up memories/skills | Coolify *Storages* → `hermes-data` volume → enable S3 backups |
| Rollback | Coolify *Deployments* → previous successful run → *Redeploy* |

## Troubleshooting

**Bot doesn't reply (409 Conflict in logs)**
A webhook is still registered. Fix:
```bash
curl "https://api.telegram.org/bot<TOKEN>/deleteWebhook"
```

**Container OOM-restart-looping**
Hermes spiked past 750 MB. Raise the limit in `docker-compose.yaml` (`memory: 1G`) and confirm swap is on (`swapon --show`).

**Coolify can't clone the repo**
The GitHub App isn't installed on this repo, or repository access was revoked. Check *GitHub App Settings* on github.com.

## Adding the dashboard later

If you upgrade to a larger plan, add this service to `docker-compose.yaml` and access it
via SSH tunnel (`ssh -L 9119:localhost:9119 user@your-server`):

```yaml
dashboard:
  image: nousresearch/hermes-agent:latest
  pull_policy: always
  restart: unless-stopped
  volumes:
    - hermes-data:/opt/data
  environment:
    - HERMES_UID=10000
    - HERMES_GID=10000
    - TZ=America/Tegucigalpa
  ports:
    - "127.0.0.1:9119:9119"
  deploy:
    resources:
      limits:
        memory: 300M
  command: ["dashboard", "--host", "0.0.0.0", "--no-open"]
```

Then from your laptop: `ssh -L 9119:localhost:9119 user@your-server` and open http://localhost:9119.
