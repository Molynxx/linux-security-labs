# Filesystem_structure

## Cel 
Zrozumieć, jak wygląda system plików w systemie Linux.

## Czym jest FHS 
System plików, na którym opiera się Linux, nosi nazwę FHS (File Hierarchy Standard). Jest to standard, który określa, który katalog do czego służy, dzięki czemu niezależnie od dystrybucji Linuxa (Kali, Ubuntu, Debian, RHEL, Arch) wiadomo, że:
- logi są w `/var/log`,
- konfiguracja systemu jest w `/etc`,
- programy użytkownika są w `/usr/bin`.
Wszystkie te foldery znajdują się w głównym folderze systemu `root` (/).

## Główne katalogi w `/` (root)

- `/bin` - podstawowe programy użytkownika (`ls`, `cp`, `mv`, `cat`), 
- `/sbin` - programy dla administratora (`fdisk`, `mount`, `reboot`),
- `/etc` - konfiguracja systemu i usług (`passwd`, `shadow`, `sudoers`),
- `/var` - zmienne dane - logi, maile, cache,
- `/tmp` - pliki tymczasowe (czyszczone przy restarcie jeśli jest tmpfs),
- `/home` - katalogi domowe zwykłych użytkowników, 
- `/root` - katalog domowy roota, 
- `/boot` - pliki potrzebne do uruchomienia systemu (jądro, GRUB),
- `/dev` - pliki urządzeń (dyski, terminale),
- `/proc` - wirtualny system plików w pamięci (procfs) - informacje o procesach i sprzęcie, 
- `/sys` - wirtualny system plików w pamięci (sysfs) - interakcja z jądrem i urządzeniami,  
- `/usr` - programy, biblioteki, dokumentacja użytkowników (drugi główny katalog programów po `/bin`),
- `/lib` - biblioteki współdzielone dla programów z `/bin` i `/sbin`,
- `/opt` - dodatkowe pakiety (oprogramowanie firm trzecich),
- `/mnt` - tymczasowe montowanie systemów plików, 
- `/media` - automatyczne montowanie nośników (pendrive, CD),
- `/srv` - dane usług (www, ftp), 
- `/run` - dane procesów od momentu startu systemu,
- `/var/tmp` - pliki tymczasowe (np. aplikacji), które muszą być zachowanee po restarcie, nie jest czyszczony przy resecie jak `/tmp`, 
- `/lib64` - w niektórych dystrybucjach istnieje `/lib64`  dla bibliotek 64-bitowych. 

## Uwaga
W nowoczesnych dystrybucjach `/bin`, `/sbin`, `/lib`  są często linkami symbolicznymi do odpowiednich katalogów  w `/usr` (np. `/bin` -> `/usr/bin`).