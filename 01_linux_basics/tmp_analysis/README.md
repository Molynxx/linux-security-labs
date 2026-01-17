# Analiza katalogu /tmp w Linuxie

## Czym jest /tmp
Katalog /tmp służy do przechowywania plików tymczasowych tworzonych przez system i aplikacje. Zawartość może być usuwana po restarcie systemu.

## Co właśnie sprawdzałam
- właściciela plików i katalogów
- uprawnienia, (szczególnie bit wykonywalny 'x')
- czas utworzenia plików
- kontekst (aktualizacja systemu)

## Obserwacje
- większość katalogów należała do użytkownika root
- pliki użytkownika nie miały uprawnień wykonywanych
- zauważalna różnica czasu (UTC vs czas lokalny)
- pliki powstały podczas 'apt full-upgrade'

## wnioski SOC
Nie stwierdzono oznak złośliwej aktywności.
Zachowanie zgodne z normalną pracą systemu.

## Bezpieczne czyszczenie /tmp
- restart sysemu
- 'sudo rm -rf /tmp/*'
- usuwanie plików starszych niż x dni

## Czego nie robić
- nie usuwać katalogu '/tmp'
- nie czyścić katalogu podczas instalacji lub aktualizacji

## Wnioski ogólne
- Katalog /tmp to katalog systemowy przechowjący pliki tymczasowe
- Zawartość katalogu /tmp jest zazwyczaj czyszczona automatycznie (np. przy restarcie systemu), jednak sam katalog nie powinien być usuwany, ponieważ jest wymagany do poprawnego działania systemu.
- Ze względu na otwarty charakter katalogu /tmp oraz fakt, że często zapisywane są w nim pliki wykonywalne i skrypty, warto regularnie monitorować jego zawartość pod kątem potencjalnych oznak złośliwego oprogramowania, nowych olików wykonywalnych lub nadużyć.
- uprawnienia katalogu /tmp i znaczenie dla bezpieczeństwa: 
	- Katalog /tmp zwykle ma uprawenienia drexrwxrwt (tzw. sticky bit). Oznacza to że:
		- każdy użytkownik może tworsyć i modyfikować własne pliki w katalogu,
		- nie może usuwać ani zmieniać plików należących do innych użytkowników,
		- dzięki temu pliki tymczasowe innych użytkowników sa chronione przed przypadkowym lub złośliwym usunięciem,
	