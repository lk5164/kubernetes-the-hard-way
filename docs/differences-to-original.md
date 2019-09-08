# Differences between original and this solution

Platform: I use Esxi 6.7 to setup a local cluster, the original one uses GCP

Nodes: 2 Master and 2 Worker vs 2 Master and 3 Worker nodes

Configure 1 worker node normally
and the second one with TLS bootstrap

Node Names: I use kube-worker0 kube-worker1 instead of worker-0 worker-1

IP Addresses: I use statically assigned IPs on private network

Certificate File Names: I use <name>.crt for public certificate and <name>.key for private key file. Whereas original one uses <name>-.pem for certificate and <name>-key.pem for private key.

I generate separate certificates for etcd-server instead of using kube-apiserver

Network:
We use weavenet
