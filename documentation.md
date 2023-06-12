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
- <a href=" "> Install ansible & dependencies</a>

<br>

<br>

## Pre-requisites
Before you begin, make sure you have the following prerequisites in place:

- An EKS cluster set up and configured.

You can find out how to do this with `terraform` from <a href="https://github.com/earchibong/terraform-eks">this tutorial</a> or with `eksctl` with <a href="https://github.com/earchibong/devops_projects/blob/main/kubenetes_05.md">this tutorial.</a>

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

- install ansible

*you can use the <a href="https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html">offical ansible documentation</a>*

```

sudo yum update
sudo amazon-linux-extras install ansible2

# Install the required dependencies for working with EKS
sudo pip install boto3 botocore

```

<br>

<br>


### Set Up Project Structure

- Create a directory structure for  Ansible project.

```

ansible-eks-project/
├── inventories/
│   └── eks_cluster/
└── playbooks/
    ├── eks_setup.yml
    └── files/
        └── eks_data.yaml


```

<br>

<br>


