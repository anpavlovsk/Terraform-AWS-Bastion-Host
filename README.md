
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

Image here

## VPC: 

## Subnets: 
## Subnets: 

The key differentiator between a private and public subnet is the map_public_ip_on_launch flag, if this is True, instances launched in this subnet will have a public IP address and be accessible via the internet gateway.

## Internet Gateway: 

For a subnet to be accessible to the internet an AWS internet gateway is required. An internet gateway allows internet traffic to and from your VPC.

Route table: A Route table specifies which external IP address are contactable from a subnet or internet gateway.

Nat Gateway: A Nat Gateway enables instances in private subnets to connect to the internet. The Nat gateway must be deployed in the public subnet with an Elastic IP. Once the resource is created, a route table associated with the the private subnet needs to point internet-bound traffic to the NAT gateway.

Security Groups: A security group acts as a virtual firewall for your instance to control incoming and outgoing traffic. The security group below enables all traffic over port 22 (SSH). Both instances in the private and public subnet require this security group.

Ec2 Instances and Keys: After all the necessary infrastructure has been defined, we can set up our Ec2 instances. The instances require an AWS key-pair to authenticate access which is created below using the aws_key_pair resource and existing ssh key created earlier.

Now that the infrastructure is complete the next step is to deploy. This can be achieved with the following Terraform commands in the terraform directory:

````
terraform init
terraform apply
````
If the deployment has been successful, you’ll be able to see two new EC-2 instances in your AWS console.
Image here

The SSH config file
The SSH config file is a great resource for storing all your configuration for the remote machines you connect to. It is located in your home directory here: .ssh/config. The config file isn’t automatically created, so if it doesn’t exist you will have to create it.

## SSH Config File

```
IdentityFile ~/Terraform-AWS-Bastion-Host/terraform/keys/mykeypair

Host bastion-instance
   HostName 34.248.244.4
   User ubuntu
Host private-instance
   HostName 10.0.0.23
   User ubuntu
   ProxyCommand ssh -q -W %h:%p bastion-instance
```
