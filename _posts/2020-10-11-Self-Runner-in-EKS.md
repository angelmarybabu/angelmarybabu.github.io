---
layout: post
title: Github Actions Self Runner in EKS
author: Angel Mary Babu
date: 2020-10-11 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops, ]
math: true
mermaid: true
description: Github Actions Self Runner in EKS

---

Let's create an EKS cluster,

##### Create EKS using AWS CLI


```sh
aws eks create-cluster \
   --region us-east-1 \
   --name cluster-new \
   --kubernetes-version 1.21 \
   --role-arn arn:aws:iam::ACCOUNT_ID:role/eksrole \
   --resources-vpc-config subnetIds=subnet-XXXXXXXXXXX,subnet-XXXXXXXXX,subnet-XXXXXXXXX,securityGroupIds=sg-XXXXXXXXX
   ```
   
   This will take a few minutes to create the eks cluster. You can check the status using the following command,
   
   ```
   aws eks --region us-east-1 describe-cluster --name cluster-new --query cluster.status
   ```
Once the EKS cluster status changes to `ACTIVE` we can connect to the EKS cluster using the following,

```
aws eks --region us-east-1 update-kubeconfig --name cluster-new
```
##### Create Worker nodes

We have our cluster is in place, now we have to deploy the Worker nodes. Here I am trying the self managed worker node using the Cloudformation. Just download the YAML file using the curl command at your local end,

```
curl -o amazon-eks-nodegroup.yaml https://raw.githubusercontent.com/awslabs/amazon-eks-ami/master/amazon-eks-nodegroup.yaml
```

Upload the downloaded YAML file in the CloudFormation. CloudFormation >>  Create Stack >> Upload a template file >> Choose this YAML file.

Always make sure to fill the EKS cluster name in the parameters so that the worker node could join the cluster.

Once the Worker nodes have been created, we have to make this worker nodes to join the cluster. Please follow the steps given below,

##### Join the cluster

- Download the configuration map:
``` 
curl -o aws-auth-cm.yaml https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-10-29/aws-auth-cm.yaml
```
- Open the file with your text editor. Replace the <ARN of instance role (not instance profile)> (It can be found from the Cloudformation output page)

Sample aws-auth-cm.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role (not instance profile)>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```
- Now, apply the configuration. 
```
kubectl apply -f aws-auth-cm.yaml
```
##### Self hoster runner on the EKS
Now let's create a self hoster runner on the EKS environment

###### To install helm:


If the helm is not installed then, install the helm with the below steps:
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
[Reference](https://helm.sh/docs/intro/install/)


###### Installing cert-manager on Kubernetes

actions-runner-controller uses cert-manager for certificate management of Admission Webhook. Make sure you have already installed cert-manager before you install. The installation instructions for cert-manager can be found below.

```
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.crds.yaml
```
Output: 

```
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
```
```
$ helm install \
 cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.4.0 \
   # --set installCRDs=true
