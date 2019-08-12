# Install Kubernetes (k8s) on ubuntu 18.04 LTS

# Install docker
```
Get the Docker gpg key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Add the Docker repository:
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

```
# Install k8s
```
Get the Kubernetes gpg key:
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

Add the Kubernetes repository:
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update

Install Docker, kubelet, kubeadm, and kubectl:
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00
sudo apt-mark hold docker-ce kubelet kubeadm kubectl

```

# Bootstrapping the Cluster

On the Kube master node, initialize the cluster:
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
You will have something like below, copy it to run on the worker nodes later
```
kubeadm join 10.0.1.198:6443 --token 04bgqi.c4c6rhek05av6opq --discovery-token-ca-cert-hash sha256:41b25121c4b7ef85db01da2a057f262f2e745dee625358a5f9c4f4ff81d30e29
```
<i>Note that yours will be different then mine above</i><br>
When it is done, set up the local kubeconfig:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify that the cluster is responsive and that Kubectl is working:
```
kubectl version
```
On worker nodes:
```
sudo kubeadm join 10.0.1.198:6443 --token 04bgqi.c4c6rhek05av6opq --discovery-token-ca-cert-hash sha256:41b25121c4b7ef85db01da2a057f262f2e745dee625358a5f9c4f4ff81d30e29
```
<i>Note that yours will be different then mine above</i><br>

# Configuring Networking with Flannel
On all three nodes and master node, run the following:
```
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
On master node:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
kubectl get nodes
```

# Test a simple nginx pod
```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```
Verify:
```
kubectl get pods
kubectl describe pod nginx
kubectl delete pod nginx
```
# Deploying the Dashboard UI
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```
create dashboard-adminuser.yaml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
Then apply
```
kubectl apply -f dashboard-adminuser.yaml
```
create rbac.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

```
Then apply
```
kubectl apply -f rbac.yaml
```
Get the token:
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
You will see something like this:
```
Name:         admin-user-token-6sbkp
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 9129c5c1-bd12-11e9-8a24-062933c5a0bc

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZzYmtwIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5MTI5YzVjMS1iZDEyLTExZTktOGEyNC0wNjI5MzNjNWEwYmMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.RPJ7fCmr91mTIpy-vT0-4xaNTg6KRVZSexnLb_UnrG4caU20iVxcQXuCAfHShPRQcrLRGDYBeKvJFCqG5VViJAwed0PoT7t1HimAxVE6Ke5Jc0BZQV5g6JRQXKggOwwhtoFtk4Mtt3h9D_QP6kWwZXLl1QYEfwtGDiHespntLiOh_6hCjyST948n1-_BiZsN0NPr9RtdmnTnt87B1k4OpKxcOfFFCCzINslTf9FDxDB8jhaiP3HqPPlW1r6fqB8dC7kFgGSC1ryx3pWPV-NfSZXTiS8TSKmPfDOXsccilBMGPpdZ7RUcj617TQp6HNxbCBKqi1j5ZbF40E0F5FM1WA

```
Copy the token, you will need it when accessing the dashboard later on.<br>
Edit the service:
```
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```
You will see something like this:
```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: 2019-07-09T04:56:47Z
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "12322"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard
  uid: f49e564a-a205-11e9-9cfe-0e65addbd148
spec:
  clusterIP: 10.107.214.6
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30760
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
Change Type from <b>ClusterIP</b> to <b>NodePort</b> then save it. It will apply new setting automatically.<br>

Find the port:
```
kubectl -n kubernetes-dashboard get service kubernetes-dashboard
```
You will see something like this:
```
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.104.193.45   <none>        443:30182/TCP   5m10s
```

Find the correct node running Dashboard (sometime it isn't a Master Node)
```
kubectl get nodes --all-namespaces
```
```
NAME            STATUS   ROLES    AGE     VERSION
ip-10-0-1-154   Ready    <none>   5m14s   v1.13.5
ip-10-0-1-198   Ready    master   6m6s    v1.13.5
ip-10-0-1-42    Ready    <none>   5m11s   v1.13.5
ip-10-0-1-60    Ready    <none>   5m23s   v1.13.5
```
```
kubectl get pods --all-namespaces -o wide
```
You will see something like this:
```
NAMESPACE              NAME                                          READY   STATUS    RESTARTS   AGE     IP           NODE            NOMINATED NODE   READINESS GATES
kube-system            coredns-54ff9cd656-gf5x5                      1/1     Running   0          6m1s    10.244.0.2   ip-10-0-1-198   <none>           <none>
kube-system            coredns-54ff9cd656-xjr7g                      1/1     Running   0          6m1s    10.244.0.3   ip-10-0-1-198   <none>           <none>
kube-system            etcd-ip-10-0-1-198                            1/1     Running   0          5m23s   10.0.1.198   ip-10-0-1-198   <none>           <none>
kube-system            kube-apiserver-ip-10-0-1-198                  1/1     Running   0          5m11s   10.0.1.198   ip-10-0-1-198   <none>           <none>
kube-system            kube-controller-manager-ip-10-0-1-198         1/1     Running   0          5m26s   10.0.1.198   ip-10-0-1-198   <none>           <none>
kube-system            kube-flannel-ds-amd64-nftfm                   1/1     Running   0          6m1s    10.0.1.198   ip-10-0-1-198   <none>           <none>
kube-system            kube-flannel-ds-amd64-p5lsg                   1/1     Running   0          5m25s   10.0.1.42    ip-10-0-1-42    <none>           <none>
kube-system            kube-flannel-ds-amd64-tnmdf                   1/1     Running   0          5m37s   10.0.1.60    ip-10-0-1-60    <none>           <none>
kube-system            kube-flannel-ds-amd64-tnzxp                   1/1     Running   1          5m28s   10.0.1.154   ip-10-0-1-154   <none>           <none>
kube-system            kube-proxy-284kn                              1/1     Running   0          6m1s    10.0.1.198   ip-10-0-1-198   <none>           <none>
kube-system            kube-proxy-8nvl4                              1/1     Running   0          5m25s   10.0.1.42    ip-10-0-1-42    <none>           <none>
kube-system            kube-proxy-kzxv2                              1/1     Running   0          5m37s   10.0.1.60    ip-10-0-1-60    <none>           <none>
kube-system            kube-proxy-rfmpr                              1/1     Running   0          5m28s   10.0.1.154   ip-10-0-1-154   <none>           <none>
kube-system            kube-scheduler-ip-10-0-1-198                  1/1     Running   0          5m18s   10.0.1.198   ip-10-0-1-198   <none>           <none>
kubernetes-dashboard   kubernetes-dashboard-7648564c7b-9wv8g         1/1     Running   0          4m      10.244.1.2   ip-10-0-1-60    <none>           <none>
kubernetes-dashboard   kubernetes-metrics-scraper-55bd7b7d95-zg77l   1/1     Running   0          4m      10.244.2.2   ip-10-0-1-154   <none>           <none>
```

In this case, the Dashboard is running on Worker node with Private IP "10.0.1.60". Then connect to the dashboard from browser using the IP and the port (30182 in this case):
```
https://<IP>:<port>   --> https://10.0.1.60:30182
```
# Installing helm
Helm is a package manager for Kubernetes that allows developers and operators to more easily configure and deploy applications on Kubernetes clusters.

```
sudo snap install helm --classic
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account=tiller
```

