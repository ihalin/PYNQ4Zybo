# PYNQ4Zybo
Instructions and packages for Zybo compatibility to PYNQ

PYNQ repository was not targeted for the Zybo board. However, since the Zybo board has a Zynq device on it, PYNQ can be ported unto it. The following steps were applied successfully on PYNQ Z1 V2.0 image.


## Precompiled Image

The first step is to  <a href="https://files.digilent.com/Products/PYNQ/pynq_z1_v2.0.img.zip" target="_blank">download the precompiled image</a> and write the image to a micro SD card. This image is targeted for PYNQ Z1 board. Its BOOT partition includes the following files:

BOOT.bin        -   Binary boot file, which includes the Zynq bitstream, FirstStageBootLoader and U-boot

uImage          -   Kernel image file

devicetree.dtb  -   device tree blob

These files were compiled for  the PYNQ-Z1 and have to be replaced with files that are compiled for the Zybo board. The 2nd partition which includes the linux root file system (and the PYNQ package), can remain as is, beside a package suggested below for dma access from python.

## Compiling u-boot and linux kernel for Zybo

The kernel files below are compiled on Ubuntu v16.04.1, installed on VM VirtualBox version 5.2.
The linux and u-boot repository version is <a href="https://www.xilinx.com/support/answers/68370.html" target="_blank">2016.4</a>.
Prior to compiling, the environment should be setup according to the steps recommended on the <a href="https://PYNQ.readthedocs.io/en/v2.0/PYNQ_sd_card.html" target="_blank">PYNQ</a> help page:

  1. Install Vivado 2016.1 and Xilinx SDK 2016.1
  2. Install dependencies using the following script from <a href="https://github.com/Xilinx/PYNQ/tree/v2.0" target="_blank">PYNQ</a> repository:
```
   <PYNQ repository>/sdbuild/scripts/setup_host.sh
```
  3. Source the appropriate settings files from Vivado and Xilinx SDK - add to ~/.bashrc:
```
source /opt/Xilinx/SDK/2016.4/settings64.sh
source /opt/Xilinx/Vivado/2016.4/settings64.sh
export CROSS_COMPILE=arm-xilinx-linux-gnueabi-
```
The boot file compilation are based on steps 1 - 3 mentioned in this <a href="https://superuser.blog/PYNQ-linux-on-zedboard/" target="_blank">Zeb board</a> PYNQ porting guide, with these modifications:
### Step 1 modifications - modify Zybo u-boot make command:
```
make zynq_zybo_config
make
```
### Step 3 modifications - Zybo devicetree blob :
The file to be edited is <a href="https://github.com/altuSemi/PYNQ4Zybo/blob/master/zynq-zybo.dts" target="_blank">/linux-xlnx/arch/arm/boot/dts/zynq-zybo.dts</a>.
On top of that, the dma package mentioned below is using 3 generic-uio which should be defined in <a href="https://github.com/altuSemi/PYNQ4Zybo/blob/master/zynq-7000.dtsi" target="_blank">/linux-xlnx/arch/arm/boot/dts/zynq-7000.dtsi</a>. The following should be added under 'amba':
```
amba: amba {
		u-boot,dm-pre-reloc;
		compatible = "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		interrupt-parent = <&intc>;
		ranges;
		axi_dma: axi-dma@40400000 {
			compatible = "generic-uio";
			interrupt-parent = <&intc>;
			interrupts = <0 29 4>;
			reg = <0x40400000 0x10000>;
		};
		dma_src@010000000 {
			compatible = "generic-uio";
			reg = <0x01000000 0x01000000>;
			interrupt-parent = <&intc>;
			interrupts = <0 30 4>;
		};

		dma_dst@020000000 {
			compatible = "generic-uio";
			reg = <0x02000000 0x01000000>;
			interrupt-parent = <&intc>;
			interrupts = <0 31 4>;
		};
```
## Booting PYNQ on Zybo
Next step is to copy the boot files over the original files in the PYNQ sdcard BOOT partition. 
The pre-compiled Zybo boot files were <a href="https://github.com/altuSemi/PYNQ4Zybo/tree/master/kernel" target="_blank"> committed</a> to this repository.

