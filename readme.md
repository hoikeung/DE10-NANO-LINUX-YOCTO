### Introduction
To build embedded linux for DE10-NANO using Yocto with xfce and frame buffer in Quartus 21.1

### Step

#### Build environment preparation
1. Download toolchain	
```wget https://developer.arm.com/-/media/Files/downloads/gnu/11.2-2022.02/binrel/gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
tar xf gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
rm gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf.tar.xz
```

2. Setup toolchain path
export PATH=`pwd`/gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf/bin:$PATH
export CROSS_COMPILE=arm-none-linux-gnueabihf-
export ARCH=arm

#### Compile FPGA and SoC
3. copy components/clocked_video_output and components/frame_reader to Quartus's ip/altera folder
4. add the following above </library> to ip/altera/altera_components.ipx
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
5. compile qsys and the project using Quartus
6. convert generated .sof file to .rbf file
go to output_files folder
sudo ~/intelFPGA_lite/21.1/quartus/bin/quartus_cpf -c -o bitstream_compression=on --configuration_mode=FPP DE10_NANO_SOC_FB.sof soc_system.rbf

#### Build bootloader
7. go to the project folder
8. make folder for bootloader
mkdir -p software/bootloader
9. build hps setting
~/intelFPGA/20.1/embedded/embedded_command_shell.sh \
bsp-create-settings \
   --type spl \
   --bsp-dir software/bootloader \
   --preloader-settings-dir "hps_isw_handoff/soc_system_hps_0" \
   --settings software/bootloader/settings.bsp
10. go to bootloader folder and donwnload U-Boot
cd software/bootloader
git clone https://github.com/altera-opensource/u-boot-socfpga
cd u-boot-socfpga
11. open configs/socfpga_de10_nano_defconfig with text editor
12. change the following
CONFIG_USE_BOOTCOMMAND=y
add CONFIG_BOOTCOMMAND="run fatscript;bridge enable; run distro_bootcmd"
13. run the qts_filter
./arch/arm/mach-socfpga/qts-filter.sh cyclone5 ../../../ ../ ./board/altera/cyclone5-socdk/qts/
14. make
make socfpga_de10_nano_defconfig
make -j 48
