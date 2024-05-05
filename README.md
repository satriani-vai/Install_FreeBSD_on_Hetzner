# Install FreeBSD 14.0 on Hetzner server

Hetzner no longer offers direct install of FreeBSD, but we can do it ourselves. Here is how :)

Boot the hetzner server in Hetnzer Debain based rescue mode. ssh into it. then:

/dev/nvme0n1
/dev/nvme1n1

or

/dev/sda
/dev/sdb

```zsh
wget https://mfsbsd.vx.sk/files/iso/14/amd64/mfsbsd-14.0-RELEASE-amd64.iso

qemu-system-x86_64 \
    -cdrom mfsbsd-14.0-RELEASE-amd64.iso   \
    -drive format=raw,file=/dev/sda        \
    -drive format=raw,file=/dev/sdb        \
    -nic user,hostfwd=tcp::2222-:22        \
    -display curses                        \
    -boot d                                \
    -m 8G                                  \
    -k de
```

> basically have a mini VPS with mfsbsd running with real disk passthrough and console access, just like a KVM, so I can install as usual - and then I can even test my installation directly by booting from it in the same way! Then when it works I just boot the server normal (ie directly into FreeBSD) and if I ever b0rk something up I boot the Linux rescue image and run mfsbsd again!

Source: https://www.reddit.com/r/freebsd/comments/wf7h34/hetzner_has_silently_dropped_support_for_freebsd/ijcxgvb/

## Start install

Log in from the console

- login: `root`
- password: `mfsroot`

Start the FreeBSD installer

```sh
bsdinstall
```

Proceed with installation. When done, "power off" the qemu VM

```sh
poweroff
```

## Check that it works

Now boot the physical disks in qemu without having the CD ISO attached.

```zsh
qemu-system-x86_64 \
    -drive format=raw,file=/dev/sda        \
    -drive format=raw,file=/dev/sdb        \
    -nic user,hostfwd=tcp::2222-:22        \
    -display curses                        \
    -boot d                                \
    -m 8G                                  \
    -k de
```

## Before you reboot the host machine

Qemu provides an emulated NIC to the VM. So if the physical network in the host
uses a NIC that needs a different driver, the NIC name will be different in the VM
from what it will be when running FreeBSD on the hardware.

The Qemu NIC will appear as `re0`.

However in my case the physical NIC in the machine uses a different driver and
appears as `igb0` when running FreeBSD on the hardware.

The Hetzner Debian based rescue system will give you a minimal description of the NIC
in the machine when you ssh into it. Make note of that. If it's intel, you can
put and entry for both `igb0` in addtion to `re0` in your `/etc/rc.conf`
and then when you boot and ssh into the machine you will see which one was used
and then you can update your `/etc/rc.conf` accordingly.

If the NIC is not Intel, you have to find out what Linux commands to use
in the Hetzner Debian based rescue system to show more details about your NIC,
and then you need to figure out which FreeBSD NIC driver is correct for that one
and edit your `/etc/rc.conf` accordingly.

For reference, here is what the complete `/etc/rc.conf` from one of my Hetzner
servers looks like currently:

```text
hostname="blacksmith"
clear_tmp_enable="YES"

syslogd_flags="-ss"
sendmail_enable="NONE"

# Used when booting in Qemu
ifconfig_re0="DHCP"
ifconfig_re0_ipv6="inet6 accept_rtadv"

# Used when booting on hardware
ifconfig_igb0_name="extif"
ifconfig_extif="DHCP"
ifconfig_extif_ipv6="inet6 2a01:4f9:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx prefixlen 80"
ipv6_defaultrouter="fe80::1%extif"

# Set dumpdev to "AUTO" to enable crash dumps, "NO" to disable
dumpdev="NO"

zfs_enable="YES"

ntpdate_enable="YES"
ntpd_enable="YES"
powerd_enable="YES"

sshd_enable="YES"

wireguard_enable="YES"
wireguard_interfaces="wg0"

jail_enable="YES"
```

## Moment of truth

Reboot the host machine. All goes well, you'll be able to ssh into it and find
a running FreeBSD system :D
