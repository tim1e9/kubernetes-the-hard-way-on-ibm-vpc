# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single availability zone.

> Ensure a default availability zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Unique Identifiers

A number of resources must be referenced by their unique identifier. This is not the same as the name of the resource.
In an effort to make this simpler to manage, environmefnt variables will be used. 

## Resource Groups

A resource group is a way for you to organize your account resources in customizable groupings.
Resource groups help control access, and provide a way to filter items when a large number of
users all work in the same account.

For this tutorial, we'll use a single resource group for all of our resources.

To create a resource group:

`RG_ID=$(ibmcloud resource group-create kube-thw-ibmvpc-rg --output JSON | jq -r ".id")`

Or, if you prefer to not store the ID in an environment variable:

`ibmcloud resource group-create kube-thw-ibmvpc-rg`

To show details about the newly created resource group:

`ibmcloud resource group kube-thw-ibmvpc-rg`



## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://www.ibm.com/cloud/learn/vpc) (VPC) network will be setup to host the Kubernetes cluster.

#### Create the VPC

Create the `kubernetes-the-hard-way` custom VPC network:

`VPC_ID=$(ibmcloud is vpc-create kube-thw-ibmvpc-vpc --resource-group-id $RG_ID --output JSON | jq -r .id)`

To viw the details of the VPC, type:

`ibmcloud is vpc $VPC_ID`

**NOTE:** You may notice silly names for the default ACL, Security Group, and Routing Table associated with the
VPC. At this point, it's possible to rename them. To do this, take note of the output from the previous command
and apply different names: 

```
SG1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_security_group.id)
ibmcloud is security-group-update $SG1 --name kube-thw-sg1
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_network_acl.id)
ibmcloud is network-acl-update $NACL_ID --name kube-thw-nacl1
RT1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_routing_table.id)
ibmcloud is vpc-routing-table-update $VPC_ID $RT1 --name kube-thw-rt1
```

Once the silly names have been replaced, it's necessary to add a rule to the security group:

```
ibmcloud is security-group-rule-add $SG1 inbound tcp --port-min 22 --port-max 22
ibmcloud is security-group-rule-add $SG1 inbound tcp --port-min 443 --port-max 443
ibmcloud is security-group-rule-add $SG1 inbound tcp --port-min 80 --port-max 80
ibmcloud is security-group-rule-add $SG1 inbound icmp --icmp-type 8
```
After these updates, the names may be simpler to understand.

### PUBLIC GATEWAY

Create a public gateway so that VSIs in the cluster can reach the public internet:

`PUB_GATEWAY=$(ibmcloud is public-gateway-create kube-thw-pub-gateway $VPC_ID us-south-1 --resource-group-id $RG_ID --output JSON | jq -r .id)`

#### Create a Subnet

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

`VPC_SUBNET_ID=$(ibmcloud is subnet-create kube-thw-ibmvpc-subnet1 $VPC_ID --ipv4-cidr-block 10.240.0.0/24 --public-gateway-id $PUB_GATEWAY --zone us-south-1 --resource-group-id $RG_ID --output JSON | jq -r .id)`

To view all subnets within the previously created resource group, type:

`ibmcloud is subnets --resource-group-id $RG_ID`


### Firewall Rules -> Access Control Lists (ACLs)

Access to an IBM VPC can be limited by leveraging either security groups
or access control lists (ACLs):

 - An Access Control List (ACL) can manage inbound and outbound traffic for a subnet
 - A security group acts as a virtual firewall that controls the traffic for one or more virtual server instances

To set up firewall rules for the subnet, we'll use an ACL. In fact, we'll refine the default ACL that was
created when we created the VPC.

By default, the network ACL allows all incoming and outgoing connections. We'll want to keep outgoing connections
open, but we'll want to lock down traffic coming into the subnet. Specifically, we'll limit incoming traffic
to SSH, ICMP, and HTTPS.

Since the network ACL ID is used quite a bit, we've used a variable - `NACL_ID`. If you haven't yet set it,
you can do it now with the following command:

`NACL_ID=$(ibmcloud is network-acls --resource-group-id $RG_ID --output JSON | jq -r .[0].id)`

To see the details of the ACL:

`ibmcloud is network-acl $NACL_ID`

Update the rules appropriately:

