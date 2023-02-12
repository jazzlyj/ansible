I use Ansible to automate this process, making it even more flexible and repeatable. My technique relies on variables to reuse the same automation, allowing me to create different types of VMs with a single command.

Start by defining an Ansible role to provision VMs in KVM.

Create the role
To make the KVM provisioning automation more reusable, create an Ansible role for it. If you've never created a role before, look at my article "8 steps to developing an Ansible role in Linux" to understand the basics.

Create a project directory for this automation and switch to it:

$ mkdir -p kvmlab/roles && cd kvmlab/roles
Then, initialize the role using the command ansible-galaxy:

$ ansible-galaxy role init kvm_provision

- Role kvm_provision was created successfully
Switch into the newly created role directory and check that it has the basic role structure ready to customize:

$ cd kvm_provision
$ ls
defaults files handlers meta README.md tasks templates tests vars
For brevity in this article, I will not document the role, but it's best practice to document roles by adding information to the README file and details and dependencies to meta/main.yaml.

This role does not use the subdirectories files, handlers, and vars, so you can delete them:

$ rm -r files handlers vars
Now that your basic role structure is ready, start working with it by defining default variables.

Define default variables
A role makes your automation reusable by allowing users to specify variables that change the role behavior or outcome. It's good practice to define a default value for all variables you expose to the end user to ensure the role doesn't fail to execute if users don't specify some of these variables.

