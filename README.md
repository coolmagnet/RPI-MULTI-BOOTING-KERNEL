# RASPBERRY PI | MULTI-BOOTING KERNEL | PATCH / BUILD / INSTALL

Instructions on how to patch, build (compile), and install a custom kernel for Raspberry Pi<br><br>
<b>Disclaimer:  This technical guide is provided without warranty of any kind. The user assumes all responsibility and risk for its use. The provider shall not be liable for any damages arising from the use of this guide.</b><br>
<br>
Reference:  https://www.raspberrypi.com/documentation/computers/linux_kernel.html#kernel<br><br>
<b>2-Step Process:</b>
1. <b>[Cross-Compile]</b> Perform a cross-compile on a x86_64 Debian VM which will be faster than compiling on Raspberry Pi hardware.
2. <b>[Multi-Kernel Booting]</b> Install the custom kernel to Raspberry Pi OS without overwriting the stock kernel.
<hr>
<b>On a x86_64 Debian VM</b><br>

1. Update and install cross-compile tools

```bash
sudo apt update
sudo apt install -y git bc bison flex libssl-dev make \
    crossbuild-essential-arm64
```

2. Get the Raspberry Pi kernel sources
<pre>
git clone --depth=1 https://github.com/raspberrypi/linux.git -b [branch]

Example:
git clone --depth=1 https://github.com/raspberrypi/linux.git -b rpi-6.12.y
cd linux
</pre>

3. Configure for Raspberry Pi 4 (creates .config)<br>

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```
4. Apply patch, make menuconfig, and/or manually change .config
<ul>
	<li><code>make menuconfig</code> requires <code>sudo apt install libncurses5-dev</code></li>
	<li>To apply a patch:  <code>cd linux</code> and then <code>patch -p1 < /path/to/patchfile</code></li>
	<li>Give the custom kernel a descriptive name:
		<pre>Note: A plus sign (+) indicates the kernel source has been modified.
		When the kernel source has been downloaded (via git) it is considered unmodified.
		Changes to any files inside the kernel source is now considered modified.
		The plus sign (+) is added automatically during 'make'</pre>
		Example:  <code>CONFIG_LOCALVERSION="-v8"</code>     will be      <code>6.12.81-v8+</code><br>
		Example:  <code>CONFIG_LOCALVERSION="-broadcom-v8"</code>     will be      <code>6.12.81-broadcom-v8+</code><br></li>
</ul>

5. Build the kernel
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```
6. Install modules and prepare files to copy to the Raspberry Pi
<ul>
	<li><code>mkdir -p ../modules</code></li>
	<li><code>make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_install INSTALL_MOD_PATH=../modules</code></li>
</ul>

7. Tarball the custom kernel
<pre>
TARBALL='/path/to/custom-kernel.tar'; \
	tar --dereference -cvpf "$TARBALL" arch/arm64/boot/dts/{broadcom,overlays}/*.dtb* && \
	tar -rvpf "$TARBALL" arch/arm64/boot/Image* ../modules arch/arm64/boot/dts/overlays/README

Example:
TARBALL='/root/6.12.81-broadcom.tar'; \
	tar --dereference -cvpf "$TARBALL" arch/arm64/boot/dts/{broadcom,overlays}/*.dtb* && \
	tar -rvpf "$TARBALL" arch/arm64/boot/Image* ../modules arch/arm64/boot/dts/overlays/README
</pre>

8. Transfer the tarball to a Raspberry Pi

<hr>
<b>On a Raspberry Pi</b><br>
Isolate custom kernel and all its boot files into a dedicated directory.

1. Extract tarball to a directory
2. <code>mkdir -p /boot/firmware/custom /boot/firmware/custom/overlays</code>
3. For 64-bit Kernel:
```bash
sudo cp -v arch/arm64/boot/Image.gz /boot/firmware/custom/kernel8.img
sudo cp -v arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/custom/
sudo cp -v arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/custom/overlays/
sudo cp -v arch/arm64/boot/dts/overlays/README /boot/firmware/custom/overlays/
```

4. Copy modules to /lib/modules

<ul>
	<li>Get the directory name of the compiled kernel</li>
		<code>ls modules/lib/modules</code><br></li>
	<li>Copy the modules over to <code>/lib/modules</code><br>
		<code>cp -vR modules/lib/modules/[kernel_directory_name] /lib/modules</code><br>
		<code>cp -vR modules/lib/modules/6.12.81-broadcom-v8+ /lib/modules</code><br></li>
</ul>

5. Mult-Kernel Booting<br>
```bash
echo 'os_prefix=custom/' >> /boot/firmware/config.txt
```

6. reboot

<hr>

Note1: To boot back to the Raspberry Pi stock kernel comment or remove <code>'os_prefix=custom/'</code> from <code>/boot/firmware/config.txt</code><br>
Note2: To perform a Raspberry Pi system update ie. <code>apt full-upgrade</code> , you must boot into the stock kernel then run the update.<br>
Note3: To mitigate "bcmgenet timeout" warnings for Broadcom BCM54213PE Gigabit Ethernet these workarounds may help:<br>
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Disable pause frames / flow control: <code>ethtool -A eth0 autoneg off rx off tx off</code><br>
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;To check settings: <code>ethtool -a eth0</code><br>
Note4: To completely remove the custom kernel and go back to the stock kernel:<br>

<b>If you are running the stock kernel:</b>
<ul>
	<li>Comment or remove <code>'os_prefix=custom/'</code> from <code>/boot/firmware/config.txt</code></li>
	<li>Delete the entire custom directory:  <code>/boot/firmware/custom</code></li>
	<li>Delete the entire custom kernel modules directory:  <code>/lib/modules/[kernel_directory_name]</code></li>
</ul>

<b>If you are running the custom kernel:</b>
<ul>
	<li>Comment or remove <code>'os_prefix=custom/'</code> from <code>/boot/firmware/config.txt</code></li>
	<li>Reboot into the stock kernel</li>
	<li>Delete the entire custom directory:  <code>/boot/firmware/custom</code></li>
	<li>Delete the entire custom kernel modules directory:  <code>/lib/modules/[kernel_directory_name]</code></li>
</ul>

<hr>

<i>Modified <code>bcmgenet-jc-mod.patch</code> for Kernel 6.12</i> = https://lore.kernel.org/all/20260325173602.3676778-1-justin.chen@broadcom.com/<br>
<i><code>PATCH-net-v3-net-bcmgenet-fix</code> for Kernel 6.18.24</i>
