# Pairing Bluetooth Devices



This document shows how to connect two devices using Bluetooth. The purpose of it is to show how to use TCP/IP over Bluetooth, but it applies to pairing any devices.

On a normal Linux system you have (at least) 4 ways to operate with the bluetooth devices:

* through the GUI tools (i.e. bluetooth device manager)
* using the interactive tool `bluetoothctl`
* using the command-line tools `bt-device`, `bt-adapter`, `bt-agent`
* through the HCI low-level interface using `hcitool` and `hciconfig`

This document is going to show how to use the interactive tools `bluetoothctl`. The command-line tools achieve the same purpose without an interactive session (using only command-line), making them suitable for scripting.



First let's make sure you have the required packages:

```
# apt install bluez bluez-tools
```

Then stop the bluetooth service. 

```
# systemctl stop bluetooth
```

The bluetoothd daemon implements a dbus API to allow application to interact with the Bluetooth interface. So all the command line (and GUI) tools instead of operating directly with the low-level device they perform RPC to the bluetooth daemon using dbus.

Now change the main configuration file `/etc/bluetooth/main.conf` to make the device discoverable forever:

```
# Make it discoverable and pairable forever
DiscoverableTimeout = 0
PairableTimeout = 0
```

Restart the service now:

```
# systemctl start bluetooth 
```

If you are planning to use the Bluetooth device as a network SERVER, start the bluetooth network service:

```
# bt-network -s nap br0 -d
```

In general you can start this service at any time **before you connect to a remote device**.

If your Rasperry PI uses an external USB Bluetooth dongle, you must specify the interface you are going to use with the `-a` parameter. I.e.:

```
# bt-network -a 00:0C:41:E1:E2:86 -s nap br0 -d
NAP server registered
```

To list the adapters available, use the `hciconfig` or the interactive tool `bluetoothctl` (see below) to see them all:

```
$ hciconfig 
hci0:	Type: Primary  Bus: USB
	BD Address: 00:0C:41:E1:E2:86  ACL MTU: 192:8  SCO MTU: 64:8
	UP RUNNING PSCAN ISCAN 
	RX bytes:401 acl:0 sco:0 events:19 errors:0
	TX bytes:317 acl:0 sco:0 commands:18 errors:0

hci1:	Type: Primary  Bus: UART
	BD Address: B8:27:EB:44:CD:F2  ACL MTU: 1021:8  SCO MTU: 64:1
	DOWN 
	RX bytes:815 acl:0 sco:0 events:56 errors:0
	TX bytes:3021 acl:0 sco:0 commands:56 errors:0

```

To identify which one is the internal or the external, look at the **Bus** parameter. For the external dongle, the value is "USB", for the internal adapter the value is "UART".







## Pairing devices with interactive tool

This section shows how to discover, pair and connect devices using the interactive text-only tool `bluetoothctl`