```
[Reference](https://cert-manager.io/docs/installation/helm/)

###### Deploy Action Runner Controller

Install the custom resource and actions-runner-controller with kubectl. This will create actions-runner-system namespace in your Kubernetes and deploy the required resources.

```
kubectl apply -f https://github.com/actions-runner-controller/actions-runner-controller/releases/download/v0.18.2/actions-runner-controller.yaml
namespace/actions-runner-system created
```
Output:

```
customresourcedefinition.apiextensions.k8s.io/horizontalrunnerautoscalers.actions.summerwind.dev created
customresourcedefinition.apiextensions.k8s.io/runnerdeployments.actions.summerwind.dev created
customresourcedefinition.apiextensions.k8s.io/runnerreplicasets.actions.summerwind.dev created
customresourcedefinition.apiextensions.k8s.io/runners.actions.summerwind.dev created
role.rbac.authorization.k8s.io/leader-election-role created
clusterrole.rbac.authorization.k8s.io/manager-role created
clusterrole.rbac.authorization.k8s.io/proxy-role created
rolebinding.rbac.authorization.k8s.io/leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/proxy-rolebinding created
service/controller-manager-metrics-service created
service/webhook-service created
deployment.apps/controller-manager created
certificate.cert-manager.io/serving-cert created
issuer.cert-manager.io/selfsigned-issuer created
mutatingwebhookconfiguration.admissionregistration.k8s.io/mutating-webhook-configuration created
validatingwebhookconfiguration.admissionregistration.k8s.io/validating-webhook-configuration created
```
###### Setting Up Authentication with GitHub API

Deploying Using PAT Authentication:

Create a personal access token,

1. Login to GitHub account and navigate to https://github.com/settings/tokens
2. Click on Generate new token button
3. Select repo (Full Control) scope.
4. Click Generate Token

Once you have created the appropriate token, deploy it as a secret to your Kubernetes cluster that you are going to deploy the solution on:

```
$ kubectl create secret generic controller-manager \
     -n actions-runner-system \
     --from-literal=github_token=${GITHUB_TOKEN}
```

Now, we have to convert the secret to base64 encoded,

```
echo -n '<Personal-Access-Token>' | base64
```
Update in the secret,

```
kubectl edit secret controller-manager     -n actions-runner-system 
```
###### Deploy Self-Hosted Runner

First, create a namespace to host self-hosted runners resources.

```
kubectl create namespace self-hosted-runners
```
Next, save the following K8s manifest file as self-hosted-runner.yaml, and modify the following:

- Replace <github-repo-name> with your own repository.
- Adjust the minReplicas and maxReplicas as required.

vim self-hosted-runner.yaml

```
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runner-deployment
spec:
  template:
    spec:
      repository: <github-repo-name>
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: runner-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - <github-repo-name>
```
And apply the Kubernetes manifest:
```
kubectl --namespace self-hosted-runners apply -f self-hosted-runner.yaml
```
Verify the runner is deployed and is in the ready state.
```
$ kubectl --namespace self-hosted-runners get runner
```
Now, navigate to your repository Settings > Actions > Runner to view the registered runner.

###### Create a workflow to test your self-hosted runner

Save and commit the following sample GitHub Actions workflow in .github/workflows/actions.yml in your repository where the self-hosted runner is registered.

actions.yaml
```
name: Deploy image to AWS ECR

# Trigger deployment during commit
on: 
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  ecr:
    # Name the Job
    name: build and deploy image to AWS ECR
    # Set the type of machine to run on
    runs-on: self-hosted  # The important part of this workflow is runs-on: self-hosted

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
        
      - name: Check out code
        uses: actions/checkout@v2

 
      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
          ECR_REPOSITORY: demo1
          IMAGE_TAG: dev-${{ github.sha }}
        run: |
          docker build -t demo1 .
          docker tag demo1:latest $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

```
To work with Amazon ECR, first, we need to set up an IAM User. An AWS Identity and Access Management (IAM) user is an entity that you create in AWS to represent the person or application that uses it to interact with AWS.

[Reference](https://github.com/aws-actions/amazon-ecr-login)

Create an IAM user and Add the following inline policy for the IAM user
Example:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPush",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "*"
        }
    ]
}
```
We need to set up these credentials in our GitHub Action, so we can access them like environment variables. Here we will be providing information related to my AWS access keys along with ECR repository information.

