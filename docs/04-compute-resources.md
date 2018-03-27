# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across multiple [Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-regions-availability-zones) (AZ).

## Supplemental resources

We will use a set of supplement resources in order to simplify cluster management, such as SSM Parameter store and EC2 instance profiles with IAM roles.

### SSM Parameter Store or S3
__TODO__

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

aws cloudformation create-stack --stack-name $CFN_RESOURCES --template-body file://templates/resources.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=$ENV --capabilities CAPABILITY_NAMED_IAM
```

Now get ARN of instance profiles to use later for instances.

```
export NODE_PROFILE=$(aws cloudformation describe-stacks --stack-name $CFN_RESOURCES --query 'Stacks[0].Outputs[?OutputKey==`NodeInstanceProfile`].OutputValue' --output text)

export MASTER_PROFILE=$(aws cloudformation describe-stacks --stack-name $CFN_RESOURCES --query 'Stacks[0].Outputs[?OutputKey==`MasterInstanceProfile`].OutputValue' --output text)
```


## Compute Instances

The compute instances in this lab will be provisioned using [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2) minimal, which provides packets that enable easy integration with AWS and tuned kernel to work on EC2. We will use [AutoScalingGroup](https://aws.amazon.com/ec2/autoscaling) to ensure that all instances are healthy and replaced in case of failure.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```


### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```


Next: [Provisioning a Certificate authority](04-certificate-authority.md)
