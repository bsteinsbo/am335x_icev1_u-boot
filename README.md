# U-boot support for AM3359 ICE V1

This project contains some needed patches to compile u-boot for the ICE V1 board.  The ICE V1 is discontinued and unsupported by TI, to the extent that information about the board is hard to find.  Some information may be found at http://processors.wiki.ti.com/index.php/AM335x_Industrial_Communication_Engine_(ICE)_EVM_HW_User_Guide

The main differences between V1 and the still supported V2 of ICE are:
- DDR2 SDRAM on V1, DDR3 on V2
- MMC1 used on V1, MMC0 used on V2
- V2 can multiplex the Ethernet phys between PRUSS and CPSW
- UART5 connected to USB debug connector on V1, while V2 uses UART3
- V1 cannot boot directly from MMC

In order to boot Linux on the ICE V1, we first need U-boot for SPI, both MLO and the full u-boot.  Apply the provided patches to the ti-u-boot-2017.01 branch of TI's u-boot sorces found at http://git.ti.com/ti-u-boot/ti-u-boot and configure and build u-boot for am335x_icev1_defconfig.  The resulting MLO.byteswap file needs some further processing in order to be bootable.
```
% dd if=MLO.byteswap of=MLO.spiboot bs=1b skip=1
```

You then need a way to flash these files to the SPI flash on the ICE V1 board.  MLO.spiboot should be stored at the beginning of the SPI flash (offset 0x0) while u-boot.img should be stored at offset 0x20000.  A flasher program for SPI can be found in the ti-processor-sdk-rtos-am335x, in the `pdk_am335x_x_x_x/packages/ti/starterware/tools/flash_writer/` directory.  Copy the source project in CCS, add the target configuration and gel file from the `spi-flash-writer_AM335x additions`, activate the target configuration and run the program.  It's very slow, but should do the job.

The default environment variables in u-boot will need some manual adjustments.  Fix it so that you get ttyS5 as the console, and boot from mmc1.  The name of the dtb must also be changed.

A basic device tree file can be found in the `devicetree` directory.  It should be copied to the boot directory on the root file system on the micro SD card.

The kernel included in ti-processor-sdk-linux-am335x-evm-04.00.00.04 contains a bug in the prueth driver.  Apply patch found in the ti-linux-kernel directory if applicable to your kernel.

The board requires the PRU internal mux register to be set to 1 (default value is 0) for the pin mux used for Ethernet port 0.  The patch to the pruss driver found in the ti-linux-kernel directory adds a quick hack to accomplish this.

One final note of warning: the list of ttys in the `/etc/securettys` stops at ttyS3.  Add ttyS5 to this list in order to be allowed to log on as root over the default serial connection.

