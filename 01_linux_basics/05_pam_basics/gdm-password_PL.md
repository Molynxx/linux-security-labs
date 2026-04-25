# /etc/pam.d/gdm-password

## Cel laboratorium
Zrozumienie, w jaki sposób PAM obsługuje logowanie graficzne w menedżerze GDM (GNOME Display Manager) – domyślnym w Ubuntu, Fedorze, Debianie z GNOME.

## Czym jest aplikacja GDM
To menedżer logowania do środowiska graficznego GNOME. Aplikacja posiada swój plik konfiguracyjny w katalogu `/etc/gdm3/custom.conf` (Ubuntu, Debian) lub `/etc/gdm/custom.conf` (RHEL, Fedora). 

## Czym jest plik /etc/pam.d/gdm-password
To plik konfiguracyjny PAM dla aplikacji GDM. Zawiera załączone pliki common (common-auth, common-account, common-session, common-password) oraz opcjonalnie moduły specyficzne dla tej aplikacji, np.:
- `pam_gdm.so` - komunikacja pomiędzy GDM i PAM,
- `pam_gnome_keyring.so` - odblokowuje klucze GNOME,
- `pam_systemd.so` - jeśli jest w common-session (a powinien), nie dodajemy go ponownie,

## Gdzie znajdują się logi 
- `/var/log/auth.log` - nieudane i udane próby logowania przez PAM,
- `/var/log/gdm/gdm.log` (Fedora) lub `/var/log/gdm3/gdm.log` (Ubuntu) - szczegóły techniczne GDM (uruchomienie sesji, greeter, błędy).

## Przykładowe wpisy w logach
- auth.log:
	- gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna
	- gdm-password: pam_unix(gdm-password:session): session opened for user anna
- gdm.log:
	- gdm-launch-environment][1234]: pam_unix(gdm-launch-environment:session): session opened for user Debian-gdm by (uid=0)
	- gdm-simple-slave][1234]: GdmSlave: Greeter started on display :0
	- gdm3][2345]: GdmSession: Emitting 'session-started' signal with pid '3456'

## Wnioski bezpieczeństwa
- plik powinien zawierać pliki common, należy przy tym sprawdzić kompletność tych plików, w tym ustawienia modułu pam_faillock.so. Poprawna konfiguracja:
	@include common-auth
	@include common-account
	@include common-session
	@include common-password
- należy weryfikować zapisy w pliku konfiguracyjnym GDM (custom.conf) czy nie zawiera ustawień opcji mogących powodować zagrożenie, 
- należy monitorować zapisy w pliku `/etc/pam.d/gdm-password` pod kątem nieznanych modułów i nieautoryzowanych zmian, 
- monitoring logów pozwala na wykrycie potencjalnych zagrożeń. 

## Case study – GDM w środowisku produkcyjnym
Kontekst  
Firma posiada stację roboczą z Ubuntu 22.04 (GNOME, więc GDM).  
Stacja jest używana przez zespół danych – użytkownicy: anna (analityk), jan (inżynier danych), admin_ksi (ma sudo).  

Polityka bezpieczeństwa firmy wymaga:  
Brak autologowania  
Logowanie root przez GUI wyłączone  
Brak możliwości drugiej sesji graficznej dla tego samego użytkownika  
Blokada konta po 5 nieudanych próbach logowania  
Logowanie graficzne tylko lokalnie (zdalne wyłączone)  

Stan faktyczny:  
Plik /etc/gdm3/custom.conf:  
[daemon]  
AutomaticLoginEnable=true  
AutomaticLogin=anna  
TimedLoginEnable=true  
TimedLogin=anna  
TimedLoginDelay=5  
DisableMultipleSessions=false  
[security]  
AllowRoot=true  
[xdmcp]  
Enable=true  

Plik /etc/pam.d/gdm-password:  
@include common-auth  
@include common-account  
@include common-session  
@include common-password  

Plik /etc/pam.d/common-auth (fragment odpowiedzialny za blokadę):  
auth required pam_faillock.so preauth  
auth [success=1 default=ignore] pam_unix.so  
auth required pam_faillock.so authfail  
auth sufficient pam_faillock.so authsucc  

Logi z /var/log/auth.log (fragment z ostatniej nocy):  

