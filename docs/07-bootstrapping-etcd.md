# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on controller instance: `master`. Login to the controller instance:

```
ssh ubuntu@${MASTER_EXT_IP}
```

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.4/etcd-v3.3.4-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
tar xvf etcd-v3.3.4-linux-amd64.tar.gz
```

```
sudo mv etcd-v3.3.4-linux-amd64/etcd* /usr/local/bin/
```

### Configure the etcd Server

```
sudo mkdir -p /etc/etcd /var/lib/etcd
```

```
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

Back on local machine:

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers.
Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance.
In our case:

```
ETCD_NAME=master
```

Create the `etcd.service` systemd unit file:

```
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://$MASTER_PRIV_IP:2380 \\
  --listen-peer-urls https://$MASTER_PRIV_IP:2380 \\
  --listen-client-urls https://$MASTER_PRIV_IP:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://$MASTER_PRIV_IP:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster $ETCD_NAME=https://$MASTER_PRIV_IP:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Copy to master:
```
scp etcd.service ubuntu@${MASTER_EXT_IP}:~/
```

### Start the etcd Server

On master node:
```
sudo mv etcd.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable etcd
```

```
sudo systemctl start etcd
```

## Verification

List the etcd cluster members:

```
ETCDCTL_API=3 etcdctl member list
```

> output

```
e8e1f8fc956bbd95, started, master, https://10.0.10.11:2380, https://10.0.10.11:2379
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