Bootlog:
```
[  339.443326] reboot: Restarting system

U-Boot SPL 2017.01-dirty (Aug 06 2017 - 12:00:45)
Trying to boot from SPI

U-Boot 2017.01-dirty (Aug 06 2017 - 12:00:45 +0200)

CPU  : AM335X-GP rev 1.0
DRAM:  256 MiB
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
SF: Detected w25q64cv with page size 256 Bytes, erase size 4 KiB, total 8 MiB
Net:   Net Initialization Skipped
No ethernet found.
Hit any key to stop autoboot:  0
switch to partitions #0, OK
mmc1 is current device
SD/MMC found on device 1
3702520 bytes read in 318 ms (11.1 MiB/s)
38750 bytes read in 44 ms (859.4 KiB/s)
## Flattened Device Tree blob at 88000000
   Booting using the fdt blob at 0x88000000
   Loading Device Tree to 8df3f000, end 8df4b75d ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.9.28-geed43d1050 (bjarne@MSI) (gcc version 6.2.1 20161016 (Linaro GCC 6.2-2016.11) ) #12 PREEMPT Fri Aug 11 09:10:37 CEST 2017
[    0.000000] CPU: ARMv7 Processor [413fc082] revision 2 (ARMv7), cr=10c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] OF: fdt:Machine model: TI AM3359 ICE-V1
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 48 MiB at 0x8a800000
[    0.000000] Memory policy: Data cache writeback
[    0.000000] CPU: All CPU(s) started in SVC mode.
[    0.000000] AM335X ES1.0 (sgx neon)
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 64960
[    0.000000] Kernel command line: console=ttyS5,115200n8 root=PARTUUID=5a7089a1-01 rw rootfstype=ext4 rootwait
[    0.000000] PID hash table entries: 1024 (order: 0, 4096 bytes)
[    0.000000] Dentry cache hash table entries: 32768 (order: 5, 131072 bytes)
[    0.000000] Inode-cache hash table entries: 16384 (order: 4, 65536 bytes)
[    0.000000] Memory: 198240K/262144K available (7168K kernel code, 373K rwdata, 2512K rodata, 1024K init, 272K bss, 14752K reserved, 49152K cma-reserved, 0K highmem)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)
[    0.000000]     vmalloc : 0xd0800000 - 0xff800000   ( 752 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xd0000000   ( 256 MB)
[    0.000000]     pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
[    0.000000]     modules : 0xbf000000 - 0xbfe00000   (  14 MB)
[    0.000000]       .text : 0xc0008000 - 0xc0800000   (8160 kB)
[    0.000000]       .init : 0xc0b00000 - 0xc0c00000   (1024 kB)
[    0.000000]       .data : 0xc0c00000 - 0xc0c5d5d0   ( 374 kB)
[    0.000000]        .bss : 0xc0c5d5d0 - 0xc0ca18b4   ( 273 kB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] Preemptible hierarchical RCU implementation.
[    0.000000] 	Build-time adjustment of leaf fanout to 32.
[    0.000000] NR_IRQS:16 nr_irqs:16 16
[    0.000000] IRQ: Found an INTC at 0xfa200000 (revision 5.0) with 128 interrupts
[    0.000000] OMAP clockevent source: timer2 at 24000000 Hz
[    0.000017] sched_clock: 32 bits at 24MHz, resolution 41ns, wraps every 89478484971ns
[    0.000040] clocksource: timer1: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
[    0.000051] OMAP clocksource: timer1 at 24000000 Hz
[    0.000262] clocksource_probe: no matching clocksources found
[    0.000459] Console: colour dummy device 80x30
[    0.000505] Calibrating delay loop... 718.02 BogoMIPS (lpj=3590144)
[    0.118987] pid_max: default: 32768 minimum: 301
[    0.119136] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes)
[    0.119149] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes)
[    0.120065] CPU: Testing write buffer coherency: ok
[    0.120492] Setting up static identity map for 0x80100000 - 0x80100060
[    0.123946] EFI services will not be available.
[    0.125646] devtmpfs: initialized
[    0.139634] VFP support v0.3: implementor 41 architecture 3 part 30 variant c rev 3
[    0.140012] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.140043] futex hash table entries: 256 (order: -1, 3072 bytes)
[    0.144144] pinctrl core: initialized pinctrl subsystem
[    0.145602] NET: Registered protocol family 16
[    0.147828] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.163998] omap_hwmod: debugss: _wait_target_disable failed
[    0.238987] cpuidle: using governor ladder
[    0.268972] cpuidle: using governor menu
[    0.273938] GPIO line 7 (FET_SWITCH_CTRL) hogged as output/high
[    0.276076] OMAP GPIO hardware version 0.1
[    0.292782] hw-breakpoint: debug architecture 0x4 unsupported.
[    0.338341] edma 49000000.edma: TI EDMA DMA engine driver
[    0.342831] omap_i2c 44e0b000.i2c: could not find pctldev for node /ocp/l4_wkup@44c00000/scm@210000/pinmux@800/i2c0_pins_default, deferring probe
[    0.342995] media: Linux media interface: v0.10
[    0.343061] Linux video capture interface: v2.00
[    0.343111] pps_core: LinuxPPS API ver. 1 registered
[    0.343120] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.343148] PTP clock support registered
[    0.343194] EDAC MC: Ver: 3.0.0
[    0.344482] omap-mailbox 480c8000.mailbox: omap mailbox rev 0x400
[    0.344886] Advanced Linux Sound Architecture Driver Initialized.
[    0.346361] clocksource: Switched to clocksource timer1
[    0.358473] NET: Registered protocol family 2
[    0.359377] TCP established hash table entries: 2048 (order: 1, 8192 bytes)
[    0.359415] TCP bind hash table entries: 2048 (order: 1, 8192 bytes)
[    0.359446] TCP: Hash tables configured (established 2048 bind 2048)
[    0.359531] UDP hash table entries: 256 (order: 0, 4096 bytes)
[    0.359555] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes)
[    0.359712] NET: Registered protocol family 1
[    0.360211] RPC: Registered named UNIX socket transport module.
[    0.360226] RPC: Registered udp transport module.
[    0.360233] RPC: Registered tcp transport module.
[    0.360240] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.361290] hw perfevents: enabled with armv7_cortex_a8 PMU driver, 5 counters available
[    0.363981] workingset: timestamp_bits=14 max_order=16 bucket_order=2
[    0.372941] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.374058] NFS: Registering the id_resolver key type
[    0.374113] Key type id_resolver registered
[    0.374122] Key type id_legacy registered
[    0.374179] ntfs: driver 2.1.32 [Flags: R/O].
[    0.376486] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 246)
[    0.376510] io scheduler noop registered
[    0.376519] io scheduler deadline registered
[    0.376719] io scheduler cfq registered (default)
[    0.377724] pinctrl-single 44e10800.pinmux: please update dts to use #pinctrl-cells = <1>
[    0.378183] pinctrl-single 44e10800.pinmux: 142 pins at pa f9e10800 size 568
[    0.450935] Serial: 8250/16550 driver, 10 ports, IRQ sharing disabled
[    0.455420] 481a8000.serial: ttyS4 at MMIO 0x481a8000 (irq = 158, base_baud = 3000000) is a 8250
[    0.456844] 481aa000.serial: ttyS5 at MMIO 0x481aa000 (irq = 159, base_baud = 3000000) is a 8250
[    1.040968] console [ttyS5] enabled
[    1.046472] omap_rng 48310000.rng: OMAP Random Number Generator ver. 20
[    1.053265] [drm] Initialized
[    1.072374] brd: module loaded
[    1.083072] loop: module loaded
[    1.089463] m25p80 spi1.0: found s25fl064k, expected w25q64
[    1.095086] m25p80 spi1.0: s25fl064k (8192 Kbytes)
[    1.100106] 6 ofpart partitions found on MTD device spi1.0
[    1.105618] Creating 6 MTD partitions on "spi1.0":
[    1.110470] 0x000000000000-0x000000020000 : "u-boot-spl"
[    1.117402] 0x000000020000-0x0000000a0000 : "u-boot"
[    1.123738] 0x0000000a0000-0x0000000c0000 : "u-boot-env1"
[    1.130585] 0x0000000c0000-0x0000000e0000 : "u-boot-env2"
[    1.137301] 0x0000000e0000-0x000000450000 : "kernel"
[    1.143564] 0x000000450000-0x000000800000 : "misc"
[    1.151943] libphy: Fixed MDIO Bus: probed
[    1.158725] mousedev: PS/2 mouse device common for all mice
[    1.164903] i2c /dev entries driver
[    1.171360] cpuidle: enable-method property 'ti,am3352' found operations
[    1.179454] omap_hsmmc 481d8000.mmc: Got CD GPIO
[    1.208919] ledtrig-cpu: registered to indicate activity on CPUs
[    1.217444] NET: Registered protocol family 10
[    1.223375] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    1.230376] NET: Registered protocol family 17
[    1.235154] Key type dns_resolver registered
[    1.239849] omap_voltage_late_init: Voltage driver support not added
[    1.276634] tps65910 0-002d: No interrupt support, no core IRQ
[    1.284819] vrtc: supplied by vbat
[    1.292153] vio: supplied by vbat
[    1.297101] vdd1: supplied by vbat
[    1.302423] vdd2: supplied by vbat
[    1.309196] vdig1: supplied by vbat
[    1.313519] random: fast init done
[    1.317599] vdig2: supplied by vbat
[    1.322551] vpll: supplied by vbat
[    1.327424] vdac: supplied by vbat
[    1.332281] vaux1: supplied by vbat
[    1.337282] vaux2: supplied by vbat
[    1.342221] vaux33: supplied by vbat
[    1.347287] vmmc: supplied by vbat
[    1.352155] vbb: supplied by vbat
[    1.357239] omap_i2c 44e0b000.i2c: bus 0 rev0.11 at 400 kHz
[    1.363651] omap_hsmmc 481d8000.mmc: Got CD GPIO
[    1.427398] hctosys: unable to open rtc device (rtc0)
[    1.433341] ALSA device list:
[    1.436468]   No soundcards found.
[    1.443937] Waiting for root device PARTUUID=5a7089a1-01...
[    1.543938] mmc0: host does not support reading read-only switch, assuming write-enable
[    1.554240] mmc0: new high speed SDHC card at address aaaa
[    1.560872] mmcblk0: mmc0:aaaa SU16G 14.8 GiB 
[    1.567411]  mmcblk0: p1
[    1.699204] EXT4-fs (mmcblk0p1): mounted filesystem with ordered data mode. Opts: (null)
[    1.707542] VFS: Mounted root (ext4 filesystem) on device 179:1.
[    1.720990] devtmpfs: mounted
[    1.726565] Freeing unused kernel memory: 1024K (c0b00000 - c0c00000)
[    2.016525] systemd[1]: System time before build time, advancing clock.
[    2.101642] systemd[1]: systemd 230 running in system mode. (+PAM -AUDIT -SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP -LIBCRYPTSETUP -GCRYPT -GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID -ELFUTILS +KMOD -IDN)
[    2.120838] systemd[1]: Detected architecture arm.

Welcome to Arago 2017.05!

[    2.158029] systemd[1]: Set hostname to <am335x-evm>.
[    2.742742] systemd[1]: Reached target Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[    2.777229] systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Dispatch Password Requests to Console Directory Watch.
[    2.817063] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password Requests to Wall Directory Watch.
[    2.857176] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[    2.887402] systemd[1]: Listening on Network Service Netlink Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[    2.927110] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[    2.956704] systemd[1]: Reached target Swap.
[  OK  ] Reached target Swap.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Listening on udev Kernel Socket.
[  OK  ] Created slice System Slice.
         Mounting Debug File System...
         Starting Setup Virtual Console...
         Starting Load Kernel Modules...
         Mounting POSIX Message Queue File System...
         Mounting Temporary Directory...
[    3.316198] cryptodev: loading out-of-tree module taints kernel.
[  OK  ] Listening on /dev/initctl Compatibility Named Pipe.
[    3.351152] cryptodev: driver 1.8 loaded.
[  OK  ] Created slice system-getty.slice.
[  OK  ] Created slice User and Session Slice.
[  OK  ] Listening on Syslog Socket.
         Starting Journal Service...
         Starting Create list of required st... nodes for the current kernel...
         Starting Remount Root and Kernel File Systems...
[  OK  ] Reached target Slices.
[  OK  ] Reached target Paths.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[    3.638665] EXT4-fs (mmcblk0p1): re-mounted. Opts: (null)
[  OK  ] Mounted Debug File System.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Temporary Directory.
[  OK  ] Started Journal Service.
[  OK  ] Started Setup Virtual Console.
[  OK  ] Started Load Kernel Modules.
[  OK  ] Started Create list of required sta...ce nodes for the current kernel.
[  OK  ] Started Remount Root and Kernel File Systems.
         Starting udev Coldplug all Devices...
         Starting Create Static Device Nodes in /dev...
         Mounting Configuration File System...
         Starting Apply Kernel Variables...
         Starting Flush Journal to Persistent Storage...
[  OK  ] Mounted Configuration File System.
[  OK  ] Started Create Static Device Nodes in /dev.
[  OK  ] Started Apply Kernel Variables.
[    4.389108] systemd-journald[99]: Received request to flush runtime journal from PID 1
[  OK  ] Reached target Local File Systems (Pre).
         Mounting /media/ram...
         Mounting /var/volatile...
         Starting udev Kernel Device Manager...
[  OK  ] Mounted /var/volatile.
[  OK  ] Mounted /media/ram.
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started udev Kernel Device Manager.
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems.
         Starting Create Volatile Files and Directories...
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started Create Volatile Files and Directories.
         Starting Network Time Synchronization...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Started Network Time Synchronization.
         Starting Synchronize System and HW clocks...
[  OK  ] Reached target System Time Synchronized.
[FAILED] Failed to start Synchronize System and HW clocks.
See 'systemctl status sync-clocks.service' for details.
[    6.465338] omap_rtc 44e3e000.rtc: already running
[    6.504915] omap_rtc 44e3e000.rtc: rtc core: registered 44e3e000.rtc as rtc0
[  OK  ] Started udev Coldplug all Devices.
[    6.608903] omap_wdt: OMAP Watchdog Timer Rev 0x01: initial timeout 60 sec
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Timers.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
[  OK  ] Started D-Bus System Message Bus.
[    7.860372] omap-sham 53100000.sham: hw accel on OMAP rev 4.3
         Starting Network Service...
[    7.998463] omap-aes 53500000.aes: OMAP AES hw accel rev: 3.2
         Starting Print notice about GPLv3 packages...
[    8.036726] omap-aes 53500000.aes: will run requests pump with realtime priority
         Starting Login Service...
[  OK  ] Started System Logging Service.
         Starting telnetd.service...
[    8.468479] remoteproc remoteproc0: wkup_m3 is available
[    8.495921] ti-pruss 4a300000.pruss: creating PRU cores and other child platform devices
[    8.537789] remoteproc remoteproc0: powering up wkup_m3
[    8.552276] remoteproc remoteproc0: Booting fw image am335x-pm-firmware.elf, size 224344
[    8.552563] remoteproc remoteproc0: remote processor wkup_m3 is now up
[    8.552593] wkup_m3_ipc 44e11324.wkup_m3_ipc: CM3 Firmware Version = 0x192
[    8.556218] PM: bootloader does not support rtc-only!
[  OK  ] Started Kernel Logging Service.
[  OK  ] Started Network Service.
[  OK  ] Started telnetd.service.
[    9.256554] davinci_mdio 4a332400.mdio: davinci mdio revision 1.6
[    9.262712] libphy: 4a332400.mdio: probed
[  OK  ] Found device /dev/ttyS5.
[    9.489759] davinci_mdio 4a332400.mdio: phy[1]: device 4a332400.mdio:01, driver TI TLK10X 10/100 Mbps PHY
[    9.622659] davinci_mdio 4a332400.mdio: phy[2]: device 4a332400.mdio:02, driver TI TLK10X 10/100 Mbps PHY
[  OK  ] Found device /dev/ttyS0.
[    9.780383] remoteproc remoteproc1: 4a334000.pru0 is available
[  OK  ] Found device /dev/ttyS3.
[    9.938300] pru-rproc 4a334000.pru0: PRU rproc node /ocp/pruss_soc_bus@4a326000/pruss@4a300000/pru@4a334000 probed successfully
[   10.098918] remoteproc remoteproc2: 4a338000.pru1 is available
[   10.104888] pru-rproc 4a338000.pru1: PRU rproc node /ocp/pruss_soc_bus@4a326000/pruss@4a300000/pru@4a338000 probed successfully
[  OK  ] Started Login Service.
         Starting thttpd.service...
[  OK  ] Reached target Network.
[   10.434100] prueth pruss_eth: port 1: using random MAC addr: fe:5c:2b:c3:01:01
         Starting Permit User Sessions...
         Starting Network Name Resolution...
[   10.777356] prueth pruss_eth: port 2: using random MAC addr: aa:fb:07:2a:f9:c2
[   10.952521] prueth pruss_eth: TI PRU ethernet (type 0, rxqSz: 194 194 194 194 0) driver initialized
[  OK  ] Started Permit User Sessions.
[  OK  ] Started thttpd.service.
[  OK  ] Started Network Name Resolution.
[  OK  ] Listening on Load/Save RF Kill Switch Status /dev/rfkill Watch.
         Starting rng-tools.service...
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyS5.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Started Serial Getty on ttyS3.
[  OK  ] Reached target Login Prompts.
[   12.604168] random: crng init done
[  OK  ] Started rng-tools.service.
         Starting thermal-zone-init.service...
[  OK  ] Started thermal-zone-init.service.
***************************************************************
***************************************************************
NOTICE: This file system contains the following GPLv3 packages:
	binutils
	dosfstools
	libreadline6
	m4

If you do not wish to distribute GPLv3 components please remove
the above packages prior to distribution.  This can be done using
the opkg remove command.  i.e.:
    opkg remove <package>
Where <package> is the name printed in the list above

NOTE: If the package is a dependency of another package you
      will be notified of the dependent packages.  You should
      use the --force-removal-of-dependent-packages option to
      also remove the dependent packages as well
***************************************************************
***************************************************************
[  OK  ] Started Print notice about GPLv3 packages.
[   15.641015] remoteproc remoteproc1: powering up 4a334000.pru0
[   15.662332] remoteproc remoteproc1: Booting fw image ti-pruss/am335x-pru0-prueth-fw.elf, size 4688
[   15.683834] ti-pruss 4a300000.pruss: configured system_events = 0x0000060000500000 intr_channels = 0x00000095 host_intr = 0x00000115
[   15.716831] remoteproc remoteproc1: remote processor 4a334000.pru0 is now up
[   15.724006] net eth0: started
[   15.749664] IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
[   15.812415] remoteproc remoteproc2: powering up 4a338000.pru1
[   15.828684] remoteproc remoteproc2: Booting fw image ti-pruss/am335x-pru1-prueth-fw.elf, size 4764
[   15.855854] ti-pruss 4a300000.pruss: configured system_events = 0x0060000000a00000 intr_channels = 0x0000012a host_intr = 0x0000022a
[   15.886673] remoteproc remoteproc2: remote processor 4a338000.pru1 is now up
[   15.893849] net eth1: started
[   15.916679] IPv6: ADDRCONF(NETDEV_UP): eth1: link is not ready
[  OK  ] Reached target Multi-User System.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
[   16.967302] prueth pruss_eth eth1: Link is Up - 100Mbps/Full - flow control rx/tx
[   16.974899] IPv6: ADDRCONF(NETDEV_CHANGE): eth1: link becomes ready

 _____                    _____           _         _   
|  _  |___ ___ ___ ___   |  _  |___ ___  |_|___ ___| |_ 
|     |  _| .'| . | . |  |   __|  _| . | | | -_|  _|  _|
|__|__|_| |__,|_  |___|  |__|  |_| |___|_| |___|___|_|  
              |___|                    |___|            

Arago Project http://arago-project.org am335x-evm ttyS5

Arago 2017.05 am335x-evm ttyS5

am335x-evm login: 
```
