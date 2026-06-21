# File permission cases

## Cel
Przećwiczenie i zrozumienie sytuacji z błędami w konfiguracji uprawnień dla katalogów i plików

## Case study 1
Katalog `/tmp` ma uprawnienia `drwxrwxrwt` (1777). Czy uprawnienia są poprawne? 

Tak, uprawnienia są poprawne i nie przypadkowe:
- został użyty sticky bit do katalogu `/tmp` reprezentowany przez `t` w miejscu uprawnień wykonania dla innych. To powoduje, że tylko właściciel pliku w tym katalogu, może usunąć swój plik. 
- gdyby nie został ustawiony sticky bit, katalog stanowiłby zagrożenie dla systemu. W `/tmp` są przechowywane pliki tymczasowe, w tym pliki usług, które mogą być w danym momencie w użyciu. Jeśli ktoś skasował by plik, który jest używany przez jakiś program czy usługę, to mogło by spowodować niepoprawne działanie. 
- aby sprawdzić czy `/tmp` ma poprawne uprawnienia można użyć polecenia `ls -la /tmp`. 

## Case study 2
Plik `/etc/shadow` w systemie na uprawnienia `rw-------` (600). Czy te uprawnienia są poprawnie ustawione?  

Tak, ustawienia są poprawnie ustawione:
- plik `/etc/shadow` to plik przechowujący hashe haseł i wszelkie informacje dotyczące ważności haseł, rodzaju użytego algorytmu hashującego, blokady haseł, itp. Dlatego ważne jest, żeby dostęp do tego pliku był ograniczony. Wyłącznie root powinien móc odczytać i zapisać hasła. 
- gdyby uprawnienia zostały ustawione na 644, każdy mógłby odczytać plik zawierający dane haseł, w tym hashe haseł. To niebezpieczna sytuacja.
- aby sprawdzić uprawnienia pliku należy użyć polecenia `ls -la /etc/shadow`,
- aby ustawić poprawne ustawienia w przypadku gdy nie są poprawne należy użyć polecenia: `chmod 600 /etc/shadow`. 

## Case study 3
`SUID` na `vim` - ktoś ustawił `chmod u+s /usr/bin/vim`.     

Czy to bezpieczne?    
Nie, to nie jest bezpieczna sytuacja:
- vim to edytor tekstowy, który ma możliwość uruchomienia powłoki, 
- `SUID` oznacza, że program uruchomi się z uprawnieniami właściciela, niezależnie od tego kto go otwiera. Ponieważ właścicielem vim jest root, każdy użytkownik mógłby uzyskać dostęp do powłoki roota,
- by wyszukać i przejrzeć pliki, które mają ustawiony `SUID` należy użyć polecenia: `find / -perm -4000 2>/dev/null`. Można także wyszukiwać `SGID` i `SUID` jednocześnie `perm -6000`. 

## Case study 4
Użytkownik ma w `~/.bashrc` wpis `umask 000`. Czy to bezpieczne ustawienie?  

Nie, ponieważ:
- wpis `umask 000` w pliku `~/.bashrc` to trwałe ustawienie, powodujące, że wszystkie nowe pliki i katalogi będą tworzone z domyślnymi ustawieniami systemowymi, czyli:
	- katalogi 777,
	- pliki 666,  
	Oznacza to, że każdy nowo tworzony katalog otrzyma pełne uprawnienia (777), a pliki każdy będzie mógł odczytać lub zapisać (666). 
- bezpieczne ustawienie: `umask 022` oznacza to, że zostają odjęte bity uprawnień, co ogranicza dostęp:
	- nowo utworzone katalogi otrzymają uprawnienia 755 (czyli pełne uprawnienia ma tylko właściciel, a pozostali mogą otwierać katalog i czytać),
	- nowo utworzone pliki otrzymają uprawnienia 644 (właściciel ma uprawnienia do odczytu i zapisu, a pozostali mogą tylko czytać). 
	UWAGA: czasami polityki bezpieczeństwa wymagają jeszcze bardziej restrykcyjnych ustawień: `umask 027`.

## Case study 5
Firma ma katalog `/shared`, w którym pracuje grupa `developers`. Katalog ma uprawnienia `drwxrwsr-x` (2775). 

Oznacza to, że:
- Katalog ma ustawiony `SGID` - pliki w tym folderze otwierają się z uprawnieniami grupy, niezależnie od tego kto je otwiera. Czy to bezpieczne? W tym przypadku nie, ponieważ uprawnienia dla others są ustawione na `r-x`, oznacza to, że każdy zwykły użytkownik może wykonać lub otworzyć plik z tego katalogu z podwyższonymi uprawnieniami. Ustawienie byłoby bezpieczne gdyby uprawnienia dla innych były wyłączone (drwxrws---).
- w tym przypadku wszystkie pliki tworzone w tym katalogu będą się otwierać z uprawnieniami grupy, choć tylko grupa i root mogą zapisywać tam pliki. 
- `SGID` na pliku wykonywalnym (np. `chmod g+s /usr/local/bin/skrypt`) powoduje, że plik uruchamia się z uprawnieniami grupy właściciela pliku, nie użytkownika. Może to być użyteczne, ale jednocześnie stanowi ryzyko eskalacji uprawnień, jeśli plik nie jest odpowiednio zabezpieczony.


