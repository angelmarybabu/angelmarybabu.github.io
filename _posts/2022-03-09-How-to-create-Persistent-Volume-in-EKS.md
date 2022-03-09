---
layout: post
title: How to create Persistent Volume in EKS?
author: Angel Mary Babu
date: 2022-03-09 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops, ]
math: true
mermaid: true
description: How to create Persistent Volume in EKS?

---
#### Prerequisite
* You should have a EKS cluster configured
* Kubectl command line tool should be available
* AWS cli should be available

In order to create a Persistent Volume in EKS, we have to check whether the cluster has the corresponding storage version, example:gp2. In the latest EKS versions, it will be automatically installed or else we can deploy it using the following steps;

1. Check whether we already have the storage class available.
```
 kubectl get sc
```
2. If not available, create with the following manifest file for storage class.

```
vim gp2-storage-class.yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4 
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-east-1c
    - us-east-1d
```

Here we have included a topology for the zones.

[Reference](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)

3. Now we could create volume, PV, PVC and a sample deployment. Since we have provided the Binding Mode as `WaitForFirstConsumer`, we have to create the corresponding resources to get this pv and pvc bounded.

Create a volume:
```
aws ec2 create-volume --availability-zone=us-east-1c --size=10 --volume-type=gp2 --region=us-east-1 --encrypted
```

Sample PV manifest file:
```
vim pv-sample.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: crypto-teleport-pv
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: aws://us-east-1c/vol-0c4680ebxxxx
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Delete
  storageClassName: gp2
  volumeMode: Filesystem
```

```kubectl apply -f pv-sample.yaml```

Sample PVC manifest file:
```
vim pvc-sample.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: test-angel 
  name: nginx-with-pvc
  labels:
    name: nginx-with-pvc

spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
```
```kubectl apply -f pvc-sample.yaml```

Sample deployment manifest file,

```
vim deployment.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-pvc
  namespace: test-angel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
       app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-with-pvc
        volumeMounts:
        - mountPath: /test-ebs
          name: my-pvc
      volumes:
      - name: my-pvc
        persistentVolumeClaim:
          claimName: nginx-with-pvc
```
[Reference](https://github.com/angelmarybabu/persistent-volume)

Now we could see that the PVC has been bounded to the PV.

In some cases, if we need a different storage class, then we have to consider ebs-sc driver. Please find the steps for the following,

`Note: Eks version must be atleast 1.18`

1. Create an IAM policy from the AWS console.
```
curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
```

```
aws iam create-policy \
    --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
    --policy-document file://example-iam-policy.json
```
2. Attach this policy to an IAM user
```
IAM >> Roles >> Create role >> Select trusted entity >> Web identity (choose the URL for your cluster) >> Audience, ( choose sts.amazonaws.com) >> Add permissions >> choose the AmazonEKS_EBS_CSI_Driver_Policy >> Role name (AmazonEKSEBSCSIRole) >> Create role.
```
Now we have to make the following changes for the role.

Select the role AmazonEKSEBSCSIRole >> Choose the Trust relationships tab >>  choose Edit trust policy.

Then change the line from, `"oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"` to `"oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"`

Replace EXAMPLED539D4633E53DE1B716D3041E with your cluster's OIDC provider ID, region-code with the AWS Region code that your cluster is in, and aud (in the previous output) to sub.

3. Install the Amazon EBS CSI driver by using Helm. Add the corresponding helm repository,
```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```
4. Install the driver.
```
helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
    --namespace kube-system \
    --set image.repository=123456789012.dkr.ecr.region-code.amazonaws.com/eks/aws-ebs-csi-driver \
    --set controller.serviceAccount.create=true \
    --set controller.serviceAccount.name=ebs-csi-controller-sa \
    --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole"
```

5. Now we could create the storage class in the cluster.
```
vim storageclass-ebs-csi.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  encrypted: 'true'
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```
[Reference](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi-self-managed-add-on.html)

Now we can dynamically provision the volume with type gp2. Also, we can change the type gp3 as per our need and it is highly recommened to use gp3 which is more efficiant than gp2.

Example:
```
vim pvc-sample.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: test-angel 
  name: nginx-with-pvc
  labels:
    name: nginx-with-pvc

spec:
  storageClassName: ebs-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
```
`kubectl apply -f pvc-sample.yaml`

Now create a deployment,
sample;
```
vim deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-pvc
  namespace: test-angel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
       app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-with-pvc
        volumeMounts:
        - mountPath: /test-ebs
          name: my-pvc
      volumes:
      - name: my-pvc
        persistentVolumeClaim:
          claimName: nginx-with-pvc
```
[Reference](https://github.com/angelmarybabu/dynamic-provisioning-pv)

That's it.
