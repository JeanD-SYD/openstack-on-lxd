# Overview

This repository provides resources and processes to support deployment of OpenStack in LXD containers using Juju, a powerful application modeling tool.

Such a process is useful in developer scenarios where OpenStack can be deployed to a single laptop or server, provided of course that enough resources are available.

These bundles, configurations and processes can be customized to fit numerous development or production scenarios.

For full details of use please refer the the [OpenStack on LXD](http://docs.openstack.org/developer/charm-guide/openstack-on-lxd.html) section of the [OpenStack Charm Guide](http://docs.openstack.org/developer/charm-guide).




## network setup

```
$ ip link | grep enp
2: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
3: enp7s0f0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
4: enp7s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
```

* enp6s0   -> cabled, but not used atm
* enp7s0f0 -> not cabled
* enp7s0f1 -> cabled, to be used as a bridge port

```
$ sudo apt install bridge-utils
```

[/etc/network/interfaces]
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto br0
iface br0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1
    bridge_ports enp7s0f1

auto enp7s0f1
iface enp7s0f1 inet manual
    mtu 1460
```

```
$ sudo reboot
```

## disk setup

```
$ lsblk 
NAME                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                     8:0    0 232.9G  0 disk 
├─sda1                  8:1    0   487M  0 part /boot
├─sda2                  8:2    0     1K  0 part 
└─sda5                  8:5    0 232.4G  0 part 
  ├─ubuntu--vg-root   252:0    0 200.5G  0 lvm  /
  └─ubuntu--vg-swap_1 252:1    0    32G  0 lvm  [SWAP]
sdb                     8:16   0 465.8G  0 disk 
└─sdb1                  8:17   0 465.8G  0 part /var/lib/lxd/containers
```

## package installation

https://docs.openstack.org/charm-guide/latest/openstack-on-lxd.html

Upgrade.

```
$ sudo apt update
$ sudo apt upgrade
```

Make sure the standard set of packages is installed.

```
$ sudo apt install minimal^ standard^ server^
```

Limit OpenSSH access.

```
$ sudo ufw default allow incoming
$ sudo ufw default allow routed

$ sudo ufw limit OpenSSH
```

Workaround for race condition of proc-sys-fs-binfmt_misc.mount.

```
$ sudo mkdir -p /etc/systemd/system/proc-sys-fs-binfmt_misc.mount.d/

$ cat <<EOF | sudo tee /etc/systemd/system/proc-sys-fs-binfmt_misc.mount.d/override.conf
[Unit]
StartLimitInterval=60
StartLimitBurst=120
EOF

$ sudo systemctl daemon-reload
```

Kernel upgrade for AMD Ryzen 7.

```
$ sudo apt-get install linux-generic-hwe-16.04
$ sudo reboot
```

Repositories.

```
$ sudo add-apt-repository cloud-archive:pike -y -u
$ sudo add-apt-repository ppa:juju/stable -y -u
```

Install LXD from xenial-proposed to leverage embedded bridge functionality.

```
$ sudo apt install -t xenial-backports lxd
```

Install the rest of packages.

```
$ sudo apt-get install juju squid-deb-proxy \
    python-novaclient python-keystoneclient python-glanceclient \
    python-neutronclient python-openstackclient curl
```

## LXD setup

Storage.

```
$ sudo ln -s /var/lib/lxd/containers /var/lib/lxd/storage-pools/default
$ lxc storage create default dir source=/var/lib/lxd/storage-pools/default
```

Network.

```
$ lxc network create lxdbr0 \
    ipv4.address=10.0.8.1/24 \
    ipv4.dhcp.ranges=10.0.8.2-10.0.8.200 \
    ipv4.nat=true \
    ipv6.address=none
```

Default profile.

```
$ lxc profile device add default root disk path=/ pool=default
$ lxc profile device add default eth0 nic nictype=bridged parent=br0 name=eth0
```

## Juju setup

juju-default profile.

```
$ lxc profile create juju-default
$ cat lxd-2.2.x-profile.yaml | lxc profile edit juju-default
```

## Juju bootstrap

```
$ juju bootstrap --model-default config.yaml localhost lxd
```

## Deploy

```
$ juju deploy ./bundle-pike.yaml
```

## Re-deploy

```
$ juju destroy-model default
$ juju add-model default

$ juju deploy ./bundle-pike.yaml
```