```
RULE_ID_TO_DELETE=$(ibmcloud is network-acl-rules $NACL_ID --output JSON | jq -r '.[] | select(.name=="allow-inbound") | .id')
ibmcloud is network-acl-rule-delete $NACL_ID $RULE_ID_TO_DELETE -f
ibmcloud is network-acl-rule-add $NACL_ID allow inbound udp 0.0.0.0/0 0.0.0.0/0 \
  --name dns1 --source-port-min 53 --source-port-max 53 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name dns2 --source-port-min 53 --source-port-max 53 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name https1 --source-port-min 443 --source-port-max 443 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name https2 --source-port-min 1 --source-port-max 65535 --destination-port-min 443 --destination-port-max 443
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name http1 --source-port-min 80 --source-port-max 80 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name http2 --source-port-min 1 --source-port-max 65535 --destination-port-min 80 --destination-port-max 80
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name ssh1 --source-port-min 1 --source-port-max 65535 --destination-port-min 22 --destination-port-max 22
ibmcloud is network-acl-rule-add $NACL_ID allow inbound icmp 0.0.0.0/0 0.0.0.0/0 \
  --name allow-icmp
```

To view all newly-updated ACL rules, use the same command as mentioned above:

`ibmcloud is network-acl $NACL_ID`


For more information about security within a VPC, please refer to:
https://cloud.ibm.com/docs/vpc?topic=vpc-security-in-your-vpc


> An [external load balancer](https://www.ibm.com/) will be used to expose the Kubernetes API Servers to remote clients.



### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

`FLOAT_IP1=$(ibmcloud is floating-ip-reserve kube-thw-fip1 --zone us-south-1 --resource-group-name kube-thw-ibmvpc-rg --output JSON | jq -r .id)`

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. The simplest way to configure SSH is to create
an SSH key resource within the resource group.

### Create a public / private keypair

To generate a key pair, run the following command:
`ssh-keygen -t rsa -b 4096 -C "kubethw"`

Sample command with output:
```
> ssh-keygen -t rsa -b 4096 -C "kubethw"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/someuser/.ssh/id_rsa): /home/someuser/.ssh/kubethw_id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/someuser/.ssh/kubethw_id_rsa
Your public key has been saved in /home/someuser/.ssh/kubethw_id_rsa.pub
```

### Import the key into the VPC

`SSH_KEY_ID=$(ibmcloud is key-create kube-thw-ssh-key @~/.ssh/kubethw_id_rsa.pub --resource-group-id $RG_ID --output JSON | jq -r .id)`


## Compute Instances (Virtual Server Instances)

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

First, we need to select the correct image to use. To see a list of all Ubuntu images which are available:

`ibmcloud is images --output JSON | jq '.[] | select(.operating_system.family=="Ubuntu Linux") | {id: .id, name: .name}'`

We'll again use an environment variable to store the name of the image:

`IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')`

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  ibmcloud is instance-create controller-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG1 --resource-group-id $RG_ID --ipv4 10.240.0.1${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The name of the worker node will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  ibmcloud is instance-create worker-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG1 --resource-group-id $RG_ID --ipv4 10.240.0.2${i}
done
```


### Verification

List the compute instances in your resource group:

`ibmcloud is instances --resource-group-id $RG_ID`

### Updating Environment Variables

If you accidentally exit a terminal, it may be painful to re-assign all variables. The following should help:

```
RG_ID=$(ibmcloud resource group kube-thw-ibmvpc-rg --output JSON | jq -r .[0].id)
VPC_ID=$(ibmcloud is vpcs --resource-group-id $RG_ID --output JSON | jq -r .[0].id) 
SG1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_security_group.id)
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_network_acl.id)
RT1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_routing_table.id)
VPC_SUBNET_ID=$(ibmcloud is subnets --resource-group-id $RG_ID --output JSON | jq -r .[0].id) 
SSH_KEY_ID=$(ibmcloud is keys --resource-group-id $RG_ID --output JSON | jq -r .[0].id) 
IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')
```

### Load Balancer

The overall solution will use a load balancer. There are multiple steps associated with setting up a load balancer.
However, for now, it's sufficient to simply create the base resource, so that we can identify the IP addresses
and overall name.

To create the load balancer:

`LB1=$(ibmcloud is load-balancer-create kube-thw-netwk-balancer public --subnet $VPC_SUBNET_ID --family network --security-group $SG1 --resource-group-id $RG_ID --output JSON | jq -r .id)`


## Verify Access to Compute Instances

Test SSH access to the `controller-0` compute instances:

1. Temporarily assign the floating IP address to an instance

    `ibmcloud is instances --resource-group-id $RG_ID`

    (Make a note of the instance ID)

    `ibmcloud is instance-network-interfaces [instance id]`

    (Make a note of the NIC ID)

    `ibmcloud is instance-network-interface-floating-ip-add [instance ID] [NIC ID] $FLOAT_IP1`

2. Using the previously-created public/private key, SSH to the instance 
    
    `ssh -i ~/.ssh/kubethw_id_rsa root@[Floating IP Address]`

3. Verify that you have access

    `pwd`

4. Log out and release the floating IP address. You may also want to delete the `known_hosts` entry for the IP address.
    `exit`

    `ibmcloud is instance-network-interface-floating-ip-remove [instance ID] [NIC ID] [Floating IP ID]`

A fully automated way to set up the Floating IP address is as follows:

```
PUB_IP_ADDR=$(ibmcloud is floating-ip $FLOAT_IP1 --output JSON | jq -r .address)
VSI_CTRL_0=$(ibmcloud is instances --resource-group-id $RG_ID --output JSON | jq -r '.[] | select(.name=="controller-0") | .id')
NIC1=$(ibmcloud is instance-network-interfaces $VSI_CTRL_0 --output JSON | jq -r .[].id)
ibmcloud is instance-network-interface-floating-ip-add $VSI_CTRL_0 $NIC1 $FLOAT_IP1
```


# Summary

In this section of the tutorial, you provisioned the necessary VPC components to support a Kubernetes deployment.
An abbreviated collection of commands are listed here, in case you revisit this tutorial, and simply want to
start at the next step.

Steps, in aggregate:

```
RG_ID=$(ibmcloud resource group-create kube-thw-ibmvpc-rg --output JSON | jq -r .id)
VPC_ID=$(ibmcloud is vpc-create kube-thw-ibmvpc-vpc --resource-group-id $RG_ID --output JSON | jq -r .id)

