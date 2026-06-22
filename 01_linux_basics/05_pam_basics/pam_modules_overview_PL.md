# Moduły PAM

## Cel laboratorium
Zapoznanie się z modułami typów PAM oraz jakie pełnią role w procesie uwierzytelnienia, autoryzacji, sesji i zmiany hasła.

## Czym są moduły PAM 
Moduły PAM to komponenty odpowiedzialne za wykonanie konkretnych operacji w ramach danego typu (`auth`, `account`, `session`, `password`). Każdy moduł realizuje określoną funkcję, np. uwierzytelnianie użytkownika, kontrolę dostępu lub zarządzanie sesją, a ich kolejność oraz opcje wpływają na końcowe działanie procesu PAM. Zachowanie modułów może być modyfikowane za pomocą opcji, które określają sposób ich działania (np. tryb pracy lub politykę bezpieczeństwa).

## Moduły typu auth
Ten typ odpowiada za uwierzytelnianie użytkowników. Wśród typowych dla tego typu modułów można wymienić:   

| Moduł | Opis | Ważne opcje |
|-------|------|-------------|
| pam_unix.so | lokalne uwierzytelnianie (hasło z `/etc/shadow`) | nullok, try_first_pass, use_first_pass |
| pam_ldap.so | uwierzytelnianie LDAP | use_first_pass |
| pam_sss.so | uwierzytelnianie SSSD (LDAP/Kerberos) | use_first_pass |
| pam_faillock.so | ochrona przed brute force | preauth, authfail, authsucc, deny, unlock_time |
| pam_faildelay.so | opóźnienie po błędzie | delay= |
| pam_deny.so | zawsze odrzuca dostęp | brak |
| pam_permit.so | zawsze pozwala | brak |   

## Moduły typu account
Typ ten przeprowadza proces autoryzacji użytkownika za pomocą modułów:   

| Moduł | Opis | Ważne opcje |
|-------|------|-------------|
| pam_unix.so | sprawdza stan konta (np. wygaszenie, blokada, dostępność) | brak |
| pam_sss.so | kontrola konta (LDAP/AD) | brak |
| pam_nologin.so | blokada logowania globalnego | brak |
| pam_faillock.so | sprawdza blokadę konta | konfiguracja globalna (faillock.conf) |
| pam_access.so | kontrola dostępu (IP, user) | brak |
| pam_time.so | ograniczenia czasowe | brak |  

## Moduły typu session
To typ odpowiedzialny za przygotowanie, ustawienie i zamknięcie sesji użytkownika. Znajdują się tu m.in. moduły:  

| Moduł | Opis | Ważne opcje |
|-------|------|-------------|
| pam_limits.so | limity zasobów (np. liczba procesów, pamięć, CPU) | brak |
| pam_env.so | ustawienie zmiennych środowiskowych dla sesji | brak |
| pam_lastlog.so | informacja o ostatnim logowaniu | silent |
| pam_motd.so | wyświetla wiadomość dnia | brak |
| pam_systemd.so | integracja z systemd | brak |
| pam_loginuid.so | identyfikacja sesji (audit) | brak |  

## Moduły typu password
Typ ten nie bierze udziału bezpośrednio w procesie logowania, lecz odpowiada za zmianę i aktualizację danych uwierzytelniających (np. hasła). Moduły typu password:   

| Moduł | Opis | Ważne opcje |
|-------|------|-------------|
| pam_unix.so | zmiana hasła lokalnego | remember, use_authtok |
| pam_pwquality.so | polityka haseł | minlen, retry, dcredit, ucredit |
| pam_pwhistory.so | historia haseł | remember |
| pam_sss.so | zmiana hasła LDAP | use_authtok |  

## Wnioski bezpieczeństwa
- sam dobór modułów nie jest wystarczający - kluczowe znaczenie ma ich kolejność, flagi kontrolne oraz opcje, które definiują ich rzeczywiste działanie, 
- ten sam moduł może pełnić różne funkcje w zależności od typu PAM (`auth`, `account`, `session`, `password`) oraz użytych opcji,
- brak odpowiednich modułów ochronnych (np. `pam_faillock.so`) zwiększa ryzyko ataków brute force,
- niepoprawne użycie flag kontrolnych (np. `sufficient` w module uwierzytelniającym) może prowadzić do pominięcia istotnych mechanizmów bezpieczeństwa,
- moduły takie jak `pam_permit.so` mogą stanowić poważne zagrożenie w systemach produkcyjnych, ponieważ mogą umożliwić obejście uwierzytelnienia,
- brak odpowiednich opcji modułów (np. `authfail` w `pam_faillock.so`) może powodować, że mechanizmy bezpieczeństwa nie działają zgodnie z założeniami,
- poprawna konfiguracja PAM wymaga zrozumienia zależności między modułami, a nie tylko ich pojedynczego działania. 