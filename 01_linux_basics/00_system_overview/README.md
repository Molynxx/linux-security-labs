# 00_system_overview

### Lab goal 
Understanding the basic Linux structure: what is located where, how the system starts, which folders are important from a SOC perspective.

### Scope
- `filesystem_structure` - the map of folders in `/` (FHS),
- `important_directories` - deep analysis of critical folders for SOC: `/etc`, `/var/log`, `/tmp`, `/home`, `/root`, `/proc`, `/boot`, `/lib`, `/bin`,
- `boot_process_basics` - stages of booting: BIOS/UEFI -> GRUB -> kernel -> initramfs -> systemd.

### Why is this important (SOC/IR)
- navigating in the system without guessing - we know where to look for logs, configuration or temporary files,
- detecting modifications in essential folders - to detect backdoors, replaced binaries, suspicious files, etc.,
- understanding where an attacker can inject a backdoor - e.g. boot process, initramfs, GRUB,
- building this basic knowledge before log analysis, process analysis and forensics. 

## Polski / Polish

### Cel 
Zrozumienie podstawowej struktury systemu Linux, czyli co, gdzie się znajduje, jak startuje system, które katalogi są kluczowe dla SOC. 

### Zakres
- `filesystem_structure` - mapa katalogów w `/` (FHS),
- `important_directories` - głębsza analiza katalogów krytycznych dla SOC: `/etc`, `/var/log`, `/tmp`, `/home`, `/root`, `/proc`, `/boot`, `/lib`, `/bin`,
- `boot_process_basics` - etapy bootowania: BIOS/UEFI -> GRUB -> kernel -> initramfs -> systemd.

### Dlaczego to ważne (SOC/IR)
- nawigacja w systemie bez zgadywania - wiadomo gdzie szukać logów, konfiguracji czy plików tymczasowych,
- wykrywanie modyfikacji w newralgicznych katalogach - czy nie ma backdoorów, podmiany binariów, podejrzanych plików, itp.,
- zrozumienie, gdzie atakujący może wstrzyknąć backdoora - np. boot process, initramfs, GRUB,
- posiadanie tej podstawowej wiedzy, przed analizą logów, procesów i forensics, ułatwia zrozumienie. 
