---
layout: post
title: Proxmox Setup
date: 2023-05-20 04:50:00 +0200
categories:
  - Infrastructure
  - Proxmox
tags:
  - security
  - hardening
  - linux
  - proxmox
image: /assets/img/headers/proxmox.png
---

## \\\\ Information - Proxmox

Proxmox is an open-source virtualization management platform that combines the power of virtualization
and containerization. It allows you to run and manage virtual machines (VMs) and containers on a single
server, providing a flexible and efficient infrastructure for hosting and managing your applications.

At its core, Proxmox is built on two key components:

- KVM (Kernel-based Virtual Machine) for hardware-based virtualization and
- LXC (Linux Containers) for lightweight containerization.

This combination offers a versatile environment where you can create and deploy different types
of workloads, whether they require full isolation and dedicated resources (VMs) or share the underlying
operating system (containers).

Proxmox provides a web-based interface that makes it easy to manage your virtualization infrastructure.
Through the intuitive interface, you can create, configure, and monitor VMs and containers, allocate resources,
and perform live migrations between hosts without service interruptions. The platform also offers a rich
set of features, including high availability clustering, backup and restore capabilities, and comprehensive
monitoring tools to ensure the reliability and performance of your virtualized environment.

One of the significant advantages of Proxmox is its open-source nature, which means it is freely available
for use and can be customized and extended to meet specific requirements. This makes it an ideal choice for
small to medium-sized businesses, educational institutions, and even enthusiasts who want to set up their
own virtualization infrastructure without incurring high costs.

Proxmox is useful in a variety of scenarios, including server consolidation, development and testing
environments, cloud hosting, and building private or hybrid cloud infrastructures. Its ability to efficiently
manage both VMs and containers provides flexibility and enables you to choose the most appropriate
approach for your specific workloads.

In summary, Proxmox is a powerful and user-friendly virtualization management platform that offers the
benefits of both virtual machines and containers. With its extensive feature set, ease of use, and
open-source nature, Proxmox provides an efficient and cost-effective solution for organizations and
individuals seeking to leverage the benefits of virtualization in their computing environments.

---

> Before making any changes to software, systems, or devices,
> it's **important to thoroughly read and understand the configuration options**,
> and verify that the proposed changes align with your requirements.
> This can help avoid unintended consequences and ensure the software, system, or device operates as intended.
> {: .prompt-warning }

---

## \\\\ Change Subscription Mode & Updatable Repo

> _Add/Change to **no-subscription**-repo if you need,
> else use the enterprise-repo when you have a subscription_

