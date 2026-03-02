# Generating Certificate Authority and TLS Certificates

In this section, you will generate a Certificate Authority (CA) and TLS certificates for the Kubernetes cluster.

## Why do we need a Certificate Authority?

Kubernetes requires PKI (Public Key Infrastructure) certificates for authentication over TLS. If you install Kubernetes with a tool like kubeadm, the certificates are automatically generated for you. In this tutorial, you will generate a Certificate Authority (CA) and use it to sign the TLS certificates for your cluster components.

We will create two separate Certificate Authorities:
1. **etcd CA**: Secured communication between etcd peers and clients (like the API Server).
2. **Kubernetes CA**: Secured communication between Kubernetes components (API Server, Kubelet, Scheduler, Controller Manager, etc.).

We want to stay modern, so we will be using **ECDSA (Elliptic Curve Digital Signature Algorithm)** instead of RSA for our keys. ECDSA offers better performance and security with smaller key sizes. specifically the `P-384` curve.

## Directory Structure

To keep things organized, we will create a structured directory to hold our CA keys and signed certificates.

Run the following commands on your jumpbox to create the directories:

```bash
mkdir -p ca/etcd ca/kubernetes
```

## 1. Etcd Certificate Authority

First, let's set up the Certificate Authority for `etcd`. This CA will define the trust root for the etcd cluster.

### Generate the CA Key and Certificate

Create the CA configuration file `ca/etcd/ca.conf`:

```bash
cat <<EOF > ca/etcd/ca.conf
[req]
distinguished_name = req_distinguished_name
prompt = no
x509_extensions = v3_ca

[req_distinguished_name]
CN = etcd-ca
O = etcd

[v3_ca]
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

Generate the `etcd-ca` private key and self-signed certificate:

```bash
# Generate private key (ECDSA P-384)
openssl ecparam -name secp384r1 -genkey -noout -out ca/etcd/ca.key

# Generate self-signed certificate (valid for 10 years)
openssl req -new -x509 -key ca/etcd/ca.key -sha256 -config ca/etcd/ca.conf \
  -days 3650 -out ca/etcd/ca.crt
```

### Clean up standard OpenSSL config for signing

We need a configuration file for signing the subsequent certificates. Create `ca/etcd/openssl.cnf`:

```bash
cat <<EOF > ca/etcd/openssl.cnf
[req]
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
CN = ETCD_CN
O = ETCD_ORG

[v3_ext]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
IP.1 = 127.0.0.1
EOF
```

## 2. Etcd Certificates

We need to generate certificates for the `etcd` servers, peers, and clients.

### A. Etcd Server Certificate

This certificate is used by the etcd server to secure client connections.

Create the request config specific to the server (we will dynamically append SANs later):

```bash
# We will use a loop to generate for all controllers later, 
# but for now let's prepare the template.
cp ca/etcd/openssl.cnf ca/etcd/server.cnf
```

We need to generate server certificates for each controller node.
Assuming you have defined your controllers in `machines.txt`:

```bash
# Loop through controllers to generate server certs
while read -r IP _ HOSTNAME _; do
    if [[ "$HOSTNAME" == controller* ]]; then
        echo "Generating etcd server cert for $HOSTNAME ($IP)"
        
        # Create config for specific host
        cp ca/etcd/openssl.cnf ca/etcd/server-${HOSTNAME}.cnf
        sed -i "s/ETCD_CN/etcd-server/" ca/etcd/server-${HOSTNAME}.cnf
        sed -i "s/ETCD_ORG/etcd/" ca/etcd/server-${HOSTNAME}.cnf
        
        # Append specific SANs
        echo "DNS.2 = $HOSTNAME" >> ca/etcd/server-${HOSTNAME}.cnf
        # echo "IP.2 = $IP" >> ca/etcd/server-${HOSTNAME}.cnf # dynamic

        # Generate Key
        openssl ecparam -name secp384r1 -genkey -noout -out ca/etcd/server-${HOSTNAME}.key

        # Generate CSR
        openssl req -new -key ca/etcd/server-${HOSTNAME}.key -out ca/etcd/server-${HOSTNAME}.csr -config ca/etcd/server-${HOSTNAME}.cnf

        # Sign Certificate
        openssl x509 -req -in ca/etcd/server-${HOSTNAME}.csr -CA ca/etcd/ca.crt -CAkey ca/etcd/ca.key -CAcreateserial \
          -out ca/etcd/server-${HOSTNAME}.crt -days 365 -sha256 -extensions v3_ext -extfile ca/etcd/server-${HOSTNAME}.cnf
    fi