[ For more Ansible tips, download Jeff Geerling's eBook Ansible for DevOps. ]

For this role, specify variables that allow a user to change the type of VM provisioned, including its name, number of CPUs, amount of RAM, and the cloud image to use.

Add these variables to the file defaults/main.yml, like this:

---
# defaults file for kvm_provision
base_image_name: Fedora-Cloud-Base-34-1.2.x86_64.qcow2
base_image_url: https://download.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/{{ base_image_name }}
base_image_sha: b9b621b26725ba95442d9a56cbaa054784e0779a9522ec6eafff07c6e6f717ea
libvirt_pool_dir: "/var/lib/libvirt/images"
vm_name: f34-dev
vm_vcpus: 2
vm_ram_mb: 2048
vm_net: default
vm_root_pass: test123
cleanup_tmp: no
ssh_key: /root/.ssh/id_rsa.pub
Remember that these are just default values. The user can overwrite them by specifying these values when calling this role from a playbook later.

Next, define a VM template file.

Define a VM template
To provision a KVM VM using Ansible, use the community.libvirt.virt module. This module requires a VM definition in XML format according to the syntax of libvirt. The most convenient way to get this file is to dump an existing VM definition using the command virsh dumpxml. For more details, consult my article "8 Linux virsh subcommands for managing VMs on the command line."

After obtaining a base XML VM definition, you must convert it to a template that allows the automation to provision different VMs according to the variables received by the user.

[ For more details on creating template files, check How to create dynamic configuration files using Ansible templates. ]

For this example, define a VM template file vm-template.xml.j2 in the templates role subdirectory:

```xml
<domain type='kvm'>
  <name>{{ vm_name }}</name>
  <memory unit='MiB'>{{ vm_ram_mb }}</memory>
  <vcpu placement='static'>{{ vm_vcpus }}</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-5.2'>hvm</type>
    <boot dev='hd'/>
  </os>
  <cpu mode='host-model' check='none'/>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='{{ libvirt_pool_dir }}/{{ vm_name }}.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
    </disk>
    <interface type='network'>
      <source network='{{ vm_net }}'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='2'/>
    </channel>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='spice' autoport='yes'>
      <listen type='address'/>
      <image compression='off'/>
    </graphics>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
    </memballoon>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </rng>
  </devices>
</domain>
```

Note that this XML code uses some of the variables you previously defined in your defaults/main instead of values where you would expect to customize the template. These variables are interpolated when you render this template later. For example, this file sets the VM name using the variable vm_name:


  <name>{{ vm_name }}</name>
Now you're ready to define the action the role performs.

Define tasks
Define the tasks required to deploy the VM based on the default variables and the template file you created before. Edit the file tasks/main.yml and define the first task to ensure the required package dependencies guestfs-tools and python3-libvirt are installed. This role requires these packages to connect to libvirt and to customize the virtual image in a later step.

```yml
---
# tasks file for kvm_provision
- name: Ensure requirements in place
  package:
    name:
      - guestfs-tools
      - python3-libvirt
    state: present
  become: yes
```

Note: These package names work on Fedora Linux. If you're using RHEL 8 or CentOS, use libguestfs-tools instead of guestfs-tools. For other distributions, adjust accordingly.

Next, add a task to obtain the list of existing VMs. You don't want your automation to overwrite an existing VM accidentally, so use this information to skip some tasks in the event that the VM is already provisioned.

```yml
- name: Get VMs list
  community.libvirt.virt:
    command: list_vms
  register: existing_vms
  changed_when: no
```


Image
IT Automation ebook
This task uses the virt module from the collection community.libvirt, which interacts with a running instance of KVM with libvirt. It obtains the list of VMs by specifying the parameter command: list_vms and saves the results in a variable existing_vms.

Note that you're specifying the parameter changed_when: no for this task to ensure that it's not marked as changed in the playbook results. This task doesn't make any change in the machine; it only checks the existing VMs. This is a good practice when developing Ansible automation to prevent false reports of changes.

Next, define a block of tasks that execute only when the VM name the user provides doesn't exist:

```yml
- name: Create VM if not exists
  block:
Add the first task in this block using the module get_url to download the base cloud image into the /tmp directory:

  - name: Download base image
    get_url:
      url: "{{ base_image_url }}"
      dest: "/tmp/{{ base_image_name }}"
      checksum: "sha256:{{ base_image_sha }}"
```
Since this same base image can provision more VMs, you don't want to change this file directly. Copy it to the final location in libvirt's pool directory to customize it and then to use it later as the VM disk:

```yml
  - name: Copy base image to libvirt directory
    copy:
      dest: "{{ libvirt_pool_dir }}/{{ vm_name }}.qcow2"
      src: "/tmp/{{ base_image_name }}"
      force: no
      remote_src: yes 
      mode: 0660
    register: copy_results
```
Now, use the command module to execute the command virt-customize to customize the cloud image, as Alex does in his article. This variation uses variables to make the command more flexible and reusable:

```yml
  - name: Configure the image
    command: |
      virt-customize -a {{ libvirt_pool_dir }}/{{ vm_name }}.qcow2 \
      --hostname {{ vm_name }} \
      --root-password password:{{ vm_root_pass }} \
      --ssh-inject 'root:file:{{ ssh_key }}' \
      --uninstall cloud-init --selinux-relabel
    when: copy_results is changed
```
Note that you're passing the extra option --selinux-relabel. This option is required only for Red Hat-based images or other images that have SELinux enabled. You must remove this option if you use an image that does not use SELinux. You could also use a variable here to make it more flexible, but that's out of scope for this article.

Next, define the VM using the module community.libvirt.virt, instantiating the VM templates you created before:

```yml
  - name: Define vm
    community.libvirt.virt:
      command: define
      xml: "{{ lookup('template', 'vm-template.xml.j2') }}"
```
Finally, close this block by adding the when condition specifying that the block should execute only if the VM name is not in the list of VMs:

  when: "vm_name not in existing_vms.list_vms"
To finish the role, add a task to ensure the new VM starts and another task to clean up the base cloud image from /tmp if the user specifies variable cleanup_tmp: yes:

```yml
- name: Ensure VM is started
  community.libvirt.virt:
    name: "{{ vm_name }}"
    state: running
  register: vm_start_results
  until: "vm_start_results is success"
  retries: 15
  delay: 2

- name: Ensure temporary file is deleted
  file:
    path: "/tmp/{{ base_image_name }}"
    state: absent
  when: cleanup_tmp | bool
```

Save and close this file. Your role is complete. Test it out by adding it to a playbook.

Use the role in a playbook
Now that the role is complete, you can finally use it.

Switch back to the initial project directory:

$ cd ../..
$ ls
roles

Create a new playbook file kvm_provision.yaml that uses the kvm_provision role, like this:

```yml
- name: Deploys VM based on cloud image
  hosts: localhost
  gather_facts: yes
  become: yes
  vars:
    pool_dir: "/var/lib/libvirt/images"
    vm: f34-lab01
    vcpus: 2
    ram_mb: 2048
    cleanup: no
    net: default
    ssh_pub_key: "/home/rgerardi/.ssh/id_rsa.pub"

  tasks:
    - name: KVM Provision role
      include_role:
        name: kvm_provision
      vars:
        libvirt_pool_dir: "{{ pool_dir }}"
        vm_name: "{{ vm }}"
        vm_vcpus: "{{ vcpus }}"
        vm_ram_mb: "{{ ram_mb }}"
        vm_net: "{{ net }}"
        cleanup_tmp: "{{ cleanup }}"
        ssh_key: "{{ ssh_pub_key }}"
```
Note: This playbook executes the command as root due to the parameter become: yes, which should work for most systems configured with the default libvirt settings. If your regular user is authorized to run VMs using libvirt, you can change it to become: no to run as a regular user. In this case, update the pool_dir variable to set the correct libvirt storage pool directory for your regular user.

Also, make sure to configure the variable ssh_pub_key to your SSH public key. If you don't have an SSH key, take a look at "Passwordless SSH using public-private key pairs" to learn how to create one.

Since the role uses modules from collection community.libvirt, install the collection in your local machine to have access to the modules:

$ ansible-galaxy collection install community.libvirt
Process install dependency map
Starting collection install process
Installing 'community.libvirt:1.0.2' to '/home/rgerardi/.ansible/collections/ansible_collections/community/libvirt'
Now, execute the playbook to provision the VM according to the specified parameters. Use option -K to ask for the sudo password required to install the necessary packages:

$ ansible-playbook -K kvm_provision.yaml
BECOME password: 
[WARNIN]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Deploys VM based on cloud image] ***************************************

