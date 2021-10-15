Joining the worker nodes
===

Joining the worker nodes to the cluster is the most simple task. You only need to configure CRI and join the node after that:

```
kubeadm join 10.1.1.90:6443 --token cua9c5.omefe9k36sx7k76p \
    --discovery-token-ca-cert-hash sha256:1a0440f749c8bb34bef9b6a1b0ca8957d8811bb95f84e2b2e0bf8e831e4e589b
```

After all control plane and worker nodes have been joined to the cluster you can verify your cluster:

```
kubectl get nodes -o wide
NAME   STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
m1     Ready    master   6d21h   v1.19.3   10.1.1.99     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
m2     Ready    master   6d21h   v1.19.3   10.1.1.98     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
m3     Ready    master   6d20h   v1.19.3   10.1.1.97     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
n1     Ready    <none>   6d20h   v1.19.3   10.1.1.96     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
n2     Ready    <none>   6d20h   v1.19.3   10.1.1.95     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
n3     Ready    <none>   6d20h   v1.19.3   10.1.1.94     <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-8-amd64   containerd://1.4.11
```

The kubernetes cluster now is fully operational! Congratulations!