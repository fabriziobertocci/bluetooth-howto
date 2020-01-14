# Network over bluetooth

Once you have a computer paired with the Raspberry PI configured as multi-protocol access point, (see document [Setting up the Raspberry PI to be a network router](RPI_Router.md)), you can start using network over bluetooth.



### Using Network Manager

Network Manager is the service that is responsible for managing all the available network connection in a computer. It is available in all the modern Linux distributions.

With Network Manager, as soon as you connect to the RaspberryPI that is listing the NAP service, you should be able to see the "Raspberry PI Network" automatically in the list of interfaces, under the "Mobile Broadband":

<img src="/home/fabrizio/working/bluetooth-howto/RPI-MultiAP/networkmanager.png" alt="networkmanager" style="zoom:50%;" />



### Manually connect

To manually connect to the network, use the `bt-network` command line tool:

```
# bt-network -c 00:1A:7D:DA:71:10 nap
```

After this you should see a new interface:

```
bnep0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::9eb6:d0ff:fee8:1e8c  prefixlen 64  scopeid 0x20<link>
        ether 9c:b6:d0:e8:1e:8c  txqueuelen 1000  (Ethernet)
        RX packets 16  bytes 3009 (3.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25  bytes 2706 (2.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

with no IP address yet.

To get an IP using DHCP from the RPI, use the dhclient:

```
root@arezzo:~# dhclient bnep0
root@arezzo:~# ifconfig
bnep0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.4.14  netmask 255.255.255.0  broadcast 192.168.4.255
        inet6 fe80::9eb6:d0ff:fee8:1e8c  prefixlen 64  scopeid 0x20<link>
        ether 9c:b6:d0:e8:1e:8c  txqueuelen 1000  (Ethernet)
        RX packets 89  bytes 35914 (35.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 135  bytes 24037 (24.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Yay! Now ping the RPI:

```
root@arezzo:~# ping 192.168.4.1
PING 192.168.4.1 (192.168.4.1) 56(84) bytes of data.
64 bytes from 192.168.4.1: icmp_seq=1 ttl=64 time=6.04 ms
64 bytes from 192.168.4.1: icmp_seq=2 ttl=64 time=103 ms
64 bytes from 192.168.4.1: icmp_seq=3 ttl=64 time=25.2 ms
64 bytes from 192.168.4.1: icmp_seq=4 ttl=64 time=23.8 m
```













