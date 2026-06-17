## 09_basics_security_chceck

## Cel
Ten katalog zawiera praktyczną checklistę do szybkiej oceny stanu bezpieczeństwa systemu Linux. Narzędzie pomocne podczas rutynowych audytów, wstępnej analizy pierwszego kontaktu z nieznanym systemem.   
Lista powstała w oparciu o wiedzę zdobytą podczas pracy z poprzednimi katalogami repozytorium - łączy w sobie praktyczne polecenia z obszarów takich jak:
- analiza użytkowników i kont (`02_users_and_groups`),
- uprawnienia plików (`03_permissions_and_ownership`),
- konfiguracja sudo (`04_sudo_and_privilege_escalation_basics`),
- logi i system plików (`07_logging_basics`).

## Zawartość 
`basic_linux_audit_checklist` - podstawowe polecenia w zakresach:
- użytkownicy i konta,
- uprawnienia i własność plików, 
- procesy i usługi, 
- sieć i połączenia, 
- logi i system plików,
- konfiguracja ssh, 
- uprawnienia sudo, 
- jądro i bezpieczeństwo systemu,
- szybka weryfikacja,
- co robić po znalezieniu podejrzanych elementów. 

## Uwagi
- `to nie jest pełny audyt` - lista nie zastępuje narzędzi takich jak Lynis, OpenSCAP czy ClamAV,
- `to nie jest IR Playbook` - lista pomaga w detekcji, ale nie zastępuje procedur reagowania na incydenty, 
- `nie wszystkie ostrzeżenia są prawdziwe` - niektóre 'podejrzane' elementy moga być normalne dla konkretnego środowiska (np. serwer web będzie miał otwarty porty 80 i 443).

## Powiązane katalogi
- `02_users_and_groups` - szczegółowa analiza użytkowników, 
- `03_permissions_and_ownership` - uprawnienia i bity SUID/SGID,
- `04_sudo_and_privilege_escalation_basics` - konfiguracja sudo,
- `07_logging_basics` - analiza logów, 
- `08_tmp_and_file_locations` - katalogi tymczasowe i pliki. 

