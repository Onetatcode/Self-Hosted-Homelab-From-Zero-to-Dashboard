# 06, Nginx Proxy Manager

Nginx Proxy Manager (NPM) is a web UI for managing an nginx reverse proxy. It lets each of your services be reached at a clean subdomain like `photos.yourdomain.com` instead of `http://<NUC_IP>:2283`. It also handles SSL certificates via Let's Encrypt when you want proper HTTPS later.

Estimated time is about thirty minutes for the base setup plus reverse proxy entries for each service.

Assumes [02, Docker and Portainer](02-portainer.md) and [05, Pi-hole](05-pihole.md) are done.

## Confirming port 80 is free

NPM binds to ports 80 and 443. Something else may be holding port 80. Check first.

```bash
sudo ss -tlnp | grep ':80 '
```

If OMV's built-in nginx shows up (which it does by default), that is because you did not move OMV's web UI off port 80 in [doc 01](01-openmediavault.md). Do that now, in OMV go to System, Workbench, and set Port to `8080`. Save and apply. Port 80 is then free.

## Deploying NPM

In Portainer, add a stack named `npm`. Paste this compose.

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt

volumes:
  npm_data:
  npm_letsencrypt:
```

Ports 80 and 443 handle the actual proxied traffic (HTTP and HTTPS). Port 81 is NPM's own admin UI. Both named volumes preserve configuration and certificates across container restarts.

Deploy the stack. Wait about thirty seconds for the image to pull and containers to boot.

## First login

Open `http://<NUC_TS_IP>:81` in your browser. Default credentials are `admin@example.com` with password `changeme`. You will be forced to set a new email and password immediately. Use something strong and save it.

## Adding proxy hosts, one per service

The general pattern is the same for every service: NPM listens for requests to a subdomain, and forwards them to the actual container running on some IP and port. Since we are using Tailscale IPs and Pi-hole has the local DNS records already, the subdomains resolve to the NUC where NPM catches them and routes them correctly.

To add one, click Hosts in the top nav, then Proxy Hosts, then Add Proxy Host.

For Immich, use domain name `photos.<YOUR_DOMAIN>`, scheme `http`, forward hostname `<NUC_TS_IP>`, forward port `2283`. Enable Block Common Exploits and Websockets Support (Immich needs websockets for the mobile app). Leave SSL off for now. Save.

For Portainer, domain `portainer.<YOUR_DOMAIN>`, scheme `https` (Portainer uses HTTPS on 9443, not HTTP), forward hostname `<NUC_TS_IP>`, forward port `9443`. Enable Block Common Exploits and Websockets Support (Portainer's UI needs it too). Save.

For OMV, domain `omv.<YOUR_DOMAIN>`, scheme `http`, forward hostname `<NUC_TS_IP>`, forward port `8080` (or whatever port you moved OMV to). Enable Block Common Exploits and Websockets. Save.

For Pi-hole, domain `pi.<YOUR_DOMAIN>`, scheme `http`, forward hostname `<NUC_TS_IP>`, forward port `8053`. Enable Block Common Exploits. Websockets can stay off, Pi-hole does not need them. Save. Access it at `http://pi.<YOUR_DOMAIN>/admin` (Pi-hole's UI lives under `/admin`).

For NPM itself, domain `npm.<YOUR_DOMAIN>`, scheme `http`, forward hostname `<NUC_TS_IP>`, forward port `81`. Enable Block Common Exploits and Websockets. Save.

## Testing

Open each of the following in your browser to confirm the routing works: `http://photos.<YOUR_DOMAIN>`, `http://portainer.<YOUR_DOMAIN>`, `http://omv.<YOUR_DOMAIN>`, `http://pi.<YOUR_DOMAIN>/admin`, `http://npm.<YOUR_DOMAIN>`.

Each should load the corresponding service's login page. No IP, no port in the URL.

If they do not load and you see `NR_NAME_NOT_RESOLVED` or similar, check that Pi-hole has the DNS records (from [doc 05](05-pihole.md)) and that Tailscale split DNS is configured (covered in [doc 08](08-split-dns.md)). Without both of those, the subdomains do not resolve to your NUC in the first place.

## About SSL certificates

You can add HTTPS via Let's Encrypt DNS-01 challenge, which does not require exposing the NUC to the public internet. It does require your DNS provider to have an API supported by Let's Encrypt. Cloudflare is easiest. If your registrar is name.com or somewhere else without a well-supported API, you have two options: transfer DNS management (not the registration) to Cloudflare (free, easy), or accept the self-signed HTTPS warning per browser. For a private homelab that only you and your household use, self-signed is fine, browsers accept once per device.

SSL setup is not covered in this guide because it depends heavily on your DNS provider. Search "NPM Let's Encrypt DNS-01 [your provider]" for a step-by-step, they are all short.

## What you have now

NPM is running with proxy hosts for every service you deployed. Each service is now reachable at `http://<service>.<YOUR_DOMAIN>` with no ports or IPs in the URL. HTTPS is optional and can be added later.

Next: [07, Homepage dashboard](07-homepage.md).
