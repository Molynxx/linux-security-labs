# /etc/pam.d/lightdm

## Cel laboratorium 
Zapoznanie się z rolą pliku lightdm w systemie Linux.

## Czym jest aplikacja Lightdm
Jest to menedżer logowania graficznego (Display Manager). To ekran wyświetlający się po uruchomieniu komputera, zawierający pola do wpisania loginu i hasła. Jak to działa:
- uruchamia się serwer graficzny (X11 lub Wayland),
- pokazuje się ekran logowania, 
- następuje uwierzytelnianie przez PAM,
- uruchamia się sesja użytkownika (środowisko graficzne).
To ważne dla SOC, ponieważ logowanie graficzne jest uwierzytelniane przez PAM, tak samo jak SSH, login czy su. Trzeba wiedzieć, że atak może nadejść przez ekran logowania (fizycznie lub przez pulpit zdalny). 
Aplikacja ma swój plik konfiguracyjny `/etc/lightdm/lightdm.conf`. Kluczowe opcje pliku:
- greeter-session - określa, który greeter jest używany, np. lightdm-gtk-greeter,
- user-session - domyślnie środowisko graficzne, np. xfce,
- autologin-user - zagrożenie - jeśli wartość nie jest pusta, użytkownik loguje się automatycznie bez hasła,
- autologin-user-timeout - opóźnienie przed automatycznym logowaniem,
- greeter-show-manual-login - jeśli opcja nie jest ustawiona na false, umożliwia ręczne wpisanie nazwy użytkownika w tym root. 

## Czym jest /etc/pam.d/lightdm
Jest to plik konfiguracyjny PAM dla aplikacji LightDm, zawiera załączone pliki common (`common-auth`, `common-account`, `common-session`, `common-password`) oraz ewentualnie dodatkowe moduły specyficzne dla tej aplikacji (rzadko). Wśród nich wyróżnić można:
- `pam_gnome-keyring.so` - (auth, session) 0 odblokowuje klucze GNOME po zalogowaniu, 
- `pam_ecryptfs.so` - (session) - montuje zaszyfrowany katalog domowy, 
- `pam_systemd.so`- (session) - jeśli nie jest ujęty w `common-session`, tworzy sesję systemd - potrzebne dla loginctl,
- `pam_xdg.so` - (session) - ustawia środowisko XDG *katalogu pulpitu, dokumentów, itp).
	Nie należy tutaj mylić modułów `pam_xdg.so` i `pam_env.so`, ponieważ oba moduły odpowiadają za coś innego:
	- `pam_env.so` - odpowiada za ustawienie zmiennych środowiskowych na podstawie plików konfiguracyjnych (PATH, LD_LIBRARY_PATH, LANG, itp),
	- `pam_xdg.so` - określa gdzie programy powinny przechowywać swoje pliki. Pomaga w systemach bez systemd, w systemach z systemd jest zbędny. 
UWAGA: żeby blokować logowanie root  przez środowisko graficzne, można umieścić też wpis:  
	`auth required pam_succeed_if.so user !=root quiet_success`  

## Gdzie znajdują się logi 
- `/var/log/auth.log` - nieudane i udane logowania przez PAM,
- `/var/log/lightdm.log` - szczegóły techniczne LightDm (uruchamianie serwera X, wybór środowiska, błędy greetera (okno logowania)).

## Przykładowe wpisy w logach:
- `auth.log` :
	-  lightdm: pam_unix(lightdm:auth): authentication failure: logname= uid=1001,
	-  lightdm: pam_unix(lightdm:session): session opened for user alice. 
- `lightdm.log`:
	- [+0.00s] DEBUG: greeter session pid=1234,
	- [+0.05s] DEBUG: user alice authenticated,
	- [+0.10s] DEBUG: session pid=5678 started for user alice.

## Wnioski bezpieczeństwa
- Należy weryfikować, czy wszystkie pliki common mają poprawnie ustawione moduły, np. w przypadku braku `pam_faillock.so` w auth z odpowiednimi opcjami, nie nastąpi zliczenia prób ani blokada logowania po przekroczeniu limitu nieudanych prób, a to ułatwia brute force, 
- należy sprawdzać, czy logowanie root przez GUI jest zablokowane w pliku konfiguracyjnym `/etc/lightdm/lightdm.conf` lub czy jest blokowane w pliku `/etc/pam.d/lightdm`.
- istotna jest weryfikacja czy plik `/etc/pam.d/lightdm` nie zawiera dodatkowych nieznanych modułów typu `pam_custom.so`,
- jeśli logowanie zdalne nie jest potrzebne, lepiej je wyłączyć w pliku konfiguracyjnym aplikacji,
- należy sprawdzać, czy użytkownicy  nie mają ustalonej opcji autologowania w pliku konfiguracyjnym. 

