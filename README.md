# ansible_ssh
This repository contains sample code to deploy the sample application on AWS linux instances 
through setting up SSH between two AWS EC2 instances using Ansible

# Directory Structure
  1. configs - contain environment specific variable
  2. inventories - contains inventory file for each environment
  3. groups_vars - contains common variables across environments
  4. roles - This will have subfolders like java,tomcat

       a) tomcat - this folder will have files related to the installation of tomcat

              - files (This will contain files which you want to copy to to your destination servers)

              - handlers ( This is used to start/stop the services)

              - templates (This will contain template files)

              - tasks ( playbook to install the software)

        b) add_devops_user - This folder will have files related to the setup of initial user

  5. main.yml - This is the main file which will execute roles in the playbook

# Agenda:
1. Create and Setup AWS EC2 instances
2. SSH to the Ansible master node
3. Setup a new user devops on the Ansible master node manually
4. Run the playbook to setup a devops user on all other nodes
5. If you do not want to create a new user and use the default user like ec2-user,ubuntu 
then you can skip the creation of user.

# Launch two AWS EC2 instances
Login to AWS Console
Search for service EC2 ->Click on EC2 -> Instances ->Launch Instance -> Linux AMI2 -> 
select default instance t2.micro -> configure security group Review and Launch -> 
create a key to connect to the instance (or use existing key in the root directory: ansible_aut.pem)

# Connect to Ansible Master Node using SSH
1. Open an SSH client (terminal) 
2. Locate your private key file (ansible_aut.pem)
3. If it is needed run ``` chmod 400 ansible_aut.pem ```
4. Connect to your instance using its Public DNS (Example): 
``` 
ssh -i "ansible_aut.pem" ec2-user@ec2-3-67-189-5.eu-central-1.compute.amazonaws.com 
```

Now you are connected to your master Ansible node
Run: ``` sudo yum update ```

# Prerequisite
1. Python should be installed (Run to check: python --version)
2. Install Ansible 
``` sudo amazon-linux-extras install ansible2 ```

Run to check: ``` ansible --version ```

# Setup a devops user on Master Node
1. Create a user devops

```sudo -i```

```useradd -m -s /bin/bash devops```
2. Set a password
```passwd devops```
3. Add the user in sudoers.d file, this allow user to run any command 
using sudo without passing their password
```
echo -e 'devops\tALL=(ALL)\tNOPASSWD:\tALL' > /etc/sudoers.d/devops
```

To check: ``` cat etc/sudoers.d/devops ```

User devops has created successfully.
Now we will generate the SSH keys for the devops user 

# Generate a SSH Key
1. Login as a devops user and follow the prompts

``` sudo -su devops ```

``` ssh-keygen -t rsa ``` 

By default both keys (id_rsa and id_rsa.pub) will store in /home/devops/.ssh/

To reach them: 
``` cd ~/.ssh ```

``` ls ``` 

# Install git and clone the git repo
``` sudo yum install git```

``` git clone https://github.com/statcio/ansible_ssh.git ```

1. Write a playbook to create a new user, set a password, add it to the sudoers file.
2. lookup command will try to find the .pub file on the master ansible node for devops user and 
put that public key in the authorized_keys on the remote servers. Put the .pub file either on your git repo or 
anywhere on the master node

```sh
- name: Add a new user named devops
     user:
          name=devops
          password={{ devops_password }}
 
   - name: Add devops user to the sudoers
     copy:
          dest: "/etc/sudoers.d/devops"
          content: "devops  ALL=(ALL)  NOPASSWD: ALL"
 
   - name: Deploy SSH Key
     authorized_key: user=devops
                     key="{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
                     state=present
```

Playbook to call the above role

```sh
- hosts: all
  become: true
  become_user: root
  gather_facts: false
  tasks:
    - include_role:
        name: add_devops_user
        tasks_from: add_user.yml
```

# How to Run the Playbook
1. You need to provide the user ec2-user and the key to connect to the remote host.
By default all the remote hosts have same keys
You need to use the .pem file to connect initially
PEM file need to have specific permission before you can use it directly. 
If the permission is not set properly you will see the error 
“It is required that your private key files are NOT accessible by others. This private key will be ignored.”
2. You need to update file: ```/inventories/dev/hosts``` according your private IP address of second AWS instance.  
3. Change the permission of the pem file
``` sudo chmod 600 ansible_aut.pem ```
4. Run playbook
```
ansible-playbook main.yml -i inventories/dev/hosts --user ec2-user --key-file /home/ec2-user/playbooks/ansible_aut.pem -e '@configs/dev.yml'

```

Once you setup the devops user then you can use the devops key and run the playbook using devops user
``` 
ansible-playbook main.yml -i inventories/dev/hosts --user devops --key-file /home/devops/.ssh/id_rsa -e '@configs/dev.yml'

```

# References


