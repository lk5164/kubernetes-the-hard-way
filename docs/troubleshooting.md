# Troubleshooting
## Viewing Certificate Content
```
openssl x509 -noout -text -in <cert name>
```
## Viewing Kubeconfig Content
```
kubectl config view --kube-config=<kube filename>
```
## Checking Recent Logs
```
journalctl -xe
```
Usually, the log includes information about errors and which component generates this error. Then ssh to that component node server and run, 
```
sudo systemctl status <component name>
```
## Pod Troubleshooting
```
kubectl logs <pod-name> -c <init-container-2>
```
## Service Troubleshooting
[Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

## Jenkins Troubleshooting
[Setting Up Jenkins On Kubernetes](deploy-jenkins.md#Troubleshooting)
