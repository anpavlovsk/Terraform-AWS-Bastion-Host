
# Connecting to a AWS Instance in a private subnet using a Bastion host

## Connecting to a private subnet 
Instances within the same VPC can connect to one another via their private IP addresses, as such it is possible to connect to an instance in a private subnet from an instance in a public subnet; otherwise known as a bastion host.

Amazon instances use SSH keys for authentication. As such connecting to the private instance will require a private key on the bastion host; in the same way connecting to the public instance requires a private key on your host machine, however this is extremely bad practise. **Never expose your private keys to a bastion host!**

An alternative solution is to use SSH agent forwarding, which allows a user to connect from the bastion to another instance without storing the private key on the bastion. An SSH agent is a program that keeps track of user’s identity keys and their pass phrases and can be configured with the following commands:

The next step is to generate the SSH keys. In the terraform directory create another directory called keys and create your keys with the following command:
```
# create the keys
ssh-keygen -f mykeypair
```


## Infrastructure
The infrastructure below has been deployed using Terraform

![alt text](https://github.com/anpavlovsk/Terraform-AWS-Bastion-Host/blob/main/screenshots/awsbastionhostsssh.png?raw=true) 

## VPC: 
````
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/27"
}
````

## Subnets:
````
resource "aws_subnet" "public-subnet" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.0/28"
  map_public_ip_on_launch = "true"
  availability_zone       = "eu-west-1a"

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_subnet" "private-subnet" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.16/28"
  map_public_ip_on_launch = "false"
  availability_zone       = "eu-west-1a"

  tags = {
    Name = "private-subnet"
  }
}
```` 
The key differentiator between a private and public subnet is the map_public_ip_on_launch flag, if this is True, instances launched in this subnet will have a public IP address and be accessible via the internet gateway.

## Internet Gateway: 
````
resource "aws_internet_gateway" "internet-gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "internet-gw"
  }
}
````

For a subnet to be accessible to the internet an AWS internet gateway is required. An internet gateway allows internet traffic to and from your VPC.

## Route table: 
````
resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.internet-gw.id
  }
}

resource "aws_route_table_association" "public-rta" {
  subnet_id      = aws_subnet.public-subnet.id
  route_table_id = aws_route_table.public-rt.id
}
Footer

````
A Route table specifies which external IP address are contactable from a subnet or internet gateway.

## Nat Gateway: 
````
resource "aws_eip" "nat" {
  vpc = true
}

resource "aws_nat_gateway" "nat-gw" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public-subnet.id
  depends_on    = [aws_internet_gateway.internet-gw]
}

resource "aws_route_table" "private-rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat-gw.id
  }
}

resource "aws_route_table_association" "private-rta" {
  subnet_id      = aws_subnet.private-subnet.id
  route_table_id = aws_route_table.private-rt.id
}
````
A Nat Gateway enables instances in private subnets to connect to the internet. The Nat gateway must be deployed in the public subnet with an Elastic IP. Once the resource is created, a route table associated with the the private subnet needs to point internet-bound traffic to the NAT gateway.

## Security Groups: 
````
resource "aws_security_group" "allow-ssh" {
  vpc_id      = aws_vpc.main.id
  name        = "allow-ssh"
  description = "security group that allows ssh and all egress traffic"
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "allow-ssh"
  }
}
````
A security group acts as a virtual firewall for your instance to control incoming and outgoing traffic. The security group below enables all traffic over port 22 (SSH). Both instances in the private and public subnet require this security group.

## Ec2 Instances and Keys: 
````
resource "aws_instance" "bastion-instance" {
  ami           = var.instance-ami
  instance_type = "t2.micro"

  subnet_id = aws_subnet.public-subnet.id

  vpc_security_group_ids = [aws_security_group.allow-ssh.id]

  key_name = aws_key_pair.mykeypair.key_name

  tags = {
    Name = "bastion-instance"
  }
}

resource "aws_instance" "private-instance" {
  ami           = var.instance-ami
  instance_type = "t2.micro"

  subnet_id = aws_subnet.private-subnet.id

  vpc_security_group_ids = [aws_security_group.allow-ssh.id]

  key_name = aws_key_pair.mykeypair.key_name
  
  tags = {
    Name = "private-instance"
  }
}

resource "aws_key_pair" "mykeypair" {
  key_name   = "mykeypair"
  public_key = file(var.key_path)
}
````
After all the necessary infrastructure has been defined, we can set up our Ec2 instances. The instances require an AWS key-pair to authenticate access which is created below using the aws_key_pair resource and existing ssh key created earlier.

Now that the infrastructure is complete the next step is to deploy. This can be achieved with the following Terraform commands in the terraform directory:

````
terraform init
terraform apply
````
If the deployment has been successful, you’ll be able to see two new EC-2 instances in your AWS console.
![alt text](https://github.com/anpavlovsk/Terraform-AWS-Bastion-Host/blob/main/screenshots/awsbastionhostspublic.png?raw=true) 
![alt text](https://github.com/anpavlovsk/Terraform-AWS-Bastion-Host/blob/main/screenshots/awsbastionhostsprivate.png?raw=true) 


The SSH config file
The SSH config file is a great resource for storing all your configuration for the remote machines you connect to. It is located in your home directory here: .ssh/config. The config file isn’t automatically created, so if it doesn’t exist you will have to create it.

## SSH Config File

```
IdentityFile ~/Terraform-AWS-Bastion-Host/terraform/keys/mykeypair

Host bastion-instance
   HostName 52.213.245.171
   User ubuntu
Host private-instance
   HostName 10.0.0.24
   User ubuntu
   ProxyCommand ssh -q -W %h:%p bastion-instance
```
Output 
````
alp@master1:~/Terraform-AWS-Bastion-Host/terraform$ ssh private-instance
The authenticity of host '52.213.245.171 (52.213.245.171)' can't be established.
ECDSA key fingerprint is SHA256:bU7Cw8G0oZvqC3Vgq9+LwpJxLaw4pLDbFboovXTWhgE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
The authenticity of host '10.0.0.24 (<no hostip for proxy command>)' can't be established.
ECDSA key fingerprint is SHA256:qgmeZQ/Uq7mIyNfo/80gmu0wHe1m1RyGJU9lW431G08.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.0.24' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-1017-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Oct 11 10:00:03 UTC 2022

  System load:  0.08              Processes:           90
  Usage of /:   13.8% of 7.69GB   Users logged in:     0
  Memory usage: 17%               IP address for eth0: 10.0.0.24
  Swap usage:   0%

0 packages can be updated.
0 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@ip-10-0-0-24:~$ ip a | grep global
    inet 10.0.0.24/28 brd 10.0.0.31 scope global dynamic eth0
ubuntu@ip-10-0-0-24:~$ ping -c 3 google.com
PING google.com (74.125.193.138) 56(84) bytes of data.
64 bytes from ig-in-f138.1e100.net (74.125.193.138): icmp_seq=1 ttl=47 time=11.8 ms
64 bytes from ig-in-f138.1e100.net (74.125.193.138): icmp_seq=2 ttl=47 time=11.4 ms
64 bytes from ig-in-f138.1e100.net (74.125.193.138): icmp_seq=3 ttl=47 time=11.4 ms

--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.438/11.601/11.876/0.195 ms
````
