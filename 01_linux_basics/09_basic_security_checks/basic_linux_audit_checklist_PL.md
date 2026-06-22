# Podstawowa Checklista Audytu Bezpieczeństwa Linux

Cel: Szybka ocena stanu bezpieczeństwa systemu Linux. Lista poleceń i miejsc do sprawdzenia podczas rutynowego audytu lub wstępnej analizy.

## 1. Użytkownicy i konta

- Polecenie: `cat /etc/passwd`
  - Co sprawdza: Listę wszystkich użytkowników systemu.
  - Zagrożenia: Nietypowe nazwy użytkowników, dodatkowe konta z UID 0 (powinien być tylko root), konta bez ścieżki powłoki lub z nieprawidłową powłoką.
  - Prawidłowy wynik: Znane konta systemowe (root, daemon, bin itp.) oraz konta użytkowników, których się spodziewamy. UID 0 tylko dla root.

- Polecenie: `sudo cat /etc/shadow`
  - Co sprawdza: Hasła użytkowników (zahashowane).
  - Zagrożenia: Drugie pole puste (brak hasła) – umożliwia logowanie bez hasła.
  - Prawidłowy wynik: Każdy wpis ma drugie pole niepuste (zawiera hash lub `*`/`!` dla zablokowanych kont).

- Polecenie: `sudo awk -F: '($2==""){print $1}' /etc/shadow`
  - Co sprawdza: Konta bez hasła.
  - Zagrożenia: Logowanie bez hasła.
  - Prawidłowy wynik: Brak wyjścia (żadne konto nie ma pustego hasła).

- Polecenie: `sudo awk -F: '($3==0){print $1}' /etc/passwd`
  - Co sprawdza: Użytkowników z UID 0 (uprawnienia roota).
  - Zagrożenia: Dodatkowe konta z UID 0 mogą być backdoorem.
  - Prawidłowy wynik: Tylko `root`.

- Polecenie: `sudo passwd -S -a | grep "NP"`
  - Co sprawdza: Konta z ustawionym hasłem "No Password" (NP).
  - Zagrożenia: Konto nie ma ważnego hasła, może być używane bez autoryzacji.
  - Prawidłowy wynik: Brak wyjścia lub tylko konta systemowe, które celowo mają zablokowane hasło (jeśli takie istnieją).

- Polecenie: `last | head -20`
  - Co sprawdza: Ostatnie udane logowania.
  - Zagrożenia: Logowania o nietypowych porach, z nieznanych IP, powtarzające się z tego samego IP.
  - Prawidłowy wynik: Znane użytkownicy, oczekiwane godziny i źródła.

- Polecenie: `lastb | head -20`
  - Co sprawdza: Nieudane próby logowania.
  - Zagrożenia: Duża liczba nieudanych prób świadczy o ataku brute-force.
  - Prawidłowy wynik: Niewiele wpisów lub brak.

- Polecenie: `who` lub `w`
  - Co sprawdza: Aktualnie zalogowanych użytkowników.
  - Zagrożenia: Nieznane sesje, użytkownicy, którzy nie powinni być zalogowani.
  - Prawidłowy wynik: Tylko znane, oczekiwane sesje.

## 2. Uprawnienia i własność plików

- Polecenie: `sudo find / -perm -4000 -type f 2>/dev/null`
  - Co sprawdza: Pliki z ustawionym bitem SUID.
  - Zagrożenia: Nieautoryzowane podniesienie uprawnień, potencjalne luki w tych plikach.
  - Prawidłowy wynik: Znane pliki SUID (np. `passwd`, `su`, `sudo`). Brak nietypowych binariów.

- Polecenie: `sudo find / -perm -2000 -type f 2>/dev/null`
  - Co sprawdza: Pliki z bitem SGID.
  - Zagrożenia: Podobnie jak SUID – możliwość eskalacji lub dostępu do grup.
  - Prawidłowy wynik: Znane pliki SGID.

