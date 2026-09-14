# 02, Docker and Portainer

Docker is the container runtime. Portainer is a web UI that lets you manage Docker without touching the CLI for every task. Together they are the foundation everything else in this guide runs on.

Estimated time is about fifteen minutes.

Assumes [01, OMV base install](01-openmediavault.md) is done.

## Installing Docker

SSH into the NUC as root. Install Docker using their official convenience script, which handles the correct repo setup for Debian automatically.

```bash
curl -fsSL https://get.docker.com | sh
```

Verify with `docker --version` and `docker ps`. The version should print something like `Docker version 27.x` or higher. The `ps` command should print an empty list because you have no containers running yet. That is expected on a fresh install.

## Installing Portainer

Portainer itself runs as a container. First create a named volume for its config so it survives container restarts.

```bash
docker volume create portainer_data
```

Then run Portainer with access to the Docker socket, which is how it manages other containers.

```bash
docker run -d \
  --name portainer \
  --restart=always \
  -p 9443:9443 \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

The `-d` flag runs it in the background. The two `-p` mappings expose HTTPS on 9443 and HTTP on 9000. The Docker socket mount is what gives Portainer the ability to see and control other containers.

## First login and the setup token

Recent Portainer versions require a setup token when creating the first admin account. This is a security measure so that a random person on your network cannot claim the admin account first. The token is printed to the container logs on startup.

```bash
docker logs portainer 2>&1 | grep -i "token\|password"
```

Look for a line like `Portainer setup token: XXXXXXXXXXXXXXXXXXXX` and copy it. Then open Portainer in your browser at `https://<NUC_LAN_IP>:9443`. Your browser will warn that the site is not secure because Portainer uses a self-signed certificate on first boot. Click through the warning. We will fix HTTPS properly later with NPM.

On the setup screen, set your admin username (usually `admin`), a strong password with at least twelve characters, and paste the setup token. Save the password in a password manager. If you lose it there is a recovery process, but it is much easier to just not lose it.

## Selecting your environment

After creating the admin account you land on the environments page. Click `local`. That is the Docker environment on the machine Portainer itself is running on. You now see Portainer's dashboard for your NUC.

## Where things live in Portainer

Inside the local environment, the sidebar has several sections. The Dashboard shows an overview. Stacks is where Docker Compose applications live and is where you will spend most of your time in this guide. Containers, Images, Volumes, and Networks let you inspect and manage individual Docker objects. Most of the following docs deploy services via Stacks because Compose files are cleaner than individual container runs.

## If you lose the admin password

Save this for reference. Portainer has a built-in reset that uses a helper image against your existing data volume.

```bash
docker stop portainer
docker run --rm -v portainer_data:/data portainer/helper-reset-password
docker start portainer
```

The helper prints a new random password to the terminal, shown exactly once, so copy it before doing anything else. Log in with `admin` and the printed password, then change it under the top-right user icon and My account. This only works if the `portainer_data` volume still exists. If you deleted it, you would need to redo the setup token flow entirely.

## What you have now

Docker is installed and running. Portainer is at `https://<NUC_LAN_IP>:9443` and can deploy any Docker service via web UI.

Next: [03, Immich](03-immich.md).