done < machines.txt
```

### B. Etcd Peer Certificate

Etcd members communicate with each other using peer certificates.

```bash
while read -r IP _ HOSTNAME _; do
    if [[ "$HOSTNAME" == controller* ]]; then
        echo "Generating etcd peer cert for $HOSTNAME ($IP)"
        
        cp ca/etcd/openssl.cnf ca/etcd/peer-${HOSTNAME}.cnf
        sed -i "s/ETCD_CN/etcd-peer/" ca/etcd/peer-${HOSTNAME}.cnf
        sed -i "s/ETCD_ORG/etcd/" ca/etcd/peer-${HOSTNAME}.cnf
        
        echo "DNS.2 = $HOSTNAME" >> ca/etcd/peer-${HOSTNAME}.cnf
        # Peer certs need clientAuth and serverAuth (already in base config)

        openssl ecparam -name secp384r1 -genkey -noout -out ca/etcd/peer-${HOSTNAME}.key

        openssl req -new -key ca/etcd/peer-${HOSTNAME}.key -out ca/etcd/peer-${HOSTNAME}.csr -config ca/etcd/peer-${HOSTNAME}.cnf

        openssl x509 -req -in ca/etcd/peer-${HOSTNAME}.csr -CA ca/etcd/ca.crt -CAkey ca/etcd/ca.key -CAcreateserial \
          -out ca/etcd/peer-${HOSTNAME}.crt -days 365 -sha256 -extensions v3_ext -extfile ca/etcd/peer-${HOSTNAME}.cnf
    fi
done < machines.txt
```

### C. Etcd Healthcheck Client Certificate

Used for checking the health of the etcd cluster (e.g., liveness probes).

```bash
# Config
cp ca/etcd/openssl.cnf ca/etcd/healthcheck-client.cnf
sed -i "s/ETCD_CN/kube-etcd-healthcheck-client/" ca/etcd/healthcheck-client.cnf
sed -i "s/ETCD_ORG/etcd/" ca/etcd/healthcheck-client.cnf
# Client only
sed -i "s/extendedKeyUsage = clientAuth, serverAuth/extendedKeyUsage = clientAuth/" ca/etcd/healthcheck-client.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/etcd/healthcheck-client.key

# Generate CSR
openssl req -new -key ca/etcd/healthcheck-client.key -out ca/etcd/healthcheck-client.csr -config ca/etcd/healthcheck-client.cnf

# Sign Certificate
openssl x509 -req -in ca/etcd/healthcheck-client.csr -CA ca/etcd/ca.crt -CAkey ca/etcd/ca.key -CAcreateserial \
  -out ca/etcd/healthcheck-client.crt -days 365 -sha256 -extensions v3_ext -extfile ca/etcd/healthcheck-client.cnf
```

### D. Kubernetes API Server Etcd Client Certificate

The Kubernetes API Server initiates a connection to etcd. It needs a client certificate signed by the etcd CA.
Wait, we need this for **all three controllers** since the API server runs on each controller.

```bash
# Config
cp ca/etcd/openssl.cnf ca/etcd/apiserver-etcd-client.cnf
sed -i "s/ETCD_CN/kube-apiserver-etcd-client/" ca/etcd/apiserver-etcd-client.cnf
# Organization must be system:masters
sed -i "s/ETCD_ORG/system:masters/" ca/etcd/apiserver-etcd-client.cnf
# Client only
sed -i "s/extendedKeyUsage = clientAuth, serverAuth/extendedKeyUsage = clientAuth/" ca/etcd/apiserver-etcd-client.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/etcd/apiserver-etcd-client.key

# Generate CSR
openssl req -new -key ca/etcd/apiserver-etcd-client.key -out ca/etcd/apiserver-etcd-client.csr -config ca/etcd/apiserver-etcd-client.cnf

# Sign Certificate
openssl x509 -req -in ca/etcd/apiserver-etcd-client.csr -CA ca/etcd/ca.crt -CAkey ca/etcd/ca.key -CAcreateserial \
  -out ca/etcd/apiserver-etcd-client.crt -days 365 -sha256 -extensions v3_ext -extfile ca/etcd/apiserver-etcd-client.cnf
