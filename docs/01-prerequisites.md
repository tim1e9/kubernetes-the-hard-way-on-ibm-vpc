# Prerequisites

## IBM Cloud Platform

This tutorial leverages the [IBM Cloud Platform](https://cloud.ibm.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://cloud.ibm.com/registration) to get an account.

There are costs associated with this tutorial. However, the costs can potentially be mitigated by reviewing
any special promotions or offers available. As of this writing, there's a $500 promotion which should more than
cover the costs. Additional information can be found [On the IBM Cloud VPC Pricing site](https://www.ibm.com/cloud/vpc/pricing).


## IBM Cloud Command Line Interface (CLI)

### Install the CLI

To get started, it's necessary to download and install the IBM Cloud Command Line Interface (CLI). Detailed instructions can be found [here](https://cloud.ibm.com/docs/cli?topic=cli-getting-started).

Briefly, the steps to follow are:
1. Download a shell script which pulls and installs the code
2. Verify the CLI has been installed: In a terminal, type `ibmcloud -v`
3. View the plugins that have been installed as part of the base installation: `ibmcloud plugin list`
   If you don't see "vpc-infrastructure", then you'll need to install it:
   - `ibmcloud plugin install vpc-infrastructure`
   - `ibmcloud plugin update`
   - `ibmcloud plugin list`
   You should now see the VPC plugin in the list.

   Below is sample output from running the `ibmcloud plugin list` command:
    ```
    $ ibmcloud plugin list
    Listing installed plug-ins...

    Plugin Name                                 Version   Status   Private endpoints supported
    cloud-functions/wsk/functions/fn            1.0.49             false
    cloud-object-storage                        1.2.2              false
    container-registry                          0.1.514            false
    container-service/kubernetes-service        1.0.233            false
    vpc-infrastructure/infrastructure-service   0.7.9              false
    ```
4. Log in to the cloud via either traditional credentials or via SSO:
   - traditional: `ibmcloud login`
   - via SSO: `ibmcloud login -sso`
5. As part of the login process, select an appropriate account. If you only have a single account,
   then you may not be prompted to select one.
6. The login process will also ask you to select a region. Be sure to select one that's close to
   you. This tutorial will assume you're using `us-south`.

### Specify a Cloud Region and Zone

Although the region can be selected as part of the login process, it's also possible to change
the region you're using after logging in. To list all available regions:

`ibmcloud is regions`

To select a default region, such as us-south:

`ibmcloud is region us-south`

Similarly, to see all availability zones within a region, use the following command:

`ibmcloud is zones`

And to set a default zone, such as us-south-1, use the following command:

`ibmcloud is zone us-south-1`

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with synchronize-panes enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
