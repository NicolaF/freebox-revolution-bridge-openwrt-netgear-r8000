# Freebox in bridge mode with a Netgear R8000 running OpenWRT
Notes on Putting a Freebox Revolution in bridge mode with a Netgear R8000 running OpenWrt.

## Goals
* Install OpenWRT on the Netgear R8000
* Put the Freebox in bridge mode
* Full IPv6
* Separate guest network
* VPN server
* Bring back as many Freebox services as possible
    * Freebox Player (TV, DLNA client & co, Chromecast,...)
    * Freebox Server (TV over RTSP, DLNA server, other services announcements)

## Before we start
There will be a fair amount of customization, and due to how OpenWrt's upgrade process works:

* When modifying a configuration file, ensure that it will be kept during sysupgrade. Add it in `/etc/sysupgrade.conf` if needed.
* Keep track of installed packages, eg. this with a file directly on the router that will be retained during sysupgrade.
* Backup your configuration regularly.

## Installing OpenWrt
### Problem
Recent Netgear firmwares do not allow to downgrade below 1.0.4.12_10.1.46 nor reinstalling the same version:

```
$ hexdump -Cn80 R8000-V1.0.4.12_10.1.46.chk
                            ------>    1  0  4 12 10  1 46
00000000  2a 23 24 5e 00 00 00 3a  01 01 00 04 0c 0a 01 2e  |*#$^...:........|
00000010  d4 14 1a 81 00 00 00 00  01 de 60 00 00 00 00 00  |..........`.....|
00000020  d4 14 1a 81 fc 71 0a 4b  55 31 32 48 33 31 35 54  |.....q.KU12H315T|
00000030  30 30 5f 4e 45 54 47 45  41 52 48 44 52 30 00 60  |00_NETGEARHDR0.`|
00000040  de 01 99 45 54 b0 00 00  01 00 1c 00 00 00 64 4a  |...ET.........dJ|
00000050
```

```
$ hexdump -Cn80 openwrt-19.07.2-bcm53xx-netgear-r8000-squashfs.chk
                            ------>    1  1 99  0  0  0  0
00000000  2a 23 24 5e 00 00 00 3a  01 01 01 63 00 00 00 00  |*#$^...:...c....|
00000010  f7 af 87 42 00 00 00 00  00 78 00 00 00 00 00 00  |...B.....x......|
00000020  f7 af 87 42 26 3b 0b 77  55 31 32 48 33 31 35 54  |...B&;.wU12H315T|
00000030  30 30 5f 4e 45 54 47 45  41 52 48 44 52 30 00 00  |00_NETGEARHDR0..|
00000040  78 00 a7 e2 d0 ed 00 00  01 00 1c 00 00 00 00 00  |x...............|
00000050
```

