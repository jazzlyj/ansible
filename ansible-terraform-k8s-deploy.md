# Using Ansible, Terraform, KVM, and Kubernetes to create an on-premises Kubernetes cluster

In this post, we use the server automation tools Ansible, Terraform, Docker, and Kubernetes to create and configure virtual machines (VMs) to host an on-premises Kubernetes cluster.

* Understanding infrastructure
Infrastructure refers to computing resources used to store, transform, and exchange data. A new approach to software development called DevOps deploys applications across distributed systems consisting of multiple physical and virtual machines.

* Understanding DevOps
DevOps is an approach to software development and system administration that views system administration as a task to be automated, so that software developers are not dependent on the services of a system administrator when they deploy software to server infrastructure.
DevOps tools automate the creation and deployment of servers to create a distributed software infrastructure on which software can be deployed and run on multiple computers whether physical or virtual.

* Understanding the difference between cloud and on-premises (“onprem”) virtualization servers
The term “cloud” refers to computing services that are offsite, outsourced, and virtualized. These cloud services are provided by companies including Amazon AWS, Google GCP, Microsoft Azure, and Digital Ocean.

The term “On-premises” (“onprem”) refers to computing services that are onsite, in-house, and virtualized. Onprem virtualization servers can provide the same software environment as cloud providers. Onprem server infrastructure can be used to develop and test new software before it is deployed to a public cloud.

# Overview of the system to be constructed

* Understanding the virtualization server:

    Virtual machines require a physical machine containing processors, memory, and storage. For this exercise, we will reformat a circa 2015 laptop (i7-4712HQ, 16GB RAM, 1TB SSD) with Ubuntu Linux 22.04 LTS as an on-premises virtualization host.
<br>
<br>
* Understanding KVM virtualization:

    KVM creates a hypervisor virtualization host on a Linux server. A KVM hypervisor can host multiple virtual machine (VM) guests.
<br>
<br>
* Understanding Terraform:

    Terraform can run scripts that automate the creation of virtual machines, on public clouds such as Amazon AWS, Google GCP, and Digital Ocean, as well as on on-premises (“onprem”) virtualization hosts including KVM.
<br>
<br>

* Understanding the libvirt provider software:

    The libvirt provider software enables Terraform to automate the creation of virtual machines on KVM hypervisor hosts.
<br>
<br>

*  Understanding Ansible:

    Ansible can run scripts that automate server administration tasks. In this exercise, multiple Ansible scripts will be executed to use Terraform to create virtual machine (VM) servers, on which software will be deployed and configured, creating a Kubernetes cluster.
<br>
<br>

* Understanding virtual machine (VM) guests
    A virtual machine (VM) guest is a server that emulates hardware as a software image, using a subset of the hypervisor host’s processor cores, memory, and storage to create a distinct computer environment, with its own software libraries, network address, and password or key entry system.
<br>
<br>

* Understanding Docker software containers

    Docker containers are software containers created by the docker-compose command. A Docker container has its own software libraries, network address, and password or key entry system, but comparisons between Docker containers and virtual machines (VMs) are discouraged.
<br>
<br>

* Understanding Kubernetes:
    
    Kubernetes is software that allows you to create a high-availability cluster consisting of a control plane server and one or more worker servers. Kubernetes allows for software to be deployed as containers stored in pods, running on clusters, running on nodes.
<br>
<br>
* Containers:
    
    Software is organized within Docker containers. A Docker container is a self-contained computing environment with its own libraries, IP address, and SSH password or key.
<br>
<br>
* Pods:
    
    Pods are a unit of computing that contain one or more containers. Pods execute on clusters, which are intermediate interfaces that distribute computing tasks across control plane and worker nodes, running on virtual machine servers.
<br>
<br>
* Clusters
    
    A cluster is an addressable interface that allows for the execution of Kubernetes Pods across a distributed system of control plane and worker nodes.
<br>
<br>

* Nodes:
    In Kubernetes, a node is the physical or virtual machine that hosts the control plane role or worker role in a distributed Kubernetes cluster. In this exercise the nodes will be hosted on KVM virtual machines (VMs).
<br>
<br>

* Understanding the automation server
    The automation server is a Linux server separate from the virtualization server. The automation server can be a physical or virtual machine.
