# 10 Gigabit Ethernet tweaks

I use network cards with Intel controller 82599 like Fujitsu D2755-A11 and Intel X520-DA2 Dual Port FC 10GbE SFP+.  
Old but has stable drivers (ixgbe) for most operating systems.  

Performance Tuning Tips from:

- Intel ixgbe driver README
- https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
- https://fasterdata.es.net/host-tuning/linux/
- https://github.com/torservers/server-config-templates/blob/master/sysctl.conf


My main problem: The driver stopped working when the network load was high,
syslog was flooded with:
```sh
ixgbe 0000:04:00.1 enp4s0f1: Detected Tx Unit Hang
ixgbe 0000:04:00.1 enp4s0f1: initiating reset due to tx timeout
```

Sollution:  
Disable virtualization (IOMMU, Intel VT-d, AMD-Vi, SR-IOV) options in BIOS / (U)EFI  
or use kernel boot option (intel_iommu=off, amd_iommu=off)  


* Install ethtool and disable GRO, TSO, GSO offloading in:

/etc/network/interfaces
```sh
iface enp4s0f0 inet manual
	pre-up	/sbin/ethtool -K $IFACE tso off
	pre-up	/sbin/ethtool -K $IFACE gso off
	pre-up	/sbin/ethtool -K $IFACE gro off

iface enp4s0f1 inet manual
	pre-up	/sbin/ethtool -K $IFACE tso off
	pre-up	/sbin/ethtool -K $IFACE gso off
	pre-up	/sbin/ethtool -K $IFACE gro off

auto bond0
iface bond0 inet manual
	bond-slaves		enp4s0f0 enp4s0f1
	bond-mode		802.3ad
	bond-miimon		100
	bond-xmit-hash-policy	layer3+4
```

* I use no conntrack, or no ip/nftables at all  
https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/

Maybe this is of interest  
https://fastnetmon.com/2017/12/04/traffic-filtration-using-nic-capabilities-on-wire-speed-10ge-14mpps/

* ToDo:  
Test TCP Congestion Control htcp & bbr (both are not in the default debian kernel)  
https://www.techrepublic.com/article/how-to-enable-tcp-bbr-to-improve-network-speed-on-linux/
