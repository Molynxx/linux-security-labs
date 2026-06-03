# proc_filesystem

## Cel
Zrozumienie plików w folderze przechowującym informacje o procesach oraz strukturę wpisu w plikach.

## Czym jest /proc
To wirtualny system plików (w pamięci RAM), który zawiera informacje o procesach, sprzęcie i  jądrze. Nie istnieje fizycznie na dysku. W tym katalogu można podejrzeć szczegóły procesów, nawet jeśli polecenia `ps` lub `top` zostały podmienione (np. przez rootkita). 

## Kluczowe pliki i katalogi
- `/proc/cpuinfo` - informacje o procesorze, 
- `/proc/meminfo` - informacje o pamięci RAM,
- `/proc/PID/` - katalog dla każdego procesu,
- `/proc/PID/cmdline` -  polecenie z tą ścieżką pozwala sprawdzić, czy proces nie ma podejrzanych argumentów (`cat /proc/12345/cmdline`). Należy zwracać uwagę na uruchomienia z `/tmp` lub `/home`, interpreterów (python, perl, bash -c), netcat (nc -e, nc -lvp), procesów podszywających się pod kworker, poleceń pobierania z adresów www,
- `/proc/PID/exe` - symboliczny link do pliku wykonywalnego, pokazuje prawdziwą ścieżkę, nawet jeśli `ps` kłamie (`ls -la /proc/23456/exe`),
- `/proc/PID/cwd` - symboliczny link do katalogu roboczego, pokazuje z jakiego katalogu uruchomiono proces (`ls -la /proc/12334/cwd`),
- `/proc/PID/environ` - zmienne środowiskowe (`cat /proc/23435/environ | tr '\0' '\n'`). 
- `/proc/PID/status` - stan procesu pokazuje UID, GID, PPID, stan, itp. (`cat /proc/23455/status`),
- `/proc/34567/stack` - stos wywołań jądra, pomocne w wykrywaniu fałszywych kworker, 
- `/proc/32423/maps` - mapa pamięci, wykrywanie wstrzykniętego kodu, należy zwracać uwagę na plik biblioteki lub programu, jeśli jest w `/home` lub `/tmp` oznacza to, że jakieś biblioteki lub programy mogły zostać podmienione, 
- `/proc/PID/fd/` - lista otwartych deskryptorów, pozwala na wyrywanie otwartych połączeń sieciowych, jeśli są sockery, to może być reverse shell. 

## Ja czytać wpisy w /proc 

### Za pomocą polecenia `ls -la /proc/PID/` można zobaczyć informacje:  
```
lrwxrwxrwx 1 root root 0 Jun 3 10:00 exe -> /usr/bin/python3  
lrwxrwxrwx 1 root root 0 Jun 3 10:00 cwd -> /tmp/.hidden  
lrwx------ 1 root root 0 Jun 3 10:00 0 -> /dev/null  
lrwx------ 1 root root 0 Jun 3 10:00 1 -> /dev/null  
lrwx------ 1 root root 0 Jun 3 10:00 2 -> /var/log/error.log  
lrwx------ 1 root root 0 Jun 3 10:00 3 -> socket:[12345]   
```  
Ważne oznaczenia:
- `exe` - link do pliku wykonywalnego (prawdziwa ścieżka),
- `cwd` - Current Working Directory - katalog roboczy, 
- `0` - standardowe wejście (stdin) - wejście procesu, skąd proces czyta dane (domyślnie klawiatura),
- `1` - standardowe wyjście (stdout) - wyjście procesu, gdzie proces wypisuje wyniki (domyślnie ekran),
- `2` - standardowe wyjście błędów (stderr) - wyjście błędów, gdzie proces wypisuje błędy (domyślnie ekran), 
- `3`, `4`, `5`... - kolejne otwarte deskryptory, dodatkowe pliki, które proces otworzył (logi, sockety, potoki),
- `3u` - deskryptor otwarty w trybie odczytu i zapisu (read + write). Inne oznaczenie sposobu w jaki proces używa otwartego pliku: `r` - read, `w` - write,
- `socket:[12345]` - gniazdo sieciowe (połączenie),
- `(deleted)` - proces trzyma otwarty plik, który został usunięty - to częsta technika ukrywania malware. 

### Za pomocą polecenia `lsof -p PID` można zobaczyć więcej informacji niż za pomocą `ls`.  
Ważne oznaczenia:
- `txt` - plik wykonywalny (kod programu), to nie jest plik tekstowy `.txt`,
- `cwd` - katalog roboczy, 
- `mem` - biblioteka załadowana do pamięci (.so),
- `rtd` - root directory,
- `DEL` - plik usunięty, ale proces go trzyma, 
- `sock` - gniazdo sieciowe (socket).  
Przykład:  
`update 1234 root txt REG ... /tmp/.hidden/update (deleted)`   
- `txt` - plik wykonywalny (nie .txt),
- (deleted) - plik usunięty, ale proces działa, 
- `REG` - regular file - zwykły plik na dysku. 

## Przydatne polecenia
Gdy PID procesu jest znany (np. z `ps aux` lub `top`):
- `ls -la /proc/1234/exe` - znajdź prawdziwą ścieżkę, 
- `cat /proc/1234/cmdline` - sprawdzenie argumentów procesu (czasem widać -c z kodem),
- `cat /proc/1234/environ` - sprawdzanie zmiennych środowiskowych,
- `ls -la /proc/1234/fd/` - sprawdzenie otwartych deskryptorów (jeśli są sockety to może być reverse shell),
- `ls -la /proc/1234/cwd ` - sprawdzenie katalogu roboczego, czy wskazuje na `/tmp`, `/home/user/.cache`.

## Wnioski bezpieczeństwa
- nie powinno się polegać wyłącznie na `ps` i `top` - rootkit może je podmienić, 
- (deleted) - plik usunięty, ale proces działa - to częsta technika rootkitów, 
- otwarte przez proces połączenia sieciowe mogą oznaczać reverse shell, 
- proces może działać na własnych zmiennych środowiskowych, dlatego warto sprawdzać, jakie są używane przez proces,
- warto sprawdzać prawdziwą ścieżkę do pliku wykonywalnego oraz katalog roboczy, ponieważ `ps` może kłamać. 
