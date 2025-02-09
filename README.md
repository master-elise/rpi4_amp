# Asymmetric MultiProcessing (AMP) on Raspberry Pi 4

## Compile Buildroot for Raspberry Pi 4:

```
wget https://buildroot.org/downloads/buildroot-2024.11.1.tar.gz
tar zxvf buildroot-2024.11.1.tar.gz
mv buildroot-2024.11.1 buildroot-2024.11.1_rpi4
cd buildroot-2024.11.1_rpi4
make raspberrypi4_64_defconfig
make
```
will create the toolchain and the basic tools needed to run GNU/Linux
on the Raspberry Pi4. Since we need U-Boot to launch FreeRTOS on one core,
still in the Buildroot directory:
```
make menuconfig
```
and go to Bootloaders -> U-Boot -> Board defconfig and fill with ``rpi_4``. Then
```
make
```
to compile U-Boot. Edit ``output/build/uboot-2024.10/.config`` and uncomment ``CONFIG_CMD_CACHE=y``
to set it to active. Recompile with ``make uboot`` to regenerate 
``output/build/uboot-2024.10/u-boot.bin``.

The Buildroot configuration we used is provided in <a href="buildroot-2024.11.1_rpi4_defconfig">
buildroot-2024.11.1_rpi4_defconfig</a>.

Configure the SD-card with ``sudo dd if=output/images/sdcard.img of=/dev/sdd bs=8M status=progress``
where ``/dev/sdd`` is replaced with the block device created when inserting the SD-card (identified
with ``sudo dmesg | tail``).

## Compile the baremetal toolchain and FreeRTOS for Raspberry Pi 4:

Following https://github.com/TImada/raspi4_freertos, compile FreeRTOS for the Raspberry Pi 4. 

First we need a cross-compilation toolchain for baremetal 64-bit ARM, which can be generated using 
Crosstool-NG:
```
git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng/
./bootstrap
./configure
make -j12
./ct-ng menuconfig
```
and select under Target Options: Target Architecture (arm) and Bitness (64-bit), then
```
./ct-ng build
export PATH=$HOME/x-tools/aarch64-unknown-elf/bin/:$PATH
```

Then ``git clone https://github.com/TImada/raspi4_freertos`` and
```
cd raspi4_freertos/Demo/CORTEX_A72_64-bit_Raspberrypi4/uart
make CROSS=aarch64-unknown-elf-
``` 

## Update the SD-card configuration to launch U-Boot

Mount the first partition of the SD-card we flashed earlier (``sudo mount /dev/sdd1 /mnt``) and
edit the content of ``config.txt`` to append with ``enable_uart=1`` and replace
``kernel=Image`` with ``kernel=u-boot.bin`` to tell the Raspberry Pi4 to launch U-Boot instead
of the Linux kernel. Copy ``output/build/uboot-2024.10/u-boot.bin`` from the Buildroot directory
to this SD-card partition, as well as ``raspi4_freertos/Demo/CORTEX_A72_64-bit_Raspberrypi4/uart/uart.elf``. Unmount the SD-card (``sudo umount /mnt``).

Launch the Raspberry Pi 4 fitted with this new SD-card: if all goes well the serial port (pins 
8 and 10) shoud display the U-Boot prompt:
```
U-Boot 2025.04-rc1-00063-g2b1c8d3b2da4-dirty (Feb 04 2025 - 06:45:40 +0000)

DRAM:  924 MiB (effective 3.8 GiB)
RPI 4 Model B (0xc03111)
Core:  213 devices, 17 uclasses, devicetree: board
MMC:   mmcnr@7e300000: 1, mmc@7e340000: 0
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1... 
In:    serial,usbkbd
Out:   serial,vidconsole
Err:   serial,vidconsole
Net:   eth0: ethernet@7d580000

PCIe BRCM: link up, 5.0 Gbps x1 (SSC)
starting USB...
Bus xhci_pci: Register 5000420 NbrPorts 5
Starting the controller
USB XHCI 1.00
scanning bus xhci_pci for devices... 2 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
Hit any key to stop autoboot:  0 
U-Boot> 
```
Run the following commands to launch the FreeRTOS demonstration:
```
setenv autostart yes
dcache off
fatload mmc 0:1 0x30000000 /uart.elf
dcache flush
bootelf 0x30000000
```
resulting, on the second serial port (pins 27 and 28) in the message

```
****************************                                                    
    FreeRTOS UART Sample                                                        
  (This sample uses UART2)                                                      
****************************                                                    
0000000000000000                                                                
00000000000003E9                                                                
00000000000007D2                                                                
0000000000000BBB                                                                
```

## Running GNU/Linux next to FreeRTOS

As opposed to https://github.com/TImada/raspi4_freertos stating to run ``dtc`` on the target, never
ever install development tools on the target but only cross-compile on the host.

```
export PATH=$HOME/buildroot-2024.11.1_rpi4/output/host/usr/bin:$PATH
```
to use Buildroot's ``dtc`` command and in ``raspi4_freertos/dts`` (make sure ``which dtc`` leads
to the Buildroot ``dtc``):
```
dtc -O dtb -I dts ./raspi4-rpmsg.dtso -o ./raspi4-rpmsg.dtbo
```
and ``sudo cp raspi4-rpmsg.dtbo /mnt/overlays`` assuming the SD-card first partition is accessible 
in ``/mnt``. 

Edit in this same partition ``config.txt`` and add at the end (after ``enable_uart=1``): 
```
dtoverlay=raspi4-rpmsg
```
and add in ``cmdline.txt`` the option
```
maxcpus=3
```
Loading and launching FreeRTOS is similar as above. Once FreeRTOS is running (as seen on the
second UART), in U-Boot [run](https://stackoverflow.com/questions/59580877/raspberry-pi-4-u-boot-on-booting-hanging-in-starting-kernel):
<img src="rpi4_FreeRTOS.jpg">

```
fatload mmc 0:1 ${kernel_addr_r} Image
fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb
setenv bootargs earlyprintk root=/dev/mmcblk0p2 rw rootwait maxcpus=3 console=ttyAMA0,115200
booti ${kernel_addr_r} - ${fdt_addr}
```