SG1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_security_group.id)
ibmcloud is security-group-update $SG1 --name kube-thw-sg1
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_network_acl.id)
ibmcloud is network-acl-update $NACL_ID --name kube-thw-nacl1
RT1=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_routing_table.id)
ibmcloud is vpc-routing-table-update $VPC_ID $RT1 --name kube-thw-rt1

ibmcloud is security-group-rule-add $SG1 inbound tcp --port-min 22 --port-max 22
ibmcloud is security-group-rule-add $SG1 inbound tcp --port-min 443 --port-max 443

VPC_SUBNET_ID=$(ibmcloud is subnet-create kube-thw-ibmvpc-subnet1 $VPC_ID --ipv4-cidr-block 10.240.0.0/24 --zone us-south-1 --resource-group-id $RG_ID --output JSON | jq -r .id)

RULE_ID_TO_DELETE=$(ibmcloud is network-acl-rules $NACL_ID --output JSON | jq -r '.[] | select(.name=="allow-inbound") | .id')
ibmcloud is network-acl-rule-delete $NACL_ID $RULE_ID_TO_DELETE -f
ibmcloud is network-acl-rule-add $NACL_ID allow inbound udp 0.0.0.0/0 0.0.0.0/0 \
  --name dns1 --source-port-min 53 --source-port-max 53 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name dns2 --source-port-min 53 --source-port-max 53 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name https1 --source-port-min 443 --source-port-max 443 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name https2 --source-port-min 1 --source-port-max 65535 --destination-port-min 443 --destination-port-max 443
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name http1 --source-port-min 80 --source-port-max 80 --destination-port-min 1 --destination-port-max 65535
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name http2 --source-port-min 1 --source-port-max 65535 --destination-port-min 80 --destination-port-max 80
ibmcloud is network-acl-rule-add $NACL_ID allow inbound tcp 0.0.0.0/0 0.0.0.0/0 \
  --name ssh1 --source-port-min 1 --source-port-max 65535 --destination-port-min 22 --destination-port-max 22
ibmcloud is network-acl-rule-add $NACL_ID allow inbound icmp 0.0.0.0/0 0.0.0.0/0 \
  --name allow-icmp

FLOAT_IP1=$(ibmcloud is floating-ip-reserve kube-thw-fip1 --zone us-south-1 --resource-group-name kube-thw-ibmvpc-rg --output JSON | jq -r .id)
SSH_KEY_ID=$(ibmcloud is key-create kube-thw-ssh-key @~/.ssh/kubethw_id_rsa.pub --resource-group-id $RG_ID --output JSON | jq -r .id)
IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')

LB1=$(ibmcloud is load-balancer-create kube-thw-netwk-balancer public --subnet $VPC_SUBNET_ID --family network --security-group $SG1 --resource-group-id $RG_ID --output JSON | jq -r .id)

for i in 0 1 2; do
  ibmcloud is instance-create controller-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG1 --resource-group-id $RG_ID --ipv4 10.240.0.1${i}
done

for i in 0 1 2; do
  ibmcloud is instance-create worker-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG1 --resource-group-id $RG_ID --ipv4 10.240.0.2${i}
done
```

And to test that all variables have been set:

```
echo Resource Group ID: $RG_ID
echo VPC ID: $VPC_ID
echo Security Group ID: $SG1
echo Network ACL ID: $NACL_ID
echo Routing Table ID: $RT1
echo VPC Subnet ID: $VPC_SUBNET_ID
echo Floating IP Address ID: $FLOAT_IP1
echo SSH Key ID: $SSH_KEY_ID
echo Application Load Balancer ID: $LB1
```


Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

