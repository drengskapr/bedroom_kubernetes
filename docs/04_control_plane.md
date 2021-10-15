Joining the rest of control plane nodes
===

Before joining control plane node to the cluster CRI must be properly configured as described in the previous chapter.

After that let us copy required certificates from the first control plane node `m1` to the second control plane node `m2`:

```
scp /etc/kubernetes/pki/ca.crt root@10.1.1.98:/etc/kubernetes/pki/ca.crt
scp /etc/kubernetes/pki/ca.key root@10.1.1.98:/etc/kubernetes/pki/ca.key
scp /etc/kubernetes/pki/sa.key root@10.1.1.98:/etc/kubernetes/pki/sa.key
scp /etc/kubernetes/pki/sa.pub root@10.1.1.98:/etc/kubernetes/pki/sa.pub
scp /etc/kubernetes/pki/front-proxy-ca.crt root@10.1.1.98:/etc/kubernetes/pki/front-proxy-ca.crt
scp /etc/kubernetes/pki/front-proxy-ca.key root@10.1.1.98:/etc/kubernetes/pki/front-proxy-ca.key
scp /etc/kubernetes/pki/apiserver-etcd-client.crt root@10.1.1.98:/etc/kubernetes/pki/apiserver-etcd-client.crt
scp /etc/kubernetes/pki/apiserver-etcd-client.key root@10.1.1.98:/etc/kubernetes/pki/apiserver-etcd-client.key
scp /etc/kubernetes/pki/etcd/ca.crt root@10.1.1.98:/etc/kubernetes/pki/etcd/ca.crt
```

Then join the node with control plane role:

```
kubeadm join 10.1.1.90:6443 --token cua9c5.omefe9k38sx7k76p \
    --discovery-token-ca-cert-hash sha256:1a0440f749c8bb34bef9b6a2b0ca8957d8811bb95f84e2b2e0bf8e831e4e589b \
    --control-plane
```
Now verify that `m2` was added to the cluster using kubectl:

```
kubectl get nodes
NAME   STATUS   ROLES    AGE  VERSION
m1     Ready    master   1h   v1.19.3
m2     Ready    master   1h   v1.19.3
```

Repeat the above steps for `m3`.