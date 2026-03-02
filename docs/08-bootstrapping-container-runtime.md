# Bootstrapping Container Runtime

**Note:** The following commands must be run on each Kubernetes machine (both controllers and workers).

In this section, we will bootstrap the container runtime. We will be using `containerd`, which is an industry-standard container runtime with an emphasis on simplicity, robustness, and portability.

While Debian provides a `containerd` package, it is often too old for Kubernetes requirements. Therefore, we will use the official packages from the Docker project.

### Prerequisites

Configure the required kernel modules and sysctl parameters. These settings persist across reboots.

1.  Load the `overlay` and `br_netfilter` modules:

    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/kubernetes-cri.conf
    br_netfilter
    overlay
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

2.  Configure the required sysctl parameters for the container runtime:

    ```bash
    # Sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    ```

3.  Configure additional system parameters for Kubernetes stability:

    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
    kernel.panic=10
    kernel.panic_on_oops=1
    vm.overcommit_memory=1
    vm.panic_on_oom=0
    vm.max_map_count=262144
    EOF
    
    # Apply all sysctl params without reboot
    sudo sysctl --system
    ```

### Install Containerd

1.  Add Docker's official GPG key:

    ```bash
    sudo apt update
    sudo apt install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

2.  Add the repository to Apt sources:

    ```bash
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/debian
    Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF

    sudo apt update
    ```

3.  Install `containerd.io`:

    ```bash
    sudo apt-get install -y containerd.io
    ```

### Install CNI Plugins

Install the `containernetworking-plugins` package to provide the standard CNI plugins (bridge, loopback, host-local, etc.):

```bash
sudo apt-get install -y containernetworking-plugins
```

Also, disable the CNI DHCP service as it is not needed:

```bash
sudo systemctl disable --now cni-dhcp.socket cni-dhcp.service
```

### Configure Containerd

1.  Generate the default configuration:

    ```bash
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    ```

2.  Configure the systemd cgroup driver.

    To use the `systemd` cgroup driver, you need to set `SystemdCgroup = true` in `/etc/containerd/config.toml`.

    You can use `sed` to update the configuration:

    ```bash
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    ```

    Alternatively, verify that the following setting is configured in `/etc/containerd/config.toml`:

    ```toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      # ...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
    ```

3.  Restart containerd:

    ```bash
    sudo systemctl restart containerd
    sudo systemctl enable --now containerd
    ```
