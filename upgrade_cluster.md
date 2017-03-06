# Upgrading the Kubernetes Cluster 
### This is the part 2 of the Self-Hosted kubernetes Cluster lab

Using bootkube 0.3.7 deploys a v1.5.2 kubernetes cluster, let's upgrade to
v1.5.3, without taking down any application.

I based this guide on official CoreOS documentation [here](https://github.com/coreos/matchbox/blob/master/Documentation/bootkube-upgrades.md)

After you see this message with bootkube finishing
```
bootkube[5]: All self-hosted control plane components successfully started  
```

You should be able to get all the k8s components running as described on [part 1](https://github.com/yazpik/SelfHosted-K8s-Lab) of this document

Check current version deployed (server version)
```
root@selfhosted-k8s-lab:~/matchbox# kubectl version
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.3", GitCommit:"029c3a408176b55c30846f0faedf56aae5992e9b", GitTreeState:"clean", BuildDate:"2017-02-15T06:40:50Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.2+coreos.1", GitCommit:"3ed7d0f453a5517245d32a9c57c39b946e578821", GitTreeState:"clean", BuildDate:"2017-01-18T01:43:45Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
```

Let's create a deployment first to test the upgrade running an actual "aplication"

For this I'm going to use one of my public containers, running Nginx 1.11.5 on Alpine and create a 50 pods deployment
```
root@selfhosted-k8s-lab:~/matchbox# kubectl run --image quay.io/yazpik/spacemonkey:v4.0 --replicas 50 spacemonkey
deployment "spacemonkey" created
```
Now expose the "spacemonkey" deployment to a service basically as a simple LB
```
root@selfhosted-k8s-lab:~/matchbox# kubectl expose deployment spacemonkey --port 80 --external-ip 172.18.0.21 --name spacemonkey-svc
service "spacemonkey-svc" exposed
```
Service (spacemonkey-svc) exposed and working as a loadbalancer with 50 replicas
```
selfhosted-k8s-lab:~/matchbox# kubectl get ep
NAME              ENDPOINTS                                             AGE
kubernetes        172.18.0.21:443                                       8m
spacemonkey-svc   10.2.0.10:80,10.2.0.11:80,10.2.0.12:80 + 47 more...   3m
```

Show the control plane daemonsets and deployments which will need to be updated.
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get daemonsets -n kube-system
NAME                   DESIRED   CURRENT   READY     NODE-SELECTOR   AGE
checkpoint-installer   1         1         1         master=true     37m
kube-apiserver         1         1         1         master=true     37m
kube-flannel           3         3         3         <none>          37m
kube-proxy             3         3         3         <none>          37m
```
And deployments
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get deployments -n kube-system
NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-controller-manager   2         2         2            2           40m
kube-dns                  1         1         1            1           40m
kube-scheduler            2         2         2            2           40m
```

Label a new node as master, so we will have an HA kubernetes cluster
```
root@selfhosted-k8s-lab:~/matchbox# kubectl label node node2.example.com master=true
node "node2.example.com" labeled
```
After this step, another api-server container will be deployed on the new node labeled as master
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get pods -n kube-system -o wide | grep apiserver
kube-apiserver-1k9pc                       1/1       Running   2          1h        172.18.0.21   node1.example.com
kube-apiserver-p4w88                       1/1       Running   0          13m       172.18.0.22   node2.example.com
```


### kube-apiserver
As a best practice and avoid any error during images download, I think is a good idea
to fetch the required image version to all nodes before to attempt the upgrade

We are running here v1.5.2 and we are targeting 1.5.3, 

#### You can use your prefered method here, Ansible, SSH loop, or manual update
Basically what is required :
```
#Docker image for control plane components
sudo docker pull quay.io/coreos/hyperkube:v1.5.3_coreos.0  <--new image tag

#rkt image for kubelet and kubeproxy
#Trust the rkt hyperkube repo
sudo rkt trust --skip-fingerprint-review --prefix quay.io/coreos/hyperkube

sudo rkt fetch quay.io/coreos/hyperkube:v1.5.3_coreos.0
```
After this all the nodes should have the new image required to the upgrade (v1.5.3) for this case.

### So let's upgrade the cluster while we are running some workload to the service external IP
For this I'm going to use [boom](https://pypi.python.org/pypi/boom/1.0) 

Running 300 seconds with concurrency 111 users to the external service IP
```
root@selfhosted-k8s-lab:~/matchbox# boom http://172.18.0.21 -d 300 -c 111

```
#### IMPORTANT NOTE 
Daemonsets do not support rolling updates yet
It seems is going to be a feature of kubernetes 1.6
- https://github.com/kubernetes/features/issues?q=is%3Aopen+is%3Aissue+milestone%3Av1.6

Also see this proposal
- https://github.com/kubernetes/community/blob/master/contributors/design-proposals/daemonset-update.md

Lets edit the daemonset kube-apiserver manifest
```
$ kubectl edit daemonset kube-apiserver -n=kube-system
```
![edit-api-server-daemonset](https://cloud.githubusercontent.com/assets/7389339/23628820/3d4bd720-027b-11e7-8685-9af01b2b8f45.jpg)

Since daemonsets don't yet support rolling updates, we have to manually delete each apiserver pod for each to be re-scheduled.
```
root@selfhosted-k8s-lab:~/matchbox# kubectl delete pod kube-apiserver-p4w88 -n kube-system
pod "kube-apiserver-p4w88" deleted
```
As we have already the new image on the new, it was just a matter of seconds to create a new one
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get pods -n kube-system
kube-apiserver-pjvl9                       0/1       ContainerCreating   0          0s
root@selfhosted-k8s-lab:~/matchbox# kubectl get pods -n kube-system
kube-apiserver-pjvl9                       1/1       Running   0          16s
```
We have to do the same for the rest of the kube-apiserver pods running (for this example are just two)
```
root@selfhosted-k8s-lab:~/matchbox# kubectl delete pod kube-apiserver-1k9pc -n kube-system
```
Both apiservers are running with the new version
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get pods -n kube-system -o wide | grep api
kube-apiserver-ntggg                       1/1       Running   0          3m        172.18.0.21   node1.example.com
kube-apiserver-pjvl9                       1/1       Running   0          7m        172.18.0.22   node2.example.com
```
### Time to work on the kube-scheduler and kube-controller-manager

### kube-scheduler

Edit the scheduler deployment to rolling update the scheduler. Change the container image name for the new hyperkube version
```
$ kubectl edit deployments kube-scheduler -n=kube-system
```
Wait for the schduler to be deployed.

### kube-controller-manager

Edit the controller-manager deployment to rolling update the controller manager. 
```
$ kubectl edit deployments kube-controller-manager -n=kube-system
```
Wait for the controller manager to be deployed.

### Verify

At this point control plane was upgraded from v1.5.2 to v1.5.3
```
root@selfhosted-k8s-lab:~/matchbox# kubectl version
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.3", GitCommit:"029c3a408176b55c30846f0faedf56aae5992e9b", GitTreeState:"clean", BuildDate:"2017-02-15T06:40:50Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.3+coreos.0", GitCommit:"8fc95b64d0fe1608d0f6c788eaad2c004f31e7b7", GitTreeState:"clean", BuildDate:"2017-02-15T19:52:15Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
```
### Kubelet and Kubeproxy
#### Another important note here
Official CoreOS documentation, consider the kubelet as a daemonset, but is no longer deployed that way.
Is needed to edit to edit the kubelet systemd unit to use the next version in the envrionment file, it's a manual process and need to be done on all the nodes that runs the kubelet, see issue [#448](https://github.com/coreos/matchbox/issues/448) 

Check kubelet and kubeproxy running on hosts
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get nodes -o yaml | grep 'kubeletVersion\|kubeProxyVersion'
      kubeProxyVersion: v1.5.2+coreos.0
      kubeletVersion: v1.5.2+coreos.0
      kubeProxyVersion: v1.5.2+coreos.0
      kubeletVersion: v1.5.2+coreos.0
      kubeProxyVersion: v1.5.2+coreos.0
      kubeletVersion: v1.5.2+coreos.0
```

First edit the kubeproxy daemonset
```
kubectl edit daemonset kube-proxy -n=kube-system
```
And same process deleting running pods for this daemonset (kubeproxy)

### Update kubelet on nodes
On each node change this environment file
e.g for node3
```
core@node3 ~ $ sudo vim /etc/kubernetes/kubelet.env
#
KUBELET_ACI=quay.io/coreos/hyperkube
KUBELET_VERSION=v1.5.3_coreos.0
# change the kubelet version to the new one
```
Restart the kubelet (since we already fetched the image required, kubelet creation with the new image should be fairly quick)
```
core@node3 ~ $ sudo systemctl restart kubelet
```

Images available on node
```
core@node3 ~ $ rkt image list
ID                      NAME                                            SIZE    IMPORT TIME     LAST USED
sha512-4b541439adba     quay.io/coreos/hyperkube:v1.5.2_coreos.0        1.2GiB  2 hours ago     2 hours ago
sha512-c4c0341425c8     coreos.com/rkt/stage1-fly:1.18.0                17MiB   56 seconds ago  56 seconds ago
sha512-d17ee4d00002     quay.io/coreos/hyperkube:v1.5.3_coreos.0        1.2GiB  34 seconds ago  1 hour ago
```
After the kubelet restart, kubelet container was launched with the new image
```
core@node3 ~ $ rkt list
UUID            APP             IMAGE NAME                                      STATE   CREATED         STARTED         NETWORKS
8f05fe9b        hyperkube       quay.io/coreos/hyperkube:v1.5.3_coreos.0        running 1 minute ago    1 minute ago
```

Repeat the same on the rest of the nodes.

When it's done, verify with
```
root@selfhosted-k8s-lab:~/matchbox#  kubectl get nodes -o yaml | grep 'kubeletVersion\|kubeProxyVersion'
      kubeProxyVersion: v1.5.3+coreos.0
      kubeletVersion: v1.5.3+coreos.0
      kubeProxyVersion: v1.5.3+coreos.0
      kubeletVersion: v1.5.3+coreos.0
      kubeProxyVersion: v1.5.3+coreos.0
      kubeletVersion: v1.5.3+coreos.0
```

Do not forget of the http load we sent earlier for 5 minutes, with 111 concurrent users on 50 pods.

Not bad :D
```
root@selfhosted-k8s-lab:~# boom http://172.18.0.21 -d 300 -c 111
Server Software: nginx/1.11.5
Running GET http://172.18.0.21:80
Running for 300 seconds - concurrency 111.
Starting the load........................................................
-------- Results --------
Successful calls		91591
Total time        		300.0118 s
Average           		0.2366 s
Fastest           		0.1170 s
Slowest           		1.2103 s
Amplitude         		1.0933 s
Standard deviation		0.144113
RPS               		305
BSI              		Pretty good

-------- Status codes --------
Code 200          		91591 times.

-------- Legend --------
RPS: Request Per Second
BSI: Boom Speed Index
```


