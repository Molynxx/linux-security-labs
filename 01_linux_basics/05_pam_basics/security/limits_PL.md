# /etc/security/limits.conf

## Cel laboratorium
Celem jest zapoznanie się z plikiem konfiguracyjnym modułu `pam_limits.so` i znajdującymi się w nim opcjami. 

## Czym jest moduł pam_limits.so
Jest to moduł stosowany w typie session, którego zadaniem jest nałożenie limitów zasobów na sesję, celem zabezpieczenia działania systemu. 

## Pliki konfiguracyjne modułu pam_limits.so
Główny plik konfiguracyjny znajduje się w `/etc/security/limits.conf`. To właśnie w nim określa się rodzaje zabezpieczeń oraz wartości im przypisanych. Format wpisu w pliku wygląda następująco:    
		[domain] [type] [item] [value]     
gdzie:  
Domain - użytkownik, grupa (`@group`), wildcard (`*`),  
Type - `soft` (ostrzeżenie), `hard` (maksymalny limit), `-` (oba),  
Item - `core`, `nproc`, `nofile`, `memlock`, `as`, `cpu`, itd,  
Value - wartość.   

Poza plikiem głównym można konfigurować także pliki dodatkowe jeśli jest potrzeba np. zrobienia wyjątków dla użytkownika/grupy. Te dodatkowe pliki znajdują się w lokalizacji `/etc/security/limits.d/*conf`. Format zapisu jest w nich taki sam jak w pliku głównym. Po co istnieją - ponieważ nie trzeba mieszać w jednym pliku limitów dla rożnych aplikacji. Jest to ważne ze względu na czytelność plików, mamy plik `/etc/security/limits.conf` z ustawieniami globalnymi dla wszystkich, a gdy potrzebujemy np. dla danej aplikacji zmienić wartości robimy to w odpowiednio nazwanym pliku w `/etc/security/limits.d/`. Taki układ znacznie ułatwia audyt i porządkuje całą konfiguracje w sposób czytelny i łatwy do zmiany. 
Jak to działa: 
- najpierw czytany jest `/etc/security/limits.conf`,
- następnie wczytywane są wszystkie pliki .conf z katalogu `/etc/security/limits.d` - w kolejności alfabetycznej, 
- ustawienia plików w `/etc/security/limits.d/` mają pierwszeństwo nad ustawieniami w pliku `/etc/security/limits.conf`, jeśli dotyczą tego samego użytkownika lub grupy. 
Przykład: 
- anna jest w grupie admin,
- grupa admin w pliku `/etc/security/limits.conf` ma ustawienie ograniczenia ilości procesów 500,
- chcemy jednak żeby anna miała to ograniczenie ustawione 250
- wtedy konfigurujemy plik `/etc/security/limits.d/anna.conf` gdzie zmieniamy tą wartość tylko dla anna.  
UWAGA: Pliki czytane są z góry do dołu, kolejne wpisy mogą nadpisać poprzednie tylko jeśli dotyczą tego samego typu (soft/hard) i tego samego elementu (item)!

## Kluczowe opcje pliku limits.conf
- `nproc` - maksymalna liczba procesów, zagrożenie: fork bomb (DoS lokalny),
- `nofile` - maksymalna liczba otwartych plików, zagrożenie: aplikacja może nie działać, 
- `core` - rozmiar pliku core dump (0=wyłączony), zagrożenie: wyciek pamięci, informacje wrażliwe, 
- `memlock` - maksymalna zablokowana pamięć (kernel), zagrożenie: DoS lokalny,
- `as` - maksymalna pamięć wirtualna (address space), zagrożenie: DoS lokalny,
- `cpu` - maksymalny czas CPU (minuty), zagrożenie: DoS lokalny.

## Przykład konfiguracji pliku limits.conf
\*			soft	core	0  
\*			hard	nproc	1024  
@admins		hard	nproc	4096  
student		soft	nofile	256

## Wnioski bezpieczeństwa
- właściwa konfiguracja opcji jest kluczowa dla bezpieczeństwa systemu, to rodzaj zabezpieczenia przed działaniami użytkownika mającymi wpływ na działanie systemu, 
- `core` ustawione na 0 wyłącza pliki core.dump - to dobra praktyka przeciwko wyciekom danych, 
- należy sprawdzać czy moduł `pam_limits.so` znajduje się w typie session oraz czy plik limits.conf nie zawiera błędów składniowych.

## Case study - konfiguracja (fragment /etc/security/limits.conf)

Plik `/etc/security/limits.conf`:  
\*				soft	nproc		100  
\*				hard	nproc		150  
\*				soft	nofile		2048  
\*				hard	nofile		4096  
@developers		hard	nproc		2048  
@developers		hard	nofile		32768   
student			soft	nofile		1024  
student			hard	nofile   	2048   
\*				soft 	core		0  
root			soft	core		0  

Dodatkowe informacje:  
- grupa developers istnieje, należą do niej użytkownicy: anna, jan, devops,
- Użytkownik student nie należy do grupy developers,
- system produkcyjny.  

Analiza:  
- anna jak wszyscy ma ustawiony nproc na wartość 150 (hard), ponieważ jednak należy do grupy developers, a wpis dotyczy tego samego typu (hard) oraz tego samego item, ustawienie to zostaje nadpisane przez ustawienia dla grupy developers, do której użytkownik anna należy, dlatego też obowiązujący limit nproc dla anna wynosi 2048,
- maksymalny limit nofile dla użytkownika student to 2048, ponieważ wpis dla tego użytkownika nadpisuje wcześniejszy wpis dla wszystkich (ten sam typ i item),
- maksymalny limit nofile dla użytkownika devops wynosi 32768, wpis dla grupy nadpisuje wartość dla wszystkich (ten sam typ i item). Jednak ustawienie jest ryzykowne, trudno bowiem wyobrazić sytuację, w której taka ilość otwartych plików była by konieczna. W przypadku kompromitacji konta, atakujący może otworzyć ponad 30 tysięcy gniazd sieciowych, powodując DoS dla usług sieciowych. Zalecane jest zmniejszenie wartości tej opcji do max 4096. 
- limit core dla root jest w typie soft, co oznacza, że może root może go podwyższyć (np. ulimit -c unlimited). To nie jest bezpieczne ustawienie, w systemach produkcyjnych celem uniknięcia core dump należy użyć dla core typu hard i wartości 0, aby całkowicie wyłączyć core dump.
