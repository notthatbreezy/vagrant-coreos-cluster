# vagrant-coreos-cluster

A Vagrant project to setup a local CoreOS cluster with:

- etcd
- Fleet
- Flannel
- Kubernetes

## Usage

Bringing up the CoreOS cluster can be accomplished with:

```
$ vagrant up
```

### etcd Test

To ensure etcd is clustered:

```
$ vagrant ssh core-01
core@core-01 ~ $ etcdctl member list
25d6ce33763c5524: name=core-02 peerURLs=http://172.17.8.102:2380 clientURLs=http://172.17.8.102:2379
6ae27f9fa2984b1d: name=core-01 peerURLs=http://172.17.8.101:2380 clientURLs=http://172.17.8.101:2379
ff32f4b39b9c47bd: name=core-03 peerURLs=http://172.17.8.103:2380 clientURLs=http://172.17.8.103:2379
```

### Fleet Test

To ensure Fleet can see all of the nodes in the cluster:

```
$ vagrant ssh core-01
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
1150400f...	172.17.8.102	role=follower
bc859c8f...	172.17.8.101	role=leader
cd9c1a72...	172.17.8.103	role=follower
```

### Flannel Test

To test the Flannel overlay network:

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

### Kubernetes Test

Testing Kubernetes is a bit more involved. First, launch the following
Kubernetes services using `fleetctl` from the virtual machine host:

```
$ fleetctl --tunnel 127.0.0.1:2222 start kube-apiserver
$ fleetctl --tunnel 127.0.0.1:2222 start kube-controller-manager
$ fleetctl --tunnel 127.0.0.1:2222 start kube-proxy
$ fleetctl --tunnel 127.0.0.1:2222 start kube-scheduler
$ fleetctl --tunnel 127.0.0.1:2222 start kubelet
```

Then, make sure all of them are running across the cluster:

```
$ fleetctl --tunnel 127.0.0.1:2222 list-units
UNIT				MACHINE				ACTIVE	SUB
kube-apiserver.service		e6b1f713.../172.17.8.101	active	running
kube-controller-manager.service	e6b1f713.../172.17.8.101	active	running
kube-proxy.service		441385c5.../172.17.8.102	active	running
kube-proxy.service		e6b1f713.../172.17.8.101	active	running
kube-proxy.service		f37d5882.../172.17.8.103	active	running
kube-scheduler.service		e6b1f713.../172.17.8.101	active	running
kubelet.service			441385c5.../172.17.8.102	active	running
kubelet.service			e6b1f713.../172.17.8.101	active	running
kubelet.service			f37d5882.../172.17.8.103	active	running
```

Let's also make sure that Kubernetes recognizes all of the nodes in the cluster:

```
$ kubectl -s 172.17.8.101:8080 get nodes
NAME           LABELS                                STATUS
172.17.8.101   kubernetes.io/hostname=172.17.8.101   Ready
172.17.8.102   kubernetes.io/hostname=172.17.8.102   Ready
172.17.8.103   kubernetes.io/hostname=172.17.8.103   Ready
```

Lastly, let's create an instance of the `memcached` container:

```
$ kubectl run-container -s 172.17.8.101:8080 memcached --image=memcached
```

And then scale it out to `5` replicas:

```
$ kubectl -s 172.17.8.101:8080 resize --replicas=5 rc memcached
```

Finally, let's try to interact with one of the `memcached` instances:

```
❯ vagrant ssh core-01
core@core-01 ~ $ echo "stats" | ncat 10.1.27.2 11211
STAT pid 1
STAT uptime 112
STAT time 1434688429
STAT version 1.4.24
STAT libevent 2.0.21-stable
STAT pointer_size 64
STAT rusage_user 0.004000
STAT rusage_system 0.012000
STAT curr_connections 10
STAT total_connections 11
STAT connection_structures 11
STAT reserved_fds 20
STAT cmd_get 0
STAT cmd_set 0
STAT cmd_flush 0
STAT cmd_touch 0
STAT get_hits 0
STAT get_misses 0
STAT delete_misses 0
STAT delete_hits 0
STAT incr_misses 0
STAT incr_hits 0
STAT decr_misses 0
STAT decr_hits 0
STAT cas_misses 0
STAT cas_hits 0
STAT cas_badval 0
STAT touch_hits 0
STAT touch_misses 0
STAT auth_cmds 0
STAT auth_errors 0
STAT bytes_read 6
STAT bytes_written 0
STAT limit_maxbytes 67108864
STAT accepting_conns 1
STAT listen_disabled_num 0
STAT threads 4
STAT conn_yields 0
STAT hash_power_level 16
STAT hash_bytes 524288
STAT hash_is_expanding 0
STAT malloc_fails 0
STAT bytes 0
STAT curr_items 0
STAT total_items 0
STAT expired_unfetched 0
STAT evicted_unfetched 0
STAT evictions 0
STAT reclaimed 0
STAT crawler_reclaimed 0
STAT crawler_items_checked 0
STAT lrutail_reflocked 0
END
```
