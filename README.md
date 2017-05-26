# AWS deployment

This repository contains the Ansible scripts for deploying a Kubernetes cluster into DBG ProductDev Amazon AWS account. This account has severe limitations in what the users can do and that is why regular tools such as Kops don't work. This repository is based on the [AWS QuickStart guide](https://aws.amazon.com/quickstart/architecture/heptio-kubernetes/) from Heptio. But the CloudFormation stack is adapted. It will probably not work in any other AWS accounts.

* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Uninstallation](#uninstallation)
* [Configuration](#configuration)
* [IAM roles](#iam-roles)
* [Create cluster](#create-kubernetes-cluster)
* [Use cluster](#use-kubernetes-cluster)
* [Delete cluster](#delete-kubernetes-cluster)
* [Add-ons](#add-ons)
* [Ingress](#ingress)
* [Tagging Lambda](#tagging-lambda)
* [Next steps](#next-steps)
  * [Creating additional users](#creating-additional-users)
  * [Services with Type=LoadBalancer](#services-with-typeloadbalancer)
  * [DNS records](#dns-records)

## Prerequisites

* Install Ansible
* Install kubectl (see below)
* Install AWS CLI
* Login to AWS, place credentials in env vars
* Install jq (json processor):

```
sudo yum install jq
```

* create ~/.kube

## Installation

1. Change the configuration (cluster name, tags, etc.)
2. Create the IAM roles
3. Create the cluster
4. Install tagging lambda function
5. Install the addons
6. Install ingress (optional)

## Uninstallation

1. Delete all namespaces from the cluster (apart from kube-system)
2. Delete the cluster
3. Delete the tagging lambda
4. Check manually in AWS that all auxilary resources has been properly deleted (loadbalancers, volumes etc.)

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
| `dns_zone` | Hosted DNS zone which has to exist in Route53 | `dev.dbgcloud.io` |
| `elb_hosted_zone` | The hosted zone in which the aliased ELB load balancers are hosted (should be dependent on the AWS region) | `Z32O12XQLNTSW2` |
| `ec2_instance_type` | EC2 size of the nodes used as workers. | `t2.medium` |
| `ec2_disk_size` | Disk size for the EC2 instances (in GB). | `40` |
| `kubernetes_node_count` | Number of EC2 worker hosts. | `2` |
| `kubernetes_network` | Defines which networking plugin should be used in Kubernetes. Tested with Calico only. | `calico` |
| `template_url` | DBG Role template URL used for the Cloud Formation stack | |
| `iam_prefix` | DBG role / policy name prefix | `DBG-DEV-` |

## IAM roles

The IAM roles needed for the setup of this Kubernetes cluster can be created by running:
```
ansible-playbook create-roles.yaml
```

Because due to some useless security restrictions, we cannot delete the IAM roles and that makes the life complicated, this is a separate playbook instead of being part of the main playbook as would be more appropriate.

## Create Kubernetes cluster

The Kubernetes cluster can be created by running
```
ansible-playbook create-cluster.yaml
```

Cluster creation will generate a CloudFormation template. IT will upload this template to a newly created bucket. IT will create a new EC2 KeyPair based on the specified SSH key. Afterwards it will create the Cloud Formation stack and execute it.

## Use Kubernetes cluster

The cluster can be accessed via kubectl. A config file is created during the setup. It can be advertised to kubectl via environment variable:

```
export KUBECONFIG=~/.kube/config-wk359-k8s-qs
kubectl cluster-info  
```

## Delete Kubernetes cluster

To delete the cluster run
```
ansible-playbook delete-cluster.yaml
```

This will delete the CloudFormation stack, the resources it created, the SSH key and the S3 bucket.

## Add-ons

Currently, the supported add-ons are:
* Kubernetes dashboard
* Heapster for resource monitoring
* Storage class for automatic provisioning of persisitent volumes

To install the add-ons run
```
ansible-playbook addons.yaml
```

## Ingress

Ingress can be used route inbound traffic from the outside of the Kubernetes cluster. It can be used for SSL termination, virtual hosts, load balancing etc. For more details about ingress, go to [Kubernetes website](https://kubernetes.io/docs/concepts/services-networking/ingress/).

To install ingress controller based on Nginx, run
```
ansible-playbook ingress.yaml
```

After installation, the Ingress will be available under the hostname `ingress.<ClusterName>.<DNSZone>`.

## Tagging lambda

The AWS Lambda function for tagging of resources (the related IAM and CloudWatch objects) can be also installed and uninstalled separately. To install it run:
```
ansible-playbook install-lambda.yaml
```

To uninstall it run:
```
ansible-playbook uninstall-lambda.yaml
```

## Next steps

### Creating additional users

The guide how to add new user has been moved to my [blog](http://blog.effectivemessaging.com/2017/05/adding-users-on-quick-start-for.html).

### Services with Type=LoadBalancer

The Deutsche Boerse ProductDev account has no connection to the internet. Therefore all ELB load balancers have to be created as internal. To create the load balancer as internal, use following annotation for your service:

```
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: '0.0.0.0/0'
```

### DNS records

The `addons.yaml` playbook will install several addons. One of them is the Route53 mapper. Route53 mapper allows you to create DNS records for your services directly from Kubernetes. To use it, you have to tag you service with tag `dns` and value `route53`:

```
  labels:
    dns: route53
    ...
```

And specify the desired domain name in annotation:

```
  annotations:
    domainName: "ghost.dev.dbgcloud.io"
```

The Route53 mapper is periodically checking the services and creating the Route53 DNS records for them. The domain you can use in ProductDev AWS account is `dev.dbgcloud.io`. Once you delete the service, the Route53 mapper doesn't delete the Route53 records automatically. You have to do it on your own.
