# /etc/pam.d/su

## Cel laboratorium 
Zapoznanie się ze strukturą pliku /etc/pam.d/su oraz jego rolą w systemie. 

## Czym jest su
Jest aplikacja, która pozwala na zmianę bieżącego użytkownika w terminalu bez wylogowywania się. Przykłady:
- `su - root` - przełącza użytkownika na root z jego własnym środowiskiem,
- `su - user` - przełącza użytkownika na user z jego środowiskiem,
- `su` - domyślnie przełącza na root. 
Należy podkreślić różnicę pomiędzy sposobami używania polecenia `su`:
- `su root` - zmienia użytkownika ale pozostawia obecne środowisko, co może stanowić zagrożenie używania błędnych poleceń z nieodpowiedniego katalogu, 
- `su - root` - ładuje pełne środowisko docelowego konta, bezpieczniejsze rozwiązanie. 

## Czym jest plik /etc/pam.d/su
Jest to plik obsługujący aplikację `su`, mówi w jaki sposób następuje przełączanie użytkownika, korzysta z modułów w typach PAM (auth, account, session). Zwykle zawiera dołączone pliki common za pomocą `@include`. Struktura wpisów w pliku:  
	`[typ] [flaga] [moduł] [opcje]`  
Plik może także zawierać ustawienia niestandardowe wprowadzone przez dodatkowe moduły, które nie są ujęte w plikach common, lub jest potrzeba zmiany ustawień danego modułu na potrzeby aplikacji `su` (np. pam_limits.so).

## Standardowe moduły 
Poza załączonymi plikami common, plik może zawierać moduły modyfikujące te z plików common lub wprowadzające zupełnie nowe, specyficzne dla tej aplikacji zabezpieczenia. Wśród typowych modułów w tym pliku wyróżnić można:
- `pam_rootok.so` - zwykle z flagą `sufficient` w typie auth. Gdy moduł zwróci success (użytkownik jest rootem) pomija wszystkie dalsze moduły w fazie auth. Dzięki temu root nie musi podawać hasła. Gdyby była flaga `required` root i tak by przeszedł, ale PAM pytałoby o hasło. Moduł ten nie występuje w innych typach PAM.
- `pam_wheel.so use_uid` - z flagą `required` to moduł, który pozwala na dostęp do polecenia `su` wyłącznie użytkownikom z grupy wheel (domyślnie). Można zmienić grupę użytkowników na inną za pomocą opcji dla modułu:
	`auth required pam_wheel.so group=admin use_uid`
To sprawia, że grupa wheel jest zastąpiona przez grupę admin. UWAGA: nie można w takim wpisie dodać wielu grup, moduł nie przyjmuje list. Nie można w prosty sposób zrobić 'grupa A lub grupa B' - dwa wpisy z required wymagają bycia w obu grupach. Teoretycznie można by jeden z tych modułów ustawić na `sufficient`, lecz to może zaburzać przepływ typu i nie jest bezpiecznym rozwiązaniem. Ten moduł występuje wyłącznie w typie auth,
- `pam_listfile.so` - to odpowiedź na powyższy problem - jest to moduł występujący z flagą `required` w typie auth, który pozwala na dostęp do polecenia `su` użytkownikom, grupom lub hostom ujętym w pliku `/etc/su/nazwa`. Bezpieczne opcje dla modułu w przypadku gdy chcemy dodać grupę, która może korzystać z polecenia `su`:  
`auth required pam_listfile.so  item=group sense=allow file=/etc/su/nazwa onerr=fail apply=admin`  
Co oznaczają opcje zastosowane we wpisie:
	- item=group - określa jaki typ obiektu sprawdzamy, w tym przypadku grupę (może być ustawiony user w przypadku gdy mamy listę użytkowników, host gdy lista zawiera hosty),
	- sense=allow - jeśli użytkownik jest w grupie -> allow, 
	- file=/etc/su/nazwa - to ścieżka do pliku z listą grup (lub użytkowników/hostów, w zależności od item),
	- onerr=fail - jeśli nie ma pliku lub nie można go otworzyć ->  zwróć błąd,
	- apply=admin - tu określamy nazwę użytkownika, który jest docelowym użytkownikiem (czyli konta, na które chcemy się zalogować) - w tym przypadku konto admin. 
