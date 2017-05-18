# AWS deployment

This repository contains the Ansible scripts for deploying a Kubernetes cluster into DBG ProductDev Amazon AWS account. This account has severe limitations in what the users can do and that is why regular tools such as Kops don't work. This repository is based on the [AWS QuickStart guide](https://aws.amazon.com/quickstart/architecture/heptio-kubernetes/) from Heptio. But the CloudFormation stack is adapted. It will probably not work in any other AWS accounts.

* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Uninstallation](#uninstallation)
* [Configuration](#configuration)
* [IAM roles](#iam-roles)
* [Create cluster](#create-kubernetes-cluster)
* [Delete cluster](#delete-kubernetes-cluster)
* [Add-ons](#add-ons)
* [Ingress](#ingress)
* [Tagging Lambda](#tagging-lambda)
* [Next steps](#next-steps)
  * [Creating additional users](#creating-additional-users)

## Prerequisites

* Install Ansible
* Install kubectl (see below)
* Install AWS CLI
* Login to AWS, place credentials in env vars.
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

The cluster is using TLS client certificates for user management. You can add new users by issuing new user certificates and assigning them to roles.

* Generate a new TLS key:
```bash
openssl genrsa -out jakub.pem 2048
```

* Generate a signing request based on this key. Use the desired username in the subject:
```bash
openssl req -new -key jakub.pem -out jakub.csr -subj "/CN=jakub"
```

* Encode the signing request with base64:
```bash
cat jakub.csr | base64 | tr -d '\n'
```

* Request Kubernetes cluster to sign the certificate by creating CertificateSigningReqquest resource YAML file:
```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user-request-jakub
spec:
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZUQ0NBVDBDQVFBd0VERU9NQXdHQTFVRUF3d0ZhbUZyZFdJd2dnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQQpBNElCRHdBd2dnRUtBb0lCQVFDNGxQMTNVZHlBTDZ1QUttTVROeDU5RUtlTE14VkNWSTJVWUhWL2hvYjdpelBGCldRSmxaY1dEWllnL1dIVTJoMFJ3TGNkeWpaVTVHcTFPdS9IMjQxSWpBL3dTMXJSWnc1c0ZrNG8zYWdBZlJ0QngKbzZDc3czS0Q3eEpvdEtpRURzOXFqbUJPNzFwU1kzK3BqdHBZbUpxWmZ5cmV2Y094Wm12NXpSU2NaWUZkb1ZMdwptb0VxeUx0a1hDNzVpeXBGTFg2WjJwK25POFE2NHFtU2VZWHRjNFZBbjZhbjdlUFJFU3k5S09NUURZTHRUZENBClBaTVpDeXV0WnBXQnFqMXZlV3NkaWxweUlyaXdRYWtNQ2tBdURHbUxGbzJhbFk3REV4Z0t0ay8rZlZSaEkyY1YKRUJzVUptaVBuZ1NrMXhKRzZjWjdUZE12UXYxSnIyaGNsc1NDY1IySkFnTUJBQUdnQURBTkJna3Foa2lHOXcwQgpBUXNGQUFPQ0FRRUFiL1BBcVp0eUovUHA2MWtXcG54azNDek9SNTdhT09JSHU3YkVUN1lmcE9GazFSN0JkV3hWClU1RXUrK3pTNVZmQm9XaVNNekV1eENrVHZsNU1CVGxDL3NvQWprT2YzQTI2aUFpeFJibksrUWw0ZGt1MGJQTisKT2syZ3pqTXZyM1hsT1ExZVZnQytjTVZzeXJpZENUMzRwcHVPNm9RTkZnVk1GblZ3bHozdjJnZkVDK3c1RnVndgpOVXcxT21td0c1RTZKN0VpL2ZSTk1scnUyTzZaUlQvMGYyWWpnSUhwT3RONTJYSDhhc3cvMjhMOWt1c1J6aHR5CnFLa0xxWGhkYW4xSDRsR1E2VUwxd1V4UU8zUWhyRnUrbkhFRTV3SVdDNEFra0N4K1NPQ3RpalNmV2Vhb3dkTzcKRG8zaWJvWGNwTzVSUDYvYUhEWE9kV0ZXc2EvQituVG9Vdz09Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  usages:
  - digital signature
  - key encipherment
  - client auth
```

and pass it to Kubernetes:
```bash
kubectl create -f csr.yaml
```

* Approve the certificate to get it signed:
```
kubectl certificate approve user-request-jakub
```

* Download the signed public key from Kubernetes and decode it from base64:
```bash
kubectl get csr user-request-jakub -o jsonpath='{.status.certificate}' | base64 -d > jakub.crt
```

* Create a new `kubectl` config file, add cluster, user and context:
```bash
kubectl --kubeconfig ~/.kube/config-jakub config set-cluster jakub --insecure-skip-tls-verify=true --server=https://api.my-cluster.dev.dbgcloud.io
kubectl --kubeconfig ~/.kube/config-jakub config set-credentials jakub --client-certificate=jakub.crt --client-key=jakub.pem --embed-certs=true
kubectl --kubeconfig ~/.kube/config-jakub config set-context jakub --cluster=jakub --user=jakub
kubectl --kubeconfig ~/.kube/config-jakub config use-context jakub
```

* Test that the user authenticates:
```bash
kubectl --kubeconfig ~/.kube/config-jakub get pods
```
You should be able to connect, but you should get and authorization error:
```
Error from server (Forbidden): User "jakub" cannot list pods in the namespace "default". (get pods)
```

* To move forward, you have to assign the user some roles. The two examples below cover a *view* role for all namespaces (see all running pods etc.) and an *edit* role for the default namespace.

* To give the user the view access, you have to create following CluertRoleBinding resource and load it into Kubernetes:
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jakub-view-all
subjects:
- kind: User
  name: jakub
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

* To give the user rights to deploy applications into specific namespace, you have to create following CluertRoleBinding resource and load it into Kubernetes:
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jakub-edit-default
  namespace: default # This only grants permissions within the "default" namespace.
subjects:
- kind: User
  name: jakub
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

* For more details about the authorization, see the [Role-Based Access Control (RBAC) documentation](https://kubernetes.io/docs/admin/authorization/rbac).

* Once you are finished, you just send the new config file to the new user. Just keep it secure :-)
