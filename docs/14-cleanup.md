# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
{
echo "Kill the worker nodes... " && \
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances --filters \
      "Name=tag:Name,Values=worker-0,worker-1,worker-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Some worker nodes take forever to die... " && \
aws ec2 wait instance-terminated \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=worker-0,worker-1,worker-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Issuing shutdown to the controllers... " && \
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances --filter \
      "Name=tag:Name,Values=controller-0,controller-1,controller-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Controllers waiting to die..." && \
aws ec2 wait instance-terminated \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=controller-0,controller-1,controller-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

aws ec2 delete-key-pair --key-name kubernetes
}
```

## Networking

Delete the `kubernetes-the-hard-way` networking nonsense:

```
{
aws elbv2 delete-load-balancer --load-balancer-arn "${LOADBALANCER}"
aws elbv2 delete-target-group --target-group-arn "${TARGETGROUP}"
aws ec2 delete-security-group --group-id "${SECURITYGROUP}"
ROUTE_TABLE_ASSOCIATION_ID="$(aws ec2 describe-route-tables \
  --route-table-ids "${ROUTETABLE}" \
  --output text --query 'RouteTables[].Associations[].RouteTableAssociationId')"
aws ec2 disassociate-route-table --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"
aws ec2 delete-route-table --route-table-id "${ROUTETABLED}"
echo "Waiting a minute for all public address(es) to be unmapped.. " && sleep 60

aws ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNETGW}" \
  --vpc-id "${VPC_ID}"
aws ec2 delete-internet-gateway --internet-gateway-id "${INTERNETGW}"
aws ec2 delete-subnet --subnet-id "${SUBNET}"
aws ec2 delete-vpc --vpc-id "${VPC}"
}
```
