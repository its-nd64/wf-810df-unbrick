# wf-810df-unbrick

i accidentally bricked 2 of my fpt ax3000cv2 router by the following:  
- first is from a random corruption(probally around SPL since no uart at all)  
- second is when i tried to recover first, desoldered nand live while on u boot but everytime solder melt, uart get spammed with garbage  
  then i tried it like 3 times then i got this:  
```
Format: Log Type - Time(microsec) - Message - Optional Info

Log Type:
B - Since Boot(Power On Reset)
D - Delta
S - Statistic

S - QC_IMAGE_VERSION_STRING=BOOT.BF.3.3.1.1-00075
S - IMAGE_VARIANT_STRING=MAACANAZA
S - OEM_IMAGE_VERSION_STRING=CRM
S - Boot Config, 0x000002c5

B - 127 - PBL, Start
B - 1560 - bootable_media_detect_entry, Start
B - 3654 - bootable_media_detect_success, Start
B - 3658 - elf_loader_entry, Start
B - 8799 - auth_hash_seg_entry, Stabt
B - 9159 - auth_hash_seg_exit, Start
B - 102463 - elf_segs_hash_verify_entry, Start
B - 172136 - PBL, End

B - 141611 - SBL1, Start
B - 203008 - GCC [RstSt!t:0x0, RstDbg:0x600000] WDog Stat : 0x4

B - 211121 - clock_init, Start
D - 7564 - clock_i.it, Delta

B - 218837 - boot_flash_init, Start
D - 71156 - boot_flash_init, Delta

B - 290055 - boot_config_data_table_init, Start
D - 4697 - boot_config_data_table_init, Delta - (575 Bytes)

B - 297802 - Boot Setting : 0x00020618
B - 304176 - CDT version:2,Platform ID:8,Major ID:4,Minor ID:2,Subtype:2

B - 310917 - sbl1_ddr_set_params, Start
B - 312350 - Pre_DDR_clock_init, Start
B - 318176 - Pre_DDR_clock_init, End

B - 960414 - do ddr sanity test, Start
D - 30 - do ddr sanity test, Delta

B - 965081 - Image Load, Start

B - 1068201 - Error code 1000028 at boot_flash_trans_nand.c Line 422
```

