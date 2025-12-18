# How to partially reconfigure a design using QDMA for Versal devices

## 1) Create and build design
Create a DFX design. Follow this [tutorial](https://adaptivesupport.amd.com/s/article/000034563?language=en_US) to add slave boot interface support. Available design tutorials can be found [here](https://github.com/Xilinx/Vivado-Design-Tutorials/tree/2025.2/Versal/DFX).


## 2) Change BIF file
When the project is built, the output PDI files can be found in `<project_root>/<project_name>.runs/impl_1`. In this directory, `<top_wrapper_name>.pdi` can be found.
Open this file and add this line `boot_device { pcie }`:
```
bitstream_master:
{
 id_code = 0x14d2f093
 extended_id_code = 0x01
 id = 0x2
 boot_device { pcie }
 image
 {
  name = pmc_subsys
  id = 0x1c000001
  partition
  {
   id = 0x01
   type = bootloader
   slr = 0
   file = gen_files/plm.elf
  }

``` 

## 3) Create PDI
In order to create the new PDI, from the modified bif file, run this command:
```bash
bootgen -arch versal -image <top_wrapper_name>.bif -w -o <new_pdi_name>.pdi 
```
`<new_pdi_name>` can be any name.

## 4) Program base design
Program the FPGA with the base design, via JTAG or dedicated software. Reboot the server or perform a PCIe hotplug operation.

## 5) Setup QDMA
In order to install QDMA software stack, follow [this](https://xilinx.github.io/dma_ip_drivers/master/QDMA/linux-kernel/html/tandem_boot_support.html) tutorial.

### Setup queues
Find your suited PCIe device that has the QDMA driver bound to:

```bash
lspci -vvd 10ee:
1d:00.0 Processing accelerators: Xilinx Corporation Device 50b4
	Subsystem: Xilinx Corporation Device 000e
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	NUMA node: 0
	Region 0: Memory at 380060000000 (64-bit, prefetchable) [disabled] [size=256M]
	Capabilities: <access denied>

<<<1d:00.1>>> Processing accelerators: Xilinx Corporation Device 50b5
	Subsystem: Xilinx Corporation Device 000e
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 32 bytes
	NUMA node: 0
	Region 0: Memory at 380070000000 (64-bit, prefetchable) [size=512K]
	Capabilities: <access denied>
	Kernel driver in use: qdma-pf
	Kernel modules: <<<qdma_pf>>>, qdma_vf

1d:00.2 RAM memory: Xilinx Corporation Device 50b6
	Subsystem: Xilinx Corporation Device 000e
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 32 bytes
	NUMA node: 0
	Region 0: Memory at b0000000 (64-bit, prefetchable) [size=4M]
	Capabilities: <access denied>
```

Create the queues based on the output. For this example `1d:00.1` is used.

```bash
echo 100 | sudo tee /sys/bus/pci/devices/0000\:1d\:00.1/qdma/qmax
dma-ctl qdma1d001 q add idx 0 mode mm dir h2c
dma-ctl qdma1d001 q add idx 1 mode mm dir c2h
dma-ctl qdma1d001 q start idx 0 dir h2c aperture_sz 4096
dma-ctl qdma1d001 q start idx 1 dir c2h aperture_sz 4096
```

## 6) Program
In order to program a partial bitstream, with the previously created queues, follow these steps:

### Identify programming image and size
```bash
ll | grep pdi
-rw-r--r--  1 root root   499056 Dec 17 23:25 top_i_rp1_rp1m2_inst_0_partial.pdi
-rw-r--r--  1 root root  6806256 Dec 17 23:21 top_wrapper.pdi
-rw-r--r--  1 root root  6806256 Dec 18 11:01 top_wrapperr.pdi
```

### Program the image 
```bash
sudo dma-to-device -d /dev/qdma1d001-MM-0 -f top_i_rp1_rp1m2_inst_0_partial.pdi -s 499056 -a 0x102100000
```
### Verify the bitstream

