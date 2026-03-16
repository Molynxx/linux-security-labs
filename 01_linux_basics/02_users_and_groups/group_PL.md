# /etc/group

## Cel laboratorium
Celem tego laboratorium było zrozumienie jak działają grupy, jakie zapisy znajdują się w pliku /etc/group oraz jak je interpretować. 

## Czym jest /etc/group
Plik ten zawiera informacje o grupach w systemie:
- nazwa grupy,
- hash / marker 'x' hasła,
- GID,
- członkowie grupy.
Plik nie powinien być edytowany ręcznie, bezpieczna modyfikacja pliku wymaga użycia narzędzi: 
- groupadd,
- groupmod,
- groupdel,
- groupadd -aG.

## Plik /etc/gshadow
W starych wersjach systemu plik ten zawierał hashe haseł grup (obecnie nie używa się ich). Zawiera informacje o grupach i administratora, dostęp ograniczony do root. Hasła grup rzadko używane. 

## Grupy pierwotne i wtórne (Primary Group, Secondary Group)
- Primary: główna grupa użytkownika, GID w /etc/passwd,
- Secondary: dodatkowe grupy.  
Uwaga bezpieczeństwa: uprawnienia do plików zależą od GID, domyślne prawa zależą od umask. Sama przynależność do grupy sudo niekoniecznie daje uprawnienia systemowe. Na wieli systemach (np. Debian / Ubuntu) uprawnienia dla grup i użytkowników są zapisane w pliku /etc/sudoers.

## Mechanizm kontroli dostępu oparty o grupy
Grupy wpływają na prawa plików i dostęp do urządzeń / logów / secketów, mogą być zarządzane też przez ALDAP / AD / NSS.

## Wykonane kroki
- przejrzano plik poleceniem: `cat /etc/group`,
- użyto `grep sudo` dla pliku, aby sprawdzić, którzy użytkownicy należą do grupy sudo,
- użyto `grep -E sudo|adm|docker etc/group`, aby sprawdzić użytkowników należących do uprzywilejowanych grup,
- celem powyższych działań było upewnienie się, że żaden nieautoryzowany użytkownik nie uzyskał dostępu,
- utworzono katalog na skrypty i ustawiono jego uprawnienia wyłącznie dla właściciela,
- stworzono skrypt wyświetlający wszystkich użytkowników z /etc/passwd oraz listę grup, do których należą, 
- dodano skrypt do zmiennej środowiskowej $PATH, aby można go było uruchomić z dowolnego miejsca w systemie.

## Napotkane błędy
- polecenie `source ~/.bashrc` zgłaszało wiele błędów, a skrypt .sh w $PATH nie działał. 

## Rozwiązanie problemu
- zrozumiano jak działają powłoki w systemie, 
- Kali Linux używa powłoki zsh a nie bash, co było przyczyną problemów, 
- aby naprawić problem, skrypt .sh przeniesiono do pliku konfiguracyjnego powłoki zsh, tj. ~/.zshrc,
- po tych krokach problem został rozwiązany i skrypt działał poprawnie. 

## Obserwacje
- Podczas modyfikacji zmiennych środowiskowych należy zwracać uwagę na używaną powłokę systemu, 
- w pliku /etc/group wyróżniono dwa rodzaje grup:
	- grupy systemowe,
	- grupy użytkowników.

## Wnioski bezpieczeństwa
- Jeśli grupa systemowa nie zawiera żadnego użytkownika, jest to normalne i można zignorować,
- jeśli grupa systemowa zawiera użytkownika interaktywnego, należy sprawdzić, czy powinien mieć do niej dostęp, szczególnie w grupach sudo, adm, docker,
- grupy użytkowników interaktywnych należy zawsze dokładnie sprawdzić, aby upewnić się, że nie ma nieautoryzowanych członków, 
- przynależność do grupy sudo nie zawsze oznacza pełne uprawnienia systemowe - by realnie nadać te uprawnienia należy mieścić stosowny zapis w pliku /etc/sudoers,
- usunięcie grupy z /etc/group nie usuwa jej GID z plików i procesów,
- zmieniając GID grupy (bez synchronizacji z /etc/passwd), należy zmienić GID w /etc/passwd oraz właścicielstwa plików (chown / chgrp rekurencyjnie),
- jeśli użytkownikowi zostanie zmieniony primary GID na numer GID grupy sudo, to zawsze stanowi to czerwoną flagę, ponieważ użytkownik w takim przypadki dzieli przestrzeń wspólną z uprzywilejowanymi użytkownikami, a w przypadki nieautoryzowanego użytkownika może to stanowić potencjalne zagrożenie,
- zasada spójności - każda zmiana GID grupy primary wymaga synchronizacji w /etc/group, /etc/passwd oraz ewentualnej aktualizacji właścielstw plików (chown / chgrp),
- Duplikacja numeru GID (co normalnie nie powinno być możliwe przy użyciu `groupadd`, ale może wystąpić przy ręcznej edycji pliku) może prowadzić do niezamierzonego rozszerzenia uprawnień. Numer GID jest dla kernela jedynym identyfikatorem grupy, nazwa nie ma znaczenia przy kontroli dostępu. 
- warto rozsądnie przydzielać uprawnienia w pliku /etc/sudoers, tj. nie nadawać uprawnień zbiorczo dla całej grupy. Bezpieczniejszą opcją jest tworzyć w katalogu /etc/sudoers.d/ osobne pliki z indywidualnymi uprawnieniami dla każdego użytkownika - ogranicz to eskalację uprawnień przez przypadkowe lub nieuprawnione dodanie kogoś do grupy. 

