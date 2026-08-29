- VPC : isolated environment in an account where all the other components will be placed within. Its its own network so must have a network range based on CIDAR notation.
- CIDR : 10.0.0.0/8 will be giving access to 256^3 ip addresses and 10.0.0.0/16 will give 256^2 ip addresses
- Subset : its a network within the VPC that gets a part of the CIDR allocation ip addresses like 10.0.1.0 - 10.0.1.255
- Route table : is a set of rules that say: "if traffic is destined for X network, send it to Y gateway/endpoint." It's a routing decision maker, not a registry.
- Public vs private subnet : one can access or be accessible by the internet whiel the other stays isolated. Public has access to IGW
- IGW : internet gateway in a VPC is taht connects the vpc components to the internet
- NAT : network address translation is used inside a private subnet which cant access internet but in case it does, like to upgrade its version then it can do just that with restrictions and does not allow incoming traffic from the internet
- AZ : availability zone is in a region, can be many, for disaster recovery, like datacentres across us-east-1

## Provisioning
1 VPC (10.0.0.0/16)
2 public subnets (one in each AZ)
10.0.1.0/24 in us-east-1a
10.0.2.0/24 in us-east-1b
2 private subnets (one in each AZ)
10.0.10.0/24 in us-east-1a
10.0.11.0/24 in us-east-1b
1 Internet Gateway attached to VPC
Route table for public subnets: route 0.0.0.0/0 → IGW
Route table for private subnets: route 0.0.0.0/0 → NAT Gateway (in a public subnet)

## Breaking 
Test 1: Launch EC2 in private subnet, try to reach internet
Test 2: Kill one AZ's resources, observe isolation