# Homelab Build Journal

**Author:** Peter Antai, Robotics & HRI Researcher  
**Stack:** Ubuntu Server 22.04 · Docker · ROS2 Humble · Gitea · Mosquitto · Portainer  
**Purpose:** Personal homelab for robotics research, HRI project development, and portfolio infrastructure

> IP addresses, usernames and disk identifiers in this public copy are replaced with placeholders such as `<SERVER_IP>`.

---

## Overview

This journal documents a two-machine homelab built from scratch: a dedicated local server and a development laptop, configured as one robotics research and development environment. Every hardware and software decision is recorded here as an engineering artifact in its own right.

The journal covers two generations of the server:

1. **Server v1 (June 2026):** a second-hand HP ProDesk 400 G3 SFF, upgraded and configured as a headless Docker host.
2. **Server v2 (November 2026):** a self-built AM4 tower that replaced the ProDesk after a hardware failure. The existing drives, operating system and services were moved across without a reinstall.

The infrastructure supports active portfolio projects, including a multimodal HRI drone platform (Python, OpenCV, ROS2, MediaPipe, Whisper ASR) and a clinical speech and vital signs anomaly dashboard.

### Timeline

| Date | Milestone |
|---|---|
| June 2026 | ProDesk server built: CPU upgrade, Ubuntu Server, static IP, Docker, Gitea, Portainer, Mosquitto |
| June 2026 | ThinkPad T450 configured as the development machine (ROS2 Humble, Docker, VS Code) |
| October 2026 | ProDesk stops powering on (HP blink code: 3 red / 4 white) |
| November 2026 | Replacement AM4 server built; drives and services migrated; ProDesk fault traced to its PSU |

---

## Current State

| | |
|---|---|
| **Server** | Self-built AM4 tower: Ryzen 7 5700G, 16GB DDR4, 256GB SSD + 1TB HDD |
| **OS** | Ubuntu Server 22.04.5 LTS, HWE kernel 6.8 |
| **Services** | Gitea (3000), Portainer CE (9000), Mosquitto MQTT (1883), plus a containerised personal alerting service |
| **Dev machine** | Lenovo ThinkPad T450, Ubuntu 22.04, ROS2 Humble |
| **Management** | Headless over SSH |

### Architecture

