Load balancer setup
===

## HAProxy installation

In this tutorial we will use `HAProxy` as a load balancer. It is fast, reliable and easy to configure. `HAProxy` packages are [available](https://haproxy.debian.net/) for Debian and Ubuntu. For example, add `HAProxy` repo and install it on Debian 11:

```
curl https://haproxy.debian.net/bernat.debian.org.gpg \
      | gpg --dearmor > /usr/share/keyrings/haproxy.debian.net.gpg

echo deb "[signed-by=/usr/share/keyrings/haproxy.debian.net.gpg]" \
      http://haproxy.debian.net bullseye-backports-2.4 main \
      > /etc/apt/sources.list.d/haproxy.list

apt-get update
apt-get install haproxy=2.4.\*
```
Verify `HAProxy` is installed correctly:

```
haproxy version

HAProxy version 2.4.7-1~bpo11+1 2021/10/07 - https://haproxy.org/
Status: long-term supported branch - will stop receiving fixes around Q2 2026.
Known bugs: http://www.haproxy.org/bugs/bugs-2.4.7.html
Running on: Linux 5.10.0-8-amd64 #1 SMP Debian 5.10.46-4 (2021-08-03) x86_64
```

## HAProxy configuration

Configuration of `HAProxy` is very simple: since kubernetes api will serve requests via https we only need to set up tcp load balancing. Append these lines to the default config:

```
frontend kube-api
  bind 10.1.1.90:6443
  option tcplog
  mode tcp
  default_backend kubernetes

backend kubernetes
  mode tcp
  balance roundrobin
  server master1 10.1.1.99:6443
  server master2 10.1.1.98:6443
  server master3 10.1.1.97:6443
```

The first block describes address and port for haproxy to listen to, mode and a default backend. The second block decribes `kubernetes` backend, which consists of three controller-manager nodes. Balance type is roundrobin.

Now start and verify that haproxy is running:

```
systemctl start haproxy.service

systemctl status haproxy service

● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2021-10-13 12:31:39 +07; 5s ago
```