**IMPORTANT**: On the Raspberry PI, since the internal Bluetooth interface has some [serious issues]( https://github.com/raspberrypi/firmware/issues/1150), you should use an external Bluetooth USB adapter. 

In the following examples the internal adapter have MAC=`B8:27:EB:44:CD:F2` and the external one has MAC=`00:0C:41:E1:E2:86`.

Start the interactive session, turn OFF the internal Bluetooth device, and select the external USB:

```
$ sudo bluetoothctl
Agent registered

[bluetooth]# list
Controller B8:27:EB:44:CD:F2 RaspberryFAB #2 [default]
Controller 00:0C:41:E1:E2:86 RaspberryFAB

[bluetooth]# power off
Changing power off succeeded

[bluetooth]# select 00:0C:41:E1:E2:86
```

Now, start and enable the agent required for handling user interaction during pairing:

```
[bluetooth]# agent on
[bluetooth]# default-agent
```



Then make the device discoverable and pairable:

```
[bluetooth]# discoverable on
[bluetooth]# pairable on
```



**NOTE**: before pairing any device, check the services that are offered by the adapter using the `show` command. In particular, make sure the NAP service is implemented and advertised by the bluetooth controller to implement network over bluetooth:

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



### Pairing from an external computer

If you start the pairing from an external computer, follow these steps:

**COMPUTER**: Enable scanning:

```
[bluetooth]# scan on
Discovery started
[CHG] Controller 9C:B6:D0:E8:1E:8C Discovering: yes
[NEW] Device D8:0F:99:62:21:34 XBR-55X700D
[NEW] Device 00:0C:41:E1:E2:86 RaspberryFAB
...
```



At any time you can get the list of discovered devices with:

```
[bluetooth]# devices
Device E3:28:E9:23:22:5F B06+
Device D8:0F:99:62:21:34 XBR-55X700D
Device 00:0C:41:E1:E2:86 RaspberryFAB
Device 00:23:12:59:0A:73 portobello
Device 6C:6D:57:61:C9:57 N00C8
```



At any time after you discover all the targets, you can stop the scan:

```
[bluetooth]# scan off
[CHG] Device 6C:6D:57:61:C9:57 RSSI is nil
[CHG] Device 00:23:12:59:0A:73 RSSI is nil
[CHG] Device 00:0C:41:E1:E2:86 RSSI is nil
[CHG] Device D8:0F:99:62:21:34 RSSI is nil
[CHG] Controller 9C:B6:D0:E8:1E:8C Discovering: no
Discovery stopped
```



Note that the discovery process allow the computer to discover the RPI, but not the vice-versa. If you run the `devices` command on the RPI you will get an empty list



Now start the pairing from the computer using the `pair` command. Depending on the adapter, you might be prompted to ask or confirm a Pairing Pin code:

```
[bluetooth]# pair 00:0C:41:E1:E2:86
Attempting to pair with 00:0C:41:E1:E2:86
[CHG] Device 00:0C:41:E1:E2:86 Connected: yes
Request PIN code
[agent] Enter PIN code: 123456
```

After entering the PIN on the computer, the agent on the RaspberryPI will ask you to confirm or re-enter the PIN:

```
Request PIN code
[agent] Enter PIN code: 123456
```



From now on the two devices are paired. When the devices are paired for the first time, they will attempt to automatically connect. 



On the RaspberryPI tell the system to trust the computer using the `trust` command. This will avoid the agent to ask for permissions when using any service.

If you don't have the computer in the trusted list:

```
Authorize service
[agent] Authorize service 0000110d-0000-1000-8000-00805f9b34fb (yes/no): yes
```

(you have to type 'yes' every time you connect to the device)



If you add the device to the trust list, you won't be prompted anymore:

```
[bluetooth]# trust 9C:B6:D0:E8:1E:8C
[CHG] Device 9C:B6:D0:E8:1E:8C Trusted: yes
Changing 9C:B6:D0:E8:1E:8C trust succeeded
```



From the computer check the capabilities of the RPI:

```
[RaspberryFAB]# info 00:0C:41:E1:E2:86
Device 00:0C:41:E1:E2:86 (public)
	Name: RaspberryFAB
	Alias: RaspberryFAB
	Class: 0x004a0000
	Paired: yes
	Trusted: no
	Blocked: no
	Connected: yes
	LegacyPairing: yes
	UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
	UUID: Headset AG                (00001112-0000-1000-8000-00805f9b34fb)
	UUID: NAP                       (00001116-0000-1000-8000-00805f9b34fb)
	UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
	UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
	Modalias: usb:v1D6Bp0246d0532

```

 









-------------------------------

### Some useful commands:

To see the existing paired devices, use the command `paired-devices`:

```
[bluetooth]# paired-devices
Device 9C:B6:D0:E8:1E:8C arezzo
```



To see the known devices, use the command `devices`:

```
[bluetooth]# devices
Device E3:28:E9:23:22:5F B06+
Device 00:0C:41:E1:E2:86 RaspberryFAB
```



Remove a device (whether is paired or just discovered), use `remove <Device>`:

```
[bluetooth]# remove 00:0C:41:E1:E2:86
[DEL] Device 00:0C:41:E1:E2:86 RaspberryFAB
Device has been removed
```



To print the status and the capabilities of a device, use `info` command:

```
[bluetooth]# info 00:0C:41:E1:E2:86
Device 00:0C:41:E1:E2:86 (public)
	Name: RaspberryFAB
	Alias: RaspberryFAB
	Class: 0x004a0000
	Paired: yes
	Trusted: yes
	Blocked: no
	Connected: no
	LegacyPairing: no
	UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
	UUID: Headset AG                (00001112-0000-1000-8000-00805f9b34fb)
	UUID: NAP                       (00001116-0000-1000-8000-00805f9b34fb)
	UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
	UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
	Modalias: usb:v1D6Bp0246d0532

```



To change the name of the device (for example to *RaspberryFAB*), use the following command:

```
# hciconfig hci0 name 'RaspberryFAB'
```

Or alternatively you can change it using the `bluetoothctl` command `set-alias`.

The name of the device can also be set in the `/etc/bluetooth/main.conf` file, with the `Name` property.