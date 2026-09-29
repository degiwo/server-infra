# server-infra

The **platform**: everything that exists once per server and serves all apps – entry point (Caddy), authentication and backup. Not a template, but a regular private repo.

## Contents

| Path             | Purpose                                                                  |
| ---------------- | ------------------------------------------------------------------------ |
| `runbook.md`     | Intent and check for every step, plus everything that can't be automated |
| `cloud-init.yml` | Base setup of a new server (single source for these commands)            |
| `platform/`      | Contents of `/srv/platform/` on the server: Compose, Caddyfile           |

## New server

Work through `runbook.md` from top to bottom.

## Architecture

```
Internet ──► Caddy (80/443, reads Docker labels)
```
