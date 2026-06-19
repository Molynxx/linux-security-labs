# 00_system_overview

## Cel 
Zrozumienie podstawowej struktury systemu Linux, czyli co, gdzie się znajduje, jak startuje system, które katalogi są kluczowe dla SOC. 

## Zakres
- `filesystem_structure` - mapa katalogów w `/` (FHS),
- `important_directories` - głębsza analiza katalogów krytycznych dla SOC: `/etc`, `/var/log`, `/tmp`, `/home`, `/root`, `/proc`, `/boot`, `/lib`, `/bin`,
- `boot_process_basics` - etapy bootowania: BIOS/UEFI -> GRUB -> kernel -> initramfs -> systemd.

## Dlaczego to ważne (SOC/IR)
- nawigacja w systemie bez zgadywania - wiadomo gdzie szukać logów, konfiguracji czy plików tymczasowych,
- wykrywanie modyfikacji w newralgicznych katalogach - czy nie ma backdoorów, podmiany binariów, podejrzanych plików, itp.,
- zrozumienie, gdzie atakujący może wstrzyknąć backdoora - np. boot process, initramfs, GRUB,
- posiadanie tej podstawowej wiedzy, przed analizą logów, procesów i forensics, ułatwia zrozumienie. 
