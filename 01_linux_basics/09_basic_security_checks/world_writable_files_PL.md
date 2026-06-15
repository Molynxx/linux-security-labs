# world_writable_files

## Cel
Zrozumienie, czym są pliki i katalogi `world-writable`, jakie stanowią zagrożenie w środowisku produkcyjnym oraz jak je lokalizować, zanalizować i eliminować z perspektywy analityka SOC/Blue Team.

## Czym są pliki world_writable
W systemie Linux każdy plik lub katalog `world-writable` oznacza, że każdy użytkownik systemu może go modyfikować. 
- dla pliku: uprawnienia `666` (`-rw-rw-rw`) - każdy może czytać i zapisywać, 
- dla katalogu: uprawnienia `777` (`drwxrwxrwx`) - każdy może tworzyć, usuwać i modyfikować pliki w środku.
- jak to sprawdzić:
	- `ls -l plik.txt` 
	- `ls -ld katalog/`

## Dlaczego to niebezpieczne
- modyfikacja ważnych plików: jeśli plik konfiguracyjny (np. w `/etc`) jest `world-writable`, atakujący może zmienić ustawienia systemu, dodać sobie uprawnienia lub wyłączyć logowanie. 
- wrzucenie własnego narzędzia: atakujący może umieścić skrypt lub narzędzie (np. reverse shell) w katalogu `world-writable` np. `/tmp`, a następnie go uruchomić, 
- brak lokalnego bitu (sticky bit): dla katalogów współdzielonych jak `/tmp` sticky bit (`+t`) zapobiega usuwaniu plików przez nieuprawnionych użytkowników. Jeśli katalog nie ma sticky bit, atakujący może usuwać lub podmieniać pliki innych użytkowników, co może prowadzić do awarii lub eskalacji uprawnień. 
	- jak sprawdzić czy sticky bit istnieje:
		- `ls -ld /tmp` jeśli jest na końcu litera `t` (`drwxrwxrwt`) oznacza, że sticky bit jest ustawiony dla katalogu.

## Jak znaleźć pliki i katalogi world-writable
- `find / -type f -perm -o+w 2>/dev/null` - wyświetli wszystkie pliki `world-writable` w systemie, 
- `find / -type d -perm -0002 ! -perm -1000 2>/dev/null` - wyświetli wszystkie katalogi `world-writable`, które nie mają ustawionego sticky bita. 
	- `type d` - tylko katalogi, 
	- `-perm -0002` - katalog musi mieć uprawnienia o+w (`world-writable`),
	- `! -perm -1000` - oraz nie może mieć ustawionego sticky bita, 
	- `2>/dev/null` - ukryj błędy. 

## Przykład zagrożenia  
W marcu 2025 roku odkryto lukę w narzędziu `below` (monitoring systemu). 
- problem: program `below` działał z uprawnieniami root i tworzył katalog `/var/log/below` z uprawnieniami 777 (`world-writable`), 
- wykorzystanie: lokalny użytkownik mógł podmienić plik loga na dowiązanie symboliczne do `/etc/shadow`, 
- skutek: program root nadpisałby plik z hasłami, zmieniające jego uprawnienia na 666, co pozwoliło by każdemu go odczytać,
- wniosek: nawet zaufane oprogramowanie może przez błąd w uprawnieniach otworzyć drzwi do kompromitacji systemu. 

## Wnioski bezpieczeństwa 
- należy wyszukiwać i weryfikować czy pliki/katalogi muszą być `world-writable`. W większości przypadków nie powinny, 
- by zmienić ustawienia uprawnień na bezpieczne należy użyć:
	- `sudo chmod 755 /ścieżka/do/katalogu`, 
	- `sudo chmod 644 /ścieżka/do/pliku`,
- dla katalogów współdzielonych należy ustawić lepki bit za pomocą polecenia: 
	- `sudo chmod +t /ścieżka/do/katalogu`, 
- monitorować w SIEM (np. Wazuh) tworzenie nowych plików `world-writable`  w newralgicznych lokalizacjach (`/etc`, `/var/www`, `/usr/local/bin`).

## Uwaga:
Więcej informacji na temat uprawnień i zagrożeń z nimi związanych znajduje się w plikach tego repozytorium w:
- 03_permissions_and_ownership:
	- chmod_chown_PL.md, 
	- file_permission_cases_PL.md,
	- suid_sgid_sticky-bit_PL.md,
- 08_tmp_and_file_locations:
	- tmp_analysis_PL.md