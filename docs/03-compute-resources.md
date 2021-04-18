# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single availability zone.

> Ensure a default availability zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Unique Identifiers

A number of resources must be referenced by their unique identifier. This is not the same as the name of the resource.
In an effort to make this simpler to manage, environment variables will be used. 

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

`VPC_ID=$(ibmcloud is vpc-create kube-thw-ibmvpc-vpc --resource-group-id $RG_ID --output JSON | jq -r ".id")`

To viw the details of the VPC, type:

`ibmcloud is vpc $VPC_ID`

**NOTE:** You may notice silly names for the default ACL, Security Group, and Routing Table associated with the
VPC. At this point, it's possible to rename them. To do this, take note of the output from the previous command
and apply different names: 

```
SG_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_security_group.id")
ibmcloud is security-group-update $SG_ID --name kube-thw-ibmvpc-sg
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_network_acl.id")
ibmcloud is network-acl-update $NACL_ID --name kube-thw-ibmvpc-nacl
RT_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_routing_table.id")
ibmcloud is vpc-routing-table-update $VPC_ID $RT_ID --name kube-thw-ibmvpc-rt
```

After these updates, the names may be simpler to understand.

### PUBLIC GATEWAY

Create a public gateway so that VSIs in the cluster can reach the public internet:

`PG_ID=$(ibmcloud is public-gateway-create kube-thw-ibmvpc-pg $VPC_ID us-south-1 --resource-group-id $RG_ID --output JSON | jq -r ".id")`

#### Create a Subnet

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

`VPC_SUBNET_ID=$(ibmcloud is subnet-create kube-thw-ibmvpc-subnet $VPC_ID --ipv4-cidr-block 10.240.0.0/24 --public-gateway-id $PG_ID --zone us-south-1 --resource-group-id $RG_ID --output JSON | jq -r ".id")`

To view all subnets within the previously created resource group, type:

`ibmcloud is subnets --resource-group-id $RG_ID`