```

## 3. Kubernetes Certificate Authority

Now we will generate the CA that will sign all Kubernetes components (outside of etcd).

### Generate the Kubernetes CA Key and Certificate

Create `ca/kubernetes/ca.conf`:
```bash
cat <<EOF > ca/kubernetes/ca.conf
[req]
distinguished_name = req_distinguished_name
prompt = no
x509_extensions = v3_ca

[req_distinguished_name]
CN = kubernetes
O = kubernetes

[v3_ca]
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

Generate the key and certificate:

```bash
# Generate private key (ECDSA P-384)
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/ca.key

# Generate self-signed certificate (valid for 10 years)
openssl req -new -x509 -key ca/kubernetes/ca.key -sha256 -config ca/kubernetes/ca.conf \
  -days 3650 -out ca/kubernetes/ca.crt

### Clean up standard OpenSSL config for signing

Create `ca/kubernetes/openssl.cnf`:

```bash
cat <<EOF > ca/kubernetes/openssl.cnf
[req]
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
CN = KUBE_CN
O = KUBE_ORG

[v3_ext]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
```

## 4. Kubernetes Certificates

We need to generate certificates for the various Kubernetes components.

### A. Admin Client Certificate

The `admin` client certificate is used by the `admin` user to authenticate with the Kubernetes cluster via `kubectl`.

```bash
# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/admin.cnf
sed -i "s/KUBE_CN/kubernetes-admin/" ca/kubernetes/admin.cnf
sed -i "s/KUBE_ORG/system:masters/" ca/kubernetes/admin.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/admin.key

# Generate CSR
openssl req -new -key ca/kubernetes/admin.key -out ca/kubernetes/admin.csr -config ca/kubernetes/admin.cnf

# Sign Certificate
openssl x509 -req -in ca/kubernetes/admin.csr -CA ca/kubernetes/ca.crt -CAkey ca/kubernetes/ca.key -CAcreateserial \
  -out ca/kubernetes/admin.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/admin.cnf
```

### B. Kube-Controller-Manager Client Certificate

The `kube-controller-manager` is a daemon that embeds the core control loops shipped with Kubernetes.

```bash
# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/controller-manager.cnf
sed -i "s/KUBE_CN/system:kube-controller-manager/" ca/kubernetes/controller-manager.cnf
sed -i "s/O = KUBE_ORG//" ca/kubernetes/controller-manager.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/controller-manager.key

# Generate CSR
openssl req -new -key ca/kubernetes/controller-manager.key -out ca/kubernetes/controller-manager.csr -config ca/kubernetes/controller-manager.cnf

# Sign Certificate
openssl x509 -req -in ca/kubernetes/controller-manager.csr -CA ca/kubernetes/ca.crt -CAkey ca/kubernetes/ca.key -CAcreateserial \
  -out ca/kubernetes/controller-manager.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/controller-manager.cnf
```

### C. Kube-Scheduler Client Certificate

The `kube-scheduler` watches for newly created Pods with no assigned node, and selects a node for them to run on.

```bash
# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/scheduler.cnf
sed -i "s/KUBE_CN/system:kube-scheduler/" ca/kubernetes/scheduler.cnf
sed -i "s/O = KUBE_ORG//" ca/kubernetes/scheduler.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/scheduler.key

# Generate CSR
openssl req -new -key ca/kubernetes/scheduler.key -out ca/kubernetes/scheduler.csr -config ca/kubernetes/scheduler.cnf

# Sign Certificate
openssl x509 -req -in ca/kubernetes/scheduler.csr -CA ca/kubernetes/ca.crt -CAkey ca/kubernetes/ca.key -CAcreateserial \
  -out ca/kubernetes/scheduler.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/scheduler.cnf
```

### D. Kube API Server Certificate

The Kubernetes API server validates and configures data for the api objects which include pods, services, replicationcontrollers, and others.

This certificate needs multiple Subject Alternative Names (SANs) because it is accessed via:
1.  The Internal Kubernetes Service IP (`10.96.0.1` by default)
2.  The Loopback IP (`127.0.0.1`)
3.  The Load Balancer IP/Hostname (Public Address)
4.  The internal DNS names (`kubernetes`, `kubernetes.default`, etc.)

**Important**: You must define your public IP (Load Balancer IP) and Service IP.

```bash
# Define the Kubernetes Service IP (First IP of service CIDR 10.96.0.0/24)
KUBERNETES_SERVICE_IP=10.96.0.1

