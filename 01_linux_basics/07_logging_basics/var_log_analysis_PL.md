# var_log_analysis

## Cel
Zrozumienie jak czytać wpisy z pliku `auth.log` i poprawnie je interpretować.

## Czym jest auth.log
Jest to plik logów uwierzytelniania. Zawiera dane związane z logowaniem, `sudo`, PAM, SSH i zmianami użytkowników. To najważniejszy plik do analizy włamań, nieudanych logowań, eskalacji uprawnień. 

## Kluczowe wpisy
- `Failed password for root from x.x.x.x` - oznacza nieudaną próbę logowania na konto roota. Jeśli widocznych jest wiele nieudanych prób może to oznaczać brute force, 
- `Failed password for invalid user` - oznacza próbę logowania na nieistniejące konto. Oznacza, próbę skanowania użytkowników, 
- `Accepted password for user from x.x.x.x` - oznacza udane logowanie. Jeśli IP jest spoza zaufanej sieci lub o nietypowej porze, może oznaczać kompromitację konta, 
- `Accepted publickey for user from x.x.x.x` - oznacza udane logowanie przez klucz SSH. Podobnie jak wyżej, nietypowe IP i godziny, mogą świadczyć o kompromitacji konta, 
- `sudo: user: TTY=pts/0 ; COMMAND=/bin/bash` - oznacza użycie `sudo`. Nietypowe polecenia na które należy zwrócić uwagę to `sudo vim`, `sudo less`, `sudo find`. Wszystkie te polecenia mogą wywołać powłokę jako root, 
- `user : TTY=pts/0 ; PWD=/home/user ; USER=root ; COMMAND=/bin/su` - oznacza użycie `su`. Podejrzane przełączanie na roota, 
- `pam_unix(sshd:auth): authentication failure` - oznacza błąd uwierzytelniania PAM. Jeśli takich wpisów jest wiele z jednego IP, oznacza atak brute force,
- `pam_faillock(sshd:auth): user root locked` - oznacza, że konto zostało zablokowane przez `faillock` po wielu nieudanych próbach, 
- `session opened for user root` - oznacza rozpoczęcie sesji roota. To podejrzane jeśli jest nieoczekiwane,
- `session closed for user root` - oznacza zakończenie sesji dla roota. 

## Monitoring - podstawowe polecenia
- `sudo grep "Failed password" /var/log/auth.log | tail -20` - ostanie 20 nieudanych logowań, 
- `sudo grep "Failed password for root" /var/log/auth.log | grep "45.33.22.11"` - nieudane logowania na roota z konkretnego IP, 
- `sudo grep "Accepted" /var/log/auth.log` - udane logowania dla wszystkich użytkowników, 
- `sudo grep "Accepted*root" /var/log/auth.log` - logowania roota przez SSH,
- `sudo grep "COMMAND" /var/log/auth.log` - użycie sudo, 
- `sudo grep "COMMAND" /var/log/auth.log | grep -E "vim|less|tar|awk"` - sudo na podejrzanych programach. 

## Case study
Fragment `auth.log`:
```
Jun 6 10:00:10 server sshd[1234]: Failed password for root from 45.33.22.11 port 54321 ssh2  
Jun 6 10:00:13 server sshd[1235]: Failed password for root from 45.33.22.11 port 54322 ssh2  
Jun 6 10:00:15 server sshd[1236]: Failed password for invalid user admin from 45.33.22.11 port 54323 ssh2  
Jun 6 10:05:10 server sshd[1500]: Accepted password for root from 192.168.1.100 port 54321 ssh2  
Jun 6 10:06:00 server sudo: jan : TTY=pts/0 ; COMMAND=/bin/su -
```  
Analiza:  
- widać tutaj dwa działania atakującego: brute force na konto root oraz skanowanie czy istnieje konto admin,
- żadna z czynności atakującego się nie powiodła, 
- podejrzane IP to 45.33.22.11,
- podejrzane zachowanie: użytkownik jan użył polecenia `sudo` po to by użyć polecenia `su -`, celem włączenia powłoki roota. Jest to podejrzane ponieważ mając dostęp do roota poprzez `sudo` mógł po prostu użyć polecenia `sudo -i`. Tu ma znaczenie różnica pomiędzy `sudo` i `su`. Należy pamiętać, że `sudo` jest zaprojektowane jako narzędzie do audytu (wykonane polecenia z użyciem `sudo` są logowane w `auth.log`). Inaczej jest z `su`, które tworzy nową powłokę, tak jakby jan zalogował się na nowo jako root. Wszystkie polecenia wydane w tej powłoce są logowanie jako działania roota, nie jana, ponieważ loginuid procesu po `su -` będzie wskazywał na roota (UID 0) a nie jana. Takie działanie sprawia, że osoba, zalogowana na koncie jan, zaciera za sobą ślady poprzez użycie `sudo su`, by utrudnić wykrycie jego działań.   
Zalecane działania:
- należy sprawdzić wpisy dla jana w `/var/log/auth.log`, czy logował się z podejrzanego IP, 
- sprawdzić uprawnienia jana - `sudo -l` i zweryfikować czy powinien mieć takie uprawnienia, 
- sprawdzić `.bash_history` dla jan, czy nie wykonywał innych podejrzanych poleceń, za pomocą polecenia `sudo cat /home/jan/.bash_history`,
- sprawdzić czy konto jan jest skompromitowane, przeglądając historię logowań (dni, godziny, IP, polecenia). Jeśli konto zostało skompromitowane należy zmienić hasło oraz sprawdzić `authorized_keys` pod kątem dodania przez atakującego swoich kluczy. 
- jeśli konto jan jest skompromitowane i ma uprawnienia `sudo`, należy sprawdzić wszystkie kluczowe dla bezpieczeństwa pliki pod kątem zmian (passwd, shadow, sshd, PAM, authorized_keys dla roota, itp).
