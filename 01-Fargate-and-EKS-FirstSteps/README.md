# AWS Fargate & EKS First Steps

## Step-01: Create EKS Cluster IAM Role
- This role allows EKS to manage clusters on our behalf.
- In simple terms, when a load balancer is created, Kubernetes (EKS) assumes the role to create an Elastic Load Balancing load balancer in our account. 
- Go to Services --> IAM --> Roles
- **Create Role**
    - Select type of trusted entity: AWS Service
    - Service: EKS
        - Select your Usecase: EKS (Allows EKS to manage clusters on your behalf.)
    - Click Next:Permissions, Next:Tags, Next:Review
    - Review
        - Role Name: eksClusterRole         
        - Role Description: Allows EKS to manage clusters on your behalf.
        - Click on **Create Role**

## Step-02: Create EKS Cluster VPC - 3 Public Subnets
- Create a VPC CloudFormation stack using below URL
```
https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-03-23/amazon-eks-vpc-sample.yaml
```
- **Go to Services --> CloudFormation --> Create Stack**
- **Create stack**
    - Prepare template: template is ready
    - Template source: Amazon S3 URL
    - Amazon S3 URL: https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-03-23/amazon-eks-vpc-sample.yaml
    - Click Next
- **Specify stack details**
    - Stack Name: eks-vpc-public
    - Parameters: 
        - VpcBlock: 192.168.0.0/16
        - Subnet01Block: 192.168.64.0/18
        - Subnet02Block: 192.168.128.0/18
        - Subnet03Block: 192.168.192.0/18
- **Configure stack options**
    - Tags
        - Key: Name
        - Value: eks-vpc-public
    - Rest all leave to defaults        
- **Review**
    - Click on **Create Stack**

## Step-03: Install and configure AWS CLI
- Reference-1: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
- Reference-2: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

### Step-03-01: Windows 10 - Install and configure AWS CLI
- The AWS CLI version 2 is supported on Windows XP or later.
- The AWS CLI version 2 supports only 64-bit versions of Windows.
- Download Binary: https://awscli.amazonaws.com/AWSCLIV2.msi
- Install the downloaded binary (standard windows install)
```
aws --version
aws-cli/2.0.8 Python/3.7.5 Windows/10 botocore/2.0.0dev12
```
- Reference: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html
### Step-03-02: MAC - Install and configure AWS CLI
- Pending
- Reference: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html

### Step-03-03: Configure AWS Command Line using Security Credentials
- Go to AWS Management Console --> Services --> IAM
- Select the IAM User: kalyan 
- **Important Note:** Use only IAM user to generate **Security Credentials**. Never ever use Root User. Highly not recommended)
- Click on **Security credentials** tab
- Click on **Create access key**
- Copy Access ID and Secret access key
- Go to command line and provide the required details
```
aws configure
AWS Access Key ID [None]: ABCDEFGHIKKLAKJKUNGG  (Replace your creds when prompted)
AWS Secret Access Key [None]: uMe7fumK1IdDB094q2sGFhM5Bqt3HQRw3IHZzBDTm  (Replace your creds when prompted)
Default region name [None]: us-east-1
Default output format [None]: json
```
- Test if AWS CLI is working after configuring the above
```
aws ec2 describe-vpcs
```

## Step-04: Install and configure kubectl 
- Kubectl binaries for EKS please prefer to use from Amazon (Amazon EKS-vended kubectl binary)
- This will help us to get the exact Kubectl client version based on our EKS Cluster version. You can use the below documentation link to download the binary.
- Reference: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

### Step-04-01: Windows 10 - Install and configure kubectl
- Install kubectl on Windows 10 
```
mkdir kubectlbinary
cd kubectlbinary
curl -o kubectl.exe https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/windows/amd64/kubectl.exe
```
- Update the system **Path** environment variable 
```
C:\Users\KALYAN\Documents\kubectlbinary
```
- Verify the kubectl client version
```
kubectl version --short --client
kubectl version --client
```
### Step-04-02: MAC - Install and configure kubectl
- Pending

## Step-05: Create EKS Cluster
- Go to Services --> Elastic Kubernetes Service
- Click on **Clusters** --> **Create Cluster**
- **General Configuration:**
    - Cluster name: my-first-eks-cluster
    - Kubernetes version: 1.15 (select default which is latest on that day)
    - Role name: eksClusterRole
- **Networking:**
    - VPC: eks-vpc-public
    - Subnets: Auto selected when VPC selected
    - Security Groups: Select Control Plane Security  Group (Example: eks-vpc-public-ControlPlaneSecurityGroup-O8KQY7OH6MB)
    - Private Access: Enabled
    - Public Access: Enabled (leave to defaults)
    - Advance Setting: leave to defaults (Allow access for API Server Endpoint from 0.0.0.0/0)
- Secrets Encryption: Disabled (leave to defaults)        
- Logging: Enable all logs
- Tags: leave to defaults (No Tags)
- Click on **Create**
- **Important Note:** 
    - Cluster provisioning usually takes between 15 to 20 minutes.
    - Wait till the cluster status shows **ACTIVE**

## Step-06: Create a kubeconfig file
- Ensure AWS CLI is 1.18.17 or later
- By default, **config** file created in home directory (homedirectory/.kube/config) 
- We can specify another path with the **--kubeconfig** option.
```
aws eks --region region-code update-kubeconfig --name cluster_name
aws eks --region us-east-1 update-kubeconfig --name my-first-eks-cluster
kubectl get svc
```
## Step-07: Create EKS Worker node IAM role
- The Amazon EKS worker node kubelet daemon (from EC2 Instances) makes calls to AWS APIs on your behalf. 
- **Important Note:** It is highly recommended that we create a new worker node IAM role for each EKS cluster.
- Go to Services --> IAM --> Roles
- **Create Role**
    - Select type of trusted entity: AWS Service
    - Service: EC2
    - Click Next:Permissions
        - Search and check **AmazonEKSWorkerNodePolicy**
        - Search and check **AmazonEKS_CNI_Policy**
        - Search and check **AmazonEC2ContainerRegistryReadOnly**
    - Click Next:Tags, Next:Review
    - Review
        - Role Name: my-first-eks-cluster-worker-node-role   (Naming Format for convenience: clustername-rolename)     
        - Role Description: Allows EC2 instances to call AWS services on your behalf..
        - Click on **Create Role**
- **Very Very Important Note:**  **Refresh the browser** to load the new roles created to list when creating Node Groups

## Step-08: Launch Managed Node Group (Kubernetes Worker Nodes)
- Go to Services --> Elastic Kubernetes Service
- Click on **my-first-eks-cluster**
- Click on **Add Node Group**
- **Configure Node Group**
    - Name: DevNodeGroup
    - Node IAM Role Name: my-first-eks-cluster-worker-node-role 
    - Subnets: all subnets selected by default (leave to defaults)
    - Allow Remote Access to Nodes: enabled (leave to defaults)
    - SSH key pair: my-first-key-pair
    - Allow remote access from: all (leave to defaults)
- **Set compute configuration**
    - AMI type: Amazon Linux 2 (AL2_x86_64) (leave to defaults)
    - Instance type: t3.medium (leave to defaults)
    - Disk Size: 20GB  (leave to defaults)  
- **Set scaling configuration**
    - Minimum size: 2 (leave to defaults)  
    - Maximum size: 2 (leave to defaults)  
    - Desired size: 2 (leave to defaults)  
- **Review and create**
    - Click on **Create**        