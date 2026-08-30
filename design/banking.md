# Memory banking proposal

## Goals

The system uses one SST39SF010 128 KiB flash ROM and one AS6C1008-55PCN
128 KiB SRAM. Each device is divided into eight physical banks of
16 KiB. The Z80's 64 KiB address space is divided into four 16 KiB
logical slots.

The design should provide:

- a deterministic ROM boot mapping;
- a conventional, contiguous 64 KiB RAM map for CP/M;
- two freely programmable lower slots, including intentional mirroring;
- a stable upper 32 KiB area for the stack, BIOS, interrupt routines, and
  mapping-transition code; and
- room for a future temporary ROM overlay for an NMI monitor.

To avoid ambiguity, this document uses **logical slot** for a region of
the Z80 address space and **physical bank** for a 16 KiB region within a
RAM or ROM chip.

## Logical memory map

| Logical slot | Z80 address range | Reset mapping | Runtime mapping |
| --- | --- | --- | --- |
| Slot 0 | `$0000-$3FFF` | ROM bank 0 | Any ROM or RAM bank |
| Slot 1 | `$4000-$7FFF` | RAM bank 1 | Any ROM or RAM bank |
| Slot 2 | `$8000-$BFFF` | RAM bank 2 | Fixed |
| Slot 3 | `$C000-$FFFF` | RAM bank 3 | Fixed |

Slots 0 and 1 may independently select any of the eight ROM banks or any
of the eight RAM banks. No restriction is placed on duplicate mappings;
for example, both slots may intentionally select the same physical bank.

Slots 2 and 3 always select RAM banks 2 and 3 respectively. This keeps
the upper half of memory stable while either lower slot is changed.

## Normal CP/M mapping

After boot, the mapper is configured as follows:

```text
Slot 0 -> RAM bank 0
Slot 1 -> RAM bank 1
Slot 2 -> RAM bank 2
Slot 3 -> RAM bank 3
```

This presents CP/M with an ordinary contiguous 64 KiB RAM address space.
RAM banks 4 through 7 remain available for purposes such as a ramdisk,
alternate programs, data buffers, or future banked-memory support.

## Mapping registers

Each programmable slot has a four-bit descriptor:

```text
bit 3     Device selection: 0 = RAM, 1 = ROM
bits 2-0  Physical bank number, 0-7
```

This gives the following descriptor values:

| Value | Mapping | Value | Mapping |
| --- | --- | --- | --- |
| `$0` | RAM bank 0 | `$8` | ROM bank 0 |
| `$1` | RAM bank 1 | `$9` | ROM bank 1 |
| `$2` | RAM bank 2 | `$A` | ROM bank 2 |
| `$3` | RAM bank 3 | `$B` | ROM bank 3 |
| `$4` | RAM bank 4 | `$C` | ROM bank 4 |
| `$5` | RAM bank 5 | `$D` | ROM bank 5 |
| `$6` | RAM bank 6 | `$E` | ROM bank 6 |
| `$7` | RAM bank 7 | `$F` | ROM bank 7 |

The complete programmable state therefore consists of two descriptors:

```text
MAP0[3:0]  Mapping for Slot 0 ($0000-$3FFF)
MAP1[3:0]  Mapping for Slot 1 ($4000-$7FFF)
```

Each descriptor must be updated atomically. Both registers, together
with mapper status such as `BOOT_MODE`, should be readable through I/O
ports so that software and interrupt handlers can save and restore the
mapping reliably.

### I/O-port interface

The current 74HC138 decodes Z80 I/O address lines `A7-A5`. Its
`/BANK0_CS` and `/BANK1_CS` outputs correspond to the following port
ranges:

```text
MAP0: $60-$7F
MAP1: $80-$9F
```

Because `A4-A0` are not decoded, each register has 32 aliases. Software
should use `$60` and `$80` as their canonical port addresses:

```text
MAP0_PORT  EQU $60
MAP1_PORT  EQU $80
MAP_RAM    EQU $00
MAP_ROM    EQU $08
```

Only `D3-D0` are required because the two descriptors have separate
chip-select signals. On a qualified I/O write, the CPLD performs:

```text
/BANK0_CS active and /WR active: MAP0 := D3-D0
/BANK1_CS active and /WR active: MAP1 := D3-D0
```