- Polecenie: `sudo find / -perm -002 -type f 2>/dev/null`
  - Co sprawdza: Pliki świat zapisywalne.
  - Zagrożenia: Modyfikacja plików przez nieupoważnionych użytkowników, potencjalne podmiany.
  - Prawidłowy wynik: Niewiele plików (np. w `/tmp`, `/var/tmp`) – poza tymi katalogami powinno być ich jak najmniej.

- Polecenie: `sudo find / -perm -002 -type d 2>/dev/null`
  - Co sprawdza: Katalogi świat zapisywalne.
  - Zagrożenia: Możliwość tworzenia/usuwania plików przez każdego, ryzyko zaśmiecenia lub ataku.
  - Prawidłowy wynik: Standardowe katalogi tymczasowe (`/tmp`, `/var/tmp`, `/dev/shm`) – inne powinny być rzadkością.

- Polecenie: `sudo find / -nouser -o -nogroup 2>/dev/null`
  - Co sprawdza: Pliki i katalogi bez właściciela lub grupy.
  - Zagrożenia: Pozostałości po usuniętych użytkownikach, mogą być wykorzystane.
  - Prawidłowy wynik: Brak lub bardzo nieliczne, najlepiej po czyszczeniu.

- Polecenie: `ls -la /tmp/`
  - Co sprawdza: Zawartość katalogu `/tmp`.
  - Zagrożenia: Skrypty, pliki wykonywalne, podejrzane nazwy.
  - Prawidłowy wynik: Tylko pliki tymczasowe, bez binariów i skryptów wykonywalnych (chyba że uzasadnione).

## 3. Procesy i usługi

- Polecenie: `ps auxf`
  - Co sprawdza: Drzewo procesów z widokiem hierarchii.
  - Zagrożenia: Ukryte procesy, procesy potomne nietypowe, procesy uruchomione z nieznanych lokalizacji.
  - Prawidłowy wynik: Znane procesy systemowe i aplikacyjne, brak podejrzanych nazw.

- Polecenie: `ss -tulpn`
  - Co sprawdza: Nasłuchujące porty TCP/UDP wraz z procesami.
  - Zagrożenia: Nieznane usługi otwarte na portach, nasłuch na wszystkich interfejsach (0.0.0.0) bez potrzeby.
  - Prawidłowy wynik: Tylko oczekiwane usługi (np. SSH, HTTP, baza danych).

- Polecenie: `ps aux | grep -v root`
  - Co sprawdza: Procesy nienależące do roota.
  - Zagrożenia: Procesy użytkowników, które powinny być nieobecne lub są podejrzane.
  - Prawidłowy wynik: Tylko procesy uruchomione przez znanych użytkowników w celach uzasadnionych.

- Polecenie: `top -b -n 1 | head -20`
  - Co sprawdza: Procesy zużywające najwięcej CPU.
  - Zagrożenia: Wysokie zużycie CPU przez nieznany proces – może wskazywać na miner kryptowalut lub złośliwe oprogramowanie.
  - Prawidłowy wynik: Znane procesy systemowe lub aplikacyjne, zużycie zgodne z oczekiwaniami.

- Polecenie: `ps aux | grep -w Z`
  - Co sprawdza: Procesy zombie.
  - Zagrożenia: Duża liczba zombie może świadczyć o problemach z aplikacją, ale też o ataku.
  - Prawidłowy wynik: Brak procesów zombie.

- Polecenie: `ps aux | awk '{print $2}' | sort -n | uniq -d`
  - Co sprawdza: Duplikaty PID (powinny być unikalne).
  - Zagrożenia: Manipulacja listą procesów, ukrywanie procesów.
  - Prawidłowy wynik: Brak wyjścia (każdy PID unikalny).

- Polecenie: `systemctl list-units --type=service --state=running`
  - Co sprawdza: Aktywne usługi systemd.
  - Zagrożenia: Nieznane lub niepotrzebne usługi.
  - Prawidłowy wynik: Tylko usługi niezbędne do działania systemu.

