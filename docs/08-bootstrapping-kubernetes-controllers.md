# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across 2 compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `kube-controller0`, and `kube-controller1`. Login to each controller instance using SSH Terminal. Example:

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```
sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Configure the Kubernetes API Server

```
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo cp ca.crt ca.key kube-apiserver.crt kube-apiserver.key \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    encryption-config.yaml /var/lib/kubernetes/
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

```
INTERNAL_IP=$(ip addr show ens160 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

Verify it is set

```
echo $INTERNAL_IP
```

Create the `kube-apiserver.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.crt \\
  --enable-admission-plugins=NodeRestriction,ServiceAccount \\
  --enable-swagger-ui=true \\
  --enable-bootstrap-token-auth=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.crt \\
  --etcd-certfile=/var/lib/kubernetes/etcd-server.crt \\
  --etcd-keyfile=/var/lib/kubernetes/etcd-server.key \\
  --etcd-servers=https://192.168.5.11:2379,https://192.168.5.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.crt \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=192.168.5.0/24 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --address=127.0.0.1 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.


### Verification

```
kubectl get componentstatuses
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

> Remember to run the above commands on each controller node: `kube-controller0`, and `kube-controller1`.

```
sudo systemctl status kube-apiserver.service
```

```
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-09-08 09:04:44 EDT; 8h ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 977 (kube-apiserver)
    Tasks: 8 (limit: 4675)
   CGroup: /system.slice/kube-apiserver.service
           └─977 /usr/local/bin/kube-apiserver --advertise-address=192.168.1.25 --allow-privileged=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/log/audit.log --authorization-mode=Node,RBAC --bind-address=0.0.0.0 --client-ca-file=/var/lib/kubernetes/ca.crt --enable-admission-plugins=NodeRestriction,ServiceAccount --enable-swagger-ui=true --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/kubernetes/ca.crt --etcd-certfile=/var/lib/kubernetes/etcd-server.crt --etcd-keyfile=/var/lib/kubernetes/etcd-server.key   --etcd-servers=https://192.168.1.25:2379,https://192.168.1.17:2379   --event-ttl=1h   --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml   --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt   --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt   --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key   --kubelet-https=true   --runtime-config=api/all   --service-account-key-file=/var/lib/kubernetes/service-account.crt   --service-cluster-ip-range=10.96.0.0/24   --service-node-port-range=30000-32767   --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt   --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key   --v=2   --storage-backend=etcd3   --storage-media-type=application/json
Sep 08 09:06:22 kube-controller0 kube-apiserver[977]: I0908 09:06:22.296676     977 trace.go:76] Trace[1747160357]: "Patch /api/v1/nodes/hadoop-slave4/status" (started: 2019-09-08 09:06:21.509817826 -0400 EDT m=+95.479646800) (total time: 786.839583ms):
Sep 08 09:06:22 kube-controller0 kube-apiserver[977]: Trace[1747160357]: [786.795641ms] [786.209359ms] Object stored in database
Sep 08 09:21:19 kube-controller0 kube-apiserver[977]: I0908 09:21:19.703040     977 trace.go:76] Trace[1598666566]: "GuaranteedUpdate etcd3: *core.Endpoints" (started: 2019-09-08 09:21:19.13040613 -0400 EDT m=+993.100235114) (total time: 572.566959ms):
Sep 08 09:21:19 kube-controller0 kube-apiserver[977]: Trace[1598666566]: [572.482621ms] [572.259493ms] Transaction committed
Sep 08 09:21:19 kube-controller0 kube-apiserver[977]: I0908 09:21:19.704118     977 trace.go:76] Trace[35931933]: "Update /api/v1/namespaces/kube-system/endpoints/kube-scheduler" (started: 2019-09-08 09:21:19.130285093 -0400 EDT m=+993.100114117) (total time: 573.773508ms):
Sep 08 09:21:19 kube-controller0 kube-apiserver[977]: Trace[35931933]: [573.675615ms] [573.588011ms] Object stored in database
Sep 08 10:25:29 kube-controller0 kube-apiserver[977]: I0908 10:25:29.077577     977 trace.go:76] Trace[2144310406]: "GuaranteedUpdate etcd3: *core.Endpoints" (started: 2019-09-08 10:25:28.535285904 -0400 EDT m=+4842.505114898) (total time: 542.211809ms):
Sep 08 10:25:29 kube-controller0 kube-apiserver[977]: Trace[2144310406]: [542.150524ms] [541.740918ms] Transaction committed
Sep 08 10:25:29 kube-controller0 kube-apiserver[977]: I0908 10:25:29.078265     977 trace.go:76] Trace[201963089]: "Update /api/v1/namespaces/kube-system/endpoints/kube-scheduler" (started: 2019-09-08 10:25:28.535157675 -0400 EDT m=+4842.504986679) (total time: 543.087004ms):
Sep 08 10:25:29 kube-controller0 kube-apiserver[977]: Trace[201963089]: [543.01548ms] [542.946952ms] Object stored in database

