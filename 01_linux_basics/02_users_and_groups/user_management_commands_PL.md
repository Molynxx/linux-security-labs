# User management commands 

## Cel 
Celem laboratorium było zrozumienie podstawowych poleceń do zarządzania użytkownikami w Linux oraz ich znaczenia dla bezpieczeństwa systemu.

## useradd
Tworzy nowego użytkownika w systemie, dodaje wpis do:
- `/etc/passwd` - login, marker x, UID, GUID, katalog domowy, shell, 
- `/etc/shadow` - tworzy wpis dla użytkownika ( domyślnie bez hasła - konto zablokowane: `!` lub `*`, useradd nie ustawia hasła),
- `/etc/group` - wpis grupy prywatnej użytkownika (jeśli jest tworzona).
Przykład:
	`useradd  john`
	`useradd -m -s /bin/bash john`
	`useradd -G sudo, developers testuser`
Ważne opcje:
- `-m` - tworzy katalog domowy,
- `-s` - ustawia powłokę,
- `-u` - ustawia UID,
- `-G` - dodaje do grupy dodatkowej,
- `-aG` - dodaje do grupy dodatkowej nie usuwając pozostałych.
Perspektywa bezpieczeństwa:
- brak katalogu domowego może powodować problemy z sesją,
- ręczne ustawiania UID stanowi ryzyko konfliktów. 

## usermod
Modyfikuje istniejącego użytkownika
Przykład:
	`usermod -aG sudo john`
	`usermod -l newuser olduser`
	`usermod -d /nowy/home - m username`
Ważne opcje:
- `-aG` - dodaje do grupy bez usuwania z pozostałych, 
- `-l` - zmiana nazwy użytkownika,
- `-d` - zmiana katalogu domowego,
- `-m` - przenosi istniejące dane ze starego katalogu do nowego.
Perspektywa bezpieczeństwa:
- dodanie do sudo - eskalacja uprawnień,
- zmiana grup - zmiana dostępu do zasobów.

## userdel
Usuwa użytkownika z systemu.
Przykład:
	`userdel john`
Ważne opcje:
- `-r` - usuwa katalog domowy i pliki.
Perspektywa bezpieczeństwa:
- brak `-r` - pozostają pliki, możliwe artefakty po ataku.

## passwd 
Zmienia hasło użytkownika.
Przykład:
	`passwd john`
Perspektywa bezpieczeństwa:
- hasła trafiają do `/etc/shadow`,
- współpraca z PAM (np. `pam_pequality.so` - może wymagać silnego hasła).

## chsh
Zmienia powłokę użytkownika.
Przykład:
	`chsh -s /bin/bash john`
Perspektywa bezpieczeństwa:
- ustawienie `/usr/sbin/nologin` - blokada logowania,
- podejrzane powłoki = Red Flag.

## id 
Wyświetla UID, GUID i grupy użytkownika.
Przykład:
	`id john`
Perspektywa bezpieczeństwa:
- szybka identyfikacja użytkownika.

## groups 
Pokazuje grupy użytkownika.
Przykład:
	`groups john`
Perspektywa bezpieczeństwa:
- analiza dostępu do zasobów.

## Wnioski
- zarządzenie użytkownikami = zarządzanie dostępem,
- zmiany w użytkownikach wpływają bezpośrednio na bezpieczeństwo systemu, 
- każda zmiana powinna być monitorowana i audytowana. 
