# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance, however we will run them manually for testing purpose and then update CloudFormation template to bootstrap master nodes automatically.

## Bootstrapping an etcd Cluster Member

### Run the etcd containers

Instead of installing and running binary we will use official etcd containers as described in [etcd documentation](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/container.md). We also will use etcdctl from container too, so we don't need any local binaries or dependencies.

Do run the service we will use systemd service.

### Configure the etcd Server

First, we create an etcd data dir on the host to preserve data if any issues with containers and set registry and version we want to use.
```bash
DATA_DIR=/var/lib/etcd
mkdir -p $DATA_DIR

REGISTRY=quay.io/coreos/etcd
ETCD_VERSION=3.2
```


```
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:

```
INTERNAL_IP=$(curl -sS --retry 3 http://169.254.169.254/latest/meta-data/local-ipv4)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance and instance id:

```
EC2_INSTANCE_ID=$(curl -sS --retry 3 http://169.254.169.254/latest/meta-data/instance-id)
HOSTNAME=$(curl -sS --retry 3 http://169.254.169.254/latest/meta-data/local-hostname)
```

__TODO__: describe bootstrapping options

In order to use [static bootstrapping process](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md#static) we need to get information about other instances in the cluster due to dynamic nature of ASG. We need this only when we launch a cluster the first time, so later we will work out a solution how to skip those steps and use [runtime configuration](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/runtime-configuration.md) to add node in the cluster in case of failure or upgrade.

```bash
# Get ASG name the instance belong to
ASG_NAME=$(aws autoscaling describe-auto-scaling-instances --instance-ids $EC2_INSTANCE_ID --query AutoScalingInstances[0].AutoScalingGroupName --out text)

# Create query string to get all inService instances and desired state
QUERY="AutoScalingGroups[?AutoScalingGroupName==\`$ASG_NAME\`].{size:DesiredCapacity,instances:Instances[?LifecycleState==\`InService\`].InstanceId}"
DESIRED_SIZE=0
ACTUAL_SIZE=0

# Wait until ASG launch all instances with retry and back-off strategy
for i in `seq 1 10`
do
  # Query all inService instances and desired state and get desired number of instances
  ASG_DATA=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name $ASG_NAME --query $QUERY  --output text)
  DESIRED_SIZE=$(echo "$ASG_DATA" | head -1)
  ACTUAL_SIZE=$(echo "$ASG_DATA" | grep -ci instance)
  # go out of cycle if the number of instances match the desired state
  test  $ACTUAL_SIZE -eq $DESIRED_SIZE && break
  # otherwise wait and try again
  echo "wait till all instances are InService"
  sleep $((i*3))
done

# Final test that ASG has been stabilized (because there are two exits from 'for' loop)
test  $ACTUAL_SIZE -eq $DESIRED_SIZE || (echo "ASG hasn't stabilized"; exit 1)
```

Now if the ASG is reached the desired state we can construct initial cluster flag that should be the same on all cluster members.

```bash
ETCD_INITIAL_CLUSTER=$(aws ec2 describe-instances --instance-ids $INSTANCES --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]' --output text | sort | awk '{ printf("%s=https://%s:2380,", $1, $2) }' | sed 's/,$//g')

```

Create the `etcd.service` systemd unit file:

```bash
cat > etcd.service <<EOF
[Unit]
Description=etcd
After=docker.service
Requires=docker.service
Documentation=https://github.com/coreos

[Service]
TimeoutStartSec=0
ExecStartPre=docker pull ${REGISTRY}:${ETCD_VERSION}
ExecStartPre=docker rm etcd
ExecStart=docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --net host \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name ${EC2_INSTANCE_ID} \
  --initial-advertise-peer-urls http://${INTERNAL_IP}:2380 \
  --listen-peer-urls http://${INTERNAL_IP}:2380 \
  --advertise-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \
  --initial-cluster ${ETCD_INITIAL_CLUSTER} \
  --initial-cluster-state new --initial-cluster-token ${ASG_NAME} \
  --auto-tls \
  --peer-auto-tls

ExecStop=docker stop etcd  
ExecStopPost=docker rm -f etcd  
ExecReload=docker restart etcd
Restart=always
RestartSec=10
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
EOF
```


 <!-- --name ${ETCD_NAME} \\
 --cert-file=/etc/etcd/kubernetes.pem \\
 --key-file=/etc/etcd/kubernetes-key.pem \\
 --peer-cert-file=/etc/etcd/kubernetes.pem \\
 --peer-key-file=/etc/etcd/kubernetes-key.pem \\
 --trusted-ca-file=/etc/etcd/ca.pem \\
 --peer-trusted-ca-file=/etc/etcd/ca.pem \\
 --peer-client-cert-auth \\
 --client-cert-auth \\ -->

### Start the etcd Server

```
sudo mv etcd.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable etcd
```

```
sudo systemctl start etcd
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## Verification

List the etcd cluster members:

```
ETCDCTL_API=3 etcdctl member list
```

> output

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

Next: [Bootstrapping the Kubernetes Control Plane](09-bootstrapping-kubernetes-controllers.md)
