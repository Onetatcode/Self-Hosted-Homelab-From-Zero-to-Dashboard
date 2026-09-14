# 04, Tailscale

Tailscale is a mesh VPN built on WireGuard. It gives every device you enrol a stable private IP in the `100.x.y.z` range, and those devices can reach each other from anywhere on the internet without port forwarding or public IPs.

For this homelab, Tailscale is the reason your services can stay completely off the public internet while still being accessible from your phone, laptop, or a friend's computer.

Estimated time is about ten minutes.

Assumes [02, Docker and Portainer](02-portainer.md) is done. Nothing else in the stack depends on Tailscale, but everything from Pi-hole onwards assumes you have it.

## Creating a Tailscale account

Go to `https://login.tailscale.com/start` and sign in with GitHub, Google, or Microsoft. The free tier allows up to a hundred devices, which is more than any homelab will ever need.

## Installing Tailscale on the NUC

SSH into the NUC as root and run the official install script.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This adds Tailscale's apt repo and installs the client. Then start it and authenticate.

```bash
tailscale up
```

The command prints a URL. Copy it into a browser on any device where you are already logged into your Tailscale account and approve the machine. The NUC is now on your Tailscale network.

Get its assigned Tailscale IP with `tailscale ip -4`. Note this down as `<NUC_TS_IP>` for the rest of the guide, typically something like `100.66.12.66`.

## Installing Tailscale on your other devices

Install on your laptop, phone, and any other device you want to use to reach the NUC. Downloads are at `https://tailscale.com/download` for macOS, Windows, iOS, and Android. On Linux the same install script works. Each device signs into the same account and appears automatically.

Verify from your laptop by pinging the NUC's Tailscale IP.

```bash
ping <NUC_TS_IP>
```

Latency should be very low (single-digit or low-teens milliseconds) since Tailscale mostly uses direct peer-to-peer connections.

## Updating your Immich app

Now that you can reach the NUC over Tailscale from anywhere, update the Immich mobile app to use the Tailscale IP as its server endpoint instead of the LAN IP. Open the app, go to settings, and change the server endpoint URL to `http://<NUC_TS_IP>:2283`. Now uploads work whether you are at home or away.

## Optional, subnet routing

If you want other devices on your home LAN (that do not have Tailscale installed) to be reachable from your Tailscale devices, you can enable subnet routing on the NUC. This is not required for this guide since everything we set up runs on the NUC itself. Documentation is at `https://tailscale.com/kb/1019/subnets` if you want to explore it later.

## What you have now

Tailscale is running on your NUC and any other devices you enrolled. The NUC has a stable private IP reachable from anywhere. Immich now works remotely, and every service from here on will be reachable the same way.

Next: [05, Pi-hole](05-pihole.md).

## Troubleshooting

If `tailscale up` prints the URL but the browser approval does nothing, check the Tailscale admin console at `https://login.tailscale.com/admin/machines`. Sometimes the machine appears there and needs manual approval, especially on ACL-restricted tailnets.

If `ping <NUC_TS_IP>` from your laptop times out but the machine shows online in the admin console, check that both devices are on the same tailnet (not accidentally signed into different accounts). Also confirm neither device is running a strict firewall blocking WireGuard.
