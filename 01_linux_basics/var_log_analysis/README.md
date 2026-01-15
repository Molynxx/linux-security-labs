# /var/log - analiza katalogu logów sytytemowych

## Cel 
Celem tego laba było zrozumienie czym jest katalog /var/log oraz jakie typy zdarzeń są tam zapisywane z punktu widzenia SOC / Blue Team.

## Czym jest  /var/log
/var/log zawiera pliki dzienników systemowych, w których system Linux zapisuje informacje o:
- logowaniech użytkowników 
- działaniach usług 
- błędach systemowych 
- zdarzeniach bezpueczeństwa

## Najważniejsze pliki z punktu widzenie bezpieczeństwa
- auth.log - logowania użytkowników, sudo, ssh
- syslog - ogólne zdarzenia systemowe
- kern.log - zdarzenia jądra systemu 
- wtmp - historia logowań
- btmp - nieudane próby logowania (teoria)

## Wnioski bezpieczeństwa 
- logi pozwalają odtworzyć historię zdarezeń w sysytemie 
- analiza /var/log jest kluczowa w pracy SOC i Incident Response 
- ważne jest rozróżnienie logów systemowych i logów aplikacji

## Oranieczenia
- nie wszystkie logi są czytelne bez doświadczenia
- analiza treści logów wymaga dalszej nauki (Jouranlctl, grep)
- Uwaga: w Kali Linux logi uwierzytalniające (np. sudo, logowania) nie są domyślnie zapisywane w pliku auth.log, lecz przechowywane w systemmd-journal (journalctl)