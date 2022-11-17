Material :
- KV260 board
- Digilent JTAG HS2 / HS3 
- Linux Ubuntu machine 

**PART 0 - INSTALL THE TOOLS**

As a first, high highly recommend to follow this 
https://github.com/tomverbeure/kv260_bringup

Some additional comments :
   * Download unified installer on Xilinx website : 2022.1 is working, I face some insolved issues in the next step of the process with 2022.2 version.
   
   * Run installer 
	- Install Vitis (this include VIVADO) and NOT vivado alone (I did the mistake, impossible to generate the design for license reason)
    	- Install PetaLinux
    	  	
   * Missing linux librairies
   	- sudo dpkg --add-architecture i386
	- sudo apt-get update
	- sudo apt-get install lib32stdc++6, libgtk2.0-0:i386, libfontconfig1:i386, libx11-6:i386, libxext6:i386, libxrender1:i386, libsm6:i386, libqtgui4:i386

   * Install Cable driver (JTAG) 
	cd ~/tools/Xilinx/Vitis/2021.1/data/xicom/cable_drivers/lin64/install_script/install_drivers
	sudo ./install_drivers
	

**PART I - Generate HW PLATFORM**



**PART II - STANDALONE APP**
At this stage we have generated hardware platform XSA

   * source VIVADO
   	source /tools/Xilinx/Vivado/2022.1/settings64.sh
   
   * open project generated at previous step
   

https://xilinx.github.io/kria-apps-docs/creating_applications/2021.1/build/html/docs/baremetal.html

**PART III - PETALINUX**

before starting, I recommend to play a bit with Petalinux folowing this : 
https://xilinx.github.io/kria-apps-docs/kv260/2020.2/build/html/docs/defect-detect/docs/app_deployment_dd.html

and dnf package manipulation tool :
https://docs.fedoraproject.org/en-US/quick-docs/dnf/


   * source Petalinux
   	source xxx/petalinux/settings64.sh

   * Generate project based on BSP
	petalinux-create -t project -s <path_to_BSP>

BSP in my case : xilinx-kv260-starterkit-v2022.1-05140151.bsp
Downloaded from Xilinx website
	
   * Go in the dir created by petalinux-create command
   	cd xilinx-kv260-starterkit-2022.1/
   	
   * Import previously generated hardware platform	
   	petalinux-config --get-hw-description xxx/kv260/platforms/vivado/kv260_ispMipiRx_rpiMipiRx_DP/project/kv260_ispMipiRx_rpiMipiRx_DP_wrapper-AXI-GPIO-3.xsa 
   	
   	
**Booting problem***
usefull link
userspace GPIO :
- https://support.xilinx.com/s/article/1141772?language=en_US
-https://easytp.cnam.fr/alexandre/index_fichiers/support/zynq_linux.pdf


Booting with BOOT.BIN generated either with:
petalinux-package --boot --fsbl images/linux/zynqmp_fsbl.elf --pmufw images/linux/pmufw.elf --fpga ./project-spec/hw-description/pl_AXI_GPIO.bit --u-boot --force
or 
petalinux-package --boot --u-boot --force
result in a read only root file system
Files on boot partition of sd card are :
    BOOT.BIN
    image.ub
    boot.scr

while booting without BOOT.BIN with the following files in the boot partition
    boot.scr
    Image
    ramdisk.cpio.gz.u-boot
    system.dtb 
    system-zynqmp-sck-kv-g-revB.dtb

But the question is WHERE IS THE PL bit file and how it can be loaded ?
alternatively generate SD card image with 
petalinux-package --wic --images-dir images/linux/ --bootfiles "ramdisk.cpio.gz.u-boot,boot.scr,Image,system.dtb,system-zynqmp-sck-kv-g-revB.dtb" --disk-name "mmcblk1"

But question remain the same where is PL bit file and how is it loaded?

Read only file system 
https://support.xilinx.com/s/question/0D52E00006iHsjtSAC/read-only-rootfs-is-causing-me-problems?language=en_US


**one partial solution here, to be loaded with FPGA Manager ???**
https://docs.xilinx.com/r/2021.1-English/ug1144-petalinux-tools-reference-guide/FPGA-Manager-Configuration-and-Usage-for-Zynq-7000-Devices-and-Zynq-UltraScale-MPSoC

0. In the petalinux-config command, select FPGA Manager > [*] Fpga Manager.

1. Add PL bit stream to petalinux (cd to petalinux folder)
petalinux-create -t apps --template fpgamanager_dtg -n <name_of_PL> -interface --srcuri <path-to-xsa>/system.xsa --enable

note : name of PL must be lower case (petalinux build complain after)
in my case I did 
petalinux-create -t apps --template fpgamanager_dtg -n basePL  --srcuri ../bitstream/AXI_GPIO_Wrapper.xsa --enable


