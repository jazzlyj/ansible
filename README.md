# Ansible


## install
* on ubuntu
```
sudo apt install ansible
sudo apt install ansible-lint
```

* alternate install method
```shell
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

* install arg complete
```
sudo apt install python3-argcomplete
```

* activate arg complete
```
sudo activate-global-python-argcomplete3
```

* make sure authorized_keys exists on the target machine for the target user
see initial quick setup at [ubuntu20](ubuntu20.md) 



## minimal setup
* create a /etc/ansible/hosts file
```
sudo vim /etc/ansible/hosts

# add the hosts, one per line
``` 



## shutdown servers
* Install
```
ansible-galaxy collection install community.general
```

* Run
```
community.general.shutdown
```


*
```
```


*
```
```














## create a playbook and then run
* create
    * Update the /etc/hosts file on each host with all the hosts in inventory file
```

---
  - name: host file update - Local DNS setup across all the servers
    hosts: all
    gather_facts: yes
    tasks:

    - name: Update the /etc/hosts file with node name
      tags: etchostsupdate
      remote_user: jay 
      become: yes
      lineinfile:
        path: "/etc/hosts"
        regexp: ".*\t{{ hostvars[item]['ansible_hostname']}}\t{{ hostvars[item]['ansible_hostname']}}"
        line: "{{ hostvars[item]['ansible_env'].SSH_CONNECTION.split(' ')[2] }}\t{{ hostvars[item]['ansible_hostname']}}\t{{ hostvars[item]['ansible_hostname']}}"
        state: present
        backup: yes
      register: etchostsupdate
      when: (ansible_hostname == item) or (ansible_hostname !=  item) 
      with_items: "{{groups['all']}}"

``` 


* run and when prompted enter your password 
```
ansible-playbook upd_etchosts.yaml --ask-become-pass
``` 




*
```

``` 









## create a playbook to deploy user accounts
* creat directory and add these files
```
sudo su - 
mkdir -p /etc/ansible/playbooks; cd !$
touch ansible.cfg inventory add_users.yml usernames.yml
```

* edit ansible.cfg and this content
```
[defaults]
inventory=inventory
remote_user=admin
ask_pass=False
ansible_python_interpreter=/usr/bin/python3
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```


* edit "inventory", add the nodes
```
[worker_nodes]
node1
node2
node3
```


* edit "add_users.yml", add this content
```
---
- name: Create New Users
  hosts: all
  remote_user: jay
  become: true
  gather_facts: false
  vars_files:
    - users_pass.yml
    - usernames.yml
  tasks:
    - name: Create Users, Home Directory and add to groups 
      user:
        name: "{{ item }}"
        password: "{{ user_pass | password_hash('sha512', user_salt) }}"
        shell: /bin/bash
        system: no
        state: present
        createhome: yes
        groups: 
        append: yes
        home: "/home/{{ item }}"
        generate_ssh_key: yes
        ssh_key_bits: 2048
        ssh_key_file: .ssh/id_rsa
        update_password: on_create
      with_items:
        - "{{ names }}"
      register: user_status
 
    - name:
      shell: chage -d 0 "{{ item }}"
      with_items:
        - "{{ names }}"
      when: user_status.changed
```

* edit "usernames.yml, add the users you want
```
names:
  - "alpha"
  - "beta"
  - "cuda"
```


* i) create the ansible vault password,  
ii) and then create the encrypted password to be used for the accountsfile
```
ansible-vault create users_pass.yml
# enter a password to serve as the ansible vault password

# then a blank window opens. enter the password string (unencrypted). 
# This will be the common password for the users being deployed

# then write and quit
:wq!  

```


* check the playbook works
```
ansible-playbook add_users.yml --syntax-check --ask-vault-pass
```


* run the playbook 
```
ansible-playbook add_users.yml --ask-vault-pass --ask-become-pass
```










*
```

```

*
```

```




## Pre-reqs
```
sudo apt install python3-pip
```


