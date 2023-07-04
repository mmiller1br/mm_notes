
This is a simple LAB on AWS (LAB001), but the idea is to demonstrate how to create it using Infrastructure as a Code (IaC) using 3 different approaches, or tools:
- CloudFormation (from AWS)
- Ansible
- Terraform

They don't have all the best practices when creating the code in terms of variables, structure, etc, but they have an excellent starting point to understand how they work and the similarities and differences.

Enjoy and fell free to copy it, use it, and send your comments to make it better.

Cheers.


## Diagram:

![aws-lab001](https://github.com/mmiller1br/mm_notes/assets/32887571/ecf02458-1b79-4d55-b787-15d121c5d512)



## AWS CloudFormation stack

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:
  VPC1:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: myvpc-1

  VPC2:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/16
      Tags:
        - Key: Name
          Value: myvpc-2

  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC1
      CidrBlock: 10.0.0.0/24
      Tags:
        - Key: Name
          Value: mypublic-subnet-1

  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC2
      CidrBlock: 192.168.0.0/24
      Tags:
        - Key: Name
          Value: mypublic-subnet-2

  InternetGateway1:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: my-ig1

  InternetGatewayAttachment1:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC1
      InternetGatewayId: !Ref InternetGateway1

  InternetGateway2:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: my-ig2

  InternetGatewayAttachment2:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC2
      InternetGatewayId: !Ref InternetGateway2

  VpcPeeringConnection1:
    Type: AWS::EC2::VPCPeeringConnection
    Properties:
      VpcId: !Ref VPC1
      PeerVpcId: !Ref VPC2
      Tags:
        - Key: Name
          Value: my-vcp-peering

  RouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC1
      Tags:
        - Key: Name
          Value: my-rt1

  Route1:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment1
    Properties:
      RouteTableId: !Ref RouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway1

  Route11:
    Type: AWS::EC2::Route
    DependsOn: VpcPeeringConnection1
    Properties:
      RouteTableId: !Ref RouteTable1
      DestinationCidrBlock: 192.168.0.0/24
      VpcPeeringConnectionId: !Ref VpcPeeringConnection1

  RouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC2
      Tags:
        - Key: Name
          Value: my-rt2

  Route2:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment2
    Properties:
      RouteTableId: !Ref RouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway2

  Route21:
    Type: AWS::EC2::Route
    DependsOn: VpcPeeringConnection1
    Properties:
      RouteTableId: !Ref RouteTable2
      DestinationCidrBlock: 10.0.0.0/24
      VpcPeeringConnectionId: !Ref VpcPeeringConnection1

  SubnetRouteTableAssociation1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet1
      RouteTableId: !Ref RouteTable1

  SubnetRouteTableAssociation2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet2
      RouteTableId: !Ref RouteTable2

  SecurityGroup1:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group for Subnet1
      VpcId: !Ref VPC1
      SecurityGroupIngress:
        - IpProtocol: icmp
          FromPort: -1
          ToPort: -1
          CidrIp: 192.168.0.0/16
        - IpProtocol: icmp
          FromPort: -1
          ToPort: -1
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: my-secgrp1

  SecurityGroup2:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group for Subnet2
      VpcId: !Ref VPC2
      SecurityGroupIngress:
        - IpProtocol: icmp
          FromPort: -1
          ToPort: -1
          CidrIp: 10.0.0.0/16
        - IpProtocol: icmp
          FromPort: -1
          ToPort: -1
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: my-secgrp2

  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-022e1a32d3f742bd8
      InstanceType: t2.micro
      KeyName: mykey001
      Tags:
        - Key: Name
          Value: my-vpc1-inst1
      NetworkInterfaces:
        - AssociatePublicIpAddress: true
          DeviceIndex: 0
          SubnetId: !Ref Subnet1
          GroupSet:
            - !Ref SecurityGroup1


  Instance2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-022e1a32d3f742bd8
      InstanceType: t2.micro
      KeyName: mykey001
      Tags:
        - Key: Name
          Value: my-vcp2-inst1
      NetworkInterfaces:
        - AssociatePublicIpAddress: true
          DeviceIndex: 0
          SubnetId: !Ref Subnet2
          GroupSet:
            - !Ref SecurityGroup2


```

