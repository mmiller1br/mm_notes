
## Diagram

![aws-lab001](https://github.com/mmiller1br/mm_notes/assets/32887571/ecf02458-1b79-4d55-b787-15d121c5d512)

## ANSIBLE playbook to create this LAB

```yaml
---
- name: Create VPC and Subnet
  hosts: localhost
  gather_facts: False
  vars:
    aws_region: us-east-1
    vpc_cidr_block: 10.0.0.0/16
    subnet_cidr_block: 10.0.0.0/24

  tasks:
    - name: create a VPC1
      amazon.aws.ec2_vpc_net:
        name: myvpc1-ansible
        cidr_block: 10.0.0.0/16
        region: us-east-1
        tags:
          module: ec2_vpc_net
        tenancy: default
      register: vpc1
#    - name: vpc1 output
#      debug:
#        var: vpc1

    - name: create a VPC2
      amazon.aws.ec2_vpc_net:
        name: myvpc2-ansible
        cidr_block: 192.168.0.0/16
        region: us-east-1
        tags:
          module: ec2_vpc_net
        tenancy: default
      register: vpc2
#    - name: vpc2 output
#      debug:
#        var: vpc2

    - name: Create subnet1
      amazon.aws.ec2_vpc_subnet:
        state: present
        vpc_id: "{{ vpc1.vpc.id }}"
        cidr: 10.0.0.0/24
        tags:
          Name: mysubnet1-ansible
      register: my_subnet1

    - name: Create subnet2
      amazon.aws.ec2_vpc_subnet:
        state: present
        vpc_id: "{{ vpc2.vpc.id }}"
        cidr: 192.168.0.0/24
        tags:
          Name: mysubnet2-ansible
      register: my_subnet2

    - name: Create Internet gateway 1
      amazon.aws.ec2_vpc_igw:
        vpc_id: "{{ vpc1.vpc.id }}"
        state: present
        tags:
          Name: myigw1-ansible
      register: igw1

    - name: Create Internet gateway 2
      amazon.aws.ec2_vpc_igw:
        vpc_id: "{{ vpc2.vpc.id }}"
        state: present
        tags:
          Name: myigw2-ansible
      register: igw2

    - name: Create local account VPC peering Connection
      community.aws.ec2_vpc_peer:
        #region: ap-southeast-2
        vpc_id: "{{ vpc1.vpc.id }}"
        peer_vpc_id: "{{ vpc2.vpc.id }}"
        state: present
        tags:
          Name: my_vpcpeering-ansible
      register: myvpc_peering

    - name: Accept local VPC peering request
      community.aws.ec2_vpc_peer:
        peering_id: "{{ myvpc_peering.peering_id }}"
        state: accept
      #register: action_peer

    - name: create Route Table 1
      amazon.aws.ec2_vpc_route_table:
        vpc_id: "{{ vpc1.vpc.id }}"
        subnets:
          - "{{ my_subnet1.subnet.id }}"
        routes:
          - dest: 0.0.0.0/0
            gateway_id: "{{ igw1.gateway_id }}"
          - dest: 192.168.0.0/24
            gateway_id: "{{ myvpc_peering.peering_id }}"
        tags:
          Name: myrt1-ansible

    - name: create Route Table 2
      amazon.aws.ec2_vpc_route_table:
        vpc_id: "{{ vpc2.vpc.id }}"
        subnets:
          - "{{ my_subnet2.subnet.id }}"
        routes:
          - dest: 0.0.0.0/0
            gateway_id: "{{ igw2.gateway_id }}"
          - dest: 10.0.0.0/24
            gateway_id: "{{ myvpc_peering.peering_id }}"
        tags:
          Name: myrt2-ansible

    - name: create security group 1
      amazon.aws.ec2_group:
        name: my-secgrp1-ansible
        description: allow ssh and http
        vpc_id: "{{ vpc1.vpc.id }}"
        rules:
          - proto: icmp
            from_port: -1 # icmp type, -1 = any type
            to_port:  -1  # icmp subtype, -1 = any subtype
            cidr_ip: 192.168.0.0/24
          - proto: icmp
            from_port: -1 # icmp type, -1 = any type
            to_port:  -1  # icmp subtype, -1 = any subtype
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports: 22
            cidr_ip: 0.0.0.0/0
      #register: my-secgrp1

    - name: create security group 2
      amazon.aws.ec2_group:
        name: my-secgrp2-ansible
        description: allow ssh and http
        vpc_id: "{{ vpc2.vpc.id }}"
        rules:
          - proto: icmp
            from_port: -1 # icmp type, -1 = any type
            to_port:  -1  # icmp subtype, -1 = any subtype
            cidr_ip: 10.0.0.0/24
          - proto: icmp
            from_port: -1 # icmp type, -1 = any type
            to_port:  -1  # icmp subtype, -1 = any subtype
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports: 22
            cidr_ip: 0.0.0.0/0
      #register: my-secgrp2

    - name: start an instance with a public IP address on VPC1
      amazon.aws.ec2_instance:
        name: "myvpc1-inst1-ansible"
        key_name: "mykey001"
        vpc_subnet_id: "{{ my_subnet1.subnet.id }}"
        instance_type: t2.micro
        security_group: my-secgrp1-ansible
        network:
          assign_public_ip: true
        image_id: ami-022e1a32d3f742bd8
        tags:
          Environment: Testing

    - name: start an instance with a public IP address on VPC2
      amazon.aws.ec2_instance:
        name: "myvpc2-inst1-ansible"
        key_name: "mykey001"
        vpc_subnet_id: "{{ my_subnet2.subnet.id }}"
        instance_type: t2.micro
        security_group: my-secgrp2-ansible
        network:
          assign_public_ip: true
        image_id: ami-022e1a32d3f742bd8
        tags:
          Environment: Testing
```

## ANSIBLE playbook to delete this LAB

```yaml
---
- name: Delete everything create with playbook aws-lab001.yml
  hosts: localhost
  gather_facts: False
  vars:
    security_group_names:
      - my-secgrp1-ansible
      - my-secgrp2-ansible
    route_table_names:
      - myrt1-ansible
      - myrt2-ansible
    internet_gateway_names:
      - myigw1-ansible
      - myigw2-ansible
    subnet_names:
      - mysubnet1-ansible
      - mysubnet2-ansible
    my_vpcs:
      - myvpc1-ansible
      - myvpc2-ansible

  tasks:
    #- name: Terminate every running instance in a region. Use with EXTREME caution.
    #  amazon.aws.ec2_instance:
    #    state: absent
    #    filters:
    #      instance-state-name: running

    # Delete instances by name
    - name: Terminate my 2 Instances created on VPC1 and VPC2
      amazon.aws.ec2_instance:
        state: absent
        filters:
          "tag:Name":
            - myvpc1-inst1-ansible
            - myvpc2-inst1-ansible

    # Delete Security Groups by name
    - name: Delete Security Groups
      amazon.aws.ec2_security_group:
        #group_id: sg-33b4ee5b
        name: "{{ item }}"
        state: absent
      loop: "{{ security_group_names }}"

    # Delete Route Table reading the info first, and then delete them
    - name: Get Route Table IDs
      amazon.aws.ec2_vpc_route_table_info:
        filters:
          "tag:Name": "{{ route_table_names }}"
      register: route_table_info

    - name: Delete Route Tables (created manually)
      amazon.aws.ec2_vpc_route_table:
        route_table_id: "{{ item.id }}"
        state: absent
        lookup: id
      loop: "{{ route_table_info.route_tables }}"

    # Delete VPC Peering
    - name: Get all vpc peers
      community.aws.ec2_vpc_peering_info:
        filters:
          "tag:Name": my_vpcpeering-ansible
      register: my_vpc_peering

    - name: Delete VPC Peering
      community.aws.ec2_vpc_peer:
        peering_id: "{{ item.vpc_peering_connection_id }}"
        state: absent
      loop: "{{ my_vpc_peering.vpc_peering_connections }}"

    # Delete Internet Gateways
    - name: Get IGW info
      amazon.aws.ec2_vpc_igw_info:
        filters:
          "tag:Name": "{{ internet_gateway_names }}"
      register: igw_info

    #- name: igw debug
    #  debug:
    #    var: igw_info.internet_gateways

    - name: Delete IGWs
      amazon.aws.ec2_vpc_igw:
        state: absent
        vpc_id: "{{ item.attachments[0].vpc_id }}"
      loop: "{{ igw_info.internet_gateways }}"

    # Delete subnets by name
    - name: Get list with my-subnets
      amazon.aws.ec2_vpc_subnet_info:
        filters:
          "tag:Name": "{{ subnet_names }}"
      register: subnet_info

    #- name: subnet debug
    #  debug:
    #    var: subnet_info

    - name: Delete Subnets
      amazon.aws.ec2_vpc_subnet:
        state: absent
        vpc_id: "{{ item.vpc_id }}"
        cidr: "{{ item.cidr_block }}"
      loop: "{{ subnet_info.subnets }}"

    # Delete VPCs
    - name: Get info from VPC1 and VPC2
      amazon.aws.ec2_vpc_net_info:
        filters:
          "tag:Name":
            - myvpc1-ansible
            - myvpc2-ansible
      register: vpc_info

    - name: Delete VPC1 and VPC2
      amazon.aws.ec2_vpc_net:
        state: absent
        vpc_id: "{{ item.vpc_id }}"
        cidr_block: "{{ item.cidr_block }}"
      loop: "{{ vpc_info.vpcs }}"
```

Command to run this playbooks:

```
 ansible@aut01:~$ ansible-playbook aws-lab01.yml

 ansible@aut01:~$ ansible-playbook aws-lab01-del.yml
```

