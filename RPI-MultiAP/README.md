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

This setup is divided into 3 sections:

* [Setting up the Raspberry PI to be a network router](RPI_Router.md)

* [Pairing Bluetooth devices](Bluetooth_Pairing.md)

* [Setting up computers to use network over bluetooth](NetworkOverBluetooth.md)


