## Upgrade the Cluster 
### This is the part 2 of the Self-Hosted kubernetes Cluster lab

Using bootkube 0.3.7 deploys a v1.5.2 kubernetes cluster, let's upgrade to
v1.5.3, without taking down any application.

I based this guide on official CoreOS documentation here
https://github.com/coreos/matchbox/blob/master/Documentation/bootkube-upgrades.md

After you see this message with bootkube finishing
```
bootkube[5]: All self-hosted control plane components successfully started  
```

You should be able to get all the k8s components running as described on part 1 of this document

Check current version deployed (server version)
```
root@selfhosted-k8s-lab:~/matchbox# kubectl version
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.3", GitCommit:"029c3a408176b55c30846f0faedf56aae5992e9b", GitTreeState:"clean", BuildDate:"2017-02-15T06:40:50Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.2+coreos.1", GitCommit:"3ed7d0f453a5517245d32a9c57c39b946e578821", GitTreeState:"clean", BuildDate:"2017-01-18T01:43:45Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
```

Let's create a deployment first to test the upgrade running an actual "aplication"
For this I'm going to use one of my containers, running nginx 1.11.5 on Alpine.
```
root@selfhosted-k8s-lab:~/matchbox# kubectl run --image quay.io/yazpik/spacemonkey:v4.0 --replicas 50 spacemonkey
deployment "spacemonkey" created
```
Now expose the deployment to a service basically as a simple LB
```
root@selfhosted-k8s-lab:~/matchbox# kubectl expose deployment spacemonkey --port 80 --external-ip 172.18.0.21 --name spacemonkey-svc
service "spacemonkey-svc" exposed
```
Service exposed and working as a loadbalancer
```
selfhosted-k8s-lab:~/matchbox# kubectl get ep
NAME              ENDPOINTS                                             AGE
kubernetes        172.18.0.21:443                                       8m
spacemonkey-svc   10.2.0.10:80,10.2.0.11:80,10.2.0.12:80 + 47 more...   3m
```

And we need to generate actual traffic to the app, I'm going to use [boom](https://pypi.python.org/pypi/boom/1.0) for this 
for e.g 

Running 120 seconds with concurrency 111 users to the external service IP
```
root@selfhosted-k8s-lab:~/matchbox# boom http://172.18.0.21 -d 120 -c 111

```

##### So let's upgrade the cluster while we are running some workload to the service external IP

Show the control plane daemonsets and deployments which will need to be updated.
```
root@selfhosted-k8s-lab:~/matchbox# kubectl get daemonsets -n kube-system
NAME                   DESIRED   CURRENT   READY     NODE-SELECTOR   AGE
checkpoint-installer   1         1         1         master=true     37m
kube-apiserver         1         1         1         master=true     37m
kube-flannel           3         3         3         <none>          37m
kube-proxy             3         3         3         <none>          37m
```

Get deployments
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
#### kube-apiserver
As a best practice and avoid any error during images download, I think is a good idea
to fetch the required image version to all nodes before to attempt the upgrade

We are running here v1.5.2 and we are targeting 1.5.3, 

##### On node1, existing images

#Docker image
sudo docker pull quay.io/coreos/hyperkube:v1.5.3_coreos.0

#Trust the rkt hyperkube repo
sudo rkt trust --skip-fingerprint-review --prefix quay.io/coreos/hyperkube
sudo rkt fetch quay.io/coreos/hyperkube:v1.5.3_coreos.0

```


Daemonsets do not support rolling updates yet
It seems is going to be a feature of kubernetes 1.6
https://github.com/kubernetes/features/issues?q=is%3Aopen+is%3Aissue+milestone%3Av1.6

And also this proposal
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/daemonset-update.md


$ kubectl edit daemonset kube-apiserver -n=kube-system
Since daemonsets don't yet support rolling, manually delete each apiserver one by one and wait for each to be re-scheduled.

