# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

**NOTE:** You can include the `-f` flag with most commands to avoid being prompted to confirm the deletion.
However, to ensure accidental deletions are minimized, the flag is not used in this tutorial.

## Environment Variables

If you accidentally exited the terminal, it may be painful to re-assign all variables. Fortunately,
there's a way to recreate all of them. [See - Refreshing Environment Variables](./03-compute-resources.md)


## Compute Instances

Delete the controller and worker compute instances:

```
VPC_INSTANCES=($(ibmcloud is instances --resource-group-id $RG_ID --output JSON | jq -r '.[] | select((.name | contains("controller")) or (.name | contains("worker"))) | .id'))

for i in "${VPC_INSTANCES[@]}"; do
  ibmcloud is instance-delete $i
done

ibmcloud is instance-delete $BASTION_ID
```

## SSH Key

Delete the SSH key which was previously created

`ibmcloud is key-delete $SSH_KEY_ID`

## Networking

Delete the external load balancer network resources:

`ibmcloud is load-balancer-delete $NLB_ID`

Release the floating IP address previously reserved:

`ibmcloud is floating-ip-release $FIP_ID`

Delete the Subnet:

`ibmcloud is subnet-delete $VPC_SUBNET_ID`

Delete the Public Gateway:

`ibmcloud is public-gateway-delete $PG_ID`

Delete the `kubernetes-the-hard-way` VPC:

`ibmcloud is vpc-delete $VPC_ID`

Delete the Resource Group:

`ibmcloud resource group-delete kube-thw-ibmvpc-rg`

If this last operation fails, it generally means that some of the resources within the group have not been deleted.
Ensure everything in the resource group has been deleted, and then repeat the command.

## Finding lost Resources

If you encounter an error deleting the resource group, it's possible that some resources were not deleted.
The following commands will attempt to find any "dangling" resources associated with the resource group.

```
ibmcloud is floating-ips --resource-group-id $RG_ID
ibmcloud is keys --resource-group-id $RG_ID
ibmcloud is subnets --resource-group-id $RG_ID
ibmcloud is vpcs --resource-group-id $RG_ID
ibmcloud is security-groups --resource-group-id $RG_ID
ibmcloud is network-acls --resource-group-id $RG_ID
ibmcloud is instances --resource-group-id $RG_ID
```

## Conclusion

Congratulations - you've both created and deleted your very own instance of Kubernetes. Even better, you did it
"the hard way". Nice work!