[Reference](https://docs.github.com/en/actions/reference/encrypted-secrets)

Now the deployment will trigger whenever any changes are pushed into the repository.
   
Hence, let's create a sample Dockerfile,
  
Dockerfile

```

# ./Dockerfile

FROM node:12-alpine as node-angular-cli

RUN apk update \
  && apk add --update alpine-sdk \
  && apk del alpine-sdk \
  && rm -rf /tmp/* /var/cache/apk/* *.tar.gz ~/.npm \
  && npm cache verify \
  && sed -i -e "s/bin\/ash/bin\/sh/" /etc/passwd

# Angular CLI
RUN npm install -g @angular/cli@8
```

Once the changes has been pushed to the repo, the actions will trigger automatically. 

You should find the logs showing that Listening for Jobs.

Sample output:

```
[ec2-user@ ~]$ kubectl -n self-hosted-runners  logs -f runner-deployment-jlgtc-fzrdz  -c runner
Github endpoint URL https://github.com/
 
--------------------------------------------------------------------------------
|        ____ _ _   _   _       _          _        _   _                      |
|       / ___(_) |_| | | |_   _| |__      / \   ___| |_(_) ___  _ __  ___      |
|      | |  _| | __| |_| | | | | '_ \    / _ \ / __| __| |/ _ \| '_ \/ __|     |
|      | |_| | | |_|  _  | |_| | |_) |  / ___ \ (__| |_| | (_) | | | \__ \     |
|       \____|_|\__|_| |_|\__,_|_.__/  /_/   \_\___|\__|_|\___/|_| |_|___/     |
|                                                                              |
|                       Self-hosted runner registration                        |
|                                                                              |
--------------------------------------------------------------------------------
 
# Authentication
 
 
√ Connected to GitHub
 
# Runner Registration
 
 
 
A runner exists with the same name
√ Successfully replaced the runner
√ Runner connection is good
 
# Runner settings
 
 
√ Settings Saved.
 
Runner has successfully been configured with the following data.
{
  "agentId": 2,
  "agentName": "runner-deployment-jlgtc-fzrdz",
  "poolId": 1,
  "poolName": "Default",
  "serverUrl": "https://pipelines.actions.githubusercontent.com/rdzYJCV3fzV1bREIFyCultSx1LtQ4F0AKUEat9ZtIqn9ue0Itn",
  "gitHubUrl": "https://github.com/ponnu777/github-actions-runner",
  "workFolder": "/runner/_work"
}16c16
< ./externals/node12/bin/node ./bin/RunnerService.js &
---
> ./externals/node12/bin/node ./bin/RunnerService.js $* &
20c20
< wait $PID
---
> wait $PID
\ No newline at end of file
19c19
< var runService = function () {
---
> var runService = function() {
23c23
<     if (!stopping) {
---
>     if(!stopping) {
27c27
<                 listener = childProcess.spawn(listenerExePath, ['run'], { env: process.env });
---
>                 listener = childProcess.spawn(listenerExePath, ['run'].concat(process.argv.slice(3)), { env: process.env });
30c30
<                 listener = childProcess.spawn(listenerExePath, ['run', '--startuptype', 'service'], { env: process.env });
---
>                 listener = childProcess.spawn(listenerExePath, ['run', '--startuptype', 'service'].concat(process.argv.slice(2)), { env: process.env });
33c33
<             console.log(`Started listener process, pid: ${listener.pid}`);
---
>             console.log('Started listener process');
43,46d42
<             listener.on("error", (err) => {
<                 console.log(`Runner listener fail to start with error ${err.message}`);
<             });
< 
64c60
<                 if (!stopping) {
---
>                 if(!stopping) {
69c65
<         } catch (ex) {
---
>         } catch(ex) {
78c74
< var gracefulShutdown = function (code) {
---
> var gracefulShutdown = function(code) {
95c91
< });
---
> });
\ No newline at end of file
.path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/runner/.local/bin
Starting Runner listener with startup type: service
Started listener process
Started running service
 
 
√ Connected to GitHub
 
2021-07-26 11:12:50Z: Listening for Jobs
2021-07-26 11:12:53Z: Running job: build and deploy image to AWS ECR
2021-07-26 11:13:33Z: Job build and deploy image to AWS ECR completed with result: Succeeded
Runner listener exited with error code 0
Runner listener exit with 0 return code, stop the service, no retry needed.

```

That's it!!!