2. Rebuild petalinux
petalinux-build

3. regenerate SD 

4. boot petalinux load P/L bitstream
bitstream will be locateds in /lib/firmware/xilinx/basePL/
load it with fpgautil -b /lib/firmware/xilinx/basePL/basePL.bit.bin
Note : /
- loading dtg does not work 
fpgautil -o /lib/firmware/xilinx/base/pl.dtbo -b /lib/firmware/xilinx/base/design_1_wrapper.bit.bin
*Nov  6 10:35:11 xilinx-kv260-starterkit-20221 kernel: fpga_region region0: Region already has overlay applied.*


5. Test the I/O
In my case AXI GPIO address in Vivado address map is 0x00A0000000
sudo devmem 0x00A0000000 8 1 --> Turn the I.O On

This a good first step FPGA can be access through low level call but are not recognize by linux
Additional  possibility to prepare a bit.bin file base on bitstream without rebuilding petalinux

1. prepare a bif file ex bitstream.bif
all:
{
        design_1_wrapper.bit /* Bitstream file name */
}

2. bootgen -image bitstream.bif -arch zynqmp -process_bitstream bin

generate design_1_wrapper.bit.bin loadable with fpga-util -b directly


**another one : base on previous one**

1. install xrt in petalinux
sudo dnf install xrt

2. go to lib/firmware/base (base = name given @step 1 above)

3. add shell.json file containing

{
    "shell_type" : "XRT_FLAT",
    "num_slots": "1"
}

4. reboot

5. xmutil listapps
app name (base in my case)  shall appear

6. xmutil unloadapp

7. xmutil loadapp <app_name>


**testing**
Now working with linux !

1. find gpio slot used by axi 
	go to /sys/devices/platform/axi/<address of AXI GPIO in vivado>.gpio/gpio/ 
	--> in my case : /sys/devices/platform/axi/a0000000.gpio/gpio/
	ls and you will find the GPIO slot number
	--> in my case gpiochip501
	
2. cd to /sys/class/gpio
	echo 501 > export
		--> this operation create a folder named gpio501
	cd export
	echo out > direction
		--> set as output
	echo 1 > value
		--> IO on
	cd ..
	echo 501 > unexport
		--> close the I/O
		
but it manage only the first bit 
to work with second bit increment by 1 
eg . echo 502 > export etc.
		
 
ok GPIO working

**PART IV - VIDEO MIPI **

Download Xilinx KV260 reference platforms 
   * Clone the repo : 
	git clone --recursive https://github.com/Xilinx/kria-vitis-platforms.git

   * Go to vivado source : xxx/kria-vitis-platforms/kv260/platforms/vivado/kv260_ispMipiRx_rpiMipiRx_DP
   
   * source VIVADO
   	source /tools/Xilinx/Vivado/2022.1/settings64.sh
   	
   * make platform 
   	make xsa

This will take a long time (1h) in my case. It may crash depending on the number of core versus quantity of RAM, more core need more RAM.


**testing Video capture app**

same process 
	- xmutil loadapp video
Error : 
xilinx-video: probe of axi:vcap_capture_pipeline_mipi_csi2_rx_subsyst_0 failed with error -22
https://githublab.com/repository/issues/Xilinx/kv260-firmware/2

problem in dtsi / dtbo files automatically generated.

Assumption : dtsi has to be manually tuned --> but how ???
Here the process to manually generate it 
https://xilinx.github.io/kria-apps-docs/creating_applications/2021.1/build/html/docs/dtsi_dtbo_generation.html


https://xilinx.github.io/kria-apps-docs/creating_applications/2022.1/build/html/docs/vitis_accel_flow.html

 

Applications and their corresponding platforms are listed in the table below
 

Developers can download the .dtsi file from Kria apps firmware and compile them using command below

https://github.com/Xilinx/kria-apps-firmware/blob/xlnx_rel_v2022.1/boards/kv260/smartcam/kv260-smartcam.dtsi

to load with no error 
- use dtsi here : https://github.com/Xilinx/kria-apps-firmware/blob/xlnx_rel_v2022.1/boards/kv260/smartcam/kv260-smartcam.dtsi
- compile it : dtc -@ -O dtb -o pl.dtbo pl.dtsi
- put firmware in /lib/firmware : ap1302_ar1335_single_fw.bin from here : https://github.com/Xilinx/ap1302-firmware/blob/release-2021.1/ap1302_ar1335_single_fw.bin

dmesg shows no error, /dev/video0 appears 
but :
v4l2-ctl --device /dev/video0 --stream-mmap --stream-to=frame.raw --stream-count=1
fails with error :
	VIDIOC_REQBUFS returned -1 (Invalid argument)





