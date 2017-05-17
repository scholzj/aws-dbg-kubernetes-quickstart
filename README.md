# AWS deployment

This repository contains the Ansible scripts for deploying a Kubernetes cluster into DBG ProductDev Amazon AWS account. This account has severe limitations in what the users can do and that is why regular tools such as Kops don't work. This repository is based on the [AWS QuickStart guide](https://aws.amazon.com/quickstart/architecture/heptio-kubernetes/) from Heptio. But the CloudFormation stack is adapted. It will probably not work in any other AWS accounts.

## Prerequisites

* Install Ansible
* Install kubectl (see below)
* Install AWS CLI

## Configuration

The main configuration of the cluster is in the variables in `group_vars/all/vars.yaml`. Following table shows the different options.

| Option | Explanation | Example |
|--------|-------------|---------|
| `cluster_name` | Name of the cluster which should be created. The name should be DNS compatible. It will be used to name different resources etc. | `my-k8s-cluster` |
| `kubeconfig_file` | Where should the `kubectl` configuration downloaded. | `~/.kube/my-k8s-cluster-config` |
| `tags` | A dictionary with different tags which should be used for resources. The tags have to full fill the DBG requirements. | |
| `ssh_public_key` | Path to the public part of the SSH key, which should be used for the SSH access to the Kubernetes hosts | `~/.ssh/id_rsa.pub` |
| `ssh_private_key` | Path to the private part of the SSH key, which should be used for the SSH access to the Kubernetes hosts | `~/.ssh/id_rsa` |
| `ssh_cidr` | A CIDR from which SSH access is allowed | `172.35.0.0/16` |
| `api_cidr` | A CIDR from which Kubernetes API access is allowed | `172.35.0.0/16` |
| `aws_region` | AWS region where the cluster should be installed. | `eu-west-1` |
| `aws_zones` | List of availability zones in which the cluster should be installed. | `eu-west-1a` |
| `vpc_id` | ID of the existing Amazon AWS VPC where the cluster should be placed. | `vpc-9694d3fe` |
| `subnet_id` | ID of the existing Amazon AWS subnet where the cluster should be placed. | `subnet-ca9cdca2` |
| `ec2_instance_type` | EC2 size of the nodes used as workers. | `t2.medium` |
| `ec2_disk_size` | Disk size for the EC2 instances (in GB). | `40` |
| `kubernetes_node_count` | Number of EC2 worker hosts. | `2` |
| `kubernetes_network` | Defines which networking plugin should be used in Kubernetes. Tested with Calico only. | `calico` |
| `template_url` | DBG Role template URL used for the Cloud Formation stack | |
| `iam_prefix` | DBG role / policy name prefix | `DBG-DEV-` |

## Create IAM roles

The IAM roles needed for the setup of this Kubernetes cluster can be created by running:
```
ansible-playbook create_roles.yaml
```

Because due to some useless security restrictions, we cannot delete the IAM roles and that makes the life complicated, this is a separate playbook instead of being part of the main playbook as would be more appropriate.

## Create Kubernetes cluster

The Kubernetes cluster can be created by running
```
ansible-playbook create-cluster.yaml
```

Cluster creation will generate a CloudFormation template. IT will upload this template to a newly created bucket. IT will create a new EC2 KeyPair based on the specified SSH key. Afterwards it will create the Cloud Formation stack and execute it.

## Delete Kubernetes cluster

To delete the cluster run
```
ansible-playbook delete-cluster.yaml
```

This will delete the CloudFormation stack, the resources it created, the SSH key and the S3 bucket.

## Install add-ons

Currently, the supported add-ons are:
* Kubernetes dashboard
* Heapster for resource monitoring
* Storage class for automatic provisioning of persisitent volumes

To install the add-ons run
```
ansible-playbook addons.yaml
```

## Install ingress

**TODO!!!**

Ingress can be used route inbound traffic from the outside of the Kubernetes cluster. It can be used for SSL termination, virtual hosts, load balancing etc. For more details about ingress, go to [Kubernetes website](https://kubernetes.io/docs/concepts/services-networking/ingress/).

To install ingress controller based on Nginx, run
```
ansible-playbook ingress.yaml
```

## Install and deleting the tagging lambda function

**TODO!!!**

The AWS Lambda function for tagging of resources (the related IAM and CloudWatch objects) can be also installed and uninstalled separately. To install it run:
```
ansible-playbook install-lambda.yaml
```

To uninstall it run:
```
ansible-playbook uninstall-lambda.yaml
```
