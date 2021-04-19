# Kubernetes The Hard Way - On IBM VPC

This tutorial is a fork of the great work that Kelsey Hightower originally created. And quite
frankly, all I've done is convert the Google Cloud directions into the IBM Virtual Private
Cloud (VPC) equivalent. I strongly recommend you read and star
[his repo](https://github.com/kelseyhightower/kubernetes-the-hard-way).


There are several automated ways to install and configure Kubernetes. Unfortunately, I believe
that they short-change your overall understanding of the platform. So, from a learning perspective,
it's important to grasp what automation is doing for you. Besides, rolling up our collective sleeves
and getting something working from scratch is fun. (Well, for many of us it is.)

Kubernetes The Hard Way is optimized for learning, which means taking the long route to ensure you understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that stop you from learning!

## Copyright

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.


## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

The IBM VPC fork has been created for those who don't necessarily have access to Google Cloud, but do
have access to the IBM Cloud.

## Cluster Details

Kubernetes The Hard Way guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.18.6
* [containerd](https://github.com/containerd/containerd) v1.3.6
* [coredns](https://github.com/coredns/coredns) v1.7.0
* [cni](https://github.com/containernetworking/cni) v0.8.6
* [etcd](https://github.com/coreos/etcd) v3.4.10

## Labs

This tutorial assumes you have access to the [IBM Cloud Platform](https://cloud.ibm.com/). For
a version of this tutorial for the Google Cloud Platform, please see the original work by
Kelsey Hightower: [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) 

* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)
