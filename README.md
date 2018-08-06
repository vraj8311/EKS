AWS Elastic Kubernetes Service (EKS) QuickStart  
===============================================

This solution shows how to create an AWS EKS Cluster and deploy a simple web application with an external Load Balancer. This readme updates an article "Getting Started with Amazon EKS" referenced below and provides a more basic step by step process.

Unfortunately this is a pretty manual effort right now.

Steps:  
  Create your Amazon EKS Service Role  
  Add Amazon EC2 Full Access to Service Role  
  Create your Amazon EKS Cluster VPC  
  Create your Amazon EKS Cluster  
  Configure Your AWS EC2 Instance  
  Configure kubectl on Your EC2 Instance  
  Launch and Configure Your Amazon EKS Worker Nodes  
  Enable Worker Nodes to Join Your Cluster  
  Deploy WebApp to Your Cluster  
  Remove Your AWS EKS Cluster  


To make this first cluster easy to deploy we'll use a docker image located in DockerHub at kskalvar/web.  This image is nothing more than a simple webapp that returns the current ip address of the container it's running in.  We'll create an external AWS Load Balancer and you should see a unique ip address as it is load balanced across containers.

The project also includes the Dockerfile for those interested in the configuration of the actual application or to build your own and deploy using ECR.


## Create your Amazon EKS Service Role

Use the AWS Console to configure the EKS IAM Role.  This is a step by step process.

### AWS IAM Dashboard
Select Roles  

Click on "Create role"  
Select "AWS Service"  
Choose the service that will use this role  
```
EKS
```  
Click on "Next: Permissions"  
Click on "Next: Review"  
Enter "Role Name"
```
eks-role
```
Click on "Create role"

## Add Amazon EC2 Full Access to Service Role

Use the AWS Console to add EC2 Full Access so you can provision EC2 Load Balancers from kubectl

### AWS IAM Dashboard
Select Roles  

Click on "eks-role"  
Click on "Attach policies"  
Enter "EC2FullAccess" in "Filter policies"  
Select "AmazonEC2FullAccess"  
Click on "Attach Policy"  


## Create your Amazon EKS Cluster VPC

Use the AWS CloudFormation to configure the Cluster VPC.  This is a step by step process.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-vpc-sample.yaml
```
Click on "Next"  
```
Stack Name: eks-vpc
```
Click on "Next"  
Click on "Next"  
Click on "Create"

Wait for Status CREATE_COMPLETE before proceeding 


## Create your Amazon EKS Cluster

Use the AWS Console to configure the EKS Cluster.  This is a step by step process.

### AWS EKS Dashboard
Click on "Clusters"  

Click on "Create cluster"  
```
Cluster name: eks-cluster
Kubernetes version:  1.10
Role ARN: eks-role
VPC: eks-vpc-VPC
Subnets:  Should preselect all available
Security groups: eks-vpc-ControlPlaneSecurityGroup-*
```

Click on "Create"  

Wait for Status ACTIVE before proceeding

## Configure Your AWS EC2 Instance

Use AWS Console to configure the EC2 Instance for kubectl.  This is a step by step process.

### AWS EC2 Dashboard  

Click on "Launch Instance"

Click on "Community AMIs"  

Search community AMIs
```
ami-14c5486b
```  
Click on "Select"

Choose Instance Type
```
t2.micro
```
Click on "Next: Configure Instance Details"

Configure Instance  
Click on "Advanced Details
```
User data
Select "As file"
Click on "Choose File" and Select "cloud-init" from project cloud-deployment directory 
```  
Click on "Next: Add Storage"

Click on "Next" Add Tags"  

Add Tags
```
Name kubectl-console
```

Click on "Next: Configure Security Group"  

Configure Security Group  
Select "Select an existing security group"  
Select "default"

Click on "Review and Launch"    
Click on "Launch"


## Configure kubectl on Your EC2 Instance

You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.

### Check to insure cloud-init has completed

See contents of "/tmp/install-eks-support" it should say "installation complete".


### Configure awscli

aws configure
```
AWS Access Key ID []: <Your Access Key ID>
AWS Secret Access Key []: <Your Secret Access Key>
Default region name [None]: us-east-1
```

### Create kubectl configuration file

Gather cluster name, endpoint, and certificate for use below
```
aws eks list-clusters                                                               
aws eks describe-cluster --name eks-cluster --query cluster.endpoint                
aws eks describe-cluster --name eks-cluster  --query cluster.certificateAuthority.data  
```

Get the github project so you'll have configuration templates available
```
git clone https://github.com/kskalvar/aws-eks-cluster-quickstart.git
```

Create kubeconfig replacing <cluster-name> <endpoint-url> <base64-encoded-ca-cert> with information above
```
mkdir -p ~/.kube  
cp ~/aws-eks-cluster-quickstart/kube-config/control-kubeconfig ~/.kube  
cd ~/.kube  
edit control-kubeconfig

