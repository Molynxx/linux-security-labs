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
Zachowanie zgodne z nirmalną pracą systemu.

## Bezpieczne czyszczenie /tmp
- restart sysemu
- 'sudo rm -rf /tmp/*'
- usuwanie plików starszych niż x dni

## Czego nie robić
- nie usuwać katalogu '/tmp'
- nie czyścić katalogu podczas instalacji lub aktualizacji