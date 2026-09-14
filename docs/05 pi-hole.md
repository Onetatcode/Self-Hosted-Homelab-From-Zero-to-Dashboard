# 05, Pi-hole

Pi-hole is a DNS server that also blocks ads and trackers at the network level. Every device using it as their DNS gets ad-blocked automatically, no browser extensions needed. In this stack it also serves as the local DNS that resolves your homelab subdomains like `photos.yourdomain.com` to your NUC's Tailscale IP.

This is the longest doc in the guide because Pi-hole's port 53 conflicts with several things and requires real setup work.

Estimated time is about thirty minutes.

Assumes [02, Docker and Portainer](02-portainer.md) and [04, Tailscale](04-tailscale.md) are done.

## Freeing port 53

Pi-hole needs port 53 (DNS) exclusively. On Debian, `systemd-resolved` binds to port 53 by default as a stub resolver. Confirm this is the problem by checking who owns port 53.

```bash
sudo ss -tlnp | grep ':53 '
sudo ss -ulnp | grep ':53 '
```

If you see `systemd-resolve` in the output, we need to disable its stub listener without killing the whole service (it still handles other name resolution logic). Create a drop-in config that turns off the stub, then update the system's resolver to use a public DNS while Pi-hole is being set up.

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
cat << 'EOF' | sudo tee /etc/systemd/resolved.conf.d/disable-stub.conf
[Resolve]
DNSStubListener=no
EOF

sudo rm -f /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

Verify port 53 is now free.

```bash
sudo ss -tlnp | grep ':53 '
sudo ss -ulnp | grep ':53 '
```

Both should return nothing.

## Deploying Pi-hole with host network mode

Pi-hole v6 works most reliably when the container runs in host network mode. That is because Docker's default port publishing goes through `docker-proxy`, which changes the source interface of incoming packets. Tailscale's built-in firewall rules (which drop packets from the 100.x.y.z range that arrive on non-Tailscale interfaces) end up blocking legitimate queries. Host network mode bypasses `docker-proxy` entirely and lets Pi-hole bind directly to the host's network stack.

In Portainer, add a stack named `pihole` and paste this compose.

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: '<TIMEZONE>'
      FTLCONF_webserver_port: '8053'
      FTLCONF_dns_listeningMode: 'all'
      FTLCONF_webserver_api_password: '<STRONG_PASSWORD>'
    volumes:
      - pihole_etc:/etc/pihole
      - pihole_dnsmasq:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
      - SYS_TIME
      - SYS_NICE

volumes:
  pihole_etc:
  pihole_dnsmasq:
```

Replace `<TIMEZONE>` and `<STRONG_PASSWORD>` with your values. The password is what you will use to log into Pi-hole's admin UI.

The `FTLCONF_webserver_port: '8053'` bit is important. Because we are using host network mode, we cannot remap ports, so we have to tell Pi-hole itself to serve its admin UI on 8053 instead of the default 80. Port 80 stays free for NPM later. The `FTLCONF_dns_listeningMode: 'all'` allows queries from any interface, which is what lets Tailscale clients reach it.

Deploy the stack. Verify it started cleanly.

```bash
docker ps | grep pihole
docker logs pihole 2>&1 | tail -10
sudo ss -tlnp | grep -E ':53 |:8053 '
```

You should see the container running healthy, clean logs with no errors, and both ports 53 and 8053 bound directly to the host (owned by `pihole-FTL`, not `docker-proxy`).

## First login and password

Open `http://<NUC_LAN_IP>:8053/admin` in your browser. Use the password you set in `FTLCONF_webserver_api_password`. If it does not accept it (special characters in YAML can be finicky), you can reset it from the command line.

```bash
docker exec -it pihole pihole setpassword 'NewSimplePassword'
```

Then log in with the new one.

## Adding blocklists

Fresh Pi-hole v6 ships with no blocklists at all, so it will not block anything until you add some. In the Pi-hole admin UI, go to Lists (in the left sidebar) and add block lists (also called adlists or deny lists depending on wording). Start with Steven Black's unified hosts list at `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`. That is the canonical Pi-hole starter list and blocks around 130,000 ad, tracker, and malware domains.

For broader coverage add a few more. AdAway's list at `https://adaway.org/hosts.txt`, Firebog's AdGuard-DNS list at `https://v.firebog.net/hosts/AdguardDNS.txt`, Firebog's Easyprivacy at `https://v.firebog.net/hosts/Easyprivacy.txt`, and Firebog's Easylist at `https://v.firebog.net/hosts/Easylist.txt` are all reliable.

If you want aggressive blocking, add hagezi's multi list at `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/multi.txt` or oisd at `https://big.oisd.nl/`. These block a lot and occasionally break legitimate services. Remove them if things get weird.

After adding lists, go to Tools and Update Gravity, then click Update. This downloads all lists, deduplicates them, and compiles the block list. Takes thirty to sixty seconds. Watch the progress log, it should finish with a domain count in the hundreds of thousands.

## Adding local DNS records for your homelab subdomains

This is the part that makes clean URLs like `photos.yourdomain.com` resolve to your NUC. In the Pi-hole UI, go to System and then Local DNS Records. Add one entry per subdomain you plan to expose, all pointing at your NUC's Tailscale IP.

For a typical setup you would add `photos.<YOUR_DOMAIN>`, `omv.<YOUR_DOMAIN>`, `npm.<YOUR_DOMAIN>`, `pi.<YOUR_DOMAIN>`, `portainer.<YOUR_DOMAIN>`, and `home.<YOUR_DOMAIN>`, each mapped to `<NUC_TS_IP>`. Save after each. These records only apply when Pi-hole is queried, which is why the next step (split DNS) matters.

## Verifying Pi-hole works over Tailscale

From your laptop, install `dig` if you do not have it, then test resolution both ways.

```bash
dig @<NUC_TS_IP> photos.<YOUR_DOMAIN>
dig @<NUC_TS_IP> google.com
```

The first should return `<NUC_TS_IP>` (your local record). The second should return real Google IPs (Pi-hole forwarding to upstream DNS). If both work, Pi-hole is fully functional and reachable via Tailscale. Also verify ad blocking with `dig @<NUC_TS_IP> doubleclick.net` which should return `0.0.0.0`.

## What you have now

Pi-hole is running on your NUC in host network mode, serving DNS on port 53 and its admin UI on port 8053. It blocks ads via loaded blocklists and resolves your homelab subdomains to your Tailscale IP. Clients that query it get both benefits automatically.

Next: [06, Nginx Proxy Manager](06-npm.md) to add clean URLs, then [08, Split DNS](08-split-dns.md) to wire Pi-hole into your Tailscale network as the DNS server.

## Troubleshooting

If you get "wrong password" on the admin UI, the env var may not have applied cleanly. Reset with `docker exec -it pihole pihole setpassword 'NewPassword'`.

If `dig @<NUC_TS_IP>` times out from your laptop but `dig @127.0.0.1` works on the NUC itself, that is the Tailscale firewall rule dropping packets. The fix is host network mode (already covered above). If you accidentally deployed with bridge networking and port mapping, redeploy with `network_mode: host`.

If port 53 shows something other than `pihole-FTL` after deployment, `systemd-resolved` came back. Re-check that the drop-in config exists and restart it, then restart Pi-hole.
