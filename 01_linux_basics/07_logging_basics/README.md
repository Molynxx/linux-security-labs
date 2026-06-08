# 07_logging_basics

## Cel
Zrozumienie, jak czytać i filtrować logi systemowe w Linuxie, kluczowe pliki logów oraz narzędzia do ich analizy.

## Zakres
- `var_log_structure` - struktura katalogu `/var/log`, ważne pliki (`auth.log`, `syslog`, `kern.log`, `dpkg.log`, logi usług), 
- `auth_log_analysis` - analiza logów uwierzytelniania (`auth.log`), kluczowe wpisy, monitoring, case study,
- `syslog_analysis` - analiza logów systemowych (`syslog`), różnica między `auth.log` a `syslog`, kluczowe wpisy (kernel, systemd, cron), case study,
- `journalctl_basics` - narzędzie `journalctl`, włączenie trwałych logów, podstawowe opcje, filtrowanie, case study, 
- `log_filtering_with_grep`- filtrowanie logów za pomocą `grep` (podstawowe opcje, praktyczne przykłady). 

## Dlaczego to ważne (SOC/IR)
- `Logi to podstawa detekcji` - nieudane logowania, ataki brute-force, eskalacja uprawnień, persistence, 
- `journalctl` - centralne logi systemd, ale domyślnie nietrwałe (wymaga konfiguracji `Storage=persistent`),
- `grep` - podstawowe narzędzie do filtrowania logów w czasie rzeczywistym i analizy historycznej.

## UWAGA
- `auth.log`i `syslog` wymagają `rsyslog` (w Kali domyślnie nie ma - logi są tylko w `journalctl`),
- w środowiskach produkcyjnych (Ubuntu/Debian) logi tekstowe są domyślnie włączone.