## Analiza wpisów /etc/group. korelacja z /etc/passwd i /etc/sudoers (Case Study)

### Przypadek 1   

Stan sytemu:  
	/etc/group:    
		sudo:x:27:  
	/etc/passwd:  
		anna:x:1001:27:Anna:/home/anna:/bin/bash    
	/etc/sudoers:  
		%sudo ALL=(ALL:ALL) ALL   

Analiza:  
- anna jest użytkownikiem, którego primary group to grupa sudo z GID 27,
- procesy użytkownika anna działają z GID 27, a więc grupy sudo. Kernel nie rozróżnia nazw grup, widzi jedynie GID, więc procesy anny mogą dziedziczyć uprawnienia grupy (np. zapis w katalogach grupy sudo). Zachodzi więc ryzyko, że anna może tworzyć i modyfikować pliki w przestrzeni współdzielonej przez grupę, mimo, że nie widać jej w grupie sudo w /etc/group,
- zgodnie z ustawieniami w pliku /etc/sudoers, anna ma jako członek grupy sudo pełne uprawnienia systemowe.   

Sugerowane działania: 
- należy zweryfikować, czy anna rzeczywiście powinna być w grupie sudo oraz kiedy i przez kogo został zmieniony GID anny na GID sudo,
- z perspektywy audytu bezpieczeństwa zaleca się, aby grupa sudo było grupą secondary, ponieważ ułatwia to identyfikację członkostwa w /etc/group. Należy więc zmienić grupę primary anny na własną grupę użytkownika i dodać ją do sudo jako secondary member. 
- należy sprawdzić czy i kiedy były edytowane pliki /etc/passwd o /etc/sudoers oraz jakie działania w ostatnim czasie anna przeprowadziła z podwyższonymi uprawnieniami, kierując się jej AUID. 

## Przypadek 2  

Stan systemu:  
	/etc/group:  
		dev:x:1050:  
		backup:x:1050:  
	/etc/passwd:   
		marek:s:1103:1050:Marek:/home/marek:/bin/bash  
	plik:  
		-rwx-rwx--- 1 root backup script.sh  

Analiza:  
-Grupa primary marka ma GID 1050, GID został zduplikowany i w obecnej sytuacji występują dwie grupy z tym GID,
- ze względu na zduplikowane GID grup oraz brak w /etc/group grupy pierwotnej o nazwie marek, nie można jednoznacznie ustalić pierwotnej grupy marka. W praktyce możliwe są dwie sytuacje:
	- GID w /etc/passwd zmieniono, a nazwa grupy w /etc/group pozostała,
	- GID zmieniono i nazwę grupy w /etc/group zmieniono (np z marek na dev),
- marek ma dostęp do plików obu grup, a więc będzie mógł wykonać script.sh,
- marek ma uprawnienia grupy dev, które obecnie ta grupa posiada w pliku /etc/sudoers.  

Sugerowane działania:  
- zweryfikować, która z grup jest pierwotna dla marka za pomocą:
	- plików historycznych:
		- /var/log/auth.log,
		- /ver/log/installer lub dpkg.log w Debian / Ubuntu,
	- jeśli system jest audytowany przez auditd -> `ausearch -f /etc/passwd lub /etc/group`,
- zmienić GID grupy primary marka na niekolidujący z innymi GID za pomocą polecenia `sudo groupmod -g <nowy_GID> dev`,
- zaktualizować wszystkie pliki należące do starego GID dev za pomocą polecenia: 
	- `sudo find / -group 1050 -exec chgrp -h dev {} \;`  
	lub
	- `sudo find / group 1050 -exec chown :dev {}\;`,  
- skontrolować uprawienia w sudoers.   

## Przypadek 3 

Była sobie grupa:  
	audit:x:1200:  
została usunięta z pliku /etc/group, ale:  
	w /etc/passwd jeden użytkownik nadal ma GID 1200 jako primary,  
	w systemie istnieją pliki: -rw-r----- 1 root 1200 raport.log    

Analiza:  
- ls -l pokaże numer GID, ale nie nazwę grupy, 
- prawa nadal działają bo kernel widzi numer, który został w passwd, 
- użytkownik będzie mógł odczytać plik raport.log, 
- prawa plików nadal będą działać dla użytkownika, który ma GID 1200 jako primary,
- nie działają uprawnienia dla grupy określone w pliku /etc/sudoers, ponieważ w zapisie ujęta jest nazwa grupy a nie GID. Nie da się więc zidentyfikować grupy na podstawie /etc/group/.    

Sugerowane działania:    
- przywrócić wpis w pliku /etc/group w takiej formie jak przed jego usunięciem, za pomocą polecenia: `sudo groupadd -g 1200 audit`,
- sprawdzić właścicielstwo plików za pomocą polecenia:  `sudo find / -grup 1200` oraz jeśli to konieczne także za pomocą polecenia `chgrp audit <plik>`,
- alternatywnie zmienić GID w /etc/passwd na istniejącą grupę, ale to wymaga przesunięcia właścicielstw wszystkich plików z GID 1200.