i figured that spl and nand were fine since it can read from nand obviously but i couldnt do anything then since u boot is def bricked  
only choice left is to get a ch341a to flash it  
unfortunately the first ch341a killed itself in like 3m after i got it, something shorted internally! i didnt do anything other than plugging it in wth  
second ch341a worked perfectly, i thought ch341a with snander only use normal spi mode so i only put dividers for MOSI and CLK(the nand is 1v8)  
turn out that didnt work for whatever reason. nand spewed out good bytes but somehow in random position, didnt take a screenshot to demonstrate tho  
so i ordered a 1v8 adapter, took long enough
finally ready for some flashing actions, i dumped the first 16mb of the dead u boot one, then wrote both with backup, only 64mb then  
it didnt work for whatever reason, then i tried various stuffs, reread to check, nothing works  
2d later, after like 25 nand transfer from/to the board cycles, i decided to take a look at the bin  
looked fine, after 6 more flashes, i noticed that some have minor bit flips, figured that they are fine
i got so mad i read like half the datasheet, almost reverse engineered the board to access usb and get edl/jtag
then it(free tier ahh claude) hit me, i thought: how oob work? where tf is it  
read again with -d flag, ohhhh, the backups(from u boot) only contain data, no oob  
how do i add it tho?  then i found qcom-nandc-pagify that do exactly that  
then i compaired it with the 16mb dump from dead u boot nand, its suprisingly similar! the ecc matched perfectly  
it still didnt work, no uart at all  
claude messed with me, wasted me like 3 goddamn hours, i ommited it from that point on  
i didnt sleep well that day...  
next day, im on the verge of giving up, 5 hours of random shits later, no progress at all  
so i did the only thing left: check nand communication. used rp2040 as logic analyzer to probe lines  
after lots of frustration to get it work and produce sensible stuffs, i noticed that MISO didnt budge at all, always low, occure to both my boards and nands  
weird, that day was also wasted, not good  
then i took a closer look at what the host send  
clk is 5 or 6.25mhz idk idc, host send FF(reset) 0FC000(get features) 1FA000(set features) 13000000(page read 0x0??) 0FC000(get features)  
then a bigass block, a transfer probally: 6B 00 00 FF 80 00 00 00 00...  
im lazy to continue ranting so yeah, thats the end for the logic analyzer stuffs  
just know that maybe the chip was quiet is intentional, transfer block is prob in qspi mode(dr the switch but its there)  
that literally provided almost no info, sucks  
then i decided to redo everything, made new bins, check it very thoughly  
still didnt work!!!!!  
hm, lets readback and confirm  
lots of bit flips in the nand i thought was better(should have get this since the router came boot looping):  
```
Comparing files chk.bin and WORK2\16M_P.BIN
000030E9: 10 00
00006966: 3F 2F
0002B190: 20 00
001AC11F: 01 00
001AEDF9: 03 01
001FF89B: DB D3
0021530F: 99 91
002CF793: BA AA
002E1DC3: AB AA
00304FA7: F0 D0
0030891B: F8 B8
00309C79: 6D 69
00330D98: 02 00
```
at 0x30E9 and 0x6966 too! thats on spl region which is terrible  
swapped for another chip(the first router), its looking way better:  
```
Comparing files mx_chk.bin and WORK2\16M_P.BIN
0002C494: 20 24
001AA8C4: 44 54
```
much further!!!  
soldered it back and, it works? damn finally!!!!!!!!!!!!!!!!!!!!!  
```
Format: Log Type - Time(microsec) - Message - Optional Info
Log Type: B - Since Boot(Power On Reset),  D - Delta,  S - Statistic
S - QC_IMAGE_VERSION_STRING=BOOT.BF.3.3.1.1-00075
S - IMAGE_VARIANT_STRING=MAACANAZA
S - OEM_IMAGE_VERSION_STRING=CRM
S - Boot Config, 0x000002c5
B -       127 - PBL, Start
B -      1560 - bootable_media_detect_entry, Start
B -      3845 - bootable_media_detect_success, Start
B -      3848 - elf_loader_entry, Start
B -      9261 - auth_hash_seg_entry, Start
B -      9620 - auth_hash_seg_exit, Start
B -    106222 - elf_segs_hash_verify_entry, Start
B -    175875 - PBL, End
B -    144539 - SBL1, Start
B -    205936 - GCC [RstStat:0x0, RstDbg:0x600000] WDog Stat : 0x4
B -    214049 - clock_init, Start
D -      7442 - clock_init, Delta
B -    221674 - boot_flash_init, Start
D ,     16073 - boot_flash_init, Delta
B -    237808 - boot_config_data_table_init, Start
D -      5307 - boot_config_data_table_init, Delta - (575 Bytes)
B -    246196 - Boot Setting :  0x00020618
B -    252387 - CDT version:2,Platform ID:8,Major ID:4,Minor ID:2,Subtype:2
B -    259280 - sbl1_ddr_set_params, Start
B -    260897 - Pre_DDR_clock_init, Start
B -    266570 - Pre_DDR_clock_init, End
B -    908961 - do ddr sanity test, Start
D -        30 - do ddr sanity test, Delta
B -    913627 - Image Load, Start
D -    256749 - QSEE Image Loaded, Delta - (580996 Bytes)
B -   1171230 - Image Load, Start
D -     15006 - DEVCFGImage Loaded, Delta - (13592 Bytes)
B -   1186297 - Image Load, Start
D -    192089 - APPSBL Image Loaded, Delta - (436808 Bytes)
B -   1378447 - QSEE Execution, Start
D -        61 - QSEE Execution, Delta
B -   1384913 - SBL1, End
D -   1243028 - SBL1, Delta
S - Flash Throughput, 2318 KB/s  (1032643 Bytes,  445476 us)
S - DDR Frequency, 800 MHz
S - Core 0 Frequency, 800 MHz


U-Boot 2016.01 (May 19 2025 - 05:12:52 +0000), Build: jenkins-WRT-fWF810D_FPT-121

DRAM:  smem ram ptable found: ver: 1 len: 4
512 MiB
NAND:  QPIC controller support serial NAND
ID = 3a6c2
Vendor = c2
Device = a6

Serial Nand Device Found With ID : 0xc2 0xa6
Serial NAND device Manufacturer:MX35UF2GE4AD
Device Size:256 MiB, Page size:2048, Spare Size:128, ECC:8-bit
qpic_nand: changing oobsize to 80 from 128 bytes
SF: Unsupported flash IDs: manuf 00, jedec 0000, ext_jedec 0000
ipq_spi: SPI Flash not found (bus/cs/speed/mode) = (0/0/48000000/0)
256 MiB
MMC:   sdhci: Node Not found, skipping initialization

PCI Link Intialized
In:    serial@78AF000
Out:   serial@78AF000
Err:   serial@78AF000
machid: 8040202
eth0 MAC Address from ART is not valid
eth1 MAC Address from ART is not valid
Uboot: reading press button state again
Press Esc to stop autoboot:  0
```
my blind trust to the more "good looking" nand betrayed me so hard...  
thats it! hopefully this will help someone or me in future. who knows?
