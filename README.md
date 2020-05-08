


# Enterprise Tanzu Kubernetes Grid with Lifecycle Management on AWS

Tanzu Kubernetes Grid provides Enterprise organizations with a consistent, upstream compatible, regional Kubernetes substrate across SDDC, Public Cloud, and Edge environments that is ready for end-user workloads and ecosystem integrations.

![TKG Architecture](https://github.com/mayureshkrishna/enterprise-tkg-on-aws/blob/master/tkg-architecture.png?raw=true)

TKG uses **management kubernetes cluster** to create and lifecycle manage ***application workload clusters.***

In this tutorial, we will use a local `kind` kubernetes cluster to deploy a **Management TKG cluster**.
And then *Management TKG* cluster to deploy **Workload Clusters.**

## Prerequisites

Tanzu Kubernetes Grid provides CLI binaries for Linux and Mac OS systems. 

**This tutorial uses Mac.**

The bootstrap environment on which you run the Tanzu Kubernetes Grid CLI must meet the following requirements:

-   [`kubectl`](https://v1-17.docs.kubernetes.io/docs/tasks/tools/install-kubectl/)  is installed.
-   [Docker Desktop](https://www.docker.com/products/docker-desktop)  is installed and running, if you are installing Tanzu Kubernetes Grid on Mac OS.
-   System time is synchronized with a Network Time Protocol (NTP) server.
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) - The `kind` container requires at least 6GB of RAM. For information about how to configure Docker Desktop so that it can run `kind`, see [Settings for Docker Desktop](https://kind.sigs.k8s.io/docs/user/quick-start/#settings-for-docker-desktop) in the `kind` documentation.

### Get tkg cli
Downloand tkg cli from [https://www.vmware.com/go/get-tkg](https://www.vmware.com/go/get-tkg)

Unzip and setup tkg cli: 
```
gunzip tkg-darwin-amd64-v1.0.0_vmware.1.gz
mv ./tkg-darwin-amd64-v1.0.0_vmware.1 /usr/local/bin/tkg
chmod +x /usr/local/bin/tkg
tkg version

Client:
Version: v1.0.0
Git commit: 60f6fd5f40101d6b78e95a33334498ecca86176e
```

## Setup AWS Environement 

### Install the  `clusterawsadm`  Utility and Set Up a CloudFormation Stack

Create the following environment variables for your AWS account

```
export AWS_ACCESS_KEY_ID=_aws_access_key_
export AWS_SECRET_ACCESS_KEY=_aws_access_key_secret_
export AWS_REGION=us-east-1
```
Get `clusterawsadm` cli from [https://www.vmware.com/go/get-tkg](https://www.vmware.com/go/get-tkg)

Unzip and setup clusterawsadm cli:
```
gunzip clusterawsadm-darwin-amd64-v0.5.2_vmware.1.gz
mv ./clusterawsadm-darwin-amd64-v0.5.2_vmware.1 /usr/local/bin/clusterawsadm
chmod +x /usr/local/bin/clusterawsadm
```
Run the following `clusterawsadm` command to create a CloudFoundation stack.
```
clusterawsadm alpha bootstrap create-stack

Attempting to create CloudFormation stack cluster-api-provider-aws-sigs-k8s-io
Following resources are in the stack:

Resource  |Type  |Status
AWS::IAM::Group |bootstrapper.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::InstanceProfile |control-plane.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::InstanceProfile |controllers.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::InstanceProfile |nodes.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::ManagedPolicy |arn:aws:iam::026170954811:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::ManagedPolicy |arn:aws:iam::026170954811:policy/nodes.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::ManagedPolicy |arn:aws:iam::026170954811:policy/controllers.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::Role  |control-plane.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::Role  |controllers.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::Role  |nodes.cluster-api-provider-aws.sigs.k8s.io  |CREATE_COMPLETE
AWS::IAM::User  |bootstrapper.cluster-api-provider-aws.sigs.k8s.io
```
### Create SSH Key to use with tkg
```
aws ec2 create-key-pair --key-name default --output json | jq .KeyMaterial -r > default.pem
```
### Create an AWS user to use with tkg

```
export AWS_CREDENTIALS=$(aws iam create-access-key --user-name bootstrapper.cluster-api-provider-aws.sigs.k8s.io --output json)
export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDENTIALS | jq .AccessKey.AccessKeyId -r)
export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDENTIALS | jq .AccessKey.SecretAccessKey -r)
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm alpha bootstrap encode-aws-credentials)
```

### Export AWS AMI for tkg

I'm using us-east-1 AMI as I created the stack for us-east-1
```
export AWS_AMI_ID=ami-0cdd7837e1fdd81f8
```
For the full list of AMI IDs for each AWS region in this release of Tanzu Kubernetes Grid, see the [Tanzu Kubernetes Grid 1.0 Release Notes](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.0/rn/VMware-Tanzu-Kubernetes-Grid-10-Release-Notes.html#amis)

## Install tkg management cluster

Run following to generate a starter config file
```
tkg get management-cluster
```
Edit the config file at ~/.tkg/config.yaml and add following values at the end of file after `images` section

```
AWS_REGION: us-east
AWS_NODE_AZ: us-east-1a
AWS_PUBLIC_NODE_CIDR: 10.0.1.0/24
AWS_PRIVATE_NODE_CIDR: 10.0.0.0/24
AWS_VPC_CIDR: 10.0.0.0/16
CLUSTER_CIDR: 100.96.0.0/11
AWS_SSH_KEY_NAME: default
CONTROL_PLANE_MACHINE_TYPE: t3.small
NODE_MACHINE_TYPE: t3.small
```
### Create the management cluster

```
tkg init --infrastructure=aws --name=tkg-mgmt

Logs of the command execution can also be found at: /var/folders/zk/nhxp0hlj21j3f3fsg7lkt8xh0000gn/T/tkg-20200508T003217491333480.log
Validating the pre-requisites...
Setting up management cluster...
Validating configuration...
Warning: The AMI ID '' provided doesn't match the AMI ID mapped to the Kubernetes version v1.17.3+vmware.2, so using the correct mapped AMI ID ami-0cdd7837e1fdd81f8
Using infrastructure provider aws:v0.5.2
Generating cluster configuration...
Setting up bootstrapper...
Installing providers on bootstrapper...
Fetching providers
Installing cert-manager
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.3" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-aws" Version="v0.5.2" TargetNamespace="capa-system"
Start creating management cluster...
Installing providers on management cluster...
Fetching providers
Installing cert-manager
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.3" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-aws" Version="v0.5.2" TargetNamespace="capa-system"
Waiting for the management cluster to get ready for move...
Moving all Cluster API objects from bootstrap cluster to management cluster...
Performing move...
Discovering Cluster API objects
Moving Cluster API objects Clusters=1
Creating objects in the target cluster
Deleting objects from the source cluster
Context set for management cluster tkg-mgmt as 'tkg-mgmt-admin@tkg-mgmt'.
Management cluster created!

You can now create your first workload cluster by running the following:
tkg create cluster [name] --kubernetes-version=[version] --plan=[plan]
```
## Create tkg workload clusters

Once the management cluster is up and running, now you can create and lifecycle manage as many workload clusters.

Here's the command to create a workload cluster:

```
tkg create cluster tkg-acme-bu1 --plan=dev

Logs of the command execution can also be found at: /var/folders/zk/nhxp0hlj21j3f3fsg7lkt8xh0000gn/T/tkg-20200508T005643445724980.log
Creating workload cluster 'tkg-acme-bu1'...
Warning: The AMI ID '' provided doesn't match the AMI ID mapped to the Kubernetes version v1.17.3+vmware.2, so using the correct mapped AMI ID ami-0cdd7837e1fdd81f8
Context set for workload cluster tkg-acme-bu1 as tkg-acme-bu1-admin@tkg-acme-bu1
Waiting for cluster nodes to be available...
Workload cluster 'tkg-acme-bu1' created
```

### Lets Deploy some Apps

```
kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
```
```
kubectl get pods

NAME 				   READY STATUS  RESTARTS AGE
nginx-86c57db685-d6hvs 1/1 	 Running 0  	  8s
```
