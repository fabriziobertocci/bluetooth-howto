# RPI-MultiAP

This document shows how to set up a Raspberry PI as a multi-protocol access point as shown in the following picture:

```
                                                                                             
           +------------+      WiFi      +---------------------+                             
           |  Client 1  |<-------------->|                     |                             
           +------------+                |   Raspberry PI 3    | Ethernet                    
                                         |                     +----------->                 
           +------------+    Bluetooth   |         NAT         |                             
           |  Client 2  |<-------------->|                     |                             
           +------------+                +---------------------+                             
                                                                                             
                       Private Area Network                   Public Network                 
                         192.168.4.0/24                           (DHCP)                     
                                                                                             
```



**Reference**:

* [Setting up a Raspberry PI as a Wireless Access Point](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md)
* [Pairing a device using command line](https://docs.ubuntu.com/core/en/stacks/bluetooth/bluez/docs/reference/pairing/outbound)
* To test: https://github.com/WayneKeenan/RaspberryPi_BTPAN_AutoConnect
  * and its original: https://github.com/mk-fg/fgtk/blob/master/bt-pan





## Rasperry PI Setup

You need to configure the following services on the Raspberry PI:

* Configure the WiFi to operate in access point mode
* Set up dnsmasq to provide DNS caching and DHCP Server for the clients in the Private Area Network
* Set up a bridge and add the wlan to the bridge
* Enable IP Forward in kernel (NAT)
* Configure ```iptables``` forwarding rules
* Set up the bluetooth network and add it to the bridge



If you start from a clean Raspbian installation, you need to install the following packages:

```
# apt install dnsmasq hostapd bluez bluez-tools bridge-utils
```

After installing the packages, make sure the services do not run (yet) until configured:

```
# systemctl stop hostapd
# systemctl stop dnsmasq
# systemctl stop bluetooth
```





--------------------

### Configure WiFi to work in AP mode

Edit the file: `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
driver=nl80211
ssid=RPINetwork                ### SET THE WI-FI SSID NAME
hw_mode=g                      ### TO USE 5GHz change it to 'a'
channel=5                      ### CHECK FOR INTERFERENCES
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=xxxxxxxxxxxx    ### SET WIFI PASSWORD
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Then tell the hostapd daemon to use this file by editing `/etc/default/hostapd`:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```





-----------

### Configure bridge and PAN Network

Create the bridge `br0` interface with the following commands:

```
# brctl addbr br0
# brctl addif br0 eth0
```

Add the required files so systemd will bring up the bridge automatically at the next reboot:



Create file: `/etc/systemd/network/bridge-br0.netdev`:

```
[NetDev]
Name=br0
Kind=bridge
```

Create file: `/etc/systemd/network/bridge-br0-slave.network`:

```
[Match]
Name=eth0

[Network]
Bridge=br0
```

Set up the bridge static IP in the `/etc/dhcpcd.conf` file:

```
...
denyinterfaces wlan0
#denyinterfaces eth0

# Static IP configuration:
interface br0
static ip_address=192.168.4.1/24
nohook wpa_supplicant
...
```



Now set up `dnsmasq` to implement DNS caching & DHCP to the connected clients:

Edit a clean copy of the dnsmasq config file:

```
# mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
# vim /etc/dnsmasq.conf
```

This is the content:

```
interface=br0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```



Enable IP forwarding and set up iptables forwarding table:

Edit the kernel system config file: `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
```

Create and save an iptables rule:

```
iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/iptables.ipv4.nat
```

Then add the command to reload the table after a reboot in: `/etc/rc.local` 

```
...
iptables-restore < /etc/iptables.ipv4.nat
...
```



----------------------

### Start the engine

At this point you should have all the pieces configured, start the services:

```
# systemctl unmask hostapd
# systemctl enable hostapd
# systemctl start hostapd
# systemctl restart dhcpcd
# systemctl start dnsmasq
```



After this you should be able to join the private network using WiFi. Make sure everything works with wifi before adding bluetooth.

Also make sure everything works again after a reboot! Reboot the device and make sure the system still work.



---------------------------

### Add Bluetooth

Stop the bluetooth service first:

```
# systemctl stop bluetooth
```

... then change the configuration file: `/etc/bluetooth/main.conf`:

```
# Make it discoverable and pairable forever
DiscoverableTimeout = 0
PairableTimeout = 0

```

Restart the service:

```
# systemctl start bluetooth
```

The bluetoothd daemon implements a D-bus API to allow application to interact with the Bluetooth interface.

Now you need to start the network service for the bluetooth stack. This process essentially adds a capability to the given bluetooth interface to support NAP.

```
# bt-network -s nap br0 -d
```



#### Using interactive command-line tools

This section shows how to connect an external machine to the raspberry PI using bluetooth network.

**ALL THE COMMANDS MUST BE RUN AS ROOT**

First of all write down the MAC address of all the devices. Use the command `hciconfig` to list the adapters. In my case I have:

* RPI Internal bluetooth adapter: `B8:27:EB:44:CD:F2` - can be identified by the `Bus: UART` text
* RPI USB adapter: `00:1A:7D:DA:71:10` ("RaspberryFAB") - can be identified by the `Bus: USB` text
* NUC adapter: `9C:B6:D0:E8:1E:8C` ("arezzo")



**RPI:** Start the bluetooth network daemon and associate it to the USB bluetooth adapter:

```
# bt-network -a 00:1A:7D:DA:71:10 -s nap br0 -d
```



RPI & NUC**: Enter the interactive bluetoothctl session:

```
# bluetoothctl
Agent registered
[bluetooth]#
```



**RPI**: By default the internal bluetooth adapter is selected. Turn it off and set the external USB adapter as the default one:

```
[bluetooth]# list
Controller B8:27:EB:44:CD:F2 RaspberryFAB #2 [default]
Controller 00:1A:7D:DA:71:10 RaspberryFAB

[bluetooth]# power off
Changing power off succeeded
[CHG] Controller B8:27:EB:44:CD:F2 Powered: no
[CHG] Controller B8:27:EB:44:CD:F2 Discovering: no
[CHG] Controller B8:27:EB:44:CD:F2 Class: 0x00000000

[bluetooth]# select 00:1A:7D:DA:71:10
Controller 00:1A:7D:DA:71:10 RaspberryFAB [default]
```



RPI**: Make sure the NAP service is implemented and advertised by the bluetooth controller:

```
[bluetooth]# show
Controller 00:1A:7D:DA:71:10 (public)
	Name: RaspberryFAB
	Alias: RaspberryFAB
	Class: 0x004a0000
	Powered: yes
	Discoverable: yes
	Pairable: yes
	UUID: Headset AG                (00001112-0000-1000-8000-00805f9b34fb)
	UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
	UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
	UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
	UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
	UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
	UUID: NAP                       (00001116-0000-1000-8000-00805f9b34fb)
	Modalias: usb:v1D6Bp0246d0532
	Discovering: no
```

(note the UUID: NAP at the end. If that line is not present, the bt-network daemon is not working)



**NUC & RPI**: Ensure there are no devices discovered and paired. If each other device is configured (from a previous run), remove it:

```
[bluetooth]# devices
Device 9C:B6:D0:E8:1E:8C arezzo

[bluetooth]# remove 9C:B6:D0:E8:1E:8C
[DEL] Device 9C:B6:D0:E8:1E:8C arezzo
Device has been removed
```



**NUC & RPI**: Enable scanning, so the two computer will discover each other:

```
[bluetooth]# scan on
Discovery started
[CHG] Controller 00:1A:7D:DA:71:10 Discovering: yes
[NEW] Device 9C:B6:D0:E8:1E:8C arezzo
...
```



**NUC & RPI**: Make sure both computers have discovered each other

**RPI**:

```
[bluetooth]# devices
Device 9C:B6:D0:E8:1E:8C arezzo
...
```

**NUC**: list the devices:

```
[bluetooth]# devices
Device 00:1A:7D:DA:71:10 RaspberryFAB
...
```



**NUC & RPI**: Enable the agent responsible for handling pairing validation requests:

```
[bluetooth]# agent on
Agent is already registered
[bluetooth]# default-agent
Default agent request successful
```



**NUC & RPI**: Trust each other device

**RPI**:

```
[bluetooth]# trust 9C:B6:D0:E8:1E:8C
[CHG] Device 9C:B6:D0:E8:1E:8C Trusted: yes
Changing 9C:B6:D0:E8:1E:8C trust succeeded
```

**NUC**:

```
[bluetooth]# trust 00:1A:7D:DA:71:10
[CHG] Device 00:1A:7D:DA:71:10 Trusted: yes
Changing 00:1A:7D:DA:71:10 trust succeeded
```



**NUC**: Enable pairing. When you start the pair command the agent on both devices will ask you to confirm the passkey. Just enter "yes" on both:

```
[bluetooth]# pair  00:1A:7D:DA:71:10
Attempting to pair with 00:1A:7D:DA:71:10
[CHG] Device 00:1A:7D:DA:71:10 Connected: yes
Request confirmation
[Rasp1m[agent] Confirm passkey 607573 (yes/no): yes
[CHG] Device 00:1A:7D:DA:71:10 Modalias: usb:v1D6Bp0246d0532
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 0000110a-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 00001112-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 00001116-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 0000111f-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 UUIDs: 00001200-0000-1000-8000-00805f9b34fb
[CHG] Device 00:1A:7D:DA:71:10 ServicesResolved: yes
[CHG] Device 00:1A:7D:DA:71:10 Paired: yes
Pairing successful
[CHG] Device 00:1A:7D:DA:71:10 ServicesResolved: no
[CHG] Device 00:1A:7D:DA:71:10 Connected: no
```



**RPI**: Repeat the pairing process:

```
pair 9C:B6:D0:E8:1E:8C
Attempting to pair with 9C:B6:D0:E8:1E:8C
[CHG] Device 9C:B6:D0:E8:1E:8C Connected: yes
Request confirmation
[agent] Confirm passkey 903296 (yes/no): yes
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001104-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001105-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001106-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001108-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 0000110a-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 0000110b-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001112-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 0000112f-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001132-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001133-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00001200-0000-1000-8000-00805f9b34fb
[CHG] Device 9C:B6:D0:E8:1E:8C UUIDs: 00005005-0000-1000-8000-0002ee000001
[CHG] Device 9C:B6:D0:E8:1E:8C ServicesResolved: yes
[CHG] Device 9C:B6:D0:E8:1E:8C Paired: yes
Pairing successful
[CHG] Device 9C:B6:D0:E8:1E:8C ServicesResolved: no
[CHG] Device 9C:B6:D0:E8:1E:8C Connected: no
```



**RPI | NUC**: Connect from one of them. In this case I issued the command on the RPI:

```
[bluetooth]# connect 9C:B6:D0:E8:1E:8C
Attempting to connect to 9C:B6:D0:E8:1E:8C
[CHG] Device 9C:B6:D0:E8:1E:8C Connected: yes
Connection successful
[CHG] Device 9C:B6:D0:E8:1E:8C ServicesResolved: yes
```

After connection completes, even the NUC sees the device as connected:

```
info 00:1A:7D:DA:71:10
Device 00:1A:7D:DA:71:10 (public)
	Name: RaspberryFAB
	Alias: RaspberryFAB
	Class: 0x004a0000
	Paired: yes
	Trusted: yes
	Blocked: no
	Connected: yes
	LegacyPairing: no
	UUID: Headset                   (00001108-0000-1000-8000-00805f9b34fb)
	UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
	UUID: Advanced Audio Distribu.. (0000110d-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
	UUID: Headset AG                (00001112-0000-1000-8000-00805f9b34fb)
	UUID: NAP                       (00001116-0000-1000-8000-00805f9b34fb)
	UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
	UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
	Modalias: usb:v1D6Bp0246d0532
```

Note the "Connected: yes" and "UUID: NAP".

At this point if you use Network manager, you should be able to see a new device in the list of network connections listed as a "Mobile Device".  You can use the Network manager to do everything automatically (start the network, request DHCP), or one service at the time:

**NUC**: Start the network service (from a shell, not from `bluetoothctl`):

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

To get an IP using DHCP from the RPI:

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



Yay! Now ping the RPI from NUC:

```
root@arezzo:~# ping 192.168.4.1
PING 192.168.4.1 (192.168.4.1) 56(84) bytes of data.
64 bytes from 192.168.4.1: icmp_seq=1 ttl=64 time=6.04 ms
64 bytes from 192.168.4.1: icmp_seq=2 ttl=64 time=103 ms
64 bytes from 192.168.4.1: icmp_seq=3 ttl=64 time=25.2 ms
64 bytes from 192.168.4.1: icmp_seq=4 ttl=64 time=23.8 ms
```





## Notes:##

To change the name of the device (for example to *RaspberryFAB*), use the following command:

```
# hciconfig hci0 name 'RaspberryFAB'
```

Or alternatively you can change it using the `bluetoothctl` command `set-alias`.

The name of the device can also be set in the `/etc/bluetooth/main.conf` file, with the `Name` property.



Issues? 

See: https://github.com/raspberrypi/firmware/issues/1150



