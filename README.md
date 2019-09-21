# pi-k8s

> A Raspberry Pi Kubernetes Cluster 

<p align="center">
<img src="images/100.jpeg" width="800"> 
</p>

## Motivation 

Kubernetes (k8s) is an awesome tool. It is also a complex beast; fortunately, most of the complexity is hidden behind a few tools and commands. Developers have to deal only with the application logic, and this logic can be expressed on a nice, declarative way. 

Nowadays, it is possible to start experimenting with k8s either with a single node solution (such as [Minikube](https://github.com/kubernetes/minikube) or [microk8s](https://microk8s.io/)), or with one of the  fully-managed cluster solutions offered by major cloud providers. In the former case, the complexity of creating a cluster is non-existent (there is not cluster), and in the latter case is taken care of by the vendor. 

 Then why to bother setting up your own cluster? On the one hand, full control solutions (e.g. kubeadm, kubespray) are more flexible and provide finer tunning possibilities. Doing it on a [Raspberry Pi Cluster](https://github.com/twaclaw/pi-cluster) is just for the sake of learning and having fun. 

## Prerequisites 

* [pi-cluster](https://github.com/twaclaw/pi-cluster): this repository describes the necessary components to assemble a 5-node Raspberry Pi cluster with Raspbian images, and contains the [Ansible](https://www.ansible.com/) playbooks to carry out the initial configuration (i.e. ssh key deployment, adding users, etc.) 

My cluster is called "Macondo", and the nodes were correspondingly christened: `ursula`, `amaranta`, `pilar`, `remedios`, and `rebeca`. All nodes have a user `macondo` (e.g. `macondo@ursula`).

## Contents of this repo

This repo consists of the following directories:

* [ansible](./ansible): contains the Ansible playbooks to configure the devices, install dependencies, or execute commands
* [kubernetes](./kubernetes): an example k8s deployment to test the cluster

## Cluster Configuration 

[Highly available clusters](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) require at least 3 masters. I opted for a simpler configuration with 1 master and 4 nodes. 

The master node is `ursula` (of course). The scope of the different Ansible playbooks is defined via  different inventory files:

* [inventory.cfg](./ansible/inventory.cfg): all nodes
* [masters.cfg](./ansible/masters.cfg): control plane (i.e., master nodes, namely `ursula`)
* [nodes.cfg](./ansible/nodes.cfg): worker nodes (i.e.: `pilar`, `remedios`, `rebeca`, and `amaranta`)

---
## Setup

The following playbooks must be run from a computer within the same network as the cluster.

### Preparation

As indicated in [this post](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3), the memory control group must be enabled. 

```console
$ ansible-playbook playbooks/enable_mem_control_group.yml -i inventory.cfg  --user macondo --ask-become-pass
```

Nodes must be able to talk to each other, so let's do the appropriate introductions (I'm not quite positive though whether this step is necessary).

The following command appends the IP addresses of all nodes in the `/etc/hosts` file of each node.

```console
$ ansible-playbook playbooks/append_k8s_hostnames.yml -i inventory.cfg  --user macondo --ask-become-pass -e ansible_hostname
```

### Installing Kubernetes

The following command installs the required dependencies: Docker, kubelet, and kubeadm.

```console
$ ansible-playbook playbooks/install_k8s_deps.yml -i inventory.cfg  --user macondo --ask-become-pass
```



### Initializing the Cluster

Initialize the master:

```console
$ ansible-playbook playbooks/initialize_k8s_master.yml -i masters.cfg --user macondo --ask-become-pass
```

At this point, running the command `kubectl get nodes`  in the master node should return one node:

```console
macondo@ursula:~ $ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
ursula   NotReady   master   25m   v1.15.3
```

To initialize the nodes, the master issues a token, and the nodes join the cluster using that token. 

```console
$ join_command=$(ssh macondo@ursula "kubeadm token create --print-join-command")

$ ansible-playbook playbooks/join_cluster.yml -i nodes.cfg --user macondo --ask-become-pass -e "join_command=$join_command" 
```


At this point, running the command `kubectl get nodes`  in the master node should return all the nodes:

```console
macondo@ursula:~ $ kubectl get nodes
NAME       STATUS     ROLES    AGE   VERSION
amaranta   NotReady   <none>   27s   v1.15.3
pilar      NotReady   <none>   10m   v1.15.3
rebeca     NotReady   <none>   27s   v1.15.3
remedios   NotReady   <none>   27s   v1.15.3
ursula     NotReady   master   55m   v1.15.3
```

The nodes are up, but still not ready. This is because the [networking part](https://kubernetes.io/docs/concepts/cluster-administration/networking/) is missing. I used [flannel](https://github.com/coreos/flannel#flannel) as suggested in [this post](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3).

Run the following command in the master node:

```console
macondo@ursula:~ $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
```

The status should change to ready

```console
macondo@ursula:~ $ kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
amaranta   Ready    <none>   2m56s   v1.15.3
pilar      Ready    <none>   13m     v1.15.3
rebeca     Ready    <none>   2m56s   v1.15.3
remedios   Ready    <none>   2m56s   v1.15.3
ursula     Ready    master   58m     v1.15.3
```

Et voilà!

---

## k8s: Application Example

The whole idea of this exercise is to be able to deploy Kubernetes applications. The Manifests in the [kubernetes](./kubernetes) deploy an `nginx` image configured to print the pod and node names. 

It is a good practice to separate name spaces, so for the sake of being tidy I created a `test` namespace. 

```console
macondo@ursula:~ $  kubectl -n test apply namespace.yml
macondo@ursula:~ $  kubectl -n test apply deployment.yml
```

```console
macondo@ursula:~ $ kubectl get pod -o wide -n test
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx-f654b8fb-h2c5k   1/1     Running   0          73s   10.244.3.12   rebeca   <none>           <none>
```

The port in which the application is running can be by inspecting the service:

```console
macondo@ursula:~ $ kubectl -n test describe service nginx
```

In this case, the port number is 32015

The following command makes a request to the nginx application every 1s. 

```console
$ while true; do curl --silent ursula:32015 |grep NODE; sleep 1;done

NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
```

```console
macondo@ursula:~ $ kubectl scale --replicas 5 deployment/nginx -n test

macondo@ursula:~ $ kubectl get pod o-o wide -n test
```





```console
macondo@ursula:~ $ kubectl get pod -o wide -n test
NAME                   READY   STATUS            RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
nginx-f654b8fb-5cnjh   0/1     PodInitializing   0          7s     10.244.4.7    amaranta   <none>           <none>
nginx-f654b8fb-6666g   0/1     PodInitializing   0          7s     10.244.2.7    remedios   <none>           <none>
nginx-f654b8fb-h2c5k   1/1     Running           0          115s   10.244.3.12   rebeca     <none>           <none>
nginx-f654b8fb-kzv8w   0/1     PodInitializing   0          7s     10.244.3.13   rebeca     <none>           <none>
nginx-f654b8fb-prq4k   0/1     PodInitializing   0          7s     10.244.1.15   pilar      <none>           <none>
```

```console
$ while true; do curl --silent ursula:32015 |grep NODE; sleep 1;done

NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: pilar     POD: nginx-f654b8fb-prq4k
NODE: remedios  POD: nginx-f654b8fb-6666g
NODE: amaranta  POD: nginx-f654b8fb-5cnjh
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: remedios  POD: nginx-f654b8fb-6666g
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: rebeca    POD: nginx-f654b8fb-h2c5k
NODE: amaranta  POD: nginx-f654b8fb-5cnjh
NODE: amaranta  POD: nginx-f654b8fb-5cnjh
NODE: remedios  POD: nginx-f654b8fb-6666g
NODE: rebeca    POD: nginx-f654b8fb-kzv8w
NODE: amaranta  POD: nginx-f654b8fb-5cnjh
NODE: pilar     POD: nginx-f654b8fb-prq4k
NODE: remedios  POD: nginx-f654b8fb-6666g
```

## Conclusions

This exercise just scratched the surface of the potential of Kubernertes. Kubernetes is very useful to create multi-tier applications, scaling up/down automatically, and keeping the status. 

## Credits

- I adapted most of the Ansible scripts from [This article](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3) by [Eduard Iskandarov](https://itnext.io/@eduard.iskandarov). 
- Source image: [AbeBooks.com](https://www.abebooks.com/books/one-hundred-years-of-solitude-50th-anniversary/index.shtml)