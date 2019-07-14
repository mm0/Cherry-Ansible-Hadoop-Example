How-to: Create a Hadoop/Spark Cluster on CherryServer using only Ansible
---

## Description

In this tutorial, we will be creating 3 servers on the CherryServers Provider 
in order to setup a 3 node hadoop/spark cluster, with 1 master server and 2 slaves.  

We will be using Ansible for both configuring the CherryServers account and in order to
reserve the servers within CherryServers as well as configuring the servers themselves.
Some experience with Ansible will make this easier to follow, but is not completely necessary.

## Requirements

- Make sure you have Ansible version 2.5+ installed locally. There are multiple ways to do this. 
Instructions can be found [here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).


- The Cherry Servers module connects to Cherry Servers Public API via cherry-python package. You need to install it with pip (this might need to be done as `sudo`):

    ```bash
    pip install cherry-python
    ```
- You will need to download [this](https://github.com/cherryservers/cherry-ansible-module/tree/master/cherryservers) directory into the `ansible/library` subdirectory of this project.  
This is the Cherry Servers Ansible Module that we will use to interact with Cherry Server's API. The `ansible.cfg` file in the `ansible` directory has a `library` entry that points to the `library` subdirectory and tells ansible where to find our custom modules.


## Directions

- Create an API Key for your CherryServers account at [https://portal.cherryservers.com/#/settings/api-keys](https://portal.cherryservers.com/#/settings/api-keys)

    Save your API Key somewhere safe and optionally add it to your `~/.profile` file like so:

    ```
    export CHERRY_AUTH_TOKEN="2b00042f7481c7b056c4b410d28f33cf"
    ```

- Next, find your Project ID from the Admin panel [https://portal.cherryservers.com/#/projects](https://portal.cherryservers.com/#/projects)

    Change the line `cherryservers_project_id: [here]` to reflect your Project ID in the file `group_vars/all.yml` 


- Next, you will need an SSH key with which to access your servers and for ansible to connect to your servers to configure them.

    ```
    ssh-keygen -f ~/.ssh/cherry
    ```

    This will create `~/.ssh/cherry` and ~/.ssh/cherry.pub`

    This is the SSH key you will use in order to SSH into your Cherry Server and use with Ansible.  
    
    You can configure your ~/.ssh/config to use this new key with the IP addresses for your servers that we will create further below.

- While we're add it.  We're going to create a second SSH key that will be used by Hadoop and Spark.  

    Hadoop and Spark master servers SSH into the slaves within their cluster in order to manage them.  
    
    Our Ansible playbooks will upload this second SSH key to the new servers we create and allow the `hadoop` and `spark` users to SSH using this key. 
    
    When generating this second key, have it save to the ansible directory here.

    ```
    cd this/directory
    ssh-keygen -f files/test-key
    ```

    This will create `ansible/files/test-key` and `ansible/files/test-key.pub`

- We can now add the `cherry` SSH Key to our Cherry Servers account. 

    The file path for the key is stored in `group_vars/all.yml` and can be overridden here or even via CLI by adding the ` --extra-vars cherryservers_keyfile=/path/to/mykey.pub` when running the command below (it defaults to `~/.ssh/cherry.pub`):

    ```
    ansible-playbook cherry_add_ssh_key_to_account.yml 
    ```
  
- Let's continue by allocating 3 IP addresses with which to access our servers that will be created in the next step.

    ```
    ansible-playbook cherry_allocate_ip_address.yml 

    ```
    This will output:
    ```bash

    PLAY [localhost] ****************************************************************************************

    TASK [Gathering Facts] **********************************************************************************
    ok: [localhost]

    TASK [Manage IP addresses] ******************************************************************************
    changed: [localhost] => (item=1)
    changed: [localhost] => (item=2)
    changed: [localhost] => (item=3)

    TASK [debug] ********************************************************************************************
    ok: [localhost] => (item=0) => {
        "changed": false, 
        "item": "0", 
        "msg": "185.150.116.211"
    }
    ok: [localhost] => (item=1) => {
        "changed": false, 
        "item": "1", 
        "msg": "185.150.116.135"
    }
    ok: [localhost] => (item=2) => {
        "changed": false, 
        "item": "2", 
        "msg": "185.150.116.5"
    }

    PLAY RECAP **********************************************************************************************
    localhost                  : ok=3    changed=1    unreachable=0    failed=0   
    ```
    
- You will then need to save the new IP addresses to your ansible inventory. They will not yet be associated with any servers. Choose any one IP to be the master, and the other two will be for the data nodes. 

    `hosts/cherryservers`:
    ```bash
    [cluster_master]
    185.150.116.211

    [cluster_nodes]
    185.150.116.135
    185.150.116.5
    ```

- Next, create your servers using the cherry_create_server.yml playbook.  The creation process will automatically assign the IPs added to `hosts/cherryservers` to the servers that are created.

  In the file `group_vars/all.yml` Change the line `cherryservers_plan_id: [here]` to reflect the plan ID of the server type you want to create 
  ```
  ansible-playbook cherry_create_server.yml
  ```

- We can now provision our Hadoop and Spark masters on the `cluster_master` server.  Replace the `ip1` in the command below, and then run the command.

    You will either have to SSH into each of the serves once before running this command or disable `host_key_checking` for ansible by running:
    
    ```bash
    export ANSIBLE_HOST_KEY_CHECKING=False
    ```
  ```
  ansible-playbook setup_hadoop.yml -e "target=cluster_master;cluster_nodes" -e "hadoop_master_ip=ip1" --private-key="~/.ssh/cherry"  --user=root
  ```
  
  This will result in each server having hadoop installed and having the HDFS formatted. 
  
- We will install Spark next:
  
  ```
  ansible-playbook setup_spark.yml -e "target=cluster_master;cluster_nodes" -e "hadoop_master_ip=[ip1]" --private-key="~/.ssh/cherry" --user=root
  ```
  Spark is now installed and configured on each server.  
  
- All that is left is for hadoop and spark to be started on the master nodes.  The master nodes will automatically start the data nodes. Notice that our target variable is only the master host group.
  ```
  ansible-playbook start_hadoop_and_spark.yml -e "target=cluster_master" --private-key="~/.ssh/cherry" 
  ```
  

- The Cluster should now be ready and you will be able to access the admin panels via:

    Hadoop UI: [http://cluster_master_ip:50700]()

    Spark UI: [http://cluster_master_ip:8080]()
