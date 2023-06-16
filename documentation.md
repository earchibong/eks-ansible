# Create Kubernetes Role for Service Account

As per the project request below...

<br>

<br>

<img width="722" alt="ansible" src="https://github.com/earchibong/eks-ansible/assets/92983658/a9c22bc0-c3cc-45a4-a935-6b605f3549c9">

<br>

<br>

### Why would we want to create these resources?

Let’s consider the following scenario

- We have deployments/pods in a namespace called `webapps`
- The deployments/pods need Kubernetes API access to manage resources in a namespace.

<br>

<br>

The solution to the above scenarios is to have a service account with roles with specific API access.

- Create a service account bound to the namespace webapps namespace
- Create a role with the list of required API access to Kubernetes resoruces.
- Create a Rolebinding to bind the role to the service account.
- Use the service account in the pod/deployment or Kubernetes Cronjobs
Lets implement it with ansible.

<br>

<br>

*note: A role provides API access only to resources present in a namespace. For cluster-wide API access, you should use a ClusterRole*

<br>

<br>

## Project Steps:
- <a href=" ">Pre-requisites</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#install-and-configure-ansible-in-host-server "> Install & Configure Ansible Host Server</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#set-up-project-structure ">Set Up Project Structure</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#set-up-ansible-inventory">Set Up ansible Inventory</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#set-up-data-file">Set Up Data File</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#create-a-playbook">Create A Playbook</a>
- <a href="https://github.com/earchibong/eks-ansible/blob/main/documentation.md#run-playbook">Run Playbook</a>

<br>

<br>

## Pre-requisites
Before you begin, make sure you have the following prerequisites in place:

- An EKS cluster set up and configured.

You can find out how to do this with `terraform` from <a href="https://github.com/earchibong/terraform-eks">this tutorial</a> or create using `eksctl` below.

<br>

### Create EKS cluster with eksctl

- create 2 keypairs for bastion host and public nodes

```

aws ec2 create-key-pair \
    --key-name my-key-pair \
    --key-type rsa \
    --key-format pem \
    --query "KeyMaterial" \
    --output text > my-key-pair.pem
    
chmod 400 my-key-pair.pem
    
```

<br>

<br>

<img width="820" alt="eypair" src="https://github.com/earchibong/eks-ansible/assets/92983658/7efb0160-d039-4f6b-9232-0d7ccf78d1c5">

<br>

<br>

- create a config file named `cluster.yaml` and add the following:

```

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: <your-region> # replace with your desired AWS region

nodeGroups:
  - name: public
    labels:
      nodegroup-type: public
    instanceType: <instance-type> # replace with desired EC2 instance type (e.g., t3.medium)
    desiredCapacity: 2
    privateNetworking: false # set to false for public nodes
    ssh:
      publicKeyName: <key-pair-name> # replace with your EC2 key pair name

managedNodeGroups:
  - name: bastion
    labels:
      nodegroup-type: bastion
    instanceType: t2.micro # adjust as needed for your bastion host
    desiredCapacity: 1
    privateNetworking: false # set to true for private nodes
    ssh:
      publicKeyName: <key-pair-name> # replace with your EC2 key pair name



# create cluster command on terminal
eksctl create cluster -f cluster.yaml



```

<br>

<br>

<img width="995" alt="cluster" src="https://github.com/earchibong/eks-ansible/assets/92983658/2a2ae323-7ced-4cfd-90c3-96921733522f">

<br>

<br>


- install `kubectl`

```

# Determine whether you already have kubectl installed on your device.
kubectl version --short --client


# Configure kubeconfig for kubectl
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
aws eks --region eu-west-2 update-kubeconfig --name ansible-eks

# List Worker Nodes
kubectl get nodes
kubectl get nodes -o wide


```

<br>

<br>

## Install And Configure Ansible

- On local machine, install ansible and other dependencies

*you can use the <a href="https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html">offical ansible documentation</a>*

```

# confirm pip
python3 -m pip -V

# install pip
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --user

# export path
nano ~/.bash_profile
export PATH=/<path>/<to>/Library/Python/3.11/bin:$PATH #add this in the bash profile
save and close nano editor
. $HOME/.bash_profile  #applies the change
echo "$PATH" # confirms new settings

# install ansible
python3 -m pip install --user ansible
ansible --version
python3 -m pip show ansible


# Install the required dependencies for working with EKS
sudo pip3 install botocore

# install kubernetes
pip install kubernetes

# install ansible-lint
sudo pip3 install ansible-lint

# install the community.kubernetes collection
ansible-galaxy collection install community.kubernetes


```

<br>

<br>

- set up `SSH-agent`

```


# on your local machine add private keys

eval `ssh-agent -s`
ssh-add ./<path-to-private-key>
ssh-add ./ansible-eks.pem   #example
ssh-add ./eks-nodes.pem    #example

# Confirm the key has been added:
ssh-add -l


# ssh into Jenkins-Ansible server using ssh-agent: 
ssh -A -i "private ec2 key" ec2-user@public_ip


```

<br>

<br>

### Set Up Project Structure

- create a folder `ansible-eks`
- Create a directory structure for Ansible project.

```

ansible-eks-project/
├── inventories/
│   └── eks_cluster.yml
└── playbooks/
    ├── eks_setup.yml
    └── files/
        └── eks_data.yml


```

<br>

<br>