```
┌──────────────────────────────────────────┐
│              Home Network (LAN)          │
│                                          │
│  ┌──────────────┐   SSH / HTTP           │
│  │  T450 Laptop │◄─────────────────┐     │
│  │  Ubuntu 22.04│                  │     │
│  │  ROS2 Humble │                  │     │
│  │  Docker      │                  │     │
│  │  VS Code     │                  │     │
│  └──────┬───────┘                  │     │
│         │ git push                 │     │
│         ▼                          │     │
│  ┌─────────────────────────────────┴──┐  │
│  │  AM4 Server (v2) · static IP       │  │
│  │  Ubuntu Server 22.04 · HWE 6.8     │  │
│  │  Ryzen 7 5700G · 16GB RAM          │  │
│  │                                    │  │
│  │  ┌─────────┐  ┌──────────────┐     │  │
│  │  │  Gitea  │  │  Portainer   │     │  │
│  │  │  :3000  │  │    :9000     │     │  │
│  │  └─────────┘  └──────────────┘     │  │
│  │  ┌──────────────────────────┐      │  │
│  │  │  Mosquitto MQTT  :1883   │      │  │
│  │  └──────────────────────────┘      │  │
│  │                                    │  │
│  │  Samsung SSD      → OS (LVM)       │  │
│  │  Hitachi 1TB HDD  → /mnt/data      │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

---

## Hardware

### Server v2: self-built AM4 tower (current)

| Component | Spec | Notes |
|---|---|---|
| CPU | AMD Ryzen 7 5700G | 8C/16T, integrated graphics, so the server runs with no discrete GPU |
| Cooler | Arctic Alpine 23 CO | Continuous-operation bearing, chosen for 24/7 use |
| Motherboard | Gigabyte B550M Aorus Elite | BIOS flashed with Q-Flash Plus before the CPU was fitted |
| RAM | 1× 16GB DDR4 2666MHz | Carried over from v1; single channel until a second stick is added |
| PSU | MSI MAG A650GL 650W 80+ Gold | Headroom for a future GPU |
| Case | CiT Jet Stream | Chosen for drive capacity: 3.5" bays plus three 5.25" bays |
| Fans | Arctic P12 PWM PST (5-pack) | Each fan tested individually before fitting |
| OS drive | Samsung 256GB 2.5" SATA SSD | Carried over from v1 with the OS intact |
| Data drive | Hitachi HTS541010A9E680 1TB 2.5" HDD | Carried over from v1 with all service data |

### Server v1: HP ProDesk 400 G3 SFF (retired)

| Component | Spec | Notes |
|---|---|---|
| CPU | Intel Core i7-6700 @ 3.40GHz | Upgraded from i5-6500; 4C/8T |
| RAM | 16GB DDR4 2666MHz | |
| OS drive | Samsung 256GB 2.5" SATA SSD | Ubuntu Server root partition |
| Data drive | Hitachi HTS541010A9E680 1TB 2.5" HDD | Bulk data, Docker volumes, rosbags |
| Mount | DEEPFOX 2.5" to 3.5" dual-drive bracket | Both drives in a single 3.5" bay |
| Network | Realtek RTL8111/8168/8411 GbE | Static IP |

**CPU upgrade notes:** the i5-6500 to i7-6700 swap needed a BIOS flash first. The ProDesk BIOS was updated to the latest version (SP94777) with the i5 still installed, then the CPU was swapped. The board recognised the i7-6700 on first POST.

### Development machine: Lenovo ThinkPad T450

| Component | Spec |
|---|---|
| CPU | Intel Core i5-5300U (Broadwell) |
| OS | Ubuntu 22.04.x LTS |
| Storage | 256GB 2.5" SATA SSD |
| Role | Primary dev machine: ROS2, Docker, VS Code, Git |

---

## Part 1: The Original Build (June 2026)

Everything in this part was done on the ProDesk. The software configuration is unchanged on the current server except where Part 2 says otherwise.

### OS installation

**Ubuntu Server 22.04.5 LTS** was installed from a bootable USB stick (created with Rufus on Windows).

Key installer decisions:
- **Installation type:** Ubuntu Server, not minimized, because the full package set is needed for dev tooling
- **Storage:** Samsung SSD as the install target; Hitachi HDD left untouched
- **LVM:** enabled, to allow partition expansion later without a reinstall
- **OpenSSH:** enabled during install, which is essential for headless management
- **Snaps:** none selected; all services run in Docker for better control

The installer allocated 100GB to the root logical volume and left 135GB free. It was expanded straight after install:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

The root partition grew from 98GB to 232GB.

**Storage layout:**

```
Samsung SSD (OS)
├── part 1   1GB    /boot/efi   (FAT32)
├── part 2   2GB    /boot       (ext4)
└── part 3   235GB  LVM group ubuntu-vg
                └── ubuntu-lv  232GB  /  (ext4, expanded to full size)

Hitachi 1TB HDD (Data)
└── whole disk  916GB  /mnt/data  (ext4)
```

### Network configuration

A static IP was assigned with Netplan so that DHCP reassignment could not break SSH or the self-hosted services. The gateway was found with `ip route | grep default`.

`/etc/netplan/00-installer-config.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - <SERVER_IP>/24
      routes:
        - to: default
          via: <GATEWAY_IP>
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Applied with `sudo netplan apply`. The SSH session stayed connected, confirming a clean move from DHCP to static. File permissions were then tightened:

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

### Data drive

The 1TB HDD was formatted and mounted as a persistent data volume:

```bash
sudo mkfs.ext4 /dev/sda
sudo mkdir /mnt/data
sudo mount /dev/sda /mnt/data

# Persist across reboots
echo '/dev/sda /mnt/data ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

Mounting by device name (`/dev/sda`) worked on the ProDesk but caused a boot failure after the move to new hardware. See [Part 2](#part-2-failure-and-rebuild-octobernovember-2026) for the fix.

**Directory structure on the HDD:**

```
/mnt/data/
├── docker/          # Docker data root (relocated from SSD)
├── gitea/           # Gitea self-hosted Git server data
├── mosquitto/       # Mosquitto MQTT broker data
│   ├── config/
│   ├── data/
│   └── log/
├── projects/
│   ├── drone/       # Drone HRI platform project
│   ├── ros2/        # ROS2 workspace data and rosbags
│   └── clinical_dashboard/  # Healthcare AI project
├── rosbags/         # Raw ROS2 bag recordings
├── datasets/        # ML training datasets
└── backups/         # Automated backups
```

### Docker

Docker CE was installed from the official Docker repository, not the Ubuntu snap, which is not recommended for server use.

```bash
# Dependencies
sudo apt install ca-certificates curl gnupg -y

# GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Repository
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install
sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add user to docker group
sudo usermod -aG docker $USER
```

The Docker data root was relocated to the HDD to keep the SSD clean.

`/etc/docker/daemon.json`:

```json
{
  "data-root": "/mnt/data/docker"
}
```

Verified with `sudo docker info | grep "Docker Root Dir"`, which returned `/mnt/data/docker`.

### Self-hosted services

All services run as Docker containers. Compose files live in `~/services/<service>/docker-compose.yml`.

#### Gitea: self-hosted Git server

**Port:** 3000 · **Data:** `/mnt/data/gitea`

```yaml
version: "3"
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    ports:
      - "3000:3000"
      - "222:22"
    volumes:
      - /mnt/data/gitea:/data
```

Configured through the web UI on first run with a SQLite3 database. Project repos are created here first and mirrored to public GitHub when ready. The first repository was `Drone-hri-platform`.

#### Portainer CE: Docker management dashboard

**Port:** 9000 · **Purpose:** browser-based management of all containers

```bash
sudo docker volume create portainer_data
sudo docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

#### Mosquitto: MQTT broker

**Port:** 1883 · **Purpose:** message broker for ROS2 and drone communication in the HRI drone platform

`/mnt/data/mosquitto/config/mosquitto.conf`:

```
listener 1883
allow_anonymous true
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
```

Anonymous access is a LAN-only development setting; adding authentication is on the task list below.

```yaml
version: "3"
services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: always
    ports:
      - "1883:1883"
    volumes:
      - /mnt/data/mosquitto/config:/mosquitto/config
      - /mnt/data/mosquitto/data:/mosquitto/data
      - /mnt/data/mosquitto/log:/mosquitto/log
```

### SSH access

The server is managed headlessly over SSH from the T450; the monitor and keyboard were disconnected after setup.

T450 `~/.ssh/config`:

```
Host homeserver
    HostName <SERVER_IP>
    User <SERVER_USER>
```

Connect with `ssh homeserver`. The Termius mobile app was used during initial setup, before the T450 was configured.

### Development machine setup (T450)

#### ROS2 Humble

Installed from the official ROS2 apt repository, native on Ubuntu 22.04 (Jammy).

```bash
# Locale
sudo apt update && sudo apt install locales -y
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# Universe repo
sudo apt install software-properties-common -y
sudo add-apt-repository universe

# ROS2 GPG key
curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key | sudo gpg --dearmor -o /usr/share/keyrings/ros-archive-keyring.gpg

# ROS2 repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# Install full desktop
sudo apt update && sudo apt install ros-humble-desktop -y

# Auto-source on terminal open
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Verified with `ros2 doctor`: all 5 checks passed. A ROS2 workspace was then created at `~/ros2_ws` and built with `colcon build`.

One problem was found and fixed along the way: Anaconda's base environment was auto-activating and shadowing the system Python, which would have broken `rclpy` imports. It was resolved with:

```bash
conda config --set auto_activate_base false
```

#### Docker CE

Installed exactly as on the server and verified with `sudo docker run hello-world`.

#### VS Code

Installed from Microsoft's official instructions (version 1.123.0), with these extensions:
- Robotics Developer Environment (Microsoft), which replaces the deprecated ROS extension
- Python (Microsoft)
- Docker (Microsoft)
- Remote SSH (Microsoft)
- GitLens (GitKraken)

#### Git workflow: T450 to Gitea

```bash
# Clone from self-hosted Gitea
git clone http://<SERVER_IP>:3000/<user>/Drone-hri-platform.git