# Define your Public IP / Load Balancer IP
# If you don't have a load balancer, use the IP of the first controller
export KUBERNETES_PUBLIC_ADDRESS=$(grep 'controller1' machines.txt | cut -d " " -f 1)
# OR if you have a specific loadbalancer entry in machines.txt:
# export KUBERNETES_PUBLIC_ADDRESS=$(grep 'loadbalancer' machines.txt | cut -d " " -f 1)

# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/apiserver.cnf
sed -i "s/KUBE_CN/kube-apiserver/" ca/kubernetes/apiserver.cnf
sed -i "s/KUBE_ORG/system:masters/" ca/kubernetes/apiserver.cnf # Although usually O is not strictly required for server cert, it doesn't hurt.
sed -i "s/extendedKeyUsage = clientAuth/extendedKeyUsage = serverAuth/" ca/kubernetes/apiserver.cnf

cat <<EOF >> ca/kubernetes/apiserver.cnf
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = ${KUBERNETES_SERVICE_IP}
IP.3 = ${KUBERNETES_PUBLIC_ADDRESS}
EOF

# Add Controller IPs to SANs (since API Server runs on each controller)
# We append to the [alt_names] section we just created
i=4
while read -r IP _ HOSTNAME _; do
    if [[ "$HOSTNAME" == controller* ]]; then
        echo "IP.$i = $IP" >> ca/kubernetes/apiserver.cnf
        i=$((i+1))
    fi
done < machines.txt

# Ensure subjectAltName is enabled in the config
# We need to make sure the [v3_ext] section uses the @alt_names.
# The base openssl.cnf doesn't have subjectAltName = @alt_names in [v3_ext] yet?
# Let's check existing config:
# basicConstraints = CA:FALSE
# keyUsage = critical, digitalSignature, keyEncipherment
# extendedKeyUsage = clientAuth
# We need to append subjectAltName line to [v3_ext] before [alt_names]... 
# Actually simpler to just echo it now.
sed -i '/extendedKeyUsage/a subjectAltName = @alt_names' ca/kubernetes/apiserver.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/apiserver.key

# Generate CSR
openssl req -new -key ca/kubernetes/apiserver.key -out ca/kubernetes/apiserver.csr -config ca/kubernetes/apiserver.cnf

# Sign Certificate
openssl x509 -req -in ca/kubernetes/apiserver.csr -CA ca/kubernetes/ca.crt -CAkey ca/kubernetes/ca.key -CAcreateserial \
  -out ca/kubernetes/apiserver.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/apiserver.cnf
```


### E. API Server Kubelet Client Certificate

The API Server needs to authenticate to the Kubelets (running on worker nodes) to retrieve logs, execute commands, etc.

```bash
# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/apiserver-kubelet-client.cnf
sed -i "s/KUBE_CN/kube-apiserver-kubelet-client/" ca/kubernetes/apiserver-kubelet-client.cnf
sed -i "s/KUBE_ORG/system:masters/" ca/kubernetes/apiserver-kubelet-client.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/apiserver-kubelet-client.key

# Generate CSR
openssl req -new -key ca/kubernetes/apiserver-kubelet-client.key -out ca/kubernetes/apiserver-kubelet-client.csr -config ca/kubernetes/apiserver-kubelet-client.cnf

# Sign Certificate
openssl x509 -req -in ca/kubernetes/apiserver-kubelet-client.csr -CA ca/kubernetes/ca.crt -CAkey ca/kubernetes/ca.key -CAcreateserial \
  -out ca/kubernetes/apiserver-kubelet-client.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/apiserver-kubelet-client.cnf
```

### F. Front Proxy Certificate Authority

The Front Proxy CA is used to support the aggregation layer, which allows extending the Kubernetes API with additional APIs.

```bash
# CA Config
cat <<EOF > ca/kubernetes/front-proxy-ca.conf
[req]
distinguished_name = req_distinguished_name
prompt = no
x509_extensions = v3_ca

[req_distinguished_name]
CN = kubernetes-front-proxy-ca

[v3_ca]
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, keyEncipherment, keyCertSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF

# Generate CA Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/front-proxy-ca.key

