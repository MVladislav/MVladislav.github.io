---
layout: post
title: OPNsense Setup
date: 2023-05-20 03:20:00 +0200
categories:
  - Infrastructure
  - OPNsense
tags:
  - security
  - hardening
  - linux
  - opnsense
  - firewall
image: /assets/img/headers/firewall.png
---

## \\\\ Information - OPNsense

OPNsense is a free and open-source firewall and routing platform that is based on FreeBSD.
It is designed to provide network security and other network services like VPN and DNS,
with a web-based graphical interface for easy configuration and management.

OPNsense offers a wide range of features and capabilities, such as:

- Firewall: OPNsense includes a stateful packet filtering firewall that can block
  unwanted traffic and protect the network from external threats.
  It also includes features such as port forwarding, NAT, and traffic shaping.
- VPN: OPNsense supports various VPN protocols, such as OpenVPN and IPsec,
  for secure remote access to the network.
- Intrusion Detection and Prevention System (IDPS): OPNsense includes Suricata,
  an open-source IDPS that can detect and prevent network intrusions.
- Web Filtering: OPNsense offers web filtering capabilities that can block access
  to specific websites and content categories. It also includes Sensei,
  a next-generation web filtering and security platform that uses artificial intelligence and
  machine learning to provide more advanced web filtering capabilities.
- DNS and DHCP: OPNsense can act as a DNS and DHCP server, providing these network services
  to devices on the network.
- High Availability: OPNsense supports high availability configurations,
  allowing for redundancy and failover in case of hardware or network failures.
- ZenArmor: ZenArmor is an add-on to OPNsense that provides advanced
  threat intelligence and security analytics capabilities.
  It uses machine learning algorithms to detect and prevent cyber attacks in real time.

OPNsense is typically used by organizations of all sizes to protect their networks and
ensure secure access to their resources. It is popular among IT professionals and
network administrators for its flexibility, ease of use, and community support.
With the addition of Sensei and ZenArmor, OPNsense provides even more
advanced security capabilities to protect against modern cyber threats.

---

> Before making any changes to software, systems, or devices,
> it's **important to thoroughly read and understand the configuration options**,
> and verify that the proposed changes align with your requirements.
> This can help avoid unintended consequences and ensure the software, system, or device operates as intended.
> {: .prompt-warning }

---

## \\\\ System > \*

### // Access

> create users (especial for admin) access with TOTP
> {: .prompt-tip }

- create **under Server** a new service for use **TOTP**
  > need to be selected after created under 'System > Settings > Administration' under topic 'Authentication' at point 'Server'
  - ![TOTP Service](/assets/img/posts/opnsense/TOTP_Service_1683906505312_0.png)
- when **TOTP** service is created, you can create for each user at point **OTP seed** the seed for the **TOTP**

### // Firmware > Plugins

> install some useful plugins
> {: .prompt-info }