# Work locally, then push back to Gitea
git add .
git commit -m "feat: descriptive commit message"
git push origin main
```

---

## Part 2: Failure and Rebuild (October–November 2026)

### What failed

In October 2026 the ProDesk stopped powering on and showed HP's **3 red / 4 white** blink code, which indicates a power fault. With the server down, the one service that had to stay running was moved temporarily to the T450.

Swapping the PSU later cleared that code, which points to the original power supply as the root cause. The ProDesk then showed 3 red / 2 white (memory), which was expected because its RAM had already been moved to the new build. It is being kept for repair and resale.

### Why build instead of buying another SFF

The ProDesk's small-form-factor chassis had already limited the lab: a low-profile GPU bought for it did not physically fit, and there was no room for more drives. The replacement therefore had to be expandable in RAM, drives and GPU, and had to reuse the existing RAM, SSD and HDD.

### Parts and cost

| Part | Source | Price |
|---|---|---|
| AMD Ryzen 7 5700G (used, 5-year warranty) | CeX | £145.00 |
| Gigabyte B550M Aorus Elite | Amazon Resale | £73.79 |
| MSI MAG A650GL 650W Gold | Amazon | £56.19 |
| CiT Jet Stream case | Amazon | £40.19 |
| Arctic P12 PWM PST fans, 5-pack | Vinted | £16.45 |
| Arctic Alpine 23 CO cooler | Amazon | £9.99 |
| RAM, SSD, HDD | Reused from v1 | £0.00 |
| **Total** | | **£341.61** |

**Selection rationale:**
- **Ryzen 7 5700G over the 5700X or 5600:** 8 cores with integrated graphics, so the server works with no GPU and idles at lower power. The trade-offs are a smaller L3 cache, PCIe 3.0 and no ECC support, none of which matter for a container host.
- **B550 on AM4:** reuses the existing DDR4 and leaves a drop-in CPU upgrade path.
- **650W Gold PSU:** sized for a future discrete GPU, not for today's load. The case's bundled 500W unit was removed and kept as a spare.
- **Jet Stream case:** about twice the drive capacity of the similarly priced alternative, and room for a full-length GPU.
- **Alpine 23 CO:** the used CPU came without a cooler, and the continuous-operation variant suits a machine that runs all day.

### Assembly

1. **BIOS first.** The board was flashed with Q-Flash Plus from a FAT32 USB stick with no CPU or RAM fitted, to guarantee 5700G support. Final version: FHd (AGESA 1.2.0.12).
2. **Bench test on the motherboard box.** CPU, cooler and RAM were fitted and the board was powered outside the case by bridging the PWR_SW pins. The BIOS detected the 5700G, the RAM at 2666MHz, and an idle CPU temperature of 32–33°C.
3. **Fan check.** Each second-hand fan was tested on its own for RPM reporting, PWM response and bearing noise before being fitted.
4. **Into the case,** with the MSI PSU replacing the bundled unit and both drives moved across.

### BIOS configuration

| Setting | Value | Reason |
|---|---|---|
| AC power loss | Always On | Server restarts by itself after a power cut |
| SVM Mode | Enabled | Hardware virtualisation for VMs |
| Initial Display Output | IGD Video | Keeps the integrated graphics as the console |
| Integrated Graphics | Forced | Stays active once a discrete GPU is added |
| Smart Fan 6 | Quiet curve keyed to CPU temperature | Near-silent at idle; fan-fail warning on the CPU fan |

### Migrating Ubuntu to the new hardware

The SSD and HDD were moved across with no reinstall. Three things broke on first boot, each with a clear cause.

#### 1. Boot dropped into emergency mode

The drive names swapped on the new board:

| Drive | ProDesk | AM4 server |
|---|---|---|
| Samsung SSD (OS) | `sdb` | `sda` |
| Hitachi HDD (`/mnt/data`) | `sda` | `sdb` |

The original fstab entry mounted `/dev/sda` at `/mnt/data`, so Ubuntu tried to mount the OS disk as the data volume and failed. The fix was to mount by UUID and add `nofail`, so a missing data drive can never block boot again:

```bash
# Disable the old device-name entry
sed -i 's,^/dev/sda /mnt/data,#&,' /etc/fstab

# Add a UUID-based entry
echo "UUID=$(blkid -s UUID -o value /dev/sdb) /mnt/data ext4 defaults,nofail 0 2" >> /etc/fstab

