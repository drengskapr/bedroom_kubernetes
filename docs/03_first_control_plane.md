Bootstrapping the first control plane node
===

In this chapter we will configure CRI for the first control plane node and bootstrap it.

### Configuring CRI

As mentioned earlier each k8s node must have `containerd`,`kubeadm`,`kubelet` installed. Container runtime (which is containerd in our case) needs additional configuration.

Create a file for containerd with required kernel modules and activate the modules:

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Setup required sysctl params and apply them:

```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```
If containerd does not have a config file yet, then create it:

```
containerd config default | sudo tee /etc/containerd/config.toml
```

Change containerd config file to use `systemd` cgroup driver (which is recommended):

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
Then restart containerd:

```
sudo systemctl restart containerd.service
```

Kubelet service must have the same cgroup driver as CRI. Add `cgroup-driver` option to the kubelet.service file:

```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cgroup-driver=systemd
```
Now restart kubelet service:

```
sudo systemctl restart kubelet.service
```

**Note:** You should repeat the above steps on each control plane and worker node.

### Initializing a kubernetes cluster

Since CA was already created when we have set up `etcd` cluster we need only to copy certificates from any etcd node to the first control plane node (let it be `m1`):

```
scp /etc/kubernetes/pki/etcd/ca.crt root@10.1.1.99:/etc/kubernetes/pki/etcd/ca.crt
scp /etc/kubernetes/pki/apiserver-etcd-client.crt root@10.1.1.99:/etc/kubernetes/pki/apiserver-etcd-client.crt
scp /etc/kubernetes/pki/apiserver-etcd-client.key root@10.1.1.99:/etc/kubernetes/pki/apiserver-etcd-client.key
```
Directories `/etc/kubernetes/pki/` and `/etc/kubernetes/pki/etcd` should be created beforehand.

Create a file called `kubeadm-config.yaml` with the following contents:

```
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "10.1.1.90:6443"
etcd:
    external:
        endpoints:
        - https://10.1.1.93:2379
        - https://10.1.1.92:2379
        - https://10.1.1.91:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
networking:
  dnsDomain: cluster.local
  podSubnet: 10.200.0.0/16
  serviceSubnet: 10.32.0.0/12
```
where `etcd` directive describes external cluster which was created earlier. The `networking` directive describes `dnsDomain` along with `podSubnet` and `serviceSubnet`. Setting of `podSubnet` is very important because `kubeadm` does not have a default value for this option and without it pod network cannot be deployed!

Now run

```
sudo kubeadm init --config kubeadm-config.yaml
```
and wait for the first control plane node to initialize. The output shoud be like this:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.1.1.90:6443 --token cua9c5.omefe9k38sx7k76p \
    --discovery-token-ca-cert-hash sha256:1a0440f749c8bb34bef9b6a2b0ca8957d8811bb95f84e2b2e0bf8e831e4e589b \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.1.90:6443 --token cua9c5.omefe9k38sx7k76p \
    --discovery-token-ca-cert-hash sha256:1a0440f749c8bb34bef9b6a2b0ca8957d8811bb95f84e2b2e0bf8e831e4e589b
```

## Deploying CNI plugin

Before joining more nodes we need to deploy a CNI plugin which will create us the pod network. In this tutorial I will use [Flannel](https://github.com/flannel-io/flannel), but any other CNI plugin which is compatible with bare metal will fit.

Deploying flannel is easy: just apply a [config file](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml) using `kubectl`. In our case we need to change flannel's network option to match with cluster's `podSubnet`:

```
  net-conf.json: |
    {
      "Network": "10.200.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

Now apply the yaml-file:

```
export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f kube-flannel.yml
```

After that `flannel` will create pod network. How to verify that it works as intended? You can check status of `coreDNS` pods since they cannot work without pod network:

```
kubectl -n kube-system get po -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP          NODE
coredns-f9fd979d6-5b5m4      1/1     Running   4          15m   10.200.0.8    m1  
coredns-f9fd979d6-72k2h      1/1     Running   4          15m   10.200.0.11   m1  
```
Not only their status is `running` but they also got addresses from the pod network!