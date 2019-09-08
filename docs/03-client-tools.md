# Installing the Client Tools

First identify a system from where you will perform administrative tasks, such as creating certificates, kubeconfig files and distributing them to the different VMs.

If you are on a Linux laptop, then your laptop could be this system. In my case I chose the kube-haproxy node to perform administrative tasks. Whichever system you chose make sure that system is able to access all the provisioned VMs through SSH to copy files over.

## Create workers file and controllers file

```
vim workers
```
```
kube-worker0
kube-worker1
```

```
vim controllers
```
```
kube-controller0
kube-controller1
```

## Install kubectl

The [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl). command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Verification

Verify `kubectl` version 1.13.0 or higher is installed:

```
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T21:04:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
```

Without --client, you will get following error. 

```
The connection to the server localhost:8080 was refused - did you spe
cify the right host or port?
```
The reason is kubectl reads kube-apiserver and authentication information from ~/.kube/config by default. We will take care of issue in [Certificate Authority](04-certificate-authority.md)

## The Kubernetes Frontend Load Balancer

In this section you will provision an external load balancer to front the Kubernetes API Servers. The `kubernetes-the-hard-way` static IP address will be attached to the resulting load balancer.


### Provision a Network Load Balancer

```
#Install HAProxy
loadbalancer# sudo apt-get update && sudo apt-get install -y haproxy

```

```
loadbalancer# cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
frontend kubernetes
    bind 192.168.1.24:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server kube-controller0 192.168.1.25:6443 check fall 3 rise 2
    server kube-controller1 192.168.1.17:6443 check fall 3 rise 2
EOF
```

```
loadbalancer# sudo service haproxy restart
```

### Verification

Make a HTTP request for the Kubernetes version info:

```
curl  https://192.168.1.24:6443/version -k
```

> output

```
{
  "major": "1",
  "minor": "13",
  "gitVersion": "v1.13.0",
  "gitCommit": "ddf47ac13c1a9483ea035a79cd7c10005ff21a6d",
  "gitTreeState": "clean",
  "buildDate": "2018-12-03T20:56:12Z",
  "goVersion": "go1.11.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```


Next: [Certificate Authority](04-certificate-authority.md)
