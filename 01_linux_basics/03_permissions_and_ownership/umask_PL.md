# umask (user file creation mask)

## Cel
Zrozumienie, jak działa polecenie umask oraz jaki ma wpływ na uprawnienia plików, katalogów i bezpieczeństwo. 

## Czym jest umask
`umask` (user file creation mask) określa jakie uprawnienia będą domyślnie odejmowane od nowo tworzonych plików i katalogów. Domyślne ustawienia:
- dla plików domyślnie zazwyczaj ustawione są uprawnienia 666 (rw-rw-rw), lecz umask odejmuje od tego określone bity. Przykład `umask 022` spowoduje ustawienie uprawnień dla nowo tworzonych plików na 644 (rw-r--r--).
- dla katalogów domyślnym ustawieniem jest 777 (rwxrwxrwx), więc przy `umask 022` nowo utworzony katalog będzie miał uprawnienia 755 (rwxr-xr-x). 
- Aby sprawdzić aktualne ustawienie maski wystarczy wpisać:
	- `umask` - pokazuje wynik w systemie ósemkowym (np. 0022)
	- `umask -S` - pokazuje wynik symbolicznie (u=rwx,g=rx,o=rx).

## Gdzie można zdefiniować wartość umask
- `PAM (pam_umask.so) w typie session` - ustawia umask dla całej sesji logowania, wartość należy ustawić bezpośrednio w PAM: `session optional pam_umask.so umask=022`, 
- `/etc/profile`, `~/.bashrc`, `~/.profile` - ustawia umask dla całej sesji. UWAGA: nadpisuje ustawienia PAM,
- ręcznie w shellu za pomocą polecenia `umask 022` - ustawienia umask będą działały wyłącznie w bieżącej sesji i jej potomnych. 

## Zagrożenia 
- Nieprawidłowe ustawienie `umask` w module `pam_umask.so` może powodować, że wszyscy użytkownicy będą tworzyć pliki z niebezpiecznymi uprawnieniami,
- zbyt niska wartość w `umask` może być śladem po ataku (np. gdy atakujący chce, żeby jego pliki były dostępne dla innych lub odwrotnie).

## Wnioski bezpieczeństwa i sposoby wykrywania zagrożeń
- należy ustawiać `umask` zgodnie z dobrą praktyką, tj.:
	- `umask 022` - domyślna wartość w wielu systemach - bezpieczne ustawienie, 
	- `umask 027` - bardziej restrykcyjnie, by inni nie mieli żadnego dostępu,
- nigdy nie ustawiać `umask 000`,
- do monitorowania poprawności ustawienia `umask` można wykorzystać polecenia:
	- `grep pam_umask /etc/pam.d/common-session /etc/pam.d/sshd` - sprawdzenie ustawień PAM. 
	- `grep -r "umask" /etc/profile /etc/bash.bashrc /home/*/.bashrc /home/*/.profile 2>/dev/null` - sprawdzenie, czy ktoś nie ustawił `umask` w plikach konfiguracyjnych,
	- `umask` - szybkie sprawdzenie ustawień `umask` dla bieżącej sesji. 

