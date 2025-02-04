Compile Buildroot for Raspberry Pi 4:
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

Configure the SD-card with ``sudo dd if=output/images/sdcard.img of=/dev/sdd bs=8M status=progress``
where ``/dev/sdd`` is replaced with the block device created when inserting the SD-card (identified
with ``sudo dmesg | tail``).

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
fatload mmc 0:1 0x28000000 /uart.elf
dcache flush
bootelf 0x28000000
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
