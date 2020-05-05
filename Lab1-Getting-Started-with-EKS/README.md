# EKS Deployment on AWS
The purpose of this simple workshop is to provide guide as a starting of your Kubernetes Journey on top of AWS. We will use eksctl, a simple CLI tool for creating clusters on EKS.
After setting up your EKS Cluster, we will deploy a microservices application. 


## 1. Turn on Cloud9 and install kubectl and aws-iam-authenticator

First step is to prepare a client machine to install kubectl and manage your EKS Cluster/WorkerNodes. We will use Cloud9 for this purpose. Spin up a Cloud9 instance by going to https://ap-southeast-1.console.aws.amazon.com/cloud9/home/product.

**IMPORTANT!** 
Cloud9 rotate credentials (secret/access key) and this is not supported by kubectl, because it detect/match the exact access key that represent IAM User that is used to initially create the cluster.
If you use Cloud9, then ensure you go to > Preferences -> AWS Settings -> Turn off "AWS managed temporary credentials"

![img1]

[img1]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/c9disableiam.png

In the IAM console, create a user with AdministratorAccess for the purpose of this lab. 

- Follow this link https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess to create an IAM role with Administrator access.
- Confirm that AWS service and EC2 are selected, then click Next to view permissions.
- Confirm that AdministratorAccess is checked, then click Next to review.
- Enter eksworkshop-admin for the Name, and select Create Role

![img2]

[img2]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/createrole.png

Next attach the IAM role to your workspace

- Follow this deep link https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=desc:launchTime to find your Cloud9 EC2 instance
- Select the instance, then choose Actions / Instance Settings / Attach/Replace IAM Role

![img3]

[img3]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab1-Getting-Started-with-EKS/img/c9instancerole.png

Next configure the region **ap-southeast-1** running aws configure. You can leave the key and secret as None.

```
aws configure

```

You will observe the following output. The "error" is a warning and can be ignored.
```
$ aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [ap-southeast-1]: 
Default output format [None]: 
usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]
To see help text, you can run:

  aws help
  aws <command> help
  aws <command> <subcommand> help
aws: error: too few arguments
```

Kubernetes uses a command-line utility called kubectl for communicating with the cluster API server. Amazon EKS clusters also require the AWS IAM Authenticator for Kubernetes to allow IAM authentication for your Kubernetes cluster. Beginning with Kubernetes version 1.10, you can configure the kubectl client to work with Amazon EKS by installing the AWS IAM Authenticator for Kubernetes and modifying your kubectl configuration file to use it for authentication. 

**Install kubectl**

   ```bash
   mkdir $HOME/bin
   curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   cp ./kubectl $HOME/bin/kubectl
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   kubectl version --client
   ```
   
**Install IAM Authenticator**

   ```bash
   curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/linux/amd64/aws-iam-authenticator
   chmod +x ./aws-iam-authenticator
   cp ./aws-iam-authenticator $HOME/bin
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   aws-iam-authenticator version
   ```

## 3. Create your Amazon EKS Control Plane and Data Plane

Download the latest release of eksctl 

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

```

Create a basic EKS cluster

```
export CLUSTER=<Your cluster name>
eksctl create cluster --name=$CLUSTER --region ap-southeast-1 --fargate

```

A cluster will be created with the following

  - uses fargate serverless compute
  - ap-southeast-1 region
  - dedicated VPC
  - public and private subnets in 3 AZs
  - fargate profile with selectors for all pods in the kube-system and default namespaces. 

Example output:

```
[ℹ]  eksctl version 0.19.0-rc.0
[ℹ]  using region ap-southeast-1
[ℹ]  setting availability zones to [ap-southeast-1b ap-southeast-1c ap-southeast-1a]
[ℹ]  subnets for ap-southeast-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for ap-southeast-1c - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for ap-southeast-1a - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.15
[ℹ]  creating EKS cluster "fargatecluster1" in "ap-southeast-1" region with Fargate profile
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-1 --cluster=fargatecluster1'
[ℹ]  CloudWatch logging will not be enabled for cluster "fargatecluster1" in "ap-southeast-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=ap-southeast-1 --cluster=fargatecluster1'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "fargatecluster1" in "ap-southeast-1"
[ℹ]  1 task: { create cluster control plane "fargatecluster1" }
[ℹ]  building cluster stack "eksctl-fargatecluster1-cluster"
[ℹ]  deploying stack "eksctl-fargatecluster1-cluster"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "fargatecluster1" have been created
[ℹ]  creating Fargate profile "fp-default" on EKS cluster "fargatecluster1"

[ℹ]  created Fargate profile "fp-default" on EKS cluster "fargatecluster1"
[ℹ]  "coredns" is now schedulable onto Fargate
[ℹ]  "coredns" is now scheduled onto Fargate
[ℹ]  "coredns" pods are now scheduled onto Fargate
[ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "fargatecluster1" in "ap-southeast-1" region is ready
```

Once you have created a cluster, you will find that cluster credentials were added in ~/.kube/config.

The script requires about 15 minutes to complete. Meanwhile you can go to lab 2 to create your App Mesh Artifacts.

## 4.Test the cluster

Test your configuration. 

```
kubectl get svc
```

You should see a kubernetes svc as an output.

```
kubectl get pods --ns kube-system 

```

You should be able to see the coredns pods running on fargate.

You have now completed setting up the eks cluster!


