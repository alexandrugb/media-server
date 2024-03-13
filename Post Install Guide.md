
### Faster Updates

- `sudo vim /etc/dnf/dnf.conf`
-  add the following lines:
- `max_parallel_downloads=10`
- `fastestmirror=true`

#### Install RPM Fusion Repo

- `sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm`
- `sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm`
- `sudo dnf groupupdate core`
- `sudo dnf makecache`

### Update

- `sudo dnf -y update`
- Reboot

### Install NVIDIA drivers

* `sudo dnf update --refresh`
* `sudo dnf install kernel-devel kernel-headers gcc make dkms acpid libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig`
* `sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda`
* reboot

## Install Docker and Docker Compose

* `sudo dnf -y install dnf-plugins-core`
* `sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo`
* install the latest version : `sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
* start docker `sudo systemctl start docker`
* enable docker `sudo systemctl start docker`
* verify that docker works `sudo docker run hello-world`
* add your user to the `docker` group `sudo usermod -aG docker $USER`
* reboot
* install docker compose `sudo dnf install docker-compose`
* create a docker network `docker network create <name-of-network>`

## Install Nvidia Container Plan

* `curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo`
* `sudo dnf config-manager --enable nvidia-container-toolkit-experimental`
* `sudo dnf install -y nvidia-container-toolkit`
* Configure the container runtime by using the `nvidia-ctk` command: `sudo nvidia-ctk runtime configure --runtime=docker`
* Restart the Docker daemon: `sudo systemctl restart docker`

## Install Portainer

* First, create the volume that Portainer Server will use to store its database: `docker volume create portainer_data`
* Then, download and install the Portainer Server container: `docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ee:latest`

## Install Pulseway

* ` rpm -ivh https://www.pulseway.com/download/pulseway_x64.rpm`





