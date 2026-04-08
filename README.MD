# Rocky Linux 9 — Fresh Install Configuration Guide

This guide walks you through configuring a freshly installed Rocky Linux 9 server with essential development tools, Python build dependencies, PostgreSQL client libraries, and NVIDIA GPU drivers.

> **Important:** All commands in this guide should be run as root. Start by switching to the root user:

```bash
sudo su
```

---

## 1. Configure DNF (Package Manager)

Open the DNF configuration file and add performance tweaks:

```bash
# Open the DNF config file for editing
vi /etc/dnf/dnf.conf
```

Append the following lines under `[main]`:

```ini
# Use the fastest available mirror
fastestmirror=True
# Download up to 10 packages in parallel
max_parallel_downloads=10
```

---

## 2. Update the System

```bash
# Pull in all available updates
dnf update -y

# Reboot to apply kernel and system updates
reboot
```

After reboot, switch back to root:

```bash
sudo su
```

---

## 3. Install Essential Tools

```bash
# Install basic CLI and build tools
dnf install -y tar vim nano micro cmake zsh fish zip unzip tmux

# Install monitoring, debugging, and network diagnostic utilities
dnf install -y \
  htop \
  btop \
  iotop \
  strace \
  lsof \
  tcpdump \
  nmap-ncat \
  util-linux-user \
  ncdu \
  gdu
```

---

## 4. Install Python Build Dependencies

These libraries are required for building Python from source (e.g., via pyenv):

```bash
# Install development headers and libraries needed to compile CPython
dnf install -y \
  python3-devel \
  openssl-devel \
  libffi-devel \
  zlib-devel \
  readline-devel \
  sqlite-devel \
  bzip2-devel \
  ncurses-devel \
  xz-devel \
  libuuid-devel \
  gdbm-devel \
  libpq-devel
```

---

## 5. Change the host machine name
```bash
# Change the hostname (replace "new-hostname" with your desired hostname)
hostnamectl set-hostname GPU24
```

---

## 6. Housekeeping — Open All TCP Ports in Firewall

```bash
# Allow all TCP ports (0-65535) through the firewall permanently
sudo firewall-cmd --permanent --add-port=0-65535/tcp

# Reload the firewall to apply the changes
sudo firewall-cmd --reload
```

---

## 6. Install NVIDIA GPU Drivers (DKMS Method)

### 6.1 Install Kernel Development Packages

```bash
# Install kernel-devel and DKMS (Dynamic Kernel Module Support)
dnf install -y kernel-devel dkms
```

### 6.2 Add the NVIDIA CUDA Repository

```bash
# Add the official NVIDIA CUDA repo for RHEL 9 (auto-detects architecture)
dnf config-manager --add-repo \
  http://developer.download.nvidia.com/compute/cuda/repos/rhel9/$(uname -i)/cuda-rhel9.repo
```

### 6.3 Install Kernel Headers and Build Dependencies

```bash
# Install the exact kernel headers for the running kernel, plus all
# build tools and libraries required by the NVIDIA driver installer
dnf install -y \
  kernel-headers-$(uname -r) \
  kernel-devel-$(uname -r) \
  tar bzip2 make automake \
  gcc gcc-c++ \
  pciutils \
  elfutils-libelf-devel \
  libglvnd-opengl libglvnd-glx libglvnd-devel \
  acpid pkgconf dkms
```

### 6.4 Install the NVIDIA Driver

```bash
# Install the latest NVIDIA driver built with DKMS so it survives kernel upgrades
dnf module install nvidia-driver:latest-dkms -y
```

### 6.5 Version-Lock NVIDIA, CUDA, and Kernel Packages

Locking versions prevents accidental upgrades that could break driver compatibility:

```bash
# Install the versionlock plugin for DNF
dnf install -y 'dnf-command(versionlock)'

# Lock all NVIDIA and CUDA packages at their current versions
dnf versionlock add *nvidia*
dnf versionlock add *cuda*

# Lock the current kernel version to prevent driver/kernel mismatch
dnf versionlock add kernel-$(uname -r)

# Verify the locked packages
dnf versionlock list
```