<br>
<br>

Tip: avoid running operations like this from your baremetal desktop. These operations involve hosts files and SSH keys for server access, and should be isolated if possible. Consider creating a virtual machine for this role using a hypervisor such as KVM on Linux, VirtualBox on Windows, or Parallels on MacOS. Use Ubuntu Linux 22.04 LTS.
<br>
<br>


## Entering commands as root
This procedure assumes that you are entering commands as root. Escalate to the root user:

```bash
sudo su
```
<br>
<br>

## Preparing the virtualization server 1/3
The virtualization server should be a minimal build: do a fresh format of Ubuntu Linux 22.04 LTS.

Using a wired network connection

If possible, use a wired Ethernet connection for the network connection on the hypervisor. This simplifies advanced operations like iptables forwarding and makes possible the later use of macvtap adapters for connecting in the hypervisor host networking space.

Setting a static IP address
Set a static IP address for the network connection of the virtualization server. Reboot.

Installing software on the virtualization server
* From a root shell on the virtualization server, enter the following command:

```bash
apt install ifconfig net-tools iptraf-ng openssh-server
```

* Configuring the SSH server on the virtualization server
From a root shell on the virtualization server, enter the following commands:

```bash
cd /etc/ssh
vim sshd_config
# uncomment and replace the following line:
PermitRootLogin yes
```

* Creating a root password
```bash
sudo su
passwd
```

<br>
<br>

## Preparing the automation server 1/2
Configure a virtual machine on a different physical machine than the virtualization server. Use Ubuntu Linux 22.04 LTS.

Adding a macvtap or bridge mode network adapter

For KVM, add a macvtap network adapter to the automation server. For VirtualBox or Parallels, add a bridge mode network adapter. This will allow the automation server to access internal subnets on the virtualization server via an ip route command.

Installing software and downloading Ansible scripts on the automation server

From a root shell on the automation server, enter the following commands:

```bash
apt install ansible git openssh-server net-tools iptraf-ng
cd /root
mkdir tmpops
cd tmpops
git clone https://github.com/kubealex/libvirt-k8s-provisioner.git
```

* Creating the Ansible hosts file

```bash
cd /etc
mkdir ansible
cd ansible
```

* create the following text file (use the IP address of the virtualization server in your setup):
```bash
vim hosts

# contents:
[vm_host]
192.168.56.60
```


* Creating an SSH key pair
From a root shell on the automation server, enter the following command:

```
ssh-keygen -f /root/.ssh/id_rsa -q -N ""
```
When prompted for a passphrase, press Enter and provide a blank value.

Copying the SSH public key to the virtualization server

* From a root shell on the automation server, enter the following commands (substitute the IP address of the virtualization server in your setup):
```bash
cd /root/.ssh
rsync -e ssh -raz id_rsa.pub root@192.168.56.60:/root/.ssh/authorized_keys
```

<br>
<br>

## Preparing the virtualization server 2/3
Verifying the automation server’s public key on the virtualization server

* From a root shell on the virtualization server, enter the following commands:
```bash
cd /root/.ssh
ls -la
```

* Verify that the file authorized_keys is listed:

```
root@henderson:/home/desktop# cd /root/.ssh
root@henderson:~/.ssh# ls -la
total 20
drwx------ 2 root root 4096 Jul 29 05:51 .
drwx------ 11 root root 4096 Jul 29 06:25 ..
-rw-r--r-- 1 root root 565 Jul 28 08:18 authorized_keys
-rw------- 1 root root 978 Jul 29 05:50 known_hosts
-rw-r--r-- 1 root root 142 Jul 29 05:50 known_hosts.old
```

Testing that the automation server can connect to the virtualization server using a public SSH key

From the automation server, enter the folowing command (substitute the IP address of your virtualization server):

```bash
ssh root@192.168.56.60
```

Note: If you are able to login without supplying a password, you have succeeded.

Using Ansible to automate operations

Ansible can run scripts called playbooks to perform automated server administration tasks. Ansible playbook scripts will use Terraform to create and configure virtual machines (VMs) on which a Kubernetes cluster will be installed.

A note about the libvirt-k8s-provisioner project

The libvirt-k8s-provisioner project provides a set of scripts that use Ansible and Terraform to create virtual machines (VMs) and to deploy a Kubernetes cluster.

