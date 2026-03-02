# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a single node etcd cluster using **static pods**.

## Prerequisites

The commands in this lab must be run on each **controller** machine.

## Bootstrapping an etcd Cluster

We will deploy `etcd` as a static pod. The Kubelet on each controller node will watch the `/etc/kubernetes/manifests` directory and run the `etcd` pod defined there.

### Create the etcd Pod Manifest

We need to create the `etcd.yaml` manifest file in `/etc/kubernetes/manifests`.

The values for `initial-cluster` need to be set correctly for the cluster to form. Since we are bootstrapping one node at a time or in parallel, we need to know the IPs and Hostnames of the other controllers if we were doing a multi-node cluster. For this lab, we will generate the config dynamically based on the current node's information.

Run the following command on **each controller node** to create the manifest:

```bash
# Get the hostname
HOSTNAME=$(hostname)

# Create the static pod directory if it doesn't exist
mkdir -p /etc/kubernetes/manifests

# Write the etcd static pod manifest
cat <<EOF > /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://\$(POD_IP):2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --feature-gates=InitialCorruptCheck=true
    - --initial-advertise-peer-urls=https://\$(POD_IP):2380
    - --initial-cluster=${HOSTNAME}=https://\$(POD_IP):2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://\$(POD_IP):2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://\$(POD_IP):2380
    - --name=${HOSTNAME}
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --volume-plugin-dir=/opt/libexec/kubernetes/kubelet-plugins/volume/exec/
    - --watch-progress-notify-interval=5s
    env:
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    image: registry.k8s.io/etcd:3.5.13-0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /livez
        port: probe-port
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: etcd
    ports:
    - containerPort: 2381
      name: probe-port
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 127.0.0.1
        path: /readyz
        port: probe-port
        scheme: HTTP
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /readyz
        port: probe-port
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priority: 2000001000
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
EOF
```

*Note: In the `volumes` section, we map `hostPath: /etc/etcd` (where we copied certificates in previous labs) to `/etc/kubernetes/pki/etcd` inside the container, matching the path expected by the `command` arguments.*

## Verification

Once the static pod manifest is created, the Kubelet currently running on the node should pick it up and launch `etcd`.

You can verify it by checking the running containers (using `crictl` since we are using `containerd`) or by checking the logs if `kubectl` is configured (though `kubectl` might not work yet if the API server isn't up).

Check if the etcd container is running:

```bash
crictl ps --name etcd
```

Next: [Bootstrapping the Kubernetes Control Plane](09-bootstrapping-kubernetes-controllers.md)