### Solution: build an image with a very high version number
See [this github PR](https://github.com/openwrt/openwrt/pull/1857) (now merged)
```
git clone https://github.com/openwrt/openwrt.git
cd openwrt
git checkout v19.07.2
patch -p1 < ....
./scripts/feeds update -a
./scripts/feeds install -a
wget  https://downloads.openwrt.org/releases/19.07.2/targets/bcm53xx/generic/config.buildinfo -O .config
make menuconfig # Target system: `BCM47xx/53xx (ARM)`, Target profile: `Netgear R8000`
make
```
Then install the built image as usual.

### What doesn't work
#### Switch LEDs control
We can still control the other LEDs, but not the switch LEDs which blink on every packet. The LED kill switch on the
back side of the router doesn't work either.

*Workaround*: Duct tape
#### WPS
*Workaround*: None, and I don't care. Access to my precious Internets is worth typing a passphrase.

### What works
#### Wifi
But we must set the country code of AC radios to US
#### Everything else

## Networking
![Network schema](net_schema.svg.drawio)

### Conventions
| Name                    | Description                                                                         |
|-------------------------|-------------------------------------------------------------------------------------|
| `IP6_LLOCAL_FBX`        | The The Freebox Server link-local address, available in *Configuration IPv6*.       |
| `IP6_LLOCAL_ROUTER_WAN` | The OpenWrt router link-local address, WAN-side. Should be `eth0.2`.                |
| | |
| `IP6_PREFIX_FBX_0`      | The first public `/64` prefix delegated by the Freebox. We cannot use it for delegation as the Freebox server assigns itself the `::1` address. We'll pick addresses for the router itself in this block instead. |
| `IP6_PREFIX_FBX_LAN`    | The public `/64` prefix delegated by the Freebox we'll use for the *LAN* network.   |
| `IP6_PREFIX_FBX_VPN`    | The public `/64` prefix delegated by the Freebox we'll use for the *VPN* network.   |
| `IP6_PREFIX_FBX_GUEST`  | The public `/64` prefix delegated by the Freebox we'll use for the *GUEST* network. |
|                         |                                                                                     |
| `IP6_ULA_PREFIX`        | The local `/48` prefix.                                                             |
| `IP6_ULA_HINT_LAN`      | The IPv6 sub prefix we'll use for the *LAN* network. The *LAN* IPv6 block will be `<IP6_ULA_PREFIX>:<IP6_ULA_HINT_LAN>::/64`.|
| `IP6_ULA_HINT_VPN`      | The IPv6 sub prefix we'll use for the *VPN* network.                                |
| `IP6_ULA_HINT_GUEST`    | The IPv6 sub prefix we'll use for the *GUEST* network.                              |
|                         |                                                                                     |
| `IP4_GW_LAN`            | The IPv4 gateway for the *LAN* network.                                             |
| `IP4_NETMASK_LAN`       | The IPv4 netmask used for the *LAN* network, eg. `255.255.255.0`.                   |
| `IP4_GW_VPN`            | The IPv4 gateway for the *VPN* network.                                             |
| `IP4_NETMASK_VPN`       | The IPv4 netmask used for the *VPN* network, eg. `255.255.255.0`.                   |
| `IP4_GW_GUEST`          | The IPv4 gateway for the *GUEST* network.                                           |
| `IP4_NETMASK_GUEST`     | The IPv4 netmask used for the *GUEST* network, eg. `255.255.255.0`.                 |
|                         |                                                                                     |
| ÌP6_SUFFIX_GW_LAN`      | The IPv6 suffix we'll assign to the *LAN* network gateway, eg. `::1234`
|                         |                                                                                     |
| `IP6_ROUTER`            | The public IPv6 for our router, picked from `<IP6_PREFIX_FBX_0>`.                   |
| `IP6_ROUTER_LAN`        | An additional public IPv6 for our router, picked from `<IP6_PREFIX_FBX_0>`. Shouldn't be needed, but OpenWrt requires an address for all "interfaces" |
| `IP6_ROUTER_VPN`        | An additional public IPv6 for our router, picked from `<IP6_PREFIX_FBX_0>`.         |
| `IP6_ROUTER_GUEST`      | An additional public IPv6 for our router, picked from `<IP6_PREFIX_FBX_0>`.         |

### Freebox Player Configuration
* Go to *Réglages* > *Système* > *Réseau*
* Enable *Client DHCP en mode bridge*

### Freebox Server Configuration
* Set *Mode Réseau* to *Bridge*
* In *Configuration IPv6*, set *Next Hop* of chosen prefixes to `<IP6_LLOCAL_ROUTER_WAN>`.

### LAN Network
#### Configure the `wan6` interface
* Static IPv6 addresses
* IPv6 addresses
* Disable IPv6 assignment length
* Set IPv6 gateway
* Set IPv6 routed prefix

It should look like this in `/etc/config/network`:
```
config interface 'wan6'
        option ifname 'eth0.2'
        option proto 'static'
        option ip6gw '<IP6_LLOCAL_FBX>'
        option ip6prefix '<IP6_PREFIX_FBX_LAN>/64'
        list ip6addr '<IP6_ROUTER>'
        list ip6addr '<IP6_ROUTER_LAN>'
```
#### Configure SSID(s)
* Create them
* Assign them to the `lan` network

#### Configure the `lan` interface
* Static address
* Set IPv4 and netmask
* 



