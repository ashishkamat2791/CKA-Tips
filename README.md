# CKA-Tips

## 1. Create alias/shortcuts for big commands

* Get resources
```
alias k=”kubectl”
alias kn=”kubectl get nodes -o wide”
alias kp=”kubectl get pods -o wide”
alias kd=”kubectl get deployment -o wide”
alias ks=”kubectl get svc -o wide”
```

* Describe K8S resources
```
alias kdp=”kubectl describe pod”
alias kdd=”kubectl describe deployment”
alias kds=”kubectl describe service”
alias kdn=”kubectl describe node”
```

## 2. Utilize kubectl comand line as much as possible

### Trick 1 : use command-line

**Generate pod yaml with below command**
```
kubectl run — generator=run-pod/v1 nginx — image=busybox --command sleep 3600 -o yaml — dry-run > nginx.yaml
```
**Generate deployment yaml with below command**
```
kubectl create deploy nginx — image=nginx — dry-run -o yaml > nginx-ds.yaml
```

**Generate service yaml with below command**
```
kubectl expose pod hello-world — type=NodePort — name=example-service

kubectl expose deployment hello-world — type=NodePort — name=example-service
```
**create role**
```
kubectl create role developer --resource=pods --verb=create,list,get,update,delete --namespace=development
```
**create role-binding**
```
kubectl create rolebinding developer-role-binding --role=developer --user=ash --namespace=development
```

**create config-map**

```
kubectl create configmap --from-literal=<key>=<value>
```

**create secrets**
```
kubectl create secrets generic my-secret --from-literal=<key>=<value>
**create cron-job**
```
**create cronjob**
```
kubectl create cronjob my-cron --image=busybox  --schedule="*/5 * * * *"
```
**create docker-regisrty-secret**
```
kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com
secret/private-reg-cred created
```
### Trick 2 : Resuse existing
```
cp pod1.ymal pod2.yaml
```

## 3. Do less score more with static pods

### Challenge #1 identify if its static POD question

* find keywords in questions like 'deployed only on master or particular node'
* Check whether directory mentioned is "/etc/kubernetes/manifests", as static pods definitions can be found here

### Challenge #2 Getting your YMAL Generator working

* once you SSH into any of the node, following command would not work
```
kubectl run — generator=run-pod/v1 nginx — image=nginx -o yaml — dry-run > nginx.yaml
```
* because kubeconfig on the specific node can’t connect to the cluster. 

* to overocome run above command in master nonde, copy the output to notepad
* Once logged in to node, paste it to file 

### Challenge #3  Make the node pick up the YAML

* To make the static pod working, kubelet configuration file should have “staticPodPath”. 
* Following steps will help to get the static pod up and running
  1. SSH to the node: “ssh my-node1”
  2. Gain admin privileges to the node: “sudo -i”
  3. Move to the manifest-path “cd /etc/kubernetes/manifests”
  4. Place the generated YAML into the folder “vi nginx.yaml”
  5. Find the kubelet config file path “ps -aux | grep kubelet” . 
  6. Edit the config file “vi /var/lib/kubelet/config.yaml” to add staticPodPath as highlighted below 
  7. Finally, restart the kubelet “systemctl restart kubelet”
   
## 4. ETCD - Jackpot
### Tip 1 Parts of backup command

```
ETCDCTL_API=3 etcdctl — endpoints=[ENDPOINT] — cacert=[CA CERT] — cert=[ETCD SERVER CERT] — key=[ETCD SERVER KEY] snapshot save [BACKUP FILE NAME]
```

The above instruction has 6 important parts to it
1. Command to take a backup — See Tip 2 on how to escape memorizing
2. ENDPOINT — See Tip 3 on how to get this value
3. CA CERT — See Tip 3 on how to get this value
4. ETCD SERVER CERT — See Tip 3 on how to get this value
5. ETCD SERVER KEY — See Tip 3 on how to get this value
6. BACKUP FILE NAME — This will be given as a part of question itself

### Tip 2 : Dont memorize

You will be allowed to refer to the Kubernetes documentation page during the exam. 
* From the Kubernetes documentation page (doc page) search for “etcd backup”,
* Look for the word “backup” in the resulting page, you will be able to locate the command for the backup.

You will get 
```
ETCDCTL_API=3 etcdctl — endpoints $ENDPOINT snapshot save snapshotdb
```
* run “ETCDCTL_API=3 etcdctl help” 



### Tip 3 Finding values

1. Exam cluster setup is done with kubeadm, this means ETCD used by the kubernetes cluster is coming from static pod. Confirm this by looking into pods in kube-system namespace.

```
kubectl get pod -n kube-system
```

2. Once you recognize the pod in kube-system namespace, just describe the pod to see command line options from container section.
```
kubectl describe pod etcd-master -n kube-system
```
You can locate the information on
* endpoint: — advertise-client-urls=https://172.17.0.15:2379
* ca certificate: — trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
* server certificate : — cert-file=/etc/kubernetes/pki/etcd/server.crt
* key: — key-file=/etc/kubernetes/pki/etcd/server.key

**Eg**
#### create snapshot and store to file
```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key snapshot status /tmp/etcd-backup.db
```

## Cluster upgrade form 1.16 to 1.17 or 1.17 to 1.18

**Eg: From 1.16.0 to 1.17.0**
### On Master
```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.17.0-00 && \
apt-mark hold kubeadm
```
choose the apporopriate kubeadm upgeade plan
```
 sudo kubeadm upgrade apply v1.17.0
 ```
 update kubelet kubectl and restart kubelet service
 ```
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.17.0-00 kubectl=1.17.0-00 && \
 apt-mark hold kubelet kubectl
sudo systemctl restart kubelet
```

### On Nodes
```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.17.0-00 && \
apt-mark hold kubeadm
```

update kubelet, kubectl and restart it

```
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.17.0-00 kubectl=1.17.0-00 && \
apt-mark hold kubelet kubectl
sudo systemctl restart kubelet
```

## Miscellenious

### vi terminal 
```
dd: delete current line
`n`dd: delete all 'n' lines from current
       Eg: 4dd - delete all 4 lines
:set nu : Enable numbering
: set paste: paste buffered/copied data to terminal
```

## networking commands
```
nslookup: to probe dns and find 
nc : to test connectivity
ping 
``
