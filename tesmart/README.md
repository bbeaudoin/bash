# PROTOCOL INFORMATION

The Tesmart HDMI KVM (8 and 16 port) offers serial and basic LAN control using an Ethernet to UART chipset similar or identical to the CH9120 or the CH9121 (http://www.chinalctech.com/m/view.php?aid=468). This device may be configurable with direct access to the UART using the supplier's configuration utility.

Factory settings from Tesmart assume no DHCP and a fixed address and subnet (192.168.1.10/24) and is configured for port 5000/tcp. It is possible they considered DHCP would not be available for autoconfiguration and that the chipset does not support a default IP when DHCP is not available on the customer's network.

Updating the firmware is possible with a utility from the part supplier with direct access to the UART. It does not appear to be possible to change the values using `arp` or other utility.

The Tesmart KVM uses the chipset to allow communication between the LAN and the UART without direct serial connection. A serial or serial to USB cable may be preferred when the switch is physically colocated with the control equipment but does not enable additional features.

# Controlling the KVM
Whether using serial or ethernet communication, a simple binary protocol expressed in hexidecimal code is used in the format

```
0xAA 0xBB 0x03 0x.. 0x.. 0xEE
```

The two bytes following the preamble and before the termination sequence are command and value respectively. In short these are:

```
0x01 - Change the active KVM port (1-based index from 0x01-0x10)
0x02 - Set the buzzer on or off (0x00 for off, 0x01 for on)
0x03 - Set the display timeout (0x00, 0x0A, 0x1E for never, 10s, 30s)
0x10 - Read the active port number (0-based index from 0x00-0x0f)
```

On successful communication, the expected response is

```
0xAA 0xBB 0x03 0x11 0x.. 0xEE
```

While the last byte is always expected to be `0xEE`, it appears this may vary based on the active port. It also appears on transmission it may ignore the value of the last byte received.
