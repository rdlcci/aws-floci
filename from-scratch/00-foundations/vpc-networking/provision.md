# Aim
- Learn about vpc how to create its basic components
    - create vpc
    - Create 2 public subnets (10.0.1.0/24, 10.0.2.0/24 in different AZs)
    - Create 2 private subnets (10.0.10.0/24, 10.0.11.0/24 in different AZs)
    - Create and attach Internet Gateway
    - Create NAT Gateway
    - Create and configure route tables

## create a vpc with 256^2 in us east 2a
- aws ec2 create-vpc --cidr-block 10.0.0.0/16 --profile floci
```json
{                                                                                          
    "Vpc": {
        "OwnerId": "000000000000",
        "InstanceTenancy": "default",
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-3f3e579f",
                "CidrBlock": "10.0.0.0/16",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false,
        "Tags": [],                                                                        
        "VpcId": "vpc-0cbb1a29",                                                           
        "State": "available",                                                              
        "CidrBlock": "10.0.0.0/16",                                                        
        "DhcpOptionsId": "dopt-default"                                                    
    }                                                                                      
}  
```

## create 2 subnet in 2 AZs with different cidr block (will become private/public based on IGW and route table)
- aws ec2 create-subnet
    --vpc-id vpc-fe87d91c
    --availability-zone us-east-1a
    --cidr-block 10.0.1.0/24
    --profile floci
```json
{                                                                                          
    "Subnet": {
        "AvailabilityZoneId": "us-east-1-az1",
        "MapCustomerOwnedIpOnLaunch": false,
        "OwnerId": "000000000000",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "Tags": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:000000000000:subnet/subnet-238269b1",
        "EnableDns64": false,                                                              
        "SubnetId": "subnet-238269b1",                                                     
        "State": "available",                                                              
        "VpcId": "vpc-0cbb1a29",                                                           
        "CidrBlock": "10.0.1.0/24",                                                        
        "AvailableIpAddressCount": 251,                                                    
        "AvailabilityZone": "us-east-1a",                                                  
        "DefaultForAz": false,                                                             
        "MapPublicIpOnLaunch": false                                                       
    }
}
```  

- aws ec2 create-subnet --vpc-id vpc-fe87d91c --availability-zone us-east-1b --cidr-block 10.0.2.0/24 --profile floci
```json
{                                                                                          
    "Subnet": {
        "AvailabilityZoneId": "us-east-1-az1",
        "MapCustomerOwnedIpOnLaunch": false,
        "OwnerId": "000000000000",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "Tags": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:000000000000:subnet/subnet-ea2e5558",
        "EnableDns64": false,                                                              
        "SubnetId": "subnet-ea2e5558",                                                     
        "State": "available",                                                              
        "VpcId": "vpc-0cbb1a29",                                                           
        "CidrBlock": "10.0.2.0/24",                                                        
        "AvailableIpAddressCount": 251,                                                    
        "AvailabilityZone": "us-east-1b",                                                  
        "DefaultForAz": false,                                                             
        "MapPublicIpOnLaunch": false                                                       
    }                                                                                      
}  
```

## Create 2 more subnets (10.0.10.0/24, 10.0.11.0/24 in different AZs), will become private later

- aws ec2 create-subnet
    --vpc-id vpc-fe87d91c
    --availability-zone us-east-1a
    --cidr-block 10.0.10.0/24
    --profile floci &&
        aws ec2 create-subnet
    --vpc-id vpc-fe87d91c
    --availability-zone us-east-1a
    --cidr-block 10.0.11.0/24
    --profile floci
```json
{                                                                                          
    "Subnet": {
        "AvailabilityZoneId": "us-east-1-az1",
        "MapCustomerOwnedIpOnLaunch": false,
        "OwnerId": "000000000000",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "Tags": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:000000000000:subnet/subnet-c038d953",
        "EnableDns64": false,                                                              
        "SubnetId": "subnet-c038d953",                                                     
        "State": "available",                                                              
        "VpcId": "vpc-0cbb1a29",                                                           
        "CidrBlock": "10.0.10.0/24",                                                       
        "AvailableIpAddressCount": 251,                                                    
        "AvailabilityZone": "us-east-1a",                                                  
        "DefaultForAz": false,                                                             
        "MapPublicIpOnLaunch": false                                                       
    }                                                                                      
}                                                                                          
                                                                                           
{                                                                                          
    "Subnet": {
        "AvailabilityZoneId": "us-east-1-az1",
        "MapCustomerOwnedIpOnLaunch": false,
        "OwnerId": "000000000000",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "Tags": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:000000000000:subnet/subnet-279da8f7",
        "EnableDns64": false,                                                              
        "SubnetId": "subnet-279da8f7",                                                     
        "State": "available",                                                              
        "VpcId": "vpc-0cbb1a29",                                                           
        "CidrBlock": "10.0.11.0/24",                                                       
        "AvailableIpAddressCount": 251,                                                    
        "AvailabilityZone": "us-east-1a",                                                  
        "DefaultForAz": false,                                                             
        "MapPublicIpOnLaunch": false                                                       
    }      
}                                                                                    
```

