# **ma35d1-evb-aarch32-heterogeneous**

## **Introduction**

For more board information, please refer to the README.md in the numaker-hmi-ma35d1 folder. The MA35D1 series features dual Cortex-A35 cores, allowing simultaneous execution of Linux on core0 and rt-thread on core1.

## **Requirements**

- [MA35D1 Buildroot](https://github.com/OpenNuvoton/MA35D1_Buildroot)
- MA35D1 rt-thread

## **Build**

Follow these steps to build a bootable SD image.

### **Build core1 execution**

1. Specify the execution address as 0x88000000 in the linking script of rt-thread.
    - File path: `<path-to-rtthread>\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\linking_scripts\aarch32.ld`

    ```bash
    SECTIONS
    {
        ......
        ......

        . = 0x88000000;

        ......
        ......
    }
    ```

2. Specify the execution address and available DDR memory size in board.h.
    - File path: `<path-to-rtthread>\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\board\board.h`

    ```bash
    #define BOARD_SDRAM_START   0x88000000
    #define BOARD_SDRAM_SIZE    0x02000000
    ```

3. Build the preload object code using the AARCH64 toolchain and rt-thread execution using the AARCH32 toolchain.
    - Download the toolchain here: [Arm GNU Toolchain - AARCH64 baremetal 8-2019-q3](https://developer.arm.com/-/media/Files/downloads/gnu-a/8.3-2019.03/binrel/gcc-arm-8.3-2019.03-i686-mingw32-aarch64-elf.tar.xz?revision=1c8636ec-0cca-455b-be17-726f1b396f46&rev=1c8636ec0cca455bbe17726f1b396f46&hash=0107DE39C8803E7C762E2FB079BF3822AA6B6AE2)
    - Set your toolchain installation path in the **env_build.bat** script.

4. After building the preload, close the environment window to reset the environment variable. Do not build the rt-thread execution directly.
    - Navigate to `<path-to-rtthread\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous\preload>`

    ```bash
    > env_build.bat

    make
    rm -f preload.o
    Build preload.o ...aarch64-elf-gcc -c -I. -I./../ -c -march=armv8-a -x assembler-with-cpp -D__ASSEMBLY__ preload.ASM -nostartfiles  -Wl,--gc-sections,-cref,-Map=preload.map,-cref,-u,_start -T ../linking_scripts/aarch32.ld
    aarch64-elf-objdump -d preload.o > preload.txt

    python transcode.py
    ```

    - Navigate to `<path-to-rtthread\bsp\nuvoton\ma35d1-evb-aarch32-heterogeneous>`

    ```bash
    > menuconfig --generate
    > scons -c
    > scons -j 8

    LINK rtthread.elf
    arm-none-eabi-objcopy -O binary rtthread.elf rtthread.bin
    arm-none-eabi-size rtthread.elf
       text    data     bss     dec     hex filename
     268892    2506   64512  335910   52026 rtthread.elf
    scons: done building targets.
    ```

### **Build SD image**

Follow the steps below to construct a heterogeneous system in buildroot. Enable the following options and fill in the related parameters.

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

The UART16 core1 used permission is assigned to SUBM by default. We should switch it to TZNS.

```bash
MA35D1_Buildroot$ vi output/build/arm-trusted-firmware-custom/fdts/ma35d1.dtsi

<To modify UART16 permission to TZNS.>
- <UART16_SUBM>,
+ <UART16_TZNS>,

MA35D1_Buildroot$ make uboot-rebuild linux-rebuild optee-os-rebuild arm-trusted-firmware-rebuild -j 8
MA35D1_Buildroot$ make -j 8
```

Finally, the SD image is located in the following path.

```bash
MA35D1_Buildroot$ ls output/images/core-image-buildroot-ma35d1-som-256m.rootfs.sdcard
```

## **Deploy**

Use the **balenaetcher** utility to flash the image to a 16GB class10 SD card. Then, insert it into the NuMaker-HMI-MA35D1 board.

## **Demo video clip on Youtube**

[![Alexa APP ACK HMCU with Lamp Board.](https://img.youtube.com/vi/bga1cw80A7w/0.jpg)](https://www.youtube.com/watch?v=bga1cw80A7w)