# An Introduction to RISC-V Boot Flow

## 01 Common boot flow
Commonly used multiple boot stages model

01 ROM
 1. Runs from On-Chip ROM
 2. Uses On-Chip SRAM
 3. SOC power-up and clock setup

02 Loader
   1. DDR initialization
   2. Loads RUNTIME and BOOTLOADER
   3. Examples:
      1. BIOS/UEFI
      2. U-Boot SPL
      3. Coreboot