## So far
All 4 subnets are technically "private" (they can't reach the internet), but they're not configured to be either public or private yet.

Public subnet = a subnet whose route table has a route to an Internet Gateway
Private subnet = a subnet whose route table does NOT have a route to an IGW

## adding internet gateway (first create as a different entity then attch to the vpc)
- aws ec2 create-internet-gateway --profile floci
```json
{                                                                                          
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-2f53b7bf",
        "OwnerId": "000000000000",
        "Tags": []
    }
}
```
- aws ec2 attach-internet-gateway 
    --internet-gateway-id igw-2f53b7bf
    --vpc-id vpc-0cbb1a29
    --profile floci
- this does nothing, so to check if IGW was created
    - aws ec2 describe-internet-gateways --profile floci 
```json
{                                                                                          
    "InternetGateways": [
        {
            "Attachments": [
                {
                    "State": "available",
                    "VpcId": "vpc-0cbb1a29"
                }
            ],
            "InternetGatewayId": "igw-68d12630",                                           
            "OwnerId": "000000000000",                                                     
            "Tags": []                                                                     
        },                                                                                 
        {                                                                                  
            "Attachments": [                                                               
                {                                                                          
                    "State": "available",                                                  
                    "VpcId": "vpc-default"                                                 
                }                                                                          
            ],                                                                             
            "InternetGatewayId": "igw-default",                                            
            "OwnerId": "000000000000",                                                     
            "Tags": []                                                                     
        }                                                                                  
    ]                                                                                      
}                      
```

## adding nat gateway (TODO)
- NAT Gateway must be placed inside a specific subnet (it's tightly bound to that subnet), so you use --subnet-id, not --vpc-id
- It needs two things one is subnet id for attachment and a allocation id (elastic ip) thats when its created
- aws ec2 allocate-address --profile floci
```json
{                                                                                                 
    "AllocationId": "eipalloc-69b011411dd3b003f",
    "Domain": "vpc",
    "PublicIp": "54.187.101.77"
}

```
- aws ec2 create-nat-gateway --subnet-id subnet-c038d953 --allocation-id eipalloc-69b011411dd3b003f --profile floci

```json
{                                                                                                 
    "NatGateway": {
        "CreateTime": "2026-08-31T05:30:06.412000+00:00",
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-69b011411dd3b003f"
            }
        ],
        "NatGatewayId": "nat-7b0a019b1bb324fd5",
        "State": "available",
        "SubnetId": "subnet-dcc7e75c",
        "VpcId": "vpc-fe87d91c",
        "Tags": [],
        "ConnectivityType": "public"
    }
}
```

The above assignment was incorrect so NAT has to be disassociated and attached to the right subnet (public)

- aws ec2 disassociate-address --allocation-id eipalloc-69b011411dd3b003f
- this will not work as the NAT is active so it has to be ressurected first
- aws ec2 delete-nat-gateway --nat-gateway-id nat-7b0a019b1bb324fd5 --profile floci

- aws ec2 create-nat-gateway --subnet-id subnet-6ad00064 --allocation-id eipalloc-69b011411dd3b003f --profile floci

```json
{                                                                                                 
    "NatGateway": {
        "CreateTime": "2026-08-31T05:49:03.510000+00:00",
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-69b011411dd3b003f"
            }
        ],
        "NatGatewayId": "nat-be7d0fb3d5df4e0ea",
        "State": "available",
        "SubnetId": "subnet-6ad00064",
        "VpcId": "vpc-fe87d91c",
        "Tags": [],
        "ConnectivityType": "public"
    }
}
```

## add route table for bith private and public
aws ec2 create-route-table --vpc-id vpc-fe87d91c --profile floci
This returns RouteTableId, e.g., rtb-xxxxx

Add route to IGW
aws ec2 create-route --route-table-id rtb-xxxxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0f51d7e0 --profile floci

Associate public subnets
aws ec2 associate-route-table --route-table-id rtb-xxxxx --subnet-id subnet-3896301c --profile floci
aws ec2 associate-route-table --route-table-id rtb-xxxxx --subnet-id subnet-ff1b7060 --profile floci

---

aws ec2 create-route-table --vpc-id vpc-fe87d91c --profile floci
This returns RouteTableId, e.g., rtb-yyyyy

Add route to NAT Gateway
aws ec2 create-route --route-table-id rtb-yyyyy --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-edc55fc0c826fe29e --profile floci

Associate private subnets
aws ec2 associate-route-table --route-table-id rtb-yyyyy --subnet-id subnet-dcc7e75c --profile floci
aws ec2 associate-route-table --route-table-id rtb-yyyyy --subnet-id subnet-6ad00064 --profile floci