<img width="900" alt="set-up" src="https://github.com/earchibong/eks-ansible/assets/92983658/712088e6-03ef-4eb9-b11f-7a5c419fa0d4">

<br>

<br>

### Set Up Ansible Inventory

- Update `inventories/eks_cluster.yml` file with this snippet of code: ...*notice that the user in this case is `ec2-user` for an `ubuntu` server it would be `ubuntu`*

```

[bastion]
bastion ansible_host=3.10.214.70 ansible_user=ec2-user 

[eks_nodes]
node1 ansible_host=18.134.196.23 ansible_user=ec2-user 
node2 ansible_host=18.133.73.103 ansible_user=ec2-user 
[eks_nodes:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q -l ec2-user 3.10.214.70"'
#ansible_ssh_common_args='-o ProxyJump=ec2-user@3.10.226.186'



```

<br>

<br>

<img width="1292" alt="eks_cluster" src="https://github.com/earchibong/eks-ansible/assets/92983658/15cbac85-9ab9-4923-b77f-cb901bdc7351">

<br>

With this inventory configuration, Ansible will connect to the `bastion host first` and then establish an SSH connection to the public nodes in the EKS cluster. This allows Ansible to execute the playbook on the target nodes through the bastion host.

<br>

### Set Up Data File

The data file does the following:
- it defines two namespaces `webapps` and `webapps2`
- it defines two service accounts named `app-service-account`  and `app-service-account2` that will be bound to the namespaces
- it defines two roles named `app-role`  and `app-role2` specific to the namespaces.
- defines role-bindings to attach the roles to the service accounts.

<br>

<br>

- update `playbooks/files/eks_data.yml`

```

---
namespaces:
  - webapps
  - webapps2

roles:
  - name: app-role
    namespace: webapps
    rules:
      - apiGroups: [""]
        resources: ["pods", "services"]
        verbs: ["get", "list", "create"]
  - name: app-role2
    namespace: webapps2
    rules:
      - apiGroups: ["apps"]
        resources: ["deployments"]
        verbs: ["get", "list", "create"]

role_bindings:
  - name: app-rolebinding
    namespace: webapps
    role: app-role
    subjects:
      - kind: ServiceAccount
        name: app-service-account 
        namespace: webapps
  - name: my-role-binding2
    namespace: my-namespace2
    role: app-rolebinding2
    subjects:
      - kind: ServiceAccount
        name: app-service-account2
        namespace: webapps2

service_accounts:
  - name: app-service-account
    namespace: webapps
  - name: app-service-account2
    namespace: webapps2


```

<br>

<br>

<img width="1203" alt="data" src="https://github.com/earchibong/eks-ansible/assets/92983658/712f0ae2-b0aa-49ae-a933-86bd105d867b">

<br>

<br>

## Create A Playbook

the `include_vars` module is used to include the YAML file as variables. The YAML data will be stored in the `yaml_data` variable. The subsequent task checks if the `yaml_data` variable is defined and fails the playbook if it is not, indicating that the YAML file could not be read or is invalid.

<br>

<br>

- update `playbooks/eks_setup.yml`

<br>

```

---
- name: Setup EKS
  hosts: eks_nodes
  remote_user: ec2-user
  become: yes
  become_method: sudo
  gather_facts: false

  vars_files:
    - files/eks_data.yml

  tasks:
    - name: include YAML file
      include_vars:
        file: ./files/eks_data.yml
        name: yaml_data

    - name: check if YAML data is valid
      fail:
        msg: "YAML file validation failed."
      when: yaml_data is not defined

    - name: Install python3-pip package
      yum:
        name: python3-pip
        state: present

    - name: Upgrade pip
      pip:
        name: pip
        state: latest
        executable: pip3

    - name: Install Kubernetes library
      pip:
        name: kubernetes
        state: present
        executable: pip3

    - name: create namespaces
      k8s:
        kubeconfig: </path/to/.kube/config>
        api_version: v1
        kind: Namespace
        name: "{{ item }}"
        state: present
      loop: "{{ namespaces }}"
    
    - name: create roles
      k8s:
        kubeconfig: </path/to/.kube/config>
        api_version: rbac.authorization.k8s.io/v1
        kind: Role
        name: "{{ item.name }}"
        namespace: "{{ item.namespace }}"
        rules: "{{ item.rules }}"
        state: present
      loop: "{{ roles }}"
    
    - name: create role bindings
      k8s:
        kubeconfig: </path/to/.kube/config>
        api_version: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        name: "{{ item.name }}"
        namespace: "{{ item.namespace }}"
        role_ref:
          api_group: rbac.authorization.k8s.io
          kind: Role
          name: "{{ item.role }}"
        subjects: "{{ item.subjects }}"
        state: present
      loop: "{{ role_bindings }}"
    
    - name: create service accounts
      k8s:
        kubeconfig: </path/to/.kube/config>
        api_version: v1
        kind: ServiceAccount
        name: "{{ item.name }}"
        namespace: "{{ item.namespace }}"
        state: present
      loop: "{{ service_accounts }}"



```

<br>

<br>

<img width="1185" alt="playbook" src="https://github.com/earchibong/eks-ansible/assets/92983658/b698c960-8635-4ea0-8271-75080f8aaa9e">

<br>

<br>

## Run Playbook
```

 ansible-playbook playbooks/eks_setup.yml -i inventories/eks_cluster.yml


```

<br>

<br>
