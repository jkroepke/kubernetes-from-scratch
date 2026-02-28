# 01 Prerequisites

In this lab, the machine requirements necessary to follow this tutorial will be reviewed.

## Virtual or Physical Machines
This tutorial requires four (4) virtual or physical ARM64 or AMD64 machines running Debian 13 (trixe).
The following table lists the four machines and their CPU, memory and storage requirements.

| Name        | Description                  | CPU | RAM   | Storage |
|-------------|------------------------------|-----|-------|---------|
| controller1 | Kubernetes controller        | 1   | 2GB   | 20GB    |
| controller2 | Kubernetes controller        | 1   | 2GB   | 20GB    |
| controller3 | Kubernetes controller        | 1   | 2GB   | 20GB    |
| worker1     | Kubernetes controller        | 1   | 2GB   | 20GB    |
| jumpbox     | optional administration host | 1   | 512MB | 10GB    |

Provisioning the machines is flexible; the only requirement is that each machine meet the above system requirements including the machine specs and OS version.

## Load Balancer

A TCP load balancer is required to route traffic to the Kubernetes API server and the ingress controller. The load balancer must listen on ports 80, 443, and 6443.

The load balancer can run on the jumpbox.

| Port | Backend | Target Port | Description |
|------|---------|-------------|-------------|
| 6443 | controller1, controller2, controller3 | 6443 | Kubernetes API Server |
| 80   | worker1 | 80 | HTTP Ingress |
| 443  | worker1 | 443 | HTTPS Ingress |

> [!TIP]
> A preconfigured `docker-compose` setup for a Traefik-based load balancer can be found in the `docker/lb/` directory.

>[!NOTE]
> The jumpbox is optional, but it can be helpful to have a separate machine to run administrative commands from.

>[!IMPORTANT]
> All machines must be reachable from each other over the network. If virtual machines are being used, ensure that they are all on the same network and can ping each other.

>[!IMPORTANT]
> The names of the machines (controller1, controller2, controller3, worker1 and jumpbox) should be resolvable to their respective IP addresses.

Once all four machines are provisioned, verify the OS requirements by viewing the `/etc/os-release` file:

```bash
cat /etc/os-release
```

The output should resemble the following:

```
PRETTY_NAME="Debian GNU/Linux 13 (trixe)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixe)"
VERSION_CODENAME=trixe
ID=debian
```
