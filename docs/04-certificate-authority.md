# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kubelet, and kube-proxy.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create the CA configuration file:

```
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
```

Create the CA certificate signing request:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "London",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "London"
    }
  ]
}
EOF
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### Set hostnames

Master:
```
ssh ubuntu@${MASTER_EXT_IP}
sudo hostname master
```

Worker:
```
ssh ubuntu@${WORKER_EXT_IP}
sudo hostname worker
```

### The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for worker node:

```
cat > worker-csr.json <<EOF
{
  "CN": "system:node:worker",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "London",
      "O": "system:nodes",
      "OU": "CA",
      "ST": "London"
    }
  ]
}
EOF

export WORKER_PRIV_IP=`aws ec2 describe-instances --instance-ids ${WORKER_ID} --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text`

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker,${WORKER_EXT_IP},${WORKER_PRIV_IP} \
  -profile=kubernetes \
  worker-csr.json | cfssljson -bare worker
```

Results:

```
worker-key.pem
worker.pem
```

### The kube-proxy Client Certificate

Create the `kube-proxy` client certificate signing request:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "London",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "London"
    }
  ]
}
EOF
```

Generate the `kube-proxy` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Results:

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Kubernetes API Server Certificate

Create the Kubernetes API Server certificate signing request:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "London",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "London"
    }
  ]
}
EOF
```

Generate the Kubernetes API Server certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${MASTER_PRIV_IP},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.pem
```

## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to worker instance:

```
scp ca.pem worker-key.pem worker.pem ubuntu@${WORKER_EXT_IP}:~/
```

Copy the appropriate certificates and private keys to controller instance:

```
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem ubuntu@${MASTER_EXT_IP}:~/
```

> The `kube-proxy` and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