## 4. Sieć i połączenia

- Polecenie: `ss -tlnp`
  - Co sprawdza: Nasłuchujące porty TCP i ich procesy.
  - Zagrożenia: Otwarte porty bez uzasadnienia, nasłuch na wszystkich interfejsach.
  - Prawidłowy wynik: Tylko wymagane porty (np. 22, 80, 443).

- Polecenie: `ss -tunp`
  - Co sprawdza: Aktywne połączenia sieciowe (TCP/UDP) z procesami.
  - Zagrożenia: Połączenia wychodzące na nieznane adresy IP, podejrzane porty.
  - Prawidłowy wynik: Połączenia związane z legalnymi usługami (np. aktualizacje, bazy danych).

- Polecenie: `ip route show`
  - Co sprawdza: Tablicę routingu.
  - Zagrożenia: Dodatkowe, niestandardowe trasy mogą wskazywać na przekierowanie ruchu.
  - Prawidłowy wynik: Standardowa trasa domyślna i trasy dla podsieci lokalnych.

- Polecenie: `ip a`
  - Co sprawdza: Interfejsy sieciowe.
  - Zagrożenia: Nieznane interfejsy (np. `tun0`, `docker0`) mogą świadczyć o tunelach lub kontenerach, ale mogą być też niepożądane.
  - Prawidłowy wynik: Tylko interfejsy, które powinny istnieć (np. `eth0`, `lo`).

- Polecenie: `lsof -i`
  - Co sprawdza: Otwarte pliki związane z siecią (gniazda).
  - Zagrożenia: Procesy korzystające z sieci, których się nie spodziewamy.
  - Prawidłowy wynik: Znane procesy z połączeniami sieciowymi.

- Polecenie: `cat /etc/resolv.conf`
  - Co sprawdza: Serwery DNS używane przez system.
  - Zagrożenia: Nieznane DNS mogą wskazywać na przekierowanie domen.
  - Prawidłowy wynik: Zaufane serwery DNS (np. firmowe, publiczne).

- Polecenie: `cat /etc/hosts`
  - Co sprawdza: Lokalne mapowania nazw hostów.
  - Zagrożenia: Przekierowania domen na podejrzane IP.
  - Prawidłowy wynik: Głównie wpisy dla localhost oraz ewentualnie znane aliasy.

## 5. Logi i system plików

- Polecenie: `sudo tail -50 /var/log/auth.log`
  - Co sprawdza: Ostatnie wpisy logów uwierzytelniania.
  - Zagrożenia: Wiele nieudanych prób logowania, podejrzane sesje.
  - Prawidłowy wynik: Normalna aktywność uwierzytelniania (logowania, sudo).

- Polecenie: `sudo tail -50 /var/log/syslog`
  - Co sprawdza: Systemowe błędy i zdarzenia.
  - Zagrożenia: Powtarzające się błędy, ostrzeżenia o naruszeniach.
  - Prawidłowy wynik: Oczekiwane komunikaty, brak krytycznych błędów.

- Polecenie: `sudo grep "Failed password" /var/log/auth.log | tail -20`
  - Co sprawdza: Ostatnie nieudane próby logowania.
  - Zagrożenia: Wiele prób z tego samego IP – atak brute-force.
  - Prawidłowy wynik: Niewiele wpisów, brak powtarzalności.

- Polecenie: `sudo grep "sudo" /var/log/auth.log | tail -20`
  - Co sprawdza: Użycie sudo.
  - Zagrożenia: Nieznani użytkownicy używający sudo, nieautoryzowane polecenia.
  - Prawidłowy wynik: Znani użytkownicy wykonujący oczekiwane polecenia.

- Polecenie: `sudo tail -50 /var/log/auth.log | grep sshd`
  - Co sprawdza: Logi związane z SSH.
  - Zagrożenia: Próby logowania, akceptacja połączeń z nieznanych IP.
  - Prawidłowy wynik: Normalny ruch SSH.