```

```
sudo systemctl status kube-controller-manager.service
```

```
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-09-08 09:04:44 EDT; 8h ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 960 (kube-controller)
    Tasks: 7 (limit: 4675)
   CGroup: /system.slice/kube-controller-manager.service
           └─960 /usr/local/bin/kube-controller-manager --address=0.0.0.0 --cluster-cidr=192.168.1.0/24 --cluster-name=kubernetes --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt --cluster-signing-key-file=/var/lib/kubernetes/ca.key --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig --leader-elect=true --root-ca-file=/var/lib/kubernetes/ca.crt --service-account-private-key-file=/var/lib/kubernetes/service-account.key --service-cluster-ip-range=10.96.0.0/24 --use-service-account-credentials=true --v=2

Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.654531     960 cleaner.go:141] Cleaning CSR "csr-gjk85" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.659545     960 cleaner.go:166] Cleaning CSR "csr-9czs9" as it is more than 1h0m0s old and approved.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.665541     960 cleaner.go:141] Cleaning CSR "csr-h25jn" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.670250     960 cleaner.go:141] Cleaning CSR "csr-w2wjh" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.675208     960 cleaner.go:141] Cleaning CSR "csr-xcpn6" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.679986     960 cleaner.go:141] Cleaning CSR "csr-f72mn" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.684851     960 cleaner.go:141] Cleaning CSR "csr-2pjhm" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.692905     960 cleaner.go:141] Cleaning CSR "csr-2z5lp" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.698029     960 cleaner.go:141] Cleaning CSR "csr-pswkz" as it is more than 24h0m0s old and unhandled.
Sep 08 10:05:48 kube-controller0 kube-controller-manager[960]: I0908 10:05:48.702861     960 cleaner.go:141] Cleaning CSR "csr-chjgf" as it is more than 24h0m0s old and unhandled.
```

```
sudo systemctl status kube-scheduler.service
```

```
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-09-08 09:04:48 EDT; 8h ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1189 (kube-scheduler)
    Tasks: 8 (limit: 4675)
   CGroup: /system.slice/kube-scheduler.service
           └─1189 /usr/local/bin/kube-scheduler --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig --address=127.0.0.1 --leader-elect=true --v=2

Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682274    1189 reflector.go:134] k8s.io/client-go/informers/factory.go:132: Failed to list *v1.Node: nodes is forbidden: User "system:kube-scheduler" cannot list resource "nodes" in API group "" at the cluster scope
Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682313    1189 reflector.go:134] k8s.io/client-go/informers/factory.go:132: Failed to list *v1.Service: services is forbidden: User "system:kube-scheduler" cannot list resource "services" in API group "" at the cluster scope
Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682352    1189 reflector.go:134] k8s.io/kubernetes/cmd/kube-scheduler/app/server.go:232: Failed to list *v1.Pod: pods is forbidden: User "system:kube-scheduler" cannot list resource "pods" in API group "" at the cluster scope
Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682397    1189 reflector.go:134] k8s.io/client-go/informers/factory.go:132: Failed to list *v1.ReplicationController: replicationcontrollers is forbidden: User "system:kube-scheduler" cannot list resource "replicationcontrollers" in API group "" at the cluster scope
Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682451    1189 reflector.go:134] k8s.io/client-go/informers/factory.go:132: Failed to list *v1.StorageClass: storageclasses.storage.k8s.io is forbidden: User "system:kube-scheduler" cannot list resource "storageclasses" in API group "storage.k8s.io" at the cluster scope
Sep 08 09:05:27 kube-controller0 kube-scheduler[1189]: E0908 09:05:27.682487    1189 reflector.go:134] k8s.io/client-go/informers/factory.go:132: Failed to list *v1.PersistentVolume: persistentvolumes is forbidden: User "system:kube-scheduler" cannot list resource "persistentvolumes" in API group "" at the cluster scope
Sep 08 09:05:29 kube-controller0 kube-scheduler[1189]: I0908 09:05:29.808847    1189 controller_utils.go:1027] Waiting for caches to sync for scheduler controller
Sep 08 09:05:29 kube-controller0 kube-scheduler[1189]: I0908 09:05:29.908986    1189 controller_utils.go:1034] Caches are synced for scheduler controller
Sep 08 09:05:29 kube-controller0 kube-scheduler[1189]: I0908 09:05:29.909024    1189 leaderelection.go:205] attempting to acquire leader lease  kube-system/kube-scheduler...
Sep 08 09:05:45 kube-controller0 kube-scheduler[1189]: I0908 09:05:45.315564    1189 leaderelection.go:214] successfully acquired lease kube-system/kube-scheduler
```

Using systemctl to check service status is critical. You need to make sure there is no error in the log. Sometimes, even the service status is available, there are still problems that causes fatal error.

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
