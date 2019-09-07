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
        
        ansible-playbook playbooks/install_k8s_deps.yml -i inventory.cfg  --user macondo --ask-become-pass
        
        ansible-playbook playbooks/append_k8s_hostnames.yml -i inventory.cfg  --user macondo --ask-become-pass -e ansible_hostname

        ansible-playbook playbooks/enable_mem_control_group.yml -i inventory.cfg  --user macondo --ask-become-pass
        ```
    * [Optional] In addition, devices hostnames can be changed. This playbook has to be applied to each individual device, for instance:
        
        ```bash
        ansible-playbook playbooks/change_hostname.yml -i "172.16.0.178," --user macondo --ask-become-pass -e hostname=remedios 
        ```

    My cluster nodes are called: Ursula, Amaranta, Rebeca, Pilar and Remedios.

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