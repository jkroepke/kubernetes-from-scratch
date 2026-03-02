# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes client configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), typically called kubeconfigs, which configure Kubernetes clients to connect and authenticate to Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `bootstrap-kubelet`, `kube-controller-manager`, `kube-scheduler`, and `admin` user.

### The Kubelet Kubernetes Configuration File

Generate a kubeconfig file for the `bootstrap-kubelet` user using the bootstrap token:

```bash
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=bootstrap-kubelet.kubeconfig

  kubectl config set-credentials kubelet \
    --token=07401b.f395accd246ae52d \
    --kubeconfig=bootstrap-kubelet.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=kubelet \
    --kubeconfig=bootstrap-kubelet.kubeconfig

  kubectl config use-context default \
    --kubeconfig=bootstrap-kubelet.kubeconfig
}
```

Results:

```text
bootstrap-kubelet.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```bash
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

Results:

```text
kube-controller-manager.kubeconfig
```

### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```bash
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

Results:

```text
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```bash
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials kubernetes-admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=kubernetes-admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```

Results:

```text
admin.kubeconfig
```

### The Kubelet Configuration File

Generate the `kubelet-config.yaml` configuration file:

```bash
cat <<EOF > kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m0s
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
  - "10.96.0.10"
clusterDomain: cluster.local
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
cpuCFSQuota: true
cpuCFSQuotaPeriod: 100ms
enableServer: true
eventRecordQPS: 0
featureGates:
  SeccompDefault: true
hairpinMode: promiscuous-bridge
kernelMemcgNotification: true
logging:
  format: text
maxPods: 110
podCIDR: "POD_CIDR"
protectKernelDefaults: true
registerNode: true
registerWithTaints:
  - key: node.kubernetes.io/unschedulable
    effect: NoSchedule
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runOnce: false
seccompDefault: true
serializeImagePulls: false
serverTLSBootstrap: true
shutdownGracePeriod: 30s
shutdownGracePeriodCriticalPods: 10s
staticPodPath: /etc/kubernetes/manifests
tlsCipherSuites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  - TLS_RSA_WITH_AES_256_GCM_SHA384
  - TLS_RSA_WITH_AES_128_GCM_SHA256
volumePluginDir: /opt/libexec/kubernetes/kubelet-plugins/volume/exec/
EOF
```

## Distribute the Kubernetes Configuration Files

Copy the `bootstrap-kubelet` kubeconfig file and `kubelet-config.yaml` to the `worker` machines. We will assign a Pod CIDR to each worker node numerically starting from `10.200.0.0/24`.

```bash
i=0
while read IP HOST; do
  if [[ "$HOST" == worker* ]]; then
    POD_CIDR="10.200.${i}.0/24"
    ssh root@${IP} "mkdir -p /var/lib/kubelet /etc/kubernetes"

    scp bootstrap-kubelet.kubeconfig \
      root@${IP}:/etc/kubernetes/bootstrap-kubelet.conf

    # We need to replace POD_CIDR with the actual CIDR for the node
    sed "s|POD_CIDR|${POD_CIDR}|g" kubelet-config.yaml > kubelet-config-${HOST}.yaml
    
    scp kubelet-config-${HOST}.yaml \
      root@${IP}:/var/lib/kubelet/config.yaml
      
    rm kubelet-config-${HOST}.yaml
    i=$((i+1))
  fi
done < machines.txt
```

Copy the `kube-controller-manager`, `kube-scheduler`, and `admin` kubeconfig files to the `server` machine:

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