Then place the sd card in the Zybo sd card slot, set the boot jumper to boot from sd-card, and turn on the board.
The Zybo should boot and load the Linux kernel. The board can be accessed via UART, remote login or the Jupyter notebook portal as described in the <a href="https://PYNQ.readthedocs.io/en/v2.0/getting_started.html" target="_blank"> PYNQ documentation page</a>.

## Overlays
This PYNQ4Zybo porting guide was verified with three overlay designs:
### 1. Adder overlay</a>.
This is a simple overlay which implements an adder in the PL as described in <a href="https://PYNQ.readthedocs.io/en/v2.0/overlay_design_methodology/overlay_tutorial.html" target="_blank"> PYNQ documentation page</a> . Also explained in this <a href="https://www.youtube.com/watch?v=Dupyek4NUoI" target="_blank"> video-guide</a>. 
PYNQ4Zybo Jupyter notebook can be found <a href="https://github.com/altuSemi/PYNQ4Zybo/blob/master/jupyter_notebooks/AdderOverlay.ipynb" target="_blank"> here</a>.

### 2. DMA overlay
This overlay implements a AXI stream dma that transfers data from one address in the memory to another. It is based on the following reference:

Lauri's Blog - AXI Direct Memory Access : 	https://lauri.xn--vsandi-pxa.com/hdl/zynq/xilinx-dma.html

The <a href="https://github.com/altuSemi/PYNQ4Zybo/tree/master/overlays/dma" target="_blank"> dma overlay</a> was designed in Vivado 2016.3.
A python c extension was designed to pass data from python to the PL and back via the AXI dma, based on the c code in the above blog. This extension is currently supporting a max buffer length of 16K word (unit32).

The package is installed by copying the <a href="https://github.com/altuSemi/PYNQ4Zybo/tree/master/dma" target="_blank">dma</a> directory to the board (via the samba server), and executing inside the dma directory:
```
pip3.6 install . 
```
The code of the dma package is based on Lauri's Blog above, as well as the following references:

Returning a numphy array from c: 		http://acooke.org/cute/ExampleCod0.html

Enhancing Python with Custom C Extensions:	https://stackabuse.com/enhancing-python-with-custom-c-extensions/

The dma overlay can be tested with the following <a href="https://busybox.net/about.html" target="_blank">busybox</a> <a href=https://github.com/altuSemi/PYNQ4Zybo/blob/master/dma/busybox.sh target="_blank">script</a>, or with this <a href="https://github.com/altuSemi/PYNQ4Zybo/blob/master/jupyter_notebooks/dma.ipynb" target="_blank">PYNQ4Zybo jupyter notebook</a>.

<a href=http://fpga.org/2013/05/28/how-to-design-and-access-a-memory-mapped-device-part-two/ target="_blank">generic-uio</a> was used instead of devmem in the c code of the dma package to overcome non-root permissions issue.

### 3. FIR overlay
This overlay accelerates a <a href="https://www.youtube.com/watch?v=LoLCtSzj9BU" target="_blank">low pass filter function</a> with PL logic and AXI dma.
The original PYNQ <a href="https://github.com/Xilinx/PYNQ/tree/v2.0/sdbuild/packages/libsds" target="_blank"> libsds</a> function for contiguous memory allocation and dma access <a href="https://groups.google.com/forum/?utm_medium=email&utm_source=footer#!msg/PYNQ_project/rvez-UpGODY/oN9FusK3BQAJfailed" target="_blank">failed </a> to work following this porting guide. Instead the dma package above was used.

The PYNQ4Zybo <a href="https://github.com/altuSemi/PYNQ4Zybo/blob/master/jupyter_notebooks/FIR%20accelerator.ipynb" target="_blank"> FIR accelerator notebook </a> is using the dma package to perform the FIR calculation acceleration using the PL DSP blocks, and achieves a X14 acceleration ratio with respect to software based FIR.
	




Enjoy!	


