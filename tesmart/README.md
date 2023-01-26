# TESmart 8 and 16 port IP-KVM switches with RS232 Serial

The Tesmart HDMI KVM (8 and 16 port) offers serial and basic LAN control using an Ethernet to UART chipset similar or identical to the CH9120 or the CH9121 (http://www.chinalctech.com/m/view.php?aid=468). This device may be configurable with direct access to the UART using the supplier's configuration utility. Waveshare has documentation on a board with that chip at (https://www.waveshare.com/w/upload/e/ef/CH9121_SPCC.pdf).

Note, I have not verified the chip in the TESmart network module so the information may not apply. I ran a packet capture to see how the address information is configured but I caution against using this information to update the network configuration. This information and these scripts are provided as-is with no warranty, using this information and these scripts are at your own risk.

# Controlling the KVM
The KVM can be controlled using the Windows utility available from the [TESmart downloads page](https://buytesmart.com/pages/downloads) using either the RS232 connection or by IP remote connection. There is a script in this directory as well, `tesmart.sh`, that can read the current port, set a new port, and toggle documented features on/off.

While the API documentation from TESmart does not include information on updating the network settings using the serial or network protocol, the controller traffic indicates configuration is performed using ASCII commands. Assuming the CH9121, I checked the Waveshare documentation on their board and how to set the IP address information, it appears this may be handled on the serial side of the controller: [CH9121 SPCC (PDF)](https://www.waveshare.com/w/upload/e/ef/CH9121_SPCC.pdf). TESmart recently switched from a 10/100 Mbps controller to a 10 Mbps one that is incompatible with 2.5/5/10 Gbps Ethernet, I photographed the boards but have not identified the chips.
<img src="/tesmart/images/tesmart_controller_1.png" alt="TESmart 8-Port Controller" width=400>

# IP Address and Interface Configuration

My personal recommendation is to use the controller and documentation from the [TESmart downloads page](https://buytesmart.com/pages/downloads). The factory default `192.168.1.10/24` and may be changed using either the RS232 connection or remotely. Once applied, the changes are persisted but do not take effect until the KVM has been rebooted.

<img src="/tesmart/images/tesmart_controller_2.png" alt="TESmart 8-Port Controller" width=400>

WARNING, confirm the IP settings before rebooting the KVM. The TESmart enterprise switch doesn't have a "factory reset" to my knowledge. If it is not possible to make a network connection to the KVM after a reboot and the settings are unknown (not default or the intended settings), the only known way to reconfigure the IP address would be to use the RS232 3-pin serial connection and appropriate cable or converter, either RS232 to USB or RS232 to TTL, to connect to the switch.

<img src="/tesmart/images/tesmart_controller_3.png" alt="TESmart 8-Port Controller" width=400>

For systems connected on the same physical or layer 2 network, there are alternatives to reconfiguring the address:

### Alternative 1: Adding a secondary address to an interface
This method only works when the KVM shares the same physical or layer 2 broadcast domain. This can be helpful if the network settings are known but are not within the normal network's valid range on an unsegmented layer 2 or shared VLAN if needed to reconfigure the networking on the switch.

Linux: (non-persistent secondary address, change `enp1s0` to your device name)
```bash
sudo ip addr add 192.168.1.11/31 dev enp1s0
```

Linux: (persistent across reboots, change `enp1s0` to your connection name)
```bash
sudo nmcli con mod enp1s0 +ipv4.addresses "192.168.1.11/31"
```

### Alternative 2: Add a route to the host/gateway to the KVM host/network
This method only works if there exists a layer 3 (IP) connection between the client and a server acting as a gateway

Linux:
```bash
sudo ip route add 192.168.1.10 via <host_IP>
```

Windows:
```bat
route add 192.168.1.10 mask 255.255.255.254 10.0.2.20
```

Neither of those commands add persistent routes.

# Protocol Information
While the API documentation goes into detail on the protocol, this is an attempt at a more general explanation. The API is not a REST API, rather it is a set of raw bytes sent via the serial port at 9600 8n1 or via Telnet protocol to the IP address and port configured (192.168.1.10:5000 by default).

```h
0xAA 0xBB 0x03 <0x..> <0x..> 0xEE
```

The preamble is always `0xAA 0xBB 0x03`. The two following bytes describe the option and value. When reading, the value is ignored. `0xEE` is the expected termination string of the pattern.

| Token | Value | Description |
|:--- |:--- |:--- |
| `0x01` | `0x01-0x10` | Change the active KVM port (1-8 or 1-16) |
| `0x02` | `0x00-0x01` | Set the buzzer off or on |
| `0x03` | `0x00, 0x0A, 0x1E` | Set the display timeout (Never, 10s, 30s) |
| `0x81` | `0x00-0x01` | Disable/Enable Input Detection |
| `0x10` | `0x00` | Read the active port number (0-7 or 0-15) |

The documentation states the return will be `0xAA 0xBB 0x03 0x11 <0x..> 0xEE` but may not always return valid data. When invalid data is returned, the script based on the protocol information will return `FF` as the output, unless the port is changed the response is ignored.

```h
0xAA 0xBB 0x03 0x11 <0x..> 0xEE
```

Should a communication error occur, the KVM may not return the expected response. It is recommended to rate limit communication by at least 1 second to avoid errors. The script `kvmctl.sh` here takes the following approach at mitigating errors:

1. Before setting, an attempt to read the active port is retried 3 times with a delay
1. If the active port is already set, the script will not attempt to set the port again
1. Should the active port not be returned after a command, an error is assumed

The sample script takes the following options:

```bash
$ kvmctl
kvmctl -- Controls a Tesmart KVM using TCP/IP
Usage:
  kvmctl get           : Retrieves the active port number.
  kvmctl set <1-8>     : Retrieves the active port number.
  kvmctl buzzer <0|1>  : Turns the buzzer off (0) or on (1).
  kvmctl lcd <0|10|30> : Disable or set the LCD timeout.
  kvmctl auto <0|1>    : Disable or enable auto input detection.
$
```
## Setting and querying the IP address configuration

This information was gathered using Wireshark to capture packets from my client to the TESmart KVM. While it may be possible to use this information as an alternative to the controller, an inproper network configuration may leave the KVM unreachable and if it is not possible to re-establish connection or the connection information was not recorded, it may not be possible to reconfigure the network interface without an RS232 connection.

The protocol seems to be simple ASCII commands from client to server without the TESmart API pre-amble or syntax.

```
0000   72 a7 41 ad 31 58 2c db 07 32 63 ec 08 00 45 00   r.A.1X,..2c...E.
0010   00 35 53 d3 00 00 80 06 00 00 0a 00 00 70 0a 00   .5S..........p..
0020   02 0c de 5b 13 88 7c f3 32 8e 00 00 5e 88 50 18   ...[..|.2...^.P.
0030   ff 46 17 d1 00 00 49 50 3a 31 30 2e 30 2e 32 2e   .F....IP:10.0.2.
0040   31 32 3b                                          12;
```

If the command is accepted as valid, the system responds with "OK". The data payloads of the communication are recorded below with comments between blocks offering some interpretation based on the behavior of the system. The first section is configuring basic networking with a known good established connection.

```
Client: IP:10.0.2.12;
Server: OK
Client: PT:5000;
Server: OK
Client: GW:10.0.2.1;
Server: OK
Client: MA:255.255.255.0;
Server: OK
```

It may be assumed the information is persisted immediately prior to the system returning "OK". If that assumption holds true, more efficient updates may be possible. However, incomplete updates or invalid information can leave the KVM unable to communicate properly on the network.

The controller follows up with a set of queries to refresh the configuration. Hitting the "Query" button is identical to the call made after applying existing or updated settings. The conversation to query the current network configuration looks like this:

```
Client: IP?
Server: IP:010.000.002.012
Server: ;
Client: PT?
Server: PT:05000;
Client: GW?
Server: GW:010.000.002.001
Server: ;
Client: MA?
Server: MA:255.255.255.000
Server: ;
```

The client will display the readout as follows:

```
IP: 010.000.002.012
Port: 05000
Gate way: 010.000.002.001
Mask: 255.255.255.000
```

## Server Limitations

It appears the maximum transmission unit (MTU) is 48 bytes. When the amount of data in the buffer exceeds this size, the payload will be split between two packets explaining why the ";" character is sent as a separate message.

## Errors while reading/writing data

Occasionally or when the port is changed using another interface, the TESmart will send a message indicating the current port number if a client is connected. If the client is in the process of sending or receiving information, it may not handle these updates appropriately. While querying the network information, the official controller will print the update as if it was information received at the cursor's current location, even if it had not yet sent the command to query. This is a bug in the controller code.

These updates do not appear to occur at regular intervals, at least not in the packet capture. It is unknown whether these asynchronous updates are generated during configuration. If so the response after acknowledgement may not be "OK", rather an instruction for the client to refresh the active port. A well-behaved client would discard the message or push it to a stack or queue so it's not missed and look instead for the actual response, be it "OK" or another response indicating an actual error. The asynchronous nature of communication can explain why short-lived read/write connections may not receive expected responses when opened and closed quickly.

## Handling the connection

If writing a client, a normal TCP/IP connection established can send and receive communication as a stream. Although the API states messages will always end in `0xEE`, values of `0x16` and `0x1c` have been observed as terminating characters, the important characters are `0xAA 0xBB 0x03 0x11` followed by one byte of data and a termination byte, regardless of value. If ASCII is received, it is likely to be in response to applying network configuration or querying network configuration. The messages may be handled with or without threading as long as unexpected updates are expected and treated accordingly. Upon proper closing of a connection, the server will stop sending updates via TCP/IP even if serial data is still observed by the serial-to-network interface.
