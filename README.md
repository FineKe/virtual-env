## Virtual Development Environment (Vagrant + VirtualBox)

### Overview
This repository uses Vagrant to spin up a Ubuntu 22.04 guest with a ready-to-use development toolchain. On the first run it installs Docker, Go, JDK, C/C++ toolchain, Rust, and configures DNS.

### Prerequisites
- Vagrant (2.3+ recommended)
- VirtualBox (7.2+ recommended)
- Internet access on the host (APT needs to fetch packages)

### Synced Folders (Host ↔ Guest)
- Host folder `./workspace` is mounted to guest path `/vagrant/workspace`.
  - Put your code under `./workspace` on the host; access it at `/vagrant/workspace` inside the VM.
  - You can adjust `config.vm.synced_folder` in the `Vagrantfile` as needed.

### VM Resources
- Provider: VirtualBox
- Box: `bento/ubuntu-22.04` (ARM64)
- CPUs: 14 (configurable in `Vagrantfile`)
- Memory: 40960 MB (configurable in `Vagrantfile`)

### Quick Start
```bash
vagrant up --provision
```
The first run triggers the provisioning script and may take a while depending on your network.

### Installed Tools & Configuration
- Docker (`docker.io`), and the `vagrant` user is added to the `docker` group
- Base tools: `curl`, `git`, `make`, `build-essential`, `jq`, `net-tools`, etc.
- Go: Ubuntu package (`golang-go`) with `GOPATH=$HOME/go`
- JDK: OpenJDK 11 (`default-jdk`)
- C/C++ and common deps: `clang`/`llvm`, `cmake`, `ninja-build`, `gdb`, `pkg-config`, `libssl-dev`, `ccache`, etc.
- Rust: Installed via `rustup` (stable), PATH exported for login shells
- DNS: Configured via `systemd-resolved` (1.1.1.1, 8.8.8.8; fallback 9.9.9.9, 8.8.4.4)

### Common Commands
```bash
# SSH into the VM
vagrant ssh

# Stop / start / reload / re-provision
vagrant halt
vagrant up
vagrant reload --provision
vagrant provision

# Destroy and recreate (erases guest data)
vagrant destroy -f && vagrant up --provision
```

### Verify Installation
Inside the guest (`vagrant ssh`):
```bash
docker --version
groups         # ensure 'vagrant' is in the 'docker' group; if not, re-login or run `newgrp docker`

go version && echo $GOPATH && which go
java -version && javac -version
clang --version && cmake --version && ninja --version && gdb --version
pkg-config --version && ccache --version
source ~/.cargo/env && rustc --version && cargo --version
```

### Adjust Tool Versions
- Go: Currently from Ubuntu repos (e.g., 1.18). If you need a specific version (e.g., 1.22.x), we can switch to the official tarball installation in the `Vagrantfile`.
- JDK: Replace `default-jdk` with a specific version (e.g., `openjdk-17-jdk`).
- Rust: Use `rustup` to switch between `stable`/`beta`/`nightly` or pin a specific toolchain.

### Apple Silicon Notes
- This project uses an ARM64 guest box (`bento/ubuntu-22.04`) which runs on Apple Silicon with VirtualBox.
- Avoid x86_64-only boxes on ARM hosts; they will fail due to architecture mismatch.

### Troubleshooting
- Docker permissions: If you see permission errors, run `newgrp docker` or re-login (`exit` then `vagrant ssh`).
- DNS resolution:
  ```bash
  resolvectl status | grep 'DNS Servers' -A1 || cat /etc/resolv.conf
  ```
- Resource pressure: If the host is short on RAM/CPU, lower `vb.memory` and `vb.cpus` in `Vagrantfile`.
- Missing mount: Ensure `./workspace` exists on the host; if needed, `vagrant reload`.

### Conventions
- Host: Put your project code under `./workspace`
- Guest: Access/build/run from `/vagrant/workspace`


