## Hybrid EKS control plane using AWS Local Zones

A hybrid AWS EKS control plane using AWS Local Zones is a control plane that spans multiple geographic locations, including Local Zones and AWS Regions. With this configuration, you can deploy applications closer to end-users for low-latency performance while maintaining a centralized management plane for your EKS control plane.

To set up a hybrid AWS EKS control plane using AWS Local Zones, you need to create an Amazon EKS control plane in an AWS Region. You can then add self-managed nodes to the EKS cluster that are located in the Local Zones. These nodes can be used to run Kubernetes pods that are designed to serve requests from users in the Local Zone.

To ensure the pods are highly available, you can use Kubernetes' node affinity and anti-affinity features to ensure that pods are scheduled on nodes in different Local Zones. 

By setting up a hybrid AWS EKS control plane using AWS Local Zones, you can take advantage of the low-latency benefits of running your application in proximity to end-users while maintaining a centralized management plane for your EKS cluster.

When you are looking to understand how to design workloads that are stretched across AWS Region and Local Zones, this project presents a sample architecture. The project also shares complementary CloudFormation that you can consider to improve operational and developer efficiency.

## Solution Overview

Hybrid EKS Cluster sample solution enable you to simplify and centralize the management of your infrastructure and applications on AWS Region and on AWS Local Zones. You can extend the AWS cloud operations experience across hybrid and Local Zones for secure and seamless management, compliance, and observability. AWS Hybrid Cloud Solutions enable you to deliver a consistent AWS experience wherever you need it—from the cloud, to the edge.

To create a hybrid EKS cluster with a mix of managed and self-managed nodes, you can use CloudFormation samples to define and deploy the necessary infrastructure. Here's a high-level overview of the solution:

- Define the VPC and Subnets
- Deploy the EKS Cluster
- Add Self-Managed Nodes
- Deploy sample applications in the Region and Local Zones

The following diagram shows the high-level architecture, for running a sample website on Amazon EKS in the Local Zone and the Region.

You will be deploying to sample applications one in the Local Zones and another backup one in the Region

When you are connecting to the Local Zone, the request is served by the ALB in the Local Zone, and the game (sample application) is hosted by Kubernetes pods, running on the self-managed Amazon EC2 nodes. 

For the backup site in the Region, there is an ALB and Kubernetes pods running on a managed node group. The backup game is used for High Availability that makes it easy for IT administrators to set up, operate, and scale in the cloud.

![architecture](/assets/architecture.jpg)


## Hybrid EKS Cluster Deployment using Local Zones

For the application deployment, we use the combination of CloudFormation YAML files. We use CloudFormation to create AWS resources such as Amazon Virtual Private Cloud (Amazon VPC), Amazon EKS, Amazon EC2, etc. For the application in Kubernetes, we use the YAML manifest files and 2048 game available in this Solution.

### Prerequisites

- An AWS account with the Administrator permissions. 

- A shell environment. An IDE (Integrated Development Environment) environment such as Visual Studio Code or AWS Cloud9 is recommended. 

- Setup IAM user and role to assume

```bash
#create IAM user
aws iam create-user --user-name hybrid-eks-user

#create user policy
cat >user-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "*",
          "iam:ListRoles",
          "sts:AssumeRole"
        ],
        "Resource": "*"
      }
    ]
  }
EOF

#create policy
aws iam create-policy --policy-name hybrid-eks-policy --policy-document file://user-policy.json

#replace account id with your account
aws iam attach-user-policy --user-name hybrid-eks-user --policy-arn "arn:aws:iam::<account id>:policy/hybrid-eks-policy"

#load-balancer-role-trust-policy 
#make sure to change the policy's principal based on your account id
cat >eks-role-trust-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<account id>"
      },
      "Action": "sts:AssumeRole"
    }
}
EOF

#update the account id in the trust policy before creating the role
aws iam create-role --role-name hybrid-eks-user-role --assume-role-policy-document file://eks-role-trust-policy.json

aws iam attach-role-policy --role-name hybrid-eks-user-role --policy-arn "arn:aws:iam::<account id>:policy/hybrid-eks-policy"

aws sts assume-role --role-arn "arn:aws:iam::<account id>:role/hybrid-eks-user-role" --role-session-name currentsession

#replace export values from the assume role command results 
export AWS_ACCESS_KEY_ID=<AAAA>
export AWS_SECRET_ACCESS_KEY=<BBBB>
export AWS_SESSION_TOKEN=<CCCC>
```

