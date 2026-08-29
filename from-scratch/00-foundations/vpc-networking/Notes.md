## Command
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --availability-zone us-east-2a

## History
- why a networking service like a VPC is managed under the aws ec2 command prefix?

The reason aws ec2 create-vpc is used is due to AWS's historical architecture. When AWS launched in 2006, Amazon EC2 (Elastic Compute Cloud) was its primary core service.

At that time, all virtual machines ran in a massive, shared flat network called EC2-Classic.When AWS later invented the Virtual Private Cloud (VPC) to give users isolated, private networks, they built it directly on top of the existing EC2 infrastructure.

Instead of creating a brand-new top-level CLI service (like aws vpc), AWS kept all networking components tightly coupled to the compute resources that live inside them

Because of this legacy design, almost all core networking components are nested under the aws ec2 namespace:
aws ec2 create-subnetaws
ec2 create-internet-gatewayaws
ec2 create-route-tableaws
ec2 create-security-group
You can think of aws ec2 in the CLI not just as "virtual servers," but as the broader compute and core networking subsystem of AWS.

- why IGW cant be directly created in vpc? why first create then attach?

IGWs are attached to a VPC — they're standalone resources that theoretically could be attached to multiple VPCs (though that's rare and not recommended). The API reflects this: create it first, then decide which VPC to attach it to.

It's not a technical limitation—it's how AWS designed the API to match the conceptual relationship:

## List all VPCs
aws ec2 describe-vpcs --profile floci
aws ec2 describe-vpcs --profile floci --output table

## List all subnets
aws ec2 describe-subnets --profile floci

## List all Internet Gateways
aws ec2 describe-internet-gateways --profile floci

## List all route tables
aws ec2 describe-route-tables --profile floci

## List all NAT Gateways
aws ec2 describe-nat-gateways --profile floci

## List all EC2 instances
aws ec2 describe-instances --profile floci

## Resources
https://docs.aws.amazon.com/cli/latest/reference/ec2/create-vpc.html
https://docs.aws.amazon.com/cli/latest/reference/ec2/create-subnet.html