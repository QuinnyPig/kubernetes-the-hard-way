# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [AWS Region](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/).

> Ensure a default AWS Region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://aws.amazon.com/vpc/) (VPC) network will be setup to host the Kubernetes cluster.

Create a custom VPC network:

```
export VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
echo $VPC
# vpc-0d8afe938a9303739
aws ec2 associate-vpc-cidr-block --vpc-id $VPC --cidr-block 10.64.0.0/24
aws ec2 modify-vpc-attribute --vpc-id ${VPC} --enable-dns-hostnames '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC} --enable-dns-support '{"Value": true}'

```

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create a public subnet inside of that VPC network:

```
export SUBNET=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 --query Subnet.SubnetId --output text)
echo $SUBNET
# subnet-0d6f9fc23c2a159da
aws ec2 create-tags --resources ${SUBNET} --tags Key=Name,Value=kubernetes

```

> The `10.0.1.0/24` IP address range can host up to 254 compute instances.

### Routing & Firewall

Create an internet gateway and attach it to the VPC network:

```
export INTERNETGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC --internet-gateway-id $INTERNETGW
```

Create and attach a route table that tells instances in that subnet to use the internet gateway, and set the subnet to automatially map public IP addresses to private IP addresses on launch:

```
export ROUTETABLE=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
echo $ROUTETABLE
# rtb-02c3ef4e030f0d5b2
aws ec2 create-route --route-table-id $ROUTETABLE --destination-cidr-block 0.0.0.0/0 --gateway-id $INTERNETGW
aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch

```

Verify that the route table is associated with the VPC network:

```
aws ec2 describe-route-tables --route-table-id $ROUTETABLE
```

> output

```
{
    "RouteTables": [
        {
            "Associations": [],
            "PropagatingVgws": [],
            "RouteTableId": "rtb-02c3ef4e030f0d5b2",
            "Routes": [
                {
                    "DestinationCidrBlock": "10.0.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                },
                {
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "GatewayId": "igw-01eef992992af026b",
                    "Origin": "CreateRoute",
                    "State": "active"
                }
            ],
            "Tags": [],
            "VpcId": "vpc-0d8afe938a9303739",
            "OwnerId": "677745302030" // TODO
        }
    ]
}
```

Create a security group and create rules that allow external SSH, ICMP, and HTTPS:

```
export SECURITYGROUP=$(aws ec2 create-security-group --group-name kubernetes --description "Kubernetes the Much Harder Way VPC" --vpc-id $VPC --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=0.0.0.0/0}]"
aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --protocol icmp --port -1  --cidr  0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --protocol all --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SECURITYGROUP --protocol all --cidr 10.200.0.0/16

```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 describe-security-group-rules --filter Name="group-id",Values=$SECURITYGROUP --output text
```

> output

```
SECURITYGROUPRULES      0.0.0.0/0       -1      sg-0a94843f57b84ff52    677745302030    icmp    False   sgr-00773748f62da6283   -1
SECURITYGROUPRULES      0.0.0.0/0       -1      sg-0a94843f57b84ff52    677745302030    -1      True    sgr-0b464d937067a19f3   -1
SECURITYGROUPRULES      0.0.0.0/0       22      sg-0a94843f57b84ff52    677745302030    tcp     False   sgr-050b4d0435f49dbd1   22
SECURITYGROUPRULES      0.0.0.0/0       6443    sg-0a94843f57b84ff52    677745302030    tcp     False   sgr-0d09805560760b24b   6443
```

### Kubernetes Public IP Address

# TODO

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
export LOADBALANCER=$(aws elbv2 create-load-balancer \
    --name kubernetes \
    --subnets ${SUBNET} \
    --scheme internet-facing \
    --type network \
    --output text --query 'LoadBalancers[].LoadBalancerArn')
export TARGETGROUP=$(aws elbv2 create-target-group \
    --name kubernetes \
    --protocol TCP \
    --port 6443 \
    --vpc-id ${VPC} \
    --target-type ip \
    --output text --query 'TargetGroups[].TargetGroupArn')
aws elbv2 register-targets --target-group-arn ${TARGETGROUP} --targets Id=10.0.1.1{0,1,2}
aws elbv2 create-listener \
    --load-balancer-arn ${LOADBALANCER} \
    --protocol TCP \
    --port 443 \
    --default-actions Type=forward,TargetGroupArn=${TARGETGROUP} \
    --output text --query 'Listeners[].ListenerArn'
export PUBLICADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION    SUBNET  STATUS
kubernetes-the-hard-way  XX.XXX.XXX.XXX  EXTERNAL                    us-west1          RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
export AMI=$(aws ssm get-parameters \
        --names '/aws/service/canonical/ubuntu/server/20.04/stable/current/amd64/hvm/ebs-gp2/ami-id' \
        --query 'Parameters[0].[Value]' --output text)

aws ec2 create-key-pair --key-name kubernetes --query 'KeyMaterial' --output text > kubernetes.rsa
chmod 600 kubernetes.rsa
for i in 0 1 2; do
      ID=$(aws ec2 run-instances \
        --image-id $AMI \
        --instance-type t3.small \
        --key-name kubernetes \
        --private-ip-address 10.0.1.1${i} \
        --security-group-ids $SECURITYGROUP \
        --subnet-id $SUBNET \
        --user-data "name=controller-${i}" \
        --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
        --associate-public-ip-address \
        --output text \
        --query 'Instances[0].InstanceId')
      aws ec2 create-tags --resources ${ID} --tags "Key=Name,Value=controller-${i}"
      aws ec2 modify-instance-attribute --instance-id ${ID} --no-source-dest-check
      echo "Created controller-${i}"
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
      ID=$(aws ec2 run-instances \
        --image-id $AMI \
        --instance-type t3.small \
        --key-name kubernetes \
        --private-ip-address 10.0.1.2${i} \
        --security-group-ids $SECURITYGROUP \
        --subnet-id $SUBNET \
        --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
        --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
        --associate-public-ip-address \
        --output text \
        --query 'Instances[0].InstanceId')
      aws ec2 create-tags --resources ${ID} --tags "Key=Name,Value=worker-${i}"
      aws ec2 modify-instance-attribute --instance-id ${ID} --no-source-dest-check
      echo "Created worker-${i}"
done
```

### Verification

TODO
List the compute instances in your default compute zone:

```
gcloud compute instances list --filter="tags.items=kubernetes-the-hard-way"
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
controller-0  us-west1-c  e2-standard-2               10.240.0.10  XX.XX.XX.XXX   RUNNING
controller-1  us-west1-c  e2-standard-2               10.240.0.11  XX.XXX.XXX.XX  RUNNING
controller-2  us-west1-c  e2-standard-2               10.240.0.12  XX.XXX.XX.XXX  RUNNING
worker-0      us-west1-c  e2-standard-2               10.240.0.20  XX.XX.XXX.XXX  RUNNING
worker-1      us-west1-c  e2-standard-2               10.240.0.21  XX.XX.XX.XXX   RUNNING
worker-2      us-west1-c  e2-standard-2               10.240.0.22  XX.XXX.XX.XX   RUNNING
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