- **[os-sensei](#-zenarmor--)**
  - os-sensei-updater
  - os-sunnyvalley
- os-wireguard
- **[os-theme-vicuna](#general)**
  > **!!!** highly recommended to install for dark mode, **bugs love light** **!!!**
  > {: .prompt-danger }
- _[os-ddclient](#-dynamic-dns)_
  > install if you need associate a domain name with a changing IP address
- _os-net-snmp_
  > install if you need to monitor over snmp
- _os-qemu-guest-agent_
  > install if run as virtualization, for example on proxmox

### // Gateways > Single

- you need first setup an WAN-interface
- add to the auto create gateways some additional configs
  - Disable Gateway Monitoring: **unchecked**
  - Monitor IP: **9.9.9.9**
    - _or any other public ip to monitor if the network is up_

### // Settings > \*

#### Administration

- Web GUI
  - Protocol: **https**
  - SSL Certificate: **_create one under '[System > Trust > \*](#-trust--)'_**
  - HTTP Strict Transport Security: **checked**
  - TCP port: **8443**
  - Listen Interfaces: **_select your admin/managemant subnet, to not allow access from every subnet_**
- Secure Shell
  - Secure Shell Server: **unchecked**
    > _only activate if needed_
  - SSH port: **2233**
    > _optional_
  - Listen Interfaces: **_select your admin/managemant subnet, to not allow access from every subnet_**
- Authentication:
  - Server: **_choose TOTP service we setup before under [Access](#-access)_**

example for whole setup:

![Administration Settings System](/assets/img/posts/opnsense/Administration_Settings_System_1683907879037_0.png)

#### General

- Domain: **home.local**
- Time zone: **Europe/Berlin**
- Theme: **vicuna**
  > **!!!** highly recommended switch to dark mode, **bugs love light** **!!!**
  > {: .prompt-danger }
- Prefer IPv4 over IPv6: **unchecked**
- DNS servers: **_not need to be setup, will be used by 'Services > Unbound DNS'_**
- DNS server options: **both disable**
- Gateway switching: **unchecked**

example for whole setup:

![General Settings System](/assets/img/posts/opnsense/General_Settings_System_1683907826749_0.png)

#### Logging

![Logging Settings System](/assets/img/posts/opnsense/Logging_Settings_System_1683907923484_0.png)

#### Tunables

perform some tuneing to improve the system

```properties
net.inet.tcp.tso=1 # TCP Offload Engine
net.inet.tcp.soreceive_stream=1 # optimized kernel socket interface
net.inet.tcp.mssdflt=1460 # improve efficiency while processing IP fragments
net.inet.tcp.abc_l_var=52 # improve efficiency while processing IP fragments
net.inet.tcp.initcwnd_segments=44 #
net.inet.tcp.minmss=536 # minimum segment size, or smallest payload of data which a single IPv4 TCP segment will agree to transmit

net.inet.udp.checksum=1 # UDP Checksums

net.inet.tcp.recvbuf_max=4194304 # Max size of automatic receive buffer
net.inet.tcp.recvspace=65536 # Maximum incoming/outgoing TCP datagram size (receive)
net.inet.tcp.sendbuf_inc=65536 # Incrementor step size of automatic send buffer
net.inet.tcp.sendbuf_max=4194304 # Max size of automatic send buffer
net.inet.tcp.sendspace=65536 # Maximum incoming/outgoing TCP datagram size (send)

kern.ipc.maxsockbuf=16777216 # Maximum socket buffer size, for net speed (10GB)
kern.ipc.nmbclusters=1000000 # Maximum number of mbuf cluster allowed
kern.ipc.nmbjumbop=524288 # Maximum number of mbuf page size jumbo cluster allowed
kern.random.fortuna.minpoolsize=128 # RNG entropy pool for vpn

net.pf.source_nodes_hashsize=1048576 # PF firewall hash table size

# Receive-side scaling (https://docs.opnsense.org/troubleshooting/performance.html)
net.isr.maxthreads=-1 # uncaps the amount of CPU’s which can be used for netisr processing
net.isr.bindthreads=1 # binds each of the ISR threads to 1 CPU core
net.inet.rss.enabled=1 # enable Receive Side Scaling
# for 4-core systems, use ‘2’
# for 8-core systems, use ‘3’
# for 16-core systems, use ‘4’
net.inet.rss.bits=3 # amount of bits representing the number of CPU cores (3=8 cores)

net.isr.dispatch=deferred # netisr dispatch policy
net.isr.defaultqlimit=2048 # Default netisr per-protocol, per-CPU queue limit if not set by protocol

hw.ibrs_disable=1 # Disable Indirect Branch Restricted Speculation (Spectre V2 mitigation) changed from default to 1
# vm.pmap.pti = 0 # Page Table Isolation (Meltdown mitigation, requires reboot.)

hw.ix.enable_aim=1 # Enable adaptive interrupt moderation
hw.ix.flow_control=0 # Default flow control used for all adapters
hw.ixl.enable_head_writeback=0 # For detecting last completed TX descriptor by hardware, use value written by HW instead of checking descriptors
hw.vtnet.lro_disable=1 # Disable hardware LRO

net.inet.ip.intr_queue_maxlen=2048 # Maximum size of IP input queue
net.inet.ip.maxfragpackets=2048 # Maximum number if IPv4 fragment reassembly queue entries
net.inet.ip.maxfragsperpacket=1000 # Maximum number of IPv4 fragments allowed per packet
net.route.multipath=1 # Enable route multipath

hw.intr_storm_threshold=9000 # Number of consecutive interrupts before storm protection is enabled
hw.pci.honor_msi_blacklist=0 # Honor chipset blacklist for MSI/MSI-X

# net.inet6.ip6.intr_queue_maxlen = 4096
# net.link.ifqmaxlen = 2048
# net.route.netisr_maxqlen = 4096
```

### // Trust > \*

#### Authorities

- create a **internal-ca** and **intermediate-ca** to manage internal certifications
- **internal-ca**
  - ![Authorities Trust System](/assets/img/posts/opnsense/Authorities_Trust_System_1683909084329_0.png)
- **intermediate-ca**
  - ![Authorities Trust System](/assets/img/posts/opnsense/Authorities_Trust_System_1683909045392_0.png)

#### Certificate

create an **OPNsense certificate**, by using the **intermediate-ca**, to handle the OPNsense web-access with TLS

![Certificates Trust System](/assets/img/posts/opnsense/Certificates_Trust_System_1683909316305_0.png)

## \\\\ Interfaces > \*

- Assignments
  - assign your network ports as interfaces
  - edit your assigned interfaces
    - example for WAN as PPPOE
    - ![7_WAN Interfaces](/assets/img/posts/opnsense/7_WAN_Interfaces_1684545473693_0.png)
- Settings
  - Disable mostly all **hardware offload**
  - ![Settings Interfaces](/assets/img/posts/opnsense/Settings_Interfaces_1683910123930_0.png)

## \\\\ Firewall > \*

### // Aliases > \*

#### Aliases

- Type: **Host(s)**
  - IP_S_DNS_NTP_INTERN
    - Content: **`192.168.1.1,fd00:affe:affe:0001::1`**
    - Statistics: **checked**
    - Description: **IP: service internal DNS,NTP (IPv 4+6)**
- Type: **Network(s)**
  - SUB_RFC1918
    - Content: **`10.0.0.0/8,172.16.0.0/12,192.168.0.0/16`**
    - Statistics: **checked**
    - Description: **SUB: RFC 1918 private network standard**
  - SUB_RFC1918_BOGON_LOCAL
    - Content: **`10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,127.0.0.0/8,224.0.0.0/4,255.255.255.255,fe80::/10,::1/128,ff00::/8,fc00::/7,ff02::1,ff02::2,2001:db8::/32`**
    - Statistics: **checked**
    - Description: **SUB: RFC1918 + Bogon + Local + Multicast + Broadcast**
  - IP_S_MULTI_BROAD
    - Content: **`ff00::/8,224.0.0.0/4,255.255.255.255,ff02::1,ff02::c`**
    - Statistics: **checked**
    - Description: **IP: sub multicast + broadcast addresses (IPv 4+6)**
  - SUB_6_GLOBAL_PUBLIC
    - Content: **`2000::/3`**
    - Statistics: **checked**
    - Description: **SUB: IPv6 Global Unicast**
  - SUB_6_UNIQUE_LOCAL_ADDRESS
    - Content: **`fd00::/8`**
    - Statistics: **checked**
    - Description: **SUB: IPv6 Unique Local Address**
- Type: **Port(s)**
  - PORT_DNS_BLOCK
    - Content: **`53,853,5353,5355,9953`**
    - Description: **PORT: DNS block ports (for outside connections)**
  - PORT_LB_NO_LOG
    - Content: **`53,2055,9200`**
    - Description: **PORT: loopback disable log for ports**

#### GeoIP settings

add following api url with your `<LICENSE-KEY>`

`https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country-CSV&license_key=<LICENSE-KEY>&suffix=zip`

### // Groups

- create groups as you need, for example create a default group for allow access to public network
  > you need later to create the firewall rules
  - Name: **G_PUB_NET_D**
  - Description: **default public network access**
  - Members: **_add the interfaces you want to allow access the network with default firewall rules_**
  - GUI groups: **checked**

### // NAT > Port Forward

for only allow using NTP from internal service,
we forward all traffic to NTP (123) to the firewall (a block firewall rule will not be created)

![Port Forward NAT Firewall](/assets/img/posts/opnsense/Port_Forward_NAT_Firewall_1684418308939_0.png)

- Description: **PF:: LAN: forward NTP to local IP**
  - Interface: **_select all interfaces, where you want forward requests_**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **UDP**
  - Source / Invert: **unchecked**
  - Source: **SUB_RFC1918**
  - Destination / Invert: **checked**
  - Destination: **IP_S_DNS_NTP_INTERN**
  - Destination port range: **123 - 123**
  - Redirect target IP: **IP_S_DNS_NTP_INTERN**
  - Log: **checked**
  - Filter rule association: **None**

### // Rules

#### Floating

![Floating Rules Firewall](/assets/img/posts/opnsense/Floating_Rules_Firewall_1684417217425_0.png)

> TODO: this part needs to be updated

- Description: **BLOCK:: F: all mdns on port 5353**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **any**
  - Destination port range: **5353 - 5353**
  - Log: **checked**
- Description: **BLOCK:: F: all mdns on ip address**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **IP_FILTER_MDNS**
  - Destination port range: **any - any**
  - Log: **checked**
- Description: **BLOCK:: F: DNS (outside)**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **checked**
  - Destination: **IP_S_DNS_NTP_INTERN**
  - Destination port range: **PORT_DNS_BLOCK - PORT_DNS_BLOCK**
  - Log: **checked**
- Description: **ALLOW: f:: NTP requested to internal NTP**
  - Action: **Pass**
  - Quick: **checked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **UDP**
  - Source / Invert: **unchecked**
  - Source: **SUB_RFC_1918**
  - Destination / Invert: **unchecked**
  - Destination: **IP_S_DNS_NTP_INTERN**
  - Destination port range: **NTP - NTP**
  - Log: **checked**
- Description: **BLOCK:: F: no rule - ipv4**
  - Action: **Block**
  - Quick: **unchecked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPv4**
  - Protocol: **any**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **any**
  - Log: **checked**
- Description: **BLOCK:: F: no rule - ipv6**
  - Action: **Block**
  - Quick: **unchecked**
  - Interface: **nothing selected**
  - Direction: **in**
  - TCP/IP Version: **IPV6**
  - Protocol: **any**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **any**
  - Log: **checked**

#### WAN

![WAN Rules Firewall](/assets/img/posts/opnsense/WAN_Rules_Firewall_1684416446355_0.png)

- Description: **BLOCK:: WAN: no rule - ipv4**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **WAN**
  - Direction: **in**
  - TCP/IP Version: **IPv4**
  - Protocol: **any**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **any**
  - Log: **checked**
- Description: **BLOCK:: WAN: no rule - ipv6**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **WAN**
  - Direction: **in**
  - TCP/IP Version: **IPV6**
  - Protocol: **any**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **unchecked**
  - Destination: **any**
  - Log: **checked**

#### G_PUB_NET_D

![G_PUB_NET_D Rules Firewall](/assets/img/posts/opnsense/GROUP_N_NET_D_Rules_Firewall_1684416035121_0.png)

- Description: **BLOCK:: GPND: DNS (outside)**
  - Action: **Block**
  - Quick: **checked**
  - Interface: **G_PUB_NET_D**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **any**
  - Destination / Invert: **checked**
  - Destination: **IP_S_DNS_NTP_INTERN**
  - Destination port range: **PORT_DNS_BLOCK - PORT_DNS_BLOCK**
  - Log: **checked**
- Description: **ALLOW:: GPND: DNS (inside)**
  - Action: **Pass**
  - Quick: **checked**
  - Interface: **G_PUB_NET_D**
  - Direction: **in**
  - TCP/IP Version: **IPv4+IPV6**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **G_PUB_NET_D net**
  - Destination / Invert: **unchecked**
  - Destination: **IP_S_DNS_NTP_INTERN**
  - Destination port range: **DNS - DNS**
  - Log: **checked**
- Description: **ALLOW:: GPND: internet access (ipv4)**
  - Action: **Pass**
  - Quick: **checked**
  - Interface: **G_PUB_NET_D**
  - Direction: **in**
  - TCP/IP Version: **IPv4**
  - Protocol: **any**
  - Source / Invert: **unchecked**
  - Source: **G_PUB_NET_D net**
  - Destination / Invert: **checked**
  - Destination: **SUB_RFC1918_BOGON_LOCAL**
  - Log: **checked**

#### LOOPBACK

- Description: **ALLOW:: LB:: 9200**
  - Action: **Pass**
  - Quick: **checked**
  - Interface: **Loopback**
  - Direction: **out**
  - TCP/IP Version: **IPv4**
  - Protocol: **TCP/UDP**
  - Source / Invert: **unchecked**
  - Source: **127.0.0.1/32**
  - Destination / Invert: **unchecked**
  - Destination: **127.0.0.1/32**
  - Destination port range: **PORT_LB_NO_LOG - PORT_LB_NO_LOG**
  - Log: **unchecked**

### // Shaper

#### Pipes

- WAN_down_P
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683912807478_0.png)
- WAN_up_P
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683912850403_0.png)

#### Queues

- WAN_down_Q
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683912883015_0.png)
- WAN_up_Q
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683913014993_0.png)

