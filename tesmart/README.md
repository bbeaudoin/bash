# TESmart 8- and 16-port IP-KVM switches

The Tesmart HDMI KVM (8 and 16 port) offers serial and basic LAN control using an Ethernet to UART chipset similar or identical to the CH9120 or the CH9121 (http://www.chinalctech.com/m/view.php?aid=468). This device may be configurable with direct access to the UART using the supplier's configuration utility.

The Tesmart KVM uses the chipset to allow communication between the LAN and the UART without direct serial connection. A serial or serial to USB cable may be preferred when the switch is physically colocated with the control equipment but does not enable additional features.

# IP Address and Interface Configuration

The factory default `192.168.1.10/24` can be changed using the Windows utility available from the [TESmart downloads page](https://buytesmart.com/pages/downloads) using either the RS232 connection or remotely (with risk of connection lost, if misconfigured).

<img src="/tesmart/images/tesmart_controller_2.png" alt="TESmart 8-Port Controller" width=400>

If your subnet is not within this range, it is possible to add a peer-to-peer IP alias _if both systems are on the same physical network_.

## Example 1: Adding a secondary address to an interface
```bash
sudo ip addr add 192.168.1.11/31 dev enp1s0
```

If it is desired to use this method permanently to communicate with the TESmart KVM, the IP alias can be added to the interface boot configuration.

## Example 2: Persistent secondary address via nmcli
```bash
sudo nmcli con mod enp1s0 +ipv4.addresses "192.168.1.11/31"
```

Should a host be serving as a gateway but the IP needs to be accessed using another host, a persistent or non-persistent route can be added via the gateway host, provided IP routing/forwarding is enabled.

## Example 3: Adding a non-persistent Linux route
```bash
sudo ip route add 192.168.1.10 via \<host_IP\>
```

When run as administrator, a route can be added to the gateway host using the Windows `cmd.exe` command line window.

## Example 4: Using the Windows cmd.exe as Administrator
```cmd
Microsoft Windows [Version 10.0.19042.804]
(c) 2020 Microsoft Corporation. All rights reserved.

C:\Windows\system32>route add 192.168.1.10 mask 255.255.255.254 10.0.2.20
 OK!

C:\Windows\system32>
```

# Controlling the KVM
The KVM can be controlled using the Windows utility available from the [TESmart downloads page](https://buytesmart.com/pages/downloads) using either the RS232 connection or by IP remote connection. There is a script in this directory as well, `tesmart.sh`, that can read the current port, set a new port, and toggle documented features on/off. The API documentation from TESmart does not include information on updating the network settings using the serial or network protocol.

<img src="/tesmart/images/tesmart_controller_1.png" alt="TESmart 8-Port Controller" width=400>

While the API documentation goes into detail on the protocol, it can be written more succinctly. This is an attempt at providing a clearer explanation of the command sequences for controlling the KVM using the protocols.

## Protocol Information
The protocol described in the documentation is not a REST API, rather bytes of data are transmitted, either via RS232 or sending bytes to the IP socket. Whether using serial or ethernet communication, hexadecimal bytes are written to the open descriptor in the format

```h
0xAA 0xBB 0x03 <0x..> <0x..> 0xEE
```

The preamble is always `0xAA 0xBB 0x03`. The two following bytes describe the option and value. When reading, the value is ignored. `0xEE` is the expected termination string of the pattern.

```
0x01 - Change the active KVM port (1-based index from 0x01-0x10)
0x02 - Set the buzzer on or off (0x00 for off, 0x01 for on)
0x03 - Set the display timeout (0x00, 0x0A, 0x1E for never, 10s, 30s)
0x10 - Read the active port number (0-based index from 0x00-0x0f)
```

On successful communication, the response is expected to return the active port number before `0xEE`.

```h
0xAA 0xBB 0x03 0x11 <0x..> 0xEE
```

Should a communication error occur, the KVM may not return the expected response. It is recommended to rate limit communication by at least 1 second to avoid errors. The script `kvmctl.sh` here takes the following approach at mitigating errors:

1. Before setting, an attempt to read the active port is retried 3 times with a delay
1. If the active port is already set, the script will not attempt to set the port again
1. Should the active port not be returned after a command, an error is assumed