Plik `/etc/su/nazwa` trzeba utworzyć z odpowiednimi uprawnieniami - 640 (root może czytać i pisać, grupa czytać, others nic nie mogą). Uprawnienia są ważne, jeśli nie są poprawnie ustawione, może to skutkować eskalacją uprawnień. W pliku są zapisane wyłącznie nazwy grup (użytkowników, hostów - zależnie od ustawienia item). Każda nazwa w nowej linii, należy unikać pustych linii, białych znaków i '#' na początku linii. 
- `pam_limits.so` - moduł nakładający limity zasobów (CPU, ilość procesów, RAM, itp), zwykle jest stosowany w plikach common, jednak zdarza się, że moduł jest powtórzony w pliku `su` by zabezpieczyć  poprawność nałożenia limitów lub zmodyfikować wartości określone w plikach common. Moduł ten występuje wyłącznie w typie session. 
- `pam_time.so` - zwykle z flagą `required`, rzadko `optional`. Moduł pozwala ograniczyć użycie polecenia `su` tylko do godzin pracy, to nie jest krytyczny moduł, ale przydatny,
- `pam_access.so` - z flagą `required`. Moduł ten ogranicza użycie polecenia `su` do konkretnego hosta/TTY/użytkownika. 

## Monitoring 
Plik: `/var/log/auth.log`
- Udane `su` - su[12345]: pam_unix(su:session): session opened for user root by james(uid=1001)
- Nieudane (brak wheel): -  su[123456]: pam_wheel(su:auth): access denied for user john
- nieudane (złe hasło): - su[234234]: pam_unix(su:auth) authentication failure; logname=anna
- zamknięcie sesji: - su[23456] pam_unix(su:session): session closed for user root

## su vs sudo
Podstawowe różnice między aplikacjami:
- Hasło: 
	- `su` wymaga hasła docelowego użytkownika (np. roota),
	- `sudo` wymaga hasła własnego użytkownika.
- Co widać w logach (`/var/log/auth.log`):
	- `su` zostawia wpis session opened - wiadomo, że ktoś przełączył użytkownika, ale nie wiadomo, co potem robił, 
	- `sudo` loguje konkretne polecenia, np. user ran command /bin/systemctl restart nginx.
- Ryzyko bezpieczeństwa:
	- `su` jest bardziej ryzykowne, bo wymaga znajomości hasła roota (często współdzielonego),
	- `sudo` jest bezpieczniejsze - każde konto ma własne hasło, a uprawnienia są indywidualne. 
- Gdzie to jest skonfigurowane:
	- `su` -> `/etc/pam.d/su` oraz grupa wheel (kto może go używać),
	- `sudo` -> `/etc/pam.d/sudo` oraz plik konfiguracyjny `/etc/sudoers` - czyli kto i na jakich warunkach może wykonać). 
Dla SOC:
- `sudo` jest bezpieczniejsze - daje audyt i kontrolę, 
- `su` powinien być używany tylko w wyjątkowych sytuacjach (awaryjnie), a najlepiej całkowicie ograniczony przez `pam_wheel.so` lub `pam_listfile.so`.

## Wnioski bezpieczeństwa 
- brak modułu `pam_wheel.so` - każdy użytkownik może próbować `su` na root. Jak wykryć: `grep pam_wheel /etc/pam.d/su` - jeśli nic nie zostanie znalezione, modułu brak,
- brak opcji use_uid w module `pam_wheel.so` - wtedy sprawdzana jest grupa docelowego użytkownika, co mija się z celem (bo root i tak jest we wszystkich grupach), ta opcja pozwala na sprawdzenie użytkownika wywołującego `su`,
- `pam_listfile.so` z opcją onerr=succeed - jeśli plik z listą dozwolonych użytkowników nie istnieje lub nie można go otworzyć, PAM i tak zezwala na `su`. To bardzo niebezpieczne. 
- plik z listy (np. /etc/su/allowed)  ze złymi uprawnieniami - jeśli plik jest czytelny lub zapisywalny przez zwykłych użytkowników, mogą oni dodać siebie do listy uprawnionych lub zobaczyć, kto ma dostęp. Uprawnienia pliku można sprawdzić za pomocą polecenie `ls -l /etc/su/allowed`,
- dodanie niepowołanego użytkownika do pliku listy - ktoś, kto nie powinien, zyskuje możliwość użycia `su`. Należy monitorować zmiany pliku (np. auditd) i alertować przy każdej modyfikacji, 
- konto systemowe (np. www-data) w grupie wheel - jeśli atakujący przejmie konto systemowe, może użyć `su - root`, jeśli konto jest w grupie wheel. Należy weryfikować kto należy do tej grupy za pomocą polecenia `getent group wheel`,
- `su` używane po godzinach pracy - potencjalny atak lub nieautoryzowana aktywność, można monitorować za pomocą polecenia `grep "session opened" /var/log/auth.log` i sprawdzić godziny. 

