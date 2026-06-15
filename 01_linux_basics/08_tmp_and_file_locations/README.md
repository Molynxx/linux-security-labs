# 08_tmp_and_file_locations

## Cel 
Zrozumienie specyfiki katalogów tymczasowych w systemie Linux oraz opanowanie umiejętności efektywnego wyszukiwania plików z perspektywy analityka SOC/Blue Team.

## Zakres
- `tmp_analysis` - analiza katalogów tymczasowych `/tmp`, `/var/tmp`, `/dev/shm`, ich znaczenie w systemie, zagrożenia, case study, 
- `file_searching` - kompleksowe omówienie narzędzi `find` i `locate`: składnia, warunki (`-name`, `-type`, `-mtime`), akcje (`-exec`, `-delete`), filtrowanie, pułapki.

## Dlaczego to ważne (SOC/IR)
- `/tmp`,`/var/tmp` i `/dev/shm` - atakujący często wykorzystują te lokalizacje do wrzucania skryptów, narzędzi (np. reverse shell) lub jako miejsce na pliki tymczasowe podczas ataku. Weryfikacja ich zawartości to podstawa, 
- `find` - precyzyjne wyszukiwanie plików po atrybutach (SUID, world-writable, brak właściciela, rozmiar, czas modyfikacji) pozwala na szybkie wykrycie potencjalnych wektorów ataku lub obecności intruza,
- `locate` - szybkie wyszukiwanie po nazwie, ale wymaga świadomości jego ograniczeń (baza danych może być nieaktualna, nie indeksuje `/tmp`). 