# docker-edge

The shared HTTPS front door for every Docker project on this Mac. One Traefik container
terminates HTTPS and routes each host name to the right project's container. Valet keeps serving
the projects that haven't moved yet, on `127.0.0.1`.

```
browser ── https://acme.tenancey.test ──► 127.0.0.2:443
            (dnsmasq: tenancey.test → 127.0.0.2)
                     │  socat, root launch daemon (mac/)
                     ▼
            127.0.0.2:18443 ──► traefik (this project)
                     │  Docker labels: Host(...) → container, on the `edge` network
                     ▼
            tenancey's nginx / reverb / minio / mailpit
```

## Why it's built this way

- **A second loopback address, `127.0.0.2`.** Valet's nginx listens on `127.0.0.1:80/443` only, so
  `127.0.0.2:80/443` is free for Docker, and URLs need no port.
- **socat in front of Traefik.** Docker Desktop 4.89+ can't publish ports below 1024 on one
  address (it falls back to `0.0.0.0`, which clashes with Valet). A root socat holds
  `127.0.0.2:80/443` and passes the bytes to Traefik's `18080/18443`. macOS's pf can't redirect the
  Mac's own loopback traffic, so it isn't used.
- **Ports `18080`/`18443`/`18090`** avoid `8080` (another project's Reverb listens on `*:8080`).

## Layout

```
docker-edge/
├── compose.yaml        # traefik + the shared `edge` network
├── dynamic/tls.yaml    # certificate list (watched, no restart needed)
├── certs/              # mkcert output, git-ignored
└── mac/                # launchd daemons installed in /Library/LaunchDaemons
    ├── local.loopback-alias.plist     # adds 127.0.0.2 at boot
    ├── local.edge-forward-80.plist    # 127.0.0.2:80  → 127.0.0.2:18080
    └── local.edge-forward-443.plist   # 127.0.0.2:443 → 127.0.0.2:18443
```

| Address | What |
|---|---|
| `127.0.0.2:80` | socat → Traefik `web`: redirects to https |
| `127.0.0.2:443` | socat → Traefik `websecure`: TLS, routing |
| `http://127.0.0.2:18090` | Traefik dashboard (read-only) |

## First-time setup

```bash
# 1. The address (and at every boot)
sudo install -o root -g wheel -m 644 mac/local.loopback-alias.plist /Library/LaunchDaemons/
sudo launchctl bootstrap system /Library/LaunchDaemons/local.loopback-alias.plist

# 2. The forwarders (needs: brew install socat)
for p in 80 443; do
  sudo install -o root -g wheel -m 644 mac/local.edge-forward-$p.plist /Library/LaunchDaemons/
  sudo launchctl bootstrap system /Library/LaunchDaemons/local.edge-forward-$p.plist
done

# 3. The local certificate authority (once)
brew install mkcert
mkcert -install

# 4. Traefik
docker compose up -d
```

## Adding a project

1. **Certificate:**
   ```bash
   mkcert -cert-file certs/<project>.test.pem -key-file certs/<project>.test-key.pem <project>.test "*.<project>.test"
   ```
   Add it to `dynamic/tls.yaml` (Traefik notices the change by itself).
2. **The project's `compose.yaml`:**
   ```yaml
   services:
     nginx:
       networks: [default, edge]
       labels:
         traefik.enable: "true"
         traefik.docker.network: edge
         traefik.http.routers.<project>.rule: Host(`<project>.test`)
         traefik.http.routers.<project>.entrypoints: websecure
         traefik.http.services.<project>.loadbalancer.server.port: "80"
   networks:
     edge:
       external: true
   ```
   - **Prefix router and service names with the project name.** Traefik's names are global
     across all projects.
   - **Remove the project's own published web ports.** The browser goes through Traefik.
   - **A wildcard rule** (`HostRegexp(...)`) is long, and Traefik ranks routers by rule length.
     Give it `priority: "1"` and named hosts a higher priority, or the wildcard takes them.
   - **`$` is special in Compose:** write `$$` in labels, for example
     `HostRegexp(`^[a-z0-9-]+\.<project>\.test$$`)`. `docker compose config` prints `$$` back, so check the real
     value on the container: `docker inspect <container> --format '{{json .Config.Labels}}'`.
3. **DNS:** a file in `~/.config/valet/dnsmasq.d/<project>.conf`:
   ```
   address=/<project>.test/127.0.0.2
   ```
   then `sudo brew services restart dnsmasq`. It covers `<project>.test` and every subdomain. Check
   with `dig +short <project>.test @127.0.0.1` → `127.0.0.2`.
4. **Valet:** `valet unlink <project>`.

## Rules

- **Start docker-edge before any project.** Otherwise a project's `up` fails with "network edge
  declared as external, but could not be found".
- **Only browser-facing containers join `edge`** (nginx, a WebSocket server, a file server's
  public port, a mail inbox). Never PHP, database or cache containers. Every container on `edge`
  can find every other one by service name, so a second `app` or `mysql` on `edge` could answer
  another project's lookups.
- **The Docker socket** is mounted read-only, but it still gives Traefik read access to Docker's
  API. That's fine on this Mac. A production server would put a socket proxy in between.

## Daily commands

```bash
docker compose ps
docker compose logs -f traefik      # access log: every request and where it went
open http://127.0.0.2:18090         # dashboard: routers, services, certificates
```

## Troubleshooting

| Symptom | Check |
|---|---|
| `curl: (52) Empty reply` on `127.0.0.2` | Traefik isn't running: `docker compose up -d` |
| Nothing listens on `127.0.0.2:80/443` | `sudo launchctl print system/local.edge-forward-443`, `/var/log/local.edge-forward-443.log`, and `ifconfig lo0` (is `127.0.0.2` there?) |
| Traefik's plain `404 page not found` | No router for that host: look in the dashboard. Is the container labelled, running and on `edge`? |
| Browser certificate warning | `mkcert -install` again. Is the certificate in `dynamic/tls.yaml`, and does its name cover the host? (A wildcard covers one level only.) |
| A changed `tls.yaml` isn't picked up | `docker compose restart traefik` |
| The name goes to Valet | `dig +short <name> @127.0.0.1` and the dnsmasq file. Flush the cache: `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |

## Undo everything

```bash
docker compose down && docker network rm edge
for d in edge-forward-80 edge-forward-443 loopback-alias; do
  sudo launchctl bootout system/local.$d
  sudo rm /Library/LaunchDaemons/local.$d.plist
done
sudo ifconfig lo0 -alias 127.0.0.2
rm ~/.config/valet/dnsmasq.d/<project>.conf && sudo brew services restart dnsmasq
valet link <project>   # in the project folder, to give it back to Valet
```
