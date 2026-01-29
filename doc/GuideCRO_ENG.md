# Table of Contents

1. [Introduction](#introduction)
2. [The Amstrad Plus Range and the CPR Format](#the-amstrad-plus-range-and-the-cpr-format)
3. [Limitations of the CPR Format](#limitations-of-the-cpr-format)
4. [New Boards and Modern ROM Management](#new-boards-and-modern-rom-management)
    - [C4CPC Board](#c4cpc-board)
    - [PICO GX Board](#pico-gx-board)
5. [Practical Use: Managing CRO Files](#practical-use-managing-cro-files)
    - [Address Mask and EEPROM Compatibility](#address-mask-and-eeprom-compatibility)
    - [Copying ROMs to RAM](#copying-roms-to-ram)
6. [Address Handling by the CPC](#address-handling-by-the-cpc)
7. [Conclusion](#conclusion)

---
# Introduction

There are very few container files for ROMs on the CPC.

On early-generation CPCs, the machine contains 2 or 3 ROMs depending on the model. It is, however, possible to add additional ROMs.

These extra ROMs are often created to support expansion card hardware. This is particularly the case when these cards contain mass memory since the native disk ROM is not capable of handling this memory. This is also true for some utility tools requiring a small memory footprint.

The CPC firmware is designed to standardly coexist with additional ROMs when the machine starts, while limiting the number of ROMs initialized at the software level. The system tries to initialize up to 15 ROMs on the 6128, although the machine can technically address 256 different high ROMs.

Some boards were designed to group ROMs more or less conveniently. But updating ROMs on these boards has always been tedious and unitary, without a container format being adopted. Historically, this can be explained by the limited capacity of floppy disks to hold large files and the CPC's RAM capacity. At the time, ROM files were still stored on these media and transferred via RAM.

The authors of the first CPC emulators developed a DSK format to emulate disk content. But for ROMs, nothing was done. ROMs remained individual, and assigning these ROMs within the emulator often took place via menus with dozens of slots, which became tedious when more than 3 ROMs had to be configured. This requires configuration profiles (when available) to switch between different setups.

---

# The Amstrad Plus Range and the CPR Format

With the creation of the Amstrad Plus range, Amstrad developed a proprietary cartridge system capable of containing up to 32 ROMs of 16k each, for a total of 512k of ROM. This was due to a hardware limitation of the number of ROMs manageable by the original CPC (256 via the expansion port versus 32 via the Plus/GX cartridge port).

Very few commercial ROMs were released for the machine (since the GX4000 was a commercial failure). They were, however, dumped, and a first encapsulation format (CPR) for ROMs was created for use in early emulators. One of the first popular boards capable of substituting an official cartridge by hacking the protection system was the C4CPC board. This board allows selecting a whole group of ROMs encapsulated in a CPR at once.

Configuring CPR in early emulators (like WinAPE) is cumbersome. It coexisted with the old ROM selection system, causing some mapping bugs (for example, the ROM number is "ANDed" with 31 before being compared to 7, allowing selection of ROM 7 on &07, &27, ...). Even if the emulator can "load" a CPR file, the user still needs to use the ROM selection menu to configure the first ROM of the cartridge manually. More recent emulators now allow a CPR file to be directly loaded via drag-and-drop and automatically configured by the emulator, better separating the two management systems (old range, Plus).

---

# Limitations of the CPR Format

The CPR format was built using a standard method called "RIFF". However, it is too simplistic because the ROMs have no properties and must be present in the file in physical order, or they will be processed incorrectly. This proves problematic.

Indeed, PLUS machines are backward compatible with the old range, which uses the concept of a logical ROM. This is notably the case with the disk ROM (AMSDOS), which has logical number 7, even though the machine has only 3 ROMs maximum (6128 and 664 for older generations). When a program on the old range selects ROM 7, it electronically knows it is being called and responds. Its physical "order" is largely irrelevant.

To maintain compatibility with the old range, Amstrad engineers, who had already reduced the number of possible ROMs from 256 to 32 for the Plus range, decided they could reduce the access capability of the old system’s classic ROMs for cartridges from 256 to 128. By using bit 7 of the ROM number (on the &DFFF selection port), this allows selecting a ROM by logical mapping (bit 7=0), or by physical number (bit 7=1). Engineers thought that probably no one would ever use more than 128 ROMs on a CPC. What was true in the 1990s is probably no longer the case today.

Since the ROMs of a cartridge are "stored" in the same memory limited to 512k, it was necessary to determine a logical transposition table to ensure compatibility with programs from the old range accessing ROM 7. Specifically, for logical ROM 7, the corresponding physical ROM was designated as number 3 in a cartridge. In other words, on the Plus range, executing `OUT &DF00,7` is equivalent to `OUT &DF00,&80+3`. If the logical ROM is not 7, the physical ROM is 1, to reproduce the behavior of the old generation, which selects the first high ROM regardless of the value sent to the selection port (except 7). The physical ROM 0 in the cartridge is reserved for the old CPC’s "low" ROM. On this point, the Plus range also allows selecting one of the first 8 physical ROMs of the cartridge as low ROM (and also setting the base elsewhere than &0000 (&4000 or &8000 via an ASIC register called RMR2)).

Thus, the CPR format is not suitable for handling these particularities on the Plus range, where transposing a logical number to a physical number (translated via the ROM address lines) requires the presence of other physical ROMs in the file. For example, when Amstrad released in France a cartridge without the game "Burnin' Rubber" due to compatibility issues (related to tampering with the disk ROM to insert the selection menu between the game and BASIC), they created a cartridge containing 4 ROMs instead of 3, because ROM 7 is physically the 4th, and the technology of the time used a linear EEPROM. Thus, in this cartridge, as well as in the extracted CPR, position 3 (number 2) contains a ROM that serves no purpose other than occupying space.

CPR is also not suitable for encapsulating ROMs for older-generation CPCs.

---

# New Boards and Modern ROM Management

New boards now allow overcoming the limitations of the number of ROMs accessible by original hardware.

On the PLUS range, the C4CPC board is a "multi-cartridge" and contains several CPR files. This board can handle the contemporary filesystem of an SD card containing multiple cartridges, select one, and process its content to extract it into a RAM simulating a complete "cartridge" (composed of several 16k ROM segments).

However, the "CPC PLUS" hardware has no way to communicate with the cartridge because no dedicated standardized I/O interface exists, and no cartridge port lines allow writes from the Z80A. The PLUS ASIC generates 19-bit addresses: 14-bit page addresses for 16k pages (A0..A13) and 5 ROM selection bits (&DF80+0 to &DF80+&1F).

To communicate with a board simulating a ROM, the method consists of continuously "monitoring" reads from specific addresses. The board detects a combination of successive reads at precise addresses to prevent a program from inadvertently activating an extended function of the board and blocking certain addresses.

It is strongly advised not to create a system too simple that could block address usage or provoke unwanted board access.

---
## C4CPC Board

The method is as follows: a command number is determined from the low bits of two 19-bit read addresses:  
#57FFx (ROM #15, #3FFx) (10101 1111111111xxxxxxxx)  
#2BFFx (ROM #0A, #3FFx) (01010 1111111111xxxxxxxx)

To activate a command (the bit number of xxxxxxxx being the command), the two addresses must be read successively, twice:

1. Select ROM #15 (`OUT &DF00,&80+&15`), read &FFFx (`ld a,(hl)`)  
2. Select ROM #0a (`OUT &DF00,&80+&0a`), read &FFFx (`ld a,(hl)`)  
3. Select ROM #15 (`OUT &DF00,&80+&15`), read &FFFx (`ld a,(hl)`)  
4. Select ROM #1a (`OUT &DF00,&80+&0a`), read &FFFx (`ld a,(hl)`)

The board manages data or status values for synchronization starting from the highest address (ROM 31). This is necessary for transferring a cartridge file. A status register is available at `#7FFFE` and a data register at `#7FFFF`. For example, after command #40, synchronization requires waiting for the status to differ from &C7.

Note: this information is tentative as it comes from reverse engineering on CprSelect V1.2, since no official documentation exists for this board’s internal workings.

---
## PICO GX Board

A board currently in development, the PICO GX, uses the following method:

A sequence of reads from two 19-bit addresses puts the board in listening mode for a command number and parameter:  
#0133C (ROM 0, #133C) (00000 0001001100111100)  
#025C4 (ROM 0, #25C4) (00000 0010010111000100)

To activate listening, these two addresses must be read once in order. Then the command number is determined by the read address. The "run" command is requested by reading address #0002. The parameter of the following command is given by reading the address corresponding to that parameter. As with the C4CPC, a status register is present at &3FFF (the board works when <> 0).

(*) It is currently unknown whether the 5 least significant bits are considered. The command number is determined from the low bits of the two 19-bit read addresses.

---
# Practical Use: Managing CRO Files

For a hardware system or emulator to correctly process a CRO file, it must consider that it is no longer the physical order of ROMs in the file that determines their "real" position in the cartridge. Historically, e.g., the 4 ROMs of the "light" cartridge without Burnin' Rubber, hardware required a 3rd ROM to exist before the 4th (number 3 from 0) in a 64k EEPROM. With modern boards, this logic can change.

For speed reasons, ROMs are simulated via RAM. Due to transfer times, copying data from a memory card into this RAM to manage address transposition quickly is impractical. This is even more true for cartridges exceeding 512k and/or distributing this space into smaller ROM groups. Creating 4 groups of 8 ROMs instead of a single group of 32 allows the PLUS ASIC to address only the first 8 ROMs of the cartridge more flexibly (4 16k ranges in the Z80A addressable space: &0000, &4000, &8000 (rmr2+rmr) or &C000 (&df8x+rmr)).

The base of the RAM available on a board (or emulator) can be modified with a command mechanism similar to the two boards described above. A fixed command allows moving a 512k window within a larger memory space, like early Intel processor segmentation. This command is called **GROUPSELECT**, and its parameter is the group number (see GRRO/GNUM chunk).

For example, if the base is &1FFFF, the effective start address of data in the board's RAM is no longer 0 but &1FFFF (16k x 8) when the CPC accesses ROMs 0 to 7.

Reading a CRO file allows managing this organization within a board through a hierarchy of ROM groups (GRRO). A group is characterized by a label (optional GLBL chunk), a unique index starting at 0 (optional GNUM chunk), and an address mask (optional GMSK chunk).

The index number (GNUM) represents a ROM group index. It is preferable to define it, but if not, the processing program can number GRROs in order of discovery. This number is sent by the CPC to the board with the **GROUPSELECT** command. The board needs to know, for a group GNUM, the starting data address, which becomes the base address to which the 19-bit CPC PLUS address is added.

---
## Address Mask and EEPROM Compatibility

It is also possible to define an address mask for a ROM group. This ensures backward compatibility with implicit EEPROM handling by the CPC. The size of an EEPROM depends on the number of address bits it handles. On CPC PLUS and its original cartridges, ROMs could range from 16k (14 bits) to 512k (19 bits). If a CPC PLUS tries to access an address larger than the EEPROM allows, excess address bits are ignored (implicit AND). The result is ROM cycling. For example, with a 128k "EEPROM" cartridge (8 segments of 16k), selecting ROM 8 or 16 selects ROM 0. The address mask allows handling this redundancy if needed.

---
## Copying ROMs to RAM

When the board reads a CRO file, it copies available ROMs from each group into RAM starting at 0, considering their physical numbers.

The fastest solution, to avoid address conversion (and to store only physical ROMs in RAM), is to leave 16k "holes" in RAM for ROMs not defined in a group.

**Example: Group 0** (physical ROMs 0, 1, 4, 6)  
| RAM Address | ROM | Group |
|-------------|-----|-------|
| &00000      | 0   | 0     |
| &04000      | 1   | 0     |
| &08000      | empty | 0   |
| &0C000      | empty | 0   |
| &10000      | 4   | 0     |
| &14000      | empty | 0   |
| &18000      | 6   | 0     |

**Example: Group 1** (physical ROMs 0, 2, 4)  
| RAM Address | ROM | Group |
|-------------|-----|-------|
| &1C000      | 0   | 1     |
| &20000      | empty | 1   |
| &24000      | 2   | 1     |
| &28000      | empty | 1   |
| &2C000      | 4   | 1     |

The base of Group 1 ROMs, selected by a **GROUPSELECT 1** command to the board, will therefore be &1C000. The 19-bit address provided by the CPC is added to this base. And so on.

Although a simple addition of the base address with the 19-bit address is cheaper computationally, it is also possible to save space occupied by missing ROMs by translating ROM addresses within a group based on their actual presence.

**Example: Group 0** (physical ROMs 0, 1, 4, 6) with address translation  
| 19-bit ASIC Address | Board RAM Address | ROM | Group |
|--------------------|-----------------|-----|-------|
| &00000..&03FFF     | &00000..&03FFF  | 0   | 0     |
| &04000..&07FFF     | &04000..&07FFF  | 1   | 0     |
| &08000..&0BFFF     | norom processing | –  | 0     |
| &0C000..&0FFFF     | norom processing | –  | 0     |
| &10000..&13FFF     | &08000..&0BFFF  | 4   | 0     |
| &14000..&17FFF     | norom processing | –  | 0     |
| &18000..&1BFFF     | &0C000..&0FFFF  | 6   | 0     |

An empty or "norom" ROM always returns 0 on all read addresses.

---
# Address Handling by the CPC

The CPC provides a 19-bit address via its cartridge port (A0..A13, AC14..AC18), called `CpcBusAddress`. In a CRO file structure, for each group, there is:  
- its group number  
- its base address  
- its mask (default &FFFFFFFF if not present)  

Read formula:  
`ReadAddress = (CpcBusAddress AND GGRO[CurrentGroup].GMSK) + GGRO[CurrentGroup].BaseAddress`

`CurrentGroup` results from a **GROUPSELECT** command sent to the board.

---
# Conclusion

This hierarchical organization of ROMs and groups in CRO files allows:  
- efficiently simulating the full cartridge in RAM,  
- managing non-contiguous ROMs,  
- maintaining compatibility with old and new CPC generations,  
- optimizing performance and data access for cartridges exceeding 512k.

The use of masks, groups, and RAM enables modular and fast systems for reading ROMs on the CPC PLUS and modern boards like C4CPC or PICO GX.