## Case study  
```
auth sufficient pam_rootok.so  
auth required pam_unix.so  
account required pam_unix.so  
session required pam_unix.so  
```
Analiza:  
- brak załączonych plików common co oznacza, że poszczególne typy poza sprawdzeniem użytkownika i hasła nie są w żaden sposób zabezpieczone a audyt nie będzie pełny, zagrożenia z tego wynikające to:
	- brak modułów ochronnych w typie `auth`:
		- `pam_faillock.so` - nie są zliczane próby logowania, brak ochrony przed brute force, 
		- `pam_faildelay.so` - brak opóźnienia po nieudanej próbie logowania może ułatwiać atak brute force, 
	- brak modułów zabezpieczających system w typie session:
		- `pam_env.so` - środowisko nie jest ustawione, istnieje możliwość podmiany PATH (podmiana binariów), LD_PRELOAD (ładowanie własnych bibliotek przed systemowymi - MitM, phishing), LD_LIBRARY_PATH (podmiana bibliotek), TMPDIR (podmiana katalogu plików tymczasowych - mogą trafiać do publicznego katalogu), LANG (podmiana języka logowania może powodować błędy w interpretacji) i wiele innych,
		- `pam_limits.so` - brak tego modułu oznacza, że na sesję nie zostały nałożone żadne ograniczenia zasobów (CPU, RAM, procesy, itp). to stwarza zagrożenie DoS (lokalny), fork bomb (zabicie systemu dużą ilością procesów), core dump (zrzuty wrażliwej pamięci),
	- brak modułów logujących dane sesji w typie session:
		- `pam_loginuid.so` - nie zostanie zarejestrowany oryginalny UID użytkownika (AUID), dzięki któremu można wykryć jaki użytkownik użył polecenia z sudo, bez tego będzie widoczne jedynie że root użył polecenia, ale nie kto go logował się jako root,
		- `pam_systemd.so` - bez niego nie zostanie zalogowany początek i koniec sesji a to utrudnia audyt i analizę incydentów,
	- brak pozostałych modułów typowych dla sesji nie jest krytyczne a moduły nie są niezbędne, chyba, że wymaga ich polityka bezpieczeństwa firmy. 
- brak modułu `pam_wheel.so` oznacza, że każdy będzie mógł spróbować użyć `su`, atakujący gdy przejmie konto ma cały wachlarz możliwości, zwłaszcza, że w obecnej konfiguracji nie ma plików common (np. brak modułów ochronnych przed brute force),
- brak modułu `pam_listfile.so` nie jest krytyczny, ponieważ moduł używany jest gdy chcemy dodać grupę lub użytkowników mających prawo do używania polecenia `su`,
- brak `pam_time.so` - również nie krytyczny lecz bywa przydatny, pozwala ograniczyć użycie `su` do czasu godzin pracy, 
- brak `pam_access.so` - nie krytyczny, ogranicza użycie `su` na podstawie hosta/TTY/użytkownika. 
Powyższa konfiguracja jest bardzo niebezpieczna!  
  
Zalecenia:  
- należy załączyć pliki common i ewentualnie dopisać w pliku `/etc/pam.d/su` odpowiednie moduły personalizowane dla tej aplikacji, zgodnie z polityką bezpieczeństwa,
- dodać moduł `pam_wheel.so` z opcją `use_uid`,
- opcjonalnie jeśli to konieczne dodać moduł `pam_listfile.so` pamiętając by nie używać opcji onerr=succeed, 
- opcjonalnie dodać moduły `pam_time.so`, `pam_access.so`,
- należy pamiętać, że plik jest czytany przez aplikację z góry do dołu, a kolejność wpisów jest kluczowa. 
- poprawna konfiguracja:  
```
	auth sufficient pam_rootok.so  
	auth required pam_wheel.so  use_uid
	@include common-auth  
	@include common-account  
	@include common-session  ```