# Passing-the-Netronome-Smartnic-to-a-KVM-vm-guest
Record the operation, the problems encountered and the solutions

reference:

https://unixwars.blogspot.com/2019/05/passing-network-card-to-kvm-vm-guest.html

https://www.server-world.info/en/note?os=Ubuntu_18.04&p=kvm&f=2

https://help.netronome.com/support/solutions/articles/36000049975-basic-firmware-user-guide#appendix-a-netronome-repositories

https://serverfault.com/questions/633183/how-do-i-enable-kvm-device-passthrough-in-linux

1. Install the VM (Ubuntu 18.04) on the server

   note: we need to set up the vm guest machine type to **q35**, which supports the ICH9 chipset which can handle a PCIe bus
```
sudo apt-get upgrade -y
sudo apt install qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils
sudo virt-install --name vm3 --ram 4096 --disk path=/var/kvm/images/vm3.img,size=20 --vcpus 2 --os-variant ubuntu18.04 --graphics none --console pty,target_type=serial --location 'http://jp.archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/' --extra-args 'console=ttyS0,115200n8 serial' --machine q35
```
after the installation is finish, shut down the vm3, and then:
```
sudo guestmount -d vm3 -i /mnt
sudo ln -s /mnt/lib/systemd/system/getty@.service /mnt/etc/systemd/system/getty.target.wants/getty@ttyS0.service
sudo umount /mnt
```
after this, you can use virsh console vm3 to get into the vm3, otherwise it will hang up 
**Connected to domain vm3**, 
**Escape character is ^]**
and can`t get into the vm3.



2. Finding the card, here I have two netronome cards, The card's PCI address is **81:00.0 and 83:00.0**.
```
p4@testbed:~$ sudo lspci -nn|grep -i netronome
81:00.0 Ethernet controller [0200]: Netronome Systems, Inc. Device [19ee:4000]
83:00.0 Ethernet controller [0200]: Netronome Systems, Inc. Device [19ee:4000]
```
3. Handing out the card to the guest. Here I choose the **83:00.0** card
```
p4@testbed:~$ virsh nodedev-list | grep pci_0000_83
pci_0000_83_00_0
```
4. let host to leave **pci-0000:83:00.0** alone
```
p4@testbed:~$ sudo virsh nodedev-reattach pci_0000_83_00_0
Device pci_0000_83_00_0 reattached
```

**Note**: if you meet 

```
error: Failed to detach device pci_0000_83_00_0
error: Operation not supported: neither VFIO nor KVM device assignment is currently supported on this system
```
You can do this by setting the following in **/etc/default/grub**
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on"
```
Then run **update-grub** and **reboot**

5. Shut the vm guest down. Here vm3.
```
virsh shutdown vm3
```

6. Edit the vm3
```
virsh edit vm3
```
7. Add something like
```
<hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
          <address domain='0x0000' bus='0x83' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
```
to the end of the **devices** session. When you save it, it will properly place and configure the entry.

8. Open vm3, and install the **NFP Driver**
https://help.netronome.com/support/solutions/articles/36000049975-basic-firmware-user-guide (Appendix A: Netronome Repositories and Appendix B: Installing the Out-of-Tree NFP Driver)

