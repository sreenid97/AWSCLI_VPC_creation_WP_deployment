# VPC creation and Wordpress deployment using AWS CLI


[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)
## Description
Terraform has been the standard choice for creating and managing AWS infrastructure. However, I am exploring the possibilities of using AWS CLI to create the infra to launch a Wordpress website.
<br />

Here is a simple documentation on how to use AWS CLI to build an AWS VPC along with private/public Subnet and Network Gateways for the VPC. We will create a VPC with three Subnets: 1 Private and 2 Public, a NAT Gateway, an Internet Gateway, and two Route Tables. We will launch a bastion instance on the first public subnet, a frontend instance on the second public subnet, and a database instance on the private subnet.
<br />

## Features

- All resources are created using AWS CLI.
- Bastion, frontend, and backend instances are used to enhance security. 
- The database server will be placed in a private subnet, and access to the backend database server will be limited to the bastion and fronted servers.

## Prerequisites for this project

- IAM user with programmatic access and permission to create the required resources.
- Knowledge of the working principles of AWS services like VPC and EC2.
- A name tag is appended to the resources to make it easier to identify them quickly.

## Required Resources

>2 Public subnets

>1 Private subnet

>1 Public route table

>1 Private route table

>1 Elastic IP

>1 NAT gateway

>1 Internet gateway

>1 Bastion instance

>1 Frontend webserver instance

>1 Backend database instance
<br />
<br />

## Configuring AWS CLI

Configure AWS CLI using the below command.
```sh
$ aws configure
AWS Access Key ID [None]:XXXXXXXXXXXX 
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXX
Default region name [None]: us-east-2
Default output format [None]: json
```
## Creating VPC

