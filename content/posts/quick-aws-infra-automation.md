---
title: "Quick AWS Infrastructure automations with python boto3"
date: 2023-03-19T14:21:40-07:00
draft: false
tags: ["aws", "automation", "python"]
---

# Introduction
AWS is excellent to quickly protoype infrastructure proposals. We can access AWS console, manually spin up any service we want, experiment with it and spin it down. When moving to production, we can manage AWS infrastructure as code using CDK and keep it up to date with cloudformation deployments, ensuring software development CI/CD best practices.

### Problem
There is an apparent gap between these two stages, if infrastructure to be setup is fairly complex and team wants to move very fast to production, to enable faster time to market. These are scenarios where manually creating all of the infrastructure is error prone and time consuming. On the other hand, auatomating using CDK/cloudaformation is too slow.

### Solution
[Python boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) client provides excellent apis to access AWS control control plane. Python's fast scripting prowess and intuitive boto3 apis, opens up a middle ground to automate AWS infrastructure operations and still provide a faster time to market. Team can then eventually use CDK to automate complete infrastructure management and deployment.

# Example
Consider a scenario of enabling cross vpc access for MSK via private link. Here is an [AWS blog](https://aws.amazon.com/blogs/big-data/how-goldman-sachs-builds-cross-account-connectivity-to-their-amazon-msk-clusters-with-aws-privatelink/), which explains the solution but in practice there are so many moving pieces that it becomes hard to even set this up once, to think if we have to perform the same setup in multiple regions/stages is daunting to say the least. 

Rest of the blog assumes pattern-2 setup, as its more frugal in terms of cost. 
{{< figure src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2020/05/26/GSMSKPrivateLink_rev2.png" caption="Source: AWS blog how goldman sachs builds cross account connectivity to their amazon msk clusters with aws privatelink" 
link="https://aws.amazon.com/blogs/big-data/how-goldman-sachs-builds-cross-account-connectivity-to-their-amazon-msk-clusters-with-aws-privatelink/" >}}

*Note: Below code examples are just to express the idea, might not work as it is.*


### Setting up VPC

Setup a vpc with 3 private subnet, 3 public subnet. 

```python
# Create the VPC
response = ec2_client.create_vpc(CidrBlock='10.0.0.0/16')
vpc_id = response['Vpc']['VpcId']

# Create the public subnets
public_subnet_cidr_blocks = ['10.0.1.0/24', '10.0.2.0/24', '10.0.3.0/24']
public_subnet_ids = []
for cidr_block in public_subnet_cidr_blocks:
    response = ec2_client.create_subnet(CidrBlock=cidr_block, VpcId=vpc_id)
    public_subnet_ids.append(response['Subnet']['SubnetId'])

# Create the private subnets
private_subnet_cidr_blocks = ['10.0.4.0/24', '10.0.5.0/24', '10.0.6.0/24']
private_subnet_ids = []
for cidr_block in private_subnet_cidr_blocks:
    response = ec2_client.create_subnet(CidrBlock=cidr_block, VpcId=vpc_id)
    private_subnet_ids.append(response['Subnet']['SubnetId'])
```


### Setting up MSK cluster

Setup a MSK with private subnets of above vpc, with a security group allowing traffic to broker and zookeeper nodes.

``` python
# Get the IDs of the VPC and private subnets
vpc_id = 'your_vpc_id'
private_subnet_ids = ['your_private_subnet_id_1', 'your_private_subnet_id_2', 'your_private_subnet_id_3']

# Create the security group for MSK
response = ec2_client.create_security_group(
    GroupName='msk-sg',
    VpcId=vpc_id,
)
msk_security_group_id = response['GroupId']
# Allow ingress traffic to msk and zookeeper ports

response = msk_client.create_cluster(
    ClusterName='your_cluster_name',
    KafkaVersion='2.7.0',
    NumberOfBrokerNodes=3,
    BrokerNodeGroupInfo={
        'InstanceType': 'kafka.m5.large',
        'ClientSubnets': private_subnet_ids,
        'SecurityGroups': [msk_security_group_id],
    }
)

```

### Setting up NLB


``` python
# Get the IDs of the VPC and private subnets
vpc_id = 'your_vpc_id'
private_subnet_ids = ['your_private_subnet_id_1', 'your_private_subnet_id_2', 'your_private_subnet_id_3']

# Set up the NLB listener ports and target groups
nlb_listener_ports = [7001, 7002, 7003]
nlb_target_groups = []

for i in range(len(nlb_listener_ports)):
    port = nlb_listener_ports[i]
    target_group_name = f'msk-tg-{port}'
    target_group_response = elbv2_client.create_target_group(
        Name=target_group_name,
        Protocol='TCP',
        Port=port,
        VpcId=vpc_id,
    )
    target_group_arn = target_group_response['TargetGroups'][0]['TargetGroupArn']
    nlb_target_groups.append(target_group_arn)

# Set up the NLB
nlb_name = 'your_nlb_name'
nlb_response = elbv2_client.create_load_balancer(
    Name=nlb_name,
    Subnets=private_subnet_ids,
    Type='network',
)
nlb_arn = nlb_response['LoadBalancers'][0]['LoadBalancerArn']
# Attach the target groups to the NLB
# Create the VPC endpoint service, so that clients can creat an endpoint to this in their vpc.
# Clients will need to create dns entry in their vpc, which will forward traffic to their endpoint.

# Get the details of the MSK brokers
msk_client = boto3.client('kafka')
msk_brokers_response = msk_client.list_nodes(ClusterArn='your_msk_cluster_arn')
msk_brokers = msk_brokers_response['NodeInfoList']

# Register the MSK brokers with the NLB target groups
for i in range(len(nlb_target_groups)):
    target_group_arn = nlb_target_groups[i]
    msk_broker = msk_brokers[i]
    client_vpc_address = msk_broker['ClientVpcIpAddress']
    ec2_instance_id = ec2_client.describe_instances(Filters=[{'Name': 'private-ip-address', 'Values': [client_vpc_address]}])['Reservations'][0]['Instances'][0]['InstanceId']
    target_response = elbv2_client.register_targets(
        TargetGroupArn=target_group_arn,
        Targets=[{'Id': ec2_instance_id, 'Port': 9092}],
    )
```

### Modify advertised listener of MSK brokers

AWS has [good documentation for this](https://aws.amazon.com/premiumsupport/knowledge-center/msk-broker-custom-ports/), essentially we need to modify advertised.listener configuration of each broker. To do this we can create a EC2 instance in public subnet of MSK, and perform [kafka cli setup](https://docs.aws.amazon.com/msk/latest/developerguide/create-topic.html). While modifying DNS we will also need to update it to match to the external dns that clients will create in their private hosted zone.

```bash
#!/bin/bash

MSK_DNS="<msk-cluster-dns-name>"
CUSTOM_DNS_NAME="<custom-external-dns-name-for-nlb-clients>"
PORT_START=7000

for ((BROKER_ID=1; BROKER_ID<=3; BROKER_ID++))
do
  ((PORT=PORT_START+BROKER_ID))
  CONFIG_VALUE="advertised.listeners=[CLIENT://$CUSTOM_DNS_NAME:$PORT,CLIENT_SECURE://b-$BROKER_ID_START.$MSK_DNS:9094,REPLICATION://b-$BROKER_ID_START-internal.$MSK_DNS:9093,REPLICATION_SECURE://b-$BROKER_ID_START-internal.$MSK_DNS:9095]"
  echo "Updating $ENTITY_TYPE $ENTITY_NAME config property to $CONFIG_VALUE"
  ./kafka-configs.sh --bootstrap-server "b-1.$MSK_DNS:9094" --entity-type "brokers" --entity-name "1" --alter --command-config client.properties --add-config $CONFIG_VALUE
done


```

## Take Aways
Even in scenarios of urgent time to market scenario, prefer to automate infrastructure setup. With high flexibility of AWS infrastructure, it also creates scenarios of manual misses, at the bare mimum use boto3 for quick and dirty automation. Long term prefer CDK for AWS infrastructure management via CI/CD integration.