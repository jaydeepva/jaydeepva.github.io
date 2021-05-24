# Secure your workload with OpenShift SCC and RBAC

## Introduction

An Operator is a method of packaging, deploying and managing a Kubernetes application. Operators may require different privileges than the resources they create (your application). This article discuss security considerations as you build operator and the application that operator deploys.

Lets first look at a couple of basics

## OpenShift SCC

Administrators can use security context constraints (SCCs) to control permissions for pods. These permissions include actions that a pod, a collection of containers, can perform and what resources it can access. You can use SCCs to define a set of conditions that a pod must run with in order to be accepted into the system.

OpenShift provides following SCCs out of the box. See more details here https://docs.openshift.com/container-platform/4.6/authentication/managing-security-context-constraints.html

```
$ oc get scc
NAME               PRIV    RUNASUSER          FSGROUP     SUPGROUP
anyuid             false   RunAsAny           RunAsAny    RunAsAny
hostaccess         false   MustRunAsRange     MustRunAs   RunAsAny
hostmount-anyuid   false   RunAsAny           RunAsAny    RunAsAny
hostnetwork        false   MustRunAsRange     MustRunAs   MustRunAs
node-exporter      true    RunAsAny           RunAsAny    RunAsAny
nonroot            false   MustRunAsNonRoot   RunAsAny    RunAsAny
privileged         true    RunAsAny           RunAsAny    RunAsAny
restricted         false   MustRunAsRange     MustRunAs   RunAsAny
```

## Service account

A service account is an OpenShift Container Platform account that allows a component to directly access the API. Service accounts are API objects that exist within each project. Service accounts provide a flexible way to control API access without sharing a regular user’s credentials. See more details at https://docs.openshift.com/container-platform/4.6/authentication/using-service-accounts-in-applications.html

# Best practices

1. Follow the principle of least privilege. Let your operator create RBAC and ServiceAccount for your application to use at runtime. Allow for only required permissions and privileges.

2. Define a ServiceAccount and use that to run your application pods. Define Role and RoleBinding for the ServiceAccount.

3. Prefer restricted SCC. If your application cannot run under restricted SCC, define a custom SCC based on restricted SCC and fine tune it for your application. Fine tuning your custom SCC requires removing as many additional permissions as you can that allows your application to run as expected. 

4. Define custom SCC for your application. This makes the your application not dependent on the cluster it is running on. Declaring a custom SCC also allows your application to define the exact security permissions required.

5. Prefer Namespace-scoped Operators than cluster-scoped as they limit the permissions required to execute.

# What you should do?

## Derive from existing SCC to build your custom SCC

Use OpenShift out of the box SCC to start building your custom SCC

```
$ oc export scc restricted > my-custom-scc.yaml
```

## Configure runAsUser in your custom SCC

Prefer MustRunAs or MustRunAsRange over RunAsAny. MustRunAs requires a runAsUser to be configured in pod specifications.

Snippet from SCC:

```
runAsUser:
  type: MustRunAs
```

Pod specification sample:

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000 
```

MustRunAsRange requires minimum and maximum values to be defined if not using pre-allocated values.

Snippet from SCC:

```
runAsUser:
  type: MustRunAsRange 
  uidRangeMin: 1000
  uidRangeMax: 2000
```

Pod specification sample:

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1001
```

## Configure fsGroups in your custom SCC

fsGroup is used for controlling access to block storage, such as Ceph RBD and iSCSI. fsGroup defines a pod’s "file system group" ID, which is added to the container’s supplemental groups. 

Prefer using MustRunAs. MustRunAs requires at least one range to be specified if not using pre-allocated values.

Snippet from SCC:

```
fsGroup:
  type: MustRunAs 
  ranges: 
  - max: 6000
    min: 5000 
```

Pod specification sample:

```
kind: Pod
...
spec:
  securityContext: 
    fsGroup: 5555 
```

## Configure supplemenGroups for your custom SCC

Supplemental groups are regular Linux groups. When a process runs in Linux, it has a UID, a GID, and one or more supplemental groups. These attributes can be set for a container’s main process. The supplementalGroups IDs are typically used for controlling access to shared storage, such as NFS and GlusterFS.

Prefer using MustRunAs. MustRunAs requires at least one range to be specified if not using pre-allocated values. Uses the minimum value of the first range as the default.

Snippet from SCC:

```
supplementalGroups:
  type: MustRunAs 
  ranges: 
  - max: 6000
    min: 5000 
```

Pod specification sample:

```
apiVersion: v1
kind: Pod
...
spec:
  securityContext: 
    supplementalGroups: [5555]
```

## Use your custom SCC

Save your custom SCC definition in a yml file and apply it in your OpenShift cluster

```
$ oc create -f my-custom-scc.yml
```

Link your custom SCC to your custom service account.

```
$ oc adm policy add-scc-to-user <your-service-account-name> -z <your-custom-scc-name>
```

Reference your custom service account in your pod definition or deployment definition

```
kind: Pod
...
spec:
  containers:
  - name: ...
  serviceAccountName: <your-service-account-name>
```

## Handling operations in your operator

Considering you are using Ansible based operator, you can create Ansible task to create your custom service account before the application pods are deployed. Its recommended to use a different service account for Operator itself since operator may need more privilages.

```
---
# tasks/main.yml
- name: Create custom service account
  k8s:
   state: "present"
   definition: "{{ lookup('template', 'my-service-account-template.yaml') }}"

---

# templates/my-service-account-template.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: '{{ meta.namespace }}'
  labels:
    ...
```
