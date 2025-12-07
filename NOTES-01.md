# Boot Flow (Code Path)
[BIOS]
  │
  └──► loads first 512 bytes (stage1.S)
            │
            ▼
   stage1: setup disk geometry & load stage1_5
      - uses BIOS int 0x13 calls
      - finds stage1_5 by absolute sector addresses
            │
            ▼
   stage1_5: knows filesystem (ext2, fat, reiserfs, etc.)
      - implemented in stage2/fsys_*.c
      - reads /boot/grub/stage2 from filesystem
            │
            ▼
   stage2: main GRUB shell
      - executes `start.S` → `stage2.c:init_stage2()`
      - parses /boot/grub/menu.lst
      - draws menu, waits for user, loads selected kernel
      - calls `boot.c:chainloader()` or `boot.c:linux_boot()`
            │
            ▼
   Kernel (Multiboot or chainloaded OS)


# Important Source Files (Stage2)
+--------------+-------------------------------------------------------------------------------------------------------------------+
| Directory    | Role                                                                                                              |
+--------------+-------------------------------------------------------------------------------------------------------------------+
| stage1/      | Tiny 512-byte boot sector (stage1.S, stage1.h). Lives in MBR or partition boot sector.                            |
| stage2/      | Main GRUB codebase (filesystem drivers, loader, UI). Files like stage2.c, disk_io.c, fsys_*.c, boot.c, etc.       |
| netboot/     | PXE / Etherboot modules — GRUB can fetch kernels via TFTP.                                                        |
| lib/         | Common utilities like getopt, device abstraction, used by both grub binary and installer utilities.               |
| util/        | Host-side utilities (run in userland, not bootloader): grub-install, mbchk, grub-md5-crypt.                       |
| grub/        | Source for the standalone grub shell program (runs in Linux userspace for installing GRUB).                       |
| docs/        | Texinfo manuals, multiboot.h reference, examples like kernel.c (the Multiboot demo).                              |
+--------------+-------------------------------------------------------------------------------------------------------------------+

# GRUB 0.97 — Dual Worlds
══════════════════════════════════════════════════════════════════════════════════════
                 GRUB 0.97 — TWO EXECUTION WORLDS
══════════════════════════════════════════════════════════════════════════════════════

                          ┌───────────────────────────────────────────┐
                          │        A. UNIX STAGE2 SIMULATOR           │
                          │     (for testing under Linux/Unix)        │
                          └───────────────────────────────────────────┘
                                       │
User runs:
   $ ./grub [options]
                                       │
                                       ▼
+────────────────────────────────────────────────────────────────────────────+
| main() – grub/main.c                                                      |
|────────────────────────────────────────────────────────────────────────────|
| • Parses command-line args (--boot-drive, --batch, etc.)                  |
| • Initializes terminal and flags (use_curses, use_pager, etc.)            |
| • Optionally waits for debugger (--hold)                                  |
| • Calls grub_stage2() to start simulated GRUB core                        |
+────────────────────────────────────────────────────────────────────────────+
                                       │
                                       ▼
+────────────────────────────────────────────────────────────────────────────+
| grub_stage2() – stage2/stage2.c                                           |
|────────────────────────────────────────────────────────────────────────────|
| • Initializes stage2 subsystems (term, device, fsys_*)                     |
| • Reads /boot/grub/menu.lst (or preset menu)                               |
| • Displays GRUB menu or “grub>” shell prompt                               |
| • Executes built-in commands (root, kernel, boot, etc.)                    |
| • Simulates kernel load (no actual hardware boot)                          |
| • Returns to Unix process when done                                        |
+────────────────────────────────────────────────────────────────────────────+
                                       │
                                       ▼
                          [Process exits normally under Unix]
──────────────────────────────────────────────────────────────────────────────────────
                                       │
                                       ▼
                          ┌───────────────────────────────────────────┐
                          │        B. BIOS BOOTLOADER FLOW            │
                          │     (real boot sequence on hardware)      │
                          └───────────────────────────────────────────┘
                                       │
BIOS power-on → reads MBR (sector 0)
                                       │
                                       ▼
+────────────────────────────────────────────────────────────────────────────+
| stage1 (stage1.S)                                                         |
|────────────────────────────────────────────────────────────────────────────|
| • 512-byte real-mode stub in MBR or partition boot sector                 |
| • Uses BIOS int 0x13 to load stage1_5 (absolute disk sectors)             |
| • Jumps to loaded stage1_5                                               |
+────────────────────────────────────────────────────────────────────────────+
                                       │
                                       ▼
+────────────────────────────────────────────────────────────────────────────+
| stage1_5 (stage1_5.c in stage2/)                                          |
|────────────────────────────────────────────────────────────────────────────|
| • Filesystem-aware loader (ext2, fat, reiserfs, etc.)                     |
| • Locates /boot/grub/stage2 within filesystem                             |
| • Loads stage2 image into memory                                          |
| • Jumps to its entry (start.S)                                            |
+────────────────────────────────────────────────────────────────────────────+
                                       │
                                       ▼
+────────────────────────────────────────────────────────────────────────────+
| stage2/start.S → cmain() (stage2/stage2.c)                                |
|────────────────────────────────────────────────────────────────────────────|
| • Switches from real to protected mode                                    |
| • Initializes console, memory, and devices                                |
| • Parses menu.lst and shows boot menu                                     |
| • Executes boot.c:linux_boot() or chainloader()                           |
| • Loads kernel/initrd into memory                                         |
| • Jumps to kernel entry point                                             |
+────────────────────────────────────────────────────────────────────────────+
                                       │
                                       ▼
                           [OS kernel executes — boot complete]
──────────────────────────────────────────────────────────────────────────────────────
                                       ▲
                                       │
═══════════════════════════════════════┴══════════════════════════════════════════════
 Summary
 ───────
 • grub/main.c  → runs under Unix, calls grub_stage2() (stage2 simulator)
 • stage1/1_5/2 → real BIOS boot chain executed at system startup
 • Both share much of the same core logic (stage2 codebase)
══════════════════════════════════════════════════════════════════════════════════════


