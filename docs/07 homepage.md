# 07, Homepage Dashboard

Homepage is a lightweight self-hosted dashboard. It gives you one page with links to all your services, service status widgets showing live stats, bookmarks, and system info like CPU and memory usage. Config is via YAML files rather than a UI, which makes it fast and easy to version control.

Estimated time is about twenty minutes.

Assumes [02, Docker and Portainer](02-portainer.md) is done. Ideally also [05, Pi-hole](05-pihole.md) and [06, NPM](06-npm.md) so you have clean URLs to link to.

## Deploying Homepage

In Portainer, add a stack named `homepage`. Paste this compose.

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      HOMEPAGE_ALLOWED_HOSTS: "home.<YOUR_DOMAIN>,<NUC_TS_IP>"
      PUID: 1000
      PGID: 1000
    volumes:
      - homepage_config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  homepage_config:
```

Replace `<YOUR_DOMAIN>` and `<NUC_TS_IP>`. The `HOMEPAGE_ALLOWED_HOSTS` variable is important, Homepage v0.10+ rejects requests from hostnames not on this list as a security measure. Include both the subdomain you will use and the raw IP as a fallback.

The Docker socket mount (read-only) lets Homepage auto-detect containers and show their status. Port 3000 is Homepage's default.

Deploy. Confirm it started with `docker ps | grep homepage` and `docker logs homepage 2>&1 | tail -10`.

## Adding the DNS record and NPM proxy

In Pi-hole, System, Local DNS Records, add `home.<YOUR_DOMAIN>` pointing to `<NUC_TS_IP>`. In NPM, add a proxy host with domain `home.<YOUR_DOMAIN>`, scheme `http`, forward hostname `<NUC_TS_IP>`, forward port `3000`, with Websockets and Block Common Exploits enabled. Save.

## Configuring services on the dashboard

Homepage config is in YAML files in `/app/config/` inside the container. Edit them from the NUC's shell.

The main file for service tiles is `services.yaml`. Here is a starter that lists everything from this guide, without widget API keys yet, so all services show as clickable cards.

```bash
docker exec -i homepage sh -c "cat > /app/config/services.yaml" << 'EOF'
- Media & Photos:
    - Immich:
        icon: immich.png
        href: http://photos.<YOUR_DOMAIN>
        description: Photo library

- Infrastructure:
    - Portainer:
        icon: portainer.png
        href: https://portainer.<YOUR_DOMAIN>
        description: Container management

    - OpenMediaVault:
        icon: openmediavault.png
        href: http://omv.<YOUR_DOMAIN>
        description: NAS admin

    - Pi-hole:
        icon: pi-hole.png
        href: http://pi.<YOUR_DOMAIN>/admin
        description: DNS and ad blocking

    - Nginx Proxy Manager:
        icon: nginx-proxy-manager.png
        href: http://npm.<YOUR_DOMAIN>
        description: Reverse proxy
EOF
```

Note the `-i` flag on `docker exec`, not `-it`. Heredoc feeds stdin directly, and `-t` (TTY) breaks that.

Icons are auto-fetched from Homepage's icon library, which has entries for most common self-hosted apps. If an icon is missing, use `mdi-` prefix (Material Design Icons) or `si-` prefix (Simple Icons), or supply a full URL.

## Bookmarks

The bookmarks file adds a "bookmarks" section for external links.

```bash
docker exec -i homepage sh -c "cat > /app/config/bookmarks.yaml" << 'EOF'
- Developer:
    - GitHub:
        - abbr: GH
          href: https://github.com/

- Social:
    - Reddit:
        - abbr: RE
          href: https://reddit.com/
    - YouTube:
        - abbr: YT
          href: https://youtube.com/
EOF
```

## Settings and layout

The settings file controls title, theme, colors, and layout.

```bash
docker exec -i homepage sh -c "cat > /app/config/settings.yaml" << 'EOF'
title: My Homelab
theme: dark
color: slate
headerStyle: boxed
layout:
  Media & Photos:
    style: row
    columns: 4
  Infrastructure:
    style: row
    columns: 4
EOF
```

Available themes are `light` and `dark`. Colors include `slate`, `gray`, `zinc`, `neutral`, `stone`, `red`, `orange`, `amber`, `yellow`, `lime`, `green`, `emerald`, `teal`, `cyan`, `sky`, `blue`, `indigo`, `violet`, `purple`, `fuchsia`, `pink`, `rose`.

## Refreshing

Homepage auto-reloads its config when files change, so just refresh your browser at `http://home.<YOUR_DOMAIN>`. You should see two rows: Media & Photos with Immich, Infrastructure with four cards.

## Adding widgets (optional)

Widgets pull live data from services and display it on the dashboard. Each service that supports widgets needs an API key or similar credential. To add a widget, extend a service's entry in `services.yaml`.

For Immich, add under its entry:

```yaml
        widget:
          type: immich
          url: http://<NUC_TS_IP>:2283
          key: <IMMICH_API_KEY>
```

Generate the Immich API key from Immich's web UI, Account Settings, API Keys, New API Key.

For Pi-hole, add:

```yaml
        widget:
          type: pihole
          url: http://<NUC_TS_IP>:8053
          key: <PIHOLE_API_PASSWORD>
          version: 6
```

The key for Pi-hole v6 is the admin web password. For v5 you generate an API token from the UI.

For Portainer, add:

```yaml
        widget:
          type: portainer
          url: https://<NUC_TS_IP>:9443
          env: 1
          key: <PORTAINER_API_KEY>
```

Generate the Portainer key from top-right user icon, My account, Access tokens. The `env: 1` refers to the environment ID (usually 1 if you only have the local Docker), but check under Portainer's Environments page if you are not sure.

Widgets update every few seconds. If they show "API Error" red text, the key is wrong or the URL is unreachable from the Homepage container. Check `docker logs homepage 2>&1 | tail -20` for the exact error.

## What you have now

Homepage is running at `http://home.<YOUR_DOMAIN>` showing all your services as clickable cards with system stats at the top. If you added widgets, live status displays too.

Next: [08, Split DNS](08-split-dns.md), the final piece that makes all this reachable from every Tailscale device automatically.
