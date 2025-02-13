# Dual FreeBSD [Bastille] VNET Jails on a UTM (macOS) VM

This guide explains how to set up a FreeBSD VM on macOS using UTM with dual VNET interfaces for jail networking. The setup includes three network interfaces:

1. **Shared Network** - Allows communication between the host and other VMs.
2. **Bridged Network (Static IP)** - Connects the VM to the host's LAN, enabling access via the local network router.
3. **Bridged Network for FreeBSD Jails** - Dedicated to jail networking with DHCP or a static IP.

## VM Settings

### UTM Network Setup


| Interface  | MAC Address         | IP Address          | Mode                     |
| ------------ | --------------------- | --------------------- | -------------------------- |
| **vtnet0** | `13:02:fa:27:97:a0` | `192.168.64.64/24`  | Shared Network           |
| **vtnet1** | `cb:90:1b:98:51:3c` | `192.168.10.254/24` | Bridged (Static IP)      |
| **vtnet2** | `25:25:e9:44:82:9f` | DHCP                | Bridged (DHCP for Jails) |

``` sh 
-device virtio-net-pci,mac=13:02:fa:27:97:a0,netdev=net0 
-netdev vmnet-shared,id=net0,start-address=192.168.64.1,end-address=192.168.64.254,subnet-mask=255.255.255.0 

-device virtio-net-pci,mac=cb:90:1b:98:51:3c,netdev=net1 
-netdev vmnet-bridged,id=net1,ifname=en0 

-device virtio-net-pci,mac=25:25:e9:44:82:9f,netdev=net2 
-netdev vmnet-bridged,id=net2,ifname=en0 
```

#### VM Network

##### `/etc/rc.conf`

```sh
hostname="hostname"

## Network Interfaces
# Bring up physical interfaces without IP aliases.

## VTNET0 (Shared Network)
ifconfig_vtnet0="192.168.64.64/24"

## VTNET0 (Bridged Network)
ifconfig_vtnet1="DHCP"

## VTNET2 (third virtual network on host for the Jails)
ifconfig_vtnet2="up -alias"

# Set the default router (this applies to the bridged network)
defaultrouter="192.168.10.1"

sshd_enable="YES"
moused_nondefault_enable="NO"
dumpdev="AUTO"
zfs_enable="YES"
sshd_enable="YES"

## TIME
timezone="America/New_York"
ntpd_enable="YES"
ntpd_sync_on_start="YES"

## QEMU GUEST TOOLS
qemu_guest_agent_enable="YES"

## PF FIREWALL
pf_enable="YES"

## BASTILLE
bastille_enable="YES"
cloned_interfaces="lo1 bridge0 bridge2"
ifconfig_lo1_name="bastille0"

# Configure bridge0:
ifconfig_bridge0="up"
ifconfig_bridge0_alias0="192.168.64.64/24"

# Remove this line since rc.local handles adding vtnet0:
# ifconfig_bridge0_addm="vtnet0"

# Create a bridge for the third NIC.
ifconfig_bridge2="up"
ifconfig_bridge2="DHCP"
# Remove this line since rc.local handles adding vtnet2:
#ifconfig_bridge2_addm="vtnet2"
```

##### `/etc/rc.local` (VM Host)

This script ensures the VM network interfaces are assigned to the appropriate bridges.

```sh
#!/bin/sh
ifconfig bridge0 addm vtnet0
ifconfig bridge2 addm vtnet2
dhclient bridge2
exit 0
```

## PF Configuration

Packet Filter configuration for Bastille jails

##### `/etc/pf.conf`

```sh
##### `/etc/pf.conf`
```sh
#
# /etc/pf.conf - Updated PF configuration for Bastille jails
#

# Define network interfaces:
#   ext_if  - External interface (typically used for NAT)
#   lan_if  - Internal LAN interface (e.g., for VM communication)
#   jail_if - Interface dedicated to Bastille jails

ext_if="vtnet0"
lan_if="vtnet1"
jail_if="vtnet2"

# Set default block policy to return an ICMP error for rejected packets
set block-policy return

# Normalize incoming packets and reassemble fragments on the external interface
scrub in on $ext_if all fragment reassemble

# Exclude loopback traffic from packet filtering
set skip on lo

# Define a persistent table for jail IP addresses
# Bastille dynamically manages this table, adding/removing jail IPs as needed
table <jails> persist

# Load port redirection (rdr) rules from the "rdr/*" anchor if configured
rdr-anchor "rdr/*"

#
# Default Filtering Policy
#

# Block all incoming traffic by default
block in all

# Allow all outbound traffic with stateful tracking
pass out quick keep state

# Prevent IP address spoofing on each interface
antispoof for $ext_if inet
antispoof for $lan_if inet
antispoof for $jail_if inet

#
# Inbound Rules for Bastille Jails
#

