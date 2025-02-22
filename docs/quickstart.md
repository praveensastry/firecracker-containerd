# Quickstart with firecracker-containerd

This quickstart guide provides simple steps to get a working
firecracker-containerd environment, with each of the major components built from
source.  Once you have completed this quickstart, you should be able to run and
develop firecracker-containerd (the components in this repository), the
Firecracker VMM, and containerd.

This quickstart will clone repositories under your `$HOME` directory and install
files into `/usr/local/bin`.

1. Get an AWS account (see
   [this article](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
   if you need help creating one)
2. Launch an i3.metal instance running Debian Stretch (you can find it in the
   [AWS marketplace](http://deb.li/awsmp) or on [this
   page](https://wiki.debian.org/Cloud/AmazonEC2Image/Stretch).  If you need
   help launching an EC2 instance, see the
   [EC2 getting started guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html).
3. Run the script below to download and install all the required components.
   This script expects to be run from your `$HOME` directory.

```bash
#!/bin/bash

cd ~

# Install git, Go 1.11, make, curl
sudo mkdir -p /etc/apt/sources.list.d
echo "deb http://ftp.debian.org/debian stretch-backports main" | \
     sudo tee /etc/apt/sources.list.d/stretch-backports.list
sudo DEBIAN_FRONTEND=noninteractive apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get \
  --target-release stretch-backports \
  install --yes \
  golang-go \
  make \
  git \
  curl \
  e2fsprogs \
  musl-tools \
  util-linux

# Install Rust
curl https://sh.rustup.rs -sSf | sh -s -- --verbose -y --default-toolchain 1.32.0
source $HOME/.cargo/env
rustup target add x86_64-unknown-linux-musl

# Check out Firecracker and build it from the v0.17.0 tag
git clone https://github.com/firecracker-microvm/firecracker.git
cd firecracker
git checkout v0.17.0
cargo build --release --features vsock --target x86_64-unknown-linux-musl
sudo cp target/x86_64-unknown-linux-musl/release/{firecracker,jailer} /usr/local/bin

cd ~

# Install Docker CE
# Docker CE includes containerd, but we need a separate containerd binary, built
# in a later step
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
apt-key finger docker@docker.com | grep '9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88' || echo '**Cannot find Docker key**'
echo "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list
sudo DEBIAN_FRONTEND=noninteractive apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get \
     install --yes \
     docker-ce aufs-tools-
sudo usermod -aG docker $(whoami)
exec newgrp docker

cd ~

# Check out firecracker-containerd and build it.  This includes:
# * block-device snapshotter gRPC proxy plugins
# * firecracker-containerd runtime, a containerd v2 runtime
# * firecracker-containerd agent, an inside-VM component
# * runc, to run containers inside the VM
# * a Debian-based root filesystem configured as read-only with a read-write
#   overlay
# * firecracker-containerd, an alternative containerd binary that includes the
#   firecracker VM lifecycle plugin and API
git clone https://github.com/firecracker-microvm/firecracker-containerd.git
cd firecracker-containerd
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y dmsetup
make all image
sudo make install

cd ~

# Download kernel
curl -fsSL -o hello-vmlinux.bin https://s3.amazonaws.com/spec.ccfc.min/img/hello/kernel/hello-vmlinux.bin

# Configure our firecracker-containerd binary to use our new snapshotter and
# separate storage from the default containerd binary
sudo mkdir -p /etc/firecracker-containerd
sudo mkdir -p /var/lib/firecracker-containerd/containerd
sudo mkdir -p /run/firecracker-containerd
sudo tee /etc/firecracker-containerd/config.toml <<EOF
disabled_plugins = ["cri"]
root = "/var/lib/firecracker-containerd/containerd"
state = "/run/firecracker-containerd"
[grpc]
  address = "/run/firecracker-containerd/containerd.sock"
[proxy_plugins]
  [proxy_plugins.firecracker-naive]
    type = "snapshot"
    address = "/var/run/firecracker-containerd/naive-snapshotter.sock"

[debug]
  level = "debug"
EOF

cd ~

# Configure the aws.firecracker runtime
# The long kernel command-line configures systemd inside the Debian-based image
# and uses a special init process to create a read-write overlay on top of the
# read-only image.
sudo mkdir -p /var/lib/firecracker-containerd/runtime
sudo cp ~/firecracker-containerd/tools/image-builder/rootfs.img /var/lib/firecracker-containerd/runtime/default-rootfs.img
sudo cp ~/hello-vmlinux.bin /var/lib/firecracker-containerd/runtime/default-vmlinux.bin
sudo mkdir -p /etc/containerd
sudo tee /etc/containerd/firecracker-runtime.json <<EOF
{
  "firecracker_binary_path": "/usr/local/bin/firecracker",
  "cpu_template": "T2",
  "log_fifo": "fc-logs.fifo",
  "log_level": "Debug",
  "metrics_fifo": "fc-metrics.fifo",
  "kernel_args": "console=ttyS0 noapic reboot=k panic=1 pci=off nomodules ro systemd.journald.forward_to_console systemd.unit=firecracker.target init=/sbin/overlay-init"
}
EOF

# Enable vhost-vsock
sudo modprobe vhost-vsock
```

4. Open a new terminal and start the `naive_snapshotter` program in the
   foreground

```bash
sudo mkdir -p /var/run/firecracker-containerd /var/lib/firecracker-containerd/naive
sudo naive_snapshotter \
     -address /var/run/firecracker-containerd/naive-snapshotter.sock \
     -path /var/lib/firecracker-containerd/naive \
     -debug
```

5. Open a new terminal and start `firecracker-containerd` in the foreground

```bash
sudo firecracker-containerd --config /etc/firecracker-containerd/config.toml
```

6. Open a new terminal, pull an image, and run a container!

```bash
sudo firecracker-ctr --address /run/firecracker-containerd/containerd.sock \
     image pull \
     --snapshotter firecracker-naive \
     docker.io/library/debian:latest
sudo firecracker-ctr --address /run/firecracker-containerd/containerd.sock \
     run \
     --snapshotter firecracker-naive \
     --runtime aws.firecracker \
     --tty \
     docker.io/library/debian:latest \
     test
```

In the commands above, note the `--address` argument targeting the
`firecracker-containerd` binary instead of the normal `containerd` binary, the
`--snapshotter` argument targeting the block-device snapshotter, and the
`--runtime` argument targeting the firecracker-containerd runtime.

When you're done, you can stop or terminate your i3.metal EC2 instance to avoid
incurring additional charges from EC2.
