---
title: 'Configuring network bridging on Scale and HG servers'
slug: network-bridging-scale
excerpt: 'Find out how to configure your virtual machines for access to the public internet'
section: 'Network management'
---

**Last updated 14th July 2021**

## Objective

Bridged networking can be used to configure virtual machines on OVHcloud dedicated servers. The appropriate routing needs to be added manually in order for it to work.

**This guide explains how to use network bridging to configure internet access for your virtual machines.**

> [!warning]
>OVHcloud is providing you with services for which you are responsible, with regard to their configuration and management. You are therefore responsible for ensuring they function correctly.
>
>This guide is designed to assist you in common tasks as much as possible. Nevertheless, we recommend that you contact a specialist service provider if you have difficulties or doubts concerning the administration, usage or implementation of services on a server.
>

## Requirements

- a [dedicated server](https://www.ovhcloud.com/en-gb/bare-metal/) in your OVHcloud account
- a [failover IP address](https://www.ovhcloud.com/en-gb/bare-metal/ip/) or a failover IP block (RIPE)
- administrative access (root) via SSH or GUI to your server
- basic networking and administration knowledge

## Instructions

The following sections contain the configurations for the most commonly used distributions / operating systems.

> [!primary]
>
Concerning different distribution releases, please note that the proper procedure to configure your network interface as well as the file names may have been subject to change. We recommend to consult the manuals and knowledge resources of the respective OS versions if you experience any issues.
> 

Code samples in the following instructions have to be replaced with your own values:

- SERVER_IP = The main IP address of your server
- FAILOVER_IP = The address of your failover IP
- GATEWAY_IP = The address of your default gateway

### Determining the gateway address

To configure your virtual machines for internet access, you will need to know the gateway of your host machine (i.e. your dedicated server). The gateway IP address is made up of the first three octets of your server's main IP address, with 254 as the last octet. For example, if your server's main IP address was:

- 169.254.10.20

Your gateway address would be:

- 169.254.10.**254**


#### Debian

##### **Step 1: Edit the configuration file**

Open the network configuration file for editing:

```bash
nano /etc/network/interfaces.d/50-cloud-init
```

Once the file is open, amend it according to the following code:

```bash
auto lo
iface lo inet loopback
 
auto ens33f0
iface ens33f0 inet dhcp
 
iface ens33f0 inet static
  address FAILOVER_IP
  netmask 255.255.255.255
  dns-nameservers 213.186.33.99
  gateway GATEWAY_IP
```

##### **Step 2: Configure DNS**

Open `/etc/resolv.conf` and add the following line:

```bash
nameserver 213.186.33.99
```

Save and exit, then add write protection to the file with this command:

```bash
chattr +i /etc/resolv.conf
```

##### **Step 3: Restart the interface**

Apply the changes with the following command:

```bash
sudo systemctl restart networking
```


#### Ubuntu

##### **Step 1: Edit the configuration file**

Open the network configuration file located in `/etc/netplan/` for editing. Here the file is called `50-cloud-init.yaml`.

```bash
nano /etc/netplan/50-cloud-init.yaml
```

Once the file is open, amend it with the following code:

```yaml
network:
    version: 2
    ethernets:
        ens33f0:
            accept-ra: false
            addresses:
              - FAILOVER_IP
            dhcp4: true
            gateway4: GATEWAY_IP
            match:
              macaddress: 0c:xx:xx:xx:xx:63
            nameservers:
              addresses:
                - 213.186.33.99
            set-name: ens33f0
```

Save and close the file.

##### **Step 2: Apply the new network configuration**

You can test your configuration using this command:

```bash
sudo netplan try
```

If it is correct, apply it using the following command:

```bash
sudo netplan apply
```

#### Proxmox


```bash
source /etc/network/interfaces.d/*
```

`/etc/network/interfaces.d/50-cloud-init`:


```bash
auto lo
iface lo inet loopback
  dns-nameservers 213.186.33.99
 
auto ens33f0
iface ens33f0 inet manual
  mtu 1500
 
iface ens33f0 inet6 static
  address 2001:41d0:203:8c05::/64
  dns-nameservers 2001:41d0:3:163::1
  gateway fe80::1
 
auto vmbr0
iface vmbr0 inet dhcp
  bridge-ports ens33f0
  bridge-stp off
  bridge-fd 0
 
iface vmbr0 inet6 static
  address 2001:41d0:203:8c05::/64
  dns-nameservers 2001:41d0:3:163::1
  gateway fe80::1
  bridge-ports ens33f0
  bridge-stp off
  bridge-fd 0
```


#### Red Hat

##### **Step 1: Edit the configuration file**

`/etc/sysconfig/network-scripts/ifcfg-eth0`:

```bash
BOOTPROTO=dhcp
DEVICE=eth0
HWADDR=0c:42:a1:23:33:8e
IPV6ADDR=2001:41d0:203:8c04::/56
IPV6_AUTOCONF=yes
IPV6_DEFAULTGW=fe80::1
IPV6_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_PEERDNS=no #prevent /etc/resolv.conf override by dhcp v6
IPV6_PRIVACY=no #this is the settings for EUI-64
ONBOOT=yes
PEERDNS=no #prevent /etc/resolv.conf override by dhcp
TYPE=Ethernet
USERCTL=no
```

##### **Step 2: Configure DNS**

Open `/etc/resolv.conf` and add the following line:

```bash
nameserver 213.186.33.99
```

##### **Step 3: Restart the interface**

Apply the changes with the following command:

```bash
sudo systemctl restart networking
```




### Troubleshooting

First, restart your server from the command line or its GUI. If you are still unable to establish a connection from the public network to your VM and suspect a network problem, you need to reboot the server in [rescue mode](../ovh-rescue/). Then you can set up the bridging network interface directly on the host.

Once you are connected to your server via SSH, enter the following command:

```bash
ip address add FAILOVER_IP dev eth0
ip route add default via GATEWAY_IP dev eth0
```

To test the connection, ping your failover IP from the outside. If it responds in rescue mode, that probably means that there is a configuration error on the VM or the host. If, however, the IP is still not working, please create a ticket in your [OVHcloud Control Panel](https://www.ovh.com/auth/?action=gotomanager&from=https://www.ovh.co.uk/&ovhSubsidiary=GB) to relay your test results to our support teams.
 
## Go further

[Activating and using rescue mode](../ovh-rescue/)

Join our community of users on <https://community.ovh.com/en/>.