## Case study - LightDm w systemie produkcyjnym 
Kontekst:
Firma ma stację roboczą Ubuntu 22.04 (z Xfce). Na stacji pracuje dział księgowości- użytkownicy anna, jan, admin_ksi (ten ma sudo). Polityka bezpieczeństwa wymaga:
- brak autologowania, 
- logowanie root przez GUI wyłączone, 
- brak logowania jako gość, 
- ukryta lista użytkowników na ekranie logowania, 
- blokada po 5 nieudanych próbach logowania. 

Stan faktyczny:  
- plik `/etc/lightdm/lightdm.conf`:    
	[seat:*]  
	autologin-user=anna  
	autologin-user-timeout=0  
	allow-guest=true  
	greeter-hide-users=false  
	greeter-show-manual-login=true   
- plik `/etc/pam.d/lightdm`:  
	@include common-auth  
	@include common-account  
	@include common-session  
- plik `/etc/pam.d/common-auth`:  
	auth required pam_faillock.so preauth  
	auth [success=1 default=ignore] pam_unix.so   
	auth required pam_faillock.so authfail  
	auth sufficient pam_faillock.so authsucc  
- logi z `/var/log/auth.log` (fragment z ostatniej nocy):   
	Mar 22 03:15:01 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:05 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:09 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:13 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:17 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:21 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=anna  
	Mar 22 03:15:25 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=admin_ksi  
	Mar 22 03:15:29 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=admin_ksi  
	Mar 22 03:15:33 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=admin_ksi  
	Mar 22 03:15:37 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=admin_ksi  
	Mar 22 03:15:41 hist lightdm: pam_unix(lightdm:auth): authentication failure; user=admin_ksi  
	Mar 22 03:15:45 hist lightdm: pam_unix(lightdm:session): session opened for user admin_ksi  

Analiza:  
- polityka bezpieczeństwa  a konfiguracja pliku .conf:
	- brak autologowania - nie spełniony, ponieważ istnieje opcja autologin-user=anna - możliwość automatycznego logowania bez hasła dla konta anna, 
	- logowanie root przez GUI wyłączone - nie spełniony, ponieważ greeter-show-manual-login=true, istnieje możliwość logowania root, 
	- brak logowania jako gość - nie spełniony, poniważ allow-guest=true oznacza, że opcja logowania jako gość jest włączona, 
	- ukryta lista użytkowników na ekranie logowania - nie spełniony, ponieważ greeter-hide-users=false, co oznacza, że ukrywanie listy jest wyłączone, 
	- blokada po 5 nieudanych próbach logowania - nie spełniony - moduły w typie auth są ustawione prawidłowo, jednak z logów wynika, że prób na jednego użytkownika było 6. Przyczyna -> niewłaściwa konfiguracja pliku `/etc/pam.d/security/faillock.conf` (brak wpisu deny=5),
- zawartość pliku `/etc/pam.d/lightdm` ustawiona poprawnie, zawiera wszystkie pliki common, a plik common-auth zawiera prawidłowo skonfigurowany moduł `pam_faillock.so`. Jeśli common-account i common-session zawierają wszystkie niezbędne moduły, nie ma potrzeby dodawania specyficznych dla aplikacji modułów, 
- z logów wynika, że nastąpił nocny atak brute force, próby logowania faillock ustawione są na 6, po tym limicie konto jest zablokowane i tak stało się dla konta anna. Dla konta admin_ksi 6 próba okazała się skuteczna, a ponieważ użytkownik ma prawa sudo, jest to krytycznie niebezpieczny incydent. 

## Sugerowane działania
- polityka bezpieczeństwa:
	- zamienić autologin-user na pustą wartość, 
	- zmienić greeter-show-manual-login=true na wartość false,
	- zmienić allow-guest=true na wartość false, 
	- zmienić greeter-hide-user=false na wartość true, 
	- zmienić w pliku `/etc/pam.d/security/faillock.conf` limit dozwolonych, nieudanych prób logowania na wartość 5. 
	Powyższe działania pozwolą spełnić politykę bezpieczeństwa przedsiębiorstwa.
- sprawdzić kompletność modułów w plikach common-account i common-session,
- przeprowadzić audyt dla konta admin_ksi, sprawdzić przeprowadzone na tym koncie działania od momentu uzyskania dostępu, zmienić hasło dla konta, wycofać wszystkie nieautoryzowane zmiany wprowadzone do systemu przez to konto.
- rozważyć dodanie do typu auth modułu `pam_faildelay.so`, by po nieudanym logowaniu opóźnić kolejną próbę, celem zwiększenia zabezpieczenia przed brute force. 