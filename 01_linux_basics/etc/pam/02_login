# /etc/pam.d/login - PAM dla usługi login

## Cel
Celem tego laboratorium jest zrozumienie działania ustawień PAM dla aplikacji login.

## Czym jest /etc/pam.d/login
To plik konfigurujący PAM dla aplikacji login, znajdującej się w /bin/login. Tu należy podkreślić czym jest aplikacja login. Login jest mechanizmem lokalnego logowania do systemu Linux, używanym przy dostępie do konsoli (TTY). Odpowiada za pełny proces logowania:
- uwierzytelnienie użytkownika, 
- sprawdzenie systemu i konta,
- utworzenie pełnej sesji użytkownika.
Ta aplikacja odpowiedzialna jest za podstawowe, niskopoziomowe zabezpieczenie systemy, które działa zawsze, nawet jeśli ktoś ominie SSH lub GUI. Podczas gdy SSH umożliwia jedynie dostęp zdalny do systemu, login robi wiele więcej: 
Login:
- pyta o użytkownika, 
- sprawdza hasło (przez MAP -> /etc./shadow),
- ustawia pełną sesję,
- ustawia UID, GID, grupy,
- ustawia środowisko,
- rejestruje sesję,
- obsługuje selinux - czyli kontekst bezpieczeństwa logowania i sesji,
- reaguje na tryb maintenance (konserwacji).

## Jakie zastosowanie ma plik /etc/pam.d/login
Plik ten jest zbiorem instrukcji jak aplikacja powinna działać w zgodzie z PAM, są w nim zdefiniowane sposoby obsługiwania modułów standardowych. W pliku tym mogę być załączone pliki common, które stanowią zbiór wspólnych zasad dla aplikacji oraz indywidualne modyfikacje działania modułów dla aplikacji login. 

## Kolejność wywoływania modułów przez login
LKogin wywołuje moduły w następującej kolejności:
- auth,
- account,
- session,
- password - tylko przy zmianie hasła.

## Jak wyglądają zapisy w /etc/pam.d/login
Zapisy w tym pliku wyglądają podobnie jak w przypadku pliku /etc/pam.d/sshd, jednak tutaj mogą ponadto wystąpić zapisy w takiej postaci:
	session [success=ok ignore=ignore module_unknown=ignore default=bad] pam_selinux.so
Ten zapis zawiera rozszerzoną flagę kontroli i mówi:
- typ session = dotyczy otwierania sesji,
- flaga - tu rozszerzona flaga kontrolna, czyli zamiast required, requisite, sufficient, optional, jest:
	- success=ok:
		- selinux istnieje,
		- jest włączony, 
		- odpowiada.
		To oznacza, że wszystko jest w porządku, sesja nie jest blokowana,
	- ignore=ignore:
		- selinux jest wyłączony,
		- sesja go nie dotyczy.
		To oznacza, że wszystko jest w porządku, sesja może być kontynuowana,
	-module_unknown=ignore:
		- selinux nie istnieje w systemie (nie jest zainstalowany),
		- dystrybucja nie używa selinux.
		Ten zapis nie oznacza błędu, mówi, że jeśli selinux nie istnieje w systemie to należy co pominąć, żeby nie nastąpiła blokada sesji,
	-default=bad:
		- wszystko inne niż powyższe oznacza błąd, wskazuje, że moduł nie działa poprawnie, że istnieją błędy np.:
			- błąd inicjalizacji, 
			- uszkodzona konfiguracja selinux,
			- wewnętrzny błąd modułu, 
			- brak dostępu do polityk.
			W tym przypadku uznawane jest to za niebezpieczne i sesja może zostac przerwana. 
		Rozszerzone flagi dla modułu pam_selinux.so działają wyłącznie z typem session.
Tego rodzaju zapisy z rozszerzonymi flagami kontroli pozwalają na większą elastyczność, ponieważ required nie rozróżnia 'nie ma' od 'nie działa'.

## Kolejność modułów w obrębie jednego typu 
Kolejność modułów w obrębie jednego typu ma ogromne znaczenie.
- kolejność modułów w typie auth:
	Przykład:
	auth optional pam_faildelay.so
	auth required pam_unix.so
	Moduł pam_faildelay.so to moduł wprowadzający opóźnienie po nieudanej próbie, w tym przypadku faildelay wykona się tylko po nieudanej próbie logowania a jego wykonanie lub nie, nie zablokuje kolejnych etapów.
- kolejność modułów w typie account:
	Przykład:
	account requisite pam_nologin.so 
	Powinien wystąpić po uwierzytelnianiu, jako pierwszy wpis typu account, ponieważ sprawdza czy system jest w trybie maitenance. Jeśli system jest w trybie maitenance, nie ma sensu iść dalej.
- kolejność modułów w typie session:
	Przykładowa kolejność logiczna:
	- ustalenie kontekstu bezpieczeństwa pam_selinux.so,
	- środowisko pam_env.so,
	- limity pam_limits.so,
	- UID sesji pam_loginuid.so,
	- grupy, mail motd, itd. 
- pliki common - na końcu.

