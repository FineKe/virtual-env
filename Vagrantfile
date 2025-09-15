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
  config.vm.synced_folder "./workspace", "/vagrant/workspace", disabled: false

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
