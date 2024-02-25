# Reverse Engineering progress

## The Twonkie Detour, or: if you want to take a photo, you must first compile the Linux Kernel

I bought prebuilt [Twonkie](https://github.com/dojoe/Twonkie) v2 hardware. I'm trying to use it in a WSL 2 Ubuntu to capture USB traffic between two proprietary devices.

I've set up [usbipd-win](https://github.com/dorssel/usbipd-win) and `usbipd list` shows the device as `18d1:500a` "Shell, USB-PD Sniffer, Commands", so apparently it already has firmware on it. I bound it via `usbipd bind --busid=<BUSID>` and attached it to WSL via `usbipd attach --wsl --busid=<BUSID>`. In Ubuntu, `lsusb` shows it as "ID 18d1:500a Google Inc. Twinkie", but I don't see the `/dev/ttyUSBx` console. I seem to be lacking the `usbserial` modprobe, which apparently means I need to compile a new Kernel.

Before I do that, I want to try out the [Python sample](https://github.com/dojoe/Twonkie/blob/master/fw/util/shell.py) on Windows, so I have to install PyPy. For some reason I cannot get pip to work, so I cannot install the required `libusb1`. Instead of trying to debug that, I'll give the kernel thing a try. [These instructions](https://github.com/rpasek/usbip-wsl2-instructions/blob/master/README.md) look promising, if I adjust them slightly.

```sh
$ uname -r
5.15.133.1-microsoft-standard-WSL2
$ git clone https://github.com/microsoft/WSL2-Linux-Kernel.git --depth 1 --branch linux-msft-wsl-5.15.y
```

Cloning the Linux Kernel turns out to take a while. I can only imagine how long it would have taken if I had not used a shallow clone. I could not find a branch for my specific Kernel version, so I went with the one above, but in hindsight I realize I should have been looking for a tag, namely `linux-msft-wsl-5.15.133.1`. Oh well, using a slightly newer Kernel should be fine, I'm not aborting my clone, it's already at 40%.

Maybe I should not have cloned onto my mounted NTFS Windows Drive...

```
warning: the following paths have collided (e.g. case-sensitive paths
on a case-insensitive filesystem) and only one from the same
colliding group is in the working tree:

  'include/uapi/linux/netfilter/xt_CONNMARK.h'
  'include/uapi/linux/netfilter/xt_connmark.h'
  'include/uapi/linux/netfilter/xt_DSCP.h'
  'include/uapi/linux/netfilter/xt_dscp.h'
  'include/uapi/linux/netfilter/xt_MARK.h'
  'include/uapi/linux/netfilter/xt_mark.h'
  'include/uapi/linux/netfilter/xt_RATEEST.h'
  'include/uapi/linux/netfilter/xt_rateest.h'
  'include/uapi/linux/netfilter/xt_TCPMSS.h'
  'include/uapi/linux/netfilter/xt_tcpmss.h'
  'include/uapi/linux/netfilter_ipv4/ipt_ECN.h'
  'include/uapi/linux/netfilter_ipv4/ipt_ecn.h'
  'include/uapi/linux/netfilter_ipv4/ipt_TTL.h'
  'include/uapi/linux/netfilter_ipv4/ipt_ttl.h'
  'include/uapi/linux/netfilter_ipv6/ip6t_HL.h'
  'include/uapi/linux/netfilter_ipv6/ip6t_hl.h'
  'net/netfilter/xt_DSCP.c'
  'net/netfilter/xt_dscp.c'
  'net/netfilter/xt_HL.c'
  'net/netfilter/xt_hl.c'
  'net/netfilter/xt_RATEEST.c'
  'net/netfilter/xt_rateest.c'
  'net/netfilter/xt_TCPMSS.c'
  'net/netfilter/xt_tcpmss.c'
  'tools/memory-model/litmus-tests/Z6.0+pooncelock+poonceLock+pombonce.litmus'
  'tools/memory-model/litmus-tests/Z6.0+pooncelock+pooncelock+pombonce.litmus'
```

Surely nothing bad will come of this. 😬

Actually, I won't fuck around and find out, I'll use this opportunity to try the precise tag instead:

```sh
$ git clone https://github.com/microsoft/WSL2-Linux-Kernel.git --depth 1 --branch linux-msft-wsl-5.15.133.1
$ cd WSL2-Linux-Kernel
$ cp /proc/config.gz config.gz
$ gunzip config.gz
$ mv config .config
$ make menuconfig
```

```txt
Device Drivers -> USB support -> USB Serial Converter support -> USB Generic Serial Driver, USB Serial Console device support, USB Serial Simple Driver
```

Hopefully that will cover it. Let's build our first Linux Kernel!

```sh
$ make -j 8
$ sudo make modules_install -j 8
arch/x86/Makefile:142: CONFIG_X86_X32 enabled but no binutils support
  INSTALL /lib/modules/5.15.133.1-microsoft-standard-WSL2+/kernel/drivers/usb/serial/usb-serial-simple.ko
  DEPMOD  /lib/modules/5.15.133.1-microsoft-standard-WSL2+
$ sudo make install -j 8
W: Couldn't identify type of root file system for fsck hook
I: The initramfs will attempt to resume from /dev/sdb
I: (UUID=646e9271-66fb-4332-bd93-07fff848add0)
I: Set the RESUME variable to override this.
run-parts: executing /etc/kernel/postinst.d/unattended-upgrades 5.15.133.1-microsoft-standard-WSL2+ /boot/vmlinuz-5.15.133.1-microsoft-standard-WSL2+
run-parts: executing /etc/kernel/postinst.d/update-notifier 5.15.133.1-microsoft-standard-WSL2+ /boot/vmlinuz-5.15.133.1-microsoft-standard-WSL2+
run-parts: executing /etc/kernel/postinst.d/xx-update-initrd-links 5.15.133.1-microsoft-standard-WSL2+ /boot/vmlinuz-5.15.133.1-microsoft-standard-WSL2+
I: /boot/vmlinuz.old is now a symlink to vmlinuz-5.15.133.1-microsoft-standard-WSL2+
I: /boot/initrd.img.old is now a symlink to initrd.img-5.15.133.1-microsoft-standard-WSL2+
I: /boot/vmlinuz is now a symlink to vmlinuz-5.15.133.1-microsoft-standard-WSL2+
I: /boot/initrd.img is now a symlink to initrd.img-5.15.133.1-microsoft-standard-WSL2+
```

Rebooted from Windows via

```sh
$ wsl --shutdown
$ wsl
```

But `uname -v` still shows a date from months ago...

Let's try [these instructions](https://askubuntu.com/questions/1373910/ch340-serial-device-doesnt-appear-in-dev-wsl/1374775#1374775)...

```sh
$ cp arch/x86/boot/bzImage /mnt/c/Users/mrwon/wsl_kernel
```

Created `C:\Users\mrwon\.wslconfig` containing

```txt
[wsl2]
kernel = C:\\Users\\mrwon\\wsl_kernel
```

An another `wsl --shutdown`.

```sh
$ uname -v
#1 SMP Sat Feb 10 03:49:34 CET 2024
```

Okay, it loaded the right Kernel. I could have skipped `make modules_install` and `make install`, presumably.

And let usbipd-win attach the device again: `usbipd attach --wsl --busid=2-5`

Still, no `/dev/ttyUSBx`, not even after `sudo modprobe usbserial`. But! I now have `/sys/bus/usb-serial/drivers/generic`. And after I

```sh
$ echo '18d1 500A' | sudo tee /sys/bus/usb-serial/drivers/generic/new_id
```

I can finally see `/dev/ttyUSB{0..2}`! Not sure how to interact with them, though. So let's just try and capture a trace.

This requires a patched version of sigrok, which apparently is tricky to build on Windows, but Google provides [binaries for Linux](https://www.chromium.org/chromium-os/twinkie/#using-as-a-pd-packet-sniffer).

```sh
$ cd $(mktemp -d)
$ wget https://storage.googleapis.com/chromeos-vpa/twinkie_20201028/ubuntu_20.04/libsigrok4_0.5.2-2+twinkie_amd64.deb \
    https://storage.googleapis.com/chromeos-vpa/twinkie_20201028/ubuntu_20.04/libsigrokcxx4_0.5.2-2+twinkie_amd64.deb \
    https://storage.googleapis.com/chromeos-vpa/twinkie_20201028/ubuntu_20.04/libsigrokdecode4_0.5.3-1_amd64.deb \
    https://storage.googleapis.com/chromeos-vpa/twinkie_20201028/ubuntu_20.04/sigrok-cli_0.7.1-1_amd64.deb \
    https://storage.googleapis.com/chromeos-vpa/twinkie_20201028/ubuntu_20.04/pulseview_0.5.0~git20200910+1acc207a-1_amd64.deb
$ sudo dpkg -i *.deb
$ sudo apt-get install -f
```

And now I'm finally ready to capture traces!


```sh
$ sudo sigrok-cli -d chromium-twinkie --continuous --output-format binary | mbuffer | lz
4 | pv > 001-boot-and-zoom.lz4q
sr: twinkie: Unable to claim USB interface. Another program or driver has already claimed it.
Failed to open device.
```

Oh.

But rebooting seemed to fix it, maybe interacting with `ttyUSB0` took ownership of the device or something... Before rebooting I took a guess at what driver held ownership and tried removing it, unsuccessfully:

```sh
$ sudo modprobe -r usb-serial-simple
```

So then I recorded my first dump. But I'll not analyze it today.

I took another dump, this time trying `pv -c` to avoid spamming newlines:

```sh
$ sudo sigrok-cli -d chromium-twinkie --continuous --output-format binary | mbuffer | lz4 | pv -c > 002-boot
-and-zoom-again.lz4q
```

I don't know what parameters to use to load that binary file into PulseView, so I tried recording a normal .sr file instead:

```sh
$ sudo sigrok-cli -d chromium-twinkie --continuous | mbuffer | pv -c > 003-boot-and-zoom-but-not-binary.sr
```

Attempting to open this with nightly PulseView on Windows gives "generic/unspecified error". Attempting to run `pulseview` in WSL gives "Segmentation fault".

Okay, reading the [sigrok Twinkie patch](https://github.com/vpalatin/libsigrok/commit/8147e36b8ffa48a23cf898332c351f6a6d85484b?diff=unified&w=0#diff-aeb7aba13ae6cea8b5bb946cf5ace8a208c7bb2145ab48ffaf3bfefe2b2ab96e) clarifies that there are 2 channels sampled at 2.4MHz.

Using this info, I can load the binary data, but it's just a repeated 0, until it turns to 1 when the device is turned on, and stays there.

As a baseline, I captured keyboard traffic instead. That worked better, and I started seeing USB PD packages, namely the sink (keyboard) sending "REQ Disc Ident SVID:ff00" packages and the source (laptop) sending "[1] [Fixed] 5V 3A (15W) [dual_role_power] [comm_cap] [dual_role_data]". I did not see my keystrokes.

That's when the penny dropped. I got a USB-PD Sniffer. PD is not Packet Data or anything like that, it's Power Delivery. I can now analyze the negotiation of how much power at what voltage a device may use. I can _not_ analyze what data is sent.

Back to square 1.

So I'm trying different cables to find out how many channels I'll need to capture to sniff this. A USB-C cable that's visibly lacking about half the pins doesn't work. The 10Gbps one for my webcam does, even though unlike the original cable it has a 2-pin-gap in the center on one side. (B6/B7? That appears to be the duplicate D+/D-, I assume it uses A6/A7 instead.)

The one that doesn't work lacks A2/A3 (TX1+/-) A5 (CC1) A10/A11 (RX2-/+) and the same on the B side. So really only GND, D+/-, VBUS, SBU1/2 (sideband use). TX+RX is SuperSpeed. I think I'm reading this mirrored, it's probably missing SBU instead of CC, CC is used to configure the orientation initially. Without superspeed and sideband, we're looking at USB 2.0.

## Fine, I'll do it myself

New approach. Let's attach a knock-off 16 channel logic analyzer to the USB-C pins. I'll need some way to attach the probes. A cursory search found no off-the-shelf solutions, but a custom PCB with simple pin headers should do nicely. I'm now designing that.

I'm done with the design, but have since found an off-the-shelf solution. The search term that got me there was "USB 3.1 Type-C Male Female Test PCB Board Adapter". Let's give that a try.
