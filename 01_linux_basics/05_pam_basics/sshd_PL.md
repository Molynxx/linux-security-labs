# /etc/pam.d/sshd - PAM dla usługi SSH

## Cel
Zrozumienie polityki PAM dla usługi SSH.

## Czym jest /etc/pam.d/sshd
Gdy UsePAM `yes` SSH deleguje część decyzji uwierzytelniania i kontroli dostępu do PAM. Parametry SSH nadal obowiązują (np. PermitRootLogin, PasswordAuthentication), ale PAM może dodatkowo zablokować dostęp, nawet jeśli SSH by go dopuścił. SSH nie decyduje sam – przekazuje żądania do PAM, a PAM decyduje, czy użytkownik może się zalogować i na jakich warunkach.
Plik `/etc/pam.d/sshd` nie jest plikiem konfiguracyjnym SSH, tylko reguluje politykę PAM dla tej usługi.

## Jakie zastosowanie ma plik /etc/pam.d/sshd
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
Plik zawiera zarówno linie zakomentowane (#), jak i aktywne. Należy skupić się na liniach niezakomentowanych.
W pliku zwykle znajdziesz:
`[typ] [flaga] [moduł] [opcje]`   
`@include [nazwa pliku common]`  

## Jak działają zapisy w pliku PAM dla aplikacji (przykład SSH)
Przykład: `auth sufficient pam_ssh_key.so`
- poprawny klucz SSH podczas logowania → sukces, PAM kończy fazę auth, kolejne moduły auth mogą zostać pominięte. Uwaga: Użycie `pam_ssh_key.so` z flagą `sufficient` jest dopuszczalne tylko wtedy, gdy moduły ochronne (`pam_faillock.so`, `pam_faildelay.so`) znajdują się przed nim. W przeciwnym razie logowanie kluczem może omijać mechanizmy ochronne.
- niepoprawny klucz SSH → sprawdzane są kolejne moduły uwierzytelniania (np. hasło).

## Kolejność modułów w typach PAM
- `Auth` – moduły związane z uwierzytelnianiem użytkownika i ochroną przed brute force:
	- Ochrona / kontrola prób: `pam_faillock.so`, `pam_tally2.so`, `pam_faildelay.so` (zawsze `required`, nie może być `sufficient`).
	- Weryfikacja tożsamości: `pam_unix.so`, `pam_sss.so`, `pam_ldap.so` (sprawdzenie hasła/klucza).
- `Account` – sprawdzanie statusu systemu i konta:
	- Blokady systemowe: `pam_nologin.so`, `pam_time.so`,
	- Status konta: `pam_unix.so`, `pam_sss.so`, `pam_access.so` (`requisite` dla kluczowych sprawdzeń),
- `Password` – zmiana haseł:
	- Jakość hasła: `pam_pwquality.so`, `pam_cracklib.so`,
	- Historia haseł: `pam_pwhistory.so`,
	- Faktyczna zmiana hasła: `pam_unix.so`,
- `Session` – działania po zalogowaniu:
	- Limity i bezpieczeństwo: `pam_limits.so`,
	- Środowisko: `pam_env.so`, `pam_systemd.so`,
	- Logowanie sesji i dodatki: `pam_lastlog.so`, `pam_motd.so`.

💡 Uwagi:
Każdy typ powinien mieć co najmniej jeden moduł z flagą `required`.
- `Requisite` przerywa tylko fazę danego typu, a nie cały proces logowania.
- `Sufficient` może zakończyć fazę wcześniej, co w SOC/IR może być problemem, jeśli moduły ochronne zostaną pominięte.
- Kolejność modułów w ramach tego samego typu jest krytyczna – np. moduły liczące próby logowania powinny być przed modułami uwierzytelniania.
- Kolejność typów (auth → account → session → password) jest standardowa i SSH i tak je wywołuje w tej kolejności.

## Logowanie SSH krok po kroku
- Klient SSH próbuje się zalogować,
- sshd sprawdza UsePAM,
- Jeśli UsePAM yes:
	- sshd woła `/etc/pam.d/sshd`,
- PAM wykonuje:
	- auth – czy hasło/klucz poprawny
	- account – czy konto nie jest zablokowane
	- session – otwarcie sesji
- Jeśli którykolwiek etap zwróci fail → brak logowania

## Czerwone flagi dla SOC/IR
- usunięcie @include common-auth lub innych plików common,
- zmiana kontroli z requisite na `sufficient` w modułach krytycznych,
- brak modułów ochronnych (`pam_faillock.so`),
- UsePAM `no` w pliku `sshd_config`,
- nieautoryzowane modyfikacje pliku `/etc/pam.d/sshd` lub plików common-* .
💡 Dodatkowo: zmiana flag modułów w plikach common-* również jest czerwoną flagą, ponieważ SSH korzysta z nich poprzez @include.

## Wnioski
- Należy regularnie sprawdzać konfigurację PAM w `/etc/pam.d/` i monitorować metadane plików (mtime/ctime) – każda zmiana może wpływać na logowania i bezpieczeństwo kont.
- Kolejność flag kontrolnych i modułów ma znaczenie dla odporności na brute force i ataki typu DoS.
- Monitoring logów (`/var/log/auth.log`) jest krytyczny dla SOC/IR – każda próba uwierzytelnienia powinna być rejestrowana.
- Każda zmiana w plikach PAM powinna być zatwierdzona i wersjonowana (np. Git), ponieważ nieautoryzowane zmiany mogą umożliwić obejście polityki bezpieczeństwa lub wprowadzić backdoor.

##  Dodatek - tabelka moduły PAM dla SSH w logicznej kolejności, z typem, flagą i krótkim opisem:

<table>
  <tr>
    <th>Typ</th>
    <th>Moduł</th>
    <th>Flaga kontrolna</th>
    <th>Krótkie wyjaśnienie</th>
  </tr>
  <tr>
    <td>auth</td>
    <td>pam_faillock.so</td>
    <td>required</td>
    <td>
      Liczy nieudane próby logowania.<br>
      Blokuje przy zbyt wielu nieudanych próbach.<br>
      Ochrona brute-force.
    </td>
  </tr>
    <tr>
    <td>auth</td>
    <td>pam_faildelay.so</td>
    <td>required</td>
    <td>
      Wprowadza opóźnienie po nieudanym logowaniu,<br>
      utrudnia ataki brute-force.
    </td>
  </tr>
  <tr>
    <td>auth</td>
    <td>pam_unix.so</td>
    <td>required</td>
    <td>Sprawdza hasło użytkownika w systemie UNIX.</td>
  </tr>
  <tr>
    <td>auth</td>
    <td>pam_sss.so / pam_ldap.so</td>
    <td>required</td>
    <td>Sprawdza hasło użytkownika w systemach SSSD / LDAP.</td>
  </tr>
  <tr>
    <td>auth</td>
    <td>pam_ssh_key.so (opcjonalny, nie wszędzie dostępny)</td>
    <td>sufficient</td>
    <td>
      Sprawdza poprawność klucza SSH.<br>
      Jeśli sukces i brak wcześniejszych błędów,<br>
      faza kończy się natychmiast.
    </td>
  </tr>
  <tr>
    <td>account</td>
    <td>pam_nologin.so</td>
    <td>requisite</td>
    <td>
      Sprawdza czy plik /etc/nologin istnieje.<br>
      Blokuje logowanie dla wszystkich oprócz root.
    </td>
  </tr>
  <tr>
    <td>account</td>
    <td>pam_time.so</td>
    <td>requisite</td>
    <td>Sprawdza ograniczenia czasowe logowania.</td>
  </tr>
  <tr>
    <td>account</td>
    <td>pam_unix.so</td>
    <td>required</td>
    <td>Sprawdza status konta (blokada, wygaśnięcie).</td>
  </tr>
  <tr>
    <td>account</td>
    <td>pam_sss.so / pam_access.so</td>
    <td>requisite</td>
    <td>Sprawdza czy użytkownik może się zalogować zgodnie z polityką systemową.</td>
  </tr>
  <tr>
    <td>password</td>
    <td>pam_pwquality.so</td>
    <td>required</td>
    <td>Sprawdza jakość nowego hasła (długość, złożoność).</td>
  </tr>
  <tr>
    <td>password</td>
    <td>pam_cracklib.so</td>
    <td>required</td>
    <td>Alternatywne sprawdzenie jakości hasła.</td>
  </tr>
  <tr>
    <td>password</td>
    <td>pam_pwhistory.so</td>
    <td>required</td>
    <td>Sprawdza, czy hasło nie było używane wcześniej.</td>
  </tr>
  <tr>
    <td>password</td>
    <td>pam_unix.so</td>
    <td>required</td>
    <td>Zapisuje nowe hasło do systemu.</td>
  </tr>
  <tr>
    <td>session</td>
    <td>pam_limits.so</td>
    <td>required</td>
    <td>Nakłada limity zasobów na sesję użytkownika.</td>
  </tr>
  <tr>
    <td>session</td>
    <td>pam_env.so</td>
    <td>required</td>
    <td>Ustawia zmienne środowiskowe.</td>
  </tr>
  <tr>
    <td>session</td>
    <td>pam_systemd.so</td>
    <td>required</td>
    <td>Rejestruje sesję w systemd.</td>
  </tr>
  <tr>
    <td>session</td>
    <td>pam_lastlog.so</td>
    <td>optional</td>
    <td>Rejestruje ostatnie logowanie użytkownika.</td>
  </tr>
  <tr>
    <td>session</td>
    <td>pam_motd.so</td>
    <td>optional</td>
    <td>Wyświetla komunikaty dnia po zalogowaniu.</td>
  </tr>
</table>

💡 Uwagi:
- W każdym typie przynajmniej jeden moduł powinien być required.
- Moduły z flagą requisite mogą przerwać tylko bieżącą fazę, ale nie cały proces logowania.
- Sufficient może zakończyć fazę szybciej, jeśli wcześniej nie było błędów.
- Optional wpływa tylko, jeśli jest jedynym modułem w fazie, w przeciwnym razie wynik jest ignorowany.