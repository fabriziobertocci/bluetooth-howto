# Raspberry PI as Router

This document shows all the steps required to turn your Raspberry PI into a network router. 

Note: those instructions don't use anything specific to Raspbian, so they should work also for other Linux versions.

### Summary

You need to configure the following services on the Raspberry PI:

- Set up Wi-Fi as AP: you need to change the WiFi to work as Access Point instead of network client. This allow other computers to connect to the RPI through WiFi. 
- Set up dnsmasq to provide DNS caching and DHCP Server for the clients in the Private Area Network
- Set up a bridge and add the wlan to the bridge
- Enable IP Forward in kernel (NAT)
- Configure `iptables` forwarding rules

If you start from a clean Raspbian installation, you need to install the following packages:

```
# apt install dnsmasq hostapd bridge-utils
```

After installing the packages, make sure the services do not run (yet) until configured:

```
# systemctl stop hostapd
# systemctl stop dnsmasq
```

---

### Configure WiFi to work in AP mode

Edit the file: `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
driver=nl80211
ssid=RPINetwork    # SSID NAME
hw_mode=g          # To use 5GHz set it to 'a'
channel=5          # Check for interference in your WiFi networks
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=xxxxxxxxxxxx    # SET WIFI PASSWORD
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Then tell the hostapd daemon to use this file by editing `/etc/default/hostapd`:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

If this file does not exist, you need to create it.

---

### Configure bridge and PAN Network

Create the bridge `br0` interface with the following commands:

```
# brctl addbr br0
# brctl addif br0 eth0
```

Now add the required files so systemd will bring up the bridge automatically at the next reboot:

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

---

### Start the engine

At this point you should have all the pieces configured, start the services:

```
# systemctl unmask hostapd
# systemctl enable hostapd
# systemctl start hostapd
# systemctl restart dhcpcd
# systemctl start dnsmasq
```

After this you should be able to join the private network using WiFi. 

Also make sure everything works again after a reboot! Reboot the device and make sure the system still work.
