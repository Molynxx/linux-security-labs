# Lab: /etc - analiza plików konfiguacyjnych systemu

## Cel
Celem tego laboratorium było zapoznanie się z plikami konfiguracyjnymi w katalogu '/etc' które mają znaczenie dla bezpieczeństwa systemu, w szczególności:
- '/etc/passwd'
- '/etc/group'
- '/etc/shadow'
- '/etc/sudores'
- '/etc/shh/sshd_config'

Analiza tych plików pozwala zrozumieć:
- jakie konta istnieją w systemie, 
- które mają dostęp interaktywny,
- jakie grupy i uprawnienia mają użytkownicy,
- które konta mogą stanowić potencjalen zagrożenie

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
- Sprawdzono ostanie logowania użytkowników za pomocą polecenia last

### Obserwacje
- Występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi - UID < 1000, powłoka /user/bin/nologin lub /bin/false,
	- użytkownicy interaktywni - UID > 1000, powłoka /bin/bash, /bin/zsh lub /bin/sh, katalog domowy w /home.
- Niektóre konta systemowe mogą mieć interaktywny shell (np. polstgres) - to są wyjątki.

### Wnioski bezpieceństwa
- Każdy użytkownik systemowy z możliwością interaktywnego logowania wymaga weryfikacji.
- Należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką, 
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- Użytkownicy z UID >= 1000 poqinni mieć powłokę interaktywną i katalog domowy; każde odstępstwo wymaga uwagi.