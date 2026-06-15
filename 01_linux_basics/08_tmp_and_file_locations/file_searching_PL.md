# file_searching

## Cel
Zrozumienie, jak korzystać z narzędzi do wyszukiwania.

## Narzędzia

### find
Narzędzie to przeszukuje system w czasie rzeczywistym. Jest bardzo dokładny, jednak może być wolny (ponieważ czyta każdy plik). 
- podstawowa składnia:
	- `find [gdzie_szukać] [warunki] [akcje]`,
- wyjaśnienie:
	- `[gdzie_szukać]` to jest punk startowy pokazuje miejsce, gdzie szukać, np:
		- `.` - bieżący katalog, 
		- `/` - główny katalog systemowy (root),
		- `/home` - katalog domowy,
		- `/var/log /etc` - można podać kilka katalogów oddzielonych spacją. 
	- `[warunki]`  - serce polecenia, należy wskazać jakie cechy mają poszukiwane pliki. Można podać kilka warunków naraz:
		- `-name "nazwa_pliku"` - wyszukiwanie po nazwie pliku, 
		- `-iname "nazwa_pliku"` - wyszukiwanie po nazwie pliku, `-i` ignoruje wielkość liter, 
		- `-type f` - wyszukiwanie po typie pliku, `f`- plik, `d` - katalog, 
		- `-mtime -1` - wyszukiwanie po czasie ostatniej modyfikacji pliku (dni), `1` - mniej niż 24h temu, `+7` - więcej niż 7 dni temu
		- `-mmin -60` - wyszukiwanie po czasie modyfikacji (minuty) `-60` - mniej niż 60 minut temu, 
		- `-size +10M` - wyszukiwanie po rozmiarze pliku, `+10M` - większy niż 10 MB, `-1k` - mniejszy niż 1 kb,
		- `-perm -4000` - wyszukiwanie po uprawnieniach pliku, `-4000` - SUID, `-perm o+w` - world-writable, 
		- `-user root` - wyszukiwanie po właścicielu pliku, `-user root` - pliku użytkownika root, `-group adm` - pliki grupy adm. 
		- `-nouser` - znajduje pliki nie mających właściciela, `-nogroup` - pliki bez grupy,
		- jak łączyć warunki: 
			- domyślnie `find` łączy warunki logiką `AND`, czyli wszystkie muszą być spełnione. Przykład:
				- `find /home -type f -size +100M` - znajdź pliki w `/home`, które są zwykłymi plikami i mają rozmiar większy niż 100 MB,
			- można jednak użyć logiki `OR` używając opcji `-o`, przykład:
				- `find / -type d -o -size +1G` - znajdź pliki, które są katalogami lub mają rozmiar większy niż 1GB.
	- `[akcja]` - czyli co zrobić z wynikiem:
		- `-print` - wypisuje ścieżkę pliku na ekran (domyślnie) - `find / -name "passwd" -print`,
		- `-ls` - wypisuje szczegółu jak `ls -l` - `find /tmp -type f -ls`,
		- `-delete` - usuwa znalezione pliki - `find /tmp -name "*.tmp" -delete`,
		- `-exec` - wykonuje dowolne polecenie w znalezionym pliku - `find /home -name "*.jpg" -exec cp {} /backup/ \;`.
			- wyjaśnienie akcji `-exec` - `-exec` mówi: 'dla każdego znalezionego pliku, uruchom polecenie'. Przykład:
				- `find /tmp -name "*.sh" -exec ls -l {} \;`
					- `{}` - to jest zmienna, która oznacza "tu wstaw ścieżkę do znalezionego pliku", 
					- `\;` - to jest znak końca dla `-exec`, mówi: "tu kończymy polecenie do wykonania".
				- `find /etc/ -name "*.conf" -exec cp {} /tmp/backup \;` - znajdź wszystkie pliki `.conf` i skopiuj je do katalogu backup.

### locate
Narzędzie do używa wcześniej zbudowanej bazy danych, jest błyskawiczne, ale może nie widzieć najnowszych plików. Baza danych `locate` nie przeszukuje dysku - on czyta wcześniej przygotowaną bazę danych. Domyślnie jest to plik: `/var/lib/mlocate/mlocate.db`. 
- jak to działa:
	- system okresowo (zwykle raz dziennie) uruchamia `updatedb`,
	- `updatedb` skanuje cały system i zapisuje listę plików i katalogów do bazy, 
	- `locate` czyści zapytanie i szybko przeszukuje bazę, 
	- ręczna aktualizacja bazy: `sudo updatedb`.
- użycie `locate`: `locate passwd` - wyświetli wszystkie pliki, które mają w nazwie passwd,
- filtrowanie wyników: 
	- `-i` - ignoruj wielkość liter - `locate -i shadow`,
	- `-b` - pokaż tylko nazwę pliku, bez ścieżki - `locate -b passwd`,
	- `-r` - używaj wyrażeń regularnych - `locate -r ".*\.conf$"` 
		- `.*` - dowolny znaj, dowolna ilość razy. 
		- `\.` - kropka musi być poprzedzona `\` ponieważ kropka ma specjalne znaczenie, 
		- `conf$` - kończy się na `conf`
	- `-l N` - pokaż tylko N wyników (limit) - `locate -l 5 passwd`,
	- `-c` - zlicz wyniki, nie pokazuj ich - `locate -c passwd`,
- pułapki `locate`:
	- baza nie zawiera wszystkiego, domyślne `updatedb` pomija:
		- katalogi wymienione w `/etc/updatedb.conf` (np. `/tmp`, `/var/tmp`, `/proc`, `/sys`),
		- systemy plików zdalne (NFS, Samba),
		- katalogi bez uprawnień do odczytu,   
		Uwaga: atakujący często zostawiają pliki w `/tmp`. `locate` ich nie znajdzie, dopóki nie zmodyfikujesz konfiguracji `updatedb.conf`. 
	- jeśli plik został usunięty, `locate` nadal może go widzieć, do czasu następnego updatedb,
	pliki utworzone przed chwila są niewidoczne do czasu aktualizacji bazy. 

## Wnioski bezpieczeństwa
- pliki zapisywalne dla wszystkich (world-writable) to potencjalny wektor, ataku. Oznacza, że każdy może modyfikować plik, w tym atakujący,
- pliki SUID - są uruchomione z uprawnieniami właściciela, to potencjalne ryzyko, należy się upewnić, że plik jest może być uruchomiony tylko przez upoważnione osoby nie przez wszystkich, 
- pliki SGID - podobna sytuacja jak SUID, pliki są otwierane z uprawnieniami grupy, należy weryfikować czy wszyscy użytkownicy mający do nich dostęp uzasadniony, 
- pliki bez właściciela mogą być pozostałością po usuniętych użytkownikach lub stworzone przez atakującego, 
- duże pliki większe niż 10 MB mogą zawierać pakiety, narzędzia, dumping danych, 
- pliki w `/tmp` mogą być stworzone przez atakującego, należy weryfikować zawartość tego folderu pod kątem podejrzanych plików, 