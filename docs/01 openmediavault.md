# 01, OpenMediaVault Base Install

OpenMediaVault is a Debian-based NAS operating system with a web UI. We use it as the base OS on the NUC. Everything else (Docker, our services) runs on top of it.

Estimated time is about forty-five minutes, most of which is waiting for the installer.

## Downloading the ISO

From your laptop, create a working directory and pull the current stable release.

```bash
mkdir -p ~/Downloads/nas && cd ~/Downloads/nas
wget -c "https://sourceforge.net/projects/openmediavault/files/iso/8.3.1/openmediavault_8.3.1-amd64.iso/download" -O openmediavault.iso
```

The `-c` flag tells wget to resume if interrupted, so you can Ctrl+C and re-run without starting over. Verify the download by comparing its SHA256 against the official hash on the OMV download page.

```bash
sha256sum openmediavault.iso
```

For version 8.3.1 the expected hash is `5ada7dea09dbb10365b6f245515e71c79c5398d8f8f1acda675ee6d8b572d2e5`. If yours does not match, delete the file and re-download.

## Flashing the USB stick

Flashing destroys everything on the target device, so identifying the right one is the most important step in this doc. Run `lsblk` before plugging the USB in and note what you see. Plug the USB in, wait three seconds, run `lsblk` again. The device that just appeared is your USB, typically `/dev/sda` or `/dev/sdb`. Confirm by size (should match your USB, so 8 GB, 16 GB, etc.) and by the fact it was not there before. Do not confuse `sda` with `nvme0n1`. The latter is likely your laptop's internal SSD. Wiping it destroys your laptop.

If your USB auto-mounted, unmount it. Then wipe the existing partition signatures.

```bash
sudo umount /dev/sda1
sudo wipefs -a /dev/sda
```

Flash the ISO with dd.

```bash
sudo dd if=~/Downloads/nas/openmediavault.iso of=/dev/sda bs=4M status=progress conv=fsync
```

Takes two to five minutes depending on your USB speed. When done, safely eject with `sudo eject /dev/sda` and physically unplug.

## Booting the NUC from USB

Plug the USB, a keyboard, and a monitor into the NUC. Power it on and immediately press F10 repeatedly. That is the boot menu key on Intel NUCs. Other brands use F12 or Esc. Select your USB from the boot menu. The OMV installer starts, text-mode with a blue background.

## Walking through the installer

Most defaults are fine. For language, country, and keyboard, pick your own. Hostname can be short and simple like `nuc` or `homelab`. Domain name during install can be left blank or set to `local`. Root password should be strong (write it down, you will rarely need it). Timezone is yours. For disk partitioning, choose "Guided, use entire disk", which wipes the internal SSD. Install GRUB to `/dev/sda` (or whatever the internal SSD shows as).

If the installer offers WiFi configuration and you have no ethernet cable, be aware that OMV's installer has limited WiFi support. If you get stuck here, see the WiFi section in the troubleshooting doc.

The installer runs and reboots. First boot lands you at a text-mode login prompt.

## First login

At the login prompt, log in as `root` with the password you set during install.

## Getting network working

Check current interfaces with `ip a`. Look for one with an IP address. If nothing has an IP, network is not configured yet.

If you plugged in ethernet, request an address from DHCP. First ensure the sbin paths are in your PATH since root's default PATH on Debian sometimes misses them.

```bash
export PATH=$PATH:/usr/sbin:/sbin
dhcpcd eno1
```

Replace `eno1` with your ethernet interface name from `ip a`. If WiFi is your only option, see the WiFi section in the troubleshooting doc, it is genuinely more painful than ethernet.

Once you have an IP, note it down.

```bash
ip a | grep 'inet ' | grep -v 127.0.0.1
```

This is your `<NUC_LAN_IP>` for the rest of the guide.

## Accessing the OMV web UI

From your laptop's browser, open `http://<NUC_LAN_IP>`. Default login is `admin` / `openmediavault`. Change this password immediately using the top-right user icon.

## Moving OMV off port 80

Later docs use Nginx Proxy Manager, which needs port 80. Free it now while it is easy. In the OMV web UI, navigate to System, then Workbench, and change Port from 80 to 8080. Save and apply. Your OMV UI is now at `http://<NUC_LAN_IP>:8080`. Update your bookmark.

## Enabling SSH

Navigate to System, Services, SSH. Toggle Enabled, save, apply. You can now SSH from your laptop with `ssh root@<NUC_LAN_IP>`. Most of the rest of this guide is done via SSH.

## What you have now

OMV is running on the NUC, its web UI is at `http://<NUC_LAN_IP>:8080`, SSH access works from your laptop, and port 80 is free for NPM later.

Next: [02, Docker and Portainer](02-portainer.md).
