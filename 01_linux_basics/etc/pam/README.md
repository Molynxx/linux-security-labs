Lab: /etc/pam - PAM (Pluggable Authentication Modlues) - analiza kontroli dostępu w systemach Linux.

## Cel
Celem laboratorium jest poznanie struktury plików odpowiedzialnych za kontrolę dostępu w systemach Linux oraz zrozumienie, w jaki sposób PAM wpływa na bezpieczeństwo systemu i polityki haseł.

## Czym jest PAM
PAM (Pluggable Authentication Modlules) jest warstwą pośredniczącą między aplikacją (np. sshd, sudo, login) a mechanizmem uwierzytelniania. 
Aplikacje zamiast same weryfikować hasła, przekazują żądanie do PAM, który decyduj:
- czy użytkownik może się zalogować, 
- w jakich warunkach i przy użyciu jakich metod uwierzytelniania.

PAM odpowiada m.in. za:
- uwierzytelnianie użytkownika (hasła, kluczem blokady kont),
- liczbę prób logowania, 
- blokowanie kont po zbyt wielu nieudanych próbach, 
- wymuszanie polityki haseł (długość, złożoność),
- weryfikację statusu konta (blokada, wygaśnięcie),
- ograniczenia czasowe i zasobowe.

## Gdzie znajduje się konfiguracja PAM
Cała konfiguracja PAM znajduje się w katalogu '/etc/pam.d/'. Znajduą się tam m.in. pliki:
- 'sshd' - logowania SSH,
- 'login' - logowania lokalne (TTY),
- 'sudo' - eskalacja uprawnień,
- 'password' - zmiana haseł,
- pliki 'common-* ' - ułatwiają konfigurację PAM dla wielu aplikacji, pozwalając na stosowanie wspólnych zasad bez konieczności powielania ustawień w każdym pliku aplikacji.

Wśród plików common wyróżniamy:
- 'common-auth' - uwierzytelnianie (hasło, klucz, liczba prób),
- 'common-account' - weryfikacja konta (czy może się zalogować, blokada lub wygaśnięcie),
- 'common-password' - zasady zmiany hasła,
- 'common-session' - działania wykonanie po zalogowaniu.
Aplikacja może korzystać z tych ustawień przez dyrektywę '@include', np.: '@include common-auth'.

## Dlaczego PAM jest istotna dla SOC/IR 
- PAM ma pierwszeństwo przed ustawieniami aplikacji
Przykład: SSH może mieć 'MaxAuthTries=6', a PAM 'retry=3' - w takim przypadku zadziała reguła z PAM, pod warunkiem, że 'UdePAm yes' SSH.
- Właściwie skonfigurowane PAM zwiększa bezpieczeństwo systemy, ponieważ kontroluje uwierzytelnianie i blokady w sposób spójny niezależnie od aplikacji. 
- Nieprawidłowa konfiguracja PAM lub wyłączenie go w aplikacji ('UsePam') może skutkować lukami w bezpieczeństwie i nieautoryzowanym dostępem. 

## Wnioski 
- Należy regularnie weryfikować konfigurację PAM w '/etc/pam.d/',
- Należy sprawdzić, czy PAM nie został wyłączony dla aplikacji, które wymagają kontroli dostępu,
- DOkumentowanie zmian w plikach PAM jest istotne z perspektywy SOC/IR, ponieważ każda modyfikacja może wpływać na logowania, polityki haseł i bezpieczeństwo kont.