Modifying the libvirt-k8s-provisioner vars file

```bash
cd /root/tmpops/libvirt-k8s-provisioner/vars
vim k8s_cluster.yml
```

    k8s:
    cluster_name: k8s-test
    cluster_os: Ubuntu
    cluster_version: 1.24
    container_runtime: crio
    master_schedulable: false
    
    # Nodes configuration
    
    control_plane:
        vcpu: 2
        mem: 2 
        vms: 3
        disk: 30
    
    worker_nodes:
        vcpu: 2
        mem: 2
        vms: 1
        disk: 30
    
    # Network configuration
    
    network:
        network_cidr: 192.168.200.0/24
        domain: k8s.test
        additional_san: ""
        pod_cidr: 10.20.0.0/16
        service_cidr: 10.110.0.0/16
        cni_plugin: cilium
    
    rook_ceph:
    install_rook: true
    volume_size: 50
        rook_cluster_size: 1
    
    # Ingress controller configuration [nginx/haproxy]
    
    ingress_controller:
    install_ingress_controller: true
    type: haproxy
        node_port:
            http: 31080
            https: 31443    
    
    # Section for metalLB setup
    
    metallb:
    install_metallb: true
    l2:
        iprange: 192.168.200.210-192.168.200.250


Installing the collection requirements for Ansible operations

* From a root shell on the virtualization server, enter the following commands:
```bash
cd /root/tmpops/libvirt-k8s-provisioner
ansible-galaxy collection install -r requirements.yml
```

Running the Ansible playbook to create and configure virtual machines on the virtualization host 1/2

* From a root shell on the virtualization server, enter the following commands:
```bash
ansible-playbook main.yml
```

The task sequence will end with this error:

    fatal: [k8s-test-worker-0.k8s.test]: FAILED! => {"changed": false, "elapsed": 600, "msg": "timed out waiting for ping module test: Failed to connect to the host via ssh: ssh: Could not resolve hostname k8s-test-worker-0.k8s.test: Name or service not known"}
    fatal: [k8s-test-master-0.k8s.test]: FAILED! => {"changed": false, "elapsed": 600, "msg": "timed out waiting for ping module test: Failed to connect to the host via ssh: ssh: Could not resolve hostname k8s-test-master-0.k8s.test: Name or service not known"}
Note: we will recover from this error in a later step.


<br>
<br>

## Preparing the virtualization server 3/3
From a root shell on the virtualization server, enter the following command:

```bash
virsh net-dhcp-leases k8s-test
```

Information about the virtual machines in the k8s-test network will be displayed:

```
virsh net-dhcp-leases k8s-test
```
    Expiry Time MAC address Protocol IP address Hostname Client ID or DUID
    --------------------------------------------------------------------------------------------------------------------------------------------------------
    2022-07-29 07:21:42 52:54:00:4a:20:99 ipv4 192.168.200.99/24 k8s-test-master-0 ff:b5:5e:67:ff:00:02:00:00:ab:11:28:1f:a1:fb:24:5c:f5:70
    2022-07-29 07:21:42 52:54:00:86:29:8f ipv4 192.168.200.28/24 k8s-test-worker-0 ff:b5:5e:67:ff:00:02:00:00:ab:11:9e:22:e1:40:72:21:cf:9d

Take note of the IP addresses starting with 192.168.200, these values will be needed in a later configuration step.

Understanding the need for IP forwarding on the virtualization server
By default, virtual machines are created with IP addresses in the 192.168.200.x subnet. This subnet is accessible within the virtualization server.

In order to make the 192.168.200.x subnet accessible to the automation server, we need to create a gateway router using iptables directives on the virtualization server.

In a later step, we will add a default route for the 192.168.200.x subnet on the automation server, allowing it to resolve IP addresses in that subnet.

Enabling IP forwarding for the 192.168.200.x subnet
From a root shell on the virtualization server, enter the following commands:

Create the following text file:


```bash
cd /etc
vim sysctl.conf
```

Add the following line to the end of the sysctl.conf file:

    net.ipv4.ip_forward = 1

Enter this command:

```bash
sysctl -p
```


* Create the following text file (substitute the wanadaptername and wanadapterip for those of the virtualization server in your setup):

