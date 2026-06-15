# checking_suid_sgid

## Cel 
Zrozumienie, jak znajdować i oceniać ryzyko związane z plikami SUID, SGID w systemie Linux z perspektywy SOC.

## Czym są SUID i SGID
- `SUID` (Set User ID) - plik uruchamia się z uprawnieniami właściciela, nie osoby, która go uruchomiła,
- `SGID` (Set Group ID) - plik uruchamia się z uprawnieniami grupy nie osoby, która go uruchomiła.   
Jak rozpoznać `SUID` i `SGID`:
	- `-rwsr-xr-x` - zamiast `x` u właściciela jest `s`), 
	- `-rwxr-sr-x` - zamiast `x` u grupy jest `s`.   
UWAGA: więcej informacji o tym czym są `SUID` i `SGID` znajduje się w pliku `03_permissions_and_ownership/suid_sgid_sticky-bit_PL.md`. 

## Dlaczego SUID/SGID mogą być niebezpieczne
- eskalacja uprawnień - jeśli atakujący znajdzie źle skonfigurowany plik `SUID` roota, może wykonać własny kod z uprawnieniami roota,
- persistence - atakujący może stworzyć własny plik `SUID` (np. w `/tmp`) i użyć go do powrotu po włamaniu, 
- lateral movement - w środowisku z wieloma użytkownikami, plik `SGID` może dać dostęp do plików grupy, do której atakujący nie powinien mieć dostępu. 
- każdy plik `SUID/SGID`, który nie jest standardowym plikiem systemowym, jest potencjalnie niebezpieczny. 

## Jak znaleźć pliki SUID/SGID
- `find / -type f -perm -4000 2>/dev/null` - znajduje wszystkie pliki z bitem `SUID` w systemie, 
- `find / -type f -perm -2000 2>/dev/null` - znajduje wszystkie pliki z bitem `SGID` w systemie. 
- `find / -type f \( -perm -4000 -o -perm -2000\) 2>/dev/null` - znajduje wszystkie pliki `SUID` i `SGID`.

## Podejrzane lokalizacje 
- katalogi domowe użytkowników (`/home`), 
- katalogi tymczasowe (`/tmp`, `/var/tmp`, `/dev/shm`), 
- katalogi z aplikacjami niestandardowymi (np. `shell`, `backdoor`, `.hidden`).

## Podejrzane uprawnienia
- plik `SUID` należący do zwykłego użytkownika (nie roota), 
- plik `SGID` należący do grupy, do której nie powinien należeć. 

## Co robić w przypadku znalezienia podejrzanego pliku
- należy niezwłocznie usunąć plik jeśli to nie jest plik standardowy:
	- `sudo rm /ścieżka/do/pliku`, 
- jeśli to standardowy plik, ale nie powinien mieć `SUID` - należy usunąć bit:
	- `sudo chmod u-s /ścieżka/do/pliku`, 

## Wnioski bezpieczeństwa
- należy monitorować pliki SUID i SGID, 
- zwracać uwagę na typ plików z tymi bitami - czy to są pliki standardowe,
- zwracać uwagę na lokalizację plików z bitami `s`(katalogi tymczasowe),
- sprawdzać przynależność plików SUID - jeśli to zwykły użytkownik należy sprawdzić czy bit jest uzasadniony, 
- monitoruj (np. Wazuh) tworzenie nowych plików SUID/SGID.

## Odnośniki w repozytorium 
- Teoria SUID/SGID: `03_permissions_and_ownership/suid_sgid_sticky-bit_PL.md`
- Wyszukiwanie plików: `08_tmp_and_file_locations/file_searching_PL.md`
- Audyt uprawnień: 09_basic_security_checks/world_writable_files_PL.md