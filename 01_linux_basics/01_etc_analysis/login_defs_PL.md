# /etc/login.defs

## Cel
Zrozumieć jakie znaczenie dla bezpieczeństwa systemu ma plik konfiguracyjny `/etc/login.defs`.

## Czym jest /etc/login.defs
Jest to plik konfiguracyjny dla narzędzi shadow-utils, czyli `useradd`, `usermod`, `userdel`, `passwd`, `login`. Jednak niektóre moduły PAM korzystają z tego pliku:
- `pam_unix.so` - określa algorytm hashowania określony w parametrze `ENCRYPT_METHOD` oraz złożoność hashowania z parametru `SHA_CRYPT_MAX_ROUNDS`,
- `pam_faildelay.so` - odczytuje wartość parametru `FAIL_DELAY`, który określa opóźnienie po nieudanej próbie logowania, 
- `pam_umask.so` - z parametru `UMASK` odczytuje domyślne uprawnienia dla nowych plików. 
Pozostałe parametry nie dotyczą PAM. 
Należy pamiętać, że `login.defs` to plik konfiguracyjny i nie należy mylić go z `login`, który jest programem i swoją konfigurację ma w PAM.

## Kluczowe parametry dla SOC
- Polityka haseł:
	- `PASS_MAX_DAYS` - określa po ilu dniach hasło musi zostać zmienione, duże wartości oznaczają ryzyko, że skompromitowane hasło działa przez bardzo długi okres, 
	- `PASS_MIN_DAYS` - to minimalny odstęp między zmianami hasła, który ma zapobiegać natychmiastowej zmianie hasła z powrotem na stare,
	- `PASS_WARN_AGE` - określa ile dni przed wygaśnięciem system ostrzega, dzięki temu użytkownicy są poinformowani o konieczności zmiany hasła.
	Należy przy tym odróżnić zadanie pliku `/etc/login.defs` od działania pliku konfiguracyjnego dla modułu `pam_pwquality.so`. Plik `/etc/login.defs` to polityka starzenia się hasła, czyli kiedy hasło wygasa i jak często można je zmieniać. Natomiast plik `/etc/security/pwquality.conf` to polityka jakości hasła, czyli jaką powinno mieć długość, jakie klasy znaków, itp. 
- Konta użytkowników:
	- `UID_MIN` / `UID_MAX` - zakres UID dla zwykłych użytkowników, w większości współczesnych dystrybucji ustawiany na 1000 i więcej, by wyraźnie oddzielić UID kont systemowych i interaktywnych, co ułatwia monitorowanie i audyt, 
	- `SYS_UID_MIN` / `SYS_UID_MAX` - analogicznie jak wyżej, to zakres dla kont systemowych, 
- Ograniczenie `su`: 
	- `SU_WHEEL_ONLY` - ustawiony na 'yes' oznacza, że tylko grupa wheel może używać `su`, to istotna warstwa zabezpieczenia przed próbami użycia `su` przez użytkowników nieuprawnionych. UWAGA: w niektórych dystrybucjach ograniczenie `su` do grupy wheel jest realizowane przez moduł PAM `pam_wheel.so` w pliku `/etc/pam.d/su`.
- Uprawnienia:
	- `UMASK` - domyślne uprawnienia dla nowo tworzonych plików. `UMASK` działa na zasadzie odejmowania uprawnień, to ważne, więc zapis 077 odejmie grupie i others wszystkie uprawnienia, nie odejmując nic właścicielowi pliku. To zapewnia poufność, bo tylko właściciel ma dostęp. (UWAGA: root ma dostęp zawsze, niezależnie od uprawnień). Zalecana wartość to 027 lub 077, aby chronić dane użytkowników przed dostępem innych, 
- Katalog domowy:
	- `CREATE_HOME` - dotyczy polecenia `useradd`, jeśli ustawione jest na 'yes' polecenie domyślnie tworzy katalog domowy podczas dodawania użytkownika. 
- Hashowanie haseł:
	- `ENCRYPT_METHOD` - określa algorytm hashowania (SHA512, YESCRYPT, itp). Parametr ten ma duże znaczenie dla bezpieczeństwa, ponieważ słaby algorytm hashowania zwiększa ryzyko złamania hasła, 
	- `SHA_CRYPT_MAX_ROUNDS` - określa złożoność hashowania, niska wartość oznacza szybsze, ale także słabsze hashowanie. Co to jest: system bierze hasło, hashuje je, a następnie bierze wynik i hashuje go ponownie i tak wiele tysięcy razy. Dzięki temu złamanie hasła przez atakującego trwa latami, nie da się złamać w kilka sekund, minut lub godzin. Dla SHA-512 domyślnie jest 5000 rund.
- Opóźnienie przy błędzie:
	- `FAIL_DELAY` - to opóźnienie w sekundach, występujące po wpisaniu nieprawidłowego hasła utrudnia ataki brute force. Z tego parametru korzysta moduł PAM `pam_faildelay.so`.  

## Zagrożenia 
Atakujący mający dostęp do root, ma tutaj kilka możliwości, może:
- wydłużyć czas w `PASS_MAX_DAYS`, aby opóźnić wymuszanie zmiany haseł dla wszystkich użytkowników, 
- zmienić `UID_MIN` na wartość z zakresu systemowego, aby jego konto wyglądało jak usługa i nie budziło podejrzeń na pierwszy rzut oka - zmiana dotyczy tylko nowo tworzonych kont, 
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
- niektóre zmiany (np. `UMASK`) w `login.defs` wymagają ponownego logowania, a inne (np. `ENCRYPT_METHOD`) są odczytywane przy tworzeniu nowego użytkownika. 