systemctl daemon-reload
mount -a
```

#### 2. No network: the adapter was renamed

The NIC changed from `enp1s0` to `enp3s0`, so the static IP configuration no longer matched any device and boot stalled on `systemd-networkd-wait-online`. The Netplan file was updated with the new name, with `optional: true` added so a missing link cannot delay boot:

```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      optional: true
      dhcp4: no
      addresses:
        - <SERVER_IP>/24
      routes:
        - to: default
          via: <GATEWAY_IP>
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

A leftover cloud-init Netplan file conflicted with this configuration and was disabled.

#### 3. Kernel too old for the platform

The HWE kernel was installed for better support of the newer hardware:

```bash
sudo apt install linux-generic-hwe-22.04 amd64-microcode
```

The server now runs kernel 6.8.0-138.

#### Services

Because Docker's data root and every service volume live on `/mnt/data`, Gitea, Portainer and Mosquitto came back as soon as the data drive mounted, with the same static IP and ports.

One container needed extra work. Its saved login profile was copied back from the T450 with rsync, but files that Docker had created as root blocked the copy. Removing the root-owned files and copying a fresh profile fixed it.

### Lessons from the migration

- **Mount by UUID, never by device name.** Device names depend on the controller and port order.
- **Add `nofail` to non-root mounts** so a data drive problem degrades the system instead of stopping boot.
- **Expect predictable interface names to change with the hardware.** They encode the PCI slot.
- **Keeping all service state on one data volume paid off.** The whole stack survived a complete hardware change without a reinstall.
- **Watch file ownership on bind mounts.** Containers running as root leave root-owned files that later block user-level tools.

---

## Design Decisions Log

| Decision | Rationale |
|---|---|
| Ubuntu 22.04 LTS over 24.04 | Native platform for ROS2 Humble; consistent across both machines; large troubleshooting base |
| Docker CE over snap | The snap version has known limitations; CE gives full control over the data root |
| Docker data root on HDD | Keeps the SSD clean, and made the hardware migration almost stateless |
| LVM for the OS partition | Online resize without reinstall; 135GB of headroom expanded on day one |
| Gitea before GitHub | Private self-hosted repos during development; mirrored to public GitHub when ready |
| SQLite3 for Gitea | Enough for a single-user homelab, with no database to maintain |
| Static IP via Netplan | DHCP reassignment would break every service URL and the SSH config |
| Mosquitto via Docker | Consistent with the other services; config persisted to the HDD |
| Self-built AM4 tower over another SFF | Expandable RAM, drives and GPU; standard parts that can be replaced individually |
| APU instead of CPU plus display card | The server runs headless with no GPU; a GPU becomes an optional compute upgrade |
| UUID and `nofail` mounts | Device names changed with the hardware and stopped the system booting |
| Kept Ubuntu 22.04 and moved the disks | Preserved a working, documented configuration; the HWE kernel covers the newer platform |

---

## Next Steps

- [x] Set up ROS2 workspace on T450 (`~/ros2_ws`)
- [x] Replace the failed ProDesk with an expandable server
- [x] Publish this journal to public GitHub
- [ ] Add a second 16GB DDR4 stick for dual channel and 32GB
- [ ] Add a discrete GPU for CUDA work
- [ ] Configure Mosquitto authentication (remove anonymous access)
- [ ] Install Mosquitto client tools on T450 for MQTT testing
- [ ] Run memtest86+ and a stress-ng burn-in on the new build
- [ ] Evaluate Proxmox for VM isolation (Kubernetes and SLURM practice clusters)
- [ ] Begin drone project Phase 1: hardware teardown and electronics audit
- [ ] Mirror remaining Gitea repos to public GitHub
- [ ] Archive repo to Zenodo for a citable DOI

---

## References

- [Ubuntu Server documentation](https://ubuntu.com/server/docs)
- [Docker CE install for Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [ROS2 Humble install guide](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)
- [Gitea Docker setup](https://docs.gitea.com/installation/install-with-docker)
- [Eclipse Mosquitto Docker image](https://hub.docker.com/_/eclipse-mosquitto)
- [Portainer CE install](https://docs.portainer.io/start/install-ce/server/docker/linux)

---

*This journal is a living document, updated as the homelab evolves.*  
*Part of Peter Antai's robotics research portfolio.*
