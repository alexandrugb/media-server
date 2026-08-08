# Post Install Guide

Host preparation for a Fedora Workstation media server. Work through these in
order, then follow `README.md` to bring the stacks up.

Commands assume Fedora 41 or newer, which uses **dnf5**. A few flags changed
from dnf4; the old `groupupdate` and `config-manager --add-repo` forms no
longer work.

## 1. RPM Fusion

```bash
sudo dnf install \
  https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf group upgrade core
sudo dnf makecache
```

## 2. Full update

```bash
sudo dnf -y upgrade --refresh
sudo reboot
```

## 3. NVIDIA drivers

```bash
sudo dnf install kernel-devel kernel-headers gcc make dkms acpid \
  libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
```

Wait for the kernel module to finish building before rebooting, otherwise you
boot to a black screen:

```bash
modinfo -F version nvidia    # should print a version, not an error
sudo reboot
```

Verify after reboot with `nvidia-smi`.

## 4. Docker Engine

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Log out and back in for the group change to apply, then verify:

```bash
docker run --rm hello-world
docker compose version    # must be v2.23 or newer
```

Do **not** `dnf install docker-compose`. That is the obsolete Python v1 tool.
The `docker-compose-plugin` package above provides `docker compose`, and this
repo needs v2.23+ for inline `configs` support in `stacks/monitoring.compose.yaml`.

## 5. NVIDIA Container Toolkit

Required for Plex hardware transcoding.

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
  | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo dnf install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Leave the `-experimental` repo disabled. Verify:

```bash
docker run --rm --runtime=nvidia --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

## 6. Shared network and directories

```bash
docker network create media

export CONFIG_ROOT=/home/$USER/docker/config
export DATA_ROOT=/mnt/media

mkdir -p "$CONFIG_ROOT"
mkdir -p "$DATA_ROOT"/{movies,tvseries}
sudo chown -R $USER:$USER "$DATA_ROOT"
```

`DATA_ROOT` must be a single filesystem. Splitting downloads and libraries
across mounts forces slow full copies instead of instant moves.

## 7. SELinux

Fedora enforces SELinux. Bind mounts into containers need a relabel or the
container gets permission denied on files it can plainly see:

```bash
sudo semanage fcontext -a -t container_file_t "$DATA_ROOT(/.*)?"
sudo semanage fcontext -a -t container_file_t "$CONFIG_ROOT(/.*)?"
sudo restorecon -Rv "$DATA_ROOT" "$CONFIG_ROOT"
```

If `semanage` is missing: `sudo dnf install policycoreutils-python-utils`.

## 8. Firewall

Nothing is exposed to the internet except Seerr, and that goes through the
Cloudflare tunnel as an outbound connection — it needs no inbound port at all.

The only port worth opening on the LAN is Plex, so TVs and phones can reach the
server directly instead of relaying:

```bash
sudo firewall-cmd --permanent --add-port=32400/tcp   # plex, LAN only
sudo firewall-cmd --reload
```

### firewalld does not block Docker

**Important.** Docker writes its own iptables rules into the `DOCKER` chain,
which is evaluated *before* firewalld's filtering. Any port published with
`ports:` in a compose file is reachable from the LAN whether or not
`firewall-cmd` lists it. Adding ports above is only about non-Docker services.

So `firewall-cmd --list-ports` showing a short list does **not** mean the *arr
apps and torrent UIs are closed. Check what is genuinely reachable:

```bash
sudo ss -tlnp | grep -v '127.0.0.1'
```

Anything bound to `0.0.0.0` is exposed to your whole LAN.

To actually restrict a service, bind its published port to a specific address
in the compose file rather than relying on the firewall:

```yaml
ports:
  - 127.0.0.1:3000:3000        # localhost only
  - 100.82.84.29:3000:3000     # Tailscale interface only
```

Note this affects the LAN only. None of it is internet-facing: the Cloudflare
tunnel makes an outbound connection and publishes Seerr alone.

## 9. Deploy

Komodo replaces Portainer as the stack manager for this setup:

```bash
cd komodo
cp .env.example .env      # then edit the secrets
docker compose up -d
```

Open <http://localhost:9120> and log in with the credentials from `komodo/.env`.
From there, create the `media` and `monitoring` Stacks using the files in
`stacks/`. See `README.md` for the rest.

## 10. Pulseway — server-down email alerts

This is the one piece of monitoring that must live outside Docker.

Prometheus and Alertmanager can only tell you a *container* is unhealthy. They
cannot tell you the *host* is down, because they die with it. A server that has
hard-frozen sends no alert about being frozen.

Pulseway runs as a systemd service and reports to an external cloud, so when
the machine stops checking in, you get the email.

```bash
sudo rpm -ivh https://www.pulseway.com/download/pulseway_x64.rpm
sudo systemctl enable --now pulseway
```

Configure the account and email notifications:

```bash
sudo /usr/share/pulseway/config
sudo systemctl restart pulseway
```

In the Pulseway web console, under the system's notification settings, enable
at minimum:

- **System offline** — the one that actually catches a freeze or power loss.
  Set the grace period to a few minutes so a reboot does not page you.
- Low disk space on the media filesystem.
- Optionally, service-stopped notifications for `docker`.

Make sure the account's notification channel is set to email.

### Division of labour

| Failure | Caught by |
| --- | --- |
| Host frozen, crashed, powered off, offline | Pulseway |
| Docker daemon stopped | Pulseway (service monitor) |
| A container unhealthy or restart-looping | Prometheus + Alertmanager |
| Disk filling up, memory or CPU pegged | Either, Prometheus with more detail |
| Radarr/Sonarr indexer or download failures | Prometheus via exportarr |

Alertmanager currently has no receiver configured, so container-level alerts
only appear in the Grafana and Alertmanager UIs. If you want those by email
too, fill in the `email_configs` block in `monitoring/alertmanager.yml` — but
note it will not fire when the host itself is gone. That remains Pulseway's job.