- Opt-in the Local Zone that you would like to run your workload in.

- Installation of the latest version AWS Command Line Interface (AWS CLI) (v2 recommended), kubectl, eksctl, 


<!-- # Install NodeJS v16 and set as default
npm install -g npm
nvm install v16.17.0
nvm alias default 16.17.0

# Install TypeScript
npm install -g typescript -->

```bash

#Download and extract the latest release of awscli with the following
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
#python3 -m pip install aws-cli
#pip install awscli

# Download and extract the latest release of eksctl with the following
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install kubectl for the self-managed nodes at the Local Zones side.
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.23.13/2022-10-31/bin/linux/amd64/kubectl
openssl sha1 -sha256 kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$PATH:$HOME/bin' >~/.bash_profile
kubectl version --short --client

#create an EC2 key pair
aws ec2 create-key-pair --key-name ws-default-keypair --query 'KeyMaterial' --output text > MyKeyPair.pem
```
### Walkthrough

#### Step 1: Create the topology along EKS cluster in the Region

-  Initiate the CloudFormation stack using eks-cluster YAML file:

```bash
 aws cloudformation create-stack \
  --stack-name stack1 \
  --template-body file://eks-cluster.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameters ParameterKey=EKSClusterName,ParameterValue=eks-cluster ParameterKey=NumWorkerNodes,ParameterValue=2  ParameterKey=KeyPairName,ParameterValue=ws-default-keypair
```

 -  When you observe the `StackId` prompt appearing in your terminal, it indicates that the execution of your CloudFormation has been initiated successfully. This prompt serves as a confirmation that your CloudFormation stack has started to execute.

 ![stackcreated](/assets/stackId.jpeg)

- Wait for 10-15 mins for the CloudFormation to create all resources "CREATE_COMPLETE" status for the stack.

#### Step 2: Deploy and setup the self-managed nodes communication

- Prepare the environment for Kubernetes and self-managed nodes communication

    
```bash
#Ensure that kubectl is configured for the Region and the newly created cluster.
aws eks update-kubeconfig \
            --region us-west-2 \
            --name eks-cluster 
#Copy and paste the Amazon IAM Authenticator configuration map details to aws-auth-cm.yaml 
kubectl get -n kube-system configmap/aws-auth -o yaml  > aws-auth-cm.yaml
#Fetch required resources generated through the previous step
ClusterControlPlaneSecurityGroup=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`EKSClusterSG`].OutputValue' --output text)
 KeyName=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`KeyPair`].OutputValue' --output text)
 Subnets=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`LZPrivateSubnetId`].OutputValue' --output text)
 VpcId=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
```

- Create the self-managed nodes

```bash
 aws cloudformation create-stack \
  --stack-name stack2 \
  --template-body file://eks-selfmanaged-node.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameters ParameterKey=ClusterName,ParameterValue=eks-cluster ParameterKey=ClusterControlPlaneSecurityGroup,ParameterValue=$ClusterControlPlaneSecurityGroup  ParameterKey=KeyName,ParameterValue=$KeyName ParameterKey=Subnets,ParameterValue=$Subnets ParameterKey=VpcId,ParameterValue=$VpcId ParameterKey=NodeGroupName,ParameterValue=eks-selfmanaged-groupnode ParameterKey=NodeImageId,ParameterValue=ami-0b149b4c68ab69dce
```