- Polecenie: `df -h`
  - Co sprawdza: Zużycie przestrzeni dyskowej.
  - Zagrożenia: Pełne partycje, zwłaszcza `/tmp`, `/var` – mogą uniemożliwić działanie systemu lub ukrywać pliki.
  - Prawidłowy wynik: Wystarczająca wolna przestrzeń.

- Polecenie: `mount | grep -E "(noexec|nosuid|nodev)"`
  - Co sprawdza: Opcje montowania partycji.
  - Zagrożenia: Brak flag bezpieczeństwa na partycjach, gdzie są potrzebne (np. `/tmp` powinien mieć `noexec,nosuid,nodev`).
  - Prawidłowy wynik: Partycje tymczasowe mają odpowiednie flagi.

- Polecenie: `crontab -l` oraz `sudo crontab -l`
  - Co sprawdza: Zadania cron dla użytkownika i roota.
  - Zagrożenia: Nieznane zadania, skrypty pobierające/uruchamiające podejrzany kod.
  - Prawidłowy wynik: Tylko znane, uzasadnione zadania.

- Polecenie: `ls -la /etc/cron.*`
  - Co sprawdza: Systemowe pliki cron.
  - Zagrożenia: Dodatkowe skrypty w katalogach cron.
  - Prawidłowy wynik: Oczekiwane skrypty systemowe.

## 6. Konfiguracja SSH

- Polecenie: `grep PermitRootLogin /etc/ssh/sshd_config`
  - Co sprawdza: Czy root może logować się przez SSH.
  - Zagrożenia: Logowanie roota przez SSH zwiększa ryzyko ataku.
  - Prawidłowy wynik: `PermitRootLogin no` lub `prohibit-password` (zalecane `no`).

- Polecenie: `grep PasswordAuthentication /etc/ssh/sshd_config`
  - Co sprawdza: Czy logowanie hasłem jest dozwolone.
  - Zagrożenia: Logowanie hasłem może być podatne na brute-force.
  - Prawidłowy wynik: `PasswordAuthentication no` (zalecane, jeśli używane są klucze).

- Polecenie: `grep Port /etc/ssh/sshd_config`
  - Co sprawdza: Port nasłuchiwania SSH.
  - Zagrożenia: Nietypowy port może utrudnić wykrycie, ale też może być używany do ukrycia usługi.
  - Prawidłowy wynik: Standardowo `Port 22`, ale zmiana na inny może być zaakceptowana, jeśli jest celowa.

- Polecenie: `ls -la ~/.ssh/authorized_keys`
  - Co sprawdza: Klucze autoryzowane dla użytkownika.
  - Zagrożenia: Dodatkowe, nieznane klucze mogą być backdoorem.
  - Prawidłowy wynik: Tylko klucze znanych użytkowników.

- Polecenie: `grep LogLevel /etc/ssh/sshd_config`
  - Co sprawdza: Poziom logowania SSH.
  - Zagrożenia: Niski poziom logowania utrudnia wykrycie ataków.
  - Prawidłowy wynik: `LogLevel VERBOSE` lub `INFO` (zalecane `VERBOSE`).

## 7. Uprawnienia Sudo

- Polecenie: `sudo cat /etc/sudoers`
  - Co sprawdza: Główny plik konfiguracyjny sudo.
  - Zagrożenia: Wpisy z `NOPASSWD` dla wszystkich, niebezpieczne aliasy.
  - Prawidłowy wynik: Ograniczone uprawnienia, brak nadmiarowych `NOPASSWD`.

- Polecenie: `ls -la /etc/sudoers.d/`
  - Co sprawdza: Dodatkowe pliki konfiguracyjne sudo.
  - Zagrożenia: Pliki z nieautoryzowanymi regułami.
  - Prawidłowy wynik: Tylko oczekiwane pliki, jeśli istnieją.