create a new file and past the **no-subscription**-repo into it (**[INFO](https://pve.proxmox.com/wiki/Package_Repositories#sysadmin_no_subscription_repo)**):

```sh
$echo 'deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription' \
> /etc/apt/sources.list.d/pve.list
```

commit the existing **enterprise**-repo (**[INFO](https://pve.proxmox.com/wiki/Package_Repositories#sysadmin_enterprise_repo)**):

```sh
$sed -i '1 s/^[^#]/#$1 /' /etc/apt/sources.list.d/pve-enterprise.list
```

remove the popup for non existing subscription ('No valid sub'):

```sh
$sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" \
/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js \
&& systemctl restart pveproxy.service
```

## \\\\ ZFS config's

> _Check for values you need, this change is mostly for smaller env. needed
> where less RAM is available or less RAM with a bigger hard-drive size._
>
> _Especially we can reduce the RAM usage for ARC on proxmox it self,
> because we for example are not a file server._
>
> > The size of the ARC, which is managed by ZFS, depends on the available RAM in the system.
> > A larger ARC can hold more data in memory, thereby reducing the number of disk accesses
> > required for read operations. This caching mechanism can greatly improve the overall
> > performance of a ZFS system.
> >
> > While RAM size influences the ARC's capacity and the amount of data that can be cached,
> > it's essential to consider the disk size as well. The overall storage capacity of the
> > disks in a ZFS system determines the amount of data that can be stored persistently.
> > Although ZFS is capable of compressing and deduplicating data to optimize disk space
> > utilization, the physical capacity of the disks plays a significant role in determining
> > the maximum amount of data that can be stored in the system.

### // ARC

create a file under **modprobe** and set **arc** `min` & `max`:

> _This example is for available RAM of 64GB
> to only allow a max usage of 8G of RAM for ARC_
>
> **_do not set less than 8G_**
>
> > - zfs_arc_min:
> >   Specifies the minimum size of the ARC, ensuring a baseline cache for performance during memory pressure.
> > - zfs_arc_max:
> >   Sets the maximum size of the ARC. Useful for managing memory resources and optimizing cache size for specific workloads.

```sh
# calc: 4 * 1024 * 1024 * 1024 = 4294967296
$echo 'options zfs zfs_arc_min=4294967296' > /etc/modprobe.d/zfs.conf
# calc: 8 * 1024 * 1024 * 1024 = 8589934592
$echo 'options zfs zfs_arc_max=8589934592' >> /etc/modprobe.d/zfs.conf

# on-disk checksum verification during the pool import process, helping to ensure data integrity
#options zfs zfs_flags=0x10
```

run refresh command to update modprobe changes to system:

```sh
$update-initramfs -u
$pve-efiboot-tool refresh
```

### // KSM

update in the **ksmtuned** file the line `KSM_THRES_COEF`:

> _Possible example for 64GB of RAM_
>
> > The `KSM_THRES_COEF` parameter controls the level of memory sharing and deduplication
> > performed by the Kernel Samepage Merging (KSM) feature in the Linux kernel.
> >
> > By adjusting `KSM_THRES_COEF`, you can specify the threshold for memory page merging.
> > Higher values encourage more aggressive merging, resulting in increased memory savings,
> > but potentially higher CPU usage. Lower values reduce merging, conserving CPU resources
> > but potentially reducing memory savings.
>
> > Please note that opting for KSM results in a trade-off wherein
> > the attack surface for **potential side channel exploits** is heightened.
> > To avoid you can also disable it by run: `systemctl disable ksmtuned`

```sh
$sed -i 's/KSM_THRES_COEF=.*/KSM_THRES_COEF=35/' /etc/ksmtuned.conf
```

restart service to perform the change:

```sh
$systemctl restart ksmtuned.service
```

## \\\\ SWAP remove

for proxmox we do not need swap usage, so we will disable it, if it not always disabled.

open `/etc/fstab` and comment the line where **swap** is defined, for example:

```properties
...
# /dev/mapper/cryptoswap none    swap    sw      0       0
...
```

also disable the current used swap on runtime:

```sh
$swapoff -a
```

## \\\\ SMTP Configuration

> Since version `> 8.1` it is possible to setup SMTP over gui under `Datacenter > Notifications`.
> Manual config below is not anymore needed.

### // GUI Setup

over the gui set up email addresses for each users
and define default options for sending emails from.

set up the default email address from which mails will be sent from:

- open page under `Datacenter > Options`
- edit the field `Email from address`

specify individual email addresses for each user to receive mail-related notifications:

- open page under `Datacenter > Permissions > Users`
- edit each user and set the field `E-Mail`

### // Terminal Setup

install **dependencies**:

```sh
$apt install libsasl2-modules mailutils postfix-pcre
```

**open** the file `/etc/postfix/main.cf` and **configure** the **Postfix main configuration**, for example like follow:

> replace:
>
> > - `<SERVER_HOSTNAME_OR_IP>`
> > - `<YOUR_SMPT_HOST>`

```properties
# See /usr/share/postfix/main.cf.dist for a commented, more complete version

# Set the SMTP server hostname or IP address
myhostname = <SERVER_HOSTNAME_OR_IP>

smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# Do not append .domain to local addresses
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
delay_warning_time = 4h

alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mynetworks = 127.0.0.0/8
inet_interfaces = loopback-only
recipient_delimiter = +

compatibility_level = 2
inet_protocols = ipv4

# Specify the relay host and its port
relayhost = [<YOUR_SMTP_HOST>]:465

# SMTP authentication settings
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous

# Use TLS for SMTP connections
smtp_use_tls = yes

# Enable the SMTPS wrapper mode
smtp_tls_wrappermode = yes

# Set the required TLS security level for SMTPS
smtp_tls_security_level = encrypt

# TLS protocol versions
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, TLSv1.1, TLSv1.2

# TLS ciphers
smtp_tls_ciphers = high

# CA file path
smtp_tls_CAfile = /etc/ssl/certs/Entrust_Root_Certification_Authority.pem

# Enable SMTP TLS session caching
smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
smtp_tls_session_cache_timeout = 3600s
```

**create** a new file for storing the **SASL password**:

```sh
$nano /etc/postfix/sasl_passwd
```

**add** the following line to the file with your **SMTP server details**:

> replace:
>
> > - `<SMTP_HOST>`
> > - `<SMTP_USERNAME>`
> > - `<SMTP_PASSWORD>`

```properties
[<SMTP_HOST>]:465    <SMTP_USERNAME>:<SMTP_PASSWORD>
```

**hash** the **sasl_passwd** file by running the following command:

```sh
$chmod 600 /etc/postfix/sasl_passwd
$postmap hash:/etc/postfix/sasl_passwd
```

**restart** the **daemon** and **Postfix** service to apply the changes:

```sh
$systemctl daemon-reload
$postfix reload && systemctl restart postfix
```

**verify** by send a test mail:

> replace:
>
> > - `<SENDER_ADDRESS>`
> > - `<RECIPIENT_ADDRESS>`

```sh
$echo "Test mail from postfix" | mail -r <SENDER_ADDRESS> -s "Test Postfix" <RECIPIENT_ADDRESS>
```

## \\\\ SSH

SSH hardening is crucial for maintaining the security and integrity of your system.
It provides stronger authentication, protects against brute-force attacks, ensures encryption
and data integrity, allows for granular access control, helps meet security compliance requirements,
and defends against vulnerabilities. By implementing SSH hardening practices, you can significantly
reduce the risk of unauthorized access and protect your sensitive information.

upload an **ssh-public-key** into `/root/.ssh/authorized_keys` for future logins.
This will allow you to login using your public key once the subsequent hardening steps are in place.

update the **root** `/root/.ssh/config` file, like follow:

```properties
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
# ~/.ssh/config

# RSA keys are favored over ECDSA keys when backward compatibility ''is required'',
# thus, newly generated keys are always either ED25519 or RSA (NOT ECDSA or DSA).
# ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_comment_$(date +%Y_%m_%d)     -C "$(hostname)-$(date +%Y-%m-%d)-comment"

# ED25519 keys are favored over RSA keys when backward compatibility ''is not required''.
# This is only compatible with OpenSSH 6.5+ and fixed-size (256 bytes).
# ssh-keygen -t ed25519     -f ~/.ssh/id_ed25519_comment_$(date +%Y_%m_%d) -C "$(hostname)-$(date +%Y-%m-%d)-comment"
# ssh-keygen -t ed25519-sk  -f ~/.ssh/id_ed25519_comment_$(date +%Y_%m_%d) -C "$(hostname)-$(date +%Y-%m-%d)-comment"
# ------------------------------------------------------------------------------
# Ensure KnownHosts are unreadable if leaked - it is otherwise easier to know which hosts your keys have access to.
HashKnownHosts yes
# Host keys the client accepts - order here is honored by OpenSSH
HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,sk-ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com

KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-128-etm@openssh.com
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

Host *
  User root
  Port 22
  LogLevel INFO
  Compression yes
  SendEnv LANG LC_*
  HashKnownHosts yes
  GSSAPIAuthentication yes
  IdentitiesOnly yes
  AddressFamily inet
  Protocol 2
  ServerAliveInterval 60
```

harden the SSH configuration by open `/etc/ssh/sshd_config` and setup like follow:

> _Note: the property `PermitRootLogin` is **not** setup as **recommended** because we not create a non root user_

```properties
Include /etc/ssh/sshd_config.d/*.conf

Port 22
AddressFamily inet
ListenAddress 0.0.0.0

HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

# Ciphers and keying
#RekeyLimit default none

# Logging
#SyslogFacility AUTH
LogLevel VERBOSE

# Authentication:

LoginGraceTime 60
# on proxmox we have not create a non root user
# so this is not setup as recommended, recommended would be 'PermitRootLogin no'
PermitRootLogin yes
StrictModes yes
MaxAuthTries 4
MaxSessions 10

PubkeyAuthentication yes

# Expect .ssh/authorized_keys2 to be disregarded by default in future.
AuthorizedKeysFile .ssh/authorized_keys

#AuthorizedPrincipalsFile none

#AuthorizedKeysCommand none
#AuthorizedKeysCommandUser nobody

# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts
HostbasedAuthentication no
# Change to yes if you don't trust ~/.ssh/known_hosts for
# HostbasedAuthentication
#IgnoreUserKnownHosts no
# Don't read the user's ~/.rhosts and ~/.shosts files
IgnoreRhosts yes

# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no
PermitEmptyPasswords no

# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
KbdInteractiveAuthentication no

# Kerberos options
#KerberosAuthentication no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes
#KerberosGetAFSToken no

# GSSAPI options
GSSAPIAuthentication no
GSSAPICleanupCredentials yes
#GSSAPIStrictAcceptorCheck yes
#GSSAPIKeyExchange no

# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the KbdInteractiveAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via KbdInteractiveAuthentication may bypass
# the setting of "PermitRootLogin without-password".
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and KbdInteractiveAuthentication to 'no'.
UsePAM yes

AllowAgentForwarding yes
AllowTcpForwarding no
#GatewayPorts no
X11Forwarding no
#X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
PrintMotd no
#PrintLastLog yes
#TCPKeepAlive yes
PermitUserEnvironment no
#Compression delayed
ClientAliveInterval 15
ClientAliveCountMax 3
UseDNS no
#PidFile /run/sshd.pid
MaxStartups 10:30:60
#PermitTunnel no
#ChrootDirectory none
#VersionAddendum none

# no default banner path
Banner /etc/issue.net

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

# override default of no subsystems
Subsystem sftp  /usr/lib/ssh/sftp-server -f AUTHPRIV -l INFO

# Example of overriding settings on a per-user basis
#Match User anoncvs
#       X11Forwarding no
#       AllowTcpForwarding no
#       PermitTTY no
#       ForceCommand cvs server
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha256
HostKeyAlgorithms ssh-ed25519-cert-v01@openssh.com,ssh-rsa-cert-v01@openssh.com,ssh-ed25519,ssh-rsa,ecdsa-sha2-nistp521-cert-v01@openssh.com,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256
AuthenticationMethods publickey
PubkeyAcceptedKeyTypes ssh-ed25519
ChallengeResponseAuthentication no
```

restart the ssh service to ensuring that the modifications take effect:

```sh
$systemctl restart ssh
```

## \\\\ Kernel and Network Tunings

open the file `/etc/sysctl.conf` and replace with following content:

> _make sure the defined values fits your needs or change them for your needs_

```properties
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

###################################################################
# ==> kernel

# https://en.wikipedia.org/wiki/Syslog#Severity_levels
# stop low-level messages on console
kernel.printk=3 4 1 3

# Enable process address space protection
# This makes it more difficult for an attacker to predict the location of key data structures in memory
kernel.randomize_va_space=2
# Disable core dumps
# This prevents core dumps from being written to disk if a setuid program crashes
fs.suid_dumpable=0

fs.inotify.max_user_instances=8192
# Cache extend
fs.inotify.max_user_watches=524288

# limit the ability of a compromised process to PTRACE_ATTACH on other processes running under the same user
kernel.yama.ptrace_scope=1

###################################################################
# Magic system request Key
# 0=disable, 1=enable all, >1 bitmask of sysrq functions
# See https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html
# for what other values do
#kernel.sysrq=438
kernel.sysrq=0

# Background save may fail under low memory condition
vm.overcommit_memory=0

# swapp set to 1, but not quite disabling it. Will prevent OOM killer from killing processes when running out of physical memory.
vm.swappiness=1

# Enable Transparent Huge Pages (THP)
vm.nr_hugepages=2048 #128
#vm.nr_hugepages_mempolicy=1

# Adjust vfs cache
# https://lonesysadmin.net/2013/12/22/better-linux-disk-caching-performance-vm-dirty_ratio/
# Decriase dirty cache to faster flush on disk
#vm.dirty_background_ratio=5
#vm.dirty_ratio=10

###################################################################
# ==> network

# Disable IPv6 usage complete, we do not need it internal
# think about let it active on device which needs it
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1

# Do not send ICMP redirects (we are not a router)
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0

# Disable IP forwarding for IPv4 & IPv6
# This prevents the system from forwarding packets between interfaces, which can help prevent man-in-the-middle attacks
net.ipv4.ip_forward=0
net.ipv4.conf.all.forwarding=0
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forwarding=0

# Do not accept IP source route packets (we are not a router)
# This makes it more difficult for an attacker to control the path that packets take through the network
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0
net.ipv6.conf.all.accept_source_route=0
net.ipv6.conf.default.accept_source_route=0

# Do not accept ICMP redirects (prevent MITM attacks)
# This makes it more difficult for an attacker to control the routing of packets on the network
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv6.conf.all.accept_redirects=0
net.ipv6.conf.default.accept_redirects=0

# Do not accept ICMP redirects only for gateways listed in our default
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0

# Log Martian Packets (disabled)
net.ipv4.conf.all.log_martians=0
net.ipv4.conf.default.log_martians=0

# Ignore bogus ICMP errors
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
net.ipv4.icmp_echo_ignore_all=0

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.lo.rp_filter=1

# Enable TCP SYN cookie protection
# This makes it more difficult for an attacker to perform a SYN flood attack
net.ipv4.tcp_syncookies=1
# Enable the use of TCP selective acknowledgements
# This allows the system to acknowledge only the packets that were received and improves performance
net.ipv4.tcp_sack=1

# Disable the ability to change the MTU
# This makes it more difficult for an attacker to control the maximum transmission unit (MTU) of packets on the network
net.ipv4.ip_no_pmtu_disc=1

# Disable TCP timestamps (RFC1323/RFC7323)
net.ipv4.tcp_timestamps=0

# uncomment below to use, for this setup we let it enabled
# Enable the use of TCP timestamps
# This makes it more difficult for an attacker to perform a TCP spoofing attack
#net.ipv4.tcp_timestamps=1

# Enable source address verification for IPv6
# This makes it more difficult for an attacker to spoof their IP address
net.ipv6.conf.all.accept_ra=0
net.ipv6.conf.default.accept_ra=0

net.ipv4.route.flush=1
net.ipv6.route.flush=1

# allow testing with buffers up to 128MB (64MB)
## Maximum receive socket buffer size
net.core.rmem_max=134217728 #67108864
## Maximum send socket buffer size
net.core.wmem_max=134217728 #67108864

# Maximum number of packets queued on the input side
net.core.netdev_max_backlog=3000

# Enable the use of TCP fast open
# This allows the system to establish a TCP connection more quickly and improves performance
# - 0: Disable TCP Fast Open (default if not explicitly set).
# - 1: Enable TCP Fast Open for outgoing connections (clients).
# - 2: Enable TCP Fast Open for incoming connections (servers).
# - 3: Enable TCP Fast Open for both outgoing and incoming connections.
#net.ipv4.tcp_fastopen=1
net.ipv4.tcp_fastopen=3

# Enable the use of TCP window scaling
# increase Linux autotuning TCP buffer limit to 128MB (64MB)
# This allows the system to handle large TCP window sizes and improves performance
net.ipv4.tcp_window_scaling=3
## Minimum, initial and max TCP Receive buffer size in Bytes
net.ipv4.tcp_rmem=4096 87380 134217728 #67108864
## Minimum, initial and max buffer space allocated
net.ipv4.tcp_wmem=4096 87380 134217728 #67108864

net.ipv4.tcp_congestion_control=bbr #bbr|cubic

# recommended for hosts with jumbo frames enabled
net.ipv4.tcp_mtu_probing=1

# recommended to enable 'fair queueing'
net.core.default_qdisc=fq

# Auto tuning
net.ipv4.tcp_moderate_rcvbuf=1
# Don't cache ssthresh from previous connection
net.ipv4.tcp_no_metrics_save=1

net.ipv4.tcp_low_latency=1

vm.max_map_count=1048576

# Netfilter should be turned off on bridge devices
net.bridge.bridge-nf-call-iptables=0
net.bridge.bridge-nf-call-arptables=0
net.bridge.bridge-nf-call-ip6tables=0

# ###################################################################
fs.file-max=2097152 #262144
net.core.somaxconn=65535 #4096
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_mem=4194304 4194304 4194304
net.core.rps_sock_flow_entries=32768

# ###################################################################
# check below if to use instead - if you have performance penalties
#net.ipv4.tcp_timestamps=1
#net.ipv4.ip_no_pmtu_disc=0
```

run following command to perform the changes:

```sh
$sysctl --system
```

## \\\\ PCI Passthrough

Unlocking the Power of **GPU** and **USB-PCI** Card **Passthrough** in Proxmox

### // Setup BIOS Boot Options

> _These steps ensure that the necessary boot parameters are properly configured for PCI passthrough in Proxmox._

**SVM (Secure Virtual Machine)** `ENABLED`:

SVM is AMD's version of Intel VT-x. It provides support for virtualization, allowing multiple operating systems to run concurrently on a single AMD processor.
Enable SVM in the BIOS if you plan to use virtualization technologies such as AMD-V for running virtual machines on your system.

**SMT (Simultaneous Multithreading)** `AUTO`:

SMT is a technology that allows multiple threads to run on a single CPU core. It is AMD's equivalent to Intel's Hyper-Threading Technology.
Enabling SMT improves multitasking performance by allowing each CPU core to handle multiple threads simultaneously. This is beneficial for applications that can utilize multiple threads.

**Above 4G Decoding** `ENABLED`:

This option allows the system to decode memory above the 4-gigabyte boundary. It's essential for systems with more than 4 GB of RAM and also relevant for devices that require memory-mapped I/O above 4 GB.
Enable this option when your system has more than 4 GB of RAM or when you are using devices that require access to memory above the 4 GB limit, such as certain types of high-performance GPUs or RAID controllers.

**RE-SIZE BAR** `DISABLED`:

Resizable BAR is a feature that allows the CPU to access the entire GPU memory directly, enhancing data transfer speeds between the CPU and GPU.
Enable Resizable BAR for improved gaming performance and faster data transfers, especially with modern GPUs and compatible CPUs.

**IOMMU (Input-Output Memory Management Unit)** `ENABLED`:

IOMMU is a memory management unit that is used to manage memory transfers between devices and the system's memory. It is crucial for virtualization, allowing hardware devices to be assigned directly to virtual machines.
Enable IOMMU for virtualization setups, especially when using GPU passthrough, allowing virtual machines direct access to specific hardware components.

**ACS (Access Control Services)** `ENABLED`:

ACS allows finer control over PCIe devices, ensuring that devices in the same IOMMU group can be separated for better virtualization support.
Enabling ACS is often essential for more advanced virtualization setups, especially when GPU passthrough is used, to prevent conflicts between devices and ensure proper isolation for virtual machines.

**ARI support (Alternative Routing-ID Interpretation Support)** `AUTO`:

ARI is a PCIe feature that provides additional capability to address more devices on the PCIe bus.
Enabling ARI support allows systems to address more PCIe devices efficiently, which is particularly relevant in high-performance computing environments where numerous PCIe devices are used.

**ARI enumeration (Alternative Routing-ID Interpretation Enumeration (ARI Forwarding Support))** `AUTO`:

ARI Enumeration is the ability of the BIOS to detect and support PCIe devices that use ARI.
Enabling ARI Enumeration ensures that PCIe devices using ARI are properly recognized and utilized by the system, ensuring efficient addressing and communication with these devices.

### // Setup Grub Boot Parameters

> _These steps ensure that the necessary boot parameters are properly configured for in general PCI passthrough and specific for GPU passthrough in Proxmox._

determine whether the system is using Grub or systemd as the bootloader:

```sh
$efibootmgr -v | grep -q 'File(\\EFI\\SYSTEMD\\SYSTEMD-BOOTX64.EFI)' && echo "systemd" || echo "grub"
```

depends on the result, use one of the two next sub topics (_Grub Configuration or Systemd Configuration_)
to setup the correct configuration.

#### Grub Configuration

open the grub configuration file:

```sh
$nano /etc/default/grub
```

locate the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and modify it as follows:

```properties
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=force_enable iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init hugepagesz=1G hugepages=16 hugepagesz=2M default_hugepagesz=2M amdgpu.sg_display=0 amd_pstate=passive amd_pstate.shared_mem=1"
```

update the changes:

```sh
$update-grub
```

verify the changes written correct and exist after restart:

```sh
$cat /proc/cmdline
```

#### Systemd Configuration

edit the kernel command line by running the following command:

> _Note: Before running the command, verify the first information `root=ZFS=rpool/ROOT/pve-1 boot=zfs`
> and update it if necessary._

```sh
$echo 'root=ZFS=rpool/ROOT/pve-1 boot=zfs quiet amd_iommu=force_enable iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init hugepagesz=1G hugepages=16 hugepagesz=2M default_hugepagesz=2M amdgpu.sg_display=0 amd_pstate=passive amd_pstate.shared_mem=1' > /etc/kernel/cmdline
```

update the changes:

```sh
$proxmox-boot-tool refresh
```

verify the changes written correct and exist after restart:

```sh
$cat /proc/cmdline
```

#### Parameter Explanation

`amd_iommu=force_enable`:

Enables AMD's IOMMU (Input-Output Memory Management Unit) technology, also known as AMD-Vi (AMD Virtualization for I/O). It is required for **GPU** passthrough, as it provides hardware support for input/output virtualization and allows direct device assignment to virtual machines.

`iommu=pt`:

Sets the IOMMU mode to "passthrough." It ensures that the IOMMU is configured to allow direct assignment of devices, such as **GPUs**, to virtual machines without any interference from the host operating system.

`pcie_acs_override=downstream,multifunction`:

Is related to PCIe ACS (Access Control Services) override. It helps in addressing potential compatibility issues when passing through certain PCIe devices. By specifying "downstream" and "multifunction," you are indicating that ACS override should be enabled for downstream devices and multifunction devices.

- Downstream:
  This value means that ACS is overridden for downstream ports of PCIe switches. This helps ensure devices connected to PCIe switches are placed into separate IOMMU groups.
- Multifunction:
  This value ensures that ACS is overridden for multifunction devices. Multifunction devices are PCIe devices that have multiple functions on a single device, such as Ethernet controllers with multiple ports.

`nofb`:

Disables the framebuffer, which can help avoid conflicts or issues with graphics devices when performing **GPU** passthrough.

`nomodeset`:

**Prevents the kernel from loading video drivers** and setting display modes. It can be useful when passing through a **GPU** to a virtual machine, as it ensures that the GPU is **not actively used by the host operating system**.

`initcall_blacklist=sysfb_init`:

Adds the **sysfb_init** function to the initcall **blacklist**. It prevents the specified function from being called during the kernel initialization process. This can be useful if there are conflicts or issues related to framebuffer initialization.

`default_hugepagesz=1G`:

Hugepages are memory pages that are significantly larger than the standard 4 KB pages.
They can be 2 MB, 1 GB, or even larger.
Hugepages help reduce overhead in memory management and TLB (Translation Lookaside Buffer) misses,
especially for applications with large memory requirements, like databases or virtual machines.

Setting the default_hugepagesz is useful when specific applications or services benefit from large memory pages.
Some virtualization setups, especially those using GPU passthrough with **amdgpu**,
can gain performance benefits by allocating large, contiguous blocks of memory,
making this setting beneficial in such scenarios.

`hugepagesz=1G`:

It allows you to explicitly set the size of hugepages to be used.
Common values include 2M for 2 MB hugepages and 1G for 1 GB hugepages.

Explicitly setting the hugepagesz is essential when you want to ensure a specific size for hugepages.
This can be crucial for certain applications that require a particular hugepage size for optimal performance,
including some virtualization setups that involve GPU passthrough.

`hugepages=1`:

Specifies how many hugepages should be reserved.
The total memory allocated for hugepages equals `hugepages * hugepagesz`.

It's beneficial in cases where you want to reserve a specific amount of memory for applications or services that can take advantage of hugepages,
such as virtual machines using GPU passthrough.
Allocating a sufficient number of hugepages ensures that these applications have fast, contiguous memory access.

`amdgpu.sg_display=0`:

Disable S/G (scatter/gather) display (i.e., display from system memory).
This option is only relevant on APUs.
Set this option to 0 to disable S/G display if you experience flickering or other issues under memory pressure and report the issue.

`amd_pstate=passive`:

The AMD CPU frequency scaling driver (`amd_pstate`) dynamically adjusts the CPU frequency and voltage based on the system's load.
When set to "passive," it allows the CPU to scale down its frequency and voltage when the system is idle but doesn't actively scale up when there's increased demand.
This can save power and reduce heat production when the CPU is not under heavy load.

`amd_pstate.shared_mem=1`:

Shared memory support allows the AMD CPU frequency scaling driver to share information between CPU cores, facilitating better coordination and synchronization between cores. This can lead to more efficient frequency scaling decisions, especially in multi-core processors, improving overall system performance and responsiveness.

### // Setup VFIO Framework

> _Verify the content in `/etc/modules`,
> the first command will clear the file_
>
> > Ensure that the VFIO framework and its necessary components are loaded automatically during system startup.
> > This is essential for successful GPU passthrough and PCI device passthrough in Proxmox.
> >
> > - vfio:
> >   - The VFIO module is the core component of the VFIO framework, providing the infrastructure for PCI device passthrough.
> > - vfio-pci:
> >   - The VFIO PCI module provides support for PCI devices within the VFIO framework, enabling the passthrough
> >     of PCI devices to virtual machines.
> > - vfio_iommu_type1:
> >   - This module enables the VFIO IOMMU (Input-Output Memory Management Unit) driver, allowing the virtual
> >     machines to directly access hardware resources.
> > - vfio_virqfd:
> >   - This module is responsible for handling interrupts from the virtual machines using VFIO, allowing efficient
> >     interrupt processing and reducing latency.
> >   - NOTE: not available in newer kernel versions (below commented out)

```sh
$echo 'vfio' > /etc/modules
$echo 'vfio_pci' >> /etc/modules
$echo 'vfio_iommu_type1' >> /etc/modules
#$echo 'vfio_virqfd' >> /etc/modules
```

### // Setup pve-blacklist.conf

> _Verify the content in `/etc/modprobe.d/pve-blacklist.conf`,
> the first command will clear the file_
>
> > Blacklisting these drivers, such as `nouveau`, `amdgpu`, `radeon`, `nvidiafb`, `nvidia`, and `nvidia-gpu`,
> > prevents them from loading during system startup. This can help avoid conflicts and ensure a smoother
> > GPU passthrough experience.

```sh
$echo 'blacklist nouveau' > /etc/modprobe.d/pve-blacklist.conf
$echo 'blacklist amdgpu' >> /etc/modprobe.d/pve-blacklist.conf
$echo 'blacklist radeon' >> /etc/modprobe.d/pve-blacklist.conf
$echo 'blacklist nvidia*' >> /etc/modprobe.d/pve-blacklist.conf
# $echo 'blacklist nvidiafb' >> /etc/modprobe.d/pve-blacklist.conf
# $echo 'blacklist nvidia' >> /etc/modprobe.d/pve-blacklist.conf
# $echo 'blacklist nvidia-gpu' >> /etc/modprobe.d/pve-blacklist.conf
```

### // Setup iommu_unsafe_interrupts.conf

> Enabling unsafe interrupts through "iommu_unsafe_interrupts.conf" **improves device performance** but poses **security risks**.
> It allows the VFIO driver to process **device interrupts without safety checks**, benefiting certain devices.
> However, use caution and assess the risks before enabling it. **GPU passthrough** dedicates a physical GPU to a VM for
> **better graphics performance**. Enabling unsafe interrupts with "iommu_unsafe_interrupts.conf" enhances GPU passthrough
> by **reducing latency and improving responsiveness**, **bypassing some safety checks**.

```sh
$echo 'options vfio_iommu_type1 allow_unsafe_interrupts=1' \
> /etc/modprobe.d/iommu_unsafe_interrupts.conf
```

### // Setup kvm.conf

> Adding options to the KVM configuration in Proxmox ignores and avoids reporting specific Model
> Specific Registers (MSRs). This improves compatibility and prevents conflicts, especially for GPU passthrough.
> By ignoring problematic MSR requests from virtual machines, it enhances stability and performance.

```sh
$echo 'options kvm ignore_msrs=1 report_ignored_msrs=0' \
> /etc/modprobe.d/kvm.conf
```

### // Setup softdep.conf

> The command configures a soft dependency between the defined driver and the VFIO PCI driver in Proxmox.
> This ensures the correct driver initialization order for defined driver passthrough and other PCI device passthrough
> scenarios like the defined drivers below for amdgpu, usb-pci-devices, ...

```sh
$echo 'softdep amdgpu pre: vfio-pci' >> /etc/modprobe.d/softdep.conf
$echo 'softdep snd_hda_intel pre: vfio-pci' >> /etc/modprobe.d/softdep.conf
$echo 'softdep xhci_hcd pre: vfio-pci' >> /etc/modprobe.d/softdep.conf
```

### // Setup vfio.conf

> These steps allow the vfio-pci module to bind to the specified GPU and GPU-AUDIO devices,
> disabling VGA output. This configuration is useful for GPU passthrough and can improve
> performance for virtual machines utilizing the GPUs.

#### Identify GPU Information

execute the following command to **find** your **GPUs**:

> _note down the GPU address, such as "2b:00.0" (excluding the ".0" at the end)_

```sh
$lspci | grep -iE "VGA"
```

use the following command to **retrieve** the **GPU** and **GPU-AUDIO** info required for the configuration file.:

> _replace `<GPU>` and `<AUDIO>` in the next step with the respective values_

```sh
$lspci -n -s 2b:00 | awk '{ print $3 }'
```

#### create vfio.conf

> _replace `<GPU>` and `<AUDIO>` with the **GPU** and **GPU-AUDIO** values **obtained earlier**._

```sh
$echo 'options vfio-pci ids=<GPU>,<AUDIO> disable_vga=1' \
>> /etc/modprobe.d/vfio.conf
```

### // Update and Reboot

> Updates the initramfs (initial RAM file system) for all installed kernels on your system,
> to ensures that the changes made to the configuration files are reflected in the initramfs

```sh
$update-initramfs -u -k all
$reboot
```

---

## \\\\ Additional Configurations

### // Create a Cloud-Init Template

> _Example for setup ubuntu-23.04 cloud-init template_
>
> > - NOTE: change **img** `noble-server-cloudimg-amd64.img` as you needed (also the url for download)
> > - NOTE: change **vm-id** `9999` as you needed
> > - NOTE: check **storage-name** `local-zfs` for your needs (maybe your's is local-lvm)

download the ubuntu-23.04 image:

```sh
$wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
-O /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
```

verify the sha256 hash:

```sh
$cd /var/lib/vz/template/iso && \
curl -s https://cloud-images.ubuntu.com/noble/current/SHA256SUMS | \
grep "noble-server-cloudimg-amd64.img" | \
sha256sum -c - ; cd --
```

create a new VM:

```sh
$qm create 9999 --name "template-s-ubuntu-noble" --memory 2048 --net0 virtio,bridge=vmbr0 \
--cpu cputype=x86-64-v2-AES --sockets 1 --cores 2 --numa 0
```

add cloud img as drive or empty drive:

```sh
# import the downloaded disk to local-zfs storage
$qm importdisk 9999 /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img local-zfs
# finally attach the new disk to the VM as scsi drive
$qm set 9999 --scsihw virtio-scsi-single --scsi0 local-zfs:vm-9999-disk-0,ssd=1,discard=on,iothread=1
```

setup additional VM properties:

```sh
# add cloud-init cd-rom drive
$qm set 9999 --balloon 0
$qm set 9999 --scsi2 local-zfs:cloudinit
$qm set 9999 --boot order='scsi0'
#$qm set 9999 --serial0 socket --vga serial0
$qm set 9999 --agent 1
$qm set 9999 --hotplug disk,network,usb
$qm set 9999 --bios ovmf
$qm set 9999 --efidisk0 local-zfs:0,efitype=4m,pre-enrolled-keys=0
$qm set 9999 --machine q35
$qm set 9999 --tablet 0
$qm set 9999 --ostype l26
```

cloud init custom config:

```sh
# add cloud init config to install guest agent on first start
$mkdir -p /var/lib/vz/snippets
$cat <<EOF >/var/lib/vz/snippets/ubuntu.yaml
#cloud-config
keyboard:
  layout: "de"
  variant: ""

runcmd:
    - apt update
    - apt install -y qemu-guest-agent
    - systemctl start qemu-guest-agent
EOF
$qm set 9999 --cicustom "vendor=local:snippets/ubuntu.yaml"
```

convert VM as template:

```sh
$qm template 9999
```

### // Create a Cloud-Init autoinstall Template

> _Example for setup ubuntu-24.04-live-server cloud-init template over autoinstall,
> which installs in this example for as client usage 'ubuntu-desktop-minimal'_
>
> > - NOTE: change **iso** `ubuntu-24.04-live-server-amd64.iso` as you needed (also the url for download)
> > - NOTE: change **vm-id** `9998` as you needed
> > - NOTE: check **storage-name** `local-zfs` for your needs (maybe your's is local-lvm)
> > - NOTE: check **SHA512_PASSPHRASE** and **LUKS_PASSPHRASE** to be updated with your credentials

download the ubuntu-server-24.04 image:

```sh
$wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso \
-O /var/lib/vz/template/iso/ubuntu-24.04-live-server-amd64.iso
```

verify the sha256 hash:

```sh
$cd /var/lib/vz/template/iso && \
curl -s https://releases.ubuntu.com/24.04/SHA256SUMS | \
grep "ubuntu-24.04-live-server-amd64.iso" | \
sha256sum -c - ; cd --
```

create a new VM:

```sh
$qm create 9998 --name "template-c-ubuntu-noble" --memory 2048 --net0 virtio,bridge=vmbr0 \
--cpu cputype=x86-64-v2-AES --sockets 1 --cores 2 --numa 0
```

add empty drive and iso:

```sh
$qm set 9998 --scsihw virtio-scsi-single --scsi0 local-zfs:32,ssd=1,discard=on,iothread=1
$qm set 9998 --scsi1 local-zfs:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom
```

setup additional VM properties:

```sh
# add cloud-init cd-rom drive
$qm set 9998 --balloon 0
$qm set 9998 --vga virtio
$qm set 9998 --agent 1
$qm set 9998 --hotplug disk,network,usb
$qm set 9998 --bios ovmf
$qm set 9998 --efidisk0 local-zfs:0,efitype=4m,pre-enrolled-keys=0
$qm set 9998 --machine q35
$qm set 9998 --tablet 0
$qm set 9998 --ostype l26
```

cloud init custom autoinstall config:

```sh
# add cloud init config to install guest agent and ubuntu client desktop on first start
$mkdir -p /var/lib/vz/snippets
$touch /var/lib/vz/snippets/meta-data
$cat <<EOF >/var/lib/vz/snippets/user-data
#cloud-config
autoinstall:
  version: 1
  locale: en_US
  keyboard:
    layout: de
  identity:
    hostname: 9GB9qiabF1kDD8W1RFQ9
    username: iwashere
    password: 'SHA512_PASSPHRASE' # pragma: allowlist secret
  storage:
    layout:
      name: lvm
      sizing-policy: all
      password: LUKS_PASSPHRASE
  ssh:
    install-server: yes
    authorized-keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPT2hL6eFCOia2N6NqaxcmCl9l6vw4pcj7H3qFMGAtST infra-default
    allow-pw: no
  source:
    search_drivers: true
  packages:
    - qemu-guest-agent
    - ubuntu-desktop-minimal
EOF

$cd /var/lib/vz/snippets && \
mkisofs -V cidata -lJR -o /var/lib/vz/template/iso/cloud-init.iso user-data meta-data && \
cd --

$qm set 9998 --scsi2 local-zfs:iso/cloud-init.iso,media=cdrom
$qm set 9998 --boot order='scsi0;scsi1'
```

convert VM as template:

```sh
$qm template 9998
```

---

## \\\\ Helpful Functions

### // qm re-scan

if a vm is:

- not load correct
- shows not correct values
- not assign volumes correct
- ...

you can re-scan and fix vm's by run:

```sh
$qm rescan --vmid <VM-ID>
```

### // qm unlock

if vm is locked for example after an power outage

```sh
$qm unlock <VM-ID>
```

### // assign complete drivers to a VM

search for **disk-id**:

```sh
$lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,UUID --exclude 7
```

add **disk** to **vm**:

```sh
$qm set <VM-ID> -scsi<NUMBER> /dev/disk/by-id/<DISK-ID/UUID>
```

---

## \\\\ Resources & More information's

- <https://www.proxmox.com/de/downloads>{:target="\_blank"}
- <https://pve.proxmox.com/pve-docs/pve-admin-guide.pdf>{:target="\_blank"}
- <https://pve.proxmox.com/wiki/Performance_Tweaks>{:target="\_blank"}
- <https://pve.proxmox.com/wiki/ZFS:_Tips_and_Tricks#Install_on_a_high_performance_system>{:target="\_blank"}
- <https://www.servethehome.com/how-to-pass-through-pcie-nics-with-proxmox-ve-on-intel-and-amd>{:target="\_blank"}
- <https://pve.proxmox.com/wiki/PCI_Passthrough>{:target="\_blank"}
- <https://www.dlford.io/memory-tuning-proxmox-zfs>{:target="\_blank"}
- <https://www.reddit.com/r/VFIO/comments/11mqtna/successful_passthrough_of_an_rx_7900_xt/>{:target="\_blank"}
- <https://forum.level1techs.com/t/vfio-passthrough-in-2023-call-to-arms/199671/101?page=4>{:target="\_blank"}
- <https://docs.kernel.org/gpu/amdgpu/module-parameters.html>{:target="\_blank"}
- <https://bbs.archlinux.org/viewtopic.php?pid=2070655#p2070655>{:target="\_blank"}
- <https://forum.manjaro.org/t/cannot-isolate-gpu-for-vfio/139292/5>{:target="\_blank"}
- <https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF>{:target="\_blank"}
- <https://www.wundertech.net/how-to-set-up-gpu-passthrough-on-proxmox/>{:target="\_blank"}
- <https://3os.org/infrastructure/proxmox/gpu-passthrough/gpu-passthrough-to-vm/#proxmox-configuration-for-gpu-passthrough>{:target="\_blank"}
- <https://pve.proxmox.com/wiki/PCI(e)_Passthrough>{:target="\_blank"}
- <https://docs.renderex.ae/posts/Enabling-hugepages/>{:target="\_blank"}
- <https://forum.proxmox.com/threads/hey-proxmox-community-lets-talk-about-resources-isolation.124256/>{:target="\_blank"}
- <https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html>{:target="\_blank"}
- <https://canonical-subiquity.readthedocs-hosted.com/en/latest/howto/autoinstall-quickstart.html>{:target="\_blank"}
- kernel-parameters
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amd_iommu>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=iommu>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=pcie_acs_override>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=nofb>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=nomodeset>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=initcall_blacklist>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=default_hugepagesz>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=hugepagesz>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=hugepages>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amdgpu.sg_display>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amd_pstate>{:target="\_blank"}
  - <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amd_pstate.shared_mem>{:target="\_blank"}
