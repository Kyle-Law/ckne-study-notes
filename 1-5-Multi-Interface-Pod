# 1.5 Multi Interface Pod

https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md

1. [Prep] (Cilium CNI) Disable cni.exclusive

`cilium config view | grep cni`
`cilium upgrade --reuse-values --set cni.exclusive=false`

2. [Prep] `ip a` - check interface name on the hosts in your cluster

```
root@controlplane:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc fq_codel state UP group default qlen 1000
    link/ether f2:a9:78:f9:c5:c1 brd ff:ff:ff:ff:ff:ff
    inet 172.30.1.2/24 brd 172.30.1.255 scope global dynamic noprefixroute enp1s0
       valid_lft 86311880sec preferred_lft 75522680sec
    inet6 fe80::848a:ecf9:1697:5aab/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1454 qdisc noqueue state DOWN group default 
...
4: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default 
...
5: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default qlen 1000
...
```

In this case, it's enp1s0

3. Apply Multus Daemonset

> Multus is a CNI Meta Plugin, standard for multi-interface pod

`kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml`

check if ds pod are ready, and `sudo ls /etc/cni/net.d` to check if there's a multus config file

4. Install `NetworkAttachmentDefinition`, a CRD installed along with step 3

> Remember to change master: "eth0" below to `master: enp1s0`, which enp1s0 is the interface you found from step 2

```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```

5. Setup sample pod, and observe the multi interfaces

root@controlplane:~$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
EOF
pod/samplepod created
root@controlplane:~$ cilium config^C
root@controlplane:~$ k get po
NAME        READY   STATUS    RESTARTS   AGE
samplepod   1/1     Running   0          5s
root@controlplane:~$ k run sp2 --image=alpine -- sleep infinity
pod/sp2 created
root@controlplane:~$ k get po
NAME        READY   STATUS    RESTARTS   AGE
samplepod   1/1     Running   0          21s
sp2         1/1     Running   0          3s
root@controlplane:~$ k exec samplepod -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: net1@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP qlen 1000
    link/ether de:30:01:8c:46:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.200/24 brd 192.168.1.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::dc30:1ff:fe8c:46e4/64 scope link 
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1400 qdisc noqueue state UP 
    link/ether 1e:d2:9d:30:4e:54 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.67/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1cd2:9dff:fe30:4e54/64 scope link 
       valid_lft forever preferred_lft forever
root@controlplane:~$ k get p^C
root@controlplane:~$ k exec sp2 -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1400 qdisc noqueue state UP 
    link/ether 02:81:a3:f1:7d:43 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.167/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::81:a3ff:fef1:7d43/64 scope link 
       valid_lft forever preferred_lft forever

