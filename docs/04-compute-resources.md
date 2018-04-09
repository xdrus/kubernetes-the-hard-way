# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across multiple [Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-regions-availability-zones) (AZ).

## Supplemental resources

We will use a set of supplement resources in order to simplify cluster management, such as SSM Parameter store and EC2 instance profiles with IAM roles.

### SSM Parameter Store key
To store sensitive data, such us master's private keys we will use [SSM parameter store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) with values encrypted by default SSM KMS key. This is done for simplicity, in production environment you would rather create a custom key with hardened IAM policy. We will need a KMS key ARN to grant access for instances to it, the default one can be get with the command below:

```bash
export CMK=$(aws kms describe-key --key-id alias/aws/ssm --query KeyMetadata.Arn --output text)
```

### IAM roles

In this section we will create EC2 [Instance profiles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) with appropriate IAM Roles for worker nodes and master nodes in order to make them work with CNI plugin as well as to use with heptio authenticator.

#### Role for CNI
Both master and workers nodes will require access to Elastic Network Interface - [ENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) in order to work with [VPC CNI plugin](https://github.com/aws/amazon-vpc-cni-k8s). The required policy is below:

```
{
    "Effect": "Allow",
    "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:AttachNetworkInterface",
        "ec2:DeleteNetworkInterface",
        "ec2:DetachNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeInstances",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:AssignPrivateIpAddresses"
    ],
    "Resource": [
        "*"
    ]
},
{
    "Effect": "Allow",
    "Action": "tag:TagResources",
    "Resource": "*"
}
```

### ECR Access

In order to have access to ECR repositories we can use AWS managed policy [AmazonEC2ContainerRegistryReadOnly](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr_managed_policies.html#AmazonEC2ContainerRegistryReadOnly)

### Create policies
As usually we use CloudFormation to create those resources for us. Please refer to IAM documentation if you want to create those resources manually.

```
export CFN_RESOURCES=$ENV-resources

aws cloudformation create-stack --stack-name $CFN_RESOURCES --template-body file://templates/resources.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=$ENV ParameterKey=CMK,ParameterValue=$CMK --capabilities CAPABILITY_NAMED_IAM
```

Now get ARN of instance profiles to use later for instances.

```
export NODE_PROFILE=$(aws cloudformation describe-stacks --stack-name $CFN_RESOURCES --query 'Stacks[0].Outputs[?OutputKey==`NodeInstanceProfile`].OutputValue' --output text)

export MASTER_PROFILE=$(aws cloudformation describe-stacks --stack-name $CFN_RESOURCES --query 'Stacks[0].Outputs[?OutputKey==`MasterInstanceProfile`].OutputValue' --output text)
```


## Compute Instances

The compute instances in this lab will be provisioned using [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2) minimal, which provides packets that enable easy integration with AWS and tuned kernel to work on EC2. We will use [AutoScalingGroup](https://aws.amazon.com/ec2/autoscaling) to ensure that all instances are healthy and replaced in case of failure.

### SSH KeyPair

First of all we need to create an [SSH KeyPair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) so we can log into our instances for maintenance and troubleshooting. You can either [create a new keypair](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-key-pair.html) or [import an existing one](https://docs.aws.amazon.com/cli/latest/reference/ec2/import-key-pair.html).

__TODO__: show how to create a new keypair.

Once you have done this set environment variable to your keypair-name.

```bash
export SSH_KEY=hardway
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane. There are few prerequisites items we need to install on instances including docker and Kubernets binaries. Also we need to install AWS CLI in order to retrieve data from SSM later. To install docker and AWS CLI we can just use standard repositories available in Amazon Linux 2:

```bash
yum install -y awscli
yum install -y docker
```

However, Kubernetes binaries are not available there, so we will get them manually. The [current way of getting Kubernetes binaries](https://kubernetes.io/docs/getting-started-guides/scratch/#downloading-and-extracting-kubernetes-binaries) is not very suitable for automation, so we will hardcode URLs in our scripts. We need kubectl, kubelet and kube-proxy only, as [other components will be run as pods](https://kubernetes.io/docs/getting-started-guides/scratch/#bootstrapping-the-cluster)

```bash
PLATFORM=amd64
KUBE_RELEASE=v1.9.6
KUBE_URL=https://storage.googleapis.com/kubernetes-release/release

DIR=/usr/local/bin/
FILES="kubelet kubectl kube-proxy"

for FILE in $FILES; do
  curl -sS --retry 3 "$KUBE_URL/$KUBE_RELEASE/bin/linux/$PLATFORM/$FILE" -o $DIR/$FILE
  chmod +x $DIR/$FILE
done
```

The above commands can be incorporated in CloudFormation template and used in Userdata section of Launch Configuration. The result template is [../templates/nodes_base.yaml]:


```bash
export CFN_CONTROL=$ENV-control-nodes

aws cloudformation create-stack --stack-name $CFN_CONTROL --template-body file://templates/nodes_base.yaml --parameters \
      ParameterKey=EnvironmentName,ParameterValue=$ENV \
      ParameterKey=VPC,ParameterValue=$VPC \
      ParameterKey=Subnets,ParameterValue=\"$CONTROL_SUBNETS\" \
      ParameterKey=SecurityGroup,ParameterValue=$CONTROL_SG \
      ParameterKey=InstanceProfile,ParameterValue=$MASTER_PROFILE \
      ParameterKey=KeyName,ParameterValue=$SSH_KEY
```

### Kubernetes Workers

The worker nodes requires the same software we have installed on the master nodes so far, so at this moment we are going to reuse the same template as of now to provision worker nodes, just change parameters to match worker nodes configuration:

```bash
export CFN_WORKERS=$ENV-worker-nodes

aws cloudformation create-stack --stack-name $CFN_WORKERS --template-body file://templates/nodes_base.yaml --parameters \
      ParameterKey=EnvironmentName,ParameterValue=$ENV \
      ParameterKey=VPC,ParameterValue=$VPC \
      ParameterKey=Subnets,ParameterValue=\"$CONTROL_SUBNETS\" \
      ParameterKey=SecurityGroup,ParameterValue=$CONTROL_SG \
      ParameterKey=InstanceProfile,ParameterValue=$MASTER_PROFILE \
      ParameterKey=KeyName,ParameterValue=$SSH_KEY
```

### Verification

List the compute instances in your default region:

```bash
aws ec2 describe-instances --filters Name=tag:Environment,Values=$ENV  --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress,PrivateIpAddress,Tags[?Key==`Name`].Value]'
```

> Outputs
```json
[
    [
        "i-01cdc9d559d0426a5",
        "13.229.152.241",
        "10.100.5.39",
        [
            "hardway-k8s-control-node"
        ]
    ],
    [
        "i-0ba2535bc0a4f86a5",
        "13.250.24.174",
        "10.100.2.146",
        [
            "hardway-k8s-control-node"
        ]
    ],
    [
        "i-07d0c3a272e3961a1",
        "52.221.184.21",
        "10.100.1.57",
        [
            "hardway-k8s-control-node"
        ]
    ]
]
```

Next: [Provisioning a Certificate authority](04-certificate-authority.md)
