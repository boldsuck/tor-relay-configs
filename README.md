# 10 Gigabit Ethernet tweaks

I use network cards with Intel controller 82599, they are old, but have stable drivers (ixgbe) for most operating systems. \
Fujitsu D2755-A11 and Intel X520-DA2 Dual Port FC 10GbE SFP+

Performance Tuning Tips in _/etc/sysctl.d/local.conf_ from:

- Intel ixgbe driver README
- https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
- https://fasterdata.es.net/host-tuning/linux/
- https://github.com/torservers/server-config-templates/blob/master/sysctl.conf

My main problem: \
A high network load or DDoS kills the driver, syslog was flooded with:
```sh
ixgbe 0000:04:00.1 enp4s0f1: Detected Tx Unit Hang
ixgbe 0000:04:00.1 enp4s0f1: initiating reset due to tx timeout
```
These 3 failures can be seen on nusenu's OrNetStats. October 2021 https://nusenu.github.io/OrNetStats/for-privacy.net.html \
Sollution: \
Disable virtualization options in BIOS / (U)EFI (IOMMU, Intel VT-d, AMD-Vi, SR-IOV) or \
use kernel boot option (intel_iommu=off, amd_iommu=off)

* Install ethtool and disable GRO, TSO, GSO offloading in _/etc/network/interfaces_
```sh
iface enp4s0f0 inet manual
	pre-up	/sbin/ethtool -K $IFACE tso off
	pre-up	/sbin/ethtool -K $IFACE gso off
	pre-up	/sbin/ethtool -K $IFACE gro off

iface enp4s0f1 inet manual
	pre-up	/sbin/ethtool -K $IFACE tso off
	pre-up	/sbin/ethtool -K $IFACE gso off
	pre-up	/sbin/ethtool -K $IFACE gro off
```

* Disable offloading on LACP bonded interfaces: \
Not all of the tx offloading features are copied from slaves to the upper device like bonds and vlans. \
More info: https://access.redhat.com/solutions/750503 \
Debian 11 comes with a newer version (2.12) of the ifenslave package. \
You no longer need the physical interface stanzas, you define them in bond-slaves.

```sh
auto bond0
iface bond0 inet manual
	bond-slaves		enp4s0f0 enp4s0f1
	bond-mode		802.3ad
	bond-miimon		100
	bond-xmit-hash-policy	layer3+4

auto bond0.123
iface bond0.123 inet static
	address 192.0.2.12/27
	up	ip route add default via 192.0.2.1 src 213.0.113.32
	pre-up	/sbin/ethtool -K enp4s0f0 tso off && /sbin/ethtool -K enp4s0f1 tso off && /sbin/ethtool -K bond0 tso off && /sbin/ethtool -K bond0.123 tso off
	pre-up	/sbin/ethtool -K enp4s0f0 gso off && /sbin/ethtool -K enp4s0f1 gso off && /sbin/ethtool -K bond0 gso off && /sbin/ethtool -K bond0.123 gso off
	pre-up	/sbin/ethtool -K enp4s0f0 gro off && /sbin/ethtool -K enp4s0f1 gro off && /sbin/ethtool -K bond0 gro off && /sbin/ethtool -K bond0.123 gro off

iface bond0.123 inet6 manual
	up	ip -6 route add default via fe80::1 dev bond0.123 src 2001:db8:1::32
```

* I use no conntrack, or no ip/nftables at all
https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/

* Maybe this is of interest
https://fastnetmon.com/2017/12/04/traffic-filtration-using-nic-capabilities-on-wire-speed-10ge-14mpps/

* ToDo:
Test TCP Congestion Control htcp & bbr (both are not in the default debian kernel)\
https://www.techrepublic.com/article/how-to-enable-tcp-bbr-to-improve-network-speed-on-linux/
