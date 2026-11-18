# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the all worker instances, then afterwards delete controller instances:

```

aws ec2 describe-instances --region us-east-2 --filters \
  "Name=tag:Name,Values=dev-k8s-worker-0,dev-k8s-worker-1,dev-k8s-worker-2" \
  "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value,InstanceId,PrivateIpAddress,State.Name]' \
  --output table

  
echo "Issuing shutdown to worker nodes.. " && \
aws ec2 terminate-instances --region us-east-2 \
  --instance-ids \
    $(aws ec2 describe-instances --region us-east-2 --filters \
      "Name=tag:Name,Values=dev-k8s-worker-0,dev-k8s-worker-1,dev-k8s-worker-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Waiting for worker nodes to finish terminating.. " && \
aws ec2 wait instance-terminated --region us-east-2 \
  --instance-ids \
    $(aws ec2 describe-instances --region us-east-2 \
      --filter "Name=tag:Name,Values=dev-k8s-worker-0,dev-k8s-worker-1,dev-k8s-worker-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

aws ec2 describe-instances --region us-east-2 --filter \
  "Name=tag:Name,Values=dev-k8s-controller-0,dev-k8s-controller-1,dev-k8s-controller-2" \
  "Name=instance-state-name,Values=running" \
  --output text --query 'Reservations[].Instances[].InstanceId'

echo "Issuing shutdown to master nodes.. " && \
aws ec2 terminate-instances --region us-east-2 \
  --instance-ids \
    $(aws ec2 describe-instances --region us-east-2 --filter \
      "Name=tag:Name,Values=dev-k8s-controller-0,dev-k8s-controller-1,dev-k8s-controller-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Waiting for master nodes to finish terminating.. " && \
aws ec2 wait instance-terminated --region us-east-2 \
  --instance-ids \
    $(aws ec2 describe-instances --region us-east-2 \
      --filter "Name=tag:Name,Values=dev-k8s-controller-0,dev-k8s-controller-1,dev-k8s-controller-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

aws ec2 delete-key-pair --key-name kubernetes
```

## Networking

Delete the external load balancer network resources:

```
aws elbv2 delete-load-balancer --load-balancer-arn "${LOAD_BALANCER_ARN}"
aws elbv2 delete-target-group --target-group-arn "${TARGET_GROUP_ARN}"
aws ec2 delete-security-group --group-id "${SECURITY_GROUP_ID}"
ROUTE_TABLE_ASSOCIATION_ID="$(aws ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --output text --query 'RouteTables[].Associations[].RouteTableAssociationId')"
aws ec2 disassociate-route-table --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"
aws ec2 delete-route-table --route-table-id "${ROUTE_TABLE_ID}"
echo "Waiting a minute for all public address(es) to be unmapped.. " && sleep 60

aws ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}" \
  --vpc-id "${VPC_ID}"
aws ec2 delete-internet-gateway --internet-gateway-id "${INTERNET_GATEWAY_ID}"
aws ec2 delete-subnet --subnet-id "${SUBNET_ID}"
aws ec2 delete-vpc --vpc-id "${VPC_ID}"

```