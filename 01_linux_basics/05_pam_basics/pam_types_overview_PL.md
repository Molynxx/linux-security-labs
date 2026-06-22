# Typy PAM

## Cel
Zrozumienie, jakie są typy PAM, jak działają oraz jakie mają znaczenie dla bezpieczeństwa systemu Linux. 

## Jak działają typy PAM
Każdy typ odpowiada za jedną fazę procesu logowania lub zarządzania kontem w systemie. Dzięki temu konfiguracja PAM jest uporządkowana i przejrzysta. Typy tworzą logiczny łańcuch decyzyjny, w którym informacje przetwarzane przez wcześniejsze etapy mogą być wykorzystywane w kolejnych. 

## Kolejność typów
Każdy typ odpowiada za inny etap procesu, dlatego ich kolejność wpływa bezpośrednio na bezpieczeństwo całego mechanizmu uwierzytelniania. 
- `auth` - uwierzytelnienie,
- `account` - autoryzacja, 
- `session` - zarządzanie sesją,
- `password` - zmiana hasła.
Nieprawidłowa kolejność może prowadzić do nieautoryzowanego dostępu lub nieskutecznej kontroli użytkownika (np. otwarcie sesji przed uwierzytelnieniem).

## Zadania poszczególnych typów
- `auth` - uwierzytelnienie użytkownika:
	- sprawdzenie loginu i hasła,
	- zliczanie nieudanych prób logowania, 
	- opóźnianie kolejnych prób,
	- decyzja o powodzeniu lub niepowodzeniu logowania.
	Uwaga SOC: 
	Typ `auth` generuje logi prób logowania (udanych i nieudanych), które są kluczowe przy wykrywaniu brute force. 
- `account` - autoryzacja dostępu:
	- sprawdzenie, czy konto nie jest zablokowane lub wygasłe,
	- blokada logowania w trybie maintenance (`/etc/nologin`),
	- blokada konta po przekroczeniu limitu prób (na podstawie danych z `auth`),
	- decyzja o przyznaniu lub odmowie dostępu.
	Uwaga bezpieczeństwa:
	Typ `account` korzysta z informacji z `auth` (np. liczba prób logowania). Brak tego typu może umożliwić logowanie mimo zablokowanego konta.
- `session` - zarządzanie sesją użytkownika:
	- tworzenie i zamykanie sesji,
	- ustawianie zmiennych środowiskowych, 
	- nakładanie limitów systemowych, 
	- rejestrowanie rozpoczęcia i zakończenia sesji.
	Uwaga bezpieczeństwa:
	Typ `session` odpowiada za środowisko użytkownika i logi sesji. Błędna konfiguracja może prowadzić do nieprawidłowego działania środowiska lub utraty widoczności aktywności użytkownika. 
- `password` - zmiana hasła użytkownika:
	- ustawienie nowego hasła, 
	- zapis hasła w postaci zahashowanej, 
	- egzekwowanie polityk haseł (siła, rotacja, historia).
	Uwaga:
	Typ `password` nie jest częścią procesu logowania. Podczas zmiany hasła:
	- wymagany jest typ `auth` (weryfikacja starego hasła),
	- czasami używany jest również `account` (sprawdzanie statusu konta).
	Moduły w tym typie mogą korzystać z opcji `use_authtok`, która pozwala wykorzystać hasło z poprzednich etapów. 

## Przepływ informacji między typami
Podczas logowania użytkownika typy działają w kolejności:
`auth -> account -> session`
- `auth` weryfikuje tożsamość użytkownika i zbiera informacje o próbach logowania, 
- `account` wykorzystuje te informacje do podjęcia decyzji o dostępie (np. blokada konta),
- `session` tworzy środowisko pracy użytkownika po pomyślnym zalogowaniu. 
- `password` nie jest częścią procesu logowania, lecz operacji zmiany hasła. Np. podczas logowania przez SSH użytkownik przechodzi przez typy auth -> account -> session, gdzie błędne hasło zostanie odrzucone na etapie auth, a zablokowane konto na etapie account. 
Uwaga:  
typ `password` działa niezależnie od procesu logowania i jest używany wyłącznie podczas operacji zmiany hasła. 

## Typy a moduły PAM
Typy definiują fazę działania, natomiast moduły realizują konkretne zadania.
- typ = etap procesu (`auth`, `account`, `session`, `password`),
- moduł = funkcja (`pam_unix.so`, `pam_faillock.so`).
Ten sam moduł może działać inaczej w różnych typach, np.:
- `pam_faillock.so` w `auth` zlicza nieudane próby logowania,
- a w `account` sprawdza, czy konto powinno zostać zablokowane. 
Kolejność modułów w obrębie typu ma kluczowe znaczenie dla bezpieczeństwa.
Uwaga:  
Ten sam moduł może mieć różne znaczenie w zależności od typu, dlatego analiza konfiguracji PAM zawsze musi uwzględniać kontekst typu.

## Typy PAM a aplikacje
Typy PAM są uniwersalne i występują w konfiguracjach różnych aplikacji (np. `sshd`, `login`, `sudo`), jednak:
- nie każdy plik musi zawierać wszystkie typy, kolejność typów ma znaczenie i powinna być zachowana tam gdzie występują,
- zestaw modułów zależy od konkretnej aplikacji.
Brak lub błędna konfiguracja typu może prowadzić do poważnych problemów bezpieczeństwa, np.:
- brak `auth` - brak weryfikacji użytkownika,
- brak `account` - brak kontroli dostępu, 
- brak `session` - brak poprawnej konfiguracji środowiska użytkownika.

## Typy PAM a logi (SOC perspective)
Typy PAM są powiązane z różnymi rodzajami zdarzeń w logach:
- `auth` - próby logowania (udane i nieudane),
- `session` - rozpoczęcie i zakończenie sesji użytkownika. 
Logowanie zależy od aplikacji i modułów PAM, a nie od samych typów.  
Uwaga:  
Typy nie logują bezpośrednio - za logowanie odpowiadają moduły i aplikacje korzystające z PAM.

## Wnioski bezpieczeństwa
Poprawna konfiguracja typów PAM:
- chroni przed brute force,
- zapobiega nieautoryzowanemu dostępowi,
- umożliwia kontrolę dostępu do kont, 
- zapewnia poprawne zarządzanie sesją,
- egzekwuje polityki haseł.
Błędna konfiguracja PAM może prowadzić do obejścia uwierzytelnienia lub eskalacji uprawnień. Typy PAM tworzą spójny mechanizm kontroli dostępu, a ich poprawna konfiguracja jest kluczowa dla bezpieczeństwa systemu. 