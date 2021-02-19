# TESmart 8 and 16 port IP-KVM switches with RS232 Serial

The Tesmart HDMI KVM (8 and 16 port) offers serial and basic LAN control using an Ethernet to UART chipset similar or identical to the CH9120 or the CH9121 (http://www.chinalctech.com/m/view.php?aid=468). This device may be configurable with direct access to the UART using the supplier's configuration utility.

# Controlling the KVM
The KVM can be controlled using the Windows utility available from the [TESmart downloads page](https://buytesmart.com/pages/downloads) using either the RS232 connection or by IP remote connection. There is a script in this directory as well, `tesmart.sh`, that can read the current port, set a new port, and toggle documented features on/off. The API documentation from TESmart does not include information on updating the network settings using the serial or network protocol.

<img src="/tesmart/images/tesmart_controller_1.png" alt="TESmart 8-Port Controller" width=400>

# IP Address and Interface Configuration

The factory default `192.168.1.10/24` can be changed using the Windows utility available from the [TESmart downloads page](https://buytesmart.com/pages/downloads) using either the RS232 connection or remotely. Changes are persistent when applied but do not take effect until the KVM switch is rebooted.

<img src="/tesmart/images/tesmart_controller_2.png" alt="TESmart 8-Port Controller" width=400>

Once the changes are made, the IP settings will be queried automatically. If corrections are not made prior to rebooting the switch, the only known way to reconfigure the IP address would be to use the RS232 3-pin serial connection and appropriate cable or converter, either RS232 to USB or RS232 to TTL.

<img src="/tesmart/images/tesmart_controller_3.png" alt="TESmart 8-Port Controller" width=400>

For systems connected on the same physical or layer 2 network, there are alternatives to reconfiguring the address:

### Alternative 1: Adding a secondary address to an interface
This method only works when the KVM shares the same physical or layer 2 broadcast domain.

Linux: (non-persistent secondary)
```bash
sudo ip addr add 192.168.1.11/31 dev enp1s0
```

Linux: (persistent across reboots)
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