> When you observe the `StackId` prompt appearing in your terminal, it indicates that the execution of your CloudFormation has been initiated successfully. This prompt serves as a confirmation that your CloudFormation stack has started to execute.

- Wait for 5 mins for the CloudFormation to create all resources "CREATE_COMPLETE" status for the stack.

- Enable self-managed nodes to join your cluster

1. Get the self-manged nodes role.

```bash
 echo $(aws cloudformation describe-stacks --stack-name "stack2" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`NodeInstanceRole`].OutputValue' --output text)
```

 ![role](/assets/role.jpeg)

2. Open the aws-auth-cm.yaml file in the left panel of your Cloud9 IDE.
3. Copy the existing `groups` node and paste it as an additional tag.
4. Set the `rolearn` of the newly added group to the value that you recorded in the previous procedure. Be sure to change the role ARN and **avoid any alignment** issues.

![aws-auth-file](/assets/authyaml.jpeg)

5. Verify that both groups have the same **alignment** and **save** the file before proceeding to the next step.
6. Apply the configuration using the appropriate command. This process may take a few minutes to complete.

```bash
 kubectl apply -f aws-auth-cm.yaml
```

 ![auth](/assets/join-self-managed.jpeg) 

7. Watch the status of your nodes and wait for them to reach the Ready status.

```bash
 kubectl get nodes --watch
```

> To stop monitoring the status of nodes in a Kubernetes cluster using the "watch" command, you can press "Ctrl + C" when you observe that new node is in the "Ready" state. This will terminate the command and exit the watch mode.

![auth2](/static/joined-self-managed.jpeg) 