Create a VPC with a 172.16.0.0/16 CIDR block using the following command, and it will return the ID of the new VPC.
```sh
$ aws ec2 create-vpc--cidr-block 172.16.0.0/16 --query Vpc.VpcId --output text
vpc-03085adf5b5ffa75e
````
Also, add a name tag to the new VPC.

```sh
$ aws ec2 create-tags --resources vpc-03085adf5b5ffa75e --tags Key=Name,Value=VPC-ProjectA
```

## Creating subnets and adding tags to each subnet

a. Create a subnet in your VPC with a 172.16.0.0/18 CIDR block.  (Here, I am creating it in the us-east-2a availability zone.  If no availability zone is specified, a subnet will be created in any of the availability zones in the us-east-2 region)
```sh
$ aws ec2 create-subnet --vpc-id vpc-03085adf5b5ffa75e --cidr-block 172.16.0.0/18 --availability-zone us-east-2a
{
    "Subnet": {
        "AvailabilityZone": "us-east-2a",
        "AvailabilityZoneId": "use2-az1",
        "AvailableIpAddressCount": 16379,
        "CidrBlock": "172.16.0.0/18",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "available",
        "SubnetId": "subnet-02dc66170b1b16f79",
        "VpcId": "vpc-03085adf5b5ffa75e",
        "OwnerId": "257637015312",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:us-east-2:257637015312:subnet/subnet-02dc66170b1b16f79"
    }
}
```
```sh
$ aws ec2create-tags --resources subnet-02dc66170b1b16f79 --tagsKey=Name,Value=Subnet1-ProjectA
```

b. Create the second subnet in your VPC with a 172.16.64.0/18 CIDR block.
```sh
$ aws ec2 create-subnet --vpc-id vpc-03085adf5b5ffa75e --cidr-block 172.16.64.0/18 --availability-zone us-east-2b
{
    "Subnet": {
        "AvailabilityZone": "us-east-2b",
        "AvailabilityZoneId": "use2-az2",
        "AvailableIpAddressCount": 16379,
        "CidrBlock": "172.16.64.0/18",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "available",
        "SubnetId": "subnet-017aa5250fa2c5b82",
        "VpcId": "vpc-03085adf5b5ffa75e",
        "OwnerId": "257637015312",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:us-east-2:257637015312:subnet/subnet-017aa5250fa2c5b82"
    }
}
```
```sh
$ aws ec2create-tags --resources subnet-017aa5250fa2c5b82 --tagsKey=Name,Value=Subnet2-ProjectA
```

d.  Create the third subnet in your VPC with a 172.16.128.0/18 CIDR block.  We are using this as a private subnet to host the database server.
```sh
$ aws ec2 create-subnet --vpc-id vpc-03085adf5b5ffa75e --cidr-block 172.16.128.0/18 --availability-zone us-east-2b
{
    "Subnet": {
        "AvailabilityZone": "us-east-2b",
        "AvailabilityZoneId": "use2-az2",
        "AvailableIpAddressCount": 16379,
        "CidrBlock": "172.16.128.0/18",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "available",
        "SubnetId": "subnet-03956c03687b5d523",
        "VpcId": "vpc-03085adf5b5ffa75e",
        "OwnerId": "257637015312",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:us-east-2:257637015312:subnet/subnet-03956c03687b5d523"
    }
}
```
```sh
$ aws ec2create-tags --resources subnet-03956c03687b5d523 --tagsKey=Name,Value=Subnet3-ProjectA
```
e. Enable public IP for the first two subnets. 
```sh
$ aws ec2 modify-subnet-attribute --subnet-id subnet-02dc66170b1b16f79 --map-public-ip-on-launch
$ aws ec2 modify-subnet-attribute --subnet-id subnet-017aa5250fa2c5b82 --map-public-ip-on-launch
```
We can see the complete information of the VPC and subnets using the below command. 
```sh
$ aws ec2 describe-subnets  --filters "Name=vpc-id,Values=vpc-03085adf5b5ffa75e"
```
## Creating Internet Gateway

Create an Internet Gateway using the following command, which returns the new Internet Gateway ID.
```sh
$ aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text
igw-0b2e7ae4a59356d8b
```
Attach the internet gateway to our VPC to allow the instances(and NAT) to communicate with the outside world.
```sh
$aws ec2 attach-internet-gateway --vpc-id vpc-03085adf5b5ffa75e--internet-gateway-id igw-0b2e7ae4a59356d8b
```

##  Creating NAT Gateway

Allocates an Elastic IP address to your AWS account using the allocate-address command
```sh
$ aws ec2 allocate-address
{
    "PublicIp": "18.189.128.38",
    "AllocationId": "eipalloc-0e1fb7d65fa3aea79",
    "PublicIpv4Pool": "amazon",
    "NetworkBorderGroup": "us-east-2",
    "Domain": "vpc"
}
```

Creating a NAT Gateway
```sh
$ aws ec2 create-nat-gateway --subnet-id subnet-017aa5250fa2c5b82 --allocation-id eipalloc-0e1fb7d65fa3aea79
{
    "ClientToken": "dc9f6645-68de-4451-bcc9-ccc1be15ed80",
    "NatGateway": {
        "CreateTime": "2022-12-09T15:16:46.000Z",
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-0e1fb7d65fa3aea79"
            }
        ],
        "NatGatewayId": "nat-06110607591e5ccc2",
        "State": "pending",
        "SubnetId": "subnet-017aa5250fa2c5b82",
        "VpcId": "vpc-03085adf5b5ffa75e"
    }
}
```

##  Updating route tables

a. When we create a VPC, a route table will be created by default.  We can find the default route table information of our VPC using the below command
```sh
$ aws ec2describe-route-tables --filters"Name=vpc-id,Values=vpc-03085adf5b5ffa75e"
{
    "RouteTables": 
           {
              "Associations": [
                {
                    "Main": true,
                    "RouteTableAssociationId": "rtbassoc-0c409c8f5a7e7c3f6",
                    "RouteTableId": "rtb-0a890bf93736cb0a2",
                    "AssociationState": {
                        "State": "associated"
                    }
                }
            ],
            "PropagatingVgws": [],
            "RouteTableId": "rtb-0a890bf93736cb0a2",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                },
                {
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "GatewayId": "igw-0b2e7ae4a59356d8b",
                    "Origin": "CreateRoute",
                    "State": "active"
                }
            ],
            "Tags": [],
            "VpcId": "vpc-03085adf5b5ffa75e",
            "OwnerId": "257637015312"
        }
}
```

b. Create a custom route table(for private subnet) using the below command and associate the private subnet with it.
```sh
$ aws ec2create-route-table --vpc-id vpc-03085adf5b5ffa75e --queryRouteTable.RouteTableId --output text
rtb-02dcf33e8169f7924
```
```sh
$ aws ec2 associate-route-table  --subnet-id subnet-03956c03687b5d523 --route-table-id rtb-02dcf33e8169f7924
```

c. Add route table entries.

Add route table entry for IGW in the default route table to allow the instances to communicate with the internet.
```sh
$ aws ec2create-route --route-table-id rtb-0a890bf93736cb0a2--destination-cidr-block 0.0.0.0/0 --gateway-id igw-0b2e7ae4a59356d8b
{
"Return":true
}
```

Add route table entry in the custom route table to enable instances in the private subnet to communicate with the NAT gateway.
```sh
$ aws ec2 create-route --route-table-id rtb-02dcf33e8169f7924 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-06110607591e5ccc2
{
    "Return": true
}
```

To confirm that the route has been created and is active, we can describe the route table using the following describe-route-tables command.
```sh
$ aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-03085adf5b5ffa75e"
```

## Creating security groups

Security group for bastion server
```sh
$ aws ec2create-security-group --group-name ProjectA-bastion --description"Allow SSH access" --vpc-id vpc-03085adf5b5ffa75e
{
"GroupId":"sg-0b7b6e9e9ea4674c4"
}
```

Security group for the frontend web server
```sh
$ aws ec2create-security-group --group-name ProjectA-webserver --description"Allow SSH access from bastion and  http access from all"--vpc-id vpc-03085adf5b5ffa75e
{
"GroupId":"sg-0f39fa6da9f0f9242"
}
```
Security group for the backend database server
```sh
$ aws ec2create-security-group --group-name ProjectA-dbserver --description"Allow SSH access from bastion"
{
"GroupId":"sg-0c2610ed47d4db9ad"
}
```

Add rules to the security groups.


> Add a rule to allow SSH access to the bastion server.
```sh
$ aws ec2 authorize-security-group-ingress --group-id sg-0b7b6e9e9ea4674c4 --protocol tcp --port 22 --cidr 0.0.0.0/0
```
> Add a rule to allow SSH access to the frontend server from the bastion server.
```sh
$ aws ec2 authorize-security-group-ingress --group-id sg-0f39fa6da9f0f9242 --protocol tcp --port 22 --source-group sg-0b7b6e9e9ea4674c4
```
> Add a rule to allow HTTP access to the frontend server from the outside world(All IPv4).
```sh
$ aws ec2 authorize-security-group-ingress --group-id sg-0f39fa6da9f0f9242 --protocol tcp --port 80 --cidr 0.0.0.0/0
```
> Add a rule to allow SSH access to the backend server from the bastion server.
```sh
$ aws ec2 authorize-security-group-ingress --group-id sg-0ec04f86bf2557add --protocol tcp --port 22 --source-group sg-0b7b6e9e9ea4674c4
```
> Add a rule to enable the frontend server to access mysql on the backend server.
```sh
$ aws ec2 authorize-security-group-ingress --group-id sg-0ec04f86bf2557add --protocol tcp --port 3306 --source-group sg-0f39fa6da9f0f9242
```
##  Creating SSH key pair

Create a key pair and save it to a file using the below command.  Also, update the permission of the file.
```sh
$ aws ec2 create-key-pair --key-name ProjectAKey.pem --query 'KeyMaterial' --output text > ProjectAKey.pem
$ chmod 400 ProjectAKey.pem
```

## Enabling public DNS hostname for the VPC 
To assign public DNS hostnames to instances with public IP addresses.
```sh
$ aws ec2 modify-vpc-attribute --vpc-id vpc-03085adf5b5ffa75e --enable-dns-hostnames "{\"Value\":true}"
```
We can verify it using describe-vpc-attribute command. 
```sh
$ aws ec2 describe-vpc-attribute --vpc-id vpc-03085adf5b5ffa75e --attribute enableDnsHostnames
{
"VpcId":"vpc-03085adf5b5ffa75e",
"EnableDnsHostnames": {
"Value":true
}
}
```

## Launching instances

Launch the bastion server with the required security group, keypair, subnet, and AMI (Here, I've used Amazon Linux AMI)
```sh
$ aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name ProjectAKey.pem --security-group-ids sg-0b7b6e9e9ea4674c4 --subnet-id subnet-02dc66170b1b16f79
```
```sh
$ aws ec2 describe-instances --instance-id i-098c6a55353a99278
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    "AmiLaunchIndex": 0,
                    "ImageId": "ami-0beaa649c482330f7",
                    "InstanceId": "i-098c6a55353a99278",
                    "InstanceType": "t2.micro",
                    "KeyName": "ProjectAKey.pem",
                    "LaunchTime": "2022-12-10T10:34:35.000Z",
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "Placement": {
                        "AvailabilityZone": "us-east-2a",
                        "GroupName": "",
                        "Tenancy": "default"
                    },
                    "PrivateDnsName": "ip-172-16-34-26.us-east-2.compute.internal",
                    "PrivateIpAddress": "172.16.34.26",
                    "ProductCodes": [],
                    "PublicDnsName": "",
                    "PublicIpAddress": "18.119.115.197",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "StateTransitionReason": "",
                    "SubnetId": "subnet-02dc66170b1b16f79",
                    "VpcId": "vpc-03085adf5b5ffa75e",
                    "Architecture": "x86_64",
                    "BlockDeviceMappings": [
                        {
                            "DeviceName": "/dev/xvda",
                            "Ebs": {
                                "AttachTime": "2022-12-09T18:17:20.000Z",
                                "DeleteOnTermination": true,
                                "Status": "attached",
                                "VolumeId": "vol-07c57cf322ec09438"
                            }
                        }
                    ],
                    "ClientToken": "faca603f-9b86-4611-b55f-f03722f44bcc",
                    "EbsOptimized": false,
                    "EnaSupport": true,
                    "Hypervisor": "xen",
                    "NetworkInterfaces": [
                        {
                            "Association": {
                                "IpOwnerId": "amazon",
                                "PublicDnsName": "",
                                "PublicIp": "18.119.115.197"
                            },
                            "Attachment": {
                                "AttachTime": "2022-12-09T18:17:19.000Z",
                                "AttachmentId": "eni-attach-0223b8f272d3ef5ed",
                                "DeleteOnTermination": true,
                                "DeviceIndex": 0,
                                "Status": "attached"
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupName": "ProjectA-bastion",
                                    "GroupId": "sg-0b7b6e9e9ea4674c4"
                                }
                            ],
                            "Ipv6Addresses": [],
                            "MacAddress": "02:b4:74:44:b5:78",
                            "NetworkInterfaceId": "eni-0e0033ca288adbdc1",
                            "OwnerId": "257637015312",
                            "PrivateIpAddress": "172.16.34.26",
                            "PrivateIpAddresses": [
                                {
                                    "Association": {
                                        "IpOwnerId": "amazon",
                                        "PublicDnsName": "",
                                        "PublicIp": "18.119.115.197"
                                    },
                                    "Primary": true,
                                    "PrivateIpAddress": "172.16.34.26"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-02dc66170b1b16f79",
                            "VpcId": "vpc-03085adf5b5ffa75e",
                            "InterfaceType": "interface"
                        }
                    ],
                    "RootDeviceName": "/dev/xvda",
                    "RootDeviceType": "ebs",
                    "SecurityGroups": [
                        {
                            "GroupName": "ProjectA-bastion",
                            "GroupId": "sg-0b7b6e9e9ea4674c4"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Name",
                            "Value": "ProjectA-bastion"
                        }
                    ],
                    "VirtualizationType": "hvm",
                    "CpuOptions": {
                        "CoreCount": 1,
                        "ThreadsPerCore": 1
                    },
                    "CapacityReservationSpecification": {
                        "CapacityReservationPreference": "open"
                    },
                    "HibernationOptions": {
                        "Configured": false
                    },
                    "MetadataOptions": {
                        "State": "applied",
                        "HttpTokens": "optional",
                        "HttpPutResponseHopLimit": 1,
                        "HttpEndpoint": "enabled"
                    }
                }
            ],
            "OwnerId": "257637015312",
            "ReservationId": "r-048de335d541c6184"
        }
    ]
}
```

Launch the frontend webserver with the required security group, keypair, subnet, and AMI.
```sh
$ aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name ProjectAKey.pem --security-group-ids sg-0f39fa6da9f0f9242 --subnet-id subnet-017aa5250fa2c5b82
```
```sh
$ aws ec2 describe-instances --instance-id i-0f569f94df6c9c5ba
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    "AmiLaunchIndex": 0,
                    "ImageId": "ami-0beaa649c482330f7",
                    "InstanceId": "i-0f569f94df6c9c5ba",
                    "InstanceType": "t2.micro",
                    "KeyName": "ProjectAKey.pem",
                    "LaunchTime": "2022-12-10T10:34:35.000Z",
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "Placement": {
                        "AvailabilityZone": "us-east-2b",
                        "GroupName": "",
                        "Tenancy": "default"
                    },
                    "PrivateDnsName": "ip-172-16-105-36.us-east-2.compute.internal",
                    "PrivateIpAddress": "172.16.105.36",
                    "ProductCodes": [],
                    "PublicDnsName": "",
                    "PublicIpAddress": "18.218.158.63",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "StateTransitionReason": "",
                    "SubnetId": "subnet-017aa5250fa2c5b82",
                    "VpcId": "vpc-03085adf5b5ffa75e",
                    "Architecture": "x86_64",
                    "BlockDeviceMappings": [
                        {
                            "DeviceName": "/dev/xvda",
                            "Ebs": {
                                "AttachTime": "2022-12-09T18:22:42.000Z",
                                "DeleteOnTermination": true,
                                "Status": "attached",
                                "VolumeId": "vol-0013d6bf25f1133ad"
                            }
                        }
                    ],
                    "ClientToken": "d4ead89e-6f0c-4032-b0ad-783fb743fe20",
                    "EbsOptimized": false,
                    "EnaSupport": true,
                    "Hypervisor": "xen",
                    "NetworkInterfaces": [
                        {
                            "Association": {
                                "IpOwnerId": "amazon",
                                "PublicDnsName": "",
                                "PublicIp": "18.218.158.63"
                            },
                            "Attachment": {
                                "AttachTime": "2022-12-09T18:22:42.000Z",
                                "AttachmentId": "eni-attach-04f3e7cc42b6d73f2",
                                "DeleteOnTermination": true,
                                "DeviceIndex": 0,
                                "Status": "attached"
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupName": "ProjectA-webserver",
                                    "GroupId": "sg-0f39fa6da9f0f9242"
                                }
                            ],
                            "Ipv6Addresses": [],
                            "MacAddress": "06:8f:27:39:7a:80",
                            "NetworkInterfaceId": "eni-0f0e4f001012a69a0",
                            "OwnerId": "257637015312",
                            "PrivateIpAddress": "172.16.105.36",
                            "PrivateIpAddresses": [
                                {
                                    "Association": {
                                        "IpOwnerId": "amazon",
                                        "PublicDnsName": "",
                                        "PublicIp": "18.218.158.63"
                                    },
                                    "Primary": true,
                                    "PrivateIpAddress": "172.16.105.36"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-017aa5250fa2c5b82",
                            "VpcId": "vpc-03085adf5b5ffa75e",
                            "InterfaceType": "interface"
                        }
                    ],
                    "RootDeviceName": "/dev/xvda",
                    "RootDeviceType": "ebs",
                    "SecurityGroups": [
                        {
                            "GroupName": "ProjectA-webserver",
                            "GroupId": "sg-0f39fa6da9f0f9242"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Name",
                            "Value": "ProjectA-webserver"
                        }
                    ],
                    "VirtualizationType": "hvm",
                    "CpuOptions": {
                        "CoreCount": 1,
                        "ThreadsPerCore": 1
                    },
                    "CapacityReservationSpecification": {
                        "CapacityReservationPreference": "open"
                    },
                    "HibernationOptions": {
                        "Configured": false
                    },
                    "MetadataOptions": {
                        "State": "applied",
                        "HttpTokens": "optional",
                        "HttpPutResponseHopLimit": 1,
                        "HttpEndpoint": "enabled"
                    }
                }
            ],
            "OwnerId": "257637015312",
            "ReservationId": "r-0ad1aaa8ae4f3bfeb"
        }
    ]
}
```

Launch the backend database server with the required security group, keypair, subnet, and AMI.
```sh
$ aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name ProjectAKey.pem --security-group-ids sg-0ec04f86bf2557add --subnet-id subnet-03956c03687b5d523
```
```sh
$ aws ec2 describe-instances --instance-id i-0dc7f42d36a489592
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    "AmiLaunchIndex": 0,
                    "ImageId": "ami-0beaa649c482330f7",
                    "InstanceId": "i-0dc7f42d36a489592",
                    "InstanceType": "t2.micro",
                    "KeyName": "ProjectAKey.pem",
                    "LaunchTime": "2022-12-10T10:34:35.000Z",
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "Placement": {
                        "AvailabilityZone": "us-east-2b",
                        "GroupName": "",
                        "Tenancy": "default"
                    },
                    "PrivateDnsName": "ip-172-16-150-110.us-east-2.compute.internal",
                    "PrivateIpAddress": "172.16.150.110",
                    "ProductCodes": [],
                    "PublicDnsName": "",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "StateTransitionReason": "",
                    "SubnetId": "subnet-03956c03687b5d523",
                    "VpcId": "vpc-03085adf5b5ffa75e",
                    "Architecture": "x86_64",
                    "BlockDeviceMappings": [
                        {
                            "DeviceName": "/dev/xvda",
                            "Ebs": {
                                "AttachTime": "2022-12-09T18:23:23.000Z",
                                "DeleteOnTermination": true,
                                "Status": "attached",
                                "VolumeId": "vol-069a471de10d367aa"
                            }
                        }
                    ],
                    "ClientToken": "db5a5b22-b3e6-4412-9dc7-f6d9d3e17e58",
                    "EbsOptimized": false,
                    "EnaSupport": true,
                    "Hypervisor": "xen",
                    "NetworkInterfaces": [
                        {
                            "Attachment": {
                                "AttachTime": "2022-12-09T18:23:22.000Z",
                                "AttachmentId": "eni-attach-0abe838734239da32",
                                "DeleteOnTermination": true,
                                "DeviceIndex": 0,
                                "Status": "attached"
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupName": "ProjectA-dbserver",
                                    "GroupId": "sg-0ec04f86bf2557add"
                                }
                            ],
                            "Ipv6Addresses": [],
                            "MacAddress": "06:fa:be:1c:7c:c0",
                            "NetworkInterfaceId": "eni-0b67337b40b201004",
                            "OwnerId": "257637015312",
                            "PrivateIpAddress": "172.16.150.110",
                            "PrivateIpAddresses": [
                                {
                                    "Primary": true,
                                    "PrivateIpAddress": "172.16.150.110"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-03956c03687b5d523",
                            "VpcId": "vpc-03085adf5b5ffa75e",
                            "InterfaceType": "interface"
                        }
                    ],
                    "RootDeviceName": "/dev/xvda",
                    "RootDeviceType": "ebs",
                    "SecurityGroups": [
                        {
                            "GroupName": "ProjectA-dbserver",
                            "GroupId": "sg-0ec04f86bf2557add"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Name",
                            "Value": "ProjectA-dbserver"
                        }
                    ],
                    "VirtualizationType": "hvm",
                    "CpuOptions": {
                        "CoreCount": 1,
                        "ThreadsPerCore": 1
                    },
                    "CapacityReservationSpecification": {
                        "CapacityReservationPreference": "open"
                    },
                    "HibernationOptions": {
                        "Configured": false
                    },
                    "MetadataOptions": {
                        "State": "applied",
                        "HttpTokens": "optional",
                        "HttpPutResponseHopLimit": 1,
                        "HttpEndpoint": "enabled"
                    }
                }
            ],
            "OwnerId": "257637015312",
            "ReservationId": "r-0ec108dd4eb6ed8f5"
        }
    ]
}
```

Adding tags to the instances to identify them quickly. 
```sh
$ aws ec2 create-tags --resources i-098c6a55353a99278 --tags Key=Name,Value=ProjectA-bastion
$ aws ec2 create-tags --resources i-0dc7f42d36a489592 --tags Key=Name,Value=ProjectA-dbserver
$ aws ec2 create-tags --resources i-0f569f94df6c9c5ba --tags Key=Name,Value=ProjectA-webserver
```

## Creating a Wordpress website

a.  SSH to the bastion server using its public IP and update its hostname.
```sh
$ ssh -i ProjectAKey.pem ec2-user@18.119.132.161
The authenticity of host '18.119.115.197 (18.119.115.197)' can't be established.
ECDSA key fingerprint is SHA256:e/RbJ58SDmtajPqvhXo3Th6sWvcrAhct4EDhRvTw4wU.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '18.119.115.197' (ECDSA) to the list of known hosts.
Last login: Fri Dec  9 18:30:32 2022 from 117.213.84.118

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-34-26 ~]$ sudo hostnamectl set-hostname bastion-ip-172-16-34-26.us-east-2.compute.internal
```

b. Create/copy the SSH key file ProjectAKey.pem and set the file permission to 400
```sh
[ec2-user@ip-172-16-34-26 ~]$ vi ProjectAKey.pem
[ec2-user@ip-172-16-34-26 ~]$ chmod 400 ProjectAKey.pem
```
c. SSH to the database server using its private IP and update the hostname.
```sh
[ec2-user@bastion-ip-172-16-34-26 ~]$ ssh -i ProjectAKey.pem ec2-user@172.16.150.110
Last login: Fri Dec  9 19:14:08 2022 from 172.16.34.26

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-150-110 ~]$ sudo hostnamectl set-hostname db-ip-172-16-150-110.us-east-2.compute.internal
```

#### On the backend server

> Installed, started, and enabled MariaDB. 
```sh
sudo yum install mariadb-server -y
sudo systemctl restart mariadb.service
sudo systemctl enable mariadb.service
Set database root password.
[ec2-user@db-ip-172-16-150-110 ~]$ mysql_secure_installation
Create a database and user, and grant the privileges.
[ec2-user@db-ip-172-16-150-110 ~]$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 5.5.68-MariaDB MariaDB Server
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
MariaDB [(none)]> create database wpdb;
Query OK, 1 row affected (0.00 sec)                                                 
MariaDB [(none)]> create user 'wpdbuser'@'%' identified by '***********';
Query OK, 0 row affected (0.00 sec)                                                 
MariaDB [(none)]> grant all privileges on wpdb.* to 'wpdbuser'@'%';
Query OK, 0 row affected (0.00 sec)                                                 
MariaDB [(none)]> flush privileges;                                                  
Query OK, 0 row affected (0.00 sec)                                              
```
d. SSH to the frontend server using its private IP and update the hostname. 
```sh
[ec2-user@bastion-ip-172-16-34-26 ~]$ ssh -i ProjectAKey.pem ec2-user@172.16.105.36
Last login: Fri Dec  9 19:21:46 2022 from 172.16.34.26

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-105-36 ~]$ sudo hostnamectl set-hostname webserver-ip-172-16-105-36.us-east-2.compute.internal
```

#### On the frontend server
>Install Apache and PHP.
>Download Wordpress, extract it, and move the contents to the document root.
>Update the database credentials(database name, username, password, and private IP of the backend database server) in the wp-config file. 
```sh
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo yum install httpd -y
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo amazon-linux-extras install php7.4
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo systemctl restart httpd.service
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo systemctl enable httpd.service
[ec2-user@webserver-ip-172-16-105-36 ~]$ wget https://wordpress.org/latest.zip
[ec2-user@webserver-ip-172-16-105-36 ~]$ unzip latest.zip
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo mv wordpress/* /var/www/html/
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo mv /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
[ec2-user@webserver-ip-172-16-105-36 ~]$ sudo vi /var/www/html/wp-config.php
```
Now we can access the website using the public IP, public DNS name, or domain name(if a domain is pointing to the server IP)  to complete the Wordpress installation and design the website.

## Conclusion

This is a simple documentation on how to use AWS CLI to build an AWS VPC along with private/public Subnet and Network Gateways for the VPC. We will create a VPC with three Subnets: 1 Private and 2 Public, a NAT Gateway, an Internet Gateway, and two Route Tables. We will launch a bastion instance on the first public subnet, a frontend webserver on the second public subnet, and a database server on the private subnet.
