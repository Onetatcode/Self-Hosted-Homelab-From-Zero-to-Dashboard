# Self-Hosted Homelab, From Zero to Dashboard

A complete, tested guide for building a private homelab on a mini PC. Everything runs in Docker, reachable only over your private Tailscale mesh, with a dashboard on top.

## What you will build

A private cloud that hosts your photos, DNS with ad blocking, container management, reverse proxy, and dashboard. Reachable from anywhere via your Tailscale network. Invisible to the public internet.

## Stack

The guide walks through OpenMediaVault as the base OS, Docker with Portainer for container management, Immich for photos, Tailscale for private remote access, Pi-hole for DNS and ad blocking, Nginx Proxy Manager for clean subdomain URLs, Homepage for a dashboard tying it all together, and split DNS so your domain routes internally without ever touching the public internet.

Each service has its own doc in `docs/`, numbered in the order you should follow them.

## Why this setup

Nothing is exposed to the public internet, so there is no port forwarding, no dynamic DNS, and no attack surface. The entire stack is Docker Compose files, so you can move it, back it up, or redeploy anywhere. You get clean URLs like `photos.yourdomain.com` on your own domain without exposing anything. The base stack takes about four hours to build. Every service added after that is ten minutes.

## Hardware

Any x86-64 machine works. Minimum viable is 8 GB RAM, 128 GB SSD, and either ethernet or WiFi. Recommended is 16 GB RAM and a 500 GB SSD if you plan to store meaningful amounts of photos or files. This guide was written and tested on an Intel NUC10i5FNB with 8 GB DDR4 and a 128 GB SSD.

## Time

Base install through dashboard takes about three and a half hours if nothing breaks. Add another hour or two for the things that will inevitably break the first time (WiFi drivers, DNS port conflicts, forgotten passwords). The troubleshooting doc at the end covers every real issue this guide's author ran into.

## Prerequisites

A domain name, any TLD, around five to fifteen dollars a year. If you would rather not buy one, a free DuckDNS subdomain works identically. A Tailscale account, free tier is enough. Your laptop with SSH, a terminal, and a text editor. Basic comfort with SSH and copy-pasting commands.

## Reading order

Follow the docs in numerical order. Each assumes the previous is done.

1. [OpenMediaVault base install](docs/01-openmediavault.md)
2. [Docker and Portainer](docs/02-portainer.md)
3. [Immich](docs/03-immich.md)
4. [Tailscale](docs/04-tailscale.md)
5. [Pi-hole](docs/05-pihole.md)
6. [Nginx Proxy Manager](docs/06-npm.md)
7. [Homepage dashboard](docs/07-homepage.md)
8. [Split DNS](docs/08-split-dns.md)
9. [Troubleshooting](docs/99-troubleshooting.md)

## Placeholders

The guide uses these throughout. Substitute your own values as you go.

`<NUC_LAN_IP>` is your NUC's IP on your home network, for example `192.168.1.50`. `<NUC_TS_IP>` is its Tailscale IP, for example `100.x.y.z`. `<YOUR_DOMAIN>` is your domain, for example `example.com`. `<TIMEZONE>` is your timezone identifier such as `Asia/Karachi` or `America/New_York`. `<STRONG_PASSWORD>` is a password you generate with a password manager.

## License

MIT. Do what you want with this.
