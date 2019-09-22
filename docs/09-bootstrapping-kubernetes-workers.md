# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap 2 Kubernetes worker nodes. We already have [Docker](https://www.docker.com) installed on these nodes.

We will now install the kubernetes components
- [kubelet](https://kubernetes.io/docs/admin/kubelet)
- [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this lab must be run on first worker instance: `192.168.1.23`. Login to first worker instance using SSH Terminal.

### Provisioning  Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for one worker node:

Worker1:

```
192.168.1.23$ cat > openssl-192.168.1.23.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kube-worker0
IP.1 = 192.168.1.23
EOF

openssl genrsa -out 192.168.1.23.key 2048
openssl req -new -key 192.168.1.23.key -subj "/CN=system:node:192.168.1.23/O=system:nodes" -out 192.168.1.23.csr -config openssl-192.168.1.23.cnf
openssl x509 -req -in 192.168.1.23.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out 192.168.1.23.crt -extensions v3_req -extfile openssl-192.168.1.23.cnf -days 1000
```

Results:

```
192.168.1.23.key
192.168.1.23.crt
```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

Get the kub-api server load-balancer IP.
```
LOADBALANCER_ADDRESS=192.168.1.24
```

Generate a kubeconfig file for the first worker node:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_ADDRESS}:6443 \
    --kubeconfig=192.168.1.23.kubeconfig

  kubectl config set-credentials system:node:192.168.1.23 \
    --client-certificate=192.168.1.23.crt \
    --client-key=192.168.1.23.key \
    --embed-certs=true \
    --kubeconfig=192.168.1.23.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:192.168.1.23 \
    --kubeconfig=192.168.1.23.kubeconfig

  kubectl config use-context default --kubeconfig=192.168.1.23.kubeconfig
}
```

Results:

```
192.168.1.23.kubeconfig
```

### Copy certificates, private keys and kubeconfig files to the worker node:

```
master-1$ scp ca.crt 192.168.1.23.crt 192.168.1.23.key 192.168.1.23.kubeconfig 192.168.1.23:~/
```

### Download and Install Worker Binaries

Going forward all activities are to be done on the `192.168.1.23` node.

```
192.168.1.23$ wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
```

Create the installation directories:

```
192.168.1.23$ sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
{
  chmod +x kubectl kube-proxy kubelet
  sudo mv kubectl kube-proxy kubelet /usr/local/bin/
}
```

### Configure the Kubelet

```
{
  sudo mv ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.crt /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

```
192.168.1.23$ cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
EOF
```

> The `resolvConf` configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`.

Create the `kubelet.service` systemd unit file:

```
192.168.1.23$ cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.crt \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}.key \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```
192.168.1.23$ sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy-config.yaml` configuration file:

```
192.168.1.23$ cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "192.168.5.0/24"
EOF
```

Create the `kube-proxy.service` systemd unit file:

```
192.168.1.23$ cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet kube-proxy
  sudo systemctl start kubelet kube-proxy
}
```

> Remember to run the above commands on worker node: `192.168.1.23`

## Verification


List the registered Kubernetes nodes from the master node:

```
192.168.1.23$ kubectl get nodes
```

> output

```
NAME       STATUS     ROLES    AGE   VERSION
kube-worker0   NotReady   <none>   93s   v1.13.0
```

> Note: It is OK for the worker node to be in a NotReady state.
  That is because we haven't configured Networking yet.
  
```
sudo systemctl status kubelet
```
```
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-09-22 16:54:51 EDT; 5s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 5744 (kubelet)
    Tasks: 12 (limit: 4675)
   CGroup: /system.slice/kubelet.service
           └─5744 /usr/local/bin/kubelet --config=/var/lib/kubelet/kubelet-config.yaml --image-pull-progress-deadline=2m --kubeconfig=/var/lib/kubelet/kubeconfig --tls-cert-file=/var/lib/kubelet/hadoop-slave2.crt --tls-private-key-file=/var/lib/kubelet/hadoop-slave2.key --network-plugin=cni --register-node=true --v=2

Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983657    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "weave-net-token-pfrh6" (UniqueName: "kubernetes.io/secret/e07b5332-dbe8-11e9-b501-00505681a1ee-weave-net-token-pfrh6") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983794    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "tiller-token-tcg4k" (UniqueName: "kubernetes.io/secret/5b03cd2f-d979-11e9-86c8-00505681a1ee-tiller-token-tcg4k") pod "tiller-deploy-6d65d78679-ppsrq" (UID: "5b03cd2f-d979-11e9-86c8-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983811    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "weavedb" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-weavedb") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983823    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "cni-bin" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-cni-bin") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983835    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "cni-bin2" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-cni-bin2") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983846    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "dbus" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-dbus") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983867    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "lib-modules" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-lib-modules") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983970    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "config-volume" (UniqueName: "kubernetes.io/configmap/72260a98-dbe9-11e9-b501-00505681a1ee-config-volume") pod "coredns-69cbb76ff8-tjwbx" (UID: "72260a98-dbe9-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.983985    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "cni-conf" (UniqueName: "kubernetes.io/host-path/e07b5332-dbe8-11e9-b501-00505681a1ee-cni-conf") pod "weave-net-7488l" (UID: "e07b5332-dbe8-11e9-b501-00505681a1ee")
Sep 22 16:54:54 hadoop-slave2 kubelet[5744]: I0922 16:54:54.984097    5744 operation_generator.go:567] MountVolume.SetUp succeeded for volume "coredns-token-xzt8h" (UniqueName: "kubernetes.io/secret/72260a98-dbe9-11e9-b501-00505681a1ee-coredns-token-xzt8h") pod "coredns-69cbb76ff8-tjwbx" (UID: "72260a98-dbe9-11e9-b501-00505681a1ee")
```
```
sudo systemctl status kube-proxy
```
```
● kube-proxy.service - Kubernetes Kube Proxy
   Loaded: loaded (/etc/systemd/system/kube-proxy.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-09-22 13:13:49 EDT; 3h 39min ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 21406 (kube-proxy)
    Tasks: 0 (limit: 4675)
   CGroup: /system.slice/kube-proxy.service
           └─21406 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/kube-proxy-config.yaml

Sep 22 13:13:49 hadoop-slave2 kube-proxy[21406]: I0922 13:13:49.911229   21406 server_others.go:178] Tearing down inactive rules.
Sep 22 13:13:49 hadoop-slave2 kube-proxy[21406]: E0922 13:13:49.958694   21406 proxier.go:583] Error removing iptables rules in ipvs proxier: error deleting chain "KUBE-MARK-MASQ": exit status 1: iptables: Too many links.
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.038555   21406 server.go:555] Version: v1.14.0
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.041354   21406 conntrack.go:52] Setting nf_conntrack_max to 131072
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.041763   21406 config.go:202] Starting service config controller
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.041774   21406 controller_utils.go:1027] Waiting for caches to sync for service config controller
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.041876   21406 config.go:102] Starting endpoints config controller
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.041885   21406 controller_utils.go:1027] Waiting for caches to sync for endpoints config controller
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.141892   21406 controller_utils.go:1034] Caches are synced for service config controller
Sep 22 13:13:50 hadoop-slave2 kube-proxy[21406]: I0922 13:13:50.142043   21406 controller_utils.go:1034] Caches are synced for endpoints config controller
```

Next: [TLS Bootstrapping Kubernetes Workers](10-tls-bootstrapping-kubernetes-workers.md)
