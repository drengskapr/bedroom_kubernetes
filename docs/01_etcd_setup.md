etcd setup
===

`etcd` is key-value store where the kubernetes cluster will store its system data. There are different approaches in etcd installation: containerized setup and classic systemd service setup. In this tutorial the latter is described. Refer to [official documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/) on how to set up external etcd cluster using containers.

### etcd installation

`etcd` does not depend on kubernetes version so the latest release can be installed. Download and install etcd on every etcd node:

```
curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
rm -f etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf etcd-${ETCD_VER}-linux-amd64
```
verify etcd installation:

```
etcd -version
etcd Version: 3.4.17
Git SHA: 19e2e70e4
Go Version: go1.12.17
Go OS/Arch: linux/amd64

etcdctl version
etcdctl version: 3.4.17
API version: 3.4
```

### Creation of certificates

After etcd is properly installed let us generate the necessary certificates. First of all we need to generate kubeadm config file for every etcd node. This script creates the config files:

```bash
#!/bin/bash
# Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
export HOST0=10.1.1.93
export HOST1=10.1.1.92
export HOST2=10.1.1.91

# Create temp directories to store files that will end up on other hosts.
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("e1" "e2" "e3")

for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
apiVersion: "kubeadm.k8s.io/v1beta2"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```
create the bash script on `e1` and run it.

Since we do not yet have a certificate authority we must generate it also. Run this command on `e1`:

```
kubeadm init phase certs etcd-ca
```

This creates two files

* `/etc/kubernetes/pki/etcd/ca.crt`
* `/etc/kubernetes/pki/etcd/ca.key`

Now create certificates for each etcd node. Run this script on `e1`:

```bash
#!/bin/bash
# Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
export HOST0=10.1.1.93
export HOST1=10.1.1.92
export HOST2=10.1.1.91

kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No need to move the certs because they are for HOST0

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```
After all the required certificates have been created it is time to move them to `e2` and `e3`. Consider using `scp`.

Here is the list of the certificates on every etcd node:

```
/etc/kubernetes/pki/
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── ca.key
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

We do not need `kubeadmcfg.yaml` anymore because `etcd` is not containerized.

### Creation of systemd unit files

Now let us create etcd systemd unit file. Example unit file for `e1`:

```
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
User=root
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=e1 \
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \
  --key-file=/etc/kubernetes/pki/etcd/server.key \
  --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt \
  --peer-key-file=/etc/kubernetes/pki/etcd/peer.key \
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
  --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
  --peer-client-cert-auth=true \
  --client-cert-auth=true \
  --initial-advertise-peer-urls=https://10.1.1.93:2380 \
  --listen-peer-urls=https://10.1.1.93:2380 \
  --listen-client-urls=https://10.1.1.93:2379,https://127.0.0.1:2379 \
  --listen-metrics-urls=http://127.0.0.1:2381 \
  --advertise-client-urls=https://10.1.1.93:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster e1=https://10.1.1.93:2380,e2=https://10.1.1.92:2380,e3=https://10.1.1.91:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Adapt this unit file for the rest of the nodes. Only listen ip-addresses will differ.

Start etcd service:

```
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```
**Note:** since `etcd` is a leader-based store you must start etcd on all nodes at once or at least within a timeout (50s) for the cluster to properly bootstrap.

Verify the `etcd` cluster:

```
ETCDCTL_API=3 etcdctl member list   --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key

5db3b4a291cbd677, started, e1, https://10.1.1.93:2380, https://10.1.1.93:2379, false
b13efc447ce19dd5, started, e2, https://10.1.1.92:2380, https://10.1.1.92:2379, false
fec755c44e5918de, started, e3, https://10.1.1.91:2380, https://10.1.1.91:2379, false

ETCDCTL_API=3 etcdctl endpoint health --cluster   --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key

https://10.1.1.93:2379 is healthy: successfully committed proposal: took = 14.340346ms
https://10.1.1.92:2379 is healthy: successfully committed proposal: took = 18.824237ms
https://10.1.1.91:2379 is healthy: successfully committed proposal: took = 21.93488ms
```