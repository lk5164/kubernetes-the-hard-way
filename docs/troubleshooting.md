# Troubleshooting
## Viewing Certificate Content
```
openssl x509 -noout -text -in <cert name>
```
## Checking Recent Logs
```
journalctl -xe
```
Usually, the log includes information about errors and which component generates this error. Then ssh to that component node server and run, 
```
sudo systemctl status <component name>
```
## Pod troubleshooting
```
kubectl logs <pod-name> -c <init-container-2>
```
## Service troubleshooting
[Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

## Jenkins troubleshooting
[Setting Up Jenkins On Kubernetes](deploy-jenkins.md#Troubleshooting)
