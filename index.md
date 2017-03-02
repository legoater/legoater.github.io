A year ago, Benjamin Herrenschmidt pushed a QEMU tree on github 

https://github.com/ozbenh/qemu/

adding a new PowerPC platform. PowerNV (as Non-Virtualized) is the
"baremetal" platform using the OPAL firmware. It runs Linux on IBM and
Open Power systems and it can be used as an hypervisor OS, running KVM
guests, or simply as a host OS.

The QEMU tree Benjamin has pushed is mostly used to test  [skiboot](http://github.com/open-power/skiboot), a
component of OPAL, as a part of a regression testsuite. It is also quite useful for distro development, especially when you can't access real hardware to test your code on the host.

Below is a port of this work on a recent version of QEMU adding a couple of new features such as IPMI.

* to clone the updated QEMU tree and compile :

    https://github.com/legoater/qemu/tree/powernv-ipmi-2.8	(stable branch on 2.8)

    https://github.com/legoater/qemu/tree/powernv-ipmi-2.9	(wip branch)

* to boot, get a skiroot kernel and rootfs. You can find them on the OpenPOWER build site :

    https://openpower.xyz/job/openpower-op-build/distro=ubuntu,target=palmetto/lastSuccessfulBuild/artifact/images/

  or use these [zImage.epapr](http://www.kaod.org/openpower/qemu/zImage.epapr) and [rootfs.cpio.xz](http://www.kaod.org/openpower/qemu/rootfs.cpio.xz) files.

* run with :

```
drive=./fedora23-ppc64le.qcow2
qemu-system-ppc64 -m 2G -machine powernv,accel=tcg -cpu POWER8 \
	-device virtio-net,netdev=netdev0,disable-modern=false \
	-netdev user,id=netdev0,hostfwd=::20022-:22,hostname=pnv \
	-device virtio-blk,drive=drive0,disable-modern=false \
	-drive file=${drive},if=none,id=drive0 \
	-device ipmi-bmc-sim,sdrfile=./palmetto-SDR.bin,fruareasize=256,frudatafile=./palmetto-FRU.bin,id=bmc0 \
	-device isa-ipmi-bt,bmc=bmc0,irq=10 \
	-kernel ./zImage.epapr  \
	-initrd ./rootfs.cpio.xz \
	-bios ./skiboot.lid \
	-nographic -nodefaults
```
  The files [palmetto-SDR.bin](http://www.kaod.org/openpower/qemu/palmetto-SDR.bin) and [palmetto-FRU.bin](http://www.kaod.org/openpower/qemu/palmetto-FRU.bin) define a Sensor Data Record repository and a Field  Replaceable  Unit inventory for the BMC simulator used in qemu.

  NOTE about using virtio:

  There's a bug (yet to investigate) that prevents virtio devices to
  guess the endianness of the guest. As a consequence, legacy virtio
  devices are always considered to run big-endian, but the skiroot
  kernel runs little-endian, so the driver will fail.

  The solution is to use virtio 1.0 devices because they always run
  little-endian per specification. This is achieved with the
  disable-modern=false property, that can be passed to any instance
  of any virtio device type. It is also possible to pass this property
  to all instances of a given class with:

```
    -global virtio-blk-pci.disable-modern=false
```

* then just log on : 

```
    $ ssh -p 20022 root@localhost
    
    root@pnv:~# cat /proc/cpuinfo 
    processor	: 0
    cpu	: POWER8 (raw), altivec supported
    clock	: 1000.000000MHz
    revision	: 2.0 (pvr 004d 0200)
    timebase	: 512000000
    platform	: PowerNV
    model	: IBM PowerNV (emulated by qemu)
    machine	: PowerNV IBM PowerNV (emulated by qemu)
    firmware	: OPAL v3
```