## pam_nologin.so
To istotny moduł, który warto zrozumieć lepiej. Moduł ten może być stosowany w plikach konfiguracji PAM dla każdej aplikacji. To jest moduł sprawdzający czy system nie jest w trybie mainenance. Administrator systemu, prowadząc prace serwisowe lub naprawcze może potrzebować spokoju w systemie. Nologin pozwala mu ograniczyć dostęp do systemu dla wszystkich użytkowników poza root, żeby jego działa mogły zakończyć się bezpiecznie. Jeśli plik PAM dla aplikacji ma zapis:
	account pam_nologin.so
to w pierwszej kolejności zostanie sprawdzone czy system nie jest w stanie maintenance, jeśli tak jest, żaden użytkownik się nie zaloguje. Sprawdzenie czy system jest w stanie mainetance polega na sprawdzeniu czy plik /etc/nologin istnieje.
Jak to działa:
Administrator tworzy plik tekstowy, który wyświetli na ekranie użytkownika komunikat, np.: 'system w trakcie prac serwisowych'. By stworzyć taki plik wystarczy, że użytkownik zalogowany jako root posłuży się takim poleceniem: 
	sudo echo "System w trakcie prac serwisowych" > /etc/nologin
lub użytkownik zalogowany jako sudo:
	sudo sh -c " echo 'system w trakcie prac serwisowych" > /etc/nologin"
to polecenie zapisuje plik nologin jako root. To bezpieczniejsza forma niż logowanie się na root-a.
Po zakończonych pracach administrator usuwa plik nologin poleceniem rm przywracając dostęp użytkownikom. 

## Czerwone flagi SOC
- usunięcie modułu pam_nologin.so,
- zmiany w pam_securitty.so,
- manipulacje w typie session (zmiana limitów, grup, selinux, itd),
- zmiany w plikach common,
- login działający gdy system powinien być zamknięty (backdoor).

## Wnioski
- W ustawieniach kolejności modułów, dobrą praktyką jest wpisywanie w plikach konfigurujących PAM dla aplikacji w pierwszej kolejności ustawień specyficznych dla danej aplikacji a dopiero później globalnych (common). 
- Bardzo ważne jest sprawdzanie kto i kiedy zmienił plik nologin w katalogu etc za pomocą polecenia stat (mtime,ctime, access),
- Należy sprawdzać czy moduły nologin są poprawnie ustawione w plikach PAM, czy nie zostały usunięte oraz czy moduł pam_nologin.so nie został usunięty,
- Należy sprawdzać metadane plików common oraz PAM dla aplikacji,
- Należy sprawdzać treść plików common oraz PAM dla aplikacji. 

## Tabela modułów dla pliku login
<table>
	<thead>
		<tr>
			<th>Typ PAM</th>
			<th>Moduł</th>
			<th>Flaga</th>
			<th>Rola w logowaniu lokalnym (login)</th>
		</tr>
	</thead>
	<tbody>

		<tr>
			<td>auth</td>
			<td>pam_unix.so</td>
			<td>required</td>
			<td>
				Sprawdza hasło użytkownika na podstawie <code>/etc/shadow</code>.<br>
			Podstawowy mechanizm uwierzytelniania przy logowaniu lokalnym.
			</td>
		</tr>

		<tr>
			<td>auth</td>
			<td>pam_faildelay.so</td>
			<td>optional</td>
			<td>
				Wprowadza opóźnienie po nieudanej próbie logowania.<br>
				Utrudnia ataki brute force, nie wpływa na decyzję PAM.
			</td>
		</tr>

		<tr>
			<td>account</td>
			<td>pam_nologin.so</td>
			<td>requisite</td>
			<td>
				Blokuje logwanie zwykłych użytkowników, gdy ustnieje plik <code>/etc/nologin</code>.<br>
				Root nadal może się zalogować.
			</td>
		</tr>

		<tr>
			<td>account</td>
			<td>pam_unix.so</td>
			<td>requred</td>
			<td>
				Sprawdza status konta (blokada, wygaśnięcie, polityka konta).<br>
				Hasło może być poprawne, ale konto nadal może być niedozwolone.
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_limits.so</td>
			<td>required</td>
			<td>
				BNakłada limity zasobów (CPU, pamięć, liczba procesów).<br>
				Chroni system przed nadużyciami i DoS ze strony użytkownika.
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_env.so</td>
			<td>required</td>
			<td>
				Ustawia zmienne środowiskowe sesji użytkownika.<br>
				Wpływa na sposób uruchamiania powłoki i programów.
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_loginuid.so</td>
			<td>required</td>
			<td>
				Przypisuje identyfikator logowania do sesji.<br>
				Kluczowe dla audytu i systemóiw monitoringu (auditd).
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_selinux</td>
			<td>required</td>
			<td>
				Ustawia kontekst SElinuc dla sesji użytkownika.<br>
				Zmienia egzekwowanie polityki MAC po zalogowaniu.
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_lastlog</td>
			<td>optional</td>
			<td>
				Rejestruje i wyświetla informacje o ostatnim logowaniu użytkownika.<br>
				Pomocne przy analizie incydentów.
			</td>
		</tr>

		<tr>
			<td>session</td>
			<td>pam_motd</td>
			<td>optional</td>
			<td>
				Wyświetla komunikaty systemowe po zalogowaniu.<br>
				Nie wpływa na bezpieczeństwo logowania.
			</td>
		</tr>

	</tbody>
</table>