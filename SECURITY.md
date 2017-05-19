# Kubernetes Security

This document contains aspects which should be considered for secure Kubernetes setup and application deployment. Every application is different and not one configuration fits them all. This is therefore not written as list of things which have to be donem, but list of things which should be considered. You should always have a good reason if you decide to skip any of these points.

* [Namespaces](#namespaces)
* [Resource Quotas](#resource-quotas)
* [Service accounts and RBAC](#service-accounts-and-rbac)
* [Security Context / Pod Security Policies](#security-context--pod-security-policies)
* [Network Policies](#network-policies)
* [AppArmor](#AppArmor)
* [SecComp](#seccomp)
* [SELinux](#selinux)

## Namespaces

Separate applications / different layers of applications into their own namespaces

## Resource quotas

Kubernetes namespaces support resource quotas. Resource quotas allow to specify how many resources can be consumed by a namespace.

## Service accounts and RBAC

Every pod is assigned an service account which it can use for communication with the Kubernetes APIs. The rights for these accounts can be configured using RBAC (Role Based Access Control). Different pods / components should have their own service accounts with only the rights they really need to have. For example most pods will not need to create another pods, namespaces etc.

## Security Context / Pod Security Policies

*A Pod Security Policy is a cluster-level resource that controls the actions that a pod can perform and what it has the ability to access. The PodSecurityPolicy objects define a set of conditions that a pod must run with in order to be accepted into the system.* (source: [kubernetes.io](https://kubernetes.io/docs/concepts/policy/pod-security-policy/))

Pod security Policy allows to define requirements which Pods must follow. This includes things like not allowing the pods to run any containers as root, setting SELinux context, access to volumes or networks. Security Context / Security Policies should be used to enforce these rules from the containers.

## Kube2IAM

When running on Amazon AWS, Kubernetes master / work nodes have assigned IAM roles. These can be in theory used by the Pods. But it is not always intended that the Pods share the same roles as the nodes and aslo to make some use of them you would need the role to be the union of all roles needed by different applications. Kube2IAM can be used to protect the node IAM role and/or add additional IAM roles to specific Pods / containers.

## Network policies

Kubernetes support ingress network policies. Ingress policies control incomming connections and define from where you can connect to your pod. Ingress policies should be used to define which of your microservices can connect to other microservices and otherwise block the access on the network level.

Some SDNs (for examle Tigera / Calico) support also egress network policies. Egress network policies control outgoing connections and can for example define which pods can connect to internet and download things. Again, the access should be restricted as possible. 

## AppArmor

*AppArmor ("Application") is a Linux kernel security module that allows the system administrator to restrict programs' capabilities with per-program profiles. Profiles can allow capabilities like network access, raw socket access, and the permission to read, write, or execute files on matching paths.* (source: [Wikipedia](https://www.google.de/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwjpue_68NDTAhXOLlAKHf4jC6MQFggoMAE&url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FAppArmor&usg=AFQjCNG7h0Gajd1CDhJqrIKHJeQ-T1Zbng&sig2=eINug07fANcqbIlpkYD8Lw))

AppArmor should be used for all applications, at least with the default profile.

## SecComp

*A large number of system calls are exposed to every userland process with many of them going unused for the entire lifetime of the process. As system calls change and mature, bugs are found and eradicated. A certain subset of userland applications benefit by having a reduced set of available system calls.  The resulting set reduces the total kernel surface exposed to the application.  System call filtering is meant for use with those applications. Seccomp filtering provides a means for a process to specify a filter for incoming system calls.* (source: [kernel.org](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt))

SecComp should be used for all applications, at least with the default profile. It can be used to for example restrict access to kernel calls which contain a security risk etc.

## SELinux

Kubernetes support SELinux. That allows to run Kubernetes on hosts with SELinux enabled and there is no need to disable it.
