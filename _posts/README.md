## Infrastructure deployment on AWS with Terraform

### Installation of Terraform

→ Verify system is up to date and prerequisites are installed

→ Add Hashicorp GPG key: ```curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -```

→ Add official Hashicorp Linux repository: ```sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"```

→ Install Terraform CLI: ```sudo apt-get update && sudo apt-get install terraform```

→ Verify the installation: ```terraform -help```

→ Enable tab completion: ```terraform -install-autocomplete```

### Installation of AWS

For the latest version of the AWS CLI, use the following command block: 
```bash
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
```

### Quick start

Create a file to define your infrastructure :

    $ touch example.tf


Open the file in your text editor, paste in the configuration below, and save the file.
```django
provider "aws" {
  profile = "default"
  region  = "eu-west-1"
}

resource "aws_instance" "example" {
  ami           = ""
  instance_type = ""
 
  tags = {
    Name = "ExampleAppServerInstance"
  }
}
```
To use aws cli, you need to set AWS environment variables as below :
```bash
$ export AWS_ACCESS_KEY_ID="anaccesskey"
$ export AWS_SECRET_ACCESS_KEY="asecretkey"
$ export AWS_DEFAULT_REGION="region"
```

The ```terraform init``` command initialize the directory in order to create a new configuration or check out an existing configuration

The ```terraform plan``` command lets you preview the changes that Terraform plans to make to your infrastructure

The ```terraform apply``` command apply the configuration.

The ```terraform destroy``` command destroy all remote objects managed by a particular Terraform configuration.

### Create Network Architecture
#### 1- VPC (Virtual Private Cloud)

VPC is a service that lets you launch AWS resources in a logically isolated virtual network that you define. You can declare a VPC as below with the resource [aws_vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc) :
```django
resource "aws_vpc" "main" {
  cidr_block = "NETWORK_IP/MASK"
  tags = {
    Name = "VPC-name"
  }
}
```

#### 2- Subnets

In order to create private subnet, you need to declare a new resource named [aws_subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
```django
resource "aws_subnet" "subnet_1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "SUBNET_IP/MASK"
  availability_zone = "eu-west-1a"
  tags = {
    Name = "subnet_1"
  }
}
```

#### 3- NAT

**First** you need to create a VPC [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) that allows communication between your VPC and the internet. 

Terraform resource: [aws_internet_gateway]()

```django
resource "aws_internet_gateway" "internet_gt" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "Internet Gateway"
  }  
}
```
**Secondly**, you need need to create an [Elastic IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)  address that is a public IPv4 address, which is reachable from the internet.

Terraform ressource: [aws_eip](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)

```django
resource "aws_eip" "one" {
  vpc = true
  depends_on = [aws_internet_gateway.internet_gt]
}
```

**Thirdly**, you need to create an [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) which is a NAT service. NAT gateway is used to connect a private subnet to services outside your VPC but external services cannot initiate a connection with those instances. 

Terraform resource: [aws_nat_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway)

```django
resource "aws_nat_gateway" "example" {
  allocation_id = aws_eip.one.id
  subnet_id = aws_subnet.public_subnet_nat.id
  tags = {
    Name = "Nat Gateway"
  }
  depends_on = [aws_internet_gateway.internet_gt]
}
```

#### 4- Route Table

In order to communicate from Internet to an instance in the VPC, you need to create **first** a [Route Table](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) : 

Terraform resource: [aws_route_table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table)

```django
resource "aws_route_table" "all_traffic_to_natGt" {
  vpc_id = aws_vpc.main.id  
  route {
    cidr_block = "0.0.0.0/0" # All traffic
    gateway_id = aws_nat_gateway.example.id
  }
 tags = {
    Name = "example"
  }
}
```

**Secondly**, you have to define the route association which that links a subnet to the route table :

Terraform resource : [aws_route_table_association](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association)

```django
resource "aws_route_table_association" "nat-gt_to_internet-gt" {
  subnet_id      = aws_subnet.public_subnet_nat.id
  route_table_id = aws_route_table.to_internet-gt.id
}
```

#### 5- Bastion with SSH connection

In order to be able to connect to the bastion with an SSH connection, you need first to create a security group for the bastion which ingress traffic of internet on the SSH port:
Terraform resource : [aws_security_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)

```django
resource "aws_security_group" "bastion_sg" {
  name   = "bastion_sg"
  vpc_id = aws_vpc.main.id

  ingress {
    protocol    = "tcp"
    from_port   = 22
    to_port     = 22
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
  protocol    = -1
  from_port   = 0 
  to_port     = 0 
  cidr_blocks = ["0.0.0.0/0"]
  }
}
```
Next, you need to associate your instance with this security group :

```django
resource "aws_instance" "bastion" {
  ...
  vpc_security_group_ids = [aws_security_group.bastion_sg.id]
}
```

Secondly, you have to create an SSH key and copy the private key in a ressource named [aws_key_pair](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair)

```django
resource "aws_key_pair" "foo" {
  key_name   = "key-name"
  public_key = "ssh-rsa ..."
}
```

Next, you have to link this key_pair to the bastion instance :

```django
resource "aws_instance" "bastion" {
  ...
  key_name = "key-name"
}
```

And now, you can connect to the bastion with ssh command :

```bash
ssh -i private-key ec2-user@X.X.X.X
```
 
Other usefuls ssh commands to connect to an instance by the bastion :

```bash
eval `ssh-agent -s`
ssh-add private-key-bastion
ssh -o ForwardAgent=yes -i private-key-instance -J ec2-user@X.X.X.X ec2-user@X.X.X.X
```

#### 6- Load Balancer

There are main steps in order to create a load Balancer :
- Create publics subnets 
- Create a security group for these subnets
- Create a load balance resource
- Create a on the load balancer a listener 
- Indicate targets a on the load balancer 

You have to create publics subnets to link your privates subnets on which are located your web application. 