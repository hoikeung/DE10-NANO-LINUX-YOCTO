### Introduction
To build embedded linux for DE10-NANO using Yocto with xfce and frame buffer in Quartus 21.1  
Requirement:  
Quartus 21.1	  
Intel EDS 20.1	  
DE10_NANO_SoC_FB	  

### Step

#### Build environment preparation
1. Download toolchain	
```
wget https://developer.arm.com/-/media/Files/downloads/gnu/11.2-2022.02/binrel/gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
tar xf gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
rm gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
```

2. Setup toolchain path
```
export PATH=`pwd`/gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf/bin:$PATH
export CROSS_COMPILE=arm-none-linux-gnueabihf-
export ARCH=arm
```

#### Compile FPGA and SoC
3. copy components/clocked_video_output and components/frame_reader to Quartus's ip/altera folder
4. add the following above </library> to ip/altera/altera_components.ipx
```
  <component
   name="alt_vip_vfr"
   file="frame_reader/full_ip/frame_reader/alt_vip_vfr_hw.tcl"
   displayName="Frame Reader"
   version="14.0"
   description="The Frame Reader Megacore can be used to read a video stream from video frames stored a memory buffer"
   tags="AUTHORSHIP=Altera Corporation /// CONNECTION_TYPES=avalon,clock,interrupt"
   categories="Video and Image Processing"
   factory="TclModuleFactory">
  <tag2 key="COMPONENT_EDITABLE" value="false" />
  <tag2 key="COMPONENT_HIDE_FROM_QUARTUS" value="true" />
  <tag2 key="ELABORATION_CALLBACK" value="vfr_elaboration_callback" />
  <tag2 key="TCL_PACKAGE_VERSION" value="10.0" />
  <documentUrl
     displayName="DATASHEET"
     type="DATASHEET"
     url="http://www.altera.com/literature/ug/ug_vip.pdf" />
 </component>
 <component
   name="alt_vip_itc"
   file="clocked_video_output/alt_vip_itc_hw.tcl"
   displayName="Clocked Video Output Intel FPGA IP"
   version="14.0"
   description="The Clocked Video Output converts Avalon-ST Video to standard video formats such as BT656 or VGA."
   tags="AUTHORSHIP=Intel Corporation /// CONNECTION_TYPES=clock"
   categories="DSP/Video and Image Processing/Legacy"
   factory="TclModuleFactory">
  <tag2 key="COMPONENT_EDITABLE" value="false" />
  <tag2 key="COMPONENT_HIDE_FROM_QUARTUS" value="true" />
  <tag2 key="ELABORATION_CALLBACK" value="cvo_elaboration_callback" />
  <tag2 key="TCL_PACKAGE_VERSION" value="11.0" />
  <documentUrl
     displayName="DATASHEET"
     type="DATASHEET"
     url="http://www.altera.com/literature/ug/ug_vip.pdf" />
 </component>
 <plugin
   name="alt_vip_itc.qprs"
   file="clocked_video_output/alt_vip_itc.qprs"
   displayName="alt_vip_itc.qprs"
   version="0.0"
   description=""
   tags=""
   categories=""
   type="com.altera.sopcmodel.util.IElementPresetList"
   subtype=""
   factory="PresetFactory">
  <tag2 key="PRESET_TYPE" value="alt_vip_itc" />
 </plugin>
```
5. compile qsys and the project using Quartus
6. go to output_files folder, convert generated .sof file to .rbf file
```
sudo ~/intelFPGA_lite/21.1/quartus/bin/quartus_cpf -c -o bitstream_compression=on --configuration_mode=FPP DE10_NANO_SOC_FB.sof soc_system.rbf
```

#### Build bootloader
7. go to the project folder
8. make folder for bootloader
```
mkdir -p software/bootloader
```
9. build hps setting
```
~/intelFPGA/20.1/embedded/embedded_command_shell.sh \
bsp-create-settings \
   --type spl \
   --bsp-dir software/bootloader \
   --preloader-settings-dir "hps_isw_handoff/soc_system_hps_0" \
   --settings software/bootloader/settings.bsp
```
10. go to bootloader folder and donwnload U-Boot
```
cd software/bootloader
git clone https://github.com/altera-opensource/u-boot-socfpga
cd u-boot-socfpga
```
11. open configs/socfpga_de10_nano_defconfig with text editor
12. change the following
```
CONFIG_USE_BOOTCOMMAND=y
add CONFIG_BOOTCOMMAND="run fatscript;bridge enable; run distro_bootcmd"
```
13. run the qts_filter
```
./arch/arm/mach-socfpga/qts-filter.sh cyclone5 ../../../ ../ ./board/altera/cyclone5-socdk/qts/
```
14. compile u-boot
```
make socfpga_de10_nano_defconfig
make -j 48
```

