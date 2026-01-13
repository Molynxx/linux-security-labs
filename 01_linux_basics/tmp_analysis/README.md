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
Nie stwierdzono oznak złośliwej aktywnośći.
Zachowanie zgodne z normalną pracą systemu.

## Bezpieczne czyszczenie /tmp
- restart sysemu
- 'sudo rm -rf /tmp/*'
- usuwanie plików starszych niż x dni

## Czego nie robić
- nie usuwać katalogu '/tmp'
- nie czyścić katalogu podczas instalacji lub aktualizacji

## Wnioski ogólne
- Ktalog /tmp to katalog systemowy przchowjący pliki tymczasowe
- Zawartość katalogu /tmp jest zazwyczaj czyszczona automatycznie (np. przy restarcie systemu), jednak sam katalog nie powinien być usuwany, ponieważ jest wymagany do poprawnego działania systemu.
- Ze względu na otwarty charaker katalogu /tmp oraz fakt, że często zapisywane są w nim pliki wykonywalne i skrypty, warto regularnie monitorować jego zawartość pod kątem potencjalnych oznak złośliwego oprogramowania lub nadużyć.