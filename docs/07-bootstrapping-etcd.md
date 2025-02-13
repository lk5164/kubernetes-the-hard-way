# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a two node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: `master-1`, and `master-2`. Login to each of these using an SSH terminal.

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address of the master(etcd) nodes:

```
INTERNAL_IP=$(ip addr show ens160 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

Create the `etcd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name  192.168.1.25\
  --cert-file=/etc/etcd/etcd-server.crt \
  --key-file=/etc/etcd/etcd-server.key \
  --peer-cert-file=/etc/etcd/etcd-server.crt \
  --peer-key-file=/etc/etcd/etcd-server.key \
  --trusted-ca-file=/etc/etcd/ca.crt \
  --peer-trusted-ca-file=/etc/etcd/ca.crt \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.1.25:2380 \
  --listen-peer-urls https://192.168.1.25:2380 \
  --listen-client-urls https://192.168.1.25:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.1.25:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 192.168.1.25=https://192.168.1.25:2380,192.168.1.17=https://192.168.1.17:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> Remember to run the above commands on each controller node: `kube-controller0`, and `kube-controller1`.

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

> output

```
7323693c5870702a, started, 192.168.1.25, https://192.168.1.25:2380, https://192.168.1.25:2379
f6f35cbe3961d862, started, 192.168.1.17, https://192.168.1.17:2380, https://192.168.1.17:2379
```

List endpoints status

```
sudo ETCDCTL_API=3 etcdctl -w table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
endpoint status
```

> output

```
+----------------+------------------+---------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+---------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | 7323693c5870702a |   3.3.9 |  4.3 MB |      true |     15300 |     363892 |
+----------------+------------------+---------+---------+-----------+-----------+------------+
```
## Trouble Shooting
If you didn't config ~/.kube/config, you may encounter error when you try the `etcdctl member list`. But if you just run `sudo ETCDCTL_API=3 etcdctl member list`, the error will go away. However, I would recommend configure ~/.kube/config first. 

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
