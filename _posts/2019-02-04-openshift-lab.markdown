---
layout: post
title:  "Install Openshift on a Lab Server"
date:   2019-02-04 16:00:00 -0400
categories: fuse openshift lab ansible
---

# Introduction 
This page details the steps I took to create a RHEL Openshift lab environment for testing various middleware containers.  The goal is to create a reproducible environment that will allow for testing and troubleshooting of various combinations of Fuse, AMQ, Kafka, and more.

# Environment
The hardware for the lab server is a single SuperMicro E200-8D.  This was chosen for a combination of performance and power efficiency.  The server will host at least four RHEL virtual guests, including a DNS server, a master openshift node, and two compute nodes.  

Note also that single-developer machines can use minishift to quickly get an openshift environment up and running; this exercise is to install multiple nodes for pedagogical and troubleshooting purposes.

```
SuperMicro E200-8D: Intel Xeon D-1528 (6-Core, 12 Threads), 32G DDR4, 512GB M.2 SSD, RHEL 7 - 192.168.1.113
|--RHEL 7 KVM Guest - dns 2GB RAM / 16GB disk - 192.168.1.114
|--RHEL 7 KVM Guest - master 8GB RAM / 80GB disk - 192.168.1.115
|--RHEL 7 KVM Guest - node1 8GB RAM / 80GB disk - 192.168.1.116
+--RHEL 7 KVM Guest - node2 8GB RAM / 80GB disk - 192.168.1.117
```

My local network is 192.168.1.0/24.

# Set up KVM host
The first step os to set up a base system on top of which the Openshift nodes will be installed.  The base system has few requirements; the following is an outline of the repos that are enabled on my base system and the packages I installed.
1.  Install rhel 7.x minimal on SuperMicro (bare metal)
2.  Configure network interface for static ip (/etc/sysconfig/network-scripts)
    * 192.168.1.113
3.  set hostname
    * kvm.lan
4.  Add rhel subscription [^1]
5.  Enable openshift repos on kvm.lan
```
    subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"
```
6.  Update all packages
7.  Install kvm, virtlib, ansible, etc:
```
    yum install virt-install libvirt-python virt-manager \
       virt-install libvirt-client wget git net-tools bind-utils \
       yum-utils iptables-services bridge-utils bash-completion \
       kexec-tools sos psacct ansible openshift-ansible
```
8.  Create storage pool for images
```
       install -d -o qemu -g qemu -m 700 /home/openshift_images
       virsh pool-define-as --name openshift_images --type dir --target /home/openshift_images
       virsh pool-autostart openshift_images
       virsh pool-start openshift_images
```
9.  Clone scripts from github[^2].


# Edit Ansible hosts file
Edit the ${SCRIPTS}/ansible/hosts-ose-install for appropriate hosts/IP addresses, etc.

# Set up secret data for ansible scripts
Secret data for the ansible scripts are stored in the ${SCRIPTS}/ansible/group_vars/all/vault.yml file.

```
ansible-vault create group_vars/all/vault.yml
```

The contents of the vault need to contain the following attributes (add appropriate values).  The pool is a valid pool id for the subscription_user:

```
secret_ansible_ssh_pass:
secret_subscription_user:
secret_subscription_password:
secret_subscription_pool:
```

# Create dns server
The DNS server could be externalized to another server or network device.  I did find that installing the DNS server on a separate VM reduced problems during install of the openshift environment.  It's not strictly necessary to host the DNS on it's own VM; it could be co-located with the master server, for instance. From ${SCRIPTS}/kickstart directory, run the following:

```
./install-node.sh -u myuser -s dns -i 192.168.1.114 -m 255.255.255.0 -g 192.168.1.1 -n 192.168.1.1 -p openshift_images -x 2048 -y 16 -z 1
```

*N.B. - the prompted password will be used for both the specified user and root*


# Create ssh key for root user on kvm host
The openshift deployer requires passwordless access to the openshift nodes.  I set up a ssh key for the kvm host's root user; that key is distributed to the nodes in the next steps.

```
ssh-keygen -t rsa
```

# Set up dns server
The DNS server can now be configured via the ansible script in the ${SCRIPTS}/ansible directory.

```
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i hosts-ose-install dns-server.yml --vault-id @prompt
```

# Create openshift servers
Use the same script that was used to create the dns server to create the remaining nodes.  An alternative would be create one node and clone it.

```
./install-node.sh -u myuser -s master -i 192.168.1.115 -m 255.255.255.0 -g 192.168.1.1 -n 192.168.1.114 -p openshift_images -x 8192 -y 96 -z 4
./install-node.sh -u myuser -s node1 -i 192.168.1.116 -m 255.255.255.0 -g 192.168.1.1 -n 192.168.1.114 -p openshift_images -x 8192 -y 96 -z 4
./install-node.sh -u myuser -s node2 -i 192.168.1.117 -m 255.255.255.0 -g 192.168.1.1 -n 192.168.1.114 -p openshift_images -x 8192 -y 96 -z 4
```

# Configure openshift servers using custom ansible scripts
This custom script sets up some prerequisites before using the openshift-ansible scripts.  In particular, repos are set up, nested virtualization is enabled, useful packages are installed, hostnames are assigned, and root's public ssh key is distributed to the nodes. Run the following from the ${SCRIPTS}/ansible directory:
```
ansible-playbook -i hosts-ose-install --vault-id @prompt openshift-prerequisites.yml
```

# Install openshift proper

```
cd /usr/share/ansible/openshift-ansible/
ansible-playbook -i ${SCRIPTS}/ansible/hosts-ose-install playbooks/prerequisites.yml --vault-id @prompt
```

*Important! Restart all openshift servers- without a restart of openshift servers, dbus/dnsmasq issues are encountered during installation of cluster.  Make sure the following resolves on each node: nslookup cdn.redhat.com*

```
ansible-playbook -i ${SCRIPTS}/ansible/hosts-ose-install playbooks/deploy_cluster.yml --vault-id @prompt
```

Note: this step takes a long time - 45 mins on my machine!

# Access openshift

Log in to the master node, run
```
oc login -u system:admin
oc get nodes
NAME                     STATUS    ROLES     AGE       VERSION
master.openshift.local   Ready     master    21m       v1.11.0+d4cacc0
node1.openshift.local    Ready     compute   8m        v1.11.0+d4cacc0
node2.openshift.local    Ready     infra     8m        v1.11.0+d4cacc0
```

Configure authentication as described in the Openshift Docs[^3]
```
provider:
      apiVersion: v1
      kind: HTPasswdPasswordIdentityProvider
      file: /etc/origin/master/htpasswd
```

```
touch /etc/origin/master/htpasswd
htpasswd -b /etc/origin/master/htpasswd admin redhat
master-restart api
master-restart controllers
oc adm policy add-cluster-role-to-user cluster-admin admin
oc login -u admin
oc project default
```

The web console can now be accessed: https://master.openshift.local:8443/console/catalog



# References
[^1]:[RHEL Registration]( https://access.redhat.com/labs/registrationassistant/)
[^2]:[Custom Scripts](https://github.com/sjhiggs/lab-openshift-install)
[^3]:[OpenShift Config](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html/getting_started/getting-started-configure-openshift)
