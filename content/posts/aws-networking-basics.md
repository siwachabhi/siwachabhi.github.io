---
title: "AWS networking basics"
date: 2023-03-19T16:21:40-07:00
draft: false
tags: ["aws", "networking"]
---

# Introduction
AWS is great for quickly prottyping infrastructure and flexibility of defining applications. This also introduces some complexity around understanding new concepts related to networking, which developers might not be familiar with. This blog introduces basic AWS networking concepts.

## Glossary
* VPC (Virtual Private Cloud): A virtual network that provides isolation and security for resources within AWS. Think of it as a private data center in the cloud.
* Subnet: A subset of a VPC that allows you to divide the network into smaller segments. Subnets only provides a logic way of grouping IP addresses(not a firewall), all IP addresses within a VPC are accessible from resources in different subnets.
* Route Table: A set of rules that determines where network traffic is directed. These are specific per subnet.
* Security Group: A virtual firewall that controls inbound and outbound traffic for resources within AWS.
* CIDR: Notation to represent IP address range, with IP address followed by a `/` and a number that specifies the number of bits in the network prefix. For example, a CIDR notation of "10.0.0.0/16" represents an IP address range that includes all IP addresses from 10.0.0.0 to 10.0.255.255.

When you create a VPC, you can specify the IP address range and the number of subnets you want to create. Each subnet has a unique IP address range and can be associated with a different availability zone. Once you have created your VPC and subnets, you can launch resources such as EC2 instances, NLB, et. within your VPC. When you launch a resource, you can specify the subnet and the security group that it should be associated with.

## Setting up a vpc with code
Using boto3 to explain the best practice of setting up a AWS vpc with separate route table for private and public subnets. This is to isolate resources in private subnet from any access from internet. A public subnet has [internet gateway(IGW)](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) and [network address translator(NAT)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) with route table of public subnet forwarding traffic to default IP(`0.0.0.0/0`) to internet gateway. Route table of private subnet forwards traffic for default IP to NAT in public subnet. NAT takes care of forwarding traffic to IGW and response back to resources in private subnet.

In production, [CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.Vpc.html) can automate most of this and apply best practices.

```python
import boto3

# Create an EC2 client
ec2 = boto3.client('ec2')

# Create the VPC
response = ec2.create_vpc(CidrBlock='10.0.0.0/16')
vpc_id = response['Vpc']['VpcId']

# Add a name tag to the VPC
ec2.create_tags(Resources=[vpc_id], Tags=[{'Key': 'Name', 'Value': 'MyVPC'}])

# Create an internet gateway and attach it to the VPC
response = ec2.create_internet_gateway()
igw_id = response['InternetGateway']['InternetGatewayId']
ec2.attach_internet_gateway(InternetGatewayId=igw_id, VpcId=vpc_id)

# Create a public subnet
response = ec2.create_subnet(CidrBlock='10.0.1.0/24', VpcId=vpc_id)
public_subnet_id = response['Subnet']['SubnetId']
ec2_client.create_route(RouteTableId='<public_route_table_id>', DestinationCidrBlock='0.0.0.0/0', GatewayId=internet_gateway_id)

# Create a private subnet
response = ec2.create_subnet(CidrBlock='10.0.2.0/24', VpcId=vpc_id)
private_subnet_id = response['Subnet']['SubnetId']

# Create a NAT gateway in the public subnet
response = ec2.create_nat_gateway(SubnetId=public_subnet_id, AllocationId='<allocation_id>')
nat_gateway_id = response['NatGateway']['NatGatewayId']

# Update the route table for the private subnet to use the NAT gateway
# NAT acts a reverse proxy to take in traffic from private subnet and forward to IGW and vice-versa.
response = ec2.create_route(RouteTableId='<private_route_table_id>', DestinationCidrBlock='0.0.0.0/0', NatGatewayId=nat_gateway_id)

# Allow resources in the public subnet to access resources in the private subnet
response = ec2.modify_subnet_attribute(SubnetId=private_subnet_id, MapPublicIpOnLaunch={'Value': False})

print(f"VPC ID: {vpc_id}")
print(f"Public Subnet ID: {public_subnet_id}")
print(f"Private Subnet ID: {private_subnet_id}")
print(f"NAT Gateway ID: {nat_gateway_id}")
```


## Setting up EC2 instance in private subnet of vpc

```python

import boto3

# Create an EC2 client
ec2 = boto3.client('ec2')

# Get the ID of the private subnet
private_subnet_id = 'private-subnet-id'

# Create a security group
security_group_name = 'private-security-group'
response = ec2.create_security_group(GroupName=security_group_name,
                                      Description='Allow TCP traffic on port 8082 from anywhere',
                                      VpcId='<vpc_id>')
security_group_id = response['GroupId']

# Add an ingress rule to allow TCP traffic on port 8082
ec2.authorize_security_group_ingress(GroupId=security_group_id,
                                      IpPermissions=[{'IpProtocol': 'tcp',
                                                      'FromPort': 8082,
                                                      'ToPort': 8082,
                                                      'IpRanges': [{'CidrIp': '10.0.0.0/16'}]}])

# Launch an EC2 instance in the private subnet with the security group
image_id = '<ami_id>'
instance_type = 't2.micro'
key_name = '<key_name>'
private_ip_address = '10.0.2.10'  # Replace with an IP address within the private subnet range
user_data = '#!/bin/bash\necho "Hello, World!" > index.html && nohup python -m SimpleHTTPServer 8082 &'
response = ec2.run_instances(ImageId=image_id,
                              InstanceType=instance_type,
                              KeyName=key_name,
                              SecurityGroupIds=[security_group_id],
                              SubnetId=private_subnet_id,
                              PrivateIpAddress=private_ip_address,
                              UserData=user_data)

instance_id = response['Instances'][0]['InstanceId']
print(f"EC2 instance {instance_id} launched in private subnet with IP address {private_ip_address} and security group {security_group_id}")

```