# Z3sec

Penetration testing framework to test the touchlink commissioning features of ZigBee-certified products that support touchlink commissioning.

## Introduction

ZigBee Light Link (ZLL) is an application profile of the ZigBee standard, primarily designed for home consumer connected lighting systems. In order to simplify the process of setting up and configuring a ZLL network, ZLL introduced an commissioning mechanism called touchlink commissioning. The tool of this framework only use legitmate features in the protocol that enables a local attacker to perform denial-of-service attacks against touchlink-enabled devices, as well as to take over and control devices.

Touchlink commissioning is supported by all devices complaint to the ZLL specifications, and an optional feature in ZigBee 3.0-complaint products.

## Installation

### Dependencies

+ gnuradio
+ gnuradio-dev
+ gr-foo
+ gr-ieee802.15.4
+ KillerBee
+ scapy-radio (https://bitbucket.org/cybertools/scapy-radio)
+ Ipyton
+ Wireshark (optional)
+ python-crypto
+ python-sphinx (docs)
+ python-numpydoc (docs)
+ setuptools
+ (see `install_dependencies.sh` for a complete list)

#### Install steps (Ubuntu 16.10 x64)

1. Install all needed dependencies with:
  - `bash ./install_dependencies.sh`
   (Password for sudo is requested when needed.)
2. Install Z3sec with:
  - `sudo python setup.py install`

### Hardware

Z3sec supports two different radio platforms: USRP and radios that are
supported by the KillerBee framework. In order to use USRPs it is necessary to
install GnuRadio. USRPs have the advantage that they have an programmable
output gain, so they achieve a higher attack range.

Tested with both the Ettus B200 USRP (GnuRadio) and the MoteIV Tmote Sky
(KillerBee).

#### USRP

1. Connect the USRP to the computer.
3. Make sure the radio is found via `sudo uhd_find_devices`.
3. Use the `--sdr` parameter in order to use it in the tools.

#### Killerbee Radio

1. Connect the device to the computer.
2. Execute `sudo zbid` to find out it's device string.
3. Use `--kb` together with the device string (e.g. `--kb /dev/ttyUSB0`) in
 order to use it in the tools.

### Tools

For a more detailed command line argument descriptions please refer to the
output generated by the `--help` argument.

#### z3sec_touchlink

This tool partially reimplements the touchlink commissioning protocol as an
initiator. It can send touchlink commands to touchlink enabled devices. The
tool comprises of several sub-tools. For further parameters for each sub-tool
please refer to the help message of each sub-tool, e.g., `z3sec_touchlink scan
--help`.

- `scan`: Actively searches for touchlink enabled devices in range and displays
  an overview of all received scan responses. Here, and for all subsequent
  sub-commands, it is possible to specify the channels on which shall be
  searched for devices. By default, only channel 11 is scanned.

        z3sec_touchlink --sdr --channels primary scan
- `anti-scan`: Suppress scan requests of other devices by impersonating them.
  This tool sniffs on channel 11 until a touchlink scan request of an other
  device is received. This scan request is cloned and transmitted on all other
  channels. Devices on these channels will respond immediately on the spoofed
  scan request, but will no longer respond to the original scan request when
  it is received later.

        z3sec_touchlink --sdr anti-scan
- `reset`: Factory reset devices in range. The target devices leave their
  current network and search for a new network. This can be used for an denial
  of service attack. The target devices have to be recommissioned before they
  can be controlled again.

        z3sec_touchlink --sdr reset
  Some (old) touchlink enabled devices can be reset, due to an
  implementation bug, even when not in touchlink range. This bug can be
  exploited in the following way:

        z3sec_touchlink --sdr reset --no_scan
- `identify`: Let a device in range identify itself (e.g. by blinking).
  The blinking duration can be extended up to a theoretical duration of
  approximately 18 hours in order to achieve a denial of service attack:

        z3sec_touchlink --sdr identify --duration 65535
  This command also supports the proximity check circumvention method with
  `--no_scan`.
- `update`: Update the channel of devices in range.  If the channel is changed,
  a device is no longer able to communicate with its current network. This can
  be used for a denial of service attack.

        z3sec_touchlink --sdr update --channel_new 26
- `join`: Request devices to join a new network. The PAN IDs are deduced from
  the own device. If not set, these will be random. All properties can be
  explicitly set:

        z3sec_touchlink --sdr --src_pan_id 0xF001 --src_pan_id_ext DE:FE:C8:ED:DE:FE:C8:ED join --channel_new 26 --network_key BAD1CECAFEFEED8BEEFD00DEB00BFA11

##### Note

The `z3sec_touchlink` tool is not able to send acknowledgements within the
required time frame. However, some devices expect an acknowledgment for their
scan response, otherwise they abort the touchlink transaction. In order to
still send touchlink commands to those devices, it is needed to impersonate
other Zigbee devices in range by setting `--src_addr`, `--src_addr_ext`,
`--src_pan_id`, and `--src_pan_id_ext` according to the values obtained by
sniffing on the same channel. The impersonated device will then send the
required acknowledgments for the scan responses.

#### z3sec_key_extract

With this tool, the network key transmitted during a touchlink key transport
can be extracted from traffic and decrypted.

As traffic source, a radio device can be used to sniff online for
network keys:

    z3sec_key_extract --sdr --channel 11
or the traffic can be read from a pre-captured pcap-file:

    z3sec_key_extract --pcap /path/to/file.pcap

#### z3sec_control
This tool keeps track of all ZigBee devices and networks in range on a specific
channel.

If the network key of one network is known, this tool can be used to
interactively send command frames to devices inside that network. The
network key for a network can be passed on startup:

    z3sec_control --sdr --channel 15 --pan_id 0xF001 --nwk_key BAD1CECAFEFEED8BEEFD00DEB00BFA11
A interactive shell is opened in which python/scapy commands can be entered in
oder to inject command packets into the specified network.
By entering

    show()
an overview of all networks is printed:

    Network: 0
    pan_id:      0xF001
    ext_pan_id:  DE:FE:C8:ED:DE:FE:C8:ED
    network_key: bad1cecafefeed8beefd00deb00bfa11
    # Devices:   2
    # |addr   |ext_addr                |mac_sqn |nwk_sqn |sec_fc  |aps_fc |zcl_sqn |zdp_sqn |
    =========================================================================================
    0 |0xBAAD |DD:FF:88:0D:EE:FF:CC:66 |148     |229     |8323398 |3298   |23      |None    |
    1 |0xFA11 |EE:EE:28:0D:55:F1:98:1E |38      |112     |7249729 |2384   |6       |None    |
Now a command packet can be send into the network. Suppose the second device in
the network is a light bulb, we can impersonate the other device in the network
and send a `light off` command to it

    send(0, 0, 1, pkt_off)
where the first argument specifies the number of the network, the second
argument is the number of the device in the network which we want impersonate,
the third argument is the number of the device in the network to which the
packet is addressed, and the last argument is the command packet.

##### Note
The command packet is automatically encrypted with the network key and all the
address fields and sequence numbers are overwritten with those of the
impersonated and spoofed device.
