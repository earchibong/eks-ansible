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
- <a href=" ">Set Up Project Structure</a>

<br>

<br>

## Pre-requisites
Before you begin, make sure you have the following prerequisites in place:

- An EKS cluster set up and configured.

You can find out how to do this with `terraform` from <a href="https://github.com/earchibong/terraform-eks">this tutorial</a> or with `eksctl` from <a href="https://github.com/earchibong/devops_projects/blob/main/kubenetes_05.md">this tutorial.</a>

<br>

<br>

## Install And Configure Ansible in Host server
- Set up an SSH agent and connect to `ansible-host` server:

```


# on your local machine:

eval `ssh-agent -s`
ssh-add ./<path-to-private-key>

# Confirm the key has been added:
ssh-add -l


# ssh into Ansible-host server using ssh-agent: 
ssh -A -i "private ec2 key" ec2-user@public_ip


```

<br>

<br>

<img width="797" alt="ssh-agent" src="https://github.com/earchibong/eks-ansible/assets/92983658/e3853666-380c-4863-a626-4c14b118967e">

<br>

<br>

- install ansible

*you can use the <a href="https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html">offical ansible documentation</a>*

```

sudo yum update
sudo amazon-linux-extras install ansible2

# install pip
curl -O https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py --user

ls -a ~
export PATH=.bash_profile:$PATH

# Install the required dependencies for working with EKS
pip install boto3 botocore

# install yamllint
sudo yum install python3-pip
pip3 install ansible-lint


```

<br>

<br>


### Set Up Project Structure

- Create a directory structure for  Ansible project.

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

image 

<br>

<br>

### Set Up Ansible Inventory

- Update `inventories/eks_cluster.yml` file with this snippet of code: ...*notice that the user in this case is `ec2-user` for an `ubuntu` server it would be `ubuntu`*

```

eks_nodes:
  hosts:
    node1:
      ansible_host: <private-ip of node 1>
      ansible_user: ec2-user
      
    node2:
      ansible_host: <private ip of node 2>
      ansible_user: ec2-user

# Alternative syntax:
[eks_nodes]
node1 ansible_host=<public_node1_public_ip> ansible_user=<node_username>
node2 ansible_host=<public_node2_public_ip> ansible_user=<node_username>



```

<br>

<br>

image

<br>

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

<img width="1100" alt="data" src="https://github.com/earchibong/eks-ansible/assets/92983658/196dc196-0d7e-4dec-9c4b-e8414afd9d49">


<br>

<br>

## Create A Playbook

In the playbook, `ansible-lint` is used to validate the YAML file before creating the resources. The `command` module is used to execute the `ansible-lint` command. The `--force-color` option is used to enable colorized output, and `--parseable-severity` option is used to output results in a parseable format.

The `--exclude` option can be used to exclude specific lint rules or directories if needed. Below, the `skip_ansible_lint` rule is excluded. Adjust the `--exclude` option as per your requirements.

The `chdir` parameter specifies the directory where the playbook is located. The register parameter captures the result of the linting command. The `failed_when` parameter ensures the task fails if the linting command returns a non-zero exit code.

In Ansible, YAML syntax validation is performed automatically when executing the playbook. If the YAML file is invalid, the playbook execution will fail with a descriptive error message pointing to the syntax issue in the YAML file.

However, the purpose of using `ansible-lint` or any external YAML linter is to catch potential issues or best practices beyond basic syntax errors. you can perform more comprehensive linting checks specific to Ansible playbooks and roles such as unused variables, missing handlers, incorrect module usage, task ordering as well as YAML syntax validation. This allows you to catch potential issues beyond basic syntax errors and enforce best practices specific to Ansible.

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
    - files/eks_data.yaml

  tasks:
    - name: Validate YAML file
    command: ansible-lint --force-color --parseable-severity --exclude=\
    skip_ansible_lint tests/
    args:
      chdir: "{{ playbook_dir }}"
    changed_when: false
    register: lint_result
    failed_when: lint_result.rc > 0
    ignore_errors: true
    tags:
      - validation
    
    - name: Create namespaces
      k8s:
        api_version: v1
        kind: Namespace
        name: "{{ item }}"
        state: present
      loop: "{{ namespaces }}"
    
    - name: Create roles
      k8s:
        api_version: rbac.authorization.k8s.io/v1
        kind: Role
        name: "{{ item.name }}"
        namespace: "{{ item.namespace }}"
        rules: "{{ item.rules }}"
        state: present
      loop: "{{ roles }}"
    
    - name: Create role bindings
      k8s:
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
    
    - name: Create service accounts
      k8s:
        api_version: v1
        kind: ServiceAccount
        name: "{{ item.name }}"
        namespace: "{{ item.namespace }}"
        state: present
      loop: "{{ service_accounts }}"



```

<br>

<br>

<img width="1112" alt="playbook" src="https://github.com/earchibong/eks-ansible/assets/92983658/0cf2d958-0c9a-4770-abae-42cbb3a43adf">

<br>

<br>

## Run Playbook
```

 ansible-playbook playbooks/eks_setup.yml -i inventories/eks_cluster.yml

```

<br>

<br>