```bash
vim forward.sh
```
contents:

    #!/usr/bin/bash
    # values
    kvmsubnet="192.168.200.0/24"
    wanadaptername="eno1"
    wanadapterip="192.168.56.60"
    kvmadaptername="k8s-test"
    kvmadapterip="192.168.200.1"
    # allow virtual adapter to accept packets from outside the host
    iptables -I FORWARD -i $wanadaptername -o $kvmadaptername -d $kvmsubnet -j ACCEPT
    iptables -I FORWARD -i $kvmadapterip -o $wanadaptername -s $kvmsubnet -j ACCEPT
    iptables --table nat --append POSTROUTING --out-interface $wanadaptername -j MASQUERADE

Enter the following commands:

```bash
chmod 755 forward.sh
bash forward.sh
```


<br>
<br>

## Preparing the automation server 2/2
Adding IP addresses for the VM hosts created in “Preparing the virtualization server 3/3”
From a root shell on the automation server, enter the following commands:

Use the vim editor to create the following text file:

```bash
cd /etc
vim sysctl.conf
```

Add the following line to the end of the sysctl.conf file:

    net.ipv4.ip_forward = 1


Enter this command:

```bash
sysctl -p
```

* Modify /etc/hosts file:

```bash
vim hosts
```

Add the following lines (substitute the IP addresses observed earlier in “Preparing the virtualization server 3/3”):

    192.168.200.99 k8s-test-master-0.k8s.test
    192.168.200.28 k8s-test-worker-0.k8s.test




* Adding a route for for the 192.168.200.x subnet:

From a root shell on the automation server, enter the following command (substitute the wanadaptername (dev) and wanadapterip for those of the *automation* server in your setup):

```bash
ip route add 192.168.200.0/24 via 192.168.56.60 dev
```


Testing the IP routing from the automation server to the 192.168.200.x subnet
Ping one of the IP addresses you observed in the preceding step “Preparing the virtualization server 3/3” (Substitute one of the IP addresses in your setup):


ping 192.168.200.99
Running the Ansible playbook to create and configure virtual machines on the virtualization host 2/2
From a root shell on the automation server, enter the following commands:

```bash
cd /root/tmpops/libvirt-k8s-provisioner
ansible-playbook main.yml
```

Verifying that the Ansible task sequence has completed without errors
Sample output:

    PLAY RECAP *******************************************************************************************************************************************************************************************************************************
    192.168.56.60 : ok=76 changed=24 unreachable=0 failed=0 skipped=32 rescued=0 ignored=0
    k8s-test-master-0.k8s.test : ok=49 changed=28 unreachable=0 failed=0 skipped=28 rescued=0 ignored=0
    k8s-test-worker-0.k8s.test : ok=38 changed=24 unreachable=0 failed=0 skipped=28 rescued=0 ignored=0

Making a test connection to a VM
From a root shell on the automation server, enter the following command (substitute the IP address of a VM in your setup):

```bash
ssh kube@192.168.200.99
```
When prompted, enter the password: “kuberocks”

Listing all container images running in the clusters

* From a root shell on the control plane (obsolete term “master”) VM server, enter the following command:
```bash
kubectl get pods –all-namespaces
```

Output:

    Pending 0 113m
    rook-ceph rook-ceph-mds-ceph-filesystem-a-868694c95d-85r54 1/1 Running 0 113m
    rook-ceph rook-ceph-mds-ceph-filesystem-b-748dc85c96-qktmb 1/1 Running 0 113m
    rook-ceph rook-ceph-mgr-a-7f6784d748-cg9v8 1/1 Running 0 115m
    rook-ceph rook-ceph-mon-a-6f9d4bc99b-fgc2g 1/1 Running 0 116m
    rook-ceph rook-ceph-operator-f4ccf8fc-f5rcl 1/1 Running 0 119m
    rook-ceph rook-ceph-osd-0-7b6fbf8657-lktsx 0/1 CrashLoopBackOff 27 (38s ago) 114m
    rook-ceph rook-ceph-osd-prepare-k8s-test-worker-0.k8s.test-j79rg 0/1 Completed 0 114m
    rook-ceph rook-ceph-rgw-ceph-objectstore-a-64b5fd4d9b-77krx 0/1 Running 20 (7m43s ago) 110m