During an I/O read, the selected descriptor is driven onto `D3-D0`.
Software must mask the unused upper nibble unless `D7-D4` are also
connected to the CPLD and explicitly driven to zero:

```z80
        in      a,(MAP0_PORT)
        and     $0f
```

## CPLD implementation

The memory mapper is implemented entirely in the CPLD; the 74HC273 bank
latch is not required. CPU address lines `A0-A13` connect directly to the
corresponding address inputs of the AS6C1008-55PCN SRAM and SST39SF010
flash. The CPLD drives:

```text
RAM_A14-RAM_A16
ROM_A14-ROM_A16
/RAM_CS
/ROM_CS
```

The selected descriptor is determined from CPU address lines `A15:A14`:

```text
A15:A14 = 00  use MAP0
A15:A14 = 01  use MAP1
A15:A14 = 10  select RAM bank 2
A15:A14 = 11  select RAM bank 3
```

For Slots 0 and 1, descriptor bits 2-0 drive the upper address inputs of
the selected memory device. For Slots 2 and 3, the RAM bank number is
fixed by the decode logic.

The effective mappings are therefore:

```text
effective_slot0 = BOOT_MODE ? ROM bank 0 : MAP0
effective_slot1 = MAP1_VALID ? MAP1 : RAM bank 1
effective_slot2 = RAM bank 2
effective_slot3 = RAM bank 3
```

During each memory cycle, `A15:A14` chooses one of these four mappings.
The mapping's lower three bits are presented on the selected device's
`A16:A14` inputs, and bit 3 determines which chip select is asserted.
The other memory device remains deselected.

Conceptually, the chip-select equations are:

```text
/RAM_CS is active when /MREQ is active and the selected mapping is RAM
/ROM_CS is active when /MREQ is active and the selected mapping is ROM
```

`/RAM_CS` and `/ROM_CS` must be mutually exclusive by construction. The
bank address and chip-select propagation delays must also satisfy the
memory access-time requirements at the selected Z80 clock frequency.

Flash writes should be disabled by default and enabled only through a
separate, deliberate programming-unlock mechanism.

### Timing of a mapping-register write

A bank switch is performed with a Z80 `OUT` instruction. During the I/O
write, `/MREQ` is inactive, so neither memory is selected. The CPLD
captures `D3-D0` while the appropriate bank-register chip select and
`/WR` are active. It then updates the physical bank-address outputs and
memory decode before the next memory cycle begins.

For predictable registered logic, the Z80 clock is connected to the
CPLD's dedicated `GCLK1` input on pin 43. CPU address line `A6` was moved
to general-purpose pin 21; `A15` is on pin 39.

## Reset and boot operation

The CPLD contains a `BOOT_MODE` state that is asserted by hardware reset.
While `BOOT_MODE` is active, Slot 0 is forced to ROM bank 0. Slot 1
defaults to RAM bank 1 but remains programmable so the bootloader can
temporarily access RAM bank 0. Slots 2 and 3 remain fixed:

```text
Slot 0 -> ROM bank 0
Slot 1 -> RAM bank 1
Slot 2 -> RAM bank 2
Slot 3 -> RAM bank 3
```

Internally, the CPLD can use a `MAP1_VALID` flag that is cleared by reset.
Until the first MAP1 write, the effective Slot 1 descriptor is RAM bank
1. A MAP1 write stores the supplied descriptor and sets `MAP1_VALID`.

Writing MAP0 stores its descriptor and clears `BOOT_MODE` atomically.
Software can subsequently map ROM bank 0 normally by writing descriptor
`$8` to MAP0; a separate command to re-enable `BOOT_MODE` is unnecessary.

The boot ROM cannot replace the slot from which it is currently
executing and then continue fetching instructions from that slot.
Transitioning to the normal CP/M map therefore uses code in a stable
upper slot:

1. Load CP/M and any required contents of RAM bank 0.
2. Copy a short transition routine into Slot 2 or Slot 3.
3. Jump to that transition routine.
4. Set MAP1 to RAM bank 1.
5. Write RAM bank 0 to MAP0, atomically disabling `BOOT_MODE`.
6. Jump to the CP/M entry point.

