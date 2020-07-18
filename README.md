# multicloudtask4


## Creating VPC | Subnet | Route table | Internet Gateway | NAT Gateway and launching Wordpress | MySQL in our VPC Using Terraform

![start](https://user-images.githubusercontent.com/45136716/87859656-6bb9dc80-c954-11ea-98e0-57189c3b10ac.png)


# Performing the following steps:
1.  Write an Infrastructure as code using terraform, which automatically create a VPC.

2.  In that VPC we have to create 2 subnets:
    1.   public  subnet [ Accessible for Public World! ] 
    2.   private subnet [ Restricted for Public World! ]

3. Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.

4. Create  a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.

5.  Create a NAT gateway for connect our VPC/Network to the internet world  and attach this gateway to our VPC in the public network

6.  Update the routing table of the private subnet, so that to access the internet it uses the nat gateway created in the public subnet

7. Create a Security Group in our created VPC in public subnet for wordpress instance which allow only HTTP port 80. So that only our clint can visit wordpress site and no one can do ssh to wordpress instance. Then our wordpress instance will be highly secure.

8. Create a Security Group in our created VPC in private subnet for MySQL DataBase instance which allow only TCP port 3306 and in source type we give wordpress security group id insted of all ip. So if any clint come to our wordpress site then wordpress connect to MySQL and get data. And no one can access our DataBase from public world. So our MySQL DataBase will be highly secure.

9. Launch an ec2 instance which has Wordpress setup already having the security group allowing port 80 sothat our client can connect to our wordpress site. Also attach the key to instance for further login into it.

10. Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our wordpress vm can connect with the same. Also attach the key with the same.

11. Create a NAT gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC in the public network. First we create Elastic IP because for creating NAT gateway we have to allocate Elastic IP.

12. Create a route table and Update the routing table of the private subnet, so that to access the internet it uses the nat gateway created in the public subnet.

Note: Wordpress instance has to be part of public subnet so that our client can connect our site. 
mysql instance has to be part of private  subnet so that outside world can't connect to it.
Don't forgot to add auto ip assign and auto dns name assignment option to be enabled. 

![start2](https://user-images.githubusercontent.com/45136716/87859659-6e1c3680-c954-11ea-8b0e-113dcb1cb7f9.png)

 ## Understand VPC

VPC is a network which keeps our infrastructure isolated from outside world known as virtual us a virtual space to create an infrastructure which looks like real. so other company can’t see what’s happening over here. Only 5 VPC Amazon allow us to create if you want to create more than we have to contact support team.

**AWS** offer highly secure and available network solutions with consistently high performance and global coverage. Today we will try to set up a public and a private subnet to launch ec2 instances in AWS. We will do this by using Terraform.


**Amazon Virtual Private Cloud (Amazon VPC)** lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. You have complete control over your virtual networking environment, including selection of your own IP address range, creation of subnets, and configuration of route tables and network gateways. You can use both IPv4 and IPv6 in your VPC for secure and easy access to resources and applications.

The AWS **VPC** is secure as it provides security groups and network access control lists, to enable inbound and outbound filtering at the instance and subnet level. The VPC has a lot of use-cases and is quite simple with a customizable environment.

## We shall continue from here.

## VPC

Now, we have to write a code block for the VPC we have to create. Classless inter-domain routing (CIDR) is a set of Internet protocol (IP) standards that is used to create unique identifiers for networks and individual devices. So, to provide this range of IP I’ve used ‘192.168.0.0/16’ in the CIDR block.

```
resource "aws_vpc" "task4vpc" {
  cidr_block       = "192.168.0.0/16"
  enable_dns_support="true"
  enable_dns_hostnames="true"
  instance_tenancy = "default"
  tags = {
    Name = "task4vpc"
  }
}
```

## Subnets

A subnet is a segmented piece of a larger network. More specifically, subnets are a logical partition of an IP network into multiple, smaller network segments.
We need to create two types of Subnets, a private and a public. The public subnet which connects to the internet, I’ve created it in the ap-south-1a region as I am planning to launch the WordPress in the same region.

```
resource "aws_subnet" "publicsubnet4" {
  vpc_id     =  aws_vpc.task4vpc.id
  cidr_block = "192.168.0.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "publicsubnet4"
  }
}
```

The private subnet is a subnet that’s associated with a route table that has a route to an Internet gateway. This subnet cannot connect to the Internet I.e., cannot be accessed publicly and I’ve launched it in the ap-south-1b region where I’ll launch MySQL .

```
resource "aws_subnet" "privatesubnet4" {
  vpc_id     =  aws_vpc.task4vpc.id
  cidr_block = "192.168.1.0/24"
  availability_zone = "ap-south-1b"
  tags = {
    Name = "privatesubnet4"
  }
}

```

## Elastic IP:

An Elastic IP address is a static IPv4 address designed for dynamic cloud computing. An Elastic IP address is associated with your AWS account. With an Elastic IP address, you can mask the failure of an instance or software by rapidly remapping the address to another instance in your account.
An EIP Address is a public IPv4 address that is accessible from the internet and is used by an instance to communicate with the internet.

```
resource "aws_eip" "elasticip"{
  vpc = true
} 

```

# terraform code for security group both MySQL & wordpress:

```
resource "aws_security_group" "wp_sec_grp" {
  name        = "wp_sec_grp"
  description = "Allows SSH and HTTP"
  vpc_id      = "${aws_vpc.main.id}"


  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [ "0.0.0.0/0" ]
  }
 
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [ "0.0.0.0/0" ]
  }


  ingress {
    description = "allow ICMP"
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = {
    Name = "wp_sec_grp"
  }
}





### creating the security group fop allowing 3306 inbound rules


resource "aws_security_group" "mysql_sec_group" {
  name        = "mysql_sec_grp"
  description = "Allows MYSQL"
  vpc_id      = "${aws_vpc.main.id}"


  tags = {
    Name = "mysql_sec_group"
  }
}


resource "aws_security_group_rule" "allow_mysql_from_wp" {
  type              = "ingress"
  from_port         = 0
  to_port           = 3306
  protocol          = "tcp"
  security_group_id = "${aws_security_group.mysql_sec_group.id}"
  source_security_group_id = "${aws_security_group.wp_sec_grp.id}"
}


  ingress {
    description = "allow ICMP"
    from_port = 0
    to_port = 0
    protocol = "-1"
    security_groups = [aws_security_group.wp_sec_grp.id]
  }



```
## Terraform code for MySQL instance :

```
resource "aws_instance" "mysql" {
  ami = "ami-08706cb5f68222d09"
  instance_type = "t2.micro"
  key_name = "${aws_key_pair.generated_key.key_name}"
    vpc_security_group_ids = [aws_security_group.mysql_sec_group.id]
    subnet_id = "${aws_subnet.private.id}"


    tags = {
        Name = "mysql_os"
    }


    depends_on = [ tls_private_key.mykey, aws_vpc.main, aws_security_group.wp_sec_grp, aws_security_group.mysql_sec_group, aws_subnet.public, aws_subnet.private, aws_internet_gateway.custom_igww ] 
}


```


## NAT Gateway:
We can use a network address translation (NAT) gateway to enable instances in a private subnet to connect to the internet or other AWS services but prevent the internet from initiating a connection with those instances.

So here our whole infrastructure will be the same except one more resource we need to add to create NAT gateway in Public Subnet and associate Private Subnet with NAT GW. Always remember that a NAT GW should always be created in Public Subnet that already has IGW, else it won't get the internet access and there will be no means to create a NAT GW.


Here we will create one EIP using TF code that will be attached to NAT-GW, then Create a Private Route table where we write to routes and finally associate Private Subnet with NAT-GW.


```
## creating the EIP for NAT Gateway



resource "aws_eip" "nat-eip" {
  vpc = true


  depends_on = [ aws_internet_gateway.custom_igww ]
}





## Creating the NAT Gateway


resource "aws_nat_gateway" "natgw" {
  allocation_id = "${aws_eip.nat-eip.id}"
  subnet_id = "${aws_subnet.public.id}"


  depends_on = [ aws_internet_gateway.custom_igww ]


  tags = {
    Name = "my-natgw"
  }
}



## To associate the route table with the private subnet for public access



resource "aws_route_table_association" "private_sn_assoc" {
  subnet_id = aws_subnet.private.id
  route_table_id = aws_route_table.private_route.id
}


```
![applyyyyyyyy](https://user-images.githubusercontent.com/45136716/87859650-5f358400-c954-11ea-9583-4c7dcf1d7cac.png)

**Wordpess**

![0000](https://user-images.githubusercontent.com/45136716/87859644-5644b280-c954-11ea-9f5f-e3035fa73337.png) 

![0001](https://user-images.githubusercontent.com/45136716/87859645-58a70c80-c954-11ea-9019-323a545dfd21.png)

Now we will have a look at the VPC to check whether NAT Gateway created or not and if the private subnet is associated with it......

![0002](https://user-images.githubusercontent.com/45136716/87859646-5b096680-c954-11ea-9adf-120a179b365c.png)

The Private route via how private instances go to the internet ( via NGW )

![0003](https://user-images.githubusercontent.com/45136716/87859648-5c3a9380-c954-11ea-8c76-87a1b34288d0.png)

The Public route via how public instances go to the internet ( via IGW )

![0004](https://user-images.githubusercontent.com/45136716/87859780-4b3e5200-c955-11ea-80c0-a023b56acd46.png)

With our entire Infrastructure build has finished we will enter in the WordPress instance and then further go inside the MySQL instance of ours and check the network connectivity.

The WordPress instance is the Bastion Host/Jump box in this case of ours. A bastion host is a server whose purpose is to provide access to a private network from an external network. Because of its exposure to potential attack, a bastion host must minimize the chances of penetration.
Before we connect to MySQL instance from the WordPress we will have to provide the key to the instance,

We can use WinSCP for this:

![win scp](https://user-images.githubusercontent.com/45136716/87859661-71afbd80-c954-11ea-9fb4-6a44d7f2b38b.png)

## To delete the whole Infrastructure we run the following command:

```
terraform destroy --auto-approve
```


thanks alot

[Fellow me ](https://github.com/ManthanChoudhury)





