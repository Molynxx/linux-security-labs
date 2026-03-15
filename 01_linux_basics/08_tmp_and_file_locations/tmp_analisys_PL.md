# Analiza katalogu /tmp w Linuxie

## Cel laboratorium
Celem laboratorium jest zrozumienie jakie zagrożenia wynikają z katalogu /tmp i jak go monitorować.

## Czym jest /tmp
Katalog /tmp służy do przechowywania plików tymczasowych tworzonych przez system i aplikacje. To miejsce gdzie aplikacje zapisują dane tymczasowe, np. sesje, cache, locki, sockety, FIFO, pliki robocze. W nowoczesnych systemach Linux występują dwa mechanizmy czyszczenia katalogu /tmp:
- tmpfs - wirtualny system plików, który przechowuje dane w pamięci RAM, a nie na dysku. Oznacza to, że pliki w tmpfs znikają po restarcie systemu (bo RAM się czyści).
- systemd-tmpfiles - to mechanizm zarządzany przez systemd, który automatycznie czyści stare pliki w katalogach tymczasowych starsze niż X dni. Katalog /tmp jest katalogiem otwartym dla wszystkich użytkowników, posiada uprawnienia pozwalające wszystkim użytkownikom na tworzenie plików, ponieważ wiele aplikacji oraz procesów systemowych musi mieć możliwość zapisywania danych tymczasowych niezależenie od kontekstu użytkownika. Dlatego też /tmp wymaga odpowiednich uprawnień i mechanizmów ochronnych (Sticky bit).
W niektórych systemach Linux dodatkową warstwę bezpieczeństwa zapewniają mechanizmy Mandatory Access Control (MAC), wprowadzając dodatkowe zasady określające, jakie operacje mogą wykonywać procesy w systemie. 

## Wykonane kroki
- sprawdzono czy tmpfs jest włączone w systemie poleceniem: mount | grep /tmp,
- sprawdzono dane dla zamontowanego katalogu poleceniem: df -hT /tmp,
- sprawdzono automatyczne czyszczenie za pomocą poleceń: 
	- systemd-analyze cat config systemd-tmpfiles | grep /tmp (polecenie nie zwróciło wyniku w analizowanym systemie),
	- cat /usr/lib/tmpfiles.d/tmp.conf,
	- cat /etc/tmpfiles.d/tmp.conf (plik nie istnieje w analizowanym systemie),
- sprawdzono czy tmpfiles działa za pomocą polecenia: systemctl status systemd-tmpfiles-clean.service,
- wymuszono czyszczenie katalogu /tmp za pomocą polecenia: sudo systemd-tmpfiles --clean,
- sprawdzono czy stucky bit istnieje dla katalogu /tmp za pomocą polecenia ls -ld /tmp,
- sprawdzono czy w systemie jest SELinux za pomocą polecenia: ls -Z,
- sprawdzono czy w systemie jest AppArmor za pomocą polecenia: aa-status.



## Obserwacje
- W analizowanym systemie Kali Linux /tmp jest zamontowany jako system plików tmpfs, cco oznacza, że pliki tymczasowe przechowywane są s pamięci RAM i znikają po restarcie systemu. Dodatkowo system wykorzystuje mechanizm systemd-tmpfiles, którego konfiguracja znajduje się w katalogu /usr/lib/tmpfiles.d/tmp.conf. Mechanizm ten usuwa stare pliki w trakcie działania systemy, bez konieczności restartu. W pliku tmp.conf znajdują się reguły określające czas przechowywania plików tymczasowych. W systemie zdefiniowano m.in. następujące zasady:
	- pliki w /tmp starsze niż 10 dni mogą zostać automatycznie usunięte, 
	- pliki w /var/tmp starsze niż 30 dni mogę zostać usunięte. 
- sticky bit jest ustawiony dla katalogu /tmp,
- w systemie nie jest zainstalowany SELinux,
- w katalogu /tmp nie zauważono podejrzanych plików, 
- w analizowanym momencie większość plików należała do root, 
- pliki należące do nieuprzywilejowanych użytkowników nie miały uprawnień wykonywalnych, 
- zauważalna różnica czasu (UTC vs czas lokalny), system zapisuje czasy UTC,
- wszystkie pliki posiadają datę z dnia bieżącego, nie ma starych plików w katalogu /tmp,
- polecenie: sudo systemd-tmpfiles --clean nie wyczyściło plików, które mają dzisiejszą datę. 

## Wnioski Bezpieczeństwa
- Na podstawie obserwowanych atrybutów plików i kontekstu czasowego nie stwierdzono oznak złośliwej aktywności. Zachowania zgodne z normalną pracą systemu,
- w analizowanym systemie funkcjonuje tmpfs oraz automatyczne czyszczenie plików starszych niż 10 dni za pomocą systemd-tmpfiles,
- sticky bit ustawiony dla katalogu /tmp zapewnia, że nikt nie może usuwać plików innych użytkowników, 
- ze względu na otwarty charakter katalogu /tmp, w którym pod kątem potencjalnych oznak nadużyć lub złośliwej aktywności. Przykładowe polecenia:
	- find /tmp - type f -mtime +7 -ls - znajduje pliki, które nie były modyfikowane przez 7 dni, 
	- losf +D /tmp - wyświetla listę plików aktualnie otwartych przez procesy w systemie, 
- SOC powinien zawsze uwzględniać UTC przy analizie plików tymczasowych, żeby time line był spójny,
- mechanizmy takie jak SELinux lub APpArmor mogą ograniczać sposób, w jaki aplikacje korzystają z katalogów tymczasowych takich jak /tmp. Dzięki temu nawet jeśli złośliwe oprogramowania zostanie uruchomione, jego możliwości działania mogą być częściowo ograniczone przez politykę bezpieczeństwa systemu. W analizowanym systemie SELinux nie jest zainstalowany, a AppArmor nie był aktywny lub nie był używany przez system, dlatego żaden z nich nie wpływał na zachowanie katalogu /tmp w trakcie przeprowadzanego laboratorium,
- nie należy usuwać katalogu /tmp, ponieważ jest on wymagany przez wiele aplikacji oraz procesów systemowych, 
- ręczne czyszczenie katalogu /tmp podczas działania systemu może powodować problemy z aplikacjami korzystającymi z plików tymczasowych,
- otwarty charakter katalogu /tmp sprawia, że jest on często wykorzystywany przez aplikacje oraz procesy systemowe do zapisu danych tymczasowych. W przypadki kompromitacji systemu katalog ten może być również używany przez złośliwe oprogramowanie do przechowywania plików roboczych lub tymczasowych komponentów malware. 

