# chmod i chown - zarządzanie uprawnieniami i właścicielami

## Cel
Zrozumienie, jak poprawnie zmieniać uprawnienia i właścicieli plików i katalogów oraz jakich błędów konfiguracyjnych unikać.

## Uprawnienia plików i katalogów
Uprawnienia plików i katalogów w systemach Linux ma postać:  
	 `rwxr-xr-- 1 alice developers 1234 May 12 11:00 script.sh`  
- Jak czytać uprawnienia:
	- `-` typ pliku (`-` = zwykły plik, `d` = katalog, `l` = link),
	- `rwx` - uprawnienia właściciela pliku (alice), 
	- `r-x` - uprawnienia grupy (developers),
	- `r--` - uprawnienia innych (others). 
- dla pliku:
	- `r` - odczyt zawartości,
	- `w` - modyfikacja,
	- `x` - wykonanie.
- dla katalogów:
	- `r` - przeglądanie zawartości (`ls`),
	- `w` - tworzenie/usuwanie plików w katalogu,
	- `x` - wchodzenie do katalogu (`cd`). 
- notacja ósemkowa - uprawnienia można zapisać także za pomocą liczb:
	- `r` = 4,
	- `w` = 2,
	- `x` = 1,
	- `-` = 0.   
	Przykład: rwxr-xr-- -> dla właściciela 4+2+1 = 7, dla grupy 4+0+1= 5, dla innych 4+0+0 = 4 stąd zapis ósemkowy to 754. 
- bity specjalne:    
	Znajdują się przed uprawnieniami (w pierwszym polu), jeśli występują. Zapis ósemkowy:
	- `1` - sticky bit, 
	- `2` - SGID,
	- `4` - SUID, 
	Aby jednocześnie dodać kilka obliczamy wartość analogicznie dla uprawnień rwx. Czyli: jeśli chcemy dodać zarówno SGID i SUID -> 2+4 = 6, jeśli chcemy nadać wszystkie 3 bity specjalne -> 1+2+4 = 7.

## chmod
To polecenie służące do zmiany uprawnień dla plików i katalogów. Polecenie to można zastosować na dwa sposoby:
- zapis ósemkowy:
	- `chmod 1755 katalog` - sticky bit jest reprezentowany przez 1 na początku, 
	- `chmod 755 plik` - ustawi uprawnienia `rwxr-xr-x`, 
- zapis symboliczny:
	- `u` (user), `g` (group), `o` (others),
	- `+` (dodaj), `-` (usuń), `=` (ustaw),
	- `r`, `w`, `x`, `s` (SUID/SGID), `t` (sticky bit).
	- Przykłady:
		- `chmod u+x script.sh` - dodaj wykonanie dla właściciela, 
		- `chmod g-w plik.txt` - usuń zapis dla grupy,
		- `chmod a=r plik.txt` - ustaw tylko czytanie dla wszystkich,
		- `chmod u+s program` - ustaw SUID,
		- `chmod 1777 /tmp` - ustaw sticky bit.
- zagrożenia: 
	- ustawienie nieprawidłowych uprawnień, zwłaszcza dla wrażliwych plików i katalogów, może powodować, że nieuprawniona osoba dostanie prawa do odczytu, zapisu bądź wykonania. 
 - detekcja: 
 	- polecenie `ls -la` pokaże uprawnienia wszystkich plików i katalogów w danej lokalizacji.  

## chown i chgrp 
To polecenia służące do zmiany właściciela (chown) i grupy (chgrp). 
- Składnia:
	- `chown alice:developers plik` - zmiana użytkownika i grupy, 
	- `chown alice plik` - zmiana tylko użytkownika, 
	- `chown :developers plik` - zmiana tylko grupy, 
	- `chgrp developers plik ` - zmiana tylko grupy, to samo co `chown :developers plik`. 
- zagrożenia: 
	- zmiana właściciela na innego użytkownika - atakujący może przejąć plik, 
	- zmiana grupy na uprzywilejowaną (wheel, sudo, docker) - jeśli atakujący przejmie konto należące do grupy, może stworzyć własny skrypt i nadać mu grupę uprzywilejowaną, do której należy skompromitowane konto. 
- detekcja:
	- `find /etc /bin /sbin /usr/bin -not -user root 2>/dev/null` - to polecenie przeszuka wskazane katalogi i wypisze wszystkie pliki i katalogi nienależące do root. W katalogach systemowych oraz konfiguracyjnych wszystkie pliki powinny należeć do roota. `2>/dev/null` ukrywa błędy. UWAGA: W `/etc` niektóre pliki mogą należeć do innych użytkowników, np. niektóre pliki konfiguracyjne usług (do www-data lub mysql). W `/bin` i `/usr/bin` wszystko powinno należeć do roota. 

## Wnioski bezpieczeństwa
- należy monitorować uprawnienia plików, pod kątem zmian uprawnień (`r`, `w`, `x`, `s` (SUID - u właściciela), `s` (SGID - u grupy), `t` (sticky bit)) oraz sprawdzać ich zasadność, 
- `umask` ma wyłącznie wpływ na nowo tworzone pliki, nie ma wpływu na proces zmiany uprawnień oraz istniejące pliki, 
- bezpieczne uprawnienia dla kluczowych plików:
	- `/etc/shadow` - 600 (rw-------),
	- `/etc/passwd` - 644 (rw-r--r--),
	- `/etc/sudoers` - 440 (r--r-----),
	- `/boot/vmlinuz-*` - 644 (rw-r--r--),
	- `/tmp` - 1777 (drwxrwxrwt). 