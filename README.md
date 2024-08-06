# **ma35d1-evb-aarch32-heterogeneous**

## **Introduction**

About more board information, please refer README.md in numaker-hmi-ma35d1 folder. More details, the MA35D1 series is with dual Cortex-A35 cores. Just note linux execution at core0 and rt-thread execution at core1 at the same time in this document.

## **Requirement**

- [MA35D1 Buildroot](https://github.com/OpenNuvoton/MA35D1_Buildroot)

- MA35D1 rt-thread

## **Build**

Follow steps to build a bootable SD image in this section.

### **Build core1 execution**

- Specified execution address to 0x88000000 in linking script of rtthread.

    File path: <path-to-rtthread>\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\linking_scripts\aarch32.ld

```bash
...

SECTIONS
{
    ......
    ......

    . = 0x88000000;

    ......
    ......
```

- Specified execution address and available DDR memory size in board.h.

    File path: <path-to-rtthread>\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\board\board.h

```bash
...
...
#define BOARD_SDRAM_START   0x88000000
#define BOARD_SDRAM_SIZE    0x02000000
...
...
```

- Finally, build the preload object code using AARCH64 toolchain and rtthread execution using AARCH32 toolchain.

- You can download the toolchain here. [Arm GNU Toolchain - AARCH64 baremetal 8-2019-q3](https://developer.arm.com/-/media/Files/downloads/gnu-a/8.3-2019.03/binrel/gcc-arm-8.3-2019.03-i686-mingw32-aarch64-elf.tar.xz?revision=1c8636ec-0cca-455b-be17-726f1b396f46&rev=1c8636ec0cca455bbe17726f1b396f46&hash=0107DE39C8803E7C762E2FB079BF3822AA6B6AE2) And, set your toolchain installation path in **env_build.bat** script.

- After preload building, **you should close env-window to reset environment variable**. Don't build the rtthread execution directly.

**path-to-rtthread\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\preload**

```bash
> env_build.bat

make
rm -f preload.o
Build preload.o ...aarch64-elf-gcc -c -I. -I./../ -c -march=armv8-a -x assembler-with-cpp -D__ASSEMBLY__ preload.ASM -nostartfiles  -Wl,--gc-sections,-cref,-Map=preload.map,-cref,-u,_start -T ../linking_scripts/aarch32.ld
aarch64-elf-objdump -d preload.o > preload.txt

python transcode.py
```

**path-to-rtthread\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous**

```bash
> menuconfig --generate
> scons -c
> scons -j 8

...
...
...
LINK rtthread.elf
arm-none-eabi-objcopy -O binary rtthread.elf rtthread.bin
arm-none-eabi-size rtthread.elf
   text    data     bss     dec     hex filename
 268892    2506   64512  335910   52026 rtthread.elf
scons: done building targets.
```

### **Build SD image**

Below steps shown you how to construct heterogeneous system in buildroot. Enable below option and fill-in related parameters.

```bash
MA35D1_Buildroot$ make numaker_som_ma35d16a81_defconfig
MA35D1_Buildroot$ make menuconfig
       Bootloaders —>
            [*] Add SCP BL2 image into FIP Image.
                Load Image into FIP Image (A35 image) -->
            (rtthread.bin) SCP_BL2 binary file names
            (0x88000000) The execution address of CORE1
            (0x2000000) The execution size of CORE1
<Save & Exit>

MA35D1_Buildroot$ mkdir output/images
MA35D1_Buildroot$ cp <path-to-rtthread.bin> output/images
MA35D1_Buildroot$ make -j 8
```

The UART16 core1 used permission is assigned to SUBM by default, we should switch to TZNS.

```bash
MA35D1_Buildroot$ vi output/build/arm-trusted-firmware-custom/fdts/ma35d1.dtsi

<To modify UART16 permission to TZNS .>
- <UART16_SUBM>,
+ <UART16_TZNS>,

MA35D1_Buildroot$ make uboot-rebuild linux-rebuild optee-os-rebuild arm-trusted-firmware-rebuild -j 8
MA35D1_Buildroot$ make -j 8
```

Finally, the SD image is in below path.

```bash
MA35D1_Buildroot$ ls output/images/core-image-buildroot-ma35d1-som-256m.rootfs.sdcard
```

## **Deploy**

Using **balenaetcher** utility to flash image to a 16GB class10 SD card.Then, insert the NuMaker-HMI-MA35D1 board.

## **Demo video clip on Youtube** ##

[![Alexa APP ACK HMCU with Lamp Board.](https://img.youtube.com/vi/bga1cw80A7w/0.jpg)](https://www.youtube.com/watch?v=bga1cw80A7w)
