# Bootstrapping Kubelet

One of the key functions of the Kubelet is to manage **static pods**.

### Static Pods

Static Pods are managed directly by the kubelet Daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment), static Pods are bound to the Kubelet on a specific node.

The Kubelet watches a directory defined in its configuration (usually `/etc/kubernetes/manifests`) for Pod manifests and creates/deletes the Pods as files appear or disappear in that directory.

### Installing Kubelet

To install `kubelet`, run the following commands. These instructions are for Kubernetes v1.35.

1. Update the apt package index and install packages needed to use the Kubernetes apt repository:

    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gpg
    ```

2. Download the public signing key for the Kubernetes package repositories:

    ```bash
    # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command.
    sudo mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

3. Add the appropriate Kubernetes apt repository:

    ```bash
    # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

    ```bash
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

5. (Optional) Enable the kubelet service:

    ```bash
    sudo systemctl enable --now kubelet
    ```

### Kubelet Configuration

The Kubelet configuration file is commonly placed at `/var/lib/kubelet/config.yaml`.

Ensure that the following files are copied to the worker node:
- `/etc/kubernetes/pki/ca.crt`
- `/etc/kubernetes/kubelet.conf`

Below is the configuration used for bootstrapping. Note the `staticPodPath` directive which enables static pods.

```yaml
# https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
  - 10.96.0.10
clusterDomain: cluster.local
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
cpuManagerReconcilePeriod: 0s
crashLoopBackOff: {}
evictionPressureTransitionPeriod: 0s
failSwapOn: false
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMaximumGCAge: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
    text:
      infoBufferSize: "0"
  verbosity: 0
memorySwap:
  swapBehavior: LimitedSwap
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

### Configure Kubelet Service

Ref: [Kubelet systemd configuration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)

**Important:** Ensure that `/etc/kubernetes/bootstrap-kubelet.conf` is present. A bootstrap token was generated in the [Kubernetes Configuration Files](06-kubernetes-configuration-files.md) lab and should have been copied to this location.

Create the directory for the drop-in configuration:

```bash
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d/
```

Create the drop-in configuration file `/usr/lib/systemd/system/kubelet.service.d/10-kubernetes-from-scratch.conf` to override the default kubelet flags and point to our configuration file:

```ini
# /usr/lib/systemd/system/kubelet.service.d/10-kubernetes-from-scratch.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

Reload the systemd daemon and restart kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