If RAM bank 0 must be populated while ROM occupies Slot 0, the bootloader
can temporarily expose RAM bank 0 through programmable Slot 1.

Mapping-control code and the active stack should be placed in Slot 2 or
Slot 3 whenever a lower slot is being changed.

## Software procedures

### Selecting a bank

For example, the following selects ROM bank 3 in Slot 1:

```z80
        ld      a,MAP_ROM + 3     ; descriptor $B
        out     (MAP1_PORT),a
```

The next memory access in `$4000-$7FFF` goes to ROM bank 3. With flash
writing locked, writes to that range have no effect.

The following selects RAM bank 5 instead:

```z80
        ld      a,MAP_RAM + 5     ; descriptor $5
        out     (MAP1_PORT),a
```

The next access in `$4000-$7FFF` goes to RAM bank 5.

### Temporarily using an additional RAM bank

Mapping code, its stack, and any interrupt handlers must remain outside
the slot being changed. This example assumes they are in fixed Slot 3:

```z80
        di

        in      a,(MAP1_PORT)     ; save the current Slot 1 mapping
        and     $0f
        push    af                ; stack is in fixed Slot 3

        ld      a,MAP_RAM + 5
        out     (MAP1_PORT),a

        ; Use RAM bank 5 through $4000-$7FFF here.
        ; Do not call code or access data that was in the old Slot 1.

        pop     af
        out     (MAP1_PORT),a     ; restore the previous mapping

        ei
```

Interrupts are disabled around the temporary mapping because an
interrupt handler might otherwise depend on code or data in Slot 1.
They may remain enabled only when every possible handler and all of its
data are confined to the fixed upper slots.

### Mirroring

Writing the same descriptor to both registers intentionally mirrors one
physical bank into both lower logical slots. The following maps RAM bank
5 at both `$0000-$3FFF` and `$4000-$7FFF`:

```z80
        ; This routine must execute from Slot 2 or Slot 3.
        di
        ld      a,MAP_RAM + 5
        out     (MAP1_PORT),a
        out     (MAP0_PORT),a
```

After the MAP0 write, the next instruction fetch must be from a fixed
slot unless the newly mapped bank deliberately contains matching code at
the next address.

### Loading RAM bank 0 during boot

At reset, Slot 0 contains ROM bank 0. To load the future CP/M page-zero
bank, the bootloader temporarily maps RAM bank 0 into Slot 1:

```z80
        ld      a,MAP_RAM + 0
        out     (MAP1_PORT),a

        ; Copy the 16 KiB image intended for $0000-$3FFF into
        ; its temporary window at $4000-$7FFF.

        ld      a,MAP_RAM + 1
        out     (MAP1_PORT),a
```

The bootloader then executes a transition routine from Slot 2 or Slot 3:

```z80
switch_to_cpm:
        di
        ld      sp,$ff00          ; stack in fixed Slot 3

        ld      a,MAP_RAM + 1
        out     (MAP1_PORT),a

        ld      a,MAP_RAM + 0
        out     (MAP0_PORT),a     ; expose RAM 0 and leave BOOT_MODE

        jp      $0000             ; enter the now-contiguous RAM system
```

After the MAP0 write, the visible memory is RAM banks 0, 1, 2, and 3 in
order, producing the normal 64 KiB CP/M map.

### General safety rules

- Execute a mapping change from a slot that will remain mapped.
- Keep the active stack in Slot 2 or Slot 3 during lower-slot changes.
- Disable interrupts unless all interrupt code and data are in fixed
  memory.
- Save and restore a temporary mapping instead of assuming its previous
  value.
- Do not switch away the bank containing the instruction following the
  `OUT`, unless that transition was deliberately arranged.
- Treat mapper writes as privileged BIOS or monitor operations under
  CP/M; arbitrary applications can otherwise remove CP/M from memory.

## Future NMI overlay

The same mapper can later support an NMI monitor without destroying the
programmed value of `MAP0`. A separate internal `NMI_OVERLAY` state can
temporarily override Slot 0 with ROM bank 0 while leaving both mapping
registers and the fixed upper RAM slots unchanged. Clearing the overlay
then restores the interrupted mapping immediately.

The exact NMI-recognition and safe overlay timing are deliberately left
for a separate design specification.
