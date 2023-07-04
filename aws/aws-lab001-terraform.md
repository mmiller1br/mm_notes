
## Diagram

![aws-lab001](https://github.com/mmiller1br/mm_notes/assets/32887571/ecf02458-1b79-4d55-b787-15d121c5d512)


## Terraform .tf file

``` tf
provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
}

# Create VPC1
resource "aws_vpc" "myvpc1" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "myvpc1-terraform"
  }
}

# Create VPC2
resource "aws_vpc" "myvpc2" {
  cidr_block = "192.168.0.0/16"
  tags = {
    Name = "myvpc2-terraform"
  }
}

# Create the Subnet on VPC1
resource "aws_subnet" "mysubnet1" {
  vpc_id     = aws_vpc.myvpc1.id
  cidr_block = "10.0.0.0/24"
  tags = {
    Name = "mysubnet1-terraform"
  }
}

# Create the Subnet on VPC2
resource "aws_subnet" "mysubnet2" {
  vpc_id     = aws_vpc.myvpc2.id
  cidr_block = "192.168.0.0/24"
  tags = {
    Name = "mysubnet2-terraform"
  }
}

# Create the IGW1 for VPC1
resource "aws_internet_gateway" "myigw1" {
  vpc_id = aws_vpc.myvpc1.id
  tags = {
    Name = "myigw1-terraform"
  }
}

# Create the IGW2 for VPC2
resource "aws_internet_gateway" "myigw2" {
  vpc_id = aws_vpc.myvpc2.id
  tags = {
    Name = "myigw2-terraform"
  }
}

# Create the VCP Peering to Route VPC1 and VPC2
resource "aws_vpc_peering_connection" "myvpcpeering" {
  peer_vpc_id   = aws_vpc.myvpc1.id
  vpc_id        = aws_vpc.myvpc2.id
  auto_accept   = true

  tags = {
    Name = "myvpcpeering-terraform"
  }
}

# Create Route Table for VPC1
resource "aws_route_table" "myrt1" {
  vpc_id = aws_vpc.myvpc1.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myigw1.id
  }
  route {
    cidr_block = "192.168.0.0/24"
    gateway_id = aws_vpc_peering_connection.myvpcpeering.id
  }
  tags = {
    Name = "myrt1-terraform"
  }
}

# Create Route Table for VPC2
resource "aws_route_table" "myrt2" {
  vpc_id = aws_vpc.myvpc2.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myigw2.id
  }
  route {
    cidr_block = "10.0.0.0/24"
    gateway_id = aws_vpc_peering_connection.myvpcpeering.id
  }
  tags = {
    Name = "myrt2-terraform"
  }
}

# Associate Route Table 1 + Subnet1
resource "aws_route_table_association" "assoc1" {
  subnet_id      = aws_subnet.mysubnet1.id
  route_table_id = aws_route_table.myrt1.id
}

# Associate Route Table 2 + Subnet2
resource "aws_route_table_association" "assoc2" {
  subnet_id      = aws_subnet.mysubnet2.id
  route_table_id = aws_route_table.myrt2.id
}

# Create SecurityGroup for VPC1
resource "aws_security_group" "mysecgrp1" {
  name        = "security group VPC1"
  description = "Allow SSH and ICMP inbound traffic"
  vpc_id      = aws_vpc.myvpc1.id

  ingress {
    from_port        = -1
    to_port          = -1
    description      = "allow ICMP"
    protocol         = "icmp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    from_port        = 22
    to_port          = 22
    description      = "allow SSH"
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "mysecgrp1"
  }
}

# Create SecurityGroup for VPC2
resource "aws_security_group" "mysecgrp2" {
  name        = "security group VPC2"
  description = "Allow SSH and ICMP inbound traffic"
  vpc_id      = aws_vpc.myvpc2.id

  ingress {
    from_port        = -1
    to_port          = -1
    description      = "allow ICMP"
    protocol         = "icmp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    from_port        = 22
    to_port          = 22
    description      = "allow SSH"
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "mysecgrp2"
  }
}

# Create EC2 instances
resource "aws_instance" "myvpc1-ec2" {
  ami                         = "ami-022e1a32d3f742bd8"
  instance_type               = "t2.micro"

  subnet_id                   = aws_subnet.mysubnet1.id
  vpc_security_group_ids      = [aws_security_group.mysecgrp1.id]
  associate_public_ip_address = true

  key_name                    = "mykey001"

  tags = {
    Name = "myvpc1-ec2"
  }
}

resource "aws_instance" "myvpc2-ec2" {
  ami                         = "ami-022e1a32d3f742bd8"
  instance_type               = "t2.micro"

  subnet_id                   = aws_subnet.mysubnet2.id
  vpc_security_group_ids      = [aws_security_group.mysecgrp2.id]
  associate_public_ip_address = true

  key_name                    = "mykey001"

  tags = {
    Name = "myvpc2-ec2"
  }
}

output "vpc1_id" {
  value = aws_vpc.myvpc1.id
}

output "vpc2_id" {
  value = aws_vpc.myvpc2.id
}
```

to create your IaC, run the following code:

```
sudo terraform apply
```

and to destroy / delete everything, run this command:

```
sudo terraform destroy
```

