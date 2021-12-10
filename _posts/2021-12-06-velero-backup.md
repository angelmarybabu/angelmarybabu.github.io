---
layout: post
title: Velero Backup for EKS
author: Angel Mary Babu
date: 2021-12-06 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops, ]
math: true
mermaid: true
description: Velero Backup for AWS

---
#### Prerequisites
* AWS CLI needs to be configured in the machine where you execute Velero commands.
* Kubectl needs to be configured with the EKS cluster where you need to take the backup.

Now we have to create an S3 bucket and IAM user to configure the Velero Backup.

######  To create S3 bucket,

From AWS Console,
go to AWS -> S3 -> create bucket

###### Create an IAM user.

From AWS Console,
go to AWS -> IAM Console -> add user

Add the below permission to the user and replace ${BUCKET} with the S3 bucket name which we created for velero.

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${VELERO_BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${VELERO_BUCKET}"
            ]
        }
    ]
}
```
Create a Velero-specific credentials file (credentials-velero) in your machine:

```bash
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
```

Replace `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` witn your Access key and secret key.

##### INSTALL VELERO Client

Below is a listing of plugin versions and respective Velero versions that are compatible.

| Plugin Version  | Velero Version |
|-----------------|----------------|
| v1.3.x          | v1.7.x         |
| v1.2.x          | v1.6.x         |
| v1.1.x          | v1.5.x         |
| v1.1.x          | v1.4.x         |
| v1.0.x          | v1.3.x         |
| v1.0.x          | v1.2.0         |

###### Install Velero binary:

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.7.0/velero-v1.7.0-linux-amd64.tar.gz
```
Ref: https://github.com/vmware-tanzu/velero/releases

###### Extract the tarball:

```bash
tar -xvf velero-v1.7.0-linux-amd64.tar.gz -C /tmp
```
###### Move the extracted velero binary to /usr/local/bin

```bash
sudo mv /tmp/velero-v1.7.0-linux-amd64/velero /usr/local/bin
```
###### Verify installation
velero version
```bash
$ velero version
Client:
	Version: v1.7.0
	Git commit: 9e52260568430ecb77ac38a677ce74267a8c2176
Server:
	Version: v1.7.0
```

###### Install and start Velero

Install Velero, including all prerequisites, into the cluster and start the deployment. This will create a namespace called velero, and place a deployment named velero in it.
```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket $BUCKET \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --secret-file ./credentials-velero
```

Example
```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket eks-backups \
    --backup-location-config region=us-west-2 \
    --snapshot-location-config region=us-west-2 \
    --secret-file ./credentials-velero
```

Inspect the resources created.
```bash
kubectl get all -n velero
```
Example
```bash
$ kubectl get all -n velero
NAME                         READY   STATUS    RESTARTS   AGE
pod/velero-fbf6dfbc8-qjlp7   1/1     Running   0          3d19h

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           3d19h

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/velero-fbf6dfbc8   1         1         1       3d19h
```

###### DEPLOY TEST APPLICATION
Create namespace and deploy the application
```bash
kubectl create namespace angel

kubectl create deployment nginx --image=nginx -n angel
```

```bash
velero backup create <backupname> --include-namespaces <namespacename>
velero backup create test1 --include-namespaces angel
```

Check the status of backup
```bash
velero backup describe <backupname>
```
Check-in S3 bucket :
backup is stored in the S3 bucket.


Let’s delete the ‘angel’ namespace to simulate a disaster
```bash
kubectl delete namespace angel
```
Restore angel namespace
restore:
Run the velero restore command from the backup created. It may take a couple of minutes to restore the namespace.
```bash
velero restore create --from-backup <backupname>
velero restore create --from-backup test1
```
Verify if deployments, replica sets, services, and pods are restored.


###### To schedule,

```bash
velero schedule create backup-sample --schedule="05 23 * * *" --include-namespaces sample
```

That's it!