> [!WARNING]
> If you receive any authorization or resource type errors, see [Unauthorized or access denied (kubectl)](https://docs.amazonaws.cn/en_us/eks/latest/userguide/troubleshooting.html#unauthorized) and [further references](https://docs.amazonaws.cn/en_us/eks/latest/userguide/eks-outposts-self-managed-nodes.html) in the troubleshooting topic.

#### Step 3: Installing the AWS Load Balancer Controller add-on

 1. Enable the communication between both managed nodes in the Region and self-managed nodes in the Local Zone.

```bash
ManagedWorkerNodesSecurityGroup=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`ManagedWorkerNodesSecurityGroup`].OutputValue' --output text)
SelfManagedWorkerNodesSecurityGroup=$(aws cloudformation describe-stacks --stack-name "stack2" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`NodeSecurityGroup`].OutputValue' --output text)

aws ec2 authorize-security-group-ingress \
    --group-id $ManagedWorkerNodesSecurityGroup \
    --protocol -1 \
    --port -1 \
    --source-group $SelfManagedWorkerNodesSecurityGroup
aws ec2 authorize-security-group-ingress \
    --group-id $SelfManagedWorkerNodesSecurityGroup \
    --protocol -1 \
    --port -1 \
    --source-group $ManagedWorkerNodesSecurityGroup
```

 2. Creating an AWS Identity and Access Management (IAM) OpenID Connect (OIDC) provider for your cluster.

   Amazon EKS supports using OpenID Connect (OIDC) identity providers as a method to authenticate users to your cluster, further [details](https://docs.aws.amazon.com/eks/latest/userguide/authenticate-oidc-identity-provider.html)

   - Retrieve your cluster's OIDC provider ID and store it in a variable.

```bash
 oidc_id=$(aws eks describe-cluster --name eks-cluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
 account_id=$(aws sts get-caller-identity --query Account --output text)
```

   - Create an IAM OIDC identity provider for your cluster with the following command.

```bash
 eksctl utils associate-iam-oidc-provider --cluster eks-cluster --approve
```

 3. Deploy the AWS Load Balancer Controller to an Amazon EKS cluster

   - Create an IAM policy using the policy downloaded in the previous step.

```bash
 curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
 aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
 
 - Create an IAM role. Create a Kubernetes service account named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller and annotate the Kubernetes service account with the name of the IAM role.

 ```bash
 eksctl create iamserviceaccount \
  --cluster=eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$account_id:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
 ```
   - Install the AWS Load Balancer Controller by applying a Kubernetes manifest

```bash

 # Install cert-manager using one of the following methods to inject certificate configuration into the webhooks.
 kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml

 # Download the controller specification. 
curl -Lo v2_4_7_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.7/v2_4_7_full.yaml

 #Make the following edits to the file.
sed -i.bak -e '561,569d' ./v2_4_7_full.yaml
sed -i.bak -e 's|your-cluster-name|eks-cluster|' ./v2_4_7_full.yaml

 # Download the IngressClass and IngressClassParams manifest to your cluster.
curl -Lo v2_4_7_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.7/v2_4_7_ingclass.yaml
```

   * Retrieve the `VpcId` created in previous steps.

```bash
 VpcId=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
 echo $VpcId
```

   * Open the v2_4_7_full.yaml file in the left panel of your Cloud9 IDE.
   * Search for `args`.
   * Add `aws-vpc-id` & `aws-region` to the containers' args.

>[!WARNING]
>Replace `vpc-xxxxxxxx` with VpcId echo before updating the file with containers' args.


```bash
- --aws-vpc-id=vpc-xxxxxxxx
- --aws-region=us-west-2
```

  * Verify that the added nodes have the same **alignment** and **save** the file before proceeding to the next step.

      ![v2_4_4_full_file](/assets/containersargs.jpeg)

   - Apply the file and the manifest to your cluster.

```bash
 kubectl apply -f v2_4_7_full.yaml
 kubectl apply -f v2_4_7_ingclass.yaml
```

   - Verify that the controller is installed.

```bash
 kubectl get deployment -n kube-system aws-load-balancer-controller
 kubectl get pod -n kube-system
```

 The example output is as follows.
  
```bash 
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1          84s
```

![cnt](/assets/controller.jpeg) 

### Step 2: Deploy two sample application to the Region and to the Local Zones


 1.  Deploy the game [2048](https://play2048.co/) as a sample application to verify that the AWS Load Balancer Controller creates an AWS ALB as a result of the ingress object or use the sample games configured along the nodeAffinity in this solution.


 2. Get the public subnet in the Local Zones
      
```bash
 #deploying to pods in a cluster in the Region (public)
 echo $(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`LZPublicSubnetId`].OutputValue' --output text)
```

 3. Open 2048_lz file and update tag **"alb.ingress.kubernetes.io/subnets"** with subnet captured from previous step, **save** file, then deploy the application using the following command
      
```bash
#Deploying to pods in a cluster in the Region (public)
 kubectl apply -f 2048_backup.yaml
#Deploying to pods in a cluster in the Local Zones (public)
 kubectl apply -f 2048_lz.yaml
```

 4.  After a few minutes, verify that the ingress resource was created with the following command.

```bash
 kubectl get ingress/ingress-2048-backup -n game-2048-backup
 kubectl get ingress/ingress-2048-lz -n game-2048-lz
```

 The example output is as follows.

>```bash
>NAME           CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
>ingress-2048   <none>   *       k8s-game2048-ingress2-xxxxxxxxxx-yyyyyyyyyy.region-code.elb.amazonaws.com   80      2m32s
>```


 5. To verify successful installation, open a browser and navigate to the ADDRESS URL from the previous commands output to see the sample application. If you don't see anything, refresh your browser and try again or [troubleshoot](https://repost.aws/knowledge-center/eks-load-balancer-webidentityerr) . 

 ![2048](/assets/2048.png)

>[!NOTE]
>-   Kubernetes assigns the service its own IP address that is accessible only from within the cluster. To access the service from outside of your cluster, deploy the AWS Load Balancer Controller to load balance [application](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html) or network traffic to the service. 
>
>- Enabling IAM user and role access to your [cluster](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
>
>- Installing the AWS Load Balancer Controller add-on further [details](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

