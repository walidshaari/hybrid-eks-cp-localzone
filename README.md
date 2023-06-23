## Hybrid EKS control plane using AWS Local Zones

A hybrid Amazon EKS cluster using AWS Local Zones is a control plane that spans multiple geographic locations, including Local Zones and AWS Regions. With this configuration, you can deploy applications closer to end-users for low-latency performance while maintaining a centralized management plane for your EKS cluster.

To set up a hybrid Amazon EKS cluster using AWS Local Zones, you need to create an Amazon EKS control plane in an AWS Region. You can then add self-managed nodes to the EKS cluster that are located in the Local Zones. These nodes can be used to run Kubernetes pods that are designed to serve requests from users in the Local Zone.

To ensure the pods are highly available, you can use Kubernetes' node affinity and anti-affinity features to ensure that pods are scheduled on nodes in different Local Zones. 

By setting up a hybrid Amazon EKS control plane using AWS Local Zones, you can take advantage of the low-latency benefits of running your application in proximity to end-users while maintaining a centralized management plane for your EKS cluster.

When you are looking to understand how to design workloads that are stretched across AWS Region and Local Zones, this project presents a sample architecture. The project also shares complementary CloudFormation that you can consider to improve operational and developer efficiency.

<br>

> :warning: **Disclaimer:** The sample code; software libraries; command line tools; proofs of concept; templates; or other related technology (including any of the foregoing that are provided by our personnel) is provided to you as AWS Content under the AWS Customer Agreement, or the relevant written agreement between you and AWS (whichever applies). You should not use this AWS Content in your production accounts, or on production or other critical data. You are responsible for testing, securing, and optimizing the AWS Content, such as sample code, as appropriate for production grade use based on your specific quality control practices and standards. Deploying AWS Content may incur AWS charges for creating or using AWS chargeable resources, such as running Amazon EC2 instances or using Amazon S3 storage.

<br> 

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

- An AWS account, IAM user and role with Administrator permissions. 

- A shell environment. An IDE (Integrated Development Environment) environment such as Visual Studio Code or AWS Cloud9 is recommended. 

![sts](/assets/sts.jpeg)

- Installation of the latest version AWS Command Line Interface (AWS CLI) (v2 recommended), kubectl, eksctl, 

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
- Assume role, using AWS Identity and Access Management (AWS IAM) Role,  For setup details, please refer to the docs ![here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html).

- Opt-in the Local Zone that you would like to run your workload in `us-west-2-lax-1a`.

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

>**Note**
>  When you observe the `StackId` prompt appearing in your terminal, it indicates that the execution of your CloudFormation has been initiated successfully. This prompt serves as a confirmation that your CloudFormation stack has started to execute.



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

>**Note**
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

![auth2](/assets/joined-self-managed.jpeg) 

>**Warning**
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


> **Warning** Replace `vpc-xxxxxxxx` with VpcId echo before updating the file with containers' args.


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

#### Step 4: Deploy two sample application to the Region and to the Local Zones


 1.  Deploy the game [2048](https://play2048.co/) as a sample application to verify that the AWS Load Balancer Controller creates an AWS ALB as a result of the ingress object or use the sample games configured along the nodeAffinity in this solution.


 2. Get the public subnet in the Local Zones
      
```bash
 #deploying to pods in a cluster in the Region (public)
 echo $(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`LZPublicSubnetId`].OutputValue' --output text)
```

 3. Open 2048_lz file and update tag **"alb.ingress.kubernetes.io/subnets"** with subnet captured from previous step, **save** file, then deploy the applications using the following command
      
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

>**Note**
>-   Kubernetes assigns the service its own IP address that is accessible only from within the cluster. To access the service from outside of your cluster, deploy the AWS Load Balancer Controller to load balance [application](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html) or network traffic to the service. 
>
>- Enabling IAM user and role access to your [cluster](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
>
>- Installing the AWS Load Balancer Controller add-on further [details](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

## Troubleshooting

If you encounter a situation where you are unable to obtain an address for a test games, it may be helpful to troubleshoot the AWS Load Balancer Controller. One possible solution is to delete the controller using the full pod name. To do this, you can run the following command: 

>```bash
>kubectl get pods -o wide -n kube-system
>kubectl delete pods <aws-loadbalancer-controller-pod> -n kube-system
>```

Replace <aws-loadbalancer-controller-pod> with the full name of the AWS Load Balancer Controller pod.

## Clean up

To terminate the resources that we created in this sample, as the following:

- Detach polices from created roles **stack2-NodeInstanceRole**-XXXXXX, **stack1-WorkerNodesRole**-YYYYYY and **stack1-ControlPlaneRole**-ZZZZZZ

- Run the following
```bash

kubectl delete -f 2048_backup.yaml
kubectl delete -f 2048_lz.yaml
kubectl delete -f v2_4_7_ingclass.yaml

# manually detach polices from created roles stack2-NodeInstanceRole-<XXXXX> and stack1-ControlPlaneRole-<XXXYYY>

ManagedWorkerNodesSecurityGroup=$(aws cloudformation describe-stacks --stack-name "stack1" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`ManagedWorkerNodesSecurityGroup`].OutputValue' --output text)
SelfManagedWorkerNodesSecurityGroup=$(aws cloudformation describe-stacks --stack-name "stack2" --region "us-west-2" --query 'Stacks[0].Outputs[?OutputKey==`NodeSecurityGroup`].OutputValue' --output text)

aws ec2 revoke-security-group-ingress \
    --group-id $ManagedWorkerNodesSecurityGroup \
    --protocol -1 \
    --port -1 \
    --source-group $SelfManagedWorkerNodesSecurityGroup
aws ec2 revoke-security-group-ingress \
    --group-id $SelfManagedWorkerNodesSecurityGroup \
    --protocol -1 \
    --port -1 \
    --source-group $ManagedWorkerNodesSecurityGroup

aws cloudformation delete-stack --stack-name stack2
aws cloudformation delete-stack --stack-name stack1

```

Then, go to the Cloudformation console and make sure the stacks were deleted. Lastly, delete created user, policy and role assumed.

## Conclusion 

Setting up a hybrid Amazon EKS cluster using AWS Local Zones offers many benefits for organizations looking to improve the performance, availability, and resiliency of their containerized applications. With this setup, you can leverage the low-latency access to compute and storage resources in geographically closer locations to your end-users or data sources.

In this solution, we have covered the essential steps to create a hybrid EKS cluster, including setting up the EKS cluster, launching self-managed node in the Local Zone, joining the node in the Local Zone to the primary cluster, deploying a sample application.

By following these steps, you built a hybrid EKS cluster that is highly available, fault-tolerant, and scalable while maintaining a centralized control plane. With this infrastructure, you can easily deploy and manage containerized applications in multiple locations, improving the user experience and reducing the risk of downtime.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