# Allow inbound TCP/UDP traffic to jails from any source
pass in on $jail_if inet proto tcp from any to <jails> keep state
pass in on $jail_if inet proto udp from any to <jails> keep state

# Allow communication between jails and VMs over the LAN interface
pass in on $lan_if inet proto tcp from any to <jails> keep state
pass in on $lan_if inet proto udp from any to <jails> keep state

#
# Host-Specific Rules
#

# Allow SSH access to the host on all interfaces
pass in inet proto tcp from any to any port ssh flags S/SA keep state

# Allow ICMP echo requests (ping) to the host on all interfaces
pass in inet proto icmp from any to any icmp-type echoreq keep state
```

## Bastille Jail Configuration

### Creating a Jail with Dual VNET Interfaces

##### Create the Jail

```sh
bastille create -B myjail 14.2-RELEASE 192.168.64.100 bridge0
```

After creation, manually add the second **VNET** interface (`bridge2`) to the jail.

##### `/usr/local/bastille/jails/myjail/jail.conf`

```sh
myjail {
  enforce_statfs = 2;
  devfs_ruleset = 13;
  exec.clean;
  exec.consolelog = /var/log/bastille/myjail_console.log;
  exec.start = '/bin/sh /etc/rc';
  exec.stop = '/bin/sh /etc/rc.shutdown';
  host.hostname = myjail;
  mount.devfs;
  mount.fstab = /usr/local/bastille/jails/myjail/fstab;
  path = /usr/local/bastille/jails/myjail/root;
  securelevel = 2;
  osrelease = 14.2-RELEASE;

vnet;
  vnet.interface = e0b_myjail;
  vnet.interface += e1b_myjail;  # Adding the second VNET interface

  # First VNET interface (bridge0)
  exec.prestart += "ifconfig epair0 create";
  exec.prestart += "ifconfig bridge0 addm epair0a";
  exec.prestart += "ifconfig epair0a up name e0a_myjail";
  exec.prestart += "ifconfig epair0b up name e0b_myjail";
  exec.prestart += "ifconfig e0a_myjail description \"vnet host interface for Bastille jail myjail\"";

  # Second VNET interface (bridge2)
  exec.prestart += "ifconfig epair1 create";
  exec.prestart += "ifconfig bridge2 addm epair1a";
  exec.prestart += "ifconfig epair1a up name e1a_myjail";
  exec.prestart += "ifconfig epair1b up name e1b_myjail";
  exec.prestart += "ifconfig e1a_myjail description \"vnet host interface for Bastille jail myjail\"";

  exec.poststop += "ifconfig bridge0 deletem e0a_myjail";
  exec.poststop += "ifconfig e0a_myjail destroy";
  exec.poststop += "ifconfig bridge2 deletem e1a_myjail";
  exec.poststop += "ifconfig e1a_myjail destroy";
}
```

##### Jail Network Configuration `/etc/rc.conf`

The jail has a static IP for UTM's shared network and a dynamically assigned IP for LAN communication.

```sh
syslogd_flags="-ss"
sendmail_enable="NO"
sendmail_submit_enable="NO"
sendmail_outbound_enable="NO"
sendmail_msp_queue_enable="NO"
cron_flags="-J 60"

# First Interface (Bridge0)
ifconfig_e0b_myjail_name="vnet0"
ifconfig_vnet0="inet 192.168.64.101/24"
defaultrouter="192.168.64.1"

# Second Interface (Bridge2)
ifconfig_e1b_myjail_name="vnet1"
ifconfig_vnet1="DHCP"
```

##### `/etc/rc.local` (Jail)

This script requests a lease from the router in the aditional VNET manually added to the jail.

```sh
#!/bin/sh
dhclient vnet1
exit 0
```

## How the Host (FreeBSD VM) Enables Jails to Get IPs from the LAN Router

### The FreeBSD host (VM) does not assign IPs to the jails. Instead, it:

1. Creates bridge2 and adds vtnet2 to it.
2. Does not assign an IP to bridge2 itself.
3. Allows jails to attach to bridge2 so they can obtain IPs from the LAN router’s DHCP server.

### How Jails Get IPs from the LAN Router via bridge2

1. The FreeBSD VM has a bridged network interface (vtnet2) that is connected to the LAN. This interface behaves as if it is plugged into the LAN like a physical NIC.
2. The VM creates bridge2 and attaches vtnet2 to it.
3. When a jail is started, its virtual network interface (e1b_myjail) is attached to bridge2.
4. Since bridge2 is connected to the LAN via vtnet2, the jail sends a DHCP request over bridge2, and the LAN router assigns it an IP.

### What Happens on Each Component?

#### On the FreeBSD VM (Host)

* Creates bridge2 to allow jails to use vtnet2.
* Ensures jails can use DHCP via bridge2.

#### On the Jail

* Requests an IP via DHCP through vnet1, which is attached to bridge2.
* Gets an IP directly from the LAN router (not from the FreeBSD host).