### 6.6 Reboot

```bash
# Reboot to load the NVIDIA kernel module
reboot
```

After reboot, verify the driver is loaded:

```bash
# Should display your GPU info and driver version
nvidia-smi
```

---

## 7. Enable RPM Fusion Repositories

[RPM Fusion](https://rpmfusion.org/) provides packages that Fedora and RHEL choose not to ship (codecs, proprietary drivers, etc.). Both the **free** and **nonfree** repos are useful on GPU workstations.

```bash
# Install EPEL first — it is a required dependency for RPM Fusion
dnf install -y epel-release

# Install RPM Fusion Free and Nonfree repositories
dnf install -y --nogpgcheck \
  https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-$(rpm -E %rhel).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-$(rpm -E %rhel).noarch.rpm
```

> **Note:** The `--nogpgcheck` flag is only needed for the initial bootstrap install. Once the RPM Fusion repo is added, all subsequent packages from it will be GPG-verified normally.

---

## 8. Install FFmpeg, GStreamer, and Multimedia Libraries

Now that RPM Fusion is enabled, install full multimedia support. Rocky ships a limited `ffmpeg-free` — swap it for the complete RPM Fusion build:

```bash
# Replace the limited ffmpeg-free with the full FFmpeg from RPM Fusion
dnf swap ffmpeg-free ffmpeg --allowerasing -y

# Install FFmpeg development headers (needed to build against libav*)
dnf install -y ffmpeg-devel
```

Install GStreamer plugins for broad codec and container format support:

```bash
# Install GStreamer runtime plugins
dnf install -y \
  gstreamer1-plugins-base \
  gstreamer1-plugins-good \
  gstreamer1-plugins-bad-free \
  gstreamer1-plugins-ugly \
  gstreamer1-libav

# Install GStreamer development headers (needed to build GStreamer-based apps)
dnf install -y \
  gstreamer1-devel \
  gstreamer1-plugins-base-devel \
  gstreamer1-plugins-bad-free-devel

# Install pkg-config helper (needed by build systems to locate .pc files)
dnf install -y pkgconf-pkg-config
```

Verify the installation:

```bash
# Confirm FFmpeg has libx264 codec support
ffmpeg -codecs | grep libx264

# Confirm GStreamer can find the x264 encoder element
gst-inspect-1.0 x264
```

---

## 9. Install NVIDIA Fabric Manager (Multi-GPU NVLink Systems)

> **When is this needed?** If you are running a multi-GPU system with **NVSwitch** (e.g., HGX A100, H100, or B200). Without the Fabric Manager, GPUs cannot communicate over the high-speed NVLink fabric and `nvidia-smi` may show them as "down" or in an error state.

### 9.1 Ensure the NVIDIA Repository is Active

If you followed section 6.2, this is already done. Verify with:

```bash
# Confirm the CUDA repo is present
dnf repolist | grep cuda
```

### 9.2 Install the Matching Fabric Manager Version

The Fabric Manager version **must match your installed driver version exactly**. A mismatch — even a minor version difference — will cause the service to fail.

```bash
# Check your current NVIDIA driver version
nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -n 1

# Install the Fabric Manager at the EXACT same version (replace with your version)
# Example: if driver is 595.58.03
dnf install -y nvidia-fabric-manager-xxx.yy.zz

# If that doesn't work, try:
dnf module install nvidia-driver/fm

# Finally, enable version-locking for the Fabric Manager to prevent future mismatches:
dnf versionlock add nvidia-fabric-manager*

# Enable fabric manager and persistenced with systemctl
systemctl enable --now nvidia-fabricmanager 
systemctl enable --now nvidia-persistenced
```

> **Warning:** Do not omit the version number. Running `dnf install nvidia-fabric-manager` without a version will pull the latest, which may not match your locked driver.

### 9.3 Start and Enable the Fabric Manager Service

```bash
# Enable and start the Fabric Manager daemon
systemctl enable --now nvidia-fabricmanager
```

### 9.4 Verify the Service is Running

```bash
# Confirm the Fabric Manager is active — look for "active (running)"
systemctl status nvidia-fabricmanager
```

### 9.5 Version-Lock the Fabric Manager

Since the driver and kernel are already locked (section 6.5), lock the Fabric Manager too. If the driver stays at version `550` but the Fabric Manager updates to `560`, GPU interconnects will break.

```bash
# Lock the Fabric Manager at the current version
dnf versionlock add nvidia-fabric-manager*
```

### 9.6 Troubleshooting

If the Fabric Manager fails to start, check the system journal:

```bash
# View Fabric Manager logs
journalctl -u nvidia-fabricmanager
```

**Common fixes:**
- **Version mismatch:** Re-check driver vs. Fabric Manager versions — they must be identical.
- **Zombie NVSwitch state:** A `reboot` is sometimes required after installation to reset the NVSwitch hardware state.
- **CUDA modular packages:** If using RHEL-style modular packages, ensure `cuda-drivers-fabricmanager` is also installed.

---

## Lock The NVIDIA Driver, CUDA, Fabric Manager, and Kernel Versions to prevent Accidental Upgrades and Crashes

If you ever need to remove a version lock (e.g., to upgrade the driver):

```bash
# Remove all NVIDIA version locks (includes Fabric Manager)
dnf versionlock delete *nvidia*

# Remove all CUDA version locks
dnf versionlock delete *cuda*

# Remove the kernel version lock
dnf versionlock delete kernel-*

# Confirm locks were removed
dnf versionlock list
```

---

## 10. Install Docker

### 10.1 Add the Docker Repository

```bash
# Add the official Docker CE repo for RHEL-based distros
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

### 10.2 Install Docker Packages

```bash
# Install Docker Engine, CLI, container runtime, Buildx, and Compose plugin
dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### 10.3 Start and Enable Docker

```bash
# Enable Docker to start on boot and start it immediately
systemctl --now enable docker
```

### 10.4 Allow Non-Root Users to Manage Docker

```bash
# Add a specific user to the docker group so they can run docker without sudo
usermod -a -G docker <username>
```

> **Note:** The user must log out and back in for the new group membership to take effect. Verify with the `id` command.

---

## 11. Install NVIDIA Container Toolkit

This enables Docker containers to access NVIDIA GPUs via `--gpus` flags.

### 11.1 Add the NVIDIA Container Toolkit Repository

```bash
# Add the official NVIDIA container toolkit repo
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  tee /etc/yum.repos.d/nvidia-container-toolkit.repo
```

### 11.2 Install the Toolkit (Version-Pinned)

Pin to a specific version to avoid mismatches with your driver:

```bash
# Set the desired toolkit version
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.0-1

# Install all NVIDIA container toolkit components at the pinned version
dnf install -y \
  nvidia-container-toolkit-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  nvidia-container-toolkit-base-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container-tools-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container1-${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

### 11.3 Configure Docker to Use the NVIDIA Runtime

```bash
# Register the NVIDIA runtime with Docker's daemon.json
nvidia-ctk runtime configure --runtime=docker

# Restart Docker to pick up the new runtime configuration
systemctl restart docker
```

Verify GPU access inside a container:

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubi9 nvidia-smi
```

---

## 12. Install Zsh Antigen Plugin Manager

[Antigen](https://github.com/zsh-users/antigen) is a plugin manager for Zsh that makes it easy to load bundles and themes.

```bash
# Create the antigen directory
mkdir -p /usr/share/zsh-antigen/

# Download the antigen script
curl -L git.io/antigen > /usr/share/zsh-antigen/antigen.zsh

# Verify the download
ls /usr/share/zsh-antigen/antigen.zsh
```

> **Next step:** Add `source /usr/share/zsh-antigen/antigen.zsh` to your `~/.zshrc` and configure your desired plugins and theme.
