# PROTOCOL INFORMATION

The Tesmart HDMI KVM (8 and 16 port) offers serial and basic LAN control using an Ethernet to UART chipset similar or identical to the CH9120 or the CH9121 (http://www.chinalctech.com/m/view.php?aid=468). This device may be configurable with direct access to the UART using the supplier's configuration utility.

The Tesmart KVM uses the chipset to allow communication between the LAN and the UART without direct serial connection. A serial or serial to USB cable may be preferred when the switch is physically colocated with the control equipment but does not enable additional features.

# IP Address and Interface Configuration

The factory default `192.168.1.10/24` cannot be changed without physical access but seems to be possible using a utility available from the part supplier if it is disconnected from the KVM. If your subnet is not within this range, it is possible to add a peer-to-peer IP alias. The only valid p2p host address available is 192.168.1.11/31 matching the hard-coded address bit boundary.

## Example 1: Adding a secondary address to an interface
```bash
ip addr add 192.168.1.11/31 dev enp1s0
```

This may alternately be added persistently using NetworkManager's `nmcli`, `nmtui`, the GUI, or other method of configuration.

## Example 2: Persistent secondary address via nmcli
```bash
sudo nmcli con mod enp1s0 +ipv4.addresses "192.168.1.11/31"
```

When using a traditional `ifcfg.<dev>:1` approach to secondary addresses, the primary is assumed to be `:0`. When instantiated, the additional configuration will be added to the parent device, it won't appear as a second device.

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
