# Deploying the Dashboard Add-on

Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.

## The Dashboard Add-on

Deploy the `dashboard` cluster add-on:

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

## Edit Configuration File

```
cat > dashboard-service.yaml <<EOF
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
EOF

kubectl apply -f ./dashboard-service.yaml
```

## Verification

Verify the deployment

```
kubectl get deployment kubernetes-dashboard -n kube-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1/1     1            1           19m
```
Verify the pod.
```
kubectl -n kube-system get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE            NOMINATED NODE   READINESS GATES
...
kubernetes-dashboard-57df4db6b-5mvtj   1/1     Running   0          20m     10.96.0.132    hadoop-slave4   <none>           <none>
...
```
Verify the service and note down the port number `30771`
```
kubectl get services kubernetes-dashboard -n kube-system
NAME                   TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.96.0.7    <none>        443:30771/TCP   21m
```
## Visit Dashboard Site

Since kubernetes-dashboard exposed NodePort, we can visit the site through
```
https://192.168.1.18:30771
or
https://192.168.1.23:30771
```
After accept risks, you may get following result on screen,

![kube-dashboard](https://raw.githubusercontent.com/lk5164/kubernetes-the-hard-way/master/images/kube-dashboard.PNG)

## Generate Token And Kubeconfig File
```
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}
```
You can use generated token to access dashboard. Or you can create a kubeconfig. 
```
LOADBALANCER_ADDRESS=192.168.1.24

{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_ADDRESS}:6443 \
    --kubeconfig=dashboard.kubeconfig

  kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=dashboard_user \
    --kubeconfig=dashboard.kubeconfig

  kubectl config use-context default --kubeconfig=dashboard.kubeconfig
}
```
Results
```
dashboard.kubeconfig
```
Select the file in your browser,

![main](https://raw.githubusercontent.com/lk5164/kubernetes-the-hard-way/master/images/dashboard-main.PNG)

Next: [Cleaning Up](docs/cleaning-up.md)