TASK [Gathering Facts] *******************************************************
ok: [localhost]

TASK [KVM Provision role] ****************************************************

TASK [kvm_provision : Ensure requirements in place] **************************
ok: [localhost]

TASK [kvm_provision : Get VMs list] ******************************************
ok: [localhost]

TASK [kvm_provision : Download base image] ***********************************
changed: [localhost]

TASK [kvm_provision : Copy base image to libvirt directory] ******************
changed: [localhost]

TASK [kvm_provision : Configure the image] ***********************************
changed: [localhost]

TASK [kvm_provision : Define vm] *********************************************
changed: [localhost]

TASK [kvm_provision : Ensure VM is started] **********************************
changed: [localhost]

TASK [kvm_provision : Ensure temporary file is deleted] **********************
skipping: [localhost]

PLAY RECAP *******************************************************************
localhost                  : ok=8    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
When the playbook completes, use virsh list to see the running VM:

$ virsh list 
 Id Name State
---------------------------
 12 f34-lab01 running
Use virsh domifaddr f34-lab01 to obtain the IP address to connect to the VM:

$ virsh domifaddr f34-lab01
 Name MAC address Protocol Address
-------------------------------------------------------------------------------
 vnet11 52:54:00:13:23:b9 ipv4 192.168.122.6/24
Connect to the VM by using your IP address and SSH key:

$ ssh -i ~/.ssh/id_rsa root@192.168.122.6
[root@f34-lab01 ~]# 
You can also provision additional VMs by providing parameters directly in the command line. For example, to provision a second VM named f34-lab02, use:

$ ansible-playbook -K kvm_provision.yaml -e vm=f34-lab02
You can change any of the variables you defined in the vars section of the playbook. For brevity, I did not add all the variables from the role, but you can do that to change, for example, the base cloud image used to deploy the VM.

Boost time and performance
In his article, Alex provisioned a lab in less than five minutes. Just for fun, let's see what you can do with Ansible. First, delete the cached image to calculate the total time, including downloading the image:

$ rm /tmp/Fedora-Cloud-Base-34-1.2.x86_64.qcow2
Now, execute the playbook again with the time command to assess how long it takes:

$ time ansible-playbook -K kvm_provision.yaml -e vm=f34-lab03

<TRUNCATED OUTPUT>

ansible-playbook -K kvm_provision.yaml -e vm=f34-lab03 9.18s user 2.35s system 31% cpu 36.203 total
It took 36 seconds to build the lab VM, including downloading the cloud image. If you execute the playbook again to provision another VM with the image cached, it's even faster:

$ time ansible-playbook -K kvm_provision.yaml -e vm=f34-lab04

<TRUNCATED OUTPUT>

ansible-playbook -K kvm_provision.yaml -e vm=f34-lab04 5.84s user 1.08s system 29% cpu 23.792 total
Provisioning a VM in 24 seconds doesn't sound bad at all.





https://www.redhat.com/sysadmin/build-VM-fast-ansible