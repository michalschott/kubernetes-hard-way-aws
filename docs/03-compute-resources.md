# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a simple Kubernetes cluster across a single compute zone.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated Virtual Private Cloud (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC:

```
export myIP=`curl -s ifconfig.co`
export vpcId=`aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text`
export internetGatewayId=`aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text`

aws ec2 create-tags --resources ${vpcId} --tags Key=Name,Value=kubernetes-the-hard-way
aws ec2 modify-vpc-attribute --vpc-id $vpcId --enable-dns-support "{\"Value\":true}"
aws ec2 modify-vpc-attribute --vpc-id $vpcId --enable-dns-hostnames "{\"Value\":true}"
aws ec2 attach-internet-gateway --internet-gateway-id $internetGatewayId --vpc-id $vpcId
```

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC:

```
export subnetId=`aws ec2 create-subnet --vpc-id $vpcId --cidr-block 10.0.10.0/24 --query 'Subnet.SubnetId' --output text`
export routeTableId=`aws ec2 create-route-table --vpc-id $vpcId --query 'RouteTable.RouteTableId' --output text`

aws ec2 associate-route-table --route-table-id $routeTableId --subnet-id $subnetId
aws ec2 create-route --route-table-id $routeTableId --destination-cidr-block 0.0.0.0/0 --gateway-id $internetGatewayId
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal and external (it will use your current external IP address) communication across all protocols:

```
export securityGroupId=`aws ec2 create-security-group --group-name my-security-group --description "my-security-group" --vpc-id $vpcId --query 'GroupId' --output text`

aws ec2 authorize-security-group-ingress --group-id $securityGroupId --protocol all --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $securityGroupId --protocol all --cidr ${myIP}/32
```

> An external load balancer with allocated static IP should be used to expose the Kubernetes API Servers to remote clients, but this will not be covered in this tutorial.

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the Docker runtime. Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process. We will use two `t2.small` instances - controller and worker.

### Kubernetes Controller

Create EC2 instance which will host the Kubernetes control plane:

```
export AMI_ID=`aws ec2 describe-images --owners 099720109477 --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' --query 'Images[*].[ImageId,CreationDate]' --output text | sort -k2 -r | head -n1 | cut -f 1`
aws ec2 create-key-pair --key-name kubernetes-the-hard-way --query 'KeyMaterial' --output text > ~/.ssh/kubernetes-the-hard-way.pem
chmod 400 ~/.ssh/kubernetes-the-hard-way.pem
ssh-add ~/.ssh/kubernetes-the-hard-way.pem >/dev/null
export MASTER_ID=`aws ec2 run-instances --image-id ${AMI_ID} --count 1 --instance-type t2.small --key-name kubernetes-the-hard-way --security-group-ids ${securityGroupId} --subnet-id ${subnetId} --private-ip-address 10.0.10.11 --associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=controller}]' --query 'Instances[*].InstanceId' --output text`
export MASTER_EXT_IP=`aws ec2 describe-instances --instance-ids ${MASTER_ID} --query 'Reservations[*].Instances[*].PublicIpAddress' --output text`
```

### Kubernetes Workers

Create EC2 instance which will host the Kubernetes worker node:

```
export WORKER_ID=`aws ec2 run-instances --image-id ${AMI_ID} --count 1 --instance-type t2.small --key-name kubernetes-the-hard-way --security-group-ids ${securityGroupId} --subnet-id ${subnetId} --private-ip-address 10.0.10.21 --associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker}]' --query 'Instances[*].InstanceId' --output text`
export WORKER_EXT_IP=`aws ec2 describe-instances --instance-ids ${WORKER_ID} --query 'Reservations[*].Instances[*].PublicIpAddress' --output text`
```

### Verification

List the compute instances in your default compute zone:

```
ssh ubuntu@${MASTER_EXT_IP}
ssh ubuntu@${WORKER_EXT_IP}
```

> You might get a warning about invalid locales. Just follow the guide - install missing package and reload the ssh session.

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
