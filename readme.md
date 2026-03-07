# Wood40 ZMK Configuration Guide

Complete step-by-step instructions for adding a custom Wood40 keyboard to ZMK in WSL.

## Hardware Specifications

- **Controller**: Tenstar Robot Pro Micro nRF52840
- **Layout**: 4 rows × 5 columns per half (split keyboard)
- **Column pins**: P1.04, P1.06, P0.10, P0.09, P1.01
- **Row pins**: P0.17, P0.24, P1.00, P0.11

## Prerequisites

1. **Install WSL2 and Ubuntu**
2. **Install Python 3.12**:

   ```bash
   sudo add-apt-repository ppa:deadsnakes/ppa
   sudo apt update
   sudo apt install python3.12-full -y
   ```

3. **Clone ZMK and Set Up Environment**:

   ```bash
   cd ~
   git clone https://github.com/zmkfirmware/zmk.git
   cd zmk
   python3.12 -m venv .venv
   source .venv/bin/activate
   pip install west
   west init -l app
   west update
   west sdk install
   ```

## Board Configuration Steps

### 1. Create the board directory structure

```bash
cd ~/zmk/app/boards/arm
mkdir -p wood40
cd wood40
```

### 2. Create board configuration files

#### `Kconfig.board`

```kconfig
config BOARD_WOOD40
    bool "Wood40 keyboard"
    depends on SOC_NRF52840_QIAA

if BOARD_WOOD40

config BOARD
    default "wood40"

endif
```

#### `Kconfig.defconfig`

```kconfig
if BOARD_WOOD40

config BOARD
    default "wood40"

config ZMK_KEYBOARD_NAME
    default "Wood40"

endif
```

#### `wood40_defconfig`

```conf
CONFIG_SOC_SERIES_NRF52X=y
CONFIG_SOC_NRF52840_QIAA=y
CONFIG_BOARD_WOOD40=y

# Enable MPU
CONFIG_ARM_MPU=y

# Enable hardware stack protection
CONFIG_HW_STACK_PROTECTION=y

# Enable GPIO
CONFIG_GPIO=y

CONFIG_USE_DT_CODE_PARTITION=y
CONFIG_BUILD_OUTPUT_UF2=y

CONFIG_MPU_ALLOW_FLASH_WRITE=y
CONFIG_NVS=y
CONFIG_SETTINGS_NVS=y
CONFIG_FLASH=y
CONFIG_FLASH_PAGE_LAYOUT=y
CONFIG_FLASH_MAP=y
```

### 3. Create the device tree file

#### `wood40.dts`

```dts
/dts-v1/;
#include <nordic/nrf52840_qiaa.dtsi>
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    model = "Wood40";
    compatible = "wood40";

    chosen {
        zephyr,code-partition = &code_partition;
        zephyr,sram = &sram0;
        zephyr,flash = &flash0;
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <5>;
        rows = <4>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4)
            RC(3,0) RC(3,1) RC(3,2) RC(3,3) RC(3,4)
        >;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-gpio-matrix";
        diode-direction = "col2row";
        wakeup-source;

        row-gpios
            = <&gpio0 17 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            , <&gpio0 24 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            , <&gpio1  0 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            , <&gpio0 11 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            ;

        col-gpios
            = <&gpio1  4 GPIO_ACTIVE_HIGH>
            , <&gpio1  6 GPIO_ACTIVE_HIGH>
            , <&gpio0 10 GPIO_ACTIVE_HIGH>
            , <&gpio0  9 GPIO_ACTIVE_HIGH>
            , <&gpio1  1 GPIO_ACTIVE_HIGH>
            ;
    };
};

&flash0 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        sd_partition: partition@0 {
            reg = <0x00000000 0x00026000>;
        };
        code_partition: partition@26000 {
            reg = <0x00026000 0x000c6000>;
        };
        storage_partition: partition@ec000 {
            reg = <0x000ec000 0x00008000>;
        };
        boot_partition: partition@f4000 {
            reg = <0x000f4000 0x0000c000>;
        };
    };
};
```

### 4. Create the keymap

#### `wood40.keymap`

```c
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <
                &kp Q     &kp W     &kp E     &kp R     &kp T
                &kp A     &kp S     &kp D     &kp F     &kp G
                &kp Z     &kp X     &kp C     &kp V     &kp B
                &kp ESC   &kp TAB   &kp LCTL  &kp LALT  &kp LGUI
            >;
        };
    };
};
```

## Building the Firmware

### 5. Build

```bash
cd ~/zmk/app
west build -b wood40 -p always
```

The firmware will be generated at: `~/zmk/app/build/zephyr/zmk.uf2`

### 6. Flash to Pro Micro

**From WSL:**

1. Put your Pro Micro in bootloader mode (double-tap reset button)
2. The board will appear as a USB mass storage device
3. Copy the firmware:

   ```bash
   cp ~/zmk/app/build/zephyr/zmk.uf2 /mnt/<drive-letter>/
   ```

**From Windows:**

1. Navigate to `\\wsl$\Ubuntu\home\<your-username>\zmk\app\build\zephyr\`
2. Copy `zmk.uf2` to the Pro Micro's drive that appears when in bootloader mode

## Directory Structure

Your final board directory should look like:

```text
~/zmk/app/boards/arm/wood40/
├── Kconfig.board
├── Kconfig.defconfig
├── wood40_defconfig
├── wood40.dts
└── wood40.keymap
```

## Troubleshooting

### Keys not registering correctly

- Try changing `diode-direction` from `"col2row"` to `"row2col"`
- Or swap `GPIO_ACTIVE_HIGH` with `GPIO_ACTIVE_LOW` in the pin definitions

### Build errors

- Make sure you're in the activated venv: `source ~/zmk/.venv/bin/activate`
- Clean build directory: `rm -rf ~/zmk/app/build`
- Rebuild: `west build -b wood40 -p always`

### Pro Micro not appearing as USB drive

- Double-tap the reset button quickly
- Try a different USB cable or port
- Check Device Manager (Windows) to verify the device is detected

## Next Steps

- Customize your keymap in `wood40.keymap`
- Add additional layers
- Configure split keyboard support (if using both halves)
- Set up GitHub Actions for automated builds