<endpoint-url>
<base64-encoded-ca-cert>
<cluster-name>
  
```

### Test Cluster
```
export KUBECONFIG=~/.kube/control-kubeconfig 
kubectl get svc 
```

## Launch and Configure Your Amazon EKS Worker Nodes

Use AWS CloudFormation to configure the Worker Nodes.  This is a step by step process.

### AWS CloudFormation Console
Click on "Create Stack"  
Select "Specify an Amazon S3 template URL"  
```
https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-nodegroup.yaml
```
Click on "Next"  
```
Stack Name: eks-nodegroup
ClusterNamme: eks-cluster
ClusterControlPlaneSecurityGroup: eks-vpc-ControlPlaneSecurityGroup-*
NodeGroupName: eks-nodegroup
NodeImageId: ami-dea4d5a1
KeyName: <Your AWS KeyName>
VpcId: eks-vpc-VPC
Subnets: Subnet01, Subnet02, Subnet03
```
Click on "Next"  
Click on "Next"  
Select Check Box "I acknowledge that AWS CloudFormation might create IAM resources"  
Click on "Create"

Wait for Status CREATE_COMPLETE before proceeding  
You should be able to see the additional nodes visible in AWS EC2 Console

## Enable Worker Nodes to Join Your Cluster

You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.

###  Get the aws-auth-cm.yaml template from the github project
```
cp ~/aws-eks-cluster-quickstart/kube-config/aws-auth-cm.yaml aws-auth-cm.yaml
```

### Edit aws-auth-cm.yaml
```
Replace "<ARN of instance role (not instance profile)>" with NodeInstanceRole from output of CloudFormation script "eks-nodegroup"
```

### Test Cluster Nodes
```
kubectl apply -f aws-auth-cm.yaml
kubectl get nodes
```
Wait till you see all nodes appear in "STATUS Ready"


## Deploy WebApp to Your Cluster

You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.

### Create Pod
```
export KUBECONFIG=~/.kube/control-kubeconfig 
kubectl run web --image=kskalvar/web --port=5000
```

### Scale Pod
```
kubectl scale deployment web --replicas=3
```

### Show Pods Running
```
kubectl get pods --output wide
```
Wait till you see all pods appear in "STATUS Running"

### Create Load Balancer
```
kubectl expose deployment web --port=80 --target-port=5000 --type="LoadBalancer"
```

### Get AWS External Load Balancer Address
Capture EXTERNAL-IP for use below
```
kubectl get service web --output wide
```
Wait till you see "EXTERNAL-IP ```*.<your account>.<region>.elb.amazon.com```" 

### Test from browser
```
Using your client-side browser goto: http://<EXTERNAL-IP>
```

### Delete Deployment, Service
```
kubectl delete deployment,service web
```

## Remove Your AWS EKS Cluster

### AWS CloudFormation
Delete "eks-nodegroup" Stack  
Wait for "eks-nodegroup" to be deleted before proceeding

### AWS EKS
Delete "eks-cluster"  
Wait for cluster to be deleted before proceeding, pretty slow?  Really slow!  

### AWS CloudFormation
Delete "eks-vpc"  
Wait for "eks-vpc" to be deleted before proceeding

### AWS EC2
Delete "kubectl-console" Instance

### AWS IAM Roles
Delete the following Roles:

```
AWSServiceRoleForAutoScaling  
AWSServiceRoleForElasticLoadBalancing  
eks-role  
```
## References
Getting Started with Amazon EKS  
https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html
