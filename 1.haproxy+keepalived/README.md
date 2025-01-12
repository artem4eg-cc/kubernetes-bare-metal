### keepalived
```bash
root@haproxy-server1:~# apt -y install keepalived

root@haproxy-server1:~# vi /etc/keepalived/keepalived.conf

global_defs {
    # set hostname
    router_id haproxy-server1
}

vrrp_instance VRRP1 {
    state MASTER
    interface enp1s0
    virtual_router_id 101
    priority 200
    advert_int 1
    virtual_ipaddress {
        192.168.124.100/24
    }
}

root@haproxy-server1:~# systemctl restart keepalived

root@haproxy-server1:~# ip address show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:55:f7:7b brd ff:ff:ff:ff:ff:ff
    inet 192.168.124.83/24 metric 100 brd 192.168.124.255 scope global dynamic enp1s0
       valid_lft 2610sec preferred_lft 2610sec
    inet 192.168.124.100/24 scope global secondary enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe55:f77b/64 scope link 
       valid_lft forever preferred_lft forever
```
```bash
root@haproxy-server2:~# apt -y install keepalived

root@haproxy-server2:~# vi /etc/keepalived/keepalived.conf

global_defs {
    router_id haproxy-server2
}

vrrp_instance VRRP1 {
    state BACKUP
    interface enp1s0
    virtual_router_id 101
    priority 100
    advert_int 1
    virtual_ipaddress {
        192.168.124.100/24
    }
}

root@haproxy-server2:~# systemctl restart keepalived
```
```bash
root@haproxy-server1:~# ip address show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:55:f7:7b brd ff:ff:ff:ff:ff:ff
    inet 192.168.124.83/24 metric 100 brd 192.168.124.255 scope global dynamic enp1s0
       valid_lft 2386sec preferred_lft 2386sec
    inet 192.168.124.100/24 scope global secondary enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe55:f77b/64 scope link 
       valid_lft forever preferred_lft forever

root@haproxy-server1:~# ip link set down enp1s0

root@haproxy-server2:~# ip address show enp1s0
```
### HAProxy
```bash
root@haproxy-server1:~# apt -y install haproxy

root@haproxy-server1:~# vi /etc/haproxy/haproxy.cfg

# add
listen kubernetes-apiserver-https
  bind *:8383
  mode tcp
  option log-health-checks
  timeout client 3h
  timeout server 3h
  server node1 192.168.124.201:6443 check check-ssl verify none inter 10000
  server node2 192.168.124.202:6443 check check-ssl verify none inter 10000
  server node3 192.168.124.203:6443 check check-ssl verify none inter 10000
  balance roundrobin
```
```bash
root@haproxy-server2:~# apt -y install haproxy

root@haproxy-server2:~# vi /etc/haproxy/haproxy.cfg

# add
listen kubernetes-apiserver-https
  bind *:8383
  mode tcp
  option log-health-checks
  timeout client 3h
  timeout server 3h
  server node1 192.168.124.201:6443 check check-ssl verify none inter 10000
  server node2 192.168.124.202:6443 check check-ssl verify none inter 10000
  server node3 192.168.124.203:6443 check check-ssl verify none inter 10000
  balance roundrobin
```