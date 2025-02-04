# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

## Background - Connecting to Compute Resources

Because we haven't provided public IP addresses for all of the servers, they won't be accessible over the internet.
To get around this, we've assigned a public IP address to the a special instance - the Bastion host. We'll use this
host as a "jump machine" to access the rest of the resources in the subnet.

The IP address of the Bastion Host should be assigned to the environment variable `$BASTION_IP`.

To test the connection, use the following command:

`ssh -i ~/.ssh/kubethw_id_rsa root@$BASTION_IP`

You may be prompted to verify the authenticity of the connection. Once verified, you should be logged in. Once
you've become familiar with the server, exit the shell.

Next, we'll verify access to a worker node. You may recall that we set the IP address of worker-0 to 10.240.0.20.
We'll use that information to "Jump" from the public machine to the other (non-public) machines.

To test Jump connectivity, use the following command:

`ssh -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" root@10.240.0.20`

After verifying the authenticity of the connection, you should be logged into the worker-0 VSI. After verifying access,
exit from the shell.

**NOTE:** If you perform these steps after deleting and recreating the machines, you may get an error like the following:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
```

Generally, removing the old entry from `~/.ssh/known_hosts` will fix this situation.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

You may want to create a special directory to store all keys, (e.g. `mkdir keys`).

Generate the CA configuration file, certificate, and private key:

```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

Config files created:

```
ca-config.json  (Used for future commands)
ca-csr.json
```

Files generated:

```
ca-key.pem
ca.pem
ca.csr (not needed)
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Generate the `admin` client certificate and private key:

```
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

Config files created:

```
admin-csr.json
```

Files generated:

```
admin-key.pem
admin.pem
admin.csr (Not used)
```

### The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:

```
IP_COUNTER=0
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

INTERNAL_IP=10.240.0.2$IP_COUNTER
let IP_COUNTER+=1

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Config files created:

```
worker-0-csr.json
worker-1-csr.json
worker-2-csr.json
```

Files generated:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
worker-0.csr (Not used)
worker-1.csr (Not used)
worker-1.csr (Not used)
```

### The Controller Manager Client Certificate

Generate the `kube-controller-manager` client certificate and private key:

```
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

Config files created:

```
kube-controller-manager-csr.json
```

Files generated:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
kube-controller-manager.csr (Not used)
```


### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:

```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

Config files created:

```
kube-proxy-csr.json
```

Files generated:

```
kube-proxy-key.pem
kube-proxy.pem
kube-proxy.csr (Not used)
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

Config files created:

```
kube-scheduler-csr.json
```

Files generated:

```
kube-scheduler-key.pem
kube-scheduler.pem
kube-scheduler.csr (Not used)
```


### The Kubernetes API Server Certificate

The `kubernetes-the-hard-way` static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:

**NOTE:** In the previous step, the environment variable `$ALL_LB_NAMES` was defined. If this variable is missing,
refer to the Supporting Material in step 3.

```
{
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${ALL_LB_NAMES},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> The Kubernetes API server is automatically assigned the `kubernetes` internal dns name, which will be linked to the first IP address (`10.32.0.1`) from the address range (`10.32.0.0/24`) reserved for internal cluster services during the [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) lab.

Config files created:

```
kubernetes-csr.json
```

Files generated:

```
kubernetes-key.pem
kubernetes.pem
kubernetes.csr (Not used)
```

## The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the `service-account` certificate and private key:

```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

Config files created:

```
service-account-csr.json
```

Files generated:

```
service-account-key.pem
service-account.pem
service-account.csr (Not used)
```

## Distribute the Client and Server Certificates


Copy the appropriate certificates and private keys to each worker instance:

```
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem worker-0-key.pem worker-0.pem root@10.240.0.20:~/
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem worker-1-key.pem worker-1.pem root@10.240.0.21:~/
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem worker-2-key.pem worker-2.pem root@10.240.0.22:~/
```

Copy the appropriate certificates and private keys to each controller instance:

```
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem root@10.240.0.10:~/
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem root@10.240.0.11:~/
scp -i ~/.ssh/kubethw_id_rsa -o ProxyCommand="ssh -i ~/.ssh/kubethw_id_rsa -W %h:%p root@$BASTION_IP" ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem root@10.240.0.12:~/

```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