9. In the vm3
```
vm3@vm3:~$ dmesg |grep nfp
[    1.209778] nfp: loading out-of-tree module taints kernel.
[    1.209921] nfp: module verification failed: signature and/or required key missing - tainting kernel
[    1.220229] nfp: NFP PCIe Driver, Copyright (C) 2014-2020 Netronome Systems
[    1.220230] nfp: NFP PCIe Driver, Copyright (C) 2021-2022 Corigine Inc.
[    1.220230] nfp src version: b1edb7c (o-o-t)
               nfp src path: /home/vm3/nfp-drv-kmods/src/
               nfp build user id: root
               nfp build user: root
               nfp build host: vm3
               nfp build path: /home/vm3/nfp-drv-kmods/src
[    1.220250] nfp-net-vnic: NFP vNIC driver, Copyright (C) 2010-2020 Netronome Systems
[    1.220251] nfp-net-vnic: NFP vNIC driver, Copyright (C) 2021-2022 Corigine Inc.
[    1.221762] nfp 0000:04:00.0: Network Flow Processor NFP4000/NFP5000/NFP6000 PCIe Card Probe
[    1.221848] nfp 0000:04:00.0: RESERVED BARs: 0.0: General/MSI-X SRAM, 0.1: PCIe XPB/MSI-X PBA, 0.4: Explicit0, 0.5: Explicit1, free: 20/24
[    1.222092] nfp 0000:04:00.0: Model: 0x62000010, SN: 00:15:4d:13:4e:d2, Ifc: 0x10ff
[    1.226165] nfp 0000:04:00.0: Assembly: SMCAMDA0097-000117291471-13 CPLD: 0x2020001
[    1.448043] nfp 0000:04:00.0: BSP: 01020d.01020d.01030d
[    1.772034] nfp 0000:04:00.0: nfp: Looking for firmware file in order of priority:
[    1.772060] nfp 0000:04:00.0: nfp:   netronome/serial-00-15-4d-13-4e-d2-10-ff.nffw: not found
[    1.772071] nfp 0000:04:00.0: nfp:   netronome/pci-0000:04:00.0.nffw: not found
[    1.772695] nfp 0000:04:00.0: nfp:   netronome/nic_AMDA0097-0001_2x40.nffw: found
[    1.772697] nfp 0000:04:00.0: Soft-resetting the NFP
[   12.568134] nfp 0000:04:00.0: nfp_nsp: Firmware from driver loaded, no FW selection policy HWInfo key found
[   12.568137] nfp 0000:04:00.0: Finished loading FW image
[   12.570591] nfp 0000:04:00.0: nfp: Failed to set VF count in sysfs: -38
[   14.571657] nfp 0000:04:00.0 eth0: NFP-6xxx Netdev: TxQs=2/32 RxQs=2/32
[   14.571661] nfp 0000:04:00.0 eth0: VER: 0.0.3.5, Maximum supported MTU: 9216
[   14.571666] nfp 0000:04:00.0 eth0: CAP: 0x78040233 PROMISC RXCSUM TXCSUM GATHER TSO2 RSS2 IRQMOD RXCSUM_COMPLETE 
[   14.575605] nfp 0000:04:00.0 eth1: NFP-6xxx Netdev: TxQs=2/31 RxQs=2/31
[   14.575609] nfp 0000:04:00.0 eth1: VER: 0.0.3.5, Maximum supported MTU: 9216
[   14.575613] nfp 0000:04:00.0 eth1: CAP: 0x78040233 PROMISC RXCSUM TXCSUM GATHER TSO2 RSS2 IRQMOD RXCSUM_COMPLETE 
[   14.578768] nfp 0000:04:00.0 enp4s0np0: renamed from eth0
[   14.616453] nfp 0000:04:00.0 enp4s0np1: renamed from eth1

```

```
vm3@vm3:~$ ifconfig -a
enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.19  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::5054:ff:fe89:82af  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:89:82:af  txqueuelen 1000  (Ethernet)
        RX packets 52  bytes 3770 (3.7 KB)
        RX errors 0  dropped 36  overruns 0  frame 0
        TX packets 15  bytes 1694 (1.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp4s0np0: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether 00:15:4d:13:4e:d3  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp4s0np1: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether 00:15:4d:13:4e:d7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 84  bytes 6368 (6.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 84  bytes 6368 (6.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

**The problem I meet and the solution:**

1. 
```
vm3@vm3:~$ echo "deb https://deb.netronome.com/apt stable main" > \
> /etc/apt/sources.list.d/netronome.list
-bash: /etc/apt/sources.list.d/netronome.list: Permission denied
```

**solution**:
```
echo "deb https://deb.netronome.com/apt stable main" | sudo tee /etc/apt/sources.list.d/netronome.list
```
2.
```
N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'https://deb.netronome.com/apt stable InRelease' doesn't support architecture 'i386'
```

**solution**:
```
echo "deb [arch=amd64] https://deb.netronome.com/apt stable main" | sudo tee /etc/apt/sources.list.d/netronome.list
```
3.
```
vm3@vm3:~$ cat /etc/apt/sources.list.d/netronome.list
cat: /etc/apt/sources.list.d/netronome.list: No such file or directory
```
**solution**:
```
sudo nano /etc/apt/sources.list.d/netronome.list
deb https://deb.netronome.com/apt stable main
sudo apt-get update
```