#### Build Linux kernel
15. clone the kernel
```
git clone https://github.com/altera-opensource/linux-socfpga
```
16. go to linux-socfpga/arch/arm/boot/dts/ folder, add the follow to socfpga_cyclone5_de0_nano_soc.dts
```
&base_fpga_region {
        ranges =  <0x00000000 0xff200000 0x00200000>;

        alt_vip_vfr_hdmi: vip@0x100031000 {
        compatible = "ALTR,vip-frame-reader-14.0", "ALTR,vip-frame-reader-9.1";
        reg = <0x00031000 0x00000080>;
        max-width = <1024>;
        max-height = <768>;
        bits-per-color = <8>;
        colors-per-beat = <4>;
        beats-per-pixel = <1>;
        mem-word-width = <128>;
    };
};
```
17. copy components/altvipfb.c to linux-socfpga/drivers/video/fbdev/
18. edit altvipfb.c line 220
```
- fbmem_virt = dma_alloc_coherent(NULL,
+ fbmem_virt = dma_alloc_coherent(&pdev->dev,
```
19. add the following to altvipfb.c line 177
```
info->var.pixclock = 6734;
info->var.left_margin = 148;
info->var.right_margin = 88;
info->var.upper_margin = 36;
info->var.lower_margin = 4;
info->var.hsync_len = 44;
info->var.vsync_len = 5;
```
20. add the following to line 14 of linux-socfpga/drivers/video/fbdev/Makefile
```
obj-$(CONFIG_FB_ALTERA_VIP_FB) += altvipfb.o
altvipfb_drv-objs := altvipfb.o
```
21. add the following to line 222 of linux-socfpga/drivers/video/fbdev/Kconfig
```
config FB_ALTERA_VIP_FB
    tristate "Altera VIP Frame Buffer framebuffer support"
    depends on FB
    select FB_CFB_FILLRECT
    select FB_CFB_COPYAREA
    select FB_CFB_IMAGEBLIT
    help
      This driver supports the Altera Video and Image Processing(VIP)
      Frame Buffer. This core driver only supports Arria 10 HW and newer
      families of FPGA
```
22. add the following to line 121 of linux-socfpga/arch/arm/configs/socfpga_defconfig
```
CONFIG_FB_ALTERA_VIP_FB=y
```
23. compile the kernel at linux-socfpga folder
```
make socfpga_defconfig
make -j 48 zImage Image dtbs modules
make -j 48 modules_install INSTALL_MOD_PATH=modules_install
rm -rf modules_install/lib/modules/*/build
rm -rf modules_install/lib/modules/*/source
```

#### Build Yocto
24. make a folder for Yocto
```
mkdir yocto && cd yocto
```
25. clone the files system
```
git clone -b honister https://git.yoctoproject.org/poky
git clone -b honister https://git.yoctoproject.org/meta-intel-fpga
git clone -b honister https://git.openembedded.org/meta-openembedded
git clone -b honister https://github.com/altera-opensource/meta-intel-fpga-refdes
```
26. change build version of LAYERSERIES_COMPAT_meta-intel-fpga-refdes at meta-intel-fpga-refdes/conf/layer.conf
```
LAYERSERIES_COMPAT_meta-intel-fpga-refdes = "master honister"
```
27. initialize the build environment
```
source poky/oe-init-build-env ./build
```
28. set build machine
```
source poky/oe-init-build-env ./build
```
29. Use systemd
```
echo 'DISTRO_FEATURES_append = " systemd"' >> conf/local.conf
echo 'VIRTUAL-RUNTIME_init_manager = "systemd"' >> conf/local.conf
```
30. Build additional rootfs type
```
echo 'IMAGE_FSTYPES += "tar.gz"' >> conf/local.conf
```
31. Ensure we build in all kernel-modules
```
echo "MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += \"kernel-modules\"" >> conf/local.conf
```
32. bblayer setting
```
echo 'BBLAYERS += " ${TOPDIR}/../meta-intel-fpga "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-intel-fpga-refdes "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-oe "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-networking "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-python "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-xfce "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-gnome "' >> conf/bblayers.conf
echo 'BBLAYERS += " ${TOPDIR}/../meta-openembedded/meta-multimedia "' >> conf/bblayers.conf
```
33. build rootfs image
```
bitbake  core-image-minimal-xfce
```

#### Make SD card image
34. make a folder for sd card image
```
mkdir sd_card && cd sd_card
```
45. download make_sdimage_p3.py
```
wget https://releases.rocketboards.org/release/2020.05/gsrd/tools/make_sdimage_p3.py
chmod +x make_sdimage_p3.py
```
46. Copy u-boot file to sd_card folder
```
cp ~/de10-nano-yocto-xfce/DE10_NANO_SoC_FB/software/bootloader/u-boot-socfpga/u-boot-with-spl.sfp .
```
47. copy linux image to sd card folder
```
mkdir fat && cd fat
cp ~/de10-nano-yocto-xfce/linux-socfpga/arch/arm/boot/zImage .
cp ~/de10-nano-yocto-xfce/linux-socfpga/arch/arm/boot/dts/socfpga_cyclone5_de0_nano_soc.dtb ./socfpga_cyclone5_de10_nano.dtb
cp ~/de10-nano-yocto-xfce/DE10_NANO_SoC_FB/output_files/soc_system.rbf .
cp ~/de10-nano-yocto-xfce/components/u-boot.scr .
cp -r ~/de10-nano-yocto-xfce/components/extlinux .
```
48. copy kernel and yocto to sdcard folder
```
mkdir rootfs && cd rootfs
sudo tar xf ~/de10-nano-yocto-xfce/yocto/build/tmp/deploy/images/cyclone5/core-image-minimal-xfce-cyclone5.tar.gz
sudo rm -rf lib/modules/*
sudo cp -r ~/de10-nano-yocto-xfce/linux-socfpga/modules_install/lib/modules/* lib/modules/
```
49. build SD card image
```
sudo python3 make_sdimage_p3.py -f \
-P u-boot-with-spl.sfp,num=3,format=raw,size=10M,type=A2 \
-P rootfs/*,num=2,format=ext2,size=1500M \
-P fat/*,num=1,format=fat32,size=500M -s 2G \
-n sdcard.img
```
50. insert SD card and burn SD card image
```
sudo umount /dev/sdc*
sudo dd  bs=4M  conv=sync  if=sdcard.img  of=/dev/sdc
```
51. done

