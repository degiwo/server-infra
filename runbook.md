# Runbook: setting up the platform

Every step has an **Intent** (what should exist), a **Command** (how it was done as of the given date), and a **Check** (how you know it is done). If a command no longer works, the intent wins – then fix the command here and update the date.

Automated steps reference the block in `cloud-init.yml` (marker `[Rn]`) instead of repeating the commands. `cloud-init.yml` is the single source for those commands. On a server without cloud-init, run the referenced blocks by hand.

---

### S1 Server with cloud-init
- **Intent:** A hardened server with Docker, firewall, users, and directories – in one go.
- **Command (2026-09):** Hetzner Console → new server, current Ubuntu LTS, contents of `cloud-init.yml` (placeholders replaced) under "Cloud config".
- **Check:** `ssh admin>@<server>` works; `cloud-init status` reports `done`.

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

### S2 Platform files onto the server
- **Intent:** The contents of `platform/` live in `/srv/platform/`. Secrets are created only on the server, never in the repo.
- **Command (2026-09):**
  ```sh
  rsync -av --exclude '.env' \
    platform/ admin@<server>:/srv/platform/
  ```
  Then on the server: copy `.env.example` → `.env` and fill in (secrets: `openssl rand -hex 64`). `chmod 600 .env`.
- **Check:** `docker compose -f /srv/platform/compose.yml config` runs without errors.

### S3 Start Caddy
- **Intent:** Caddy accepts requests.
- **Command (2026-09):** `cd /srv/platform && docker compose up -d`
- **Check:** `curl http://<server>` returns 'Caddy works!'.
