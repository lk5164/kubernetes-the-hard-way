# Setting Up Jenkins On Kubernetes
## Install Helm
```
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz

tar zxfv helm-v2.14.1-linux-amd64.tar.gz
sudo chmod +x linux-amd64/helm
sudo cp linux-amd64/helm /usr/local/bin/
```
Creating RBAC roles for cluster admin
```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin \
```
Grant Tiller cluster-admin role in your cluster.
```
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin \
    --serviceaccount=kube-system:tiller
```
Init helm with tiller service account and update repository
```
./helm init --service-account=tiller
./helm repo update
```
Verifying helm
```
./helm version
```
> output
```
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```
Creating Persistent Volumes and Persistent Volumes Claim
```
cat <<EOF | tee kubernetes-jenkins-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity: 
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /opt/jenkins
EOF    
```
```
kubectl apply -f ./kubernetes-jenkins-pv.yaml
```
```
cat <<EOF | tee kubernetes-jenkins-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF      
```
```
kubectl apply -f kubernetes-jenkins-pvc.yaml
```
Customizing values.yaml to override default [values.yaml](https://github.com/helm/helm/blob/master/docs/chart_template_guide/values_files.md) file.
```
cat <<EOF | tee jenkins-values.yaml
master:
  installPlugins:
    - kubernetes:1.13.0
    - workflow-multibranch:latest
    - workflow-aggregator:latest
    - workflow-job:latest
    - credentials-binding:latest
    - git:latest
    - blueocean:latest
  resources:
    requests:
      cpu: "50m"
      memory: "1024Mi"
    limits:
      cpu: "1"
      memory: "3500Mi"
  javaOpts: "-Xms3500m -Xmx3500m"
  serviceType: ClusterIP
agent:
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
serviceAccount:
  name: kube-jenkins
persistence:
  existingClaim: jenkins-pvc
EOF  
```
## Deploying Jenkins
```
helm install -n kubernetes stable/jenkins -f jenkins-values.yaml --wait
```
Make sure the helm status is DEPLOYED
```
helm list
NAME      	REVISION	UPDATED                 	STATUS  	CHART        	APP VERSION	NAMESPACE
kubernetes	1       	Sun Sep 22 16:35:44 2019	DEPLOYED	jenkins-1.7.3	lts        	default  
```
Modify jenkins service to NodePort type, so that jenkins can be accessed outside the cluster.
```
cat <<EOF | tee kubernetes-jenkins-svc.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: kube
    app.kubernetes.io/managed-by: Tiller
    app.kubernetes.io/name: jenkins
    helm.sh/chart: jenkins-1.7.3
  name: kubernetes-jenkins
  namespace: default
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30000
  selector:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: kubernetes
  sessionAffinity: None
  type: NodePort
EOF
```
```
kubectl apply -f kubernetes-jenkins-svc.yaml
```
## Get password for login
```
printf $(kubectl get secret kubernetes-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```
## Verification
```
kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
kubernetes-jenkins-5ff54c7c88-sgvzg   1/1     Running   0          2m
```
```
kubectl get svc
NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
...
kubernetes-jenkins         NodePort    10.96.0.26    <none>        8080:30000/TCP   57m
kubernetes-jenkins-agent   ClusterIP   10.96.0.238   <none>        50000/TCP        57m
```
Now open your browser and visit jenkins dashboard with the node IP address.

![jenkins screenshot](https://raw.githubusercontent.com/lk5164/kubernetes-the-hard-way/master/images/jenkins-login.PNG)
## Troubleshooting
If deploying jenkins is not successful. redeploy with `helm upgrade`.
```
helm upgrade -n kube stable/jenkins -f jenkins-values.yaml --wait
```
If jenkins pod is not running, check the log.
```
kubectl logs kubernetes-jenkins-5ff54c7c88-sgvzg -c jenkins
```
If you don't have DNS in your cluster or DNS setting up is not correct, helm plugins will not be able to install. You may find following information in the log.
```
Downloading plugins...
...
16:28:13 Failure (28) Retrying in 1 seconds...
```
You can check if your dns is running or follow the steps to [debug service](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

Another option is to put an emply list under master.installPlugins in jenkins-values.yaml file. Then use `helm upgrade` to reploy jenkins. 
