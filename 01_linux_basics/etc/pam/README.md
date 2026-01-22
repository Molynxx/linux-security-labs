# Lab: /etc/pam - PAM (Pluggable Authentication Modules) - analiza kontroli dostępu w systemach Linux.

## Cel
Celem laboratorium jest poznanie struktury plików odpowiedzialnych za kontrolę dostępu w systemach Linux oraz zrozumienie, w jaki sposób PAM wpływa na bezpieczeństwo systemu i polityki haseł.

## Czym jest PAM
PAM (Pluggable Authentication Modules) jest warstwą pośredniczącą między aplikacją (np. sshd, sudo, login) a mechanizmem uwierzytelniania. 
Aplikacje zamiast same weryfikować hasła, przekazują żądanie do PAM, który decyduje:
- czy użytkownik może się zalogować, 
- w jakich warunkach i przy użyciu jakich metod uwierzytelniania.

PAM odpowiada m.in. za:
- uwierzytelnianie użytkownika (hasła, klucze, blokady kont),
- liczbę prób logowania, 
- blokowanie kont po zbyt wielu nieudanych próbach, 
- wymuszanie polityki haseł (długość, złożoność),
- weryfikację statusu konta (blokada, wygaśnięcie),
- ograniczenia czasowe i zasobowe.

## Fazy PAM
- auth - kim jesteś? - uwierzytelnianie (hasło, klucz, 2FA, liczba prób),
- account  - czy wolno Ci się zalogować? - konto zablokowane, wygasłe, nologin, ograniczenia czasowe,
- password - czy hasło spełnia politykę? - długość, złożoność, historia, zmiana hasła, 
- session - co zrobić po zalogowaniu? - tworzenie sesji, limity, środowisko, logowanie zdarzeń. 

## Flagi kontrolne
- required - moduł musi zakończyć się sukcesem, aby uwierzytelnienie było możliwe. PAM wykonuje jednak wszystkie kolejne moduły w danej fazie, nawet jeśli ten zakończy się niepowodzeniem. Błędy są raportowane dopiero na końcu. Wszystkie kroki i informacje na temat logowania trafiają do logów, więc można sprawdzić każdy krok,
- requisite - moduł musi zakończyć się sukcesem. W przypadku błędu PAM natychmiast, przerywa dalsze przetwarzanie danej fazy i odrzuca logowanie. Informacje o niepowodzeniu są rejestrowane w logach systemowych,
- sufficient - jeśli moduł zakończy się sukcesem i nie wystąpił wcześniejszy błąd modułu required, PAM może zakończyć fazę sukcesem natychmiast - kolejne moduły w tej fazie mogą zostać pominięte,
- optional - wynik modułu nie ma wpływu na końcową decyzję, o ile nie jest jedynym modułem w danej fazie. Czyli sprawdza, ale wynik nie ma znaczenia.

## Moduły PAM 
- Moduły uwierzytelniania np. pam_unix.so, pam_sss.so, pam_ldap.so - odpowiadają za weryfikację tożsamości użytkownika,
- Moduły kontroli dostępu i kont np pam_nologin.so, pam_access.so, pam_time.so - decydują, czy użytkownik *może* się zalogować,
- Moduły polityki haseł np. pam_pwquality.so, pam_cracklib.so, pam_pwhistory.so - wymuszają zasady dotyczące haseł,
- Moduły sesji np pam_limits.so, pam_systemd.so, pam_env.so - wykonują działania po zalogowaniu użytkownika. 
Niektóre dystrybucje jak np. Kali Linux nie posiadają wszystkich modułów, w takim wypadku należy je zainstalować za pomocą polecenia sudo apt install, np. w przypadku gdy Linux nie posiada modułu pam_pwquality.so należy użyć polecenia: 'sudo apt install libpam-pwquality'.

## Gdzie znajduje się konfiguracja PAM
Cała konfiguracja PAM znajduje się w katalogu '/etc/pam.d/'. Znajdują się tam m.in. pliki:
- 'sshd' - logowania SSH,
- 'login' - logowania lokalne (TTY),
- 'sudo' - eskalacja uprawnień,
- 'password' - zmiana haseł,
- pliki 'common-* ' - ułatwiają konfigurację PAM dla wielu aplikacji, pozwalając na stosowanie wspólnych zasad bez konieczności powielania ustawień w każdym pliku aplikacji.

Wśród plików common wyróżniamy:
- 'common-auth' - reguły fazy auth,
- 'common-account' - reguły fazy account,
- 'common-password' - reguły fazy password,
- 'common-session' - reguły fazy session.
Aplikacja może korzystać z tych ustawień poprzez dyrektywę '@include', np.: '@include common-auth'.

## Dlaczego PAM jest istotna dla SOC/IR 
- PAM ma pierwszeństwo przed ustawieniami aplikacji
Przykład: SSH może mieć 'MaxAuthTries=6', a PAM 'retry=3' - w takim przypadku zadziała reguła z PAM, pod warunkiem, że 'UdePam yes' SSH.
- Właściwie skonfigurowane PAM zwiększa bezpieczeństwo systemu, ponieważ kontroluje uwierzytelnianie i blokady w sposób spójny niezależnie od aplikacji. 
- Nieprawidłowa konfiguracja PAM lub wyłączenie go w aplikacji ('UsePam') może skutkować lukami w bezpieczeństwie i nieautoryzowanym dostępem. 
- Kolejność plików oraz poprawność ich flag również ma duże znaczenie dla bezpieczeństwa systemu, ponieważ:
	- każda faza PAM ma swoje zadanie, 
	- właściwe ustawienie flag pozwala na zmniejszenie ryzyka brute force i DoS,
	- required zamienione na sufficient oraz flagi w niewłaściwej kolejności może świadczyć o potencjalnym backdoorze.
- PAM zapewnia centralny punkt kontroli uwierzytelniania, więc monitoring i audytowanie jego plików pozwala SOC/IR wykrywać potencjalne manipulacje (backdoory, zmiany flag, nieautoryzowane modyfikacje).

## Wnioski 
- Należy regularnie weryfikować konfigurację PAM w '/etc/pam.d/',
- Należy sprawdzić, czy PAM nie został wyłączony dla aplikacji, które wymagają kontroli dostępu,
- Dokumentowanie zmian w plikach PAM jest istotne z perspektywy SOC/IR, ponieważ każda modyfikacja może wpływać na logowania, polityki haseł i bezpieczeństwo kont, np. intruz mógłby zmienić słowo required na sufficiente by stworzyć backdoor,
- Każda zmiana w plikach PAM powinna być zatwierdzona i wersjonowana (np. w Git), ponieważ nieautoryzowane zmiany mogą umożliwić obejście polityki bezpieczeństwa.