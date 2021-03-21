# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

**NOTE:** You can include the `-f` flag with most commands to avoid being prompted to confirm the deletion.
However, to ensure accidental deletions are minimized, the flag is not used in this tutorial.

## Environment Variables

If you accidentally exited the terminal, it may be painful to re-assign all variables. The following
commands will re-assign the variables required to clean up all resources.

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


## Compute Instances

Delete the controller and worker compute instances:

```
VPC_INSTANCES=($(ibmcloud is instances --resource-group-id $RG_ID --output JSON | jq -r '.[] | select((.name | contains("controller")) or (.name | contains("worker"))) | .id'))

for i in "${VPC_INSTANCES[@]}"; do
  ibmcloud is instance-delete $i
done
```

## SSH Key

Delete the SSH key which was previously created

`ibmcloud is key-delete $SSH_KEY_ID`

## Networking

Delete the external load balancer network resources:

----- TBD ------

Release the floating IP address previously reserved:

`ibmcloud is floating-ip-release $(ibmcloud is floating-ips --resource-group-id $RG_ID --output JSON | jq -r .[].id)`

Delete the Subnet:

`ibmcloud is subnet-delete $VPC_SUBNET_ID`

Delete the `kubernetes-the-hard-way` VPC:

`ibmcloud is vpc-delete $VPC_ID`

Delete the Resource Group:

`ibmcloud resource group-delete kube-thw-ibmvpc-rg`

If this last operation fails, it generally means that some of the resources within the group have not been deleted.
Ensure everything in the resource group has been deleted, and then repeat the command.

## Conclusion

Congratulations - you've both created and deleted your very own instance of Kubernetes. Even better, you did it
"the hard way". Nice work!