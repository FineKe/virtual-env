# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.box_version = "202508.03.0"
  config.vm.provider :virtualbox
  config.vm.disk :disk, size: "100GB", primary: true
  config.vm.disk :disk, size: "100GB", name: "workspace-disk"


  # Set VirtualBox disk size to 100GB (requires plugin: vagrant-disksize)
  # if Vagrant.has_plugin?("vagrant-disksize")
  #   config.disksize.size = "100GB"
  # else
  #   warn "[vagrant-disksize] plugin not installed. Run: vagrant plugin install vagrant-disksize"
  # end

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "./workspace", "/vagrant/workspace"

  # Disable the default share of the current code directory. Doing this
  # provides improved isolation between the vagrant box and your host
  # by making sure your Vagrantfile isn't accessible to the vagrant box.
  # If you use this you may want to enable additional shared subfolders as
  # shown above.
  # config.vm.synced_folder "./workspace", "/vagrant/workspace", disabled: false

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    vb.gui = false
    vb.cpus = 14

    # Customize the amount of memory on the VM:
    vb.memory = "40960"
  end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: <<-SHELL
    set -euo pipefail
    export DEBIAN_FRONTEND=noninteractive

    echo "[1/5] Updating package index..."
    apt-get update -y

    echo "[2/7] Installing base tools (curl, ca-certificates, gnupg, lsb-release, make, build-essential, git, jq, net-tools)..."
    apt-get install -y \
      curl ca-certificates gnupg lsb-release \
      make build-essential git jq net-tools apt-transport-https software-properties-common

    echo "[3/7] Installing JDK and extra C/C++ tools..."
    # JDK (default-jdk maps to supported OpenJDK on Ubuntu 22.04)
    apt-get install -y default-jdk
    # Extra C/C++ toolchain utilities frequently used in projects
    apt-get install -y cmake ninja-build clang gdb pkg-config libssl-dev ccache

    echo "[4/7] Installing Docker (if missing)..."
    if ! command -v docker >/dev/null 2>&1; then
      # Prefer Ubuntu docker.io for simplicity and reliability in VMs
      apt-get install -y docker.io
      systemctl enable --now docker
    else
      echo "Docker already installed: $(docker --version)"
    fi

    # Add vagrant user to docker group (idempotent)
    if getent passwd vagrant >/dev/null 2>&1; then
      if ! id -nG vagrant | grep -qw docker; then
        usermod -aG docker vagrant || true
        echo "Added user 'vagrant' to docker group (relogin required)."
      fi
    fi

    echo "[5b] Preparing and mounting secondary workspace disk (/home/vagrant/workspace)..."
    # Ensure required tools are present
    apt-get install -y parted e2fsprogs
    DISK="/dev/sdb"
    PART="${DISK}1"
    MNT="/home/vagrant/workspace"
    if [ -b "$DISK" ]; then
      # Partition if not already partitioned
      if ! lsblk -no NAME "$PART" >/dev/null 2>&1; then
        parted -s "$DISK" mklabel gpt || true
        parted -s "$DISK" mkpart primary ext4 0% 100% || true
        # Wait for the kernel to recognize the new partition
        udevadm settle || true
      fi
      # Make filesystem if missing
      if [ -b "$PART" ] && [ -z "$(lsblk -no FSTYPE "$PART")" ]; then
        mkfs.ext4 -F "$PART"
      fi
      # Mount point and fstab
      mkdir -p "$MNT"
      if [ -b "$PART" ]; then
        UUID=$(blkid -s UUID -o value "$PART")
        if [ -n "$UUID" ] && ! grep -q "$UUID" /etc/fstab; then
          echo "UUID=$UUID $MNT ext4 defaults,nofail 0 2" >> /etc/fstab
        fi
        mount -a || true
        chown -R vagrant:vagrant "$MNT" || true
      fi
    fi

    echo "[5c] Configuring Docker data-root to $MNT/docker ..."
    apt-get install -y rsync || true
    mkdir -p /etc/docker
    mkdir -p "$MNT/docker"
    # If old data exists and target is empty, migrate once
    if [ -d /var/lib/docker ] && [ -z "$(ls -A "$MNT/docker" 2>/dev/null)" ]; then
      systemctl stop docker || true
      rsync -a /var/lib/docker/ "$MNT/docker/" || true
    else
      systemctl stop docker || true
    fi
    printf '%s\n' \
      '{' \
      '  "data-root": "/home/vagrant/workspace/docker"' \
      '}' \
      > /etc/docker/daemon.json
    chown -R root:root "$MNT/docker"
    systemctl daemon-reload || true
    systemctl enable --now docker || true

    # Ensure OpenSSH server is installed and enabled at boot
    apt-get install -y openssh-server
    systemctl enable --now ssh || true

    # Auto-start ssh-agent for login shells (idempotent, per-user)
    printf '%s\n' \
      'if [ -z "$SSH_AUTH_SOCK" ]; then' \
      '  AGENT_ENV="$HOME/.ssh/agent.env"' \
      '  if [ -r "$AGENT_ENV" ]; then . "$AGENT_ENV" >/dev/null 2>&1; fi' \
      '  if ! kill -0 "$SSH_AGENT_PID" >/dev/null 2>&1; then' \
      '    eval "$(ssh-agent -s)" >/dev/null' \
      '    mkdir -p "$HOME/.ssh"' \
      '    umask 077' \
      '    echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK; export SSH_AUTH_SOCK;" > "$AGENT_ENV"' \
      '    echo "SSH_AGENT_PID=$SSH_AGENT_PID; export SSH_AGENT_PID;" >> "$AGENT_ENV"' \
      '  fi' \
      'fi' \
      > /etc/profile.d/10-ssh-agent.sh
    chmod 0644 /etc/profile.d/10-ssh-agent.sh

    echo "[5a/7] Installing zsh and oh-my-zsh for vagrant..."
    apt-get install -y zsh
    # Install oh-my-zsh for vagrant user (unattended)
    su - vagrant -c 'if [ ! -d "$HOME/.oh-my-zsh" ]; then \
        export RUNZSH=no CHSH=no KEEP_ZSHRC=yes; \
        sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"; \
      fi'
    # Ensure .zshrc exists and set basic plugins
    su - vagrant -c 'if [ ! -f "$HOME/.zshrc" ] && [ -d "$HOME/.oh-my-zsh" ]; then \
        cp "$HOME/.oh-my-zsh/templates/zshrc.zsh-template" "$HOME/.zshrc"; \
      fi'
    su - vagrant -c 'if [ -f "$HOME/.zshrc" ]; then \
        sed -i "s/^plugins=(.*/plugins=(git docker)/" "$HOME/.zshrc" || true; \
        if ! grep -q "99-golang.sh" "$HOME/.zshrc"; then \
          printf "\n# env from provisioning\n[ -f /etc/profile.d/99-golang.sh ] && source /etc/profile.d/99-golang.sh\n[ -f /etc/profile.d/99-rust.sh ] && source /etc/profile.d/99-rust.sh\n" >> "$HOME/.zshrc"; \
        fi; \
      fi'
    # Set default shell to zsh for vagrant
    if [ "$(getent passwd vagrant | cut -d: -f7)" != "/usr/bin/zsh" ]; then
      chsh -s /usr/bin/zsh vagrant || usermod -s /usr/bin/zsh vagrant || true
    fi

    echo "[5/7] Configuring DNS via systemd-resolved..."
    mkdir -p /etc/systemd/resolved.conf.d
    printf '%s\n' \
      '[Resolve]' \
      'DNS=1.1.1.1 8.8.8.8' \
      'FallbackDNS=9.9.9.9 8.8.4.4' \
      'DNSSEC=no' \
      > /etc/systemd/resolved.conf.d/99-custom.conf

    # Ensure resolv.conf points to systemd-resolved stub
    if [ -e /run/systemd/resolve/stub-resolv.conf ]; then
      ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf || true
    elif [ -e /run/systemd/resolve/resolv.conf ]; then
      ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf || true
    fi

    systemctl restart systemd-resolved || true

    echo "[6/7] Installing Go toolchain and setting GOPATH..."
    if ! command -v go >/dev/null 2>&1; then
      apt-get install -y golang-go
    else
      echo "Go already installed: $(go version)"
    fi

    # Configure GOPATH for all users via profile.d
    printf '%s\n' \
      'export GOPATH="$HOME/go"' \
      'export PATH="$PATH:$GOPATH/bin"' \
      > /etc/profile.d/99-golang.sh
    chmod 0644 /etc/profile.d/99-golang.sh

    # Ensure GOPATH directory exists for vagrant and root
    mkdir -p /home/vagrant/go || true
    chown -R vagrant:vagrant /home/vagrant/go || true
    mkdir -p /root/go || true

    # Print versions for log
    go version || true

    echo "[7/7] Installing Rust toolchain (rustup, stable) for vagrant user..."
    if ! su - vagrant -c 'command -v cargo >/dev/null 2>&1'; then
      su - vagrant -c 'curl -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable'
    else
      su - vagrant -c 'rustup self update || true; rustup update stable || true'
    fi

    # Make Cargo available for all login shells
    printf '%s\n' \
      'export PATH="$HOME/.cargo/bin:$PATH"' \
      > /etc/profile.d/99-rust.sh
    chmod 0644 /etc/profile.d/99-rust.sh

    # Print Rust versions for log
    su - vagrant -c 'source $HOME/.cargo/env && rustc --version && cargo --version' || true

    echo "Provisioning complete. If 'vagrant' was added to docker group, please 'vagrant ssh' again."
  SHELL
end
