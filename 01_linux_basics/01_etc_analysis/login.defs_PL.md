# /etc/login.defs

## Cel
Zrozumieć jakie znaczenie dla bezpieczeństwa systemu ma plik konfiguracyjny `/etc/login.defs`.

## Czym jest /etc/login.defs
Jest to plik konfiguracyjny dla narzędzi shadow-utils, czyli `useradd`, `usermod`, `userdel`, `passwd`, `login`. Jednak niektóre moduły PAM korzystają z tego pliku:
- `pam_unix.so` - określa algorytm hashowania określony w parametrze `ENCRYPT_METHOD` oraz złożoność hashowania z parametru `SHA_CRYPT_MAX_ROUNDS`,
- `pam_faildelay.so` - odczytuje wartość parametru `FAIL_DELAY`, który określa opóźnienie po nieudanej próbie logowania, 
- `pam_umask.so` - z parametru `UMASK` odczytuje domyślne uprawnienia dla nowych plików. 
Należy pamiętać, że `login.defs` to plik konfiguracyjny i nie należy mylić go z `login`, który jest programem i swoją konfigurację ma w PAM.

## Kluczowe parametry dla SOC
- Polityka haseł:
	- `PASS_MAX_DAYS` - określa określa częstotliwość zmiany hasła, 
	- `PASS_MIN_DAYS` - to minimalny odstęp między zmianami hasła,
	- `PASS_WARN_AGE` - określa ile dni przed wygaśnięciem system ostrzega użytkownika o wygaśnięciu hasła.
	Należy przy tym odróżnić zadanie pliku `/etc/login.defs` od działania pliku konfiguracyjnego dla modułu `pam_pwquality.so`. Plik `/etc/login.defs` to polityka starzenia się hasła, natomiast plik `/etc/security/pwquality.conf` to polityka jakości hasła. 
- Konta użytkowników:
	- `UID_MIN` / `UID_MAX` - zakres UID dla zwykłych użytkowników, 
	- `SYS_UID_MIN` / `SYS_UID_MAX` -  zakres dla kont systemowych, 
- Ograniczenie `su`: 
	- `SU_WHEEL_ONLY` - ustawiony na 'yes' oznacza, że tylko grupa wheel może używać `su`. UWAGA: w niektórych dystrybucjach ograniczenie `su` do grupy wheel jest realizowane przez moduł PAM `pam_wheel.so` w pliku `/etc/pam.d/su`.
- Uprawnienia:
	- `UMASK` - domyślne uprawnienia dla nowo tworzonych plików. 
- Katalog domowy:
	- `CREATE_HOME` - dotyczy polecenia `useradd`, jeśli ustawione jest na 'yes' polecenie domyślnie tworzy katalog domowy podczas dodawania użytkownika. 
- Hashowanie haseł:
	- `ENCRYPT_METHOD` - określa algorytm hashowania (SHA512, YESCRYPT, itp),
	- `SHA_CRYPT_MAX_ROUNDS` - określa złożoność hashowania, niska wartość oznacza szybsze, ale także słabsze hashowanie. 
- Opóźnienie przy błędzie:
	- `FAIL_DELAY` - to opóźnienie w sekundach po wpisaniu błędnego hasła.   

## Zagrożenia 
Atakujący mający dostęp do root, ma tutaj kilka możliwości, może:
- wydłużyć czas w `PASS_MAX_DAYS`, aby jego konto (lub konto roota) nie wymagało zmiany hasła, 
- zmienić `UID_MIN` na wartość z zakresu systemowego, aby jego konto wyglądało jak usługa i nie budziło podejrzeń na pierwszy rzut oka, 
- wyłączyć `SU_WHEEL_ONLY` oraz dodać swoje konto do grupy wheel, żeby móc używać `su`, 
- zmienić `ENCRYPT_METHOD` na słaby algorytm hashowania (np. DES), by móc łatwo łamać hasła, 
- ustawić `FAIL_DELAY=0`, by przyspieszyć ataki brute force. 

## Wnioski bezpieczeństwa
Ważny jest monitoring kluczowych parametrów pliku `/etc/login.defs` pod kątem nieuprawnionych zmian. W tym celu przydatne są polecenia:
- `grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs` - sprawdzenie polityki starzenia się haseł, 
- `grep -E "UID_MIN|UID_MAX|SYS_UID_MIN|SYS_UID_MAX" /etc/login.defs` - sprawdzenie zakresów UID,
- `grep -E SU_WHEEL_ONLY /etc/login.defs` - sprawdzenie ograniczenia `su`,
- `grep -E "UMASK|ENCRYPT_METHOD|FAIL_DELAY" /etc/login.defs` - sprawdzenie uprawnień, algorytmu hashowania i opóźnienia, 
- `sudo find /etc/login.defs -type f -mtime -1 -ls` - sprawdzenie, czy plik nie został zmodyfikowany w ostatnich dniach. 
- zmiany w `login.defs` nie działają na już istniejące konta, czyli np. UID_MIN nie wpływa na już istniejących użytkowników. 