> An [external load balancer](https://www.ibm.com/) will be used to expose the Kubernetes API Servers to remote clients.



### Allocating a Public IP Address

A static IP address will be used to "jump" into the VPC. Specifically, an extra VSI will be created, and it will have
a public IP address. By logging into this machine via `ssh`, you'll then be able to jump (also via `ssh`) onto the other
VSIs in the cluster.

Allocate a static IP address that will be attached to Bastion server:

```
FIP_ID=$(ibmcloud is floating-ip-reserve kube-thw-ibmvpc-fip --zone us-south-1 --resource-group-name kube-thw-ibmvpc-rg --output JSON | jq -r ".id")
BASTION_IP=$(ibmcloud is floating-ip $FIP_ID --output JSON | jq -r ".address")
```


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

`SSH_KEY_ID=$(ibmcloud is key-create kube-thw-ibmvpc-ssh-key @~/.ssh/kubethw_id_rsa.pub --resource-group-id $RG_ID --output JSON | jq -r ".id")`

## Compute Instances (Virtual Server Instances)


The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

First, we need to select the correct image to use. To see a list of all Ubuntu images which are available:

`ibmcloud is images --output JSON | jq '.[] | select(.operating_system.family=="Ubuntu Linux") | {id: .id, name: .name}'`

We'll again use an environment variable to store the name of the image:

`IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')`

### Bastion Host

As mentioned previously, we'll connect to the various virtual machines (or VSIs) via a Bastion host:

```
BASTION_ID=$(ibmcloud is instance-create kube-thw-ibmvpc-bastion $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID \
  --image-id $IMAGE_ID --key-ids $SSH_KEY_ID --security-group-ids $SG_ID --resource-group-id  $RG_ID \
  --ipv4 10.240.0.30 --output JSON | jq -r ".id")
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  ibmcloud is instance-create controller-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG_ID --resource-group-id $RG_ID --ipv4 10.240.0.1${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The name of the worker node will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

At a later step in the tutorial, we'll associate the CIDR ranges with each worker node.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  ibmcloud is instance-create worker-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID
    --image-id $IMAGE_ID --allow-ip-spoofing true --key-ids $SSH_KEY_ID \
    --security-group-ids $SG_ID --resource-group-id $RG_ID --ipv4 10.240.0.2${i}
done
```

**NOTE:** To support Kubernetes IP addresses, it's important to enable "IP Spoofing". To read more about this feature,
see the documentation: [About IP spoofing checks](https://cloud.ibm.com/docs/vpc?topic=vpc-ip-spoofing-about)


### Assign the Public IP Address to the Bastion Host

To make the Bastion host available, it's necessary to assign the floating IP address to the Bastion VSI:

Note: We previously defined the ID of the Bastion host to the variable `BASTION_ID`

```
BN_NIC_ID=$(ibmcloud is instance $BASTION_ID | jq -r ".primary_network_interface.id")
ibmcloud is instance-network-interface-floating-ip-add $BASTION_ID $BN_NIC_ID $FIP_ID
```

### Verification

List the compute instances in your resource group:

`ibmcloud is instances --resource-group-id $RG_ID`

## Load Balancer

The overall solution will use a load balancer. There are multiple steps associated with setting up a load balancer.
However, for now, it's sufficient to simply create the base resource, so that we can identify the IP address
and hostname.

To create the network load balancer:

`NLB_ID=$(ibmcloud is load-balancer-create kube-thw-ibmvpc-nlb public --subnet $VPC_SUBNET_ID --family network --security-group $SG_ID --resource-group-id $RG_ID --output JSON | jq -r ".id")`

To gather the private IPs, public IP, and hostname of the load balancer, use the following:

```
LB_PRIVATE_IPS_ARR=($(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r '.private_ips[].address | @sh' | tr -d \'))
LB_PRIVATE_IPS=$(echo ${LB_PRIVATE_IPS_ARR[@]} | tr ' ' ',')
LB_PUBLIC_IPS_ARR=($(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r '.public_ips[].address | @sh' | tr -d \'))
LB_PUBLIC_IP=$(echo ${LB_PUBLIC_IPS_ARR[@]} | tr ' ' ',')
LB_PUBLIC_HOSTNAME=$(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r .hostname)
ALL_LB_NAMES=${LB_PUBLIC_HOSTNAME},${LB_PRIVATE_IPS},${LB_PUBLIC_IP}
```

## Verify Access to Compute Instances

Test SSH access to the `controller-0` compute instances:
    `ssh -i ~/.ssh/kubethw_id_rsa root@$BASTION_IP`

Verify that you have access
    `pwd`

Log out:
    `exit`


# Summary

In this section of the tutorial, you provisioned the necessary VPC components to support a Kubernetes deployment.
An abbreviated collection of commands are listed here, in case you revisit this tutorial, and simply want to
start at the next step.

# Supporting Material

## The Less Hard Way (Kinda Sorta)

In case you've previously gone step-by-step through this tutorial, and simply want to reconstitute all of these
steps at once, you can do the following:

```
RG_ID=$(ibmcloud resource group-create kube-thw-ibmvpc-rg --output JSON | jq -r ".id")
VPC_ID=$(ibmcloud is vpc-create kube-thw-ibmvpc-vpc --resource-group-id $RG_ID --output JSON | jq -r ".id")

SG_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_security_group.id")
ibmcloud is security-group-update $SG_ID --name kube-thw-ibmvpc-sg
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_network_acl.id")
ibmcloud is network-acl-update $NACL_ID --name kube-thw-ibmvpc-nacl
RT_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r ".default_routing_table.id")
ibmcloud is vpc-routing-table-update $VPC_ID $RT_ID --name kube-thw-ibmvpc-rt

ibmcloud is security-group-rule-add $SG_ID inbound tcp --port-min 22 --port-max 22
ibmcloud is security-group-rule-add $SG_ID inbound icmp --icmp-type 8

PG_ID=$(ibmcloud is public-gateway-create kube-thw-ibmvpc-pg $VPC_ID us-south-1 --resource-group-id $RG_ID --output JSON | jq -r ".id")

VPC_SUBNET_ID=$(ibmcloud is subnet-create kube-thw-ibmvpc-subnet $VPC_ID --ipv4-cidr-block 10.240.0.0/24 --zone us-south-1 --resource-group-id $RG_ID --output JSON | jq -r ".id")

FIP_ID=$(ibmcloud is floating-ip-reserve kube-thw-ibmvpc-fip --zone us-south-1 --resource-group-name kube-thw-ibmvpc-rg --output JSON | jq -r ".id")
BASTION_IP=$(ibmcloud is floating-ip $FIP_ID --output JSON | jq -r ".address")

SSH_KEY_ID=$(ibmcloud is key-create kube-thw-ibmvpc-ssh-key @~/.ssh/kubethw_id_rsa.pub --resource-group-id $RG_ID --output JSON | jq -r ".id")
IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')

BASTION_ID=$(ibmcloud is instance-create kube-thw-ibmvpc-bastion $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID \
  --image-id $IMAGE_ID --key-ids $SSH_KEY_ID --security-group-ids $SG_ID --resource-group-id  $RG_ID \
  --ipv4 10.240.0.30 --output JSON | jq -r ".id")

for i in 0 1 2; do
  INST_ID=$(ibmcloud is instance-create controller-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID --image-id $IMAGE_ID \
  --key-ids $SSH_KEY_ID --security-group-ids $SG_ID --resource-group-id $RG_ID --ipv4 10.240.0.1${i} | jq -r ".id")
  declare "CTRLR${i}_ID=${INST_ID}"
done

for i in 0 1 2; do
  INST_ID=$(ibmcloud is instance-create worker-${i} $VPC_ID us-south-1 bx2-2x8 $VPC_SUBNET_ID
    --image-id $IMAGE_ID --allow-ip-spoofing true --key-ids $SSH_KEY_ID \
    --security-group-ids $SG_ID --resource-group-id $RG_ID --ipv4 10.240.0.2${i} | jq -r ".id")
  declare "WORKER${i}_ID=${INST_ID}"
done

BN_NIC_ID=$(ibmcloud is instance $BASTION_ID | jq -r ".primary_network_interface.id")
ibmcloud is instance-network-interface-floating-ip-add $BASTION_ID $BN_NIC_ID $FIP_ID

NLB_ID=$(ibmcloud is load-balancer-create kube-thw-ibmvpc-nlb public --subnet $VPC_SUBNET_ID --family network --resource-group-id $RG_ID --output JSON | jq -r ".id")

```

## Refreshing Environment Variables

If you accidentally exit a terminal, it may be painful to re-assign all variables. The following should help:

```
RG_ID=$(ibmcloud resource group kube-thw-ibmvpc-rg --output JSON | jq -r .[0].id)
VPC_ID=$(ibmcloud is vpcs --resource-group-id $RG_ID --output JSON | jq -r .[0].id) 
SG_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_security_group.id)
NACL_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_network_acl.id)
RT_ID=$(ibmcloud is vpc $VPC_ID --output JSON | jq -r .default_routing_table.id)
PG_ID=$(ibmcloud is public-gateways --resource-group-id $RG_ID --output JSON | jq -r ".[0].id")
VPC_SUBNET_ID=$(ibmcloud is subnets --resource-group-id $RG_ID --output JSON | jq -r .[0].id) 
SSH_KEY_ID=$(ibmcloud is keys --resource-group-id $RG_ID --output JSON | jq -r .[0].id)
FIP_ID=$(ibmcloud is floating-ips --resource-group-id $RG_ID --output JSON | jq -r ".[0].id")
IMAGE_ID=$(ibmcloud is images --output JSON | jq -r '.[] | select(.operating_system.name=="ubuntu-20-04-amd64") | .id')
NLB_ID=$(ibmcloud is load-balancers --resource-group-id $RG_ID --output JSON | jq -r ".[0].id")
LB_PRIVATE_IPS_ARR=($(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r '.private_ips[].address | @sh' | tr -d \'))
LB_PRIVATE_IPS=$(echo ${LB_PRIVATE_IPS_ARR[@]} | tr ' ' ',')
LB_PUBLIC_IPS_ARR=($(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r '.public_ips[].address | @sh' | tr -d \'))
LB_PUBLIC_IP=$(echo ${LB_PUBLIC_IPS_ARR[@]} | tr ' ' ',')
LB_PUBLIC_HOSTNAME=$(ibmcloud is load-balancer $NLB_ID --output JSON | jq -r .hostname)
ALL_LB_NAMES=${LB_PUBLIC_HOSTNAME},${LB_PRIVATE_IPS},${LB_PUBLIC_IP}
BASTION_ID=$(ibmcloud is instances --resource-group-id $RG_ID --output JSON | jq -r '.[] | select(.name=="bastion") | .id')
BASTION_IP=$(ibmcloud is floating-ip $FIP_ID --output JSON | jq -r ".address")
KUBERNETES_PUBLIC_ADDRESS=$LB_PUBLIC_IP
```

And to test that all variables have been set:

```
echo Resource Group ID: $RG_ID
echo VPC ID: $VPC_ID
echo Security Group ID: $SG_ID
echo Network ACL ID: $NACL_ID
echo Routing Table ID: $RT_ID
echo Public Gateway ID: $PG_ID
echo VPC Subnet ID: $VPC_SUBNET_ID
echo SSH Key ID: $SSH_KEY_ID
echo Floating IP Address ID: $FIP_ID
echo VSI Image ID: $IMAGE_ID
echo Load Balancer ID: $NLB_ID
echo Load Balancer Public IP Address: $LB_PUBLIC_IP
echo Load Balancer IPs and Hostname: $ALL_LB_NAMES
echo Bastion Host: $BASTION_ID
echo Bastion IP Address: $BASTION_IP
echo Kubernetes public address: $KUBERNETES_PUBLIC_ADDRESS
```


Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

