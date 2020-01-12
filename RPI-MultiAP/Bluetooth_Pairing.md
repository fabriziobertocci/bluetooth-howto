

This document shows how to pair and connect to devices using Bluetooth.  The direct purpose of it is to show how to use TCP/IP over Bluetooth, but it applies to pairing any devices.

On a normal Linux system you have 3 (or more) ways to operate with the bluetooth devices:

* through the GUI tools (i.e. bluetooth device manager)
* using the interactive tool `bluetoothctl`
* using the command-line tools `bt-device`, `bt-adapter`, `bt-agent`

This document is not going to describe how to use the GUI tools (too easy...)



First let's make sure you have the required packages:

```
# apt install bluez bluez-tools
```

Then stop the bluetooth service. The bluetoothd daemon implements a dbus API to allow application to interact with the Bluetooth interface. So all the command line (and GUI) tools instead of operating directly with the low-level device they perform RPC to the bluetooth daemon using dbus.

```
# systemctl stop bluetooth
```

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

## Pairing devices with interactive tool

This section shows how to discover, pair and connect devices using the interactive text-only tool `bluetoothctl`

On the Raspberry PI, since the internal Bluetooth interface has some [serious issues]( https://github.com/raspberrypi/firmware/issues/1150), you should use an external Bluetooth USB adapter. 

In the following examples the internal adapter have MAC=`AA:AA:AA:AA:AA:AA` and the external one has MAC=`BB:BB:BB:BB:BB:BB`.

Start the interactive session, and turn OFF the internal Bluetooth device and select the external USB:

```
$ sudo bluetoothctl
Agent registered

[bluetooth]# list
Controller AA:AA:AA:AA:AA:AA RaspberryFAB #2 [default]
Controller BB:BB:BB:BB:BB:BB RaspberryFAB

[bluetooth]# power off
Changing power off succeeded

[bluetooth]# select BB:BB:BB:BB:BB:BB
```

Now, start and enable the agent required for handling user interaction during pairing:

```
[bluetooth]# agent on
[bluetooth]# default-agent
```



And make the device discoverable and pairable:

```
[bluetooth]# discoverable on
[bluetooth]# pairable on
```



From 