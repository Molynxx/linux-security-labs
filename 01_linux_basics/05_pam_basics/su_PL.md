# /etc/pam.d/su

## Cel laboratorium 
Zapoznanie się ze strukturą pliku /etc/pam.d/su oraz jego rolą w systemie. 

## Czym jest plik /etc/pam.d/su
Jest to plik obsługujący aplikację `su`, mówi w jaki sposób następuje przełączanie użytkownika, korzysta z modułów w typach PAM (auth, account, session). Struktura wpisów w pliku:  
	`[typ] [flaga] [moduł] [opcje]`  
Plik może także zawierać ustawienia niestandardowe wprowadzone przez dodatkowe moduły, które nie są ujęte w plikach common, lub jest potrzeba zmiany ustawień danego modułu na potrzeby aplikacji `su` (np. pam_limits.so).

## Standardowe moduły 
Poza załączonymi plikami common, plik może zawierać moduły:
- `pam_rootok.so` - moduł zwróci success gdy użytkownik jest rootem, pomija wszystkie dalsze moduły w fazie auth. 
- `pam_wheel.so use_uid` - pozwala na dostęp do polecenia `su` wyłącznie użytkownikom z grupy wheel 
- `pam_listfile.so` - moduł, który pozwala na dostęp do polecenia `su` użytkownikom, grupom lub hostom ujętym w pliku `/etc/su/nazwa`. 
Plik `/etc/su/nazwa` trzeba utworzyć z odpowiednimi uprawnieniami - 640 (-rw-r-----),
- `pam_limits.so` - moduł nakładający limity zasobów (CPU, ilość procesów, RAM, itp),
- `pam_time.so` - moduł pozwala ograniczyć użycie polecenia `su` tylko do godzin pracy,
- `pam_access.so` - moduł ten ogranicza użycie polecenia `su` do konkretnego hosta/TTY/użytkownika. 

## Monitoring 
Plik: `/var/log/auth.log`
- Udane `su` - su[12345]: pam_unix(su:session): session opened for user root by james(uid=1001)
- Nieudane (brak wheel): -  su[123456]: pam_wheel(su:auth): access denied for user john
- nieudane (złe hasło): - su[234234]: pam_unix(su:auth) authentication failure; logname=anna
- zamknięcie sesji: - su[23456] pam_unix(su:session): session closed for user root

## Wnioski bezpieczeństwa 
- brak modułu `pam_wheel.so` - każdy użytkownik może próbować `su` na root. 
- brak opcji use_uid w module `pam_wheel.so` -  bark powoduje sprawdznie grupy docelowej a nie  użytkownika wywołującego `su`,
- `pam_listfile.so` z opcją onerr=succeed - jeśli plik z listą dozwolonych użytkowników nie istnieje lub nie można go otworzyć, PAM i tak zezwala na `su`. 
- plik z listy (np. /etc/su/allowed)  ze złymi uprawnieniami - wskalacja uprawnień, uprawnienia pliku można sprawdzić za pomocą polecenie `ls -l /etc/su/allowed`,
- dodanie niepowołanego użytkownika do pliku listy - ktoś, kto nie powinien, zyskuje możliwość użycia `su`. Należy monitorować zmiany pliku (np. auditd) i alertować przy każdej modyfikacji, 
- konto systemowe (np. www-data) w grupie wheel - jeśli atakujący przejmie konto systemowe, może użyć `su - root`, jeśli konto jest w grupie wheel. Należy weryfikować kto należy do tej grupy za pomocą polecenia `getent group wheel`,
- `su` używane po godzinach pracy - potencjalny atak lub nieautoryzowana aktywność, można monitorować za pomocą polecenia `grep "session opened" /var/log/auth.log` i sprawdzić godziny. 

## Case study  

auth sufficient pam_rootok.so  
auth required pam_unix.so  
account required pam_unix.so  
session required pam_unix.so  

Analiza:  
- brak załączonych plików common co oznacza, że poszczególne typy poza sprawdzeniem użytkownika i hasła nie są w żaden sposób zabezpieczone a audyt nie będzie pełny, zagrożenia z tego wynikające to:
	- brak modułów ochronnych w typie `auth`:
		- `pam_faillock.so`- brak zliczania prób nieudanego logowania, brak ochrony przed brute force, 
		- `pam_faildelay.so` - brak opóźnienia po nieudanej próbie logowania, ułatwia brute force,
	- brak modułów zabezpieczających system w typie session:
		- `pam_env.so` - środowisko nie jest ustawione, istnieje możliwość podmiany PATH, LD_PRELOAD, LD_LIBRARY_PATH, TMPDIR, LANG. itp,
		- `pam_limits.so` - brak limitów sesji - możliwe: DoS, fork bomb, core dump,
	- bark modułów logujących dane sesji w typie session:
		- `pam_loginuid.so` - brak zarejestrowanego oryginalnego UID użytkownika (AUID), utorudniony aydyt,
		- `pam_systemd.so` - brak rejestracji początku i zakończenia sesji, utrudnienie audytu,
- brak modułu `pam_wheel.so` - każdy będzie mógł spróbować użyć `su`, 
- brak modułu `pam_listfile.so` nie jest krytyczna, 
- brak `pam_time.so` - nie jest krytyczne,
- brak `pam_access.so` - nie jest krytyczne,
Powyższa konfiguracja jest niebezpieczna. 

Zalecenia:  
- należy załączyć pliki common i ewentualnie dopisać w pliku `/etc/pam.d/su` odpowiednie moduły personalizowane dla tej aplikacji, zgodnie z polityką bezpieczeństwa,
- dodać moduł `pam_wheel.so` z opcją `use_uid`,
- opcjonalnie jeśli to konieczne dodać moduł `pam-listfile.so` pamiętając by nie używać opcji onerr=succeed, 
- opcjonalnie dodać moduły `pam_time.so`, `pam_access.so`,
- należy pamiętać, że plik jest czytany przez aplikację z góry do dołu, a kolejność wpisów jest kluczowa. 
- poprawna konfiguracja:  
	auth sufficient pam_rootok.so  
	auth required pam_wheel.so  use_uid
	@include common-auth  
	@include common-account  
	@include common-session  