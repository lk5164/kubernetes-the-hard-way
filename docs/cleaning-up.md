# Cleaning Up
## Stopping Services
```
for instance in `cat workers`; do
  sudo systemctl stop kubelet kube-proxy
done
```
```
for instance in `cat controllers`; do
  sudo systemctl stop kube-apiserver kube-controller-manager kube-scheduler etcd
done
```
## Removing Directory And Configuration Files
```
for instance in `cat workers`; do
  sudo rm -rf /var/lib/kubelet
  sudo rm /etc/systemd/system/kubelet.service
  sudo rm /etc/systemd/system/kube-proxy.service
  sudo rm /usr/local/bin/kubelet
  sudo rm /usr/local/bin/kube-proxy
  sudo rm -rf /var/lib/kubernetes
done
```
```
for instance in `cat controllers`; do
  sudo rm /etc/systemd/system/kube-apiserver.service
  sudo rm /etc/systemd/system/kube-controller-manager.service
  sudo rm /etc/systemd/system/kube-scheduler.service
  sudo rm /etc/systemd/system/etcd.service
  sudo rm /usr/local/bin/kube-apiserver
  sudo rm /usr/local/bin/kube-controller-manager
  sudo rm /usr/local/bin/kube-scheduler
  sudo rm /usr/local/bin/etcd
  sudo rm -rf /var/lib/kubernetes
done  
```
## Removing iptables
```
for instance in `cat workers && cat controllers`; do
  sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat &&
  sudo iptables -X -t nat
  ip link del weave
done
```
Next: [Troubleshooting](troubleshooting.md)