# Generate CA Cert
openssl req -new -x509 -key ca/kubernetes/front-proxy-ca.key -sha256 -config ca/kubernetes/front-proxy-ca.conf \
  -days 3650 -out ca/kubernetes/front-proxy-ca.crt
```

### G. Front Proxy Client Certificate

This certificate is used by the API Server to authenticate to the aggregated API servers.

```bash
# Config
cp ca/kubernetes/openssl.cnf ca/kubernetes/front-proxy-client.cnf
sed -i "s/KUBE_CN/front-proxy-client/" ca/kubernetes/front-proxy-client.cnf
sed -i "s/O = KUBE_ORG//" ca/kubernetes/front-proxy-client.cnf

# Generate Key
openssl ecparam -name secp384r1 -genkey -noout -out ca/kubernetes/front-proxy-client.key

# Generate CSR
openssl req -new -key ca/kubernetes/front-proxy-client.key -out ca/kubernetes/front-proxy-client.csr -config ca/kubernetes/front-proxy-client.cnf

# Sign Certificate (Signed by Front Proxy CA, NOT Kubernetes CA)
openssl x509 -req -in ca/kubernetes/front-proxy-client.csr -CA ca/kubernetes/front-proxy-ca.crt -CAkey ca/kubernetes/front-proxy-ca.key -CAcreateserial \
  -out ca/kubernetes/front-proxy-client.crt -days 365 -sha256 -extensions v3_ext -extfile ca/kubernetes/front-proxy-client.cnf
```

### H. Service Account Key Pair

The Kubernetes Controller Manager uses a key pair to generate and sign service account tokens.
**Note:** We use RSA for this key pair because AWS IAM Authenticator and some OIDC providers have historically had better support for RSA signed JWTs, although ECDSA is supported by Kubernetes. We will stick to RSA 4096 to ensure broad compatibility.

```bash
# Generate Private Key
openssl genrsa -out ca/kubernetes/sa.key 4096

# Generate Public Key
openssl rsa -in ca/kubernetes/sa.key -pubout -out ca/kubernetes/sa.pub
```

## 5. Distribute the Certificates

Now that we have generated all the necessary certificates, we need to copy them to the appropriate machines.

We will create the necessary directories on the controller nodes and copy the certificates.

```bash
while read -r IP _ HOSTNAME _; do
    if [[ "$HOSTNAME" == controller* ]]; then
        echo "Distributing certificates to $HOSTNAME..."
        
        # Create directories
        ssh -i ~/.ssh/id_rsa root@${IP} "mkdir -p /etc/etcd /etc/kubernetes/pki"

        # Copy Etcd Certificates
        scp -i ~/.ssh/id_rsa \
          ca/etcd/ca.crt \
          ca/etcd/server-${HOSTNAME}.key \
          ca/etcd/server-${HOSTNAME}.crt \
          ca/etcd/peer-${HOSTNAME}.key \
          ca/etcd/peer-${HOSTNAME}.crt \
          ca/etcd/healthcheck-client.key \
          ca/etcd/healthcheck-client.crt \
          ca/etcd/apiserver-etcd-client.key \
          ca/etcd/apiserver-etcd-client.crt \
          root@${IP}:/etc/etcd/

        # Copy Kubernetes Certificates
        scp -i ~/.ssh/id_rsa \
          ca/kubernetes/ca.key \
          ca/kubernetes/ca.crt \
          ca/kubernetes/apiserver.key \
          ca/kubernetes/apiserver.crt \
          ca/kubernetes/apiserver-kubelet-client.key \
          ca/kubernetes/apiserver-kubelet-client.crt \
          ca/kubernetes/front-proxy-ca.key \
          ca/kubernetes/front-proxy-ca.crt \
          ca/kubernetes/front-proxy-client.key \
          ca/kubernetes/front-proxy-client.crt \
          ca/kubernetes/sa.key \
          ca/kubernetes/sa.pub \
          ca/kubernetes/controller-manager.key \
          ca/kubernetes/controller-manager.crt \
          ca/kubernetes/scheduler.key \
          ca/kubernetes/scheduler.crt \
          ca/kubernetes/admin.key \
          ca/kubernetes/admin.crt \
          root@${IP}:/etc/kubernetes/pki/
    fi
done < machines.txt
```

You have now successfully generated and distributed all the required certificates for your Kubernetes cluster!
