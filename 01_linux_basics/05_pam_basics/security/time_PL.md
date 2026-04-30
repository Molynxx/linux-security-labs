# /etc/security/time.conf

## Cel laboratorium
Celem tego laboratorium jest zapoznanie się z plikiem `/etc/security/time.conf` oraz jego wpływem na bezpieczeństwo systemu. 

## Czym jest moduł pam_time.so
Jest to moduł PAM kontrolujący dostęp w czasie - nakłada ograniczenia kiedy użytkownik może się logować (dzień tygodnia, godzina). Pozawala zabezpieczyć system np. przed logowaniem poza godzinami pracy. Moduł jest umieszczany w typie account (w plikach: common-account, sshd, login) z flagą required. Przykład wpisu w PAM:  
	`account required pam_time.so`  

## Czym jest /etc/security/time.conf
Jest to plik konfiguracyjny dla modułu `pam_time.so`, to w nim ustawia się wszystkie limity, które egzekwuje moduł. Składnia wpisu:  
	`<usługa> ; <tty> ; <użytkownik> ; <czasy>`    
Pola:  
- `usługa` - nazwa usługi PAM (np. sshd, login, gdm-password, sudo, cron),
- `tty` - terminal (można użyć * dla wszystkich, tty* dla konsoli fizycznych),
- `użytkownicy` - nazwy użytkowników lub grupy (z `@`), ALL, !root (wszyscy oprócz roota),
- `czasy` - format [dni] [godziny]:
	- dni: Mo, Tu, We, Th, Fr, Sa, Su, Wk (weekend), Wd (weekday), Al (all days),
	- godziny: HHMM-HHMM (zakres, może przekraczać północ).
	Wpis określa kiedy dostęp jest dozwolony, nie ma możliwości wpisów określających odmowę dostępu. 

## Logi
Logi tego modułu znajdują się w pliku `/var/log/auth.log`. Przykładowe wpisy:  
	`pam_time(sshd:account): access denied for user janusz (time restricted)`  
	`pam_time(sshd:account): access denied for user anna (time restricted)`  
Należy monitorować odmowy o nietypowych porach, wielu odmów dla jednego użytkownika. To może oznaczać przejęcie konta lub brute force poza dozwolonymi godzinami.

## Wnioski bezpieczeństwa
- `pam_time.so`  nie zastępuje polityki dostępu, to tylko dodatkowa warstwa, 
- Root często jest wyłączany z ograniczeń (!root) - bo w przypadku awarii zawsze musi mieć dostęp, 
- Zakresy czasu mogą przekraczać północ, 
- kolejność reguł w `/etc/security/time.conf` ma znaczenie - pierwsze dopasowanie decyduje,
- brak reguły to brak dopasowania a to oznacza odmowę dostępu,
- należy weryfikować czy `/etc/security/time.conf` jest poprawnie skonfigurowany, jeśli moduł `pam_time.so` występuje w politykach dostępu. Jeśli moduł nie jest podpięty w PAM do żadnej usługi- nie działa. 

## Case study - time.conf

Kontekst:  
Firma ma serwer backupów.  
Użytkownicy z grupy backup_team mogą logować się przez SSH tylko w niedziele w godzinach 02:00 -04:00 (okno na nocny backup).  
Root może logować się każdym sposobem, zawsze (24/7).  
Wszyscy inni użytkownicy nie mają dostępu przez SSH w ogóle, mają natomiast dostęp do logowania lokalnego w godzinach od 6:00 do 18:00. 

Zadanie: Napisz reguły w `/etc/security/time.conf`.  

Rozwiązanie:  
Potrzebnych reguł - 2, ponieważ mamy dwa rodzaje użytkowników (grupa backup_team i root), którzy mają zupełnie inne godziny dostępu. Wszyscy inni nie mają dostępu, domyślnie brak dopasowania = odmowa dostępu.   
Reguły:  
	`sshd;*;@backup_team;Su0200-0400`  
	`\*;\*;root;Al0000-2400`  
	`login;tty*;!root;Wd0600-1800`  
