## /etc/sudoers

### Czym jest /etc/sudoers
Plik /etc/sudoers definiuje kto i jakie polecenia może uruchamiać z podwyższonymi uprawnieniami. Obejmuje dane:
- kto,
- na jakim hoście,
- jako jaki użytkownik,
- jakie polecenie może uruchomić.

### Wykonane kroki
- Przejrzano plik /etc/sudoers za pomocą polecenia sudo less,
- Przejrzano pliki znajdujące się w katalogu /etc/sudoers.d/* za pomocą polecenia sudo ls,
- Uruchomiono sudo visudo żeby otworzyć plik /etc/sudoers z możliwością edycji,
- Utworzono nowego użytkownika nienależącego do żadnej uprzywilejowanej grupy,
- Utworzono plik w /etc/sudoers.d/ o nazwie zgodnej z nazwą użytkownika za pomocą polecenia sudo visudo -f /etc/sudoers.d/user-apt,
- W powyższym pliku nadano user uprawnienia root dla sudo apt update i sudo apt upgrade wpisując linię pliku 'user ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade',
- Sprawdzono jakie polecenia z uprawnieniami root może uruchomić user, za pomocą polecenia sudo -l -U user.

### Obserwacje 
- Grupa root ma pełne uprawnienia bez użycia sudo, natomiast użytkownicy z grupy sudo uzyskują pełne uprawnienia za pomocą polecenia sudo, zgodnie z konfiguracją pliku /etc/sudoers,
- Powyższe oznacza, że każdy użytkownik dodany do jednej z tych grup zyska pełne prawa do wszystkiego jako root,
- Plik /etc/sudoers jest chroniony uprawnieniami i nie może być odczytany przez zwykłego użytkownika. Do bezpiecznej edycji używa się wyłącznie visudo,
- Plik /etc/sudoers można otworzyć bezpiecznie poleceniem sudo less, ponieważ używając tego polecenia nie ma możliwości edycji pliku,
- W katalogu /etc/sudoers.d/* znadują się pliki konfiguracyjne uprawnień dla poszczególnych użytkowników, m.in. plik zdefiniowany przeze mnie dla nowego użytkownika user.

### Wnioski bezpieczeństwa
Na co trzeba zwrócić uwagę:
- Kto ma pełny dostęp, w pliku /etc/sudoers widać jakie prawa mają grupy root i sudo, sprawdzić w /etc/group kto należy do tych grup i czy na pewno powinien tam należeć, 
- Kto ma NOPASSWD,
- Czy są nietypowe wpisy w pliki /etc/sudoers,
- Czy są nietypowe, nieznane pliki w katalogu /etc/sudoers.d/

### Potencjalne zagrożenia
- Każdy użytkownik z NOPASSWD to potencjalny wektor ataku, 
- Nietypowe polecenia pozwalające na zapis do systemowych katalogów (/etc, /root, var) powinny być dokładnie sprawdzone, czy rzeczywiście powinny tam być, 
- Jeśli użytkownik jest w grupie sudo, dla której konfiguracja pliku /etc/sudoers nadaje pełne uprawnienia, nie ma możliwości zablokowania mu czegokolwiek, dlatego jeśli użytkownik ma mieć tylko niektóre prawa roota należy usunąć go z grupy sudo i dodać stosowny plik w /etc/sudoers.d/ by nadać mu indywidualne uptrawenienia,
- Chcąc zabronić użytkownikowi dostępu  do /bin/bash ze względów bezpieczeństwa, nie można zrobić tego dodając wpis w /etc/sudoers.d/plikusera 'user ALL = (ALL) ALL !/bin/bash', ponieważ można to obejść korzystając z innych powłok niż bash, np użycie podpowłoki zsh,
- Nie zadziała również zapis 'user ALL =(ALL) ALL !/bin/bash, !/bin/sh, !/bin/zsh', ponieważ nadal można to obejść na kilka sposobów:
	- sudo -s,
	- sudo -i,
	- env.
- Najskuteczniejszym sposobem zabezpieczenia jest zapis w osobnym pliku dla danego użytkownika w /etc/sudoers.d/, gdzie można przykładowo nadać uprawnienia root użytkownikowi dla poleceń apt update i apt upgrade za pomocą zapisu w pliku : 'user ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade.
- Tworzenie oddzielnych plików w /etc/sudoers.d/ umożliwia nadawanie granularnych uprawnień, ograniczając ryzyko pełnego dostępu do roota. 
- Każdy mowy wpis w sudoers lub sudoers.d powinien być weryfikowany pod kątem nietypowych poleceń, zwłaszcza zapisów umożliwiających zapis do systemowych katalogów (/etc, /root, /var).