Mar 23 02:10:01 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:05 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:09 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:13 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:17 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:21 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=anna  
Mar 23 02:10:25 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=admin_ksi  
Mar 23 02:10:29 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=admin_ksi  
Mar 23 02:10:33 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=admin_ksi  
Mar 23 02:10:37 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=admin_ksi  
Mar 23 02:10:41 host gdm-password: pam_unix(gdm-password:auth): authentication failure; user=admin_ksi  
Mar 23 02:10:45 host gdm-password: pam_unix(gdm-password:session): session opened for user admin_ksi  
Dodatkowe informacje z wywiadu:  
Stacja ma włączone logowanie przez VNC (zdalny pulpit)  
admin_ksi zgłosił, że w nocy nie pracował  
anna mówi, że ma trudności z logowaniem od kilku dni.  

Analiza:  
- Polityka bezpieczeństwa:  
	- `AutomaticLoginEnable=true` - nie spełniony, autologowanie jest włączone,
	- `AutomaticLogin=anna` - nie spełniony, ustawione zostało konto do logowania automatycznego,
	- `TimedLoginEnable=true` - nie spełniony, niepotrzebny przy opcji wyłączonego automatycznego logowania, 
	- `TimedLogin=anna` - nie spełniony, niepotrzebny przy opcji wyłączonego automatycznego logowania, 
	- `TimedLoginDelay=5` - nie spełniony, niepotrzebny przy opcji wyłączonego automatycznego logowania, 
	- `DisableMultipleSessions=false` - nie spełniony, nie blokuje kolejnej sesji graficznej użytkownika
	- `AllowRoot=true` - nie spełniony - logowanie roota przez GUI jest włączone, 
	- [xdmcp]
	  `Enable=true` - nie spełniony - dostęp zdalny jest włączony,
	- Blokada konta po 5 nieudanych próbach logowania - nie spełniony, z logów wynika, że blokada następuje dopiero po 6 próbie. Plik common-auth oraz plik gdm-password są skonfigurowane poprawnie -> problem zapewne leży w pliku konfiguracyjnym dla modułu `pam_faillock.so` - wpisany limit nieudanych prób został ustawiony na 6 zamiast na 5. 
- z logów wynika, że doszło do nocnego ataku brute force, 
	- najpierw zostało zaatakowane konto anna, po 6 nieudanej próbie zostało zablokowane przez moduł faillock, to dlatego anna ma trudności z zalogowaniem, 
	- w dalszej kolejności zaatakowano konto admin_ksi, w 6 próbie uzyskano dostęp i została otworzona sesja dla tego użytkownika, konto zostało skompromitowane.   

Sugerowane działania:  
- Polityka bezpieczeństwa:
	- `AutomaticLoginEnable=true` - zmienić na false,
	- `AutomaticLogin=anna` - zmienić wartość na pustą,
	- `TimedLoginEnable=true` - zmienić wartość na false, 
	- `TimedLogin=anna` - przy wyłączonym autologowaniu opcja nie działa, jednak dla bezpieczeństwa - ustawić na wartość pustą,  
	- `TimedLoginDelay=5` - niepotrzebny przy wyłączonym logowaniu, może pozostać jak jest, 
	- `DisableMultipleSessions=false` - zmienić na true,
	- `AllowRoot=true` - zmienić na false, 
	- [xdmcp]
	  `Enable=true` - zmienić na wartość false,
	- Blokada konta po 5 nieudanych próbach logowania - sprawdzić plik konfiguracyjny modułu `pam_faillock.so` w `/etc/security/faillock.conf` i ustawić prawidłową wartość dla limitu nieudanych prób logowania (deny=5).   
	Powyższe działania pozwolą spełnić politykę bezpieczeństwa firmy.
- konto admin_ksi zostało skompromitowane, zalecane natychmiastowa zmiana hasła użytkownika, pełny audyt konta lub ewentualne zablokowanie konta. 
- należy sprawdzić działania przeprowadzone na koncie admin_ksi od czasu uzyskania dostępu po ataku brute force i wycofać wszystkie nieautoryzowane działania. 
- sprawdzić konfigurację wszystkich plików konfiguracyjnych PAM dla aplikacji celem sprawdzenia czy atakujący nie pozostawił backdoora. 


