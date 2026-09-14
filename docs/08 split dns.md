# 08, Split DNS

Split DNS is the trick that makes your homelab feel complete. It configures Tailscale to route DNS queries for your specific domain through your NUC's Pi-hole, while every other query goes to Tailscale's default resolver. The result is that your Tailscale devices resolve `photos.yourdomain.com` to your NUC's private IP, while the public internet cannot resolve it at all.

This is what turns clean subdomain URLs into a real experience across all your devices without any manual DNS config on each one.

Estimated time is about fifteen minutes.

Assumes [04, Tailscale](04-tailscale.md) is up and [05, Pi-hole](05-pihole.md) is working (verified with `dig @<NUC_TS_IP>` returning real results). Also assumes you have added Local DNS records in Pi-hole for each subdomain you plan to use.

## The concept

Public DNS for your domain (managed by your registrar) resolves whatever public records you have. Your marketing website, MX records for email, and so on. This does not change. When a Tailscale-connected device queries anything ending in your domain, Tailscale intercepts the query and sends it to Pi-hole on your NUC instead of Tailscale's default DNS. Pi-hole either returns a local record (for your homelab subdomains) or forwards it to public DNS if nothing matches. All other queries (google.com, github.com) go through Tailscale's default DNS as normal, not through Pi-hole.

The result: your Tailscale devices know how to reach `photos.yourdomain.com` (via your NUC), but nothing outside Tailscale can even resolve that name. Total privacy for your homelab subdomains, no changes to your public DNS.

## Prerequisites check

Before enabling split DNS, verify Pi-hole is fully reachable via Tailscale from your laptop.

```bash
dig @<NUC_TS_IP> photos.<YOUR_DOMAIN>
dig @<NUC_TS_IP> google.com
```

The first should return `<NUC_TS_IP>`. The second should return real Google IPs. If either fails, do not proceed. Fix Pi-hole first (see [doc 05](05-pihole.md) troubleshooting).

## Configuring split DNS in Tailscale

Open the Tailscale admin console at `https://login.tailscale.com/admin/dns` in your browser.

Under Nameservers, click Add nameserver and select Custom. Fill in your NUC's Tailscale IP as the nameserver address. Toggle "Restrict to search domain" on. Set the search domain to your domain (`<YOUR_DOMAIN>`). Save.

That is it. Tailscale immediately pushes the config to every device on your tailnet.

## Verifying it works

From your laptop, do NOT specify a DNS server. Just query normally.

```bash
dig photos.<YOUR_DOMAIN>
```

If split DNS is working, the answer section should show your NUC's Tailscale IP, and the SERVER line should show `127.0.0.53` or similar local resolver (not `<NUC_TS_IP>` directly, because Tailscale routes the query behind the scenes).

You can also check what your system sees:

```bash
resolvectl status | grep -B1 -A3 "<YOUR_DOMAIN>"
```

You should see your domain listed as a routed domain.

## Testing in a browser

Open `http://home.<YOUR_DOMAIN>` (assuming you set up Homepage in [doc 07](07-homepage.md)). It should load your dashboard. Same for every other service subdomain. All of it works from your laptop, phone, or any Tailscale-enrolled device.

If you disconnect Tailscale on your laptop and try again, the URLs should stop working. That is the whole point, nothing is exposed publicly.

## What you have now

Your homelab is fully connected. Any Tailscale device can reach `photos.yourdomain.com`, `omv.yourdomain.com`, and every other service with no manual config. Public queries for those names return nothing, keeping the homelab invisible from the outside.

Your public website (if any) at `www.yourdomain.com` continues working exactly as before because Tailscale's split DNS only intercepts queries when the device asking is on your tailnet.

## Troubleshooting

If subdomains do not resolve on your laptop, check that Tailscale is actually running and connected: `tailscale status`. If Tailscale shows offline, your laptop is not using the split DNS override.

If a subdomain resolves but the page does not load, that is not a DNS problem. Check that NPM's proxy host is configured for that subdomain and that the target service is running.

If everything works on your laptop but not on your phone, force the Tailscale app on the phone to refresh (toggle off and on). Some mobile OSes cache DNS aggressively.

If you disabled Tailscale's default DNS (MagicDNS) somewhere along the way, split DNS still works but you lose the ability to reach machines by name like `nuc.tail-xxxx.ts.net`. Re-enable MagicDNS if you want both features.
