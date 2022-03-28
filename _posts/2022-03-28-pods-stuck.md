---
layout: post
title: Pods stuck in ContainerCreating status -  failed to assign an IP address to the container
author: Angel Mary Babu
date: 2022-03-28 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops ]
math: true
mermaid: true
description: Pods stuck in ContainerCreating status -  failed to assign an IP address to the container

---


# Pods stuck in ContainerCreating status -  failed to assign an IP address to the container  

In this blog, I would like to describe a situation where we faced an issue that the Pods are stuck in ContainerCreating state after the deployment in an EKS environment and How I resolved the issue. 

##### Situation:
We were having the EKS version 1.20 and with self-managed worker nodes with VPC CNI add-on. The worker nodes are in type t3a.xlarge and when we initiate a deployment, every time the Pods are getting in a ContainerCreating state with the following errors,
```
Event  : sample-deployment   Pod    sample-deployment-76845697dc-vdlzp   sample     Failed to create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container "a25a3b78c48165349e770b7e446ff0ca1d39bb61cbf6ba02e1fcf737e23432ee" network for pod "sample-deployment-76845697dc-vdlzp": networkPlugin cni failed to set up pod "sample-deployment-76845697dc-vdlzp_payroll" network: add cmd: failed to assign an IP address to container   FailedCreatePodSandBox
```

In this environment, we had simultaneous deployments at every time and this issue caused financial problems. Also, the work was way behind the deadline. 

##### Troubleshooting 

1. We were able to recreate the issue during the rolling update.
2. We tried to upgrade the EKS version to 1.21 and upgraded the VPC CNI plugin.
3. The EKS Cluster and Worker Nodes are in a separate Virtual Private Network. Worker nodes are in the private subnet which has enough IP addresses to allocate pods. However, we tried to add a subnet but it didn't resolve the issue.
4. We had an assumption that the max ENIs is reached and it is not able to add any more pods. It happens when the pod IPs are in cooldown and new pods get scheduled on these nodes.
5. The current instance type t3a.xlarge could hold up to 58 pods in the nodes which weren't sufficient in this case during the rolling update.
Reference:  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
6. Hence, we tried to change the instance type to m5 series, and still we faced the same issue.
7. I have tried almost every solution and we were in the same situation as before during the rolling update.

##### Solution: 

 To mitigate the issue, we enabled the option `ENABLE_PREFIX_DELEGATION` which will increase the number of available IP addresses in an EC2 node up to 110.

```
~$ ./max-pods-calculator.sh --instance-type t3a.xlarge --cni-version 1.9.3-eksbuild.1 --cni-prefix-delegation-enabled
110
```

This feature enabled us more room for the pods in the worker node and the issue never occurred again in the environment.

##### Steps followed:

Please note the following before applying the feature.
* The worker nodes must be Nitro-based instances. 
( Reference: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances) 
* The VPC CNI version must be 1.9.0 or higher. If the version is lower, you have to update it. Please refer to the following: https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html
* The VPC subnet must have enough free IP addresses.

Steps:
1. Confirm the current version of the VPC CNI plugin.
```
kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
```
2. Now we have to enable the parameter to assign prefixes to network interfaces for the Amazon VPC CNI daemonset. 
```
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
```
3. The variable `WARM_PREFIX_TARGET` has the default value 1. Hence, there are no changes needed.
4. Since the Worker nodes are configured as self-managed, we have to add the following argument in the `BootstrapArguments` in the CloudFormation.
```
--use-max-pods false --kubelet-extra-args '--max-pods=110'

```
5. Once the nodes are ready, we can see the allocation of the nodes has been increased.
```
kubectl describe node node-name ip-192-168-22-103.us-west-2.compute.internal

Output:
...
Allocatable:
  attachable-volumes-aws-ebs:  25
  cpu:                         1930m
  ephemeral-storage:           76224326324
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      7244720Ki
  pods:                        110
...
```
Reference: https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html

That's it!
