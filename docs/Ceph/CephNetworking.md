# Networking  


## Baremetal Lab Network Bonds

```
nmcli conn add type bond con-name bond1 ifname bond1 bond.options "mode=802.3ad,miimon=100"
nmcli conn add type ethernet slave-type bond con-name bond-ens4f0 ifname ens4f0 master bond1
nmcli conn add type ethernet slave-type bond con-name bond-ens4f1 ifname ens4f1 master bond1
nmcli conn add type vlan con-name vlan1100 dev bond1 id 1100 ip4 172.20.0.11/24 gw4 172.20.0.1 
nmcli conn add type vlan con-name vlan1101 dev bond1 id 1101 ip4 172.20.1.21/24 
nmcli conn del ens4f0
nmcli conn del ens4f1
nmcli conn mod bond1 ipv4.method disabled
nmcli conn mod bond1 ipv6.method disabled
nmcli conn mod bond1 802-3-ethernet.mtu 9000
nmcli conn mod vlan1100 ipv4.dns 172.20.129.10
nmcli conn mod vlan1101 ipv4.dns 172.20.129.10
nmcli conn mod bond1 bond.options 'mode=4,lacp_rate=fast,updelay=1000,miimon=100,xmit_hash_policy=layer3+4'
nmcli conn reload
nmcli networking off && nmcli networking on
```


## Routes

Add a route to *network* via *gateway* on device *interface*
```
ip route add 10.20.0.0/24 via 10.40.0.1 dev eth1
```

Using NMCLI:
```
nmcli conn modify eth1 _ipv4.routes "10.20.0.0/24 10.40.0.1"
```

