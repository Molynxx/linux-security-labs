# visudo usage

## Cel
Zrozumienie jak ważny jest dedykowany edytor dla plików sudoers oraz jak się nim posługiwać.

## Czym jest visudo
Jest to bezpieczny edytor dla plików `/etc/sudoers` oraz `/etc/sudoers.d`. Program ten sprawdza składnię przed zapisem i nie pozwala zapisać gdy są w niej błędy. Ma też funkcję zabezpieczającą przed jednoczesną edycją pliku przez więcej niż jedną osobę - jeśli ktoś już edytuje plik, nie pozwoli nikomu innemu na edycję tego pliku.   
Nie należy używać zwykłych edytorów do edycji plików sudoers, ponieważ one nie sprawdzają składni, a każdy błąd może sprawić, że sudo przestanie działać. 

## Jak używać programu
- `sudo visudo` - edytuje plik `/etc/sudoers`,
- `sudo visudo -f /etc/sudoers.d/nazwa_pliku` - edytuje plik w katalogu `sudoers.d`.  
Flaga `-f` jest kluczowa, jeśli chcemy edytować plik `/etc/sudoers.d/nazwa_pliku`, ponieważ bez tej flagi, program ignoruje wszystkie argumenty i edytuje `/etc/sudoers`.   

## Działanie visudo
Jest to najbezpieczniejszy program do edycji plików sudoers ponieważ:
- tworzy tymczasową kopię pliku, 
- otwiera edytor (domyślnie `vi` - choć można to zmienić),
- po zapisaniu pliku sprawdza, czy składnia jest poprawna, 
- jeśli składnia jest poprawna zastępuje oryginalny plik, 
- jeśli będzie błąd w składni, nie pozwoli zapisać i wyświetli komunikat, pytając co robić:
	- `e` - wróć do edycji (aby poprawić błąd),
	- `x` - wyjdź z programu bez zapisywania, 
	- `q` - wyjdź (w przypadku gdy nie wprowadzono zmian w pliku).  
- po wpisaniu polecenia `sudo visudo -f /etc/sudoers.d/nazwa_pliku` program w pierwszej kolejności sprawdzi czy plik istnieje - a jeśli nie, utworzy go pod wskazaną ścieżką. 

## Wnioski bezpieczeństwa
- należy zawsze używać visudo do zmian w plikach `sudoers` i `sudoers.d`,
- należy pamiętać o fladze `-f`, jeśli celem jest edycja plików `sudoers.d`, 
- dobrą praktyką jest sprawdzanie składni po zmianach za pomocą `visudo -c /ścieżka/do/pliku`,
- można ustawić inny edytor (np. `export EDITOR=nano` w `~/.bashrc`).

## Przykład użycia case study
Administrator chce dodać uprawnienia dla użytkownika `jan` do pliku `/etc/sudoers.d/jan`.
- krok 1 - otworzenie/utworzenie pliku poleceniem `sudo visudo -f /etc/sudoers.d/jan`,
- krok 2 - wpisanie treści pliku za pomocą polecenia `jan ALL=(ALL) /usr/bin/apt update`,
- krok 3 - zapis pliku i wyjście,
- krok 4 - `visudo` sprawdza składnię, jeśli jest poprawna, plik zostaje zapisany. 
