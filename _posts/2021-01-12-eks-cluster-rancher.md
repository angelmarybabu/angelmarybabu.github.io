---
layout: post
title: How to add an EKS cluster in Rancher?
author: Angel Mary Babu
date: 2022-01-12 11:00:00 +0800
categories: [AWS]
tags: [aws, amazon, rancher]
math: true
mermaid: true
description: How to add an EKS cluster in Rancher?
comments: true
---


Rancher is an open source software platform where we can manage the containerized environments. It helps to create and manage kubernetes clusters via Rancher. Also, we can import already existing cluster to Rancher to manage and configure.

##### Pre-requistes.

* We should have an EKS cluster provisioned.
* Kubectl command line tool should be available


##### Steps Involved: 
1. Login to Rancher Dashboard.
2. Click on cluster.
3. Choose the option `Add Cluster`
<br>
![image-1](https://raw.githubusercontent.com/angelmarybabu/angelmarybabu.github.io/master/assets/media/image-1.png)
<br>
4. Click on the option `Other Cluster`
<br>
![image-2](https://raw.githubusercontent.com/angelmarybabu/angelmarybabu.github.io/master/assets/media/image-2.png)
<br>
5. Provide a meaningful cluster name.
6. Click on `Create`
<br>
![image-1](https://raw.githubusercontent.com/angelmarybabu/angelmarybabu.github.io/master/assets/media/image-3.png)
<br>
7. Apply the given command on the EKS cluster via CLI.

   ```
   kubectl apply -f https://rancher.sample.com/v3/import/g42lk6_c-gfd2j.yaml
   ```
8. Click on `Done`

<br>
![image-1](https://raw.githubusercontent.com/angelmarybabu/angelmarybabu.github.io/master/assets/media/image-4.png)
<br>

Note: Once we have applied the command, a new namespace will be created on the cluster.

Sample output:
```
$ kubectl apply -f https://rancher.domain.com/v3/import/6xhtqc6_c-ggtmz.yaml
clusterrole.rbac.authorization.k8s.io/proxy-clusterrole-kubeapiserver created
clusterrolebinding.rbac.authorization.k8s.io/proxy-role-binding-kubernetes-master created
namespace/cattle-system created
serviceaccount/cattle created
clusterrolebinding.rbac.authorization.k8s.io/cattle-admin-binding created
secret/cattle-credentials-71f3460 created
clusterrole.rbac.authorization.k8s.io/cattle-admin created
deployment.apps/cattle-cluster-agent created
```

We have to allow the NAT gateway IP address for 443 port in the security group of the rancher server. Otherwise we end up in the following error

```
$ kubectl -n cattle-system logs cattle-cluster-agent-5d6b7cd-dwd7d
ERROR: https://rancher.domain.com/ping is not accessible (Failed to connect to rancher.domain.com port 443: Connection timed out)
```

That's it.