#### Rules

- WAN_down_S
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683912973653_0.png)
- WAN_up_S
  - ![Shaper Firewall](/assets/img/posts/opnsense/Shaper_Firewall_1683912953255_0.png)

### // Settings > Advanced

- Disable anti-lockout: **checked**

example for whole setup:

TODO: fix image

![Advanced Settings Firewall](/assets/img/posts/opnsense/Advanced_Settings_Firewall_1683913177897_0.png)

## \\\\ Services > \*

### // DHCPv4 > \*

when creating subnets,
following fields are recommended to be filled to configure the dhcp service:

- Enable: **checked**
- Range: **_specify DHCP range for automatic IP assignment to devices_**
- DNS servers: **192.168.1.1**
  > _remember we defined the **DNS** server under **[Aliases](#aliases)** with the alias `IP_S_DNS_NTP_INTERN`_
- Gateway: **_set the gateway for your subnet_**
- NTP servers: **192.168.1.1**
  > _remember we defined the **NTP** server under **[Aliases](#aliases)** with the alias `IP_S_DNS_NTP_INTERN`_

### // Dynamic DNS

#### Settings

- Cloudflare Example
  - Enabled: true
  - Service: Cloudflare
  - Username: E-Mail address of your account
  - Password: Global API key for your account (Open Cloudflare > My Account > API Tokens > Global API Key > View)
  - Zone: your.domain (e.g. example.com)
  - Hostname: full domain name you want to update (e.g. dyn.example.com)
  - Check ip method: Interface
  - Force SSL: true
  - Interface to monitor: WAN

#### General Settings

- Enable: **checked**
- Interval: **300**

### // Intrusion Detection

#### Settings

![Administration Intrusion Detection Services](/assets/img/posts/opnsense/Administration_Intrusion_Detection_Services_1683913278351_0.png)

#### Download

- select **all** and click **Download & Update Rules**
- select **all** and click **Enable selected**

#### Schedule

![Cron Settings System](/assets/img/posts/opnsense/Cron_Settings_System_1683913306303_0.png)

### // Monit

#### General Settings

add your information and credentials

#### Alert Settings

create new entry, below an example for mail format to be added

Mail format ([info](https://mmonit.com/monit/documentation/monit.html#Message-format)):

> update `monit <no-reply@DOMAIN>` in **from** to your needs

```yaml
from: monit <no-reply@DOMAIN>
subject: $SERVICE $EVENT at $DATE
message: Monit $ACTION $SERVICE at $DATE on $HOST:
  $DESCRIPTION

Yours sincerely,
Monit
```

### // Unbound DNS

#### General

![General Unbound DNS Services](/assets/img/posts/opnsense/General_Unbound_DNS_Services_1684415442105_0.png)

#### Advanced

![Advanced Unbound DNS Services](/assets/img/posts/opnsense/Advanced_Unbound_DNS_Services_1683913593814_0.png)

#### DNS over TLS

![DNS over TLS Unbound DNS Services](/assets/img/posts/opnsense/DNS_over_TLS_Unbound_DNS_Services_1683913789510_0.png)

| Enabled | Address              | Port | Hostname        |
| ------- | -------------------- | ---- | --------------- |
| x       | 194.242.2.2          | 853  | doh.mullvad.net |
|         | 2a07:e340::2         | 853  | doh.mullvad.net |
|         | 149.112.112.112      | 853  | dns.quad9.net   |
| x       | 9.9.9.9              | 853  | dns.quad9.net   |
|         | 2620:fe::9           | 853  | dns.quad9.net   |
|         | 2620:fe::fe          | 853  | dns.quad9.net   |
|         | 1.1.1.1              | 853  | one.one.one.one |
|         | 2606:4700:4700::64   | 853  | one.one.one.one |
|         | 2606:4700:4700::6400 | 853  | one.one.one.one |
|         | 1.0.0.1              | 853  | one.one.one.one |
|         | 8.8.4.4              | 853  | dns.google      |
|         | 8.8.8.8              | 853  | dns.google      |

## \\\\ Zenarmor > \*

### // Policies (Default)

#### Security

![Policies Zenarmor](/assets/img/posts/opnsense/Policies_Zenarmor_1683914462056_0.png)

#### App Controls

- Cloud Services
  - Apple Cloud
- Conferencing
  - Google Hangouts Meet
- Gaming
  - Facebook Games
  - Fortnite
  - Fortnite Tracker
  - Microsoft Xbox
  - Roblox Game
  - Samsung Games
- Instant Messaging
  - Facebook Chat
  - Facebook Messenger
  - Facebook Video call
  - Google Chat
  - Google Hangouts
- Media Streaming
  - Apple\*
- Mobile Applications
  - Amazon Firestick TV
- News
  - Apple News
  - Bild.de
- Online Shopping
  - Apple Appstore
  - Apple Store
  - Microsoft Wallet
- Online Utility
  - Apple\*
  - Microsoft Cortana
  - Microsoft MSDN
  - Microsoft Weather
  - Pivotal Tracker
- Proxy
  - iCloud Private Relay
- Remoute Access
  - all except
    - Microsoft Continuum
    - Secure Shell
    - Teamviewer
- Search
  - Microsoft Bing
- Social Network
  - Facebook\*
  - Google\*
  - facebook.comment
  - facebook.statusUpdate
- Software Updates
  - Apple Pipeline
  - Apple Telemetry
  - Intel Telemetry
  - Malwarebytes Telemetry
  - Microsoft Telemetry
  - Mozilla Telemetry
  - Windows Problem Reporting
- Storage & Backup
  - all except
    - Google Drive
    - Microsoft OneDrive
- VOIP
  - Facebook Call

![Policies Zenarmor](/assets/img/posts/opnsense/Policies_Zenarmor_1683915468914_0.png)

#### Web Controls

![Policies Zenarmor](/assets/img/posts/opnsense/Policies_Zenarmor_1683914518793_0.png)

### // Configuration > \*

#### General

- choose: **Routed Mode (L3 Mode, Reporting + Blocking) with native netmap driver**
- select your **Interfaces Selection**
- defined your needs for **Deployment**
- set your needs for **Logger**

#### Cloud Threat Intel

- Local Domains Name To Exclude From Cloud Queries: **`home.local,local`**

#### Updates & Health

- Help Sunny Valley Networks improve its products and services by sharing health and system utilization statistics: **unchecked**

#### Reporting & Data

- set your needs for **Reports Data Management**
- active **Scheduled Reports** if needed

#### Privacy

![Configuration Zenarmor](/assets/img/posts/opnsense/Configuration_Zenarmor_1683914355203_0.png)

---

## \\\\ Resources & More information's

- <https://docs.opnsense.org/>{:target="\_blank"}
- <https://en.wikipedia.org/wiki/Private_network>{:target="\_blank"}
- <https://ipgeolocation.io/resources/bogon.html>{:target="\_blank"}
- <https://forum.opnsense.org/index.php?PHPSESSID=ahi5e19a2tl303rir594sgmn88&topic=27394.msg160740#msg160740>{:target="\_blank"}
- tunings
  - <https://docs.opnsense.org/troubleshooting/performance.html>{:target="\_blank"}
  - <https://teklager.se/en/knowledge-base/opnsense-performance-optimization>{:target="\_blank"}
  - <https://binaryimpulse.com/2022/11/opnsense-performance-tuning-for-multi-gigabit-internet>{:target="\_blank"}
  - <https://www.reddit.com/r/OPNsenseFirewall/comments/b2uhpw/performance_tuning_help>{:target="\_blank"}
  - <https://calomel.org/freebsd_network_tuning.html>{:target="\_blank"}
  - <https://docs.opnsense.org/troubleshooting/performance.html>{:target="\_blank"}
  - <https://www.reddit.com/r/opnsense/comments/14li2c7/10gbps_speed/>{:target="\_blank"}
  - <https://www.reddit.com/r/opnsense/comments/17fjbbw/opnsense_on_proxmox_10gb_network_woes/>{:target="\_blank"}
  - <https://forum.opnsense.org/index.php?topic=31830.0>{:target="\_blank"}
  - <https://forum.opnsense.org/index.php?topic=18754.150>{:target="\_blank"}
