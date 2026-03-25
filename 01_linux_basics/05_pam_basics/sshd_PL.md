# /etc/pam.d/sshd - PAM dla usługi SSH

## Cel
Zrozumienie polityki PAM dla usługi SSH.

## Czym jest /etc/pam.d/sshd
Gdy UsePam 'yes' - SSH deleguje część decyzji uwierzytelniania i kontroli dostępu do PAM. Parametry SSH nadal obowiązują (np. PermitRootLogin, PasswordAuthentication), ale PAM może dodatkowo zablokować dostęp, nawet jeśli SSH by go dopuścił. SSH nie decyduje sam – przekazuje żądania do PAM, a PAM decyduje, czy użytkownik może się zalogować i na jakich warunkach.
Plik `/etc/pam.d/sshd` nie jest plikiem konfiguracyjnym SSH, tylko reguluje politykę PAM dla tej usługi.

## Jakie zastosowanie ma plik `/etc/pam.d/sshd`
Ten plik nie:
- ustala długości haseł,
- nie liczy prób logowania (robi to common-auth i moduł `pam_faillock.so`),
- nie blokuje kont (robi to common-account przy pomocy modułu `pam_faillock.so`).
Plik mówi, do kogo przesłać żądanie PAM, tzn.:
- zwykle zawiera dołączone pliki polityki PAM za pomocą @include:
- common-auth,
- common-account,
- common-password,
- common-session,
Dzięki tym załączonym plikom SSH wie, gdzie znajdują się mechanizmy uwierzytelniania. Może zawierać indywidualne zapisy dla danej aplikacji, jeśli jest to konieczne lub zalecane, 

## Kolejność wywołania plików przez SSH
- auth,
- account,
- session,
- password (tylko przy zmianie hasła; jeśli plik zawiera ustawienia retry, to dotyczy liczby prób ustawienia hasła zgodnego z polityką PAM, nie logowania).

## Jak wyglądają zapisy w PAM
Plik zawiera zarówno linie zakomentowane - nieaktywne (#), jak i aktywne. 
Przykład:
  <typ> <flaga> <moduł> [opcje]
  @include common-auth

## Jak działają zapisy w pliku PAM dla aplikacji (przykład SSH)
Przykład: 
auth sufficient pam_ssh_key.so
- poprawny klucz SSH podczas logowania → sukces, PAM kończy fazę auth, kolejne moduły auth mogą zostać pominięte. Uwaga: Użycie `pam_ssh_key.so` z flagą `sufficient` jest dopuszczalne tylko wtedy, gdy moduły ochronne (`pam_faillock.so`, `pam_faildelay.so`) znajdują się przed nim. W przeciwnym razie logowanie kluczem może omijać mechanizmy ochronne.
- niepoprawny klucz SSH → sprawdzane są kolejne moduły uwierzytelniania (np. hasło).

## Logowanie SSH krok po kroku
- Klient SSH próbuje się zalogować,
- sshd sprawdza UsePam,
- Jeśli UsePam yes:
	- sshd woła /etc/pam.d/sshd,
- PAM wykonuje:
	- auth – czy hasło/klucz poprawny,
	- account – czy konto nie jest zablokowane,
	- session – otwarcie sesji,
- Jeśli którykolwiek etap zwróci fail → brak logowania.

## Czerwone flagi dla SOC/IR
- usunięcie @include common-auth lub innych plików common,
- zmiana kontroli z requisite na sufficient w modułach krytycznych,
- brak modułów ochronnych (`pam_faillock.so`),
- UsePam `no` w pliku `sshd_config`,
- nieautoryzowane modyfikacje pliku /etc/pam.d/sshd lub plików common-*.

## Wnioski
- należy regularnie sprawdzać konfigurację PAM w `/etc/pam.d/` i monitorować metadane plików (`mtime`/`ctime`) – każda zmiana może wpływać na logowania i bezpieczeństwo kont.
- kolejność flag kontrolnych i modułów ma kluczowe znaczenie dla odporności na brute force i ataki typu DoS.
- monitoring logów (`/var/log/auth.log`) jest krytyczny dla SOC/IR – każda próba uwierzytelnienia powinna być rejestrowana.
- każda zmiana w plikach PAM powinna być zatwierdzona i wersjonowana (np. Git), ponieważ nieautoryzowane zmiany mogą umożliwić obejście polityki bezpieczeństwa lub wprowadzić backdoor.
