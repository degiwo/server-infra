# Runbook: setting up the platform

Every step has an **Intent** (what should exist), a **Command** (how it was done as of the given date), and a **Check** (how you know it is done). If a command no longer works, the intent wins – then fix the command here and update the date.

Automated steps reference the block in `cloud-init.yml` (marker `[Rn]`) instead of repeating the commands. `cloud-init.yml` is the single source for those commands. On a server without cloud-init, run the referenced blocks by hand.

---

### S1 Server with cloud-init
- **Intent:** A hardened server with Docker, firewall, users, and directories – in one go.
- **Command (2026-09):** Hetzner Console → new server, current Ubuntu LTS, contents of `cloud-init.yml` (placeholders replaced) under "Cloud config".
- **Check:** `ssh <admin>@<server>` works; `cloud-init status` reports `done`.

What cloud-init takes care of:

| Marker | Intent                                                                                    | Check                                                       |
| ------ | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| R1     | Admin with sudo, deploy user without sudo                                                 | `id deploy` shows no sudo group                             |
| R2     | SSH key-only, no root                                                                     | `ssh -o PubkeyAuthentication=no admin@<server>` is rejected |
| R3     | Security updates are automatic                                                            | `systemctl is-enabled unattended-upgrades`                  |
| R4     | Docker installed, deploy user may use Docker (equivalent to root – accepted deliberately) | `docker info` as deploy                                     |
| R5     | Container logs rotate (3 × 10 MB)                                                         | `cat /etc/docker/daemon.json`                               |
| R6     | ufw: only 22, 80, 443                                                                     | `sudo ufw status`                                           |
| R7     | `/srv/platform`, `/srv/apps`, `/srv/backups` with matching owners                         | `ls -ld /srv/*`                                             |

### S2 DNS
- **Intent:** `*.apps.<domain>` point to the server IP.
- **Command (2026-10):** Cloudflare → `<domain>` → DNS → Records: Name `*.apps` Type `A` → `<server IPv4>`, Proxy status **DNS only**
- **Check:** `dig +short anything.apps.<domain>` returns the server IP.

### S3 Platform files and secrets
- **Intent:** The contents of `platform/` live in `/srv/platform/`. Secrets are created only on the server, never in the repo.
- **Command (2026-10):**
  ```sh
  rsync -av --exclude '.env' --exclude 'authelia/users.yml' \
    platform/ admin@<server>:/srv/platform/
  ```
  Then on the server in `/srv/platform`:
    1. Copy `.env.example` → `.env` and fill in (secrets: `openssl rand -hex 64`). `chmod 600 .env`.
    2. Create the Authelia users file (Authelia does not start without it):
    ```sh
    docker compose run --rm authelia authelia crypto hash generate argon2
    cp authelia/users.yml.example authelia/users.yml  # insert user + hash
    chmod 600 authelia/users.yml
    ```
- **Check:** `docker compose -f /srv/platform/compose.yml config` runs without errors.

### S4 Start the platform
- **Intent:** Caddy accepts requests and picks up containers via labels; Authelia serves the login page at `auth.apps.<domain>`.
- **Command (2026-10):** `cd /srv/platform && docker compose up -d`
- **Check:** `docker compose ps` shows `caddy` as `running` and `authelia` as `healthy`; `https://auth.apps.<domain>` shows the login page.


### S5 End-to-end check with a throwaway container
- **Intent:** An app gets HTTPS just by its labels, and a protected is only reachable after login. One login is valid for all `*.apps.<domain>`.
- **Command (2026-10):** On the server, first unprotected:
  ```sh
  docker run -d --rm --name whoami --network platform \
    --label caddy=whoami.apps.<domain> \
    --label 'caddy.reverse_proxy={{upstreams 80}}' \
    traefik/whoami
  curl -I https://whoami.apps.<domain>  # 200, valid certificate
  docker stop whoami
  ```
  Then protected:
  ```sh
  docker run -d --rm --name whoami --network platform \
    --label caddy=whoami.apps.<domain> \
    --label caddy.import=authelia \
    --label 'caddy.reverse_proxy={{upstreams 80}}' \
    traefik/whoami
  curl -I https://whoami.apps.<domain>  # 302 to auth.apps.<domain>
  docker stop whoami
  ```
- **Check:** In the browser: open `whoami.apps.<domain>`, log in with username and password.
