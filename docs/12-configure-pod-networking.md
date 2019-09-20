# Provisioning Pod Network

We chose to use CNI - [weave](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/) as our networking option.

### Install CNI plugins

Download the CNI Plugins required for weave on each of the worker nodes - `kube-worker0` and `kube-worker1`

`wget https://github.com/containernetworking/plugins/releases/download/v0.7.5/cni-plugins-amd64-v0.7.5.tgz`

Extract it to /opt/cni/bin directory

`sudo tar -xzvf cni-plugins-amd64-v0.7.5.tgz  --directory /opt/cni/bin/`

### Deploy Weave Network

Deploy weave network. Run only once on the `master` node.


`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"&env.IPALLOC_RANGE=10.96.1.0/24&env.log-level=debug"`

Weave uses POD CIDR of `10.32.0.0/12` by default.
Note. the ip range cannot overlapp with other kubernetes service. The iptable won't able to save records if that happens. And kube-proxy won't say anything in the log.  

## Verification

List the registered Kubernetes nodes from the master node:

```
master-1$ kubectl get pods -n kube-system
```

> output

```
NAME              READY   STATUS    RESTARTS   AGE
weave-net-58j2j   2/2     Running   0          89s
weave-net-rr5dk   2/2     Running   0          89s
```

### Verify Pod Communication

```
for node_ip in `cat workers`
do
echo ">>> ${node_ip}"
ssh ${node_ip} "ip addr show weave|grep -w inet"
done
>>> 192.168.1.23
    inet 10.96.0.1/24 brd 10.96.0.255 scope global weave
>>> 192.168.1.18
    inet 10.96.0.128/24 brd 10.96.0.255 scope global weave
```
ssh to all nodes and ping weave ip

```
for node_ip in `cat workers`
do
echo ">>> ${node_ip}"
ssh ${node_ip} "ping -c 1 10.96.0.1"
ssh ${node_ip} "ping -c 1 10.96.0.128"
done
>>> 192.168.1.23
PING 10.96.0.1 (10.96.0.1) 56(84) bytes of data.
64 bytes from 10.96.0.1: icmp_seq=1 ttl=64 time=0.033 ms

--- 10.96.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.033/0.033/0.033/0.000 ms
PING 10.96.0.128 (10.96.0.128) 56(84) bytes of data.
64 bytes from 10.96.0.128: icmp_seq=1 ttl=64 time=1.03 ms

--- 10.96.0.128 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.039/1.039/1.039/0.000 ms
>>> 192.168.1.18
PING 10.96.0.1 (10.96.0.1) 56(84) bytes of data.
64 bytes from 10.96.0.1: icmp_seq=1 ttl=64 time=0.250 ms

--- 10.96.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.250/0.250/0.250/0.000 ms
PING 10.96.0.128 (10.96.0.128) 56(84) bytes of data.
64 bytes from 10.96.0.128: icmp_seq=1 ttl=64 time=0.034 ms

--- 10.96.0.128 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.034/0.034/0.034/0.000 ms
```

Next: [Kube API Server to Kubelet Connectivity](13-kube-apiserver-to-kubelet.md)