- Polecenie: `grep -E "^[^#].*ALL=" /etc/sudoers`
  - Co sprawdza: Użytkownicy z pełnym dostępem (ALL).
  - Zagrożenia: Zbyt wiele kont ma pełne sudo.
  - Prawidłowy wynik: Tylko administratorzy.

- Polecenie: `sudo grep sudo /var/log/auth.log | tail -20`
  - Co sprawdza: Ostatnie użycia sudo.
  - Zagrożenia: Wykonywanie podejrzanych poleceń przez nieznanych użytkowników.
  - Prawidłowy wynik: Znane polecenia, znani użytkownicy.

## 8. Jądro i bezpieczeństwo systemu

- Polecenie: `uname -a`
  - Co sprawdza: Wersję jądra i architekturę.
  - Zagrożenia: Starsze jądro z niezałatanymi lukami.
  - Prawidłowy wynik: Aktualna, wspierana wersja.

- Polecenie: `lsmod`
  - Co sprawdza: Załadowane moduły jądra.
  - Zagrożenia: Nieznane lub podejrzane moduły (np. rootkity).
  - Prawidłowy wynik: Tylko standardowe moduły.

- Polecenie: `cat /proc/sys/kernel/randomize_va_space`
  - Co sprawdza: ASLR (Address Space Layout Randomization).
  - Zagrożenia: Wyłączone ASLR ułatwia exploity.
  - Prawidłowy wynik: `1` lub `2` (zalecane `2`).

- Polecenie: `sudo aa-status` (dla AppArmor) lub `sestatus` (dla SELinux)
  - Co sprawdza: Czy obowiązkowa kontrola dostępu jest włączona.
  - Zagrożenia: Brak dodatkowej warstwy bezpieczeństwa.
  - Prawidłowy wynik: AppArmor/SELinux włączone.

- Polecenie: `sudo iptables -L -n` lub `firewall-cmd --list-all`
  - Co sprawdza: Reguły firewalla.
  - Zagrożenia: Brak reguł lub reguły zezwalające na wszystko.
  - Prawidłowy wynik: Zdefiniowane reguły ograniczające ruch.

## 9. Szybka weryfikacja (jedno polecenie – wiele informacji)

- Polecenie: `sudo awk -F: '{print $1 ":" $3 ":" $7}' /etc/passwd`
  - Co sprawdza: Podsumowanie użytkowników, UID i powłok.
  - Zagrożenia: Szybkie wykrycie nietypowych UID lub powłok.
  - Prawidłowy wynik: Znane użytkownicy i standardowe powłoki.

- Polecenie: `sudo find / -perm -4000 -type f 2>/dev/null | while read f; do ls -la "$f"; done`
  - Co sprawdza: Lista plików SUID z uprawnieniami.
  - Zagrożenia: Jak wyżej.
  - Prawidłowy wynik: Znane pliki SUID.

- Polecenie: `ss -tulpn`
  - Co sprawdza: Nasłuchujące porty.
  - Zagrożenia: Otwarte porty.
  - Prawidłowy wynik: Oczekiwane usługi.

- Polecenie: `sudo journalctl -n 50 --no-pager`
  - Co sprawdza: Ostatnie 50 wpisów dziennika systemowego.
  - Zagrożenia: Krytyczne błędy, podejrzane wpisy.
  - Prawidłowy wynik: Brak alarmujących komunikatów.

## 10. Co zrobić, jeśli znajdziesz coś podejrzanego

- Zadokumentuj – zapisz dokładnie, co znalazłeś, o której godzinie, na jakim systemie.
- Nie paniczkuj – niektóre rzeczy mogą być fałszywie pozytywne (np. znane pliki SUID).
- Zweryfikuj – potwierdź innym narzędziem lub sprawdź w dokumentacji.
- Zgłoś – powiadom zespół bezpieczeństwa lub administratora.
- Zbierz dowody – zachowaj kopie logów, wyjść poleceń, zrzuty procesów (jeśli to konieczne).
- Działaj zgodnie z procedurą – jeśli masz IR Playbook, postępuj według niego.
