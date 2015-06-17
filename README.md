# vagrant-coreos-cluster

A Vagrant project to setup a local CoreOS cluster with:

- `etcd2`
- `fleet`
- `flannel`

## Usage

Bringing up the cluster can be accomplished with:

```
$ vagrant up
```

From there, ensure `fleet` can see all of the nodes in the cluster:

```
$ vagrant ssh core-01
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0b5a65ce...	172.17.8.102	-
2f2dc9c5...	172.17.8.103	-
ad0c27da...	172.17.8.101	-
```

Lastly, we can test the `flannel` overlay network:

```
$ vagrant ssh core-01
core@core-01 ~ $ docker run -ti debian:jessie /bin/bash
root@32d21349c674:/# ip addr s eth0
6: eth0: <BROADCAST,UP,LOWER_UP> mtu 1472 qdisc noqueue state UP group default
    link/ether 02:42:0a:01:14:02 brd ff:ff:ff:ff:ff:ff
    inet 10.1.20.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe01:1402/64 scope link
       valid_lft forever preferred_lft forever
```

Then `ping` the container above from `core-02`:

```
$ vagrant ssh core-02
core@core-02 ~ $ docker run -ti debian:jessie /bin/bash
root@d5010a9b4267:/# ping -c1 10.1.20.2
PING 10.1.20.2 (10.1.20.2): 56 data bytes
64 bytes from 10.1.20.2: icmp_seq=0 ttl=60 time=0.858 ms
--- 10.1.20.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.858/0.858/0.858/0.000 ms
```
