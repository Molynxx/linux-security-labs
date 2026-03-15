# Lab: /etc - analiza plików konfiguracyjnych systemu

## Cel
Celem tego laboratorium było zapoznanie się z plikami konfiguracyjnymi w katalogu '/etc' które mają znaczenie dla bezpieczeństwa systemu, w szczególności:
- '/etc/passwd'
- '/etc/group'
- '/etc/shadow'
- '/etc/sudoers'
- '/etc/ssh/sshd_config'

Analiza tych plików pozwala zrozumieć:
- jakie konta istnieją w systemie, 
- które mają dostęp interaktywny,
- jakie grupy i uprawnienia mają użytkownicy,
- które konta mogą stanowić potencjalne zagrożenie

## /etc/passwd
 
### Czym jest /etc/passwd
Plik '/etc/passwd' zawiera informacje o użytkownikach systemu:
- nazwa użytkownika, 
- hash hasła lub marker 'x',
- UID (User ID),
- GID (Group ID),
- katalog domowy,
- powłoka logowania (shell).

### Wykonane kroki
- Przejrzano plik poleceniem cat /etc/passwd,
- Sprawdzono ostatnie logowania użytkowników za pomocą polecenia last

### Obserwacje
- Występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi - UID < 1000, powłoka /user/bin/nologin lub /bin/false,
	- zakresy UID mogą się różnić w zależności od dystrybucji i konfiguracji systemu,
	- użytkownicy interaktywni - UID > 1000, powłoka /bin/bash, /bin/zsh lub /bin/sh, katalog domowy w /home.
- Niektóre konta systemowe mogą mieć interaktywny shell (np. postgres) - to są wyjątki.

### Wnioski bezpieczeństwa
- Każdy użytkownik systemowy z możliwością interaktywnego logowania wymaga weryfikacji.
- Należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką, 
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- Użytkownicy z UID >= 1000 powinni mieć powłokę interaktywną i katalog domowy; każde odstępstwo wymaga uwagi.
- UID 0 oznacza konto root - w systemie powinno istnieć tylko jedno konto z UID 0. Obecność dodatkowego konta z UID 0 może wskazywać na backdor lub nieautoryzowane konto.
- Konto systemowe z interaktywnym shellem (/bin/bash, /bin/zsh) wymaga weryfikacji, nawet jkeśli uUID <1000.