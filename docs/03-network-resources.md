# Provisioning Network Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the underlying network resources for running a secure and highly available Kubernetes cluster across multiple [Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-regions-availability-zones) (AZ).

> Ensure a default region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region) lab.

## Kubernetes Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

We are going to use [VPC CNI plugin](https://github.com/aws/amazon-vpc-cni-k8s) created by [AWS EKS](https://aws.amazon.com/eks/) team so Kubernetes pods will be using VPC network ip addresses without any overlay network.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://aws.amazon.com/vpc/) (VPC) network will be setup to host the Kubernetes cluster. To simplify this you can use Cloudformation template [network.yaml](../templates/network.yaml):

```
export ENV=hardway-k8s
export CFN_NET=$ENV-net

aws cloudformation create-stack --stack-name $CFN_NET --template-body file://./templates/network.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=$ENV
```

This template will provision a VPC with 10.100.0.0/16 CIDR divided into three small subnets in three AZ for masters nodes with ip ranges:
- 10.100.0.0/23
- 10.100.2.0/23
- 10.100.4.0/23

and three bigger subnets in three AZs for worker nodes and pods:
- 10.100.16.0/20
- 10.100.32.0/20
- 10.100.48.0/20

Worker subnets must be provisioned with an IP address range large enough to assign a private IP address to each node and pods on this node in the Kubernetes cluster, since pods use IP addresses from the same subnet where their node is ([more info about CNI over VPC plugin for AWS](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/proposals/cni-proposal.md)).

CloudFormation template also creates other related resources, such as Internet Gateway and route table. You can override the VPC CIDR and subnets' size providing parameters differ from defaults.

If you want to create VPC and subnets manually with AWS CLI you can follow [official AWS guide](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-subnets-commands-example.html) just remember to spread your subnets for master and worker nodes per AZ and choose appropriate subnet sizes with the above considerations.

Once stack is created we can grab output values for VPC and subnet IDs we need later.

```
export VPC=$(aws cloudformation describe-stacks --stack-name $CFN_NET --query 'Stacks[0].Outputs[?OutputKey==`VPCId`].OutputValue' --output text)

export WORKER_SUBNETS=$(aws cloudformation describe-stacks --stack-name $CFN_NET --query 'Stacks[0].Outputs[?OutputKey==`WorkerSubnets`].OutputValue' --output text)

export CONTROL_SUBNETS=$(aws cloudformation describe-stacks --stack-name $CFN_NET --query 'Stacks[0].Outputs[?OutputKey==`ControlSubnets`].OutputValue' --output text)
```

### Security groups

In this section we will create a set of security groups to allow communications between masters and nodes and incoming connections to masters API. Again we will use a CloudFormation template [sg.yaml](../templates/sg.yaml) for this as creating the underlying infrastructure is not the main goal of this guide.

```
export CFN_SG=$ENV-sg

aws cloudformation create-stack --stack-name $CFN_SG --template-body file://templates/sg.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=$ENV ParameterKey=VPC,ParameterValue=$VPC
```

This template will create three security groups:
- For worker nodes with all traffic allowed from masters and other workers
- For master nodes with all traffic allowed from other masters, and API port access from Load balancer and workers
- For a load balancer

> An [Application load balancer](https://aws.amazon.com/elasticloadbalancing/) will be used to expose the Kubernetes API Servers to remote clients.

Get security groups and list rules:

```
export LB_SG=$(aws cloudformation describe-stacks --stack-name $CFN_SG --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerSecurityGroup`].OutputValue' --output text)

export CONTROL_SG=$(aws cloudformation describe-stacks --stack-name $CFN_SG --query 'Stacks[0].Outputs[?OutputKey==`ControlNodesSecurityGroup`].OutputValue' --output text)

export WORKER_SG=$(aws cloudformation describe-stacks --stack-name $CFN_SG --query 'Stacks[0].Outputs[?OutputKey==`WorkerNodesSecurityGroup`].OutputValue' --output text)

aws ec2 describe-security-groups --group-ids $LB_SG $WORKER_SG $CONTROL_SG
```

### Kubernetes Public Access
__TODO__: confirm the statement below

In this lab AWS [Elastic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html) is used to provide access to Kubernetes API for external clients. It is configured in TCP mode to allow SSL passthrough and terminate SSL on master nodes. It allows to use [client certificate authentication](https://kubernetes.io/docs/admin/authentication/#x509-client-certs) alongside with [webhook authentication](https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication) based on IAM provided by [heptio authenticator](https://github.com/heptio/authenticator).

To simplify this step we will use CloudFormation template [elb.yaml](../templates/elb.yaml) to provision an Elastic Load Balancer.

```
export CFN_ELB=$ENV-elb

aws cloudformation create-stack --stack-name $CFN_ELB --template-body file://templates/elb.yaml --parameters \
    ParameterKey=EnvironmentName,ParameterValue=$ENV \
    ParameterKey=Subnets,ParameterValue=\"$CONTROL_SUBNETS\" \
    ParameterKey=LoadBalancerSecurityGroup,ParameterValue=$LB_SG
```


Now we can get URL of the Load Balancer and Load Balancer name for further usage.

```
export LB_NAME=$(aws cloudformation describe-stacks --stack-name $CFN_ELB --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancer`].OutputValue' --output text)

export LB_URL=$(aws cloudformation describe-stacks --stack-name $CFN_ELB --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerUrl`].OutputValue' --output text)
```


Next: [Provisioning compute resources](04-compute-resources.md)
