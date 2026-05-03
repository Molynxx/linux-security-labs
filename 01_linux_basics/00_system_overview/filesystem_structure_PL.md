# Filesystem Structure

## Cel 
Zrozumieć jak wygląda system plików w systemach Linux

## Czym jest FHS 
System plików, na którym opiera się Linux nosi nazwę FHS (File Hierarchy Standard). Jest to standard, który określa, który katalog do czego służy, dzięki czemu niezależnie od dystrybucji Linuxa (Kali, Ubuntu, Debian, RHEL, Arch) wiadomo, że:
- logi są w `/var/log`,
- konfiguracja systemu jest w `/etc`,
- programy użytkownika są w `/usr/bin`.
Wszystkie te foldery znajdują się w głównym folderze systemu `root` (/).

## Główne katalogi w `/` (root)

- `/bin` - podstawowe programy użytkownika (`ls`, `cp`, `mv`, `cat`), 
- `/sbin` - programy dla administratora (`fdisk`, `mount`, `reboot`),
- `/etc` - konfiguracja systemu i usług (`passwd`, `shadow`, `sudoers`),
- `/var` - zmienne dane - logi, maile, cache,
- `/tmp` - pliki tymczasowe (czyszczone przy restarcie),
- `/home` - katalogi domowe zwykłych użytkowników, 
- `/root` - katalog domowy roota, 
- `/boot` - pliki potrzebne do uruchomienia systemu (jądro, GRUB),
- `/dev` - pliki urządzeń (dyski, terminale),
- `/proc` - wirtualny system plików - informacje o procesach i sprzęcie, 
- `/sys` - wirtualny system plików - interakcja z jądrem i urządzeniami, 
- `/usr` - programy, biblioteki, dokumentacja użytkowników, 
- `/lib` - biblioteki współdzielone dla programów z `/bin` i `/sbin`,
- `/opt` - dodatkowe pakiety (oprogramowanie firm trzecich),
- `/mnt` - tymczasowe montowanie systemów plików, 
- `/media` - automatyczne montowanie nośników (pendrive, CD),
- `/srv` - dane serwisów (www, ftp).  

## Uwaga
W nowoczesnych dystrybucjach `/bin`, `/sbin`, `/lib`  są często linkami symbolicznymi do odpowiednich katalogów  w `/usr` (np. `/bin` -> `/usr/bin`).