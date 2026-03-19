# /etc/sudoers

## Cel laboratorium 
Celem tego laboratorium było zapoznanie się jak działają zapisy w pliku `/etc/sudoers` oraz jaki to ma wpływ na uprawnienia użytkowników.


## Czym jest /etc/sudoers
Plik `/etc/sudoers` określa, które polecenia użytkownik może uruchamiać z uprawnieniami innego użytkownika (najczęściej root).

## Defaults
Defaults to ustawienia sterujące zachowaniem polecenia sudo, np. reset zmiennych środowiskowych (env_reset) czy bezpieczna ścieżka (secure_path).

## Wykonane kroki
- przejrzano plik `/etc/sudoers` (`sudo less`) oraz katalog `/etc/sudoers.d/` (`sudo ls`),
- utworzono nowego użytkownika nienależącego do żadnej uprzywilejowanej grupy,
- utworzono plik w `/etc/sudoers.d/` o nazwie zgodniej z nazwą użytkownika za pomocą polecenia `sudo visudo -f /etc/sudoers.d/user-apt`, 
- nadano użytkownikowi uprawnienia do uruchamiania poleceń jako root `apt update` i `apt upgrade`,
- sprawdzono dostępne polecenia `sudo -l` oraz `sudo -l -U user`.

## Obserwacje 
- UID 0 ma pełne uprawnienia bez użycia sudo, natomiast użytkownicy z grupy sudo uzyskują pełne uprawnienia za pomocą polecenia `sudo`, zgodnie z konfiguracją pliku `/etc/sudoers`,
- plik `/etc/sudoers` jest chroniony uprawnieniami i nie może być odczytany przez zwykłego użytkownika,
- w katalogu `/etc/sudoers.d/` znajdują się pliki konfiguracyjne uprawnień dla poszczególnych użytkowników. 

## Wnioski bezpieczeństwa
Na co trzeba zwrócić uwagę:
- kto ma pełny dostęp - w pliku `/etc/sudoers` widać jakie prawa mają grupy root i sudo, w `/etc/group` można sprawdzić kto należy do tych grup i czy na pewno powinien tam należeć, 
- kto ma NOPASSWD (użycie sudo bez podawania hasła),
- czy są nietypowe wpisy w pliku `/etc/sudoers` lub nieznane pliki w katalogu `/etc/sudoers.d/`,
- należy monitorować zmiany w pliku `/etc/sudoers` jako potencjalne zdarzenia bezpieczeństwa.

## Potencjalne zagrożenia
- każdy użytkownik z NOPASSWD to potencjalny wektor ataku, 
- nietypowe polecenia pozwalające na zapis do systemowych katalogów, należy sprawdzić czy są zasadne,
- zamiast dodawać wszystkich do grupy sudo (mającej pełne uprawnienia w sudoers), lepiej umieszczać konkretnych użytkowników w plikach `/etc/sudoers.d/` nadając im tylko konieczne uprawnienia,
- blokowanie pojedynczych poleceń w sudoers (np. /bin/bash) nie jest skutecznym mechanizmem zabezpieczeń, ponieważ może zostać łatwo ominięte (np. przez użycie innych powłok lub narzędzi umożliwiających wykonanie shell). Zalecanym podejściem jest whitelistowanie konkretnych poleceń zgodnie z zasadą least privilege.

## Analiza wpisów w pliku `/etc/sudoers` (Case Study)

Konfiguracja:  
	Defaults env_reset  
	Defaults:user1 !requiretty  
	user1 ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade  

Sytuacja:  
	user1 próbuje uruchomić dozwolone polecenie `apt update` oraz niedozwolone `id`, `sudo -i`  

Analiza:  
- dostęp do `apt update` działa zgodnie z konfiguracją,
- próby uruchomienia innych poleceń kończą się niepowodzeniem,
- `env_reset` usuwa zmienne środowiskowe użytkownika przed wykonaniem polecenia, 
- `!requiretty` umożliwia użycie sudo bez interaktywnego terminala.
