# Macondo: Yet Another Raspberry Pi Cluster

<p align="center">
<img src="images/raspberries.jpg" width="800"> 
</p>

There are some practical reasons motivating this exercise;
however, the main one is just for fun... Why not? 

<!-- <div style="float: right">
<img src="./images/cluster.jpg"  height="250">
</div>
-->


## Setup

### Bootstrap and Configuration

The SD cards must be flashed (I used [raspbian](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)),
ssh enabled (just create an empty file called "ssh" in the boot partition: `touch /mount-point/boot/ssh`), and the cluster powered up (of course) before starting with the configuration. I used Ansible to configure the devices. By using Ansible, most of the configuration can be done simultaneously on all the devices.

* `nmap` or similar can be used to discover the devices IP addresses (my network IP address is 172.16.0.0/24).  The IP addresses can be listed in an Ansible [inventory.cfg](ansible/inventory.cfg).

    ```bash
    sudo nmap -sn 172.16.0.0-255 |grep rasp -i  -B 2
    ```
* The ansible playbooks are located in the ansible folder ( `cd ansible` ) and support the following tasks:

    * Creating a new user (e.g. `macondo`), deploying an ssh public key, and, finally, deleting the old user `pi`:
    
        ```bash
        ansible-playbook playbooks/deploy_keys.yml -i inventory.cfg --user macondo --ask-pass  -e ssh_key=FULL_PATH_TO_ID_RSA_PUB 

        ansible-playbook playbooks/enable_mem_control_group.yml -i inventory.cfg  --user macondo --ask-become-pass

        ansible-playbook playbooks/append_k8s_hostnames.yml -i inventory.cfg  --user macondo --ask-become-pass -e ansible_hostname

        ansible-playbook playbooks/install_k8s_deps.yml -i inventory.cfg  --user macondo --ask-become-pass
        ```

     At this point `kubectl get nodes` should return one node
     ```shell
        macondo@ursula:~ $ kubectl get nodes
        NAME     STATUS     ROLES    AGE   VERSION
        ursula   NotReady   master   25m   v1.15.3
    ```
    * call 
    ```
        kubeadm token create --print-join-command
    ```
    * pass the result as an argument to 
    ```shell
         join_command=$(ssh macondo@ursula "kubeadm token create --print-join-command")
         
         ansible-playbook playbooks/join_cluster.yml -i nodes.cfg --user macondo --ask-become-pass -e "join_command=$join_command" 

         macondo@ursula:~ $ kubectl get nodes
            NAME       STATUS     ROLES    AGE   VERSION
            amaranta   NotReady   <none>   27s   v1.15.3
            pilar      NotReady   <none>   10m   v1.15.3
            rebeca     NotReady   <none>   27s   v1.15.3
            remedios   NotReady   <none>   27s   v1.15.3
            ursula     NotReady   master   55m   v1.15.3
    ```

    In the master run

    ```
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
    ```

    The status should change to ready

    ```
    macondo@ursula:~ $ kubectl get nodes
    NAME       STATUS   ROLES    AGE     VERSION
    amaranta   Ready    <none>   2m56s   v1.15.3
    pilar      Ready    <none>   13m     v1.15.3
    rebeca     Ready    <none>   2m56s   v1.15.3
    remedios   Ready    <none>   2m56s   v1.15.3
    ursula     Ready    master   58m     v1.15.3

    ```
    
    ```shell
    macondo@ursula:~ $ kubectl get pods -n "kube-system"
    NAME                             READY   STATUS    RESTARTS   AGE
    coredns-5c98db65d4-26sxz         1/1     Running   1          63m
    coredns-5c98db65d4-q6n67         1/1     Running   1          63m
    etcd-ursula                      1/1     Running   3          62m
    kube-apiserver-ursula            1/1     Running   3          63m
    kube-controller-manager-ursula   1/1     Running   3          63m
    kube-flannel-ds-arm-5dfxz        1/1     Running   1          5m55s
    kube-flannel-ds-arm-67sl6        1/1     Running   1          5m55s
    kube-flannel-ds-arm-7ndxx        1/1     Running   1          5m55s
    kube-flannel-ds-arm-fmbvn        1/1     Running   1          5m55s
    kube-flannel-ds-arm-mrgjs        1/1     Running   2          5m55s
    kube-proxy-4dsrv                 1/1     Running   1          8m13s
    kube-proxy-9t8r9                 1/1     Running   1          8m13s
    kube-proxy-cfnxf                 1/1     Running   3          63m
    kube-proxy-lwjcc                 1/1     Running   1          8m13s
    kube-proxy-wmt5v                 1/1     Running   1          18m
    kube-scheduler-ursula            1/1     Running   3          62m

    ```

    * [Optional] In addition, devices hostnames can be changed. This playbook has to be applied to each individual device, for instance:
        
        ```bash
        ansible-playbook playbooks/change_hostname.yml -i "172.16.0.178," --user macondo --ask-become-pass -e hostname=remedios 
        ```

    My cluster nodes are called: Ursula, Amaranta, Rebeca, Pilar and Remedios.


## Kubernetes

```console
    kubectl -n test apply namespace.yml
    kubectl -n test apply deployment.yml


    while true; do curl --silent ursula:31151  |grep node; sleep 1;done

    kubectl scale --replicas 5 deployment/nginx -n test

    kubectl get pod o-o wide -n test

```
```
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
macondo@ursula:~ $ kubectl get pod -o wide -n test
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx-f654b8fb-h2c5k   1/1     Running   0          73s   10.244.3.12   rebeca   <none>           <none>
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
$     while true; do curl --silent ursula:32015 |grep NODE; sleep 1;done
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

## Additional Ansible Scripts

### Shutdown and Reboot
```bash
ansible-playbook playbooks/shutdown.yml -i inventory.cfg --user macondo --ask-become-pass

ansible-playbook playbooks/reboot.yml -i inventory.cfg --user macondo --ask-become-pass
```

### Adhoc commands

For instance:
```bash
ansible all -m ping -i inventory.cfg -u macondo

ansible all -m apt -a 'name=kubeadm state=absent purge=yes autoremove=yes' --become -i inventory.cfg  --ask-become-pass -u macondo


```

## Credits
- https://github.com/garthvh/ansible-raspi-playbooks
- https://github.com/vicchi/ansible-pi-lockdown
- [Image source](https://www.cuisineaz.com/recettes/tartelettes-aux-framboises-a-la-creme-de-mascarpone-2922.aspx)
