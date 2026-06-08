# log_filtering_with_grep

## Cel
Utrwalenie umiejętności filtrowania logów za pomocą `grep` (niezbędne w SOC).

## Podstawowe opcje grep
- `grep "tekst" plik` - wyświetla linie zawierające "tekst", przykład:
	- `grep "Failed password" /var/log/auth.log`,
- `grep -v "tekst" plik` - wyświetla linie NIE zawierające "tekst", przykład:
	- `grep -v "Accepted" /var/log/auth.log`,
- `grep -E "tekst|tekst2" plik` - OR - linie zawierające "tekst1" lub "tekst2", przykład:
	- `grep -E "Failed|Accepted" /var/log/auth.log`,
- `grep -i "tekst" plik` - ignoruje wielkość liter, przykład:
	- `grep -i "error" /var/log/syslog`,
- `tail -f plik | grep "tekst"` - monitorowanie na żywo + filtrowanie, przykład:
	- `sudo tail -f /var/log/auth.log | grep "Failed"`. 

## Praktyczne przykłady
- `sudo grep "Failed password" /var/log/auth.log | tail -20` - ostatnie 20 nieudanych logowań, 
- `sudo grep "Failed password for root" /var/log/auth.log | grep "45.33.22.11"` - Nieudane logowania  na roota z konkretnego IP,
- `sudo grep "Accepted" /var/log/auth.log` - udane logowania (wszyscy użytkownicy),
- `sudo grep -E "/tmp|/home" /var/log/auth.log` - podejrzane lokalizacje, 
- `sudo grep "COMMAND" /var/log/auth.log | grep -E "wget|curl|nc|python"` - podejrzane polecenia w sudo, 
- `sudo tail -f /var/log/auth.log | grep -E "Failed|Accepted"`

## Wnioski
- `grep` to podstawowe narzędzie do filtrowania logów, 
- `-E` pozwala na bardziej złożone filtrowanie, szukanie wielu rzeczy na raz, 
- `tail -f` + `grep` daje monitorowanie na żywo. 