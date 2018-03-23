# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across multiple [Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-regions-availability-zones) (AZ).

> Ensure a default region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region) lab.

## Networking

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


## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/containerd/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

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

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
