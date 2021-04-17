# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## Hostname Resolution

In some VPCs, there are services to automatically resolve hostnames on a subnet. To manually apply these updates, update
the file named `/etc/hosts` on each of the three controllers and the three workers. The extra lines to add are as follows:\

IP ADDRESSES
------------
10.240.0.10 controller-0
10.240.0.11 controller-1
10.240.0.12 controller-2
10.240.0.20 worker-0
10.240.0.21 worker-1
10.240.0.22 worker-2

**NOTE:** You should remove the entry associated with the current compute instance. So for example, when updating the
`hosts` file for worker-0, remove the following line from the above list: `10.240.0.20 worker-0`

After making the changes, you should be able to 'ping' each machine from the other machines using the short name / hostname.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Recall IP address and Pod CIDR range for each worker instance:

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

Create network routes for each worker instance:

```
ibmcloud is vpc-routing-table-route-create $VPC_ID $RT_ID -zone us-south-1 --destination 10.200.0.0/24 --next-hop 10.240.0.20
ibmcloud is vpc-routing-table-route-create $VPC_ID $RT_ID -zone us-south-1 --destination 10.200.1.0/24 --next-hop 10.240.0.21
ibmcloud is vpc-routing-table-route-create $VPC_ID $RT_ID -zone us-south-1 --destination 10.200.2.0/24 --next-hop 10.240.0.22
```

List the routes in the `kubernetes-the-hard-way` VPC network:

```
ibmcloud is vpc-routing-table-routes $VPC_ID $RT_ID
```

> output

All three custom routes should be displayed.

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
