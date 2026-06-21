# User management commands

## Cel laboratorium 
Celem tego laboratorium było zrozumienie podstawowych poleceń do zarządzania użytkownikami w Linux oraz ich znaczenia dla bezpieczeństwa systemu. 

## useradd
Tworzy nowego użytkownika w systemie, dodaje wpis do:
- `/etc/passwd` - login, marker x, UID, GUID, opis, katalog domowy, shell,
- `/etc/shadow` - tworzy wpis dla hasła użytkownika (domyślnie bez hasła - w pliku pojawia się: `!!`, co oznacza, że konto nie ma jeszcze ustawionego hasła, useradd nie ustawia hasła),
- `/etc/group` - wpis grupy prywatnej użytkownika (jeśli jest tworzona),
Przykład:
	`useradd john`
	`useradd -m -s /bin/bash john` 
	`useradd -G sudo,developers testuser`
Ważne opcje:
- `-m` - tworzy katalog domowy (`/home/john`), gdy brak `-m` katalog domowy nie zostanie utworzony i użytkownik nie będzie miał domyślnej ścieżki do `/home`,
- `-s` - ustawia powłokę (np. `/bin/bash`),
- `-u` - ustawia UID,
- `-G` - dodaje do grupy dodatkowej,
- `-aG` - dodaje do grupy dodatkowej nie usuwając użytkownika z dotychczasowych grup.
Perspektywa bezpieczeństwa:
- brak katalogu domowego dla konta z powłoką interaktywną może wskazywać na backdoor (np. atakujący tworzy konto z `/bin/bash`. ale bez `/home`, żeby nie rzucało się w oczy). Ponadto brak katalogu `/home` może powodować problemy z sesją - problem z perspektywy funkcjonalności, 
- ręczne ustawianie UID wiąże się z ryzykiem konfliktów. 

## usermod
Modyfikuje istniejącego użytkownika.
Przykłady:
	`usermod -aG sudo john`
	`usermod -l newuser olduser`
	`usermod -d /nowy/home -m john`
Ważne opcje:
- `-aG` - dodaje do grupy bez usuwania z innych,
- `-l` - zmiana nazwy użytkownika, nie zmienia katalogu domowego automatycznie, nie zmienia nazwy grupy, wymaga dodatkowych kroków (np. `-d`, `-m`) dla spójności systemu, 
- `-d` - zmiana katalogu domowego, 
- `-m` - przenosi istniejące dane ze starego katalogu domowego do nowego. 
Perspektywa SOC:
- dodanie do sudo to ryzyko eskalacji uprawnień, 
- zmiana grup to ryzyko zmiany dostępu do zasobów. 

## userdel 
Usuwa użytkownika z systemu. 
Przykład:
	`userdel john`
Ważne opcje:
- `-r` - usuwa katalog domowy i pliki.
Perspektywa bezpieczeństwa:
- brak `-r` sprawia, że pozostają pliki (możliwe artefakty po ataku). Ponadto, pozostawienie plików może być celowe - atakujący usuwa konto, żeby ukryć wektor wejścia, ale zostawia pliki (np. backdoor w `/tmp`, klucz SSH w `~/.ssh/authorized_keys`).

## passwd
Zmienia hasło użytkownika.
Przykład:
	`passwd john`
Perspektywa bezpieczeństwa:
- hasła trafiają do `/etc/shadow`,
- współpraca z PAM (np. `pam_pwquality.so` - może wymagać silnego hasła).

## chsh
Zmienia powłokę użytkownika.
Przykład:
	`chsh -s /bin/bash john`
Perspektywa bezpieczeństwa:
- ustawienie `/usr/sbin/nologin` = blokada logowania,
- podejrzane powłoki to zawsze Red Flag.

## id 
Wyświetla UID, GID i grupy użytkownika.
Przykład:
	`id john` 
Perspektywa bezpieczeństwa:
- szybka identyfikacja uprawnień użytkownika.

## groups
Pokazuje grupy użytkownika. 
Przykład:
	`groups john` 
Perspektywa bezpieczeństwa:
- analiza dostępu do zasobów.

## Wnioski
- zarządzanie użytkownikami = zarządzanie dostępem, 
- zmiany w użytkownikach wpływają na bezpieczeństwo systemu, 
- każda zmiana powinna być monitorowana i audytowana. 

## Case study

Scenariusz:  
Atakujący z uprawnieniami root/sudo dodał konto hidden z UID 0 za pomocą `useradd -u 0 -o hidden`.   

Analiza i remediacja:  
- należy sprawdzić za pomocą polecenia `cat` pliki:
	- `/etc/passwd` - sprawdzenie UID, jaka jest powłoka dla konta, czy jest katalog domowy. Dodatkowa powłoka z UID 0 to zawsze jest podejrzane, podobnie jak brak katalogu domowego oraz powłoka interaktywna,
	- `/etc/shadow` - czy występuje hash hasła, jaka jest ważność hasła i wartość w inactive. Długa ważność hasła, wysoka wartość w inactive oraz brak hasha hasła powinny budzić podejrzenia, 
	- `/etc/group` - należy sprawdzić do jakich grup należy użytkownik hidden i jaka jest jego primary group, 
	- `/etc/sudoers` i `/etc/sudoers.d/` - czy dla użytkownika zostały dodane reguły w ustawianiach sudoers,
- zmian w tych plikach może dokonać root lub użytkownik posiadający sudo, należy więc ustalić, kiedy i z którego konta wprowadzono zmiany w tych plikach, to może pomóc w odnalezieniu skompromitowanego konta, 
- sprawdzić logowania na podejrzanym koncie, kiedy wystąpiły i skąd (`last`, `lastb`, logi SSH), 
- należy przeprowadzić pełny audyt działań podejrzanego konta, oraz utworzonego konta hidden od czasu jego utworzenia, 
- należy usunąć konto hidden, oraz konta które zostały przez niego utworzone (jeśli jakieś powstały - należy to sprawdzić), 
- należy usunąć wszystkie pliki użytkownika hidden i stworzonych przez niego kont (w tym pliki w `/tmp`, `/var/tmp`, `/dev/shm`), 
- należy sprawdzić plik `~hidden/.ssh/authorized_keys` i usunąć wpisy,
- sprawdzić, czy działają jakieś usługi dla konta hidden (i ewentualnych utworzonych przez to konto kont), jeśli występują należy je zakończyć - to może być reverse shell lub cryptominer,
- należy usunąć konto hidden oraz konta przez niego stworzone (jeśli występują), 
- należy usunąć wpisy z sudoers i sudoers.d jeśli występują dla tych kont,
- należy zresetować hasła dla root, oraz wszystkich kont z prawami sudo i su.   
Po przeprowadzeniu wszystkich kroków należy wdrożyć monitoring (`auditd`) na plikach `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, aby wykryć podobne próby w przyszłości. 
