# Flagi kontrolne

## Cel
Zrozumienie czym są flagi kontrolne, jakie są ich rodzaje oraz jaki mają wpływ na funkcjonowanie polityk PAM.

## Czym są flagi kontrolne
Flagi PAM definiują logikę decyzyjną, określając czy wynik działania modułu powoduje kontynuację, przerwanie lub zakończenie procesu uwierzytelniania. Flagi kontrolne dzielą się na:
- standardowe (`required`, `requisite`, `sufficient`, `optional`),
- rozszerzone (np. `[success=1 default=ignore]`).

## Standardowe flagi kontrolne
- `required` - moduł oznaczony tą flagą musi się wykonać, jednak nie przerywa wykonania typu, decyzja o pozytywnym zakończeniu zapada na końcu typu, jeśli żaden moduł `required` nie zakończył się błędem. Jeśli którykolwiek moduł z tą flagą nie wykona się prawidłowo, cały typ zakończy się niepowodzeniem. Flaga ta pozwala na logowanie wszystkich działań przeprowadzonych w typie nie alarmując przy tym atakującego. To oznacza, że wszystkie moduły zostaną przeprowadzone i logowane zanim atakujący otrzyma odpowiedz o niepowodzeniu. 
- `requisite` - flaga ta natychmiast przerywa wykonanie typu w przypadku niepowodzenia modułu, przy którym stoi, a typ zakończy się niepowodzeniem. Jednak ma to pewne wady - wraz z przerwaniem typu, `requisite` może ograniczyć widoczność dalszych zdarzeń w logach. 
Uwaga: flaga ta na module, który nie ma znaczenia dla uwierzytelniania, bezpieczeństwa (zwłaszcza jeśli moduły są w nieprawidłowej kolejności) może uniemożliwić logowanie. 
- `sufficient` - ta flaga oznacza, że jeśli moduł zakończy się sukcesem i nie było wcześniejszych błędów `required`, PAM kończy typ natychmiast sukcesem. Czasami jest to użyteczne np. dla modułu `pam_sss.so`. To moduł uwierzytelniający i autoryzujący dla zdalnego połączenia. Jeśli po tym module występuje `pam_unix.so` (moduł logowania lokalnego), a logowanie zdalne się powiedzie, nie ma potrzeby, by się wykonywał. Flaga ta nie może pomijać istotnych dla bezpieczeństwa modułów. Przykładowo: umieszczenie tej flagi z modułem `pam_faillock.so` przed modułem `pam_unix.so`, może pozwalać na nieautoryzowany dostęp i brak uwierzytelnienia. 
Uwaga: `sufficient` działa TYLKO jeśli wcześniejsze moduły `required` NIE zawiodły - `required` FAIL +`sufficient` SUCCESS -> CAŁOŚĆ FAIL. 
- `optional` - nie wpływa na decyzję końcową typu (chyba że jest jedynym modułem lub jeśli inne moduły nie zwróciły jednoznacznego wyniku).
Uwaga: ta flaga nie może występować z żadnymi kluczowymi dla uwierzytelniania, autoryzacji lub bezpieczeństwa modułami. Zwykle stosowana jest z modułami nieistotnymi dla całego procesu PAM np. `pam_motd.so` (wyświetlenie wiadomości dnia po zalogowaniu).

## Rozszerzone flagi kontrolne
To są flagi dające znacznie większą elastyczność, ponieważ pozwalają ujmować sytuacje warunkowe if/else. Struktura zapisu takiej flagi to:
	`[typ] [wynik=akcja wynik=akcja ...] moduł`
	czyli jeśli wynik -> zrób to.
Struktura zapisu składa się z wyniku i akcji, czyli każdemu wynikowi przypisana jest akcja, jeśli nie zostanie we wpisie podany wynik, wpada on do `default`.
- podstawowe wartości wyniku (czyli co zwrócił moduł):
	- success - moduł zakończył się sukcesem,
	- ignore - wynik modułu jest ignorowany - PAM nie traktuje go jako sukces ani jako błąd, 
	- module_unknown - oznacza, że PAM napotkał moduł, którego nie zna lub nie może załadować. PAM traktuje go jako błąd - moduł się nie wykonał,
	- auth_err - błąd uwierzytelniania,
	- user_unknown - użytkownik nie istnieje, 
	- default - wszystkie inne przypadki. 
- podstawowe akcje (czyli co PAM ma z tym zrobić):
	- ignore - PAM ignoruje wynik modułu i przechodzi do następnego modułu w tym samym typie. Nie wpływa na decyzję końcową typu. 
		Przykład: 
		Moduł zwrócił user_unknown, a akcja to ignore -> PAM przechodzi dalej, nie odrzuca logowania.
	- `ok` - wynik modułu traktowany jest jako sukces. PAM kontynuuje sprawdzanie pozostałych modułów w typie. Typ zakończy się sukcesem tylko jeśli wszystkie moduły zakończą się odpowiednio (zależnie od innych flag).
		Przykład:
		Moduł sprawdzający login zwraca sukces, akcja ok -> auth traktuje jako sukces.
	- `bad` - wynik traktowany jako błąd i wpływa na końcową decyzję typu, ale przetwarzanie może być kontynuowane. 
		Przykład:
		auth_err=bad -> wynik traktowany jako błąd i wpływa na końcową decyzje typu (wykonanie może być kontynuowane).
	- `die` - kończy cały stos PAM dla tej operacji. Używane w sytuacjach krytycznych, np. moduł bezpieczeństwa wykrył coś poważnego. 
		Przykład SOC:
		Wykryto kompromitację konta -> die -> natychmiast kończymy proces logowania. 
	- `done` - typ traktuje moduł jako ostatni decydujący moduł. PAM nie sprawdza już kolejnych modułów w tym typie. 
		Przykład:
		Masz wiele modułów `auth`, ale jeden moduł 'finalny' decyduje -> done powoduje, że pozostałe są pomijane. 
	- `1, 2` (skip) - przeskakuje kolejne N modułów w tym typie. Jest liczbą - ile modułów ma być pominiętych.
		Przykład:
		[success=1] -> jeśli moduł zakończył się sukcesem, PAM przeskakuje następny moduł i sprawdza moduł znajdujący się po nim. 
Niepoprawne ustawienie rozszerzonych flag może prowadzić do luk w logowaniu, obejścia uwierzytelnienia lub braku audytów. `Default` zawsze obejmuje wszystkie nieprzewidziane wyniki, co jest kluczowe dla bezpieczeństwa systemu. 

## Wnioski bezpieczeństwa 
- dbałość o właściwie przypisanie flag jest kluczowa, jeśli flaga jest błędna, nie tylko może uniemożliwić logowanie, ale też może pozwolić na:
	- obejście uwierzytelnienia,
	- brak logowania krytycznych zdarzeń (audyt),
	- problemy z sesją lub przyznawaniem praw (`account`, `session`).
- by właściwie stosować flagi należy dobrze znać funkcje typów i modułów oraz ich kolejność, 
- należy znać dobrze wyniki i akcje rozszerzonych flag, żeby uniknąć błędów konfiguracji.   
Najbezpieczniejszą strategią konfiguracji PAM jest podejście default=deny, gdzie brak dopasowania skutkuje odmową dostępu. 
   
## Analiza przypadków (case study)

### Case study 1
```
auth required pam_unix.so  
auth requisite pam_faillock.so  
auth sufficient pam_ldap.so  
auth optional pam_permit.so  
```
Analiza:  
- analizowany typ - auth,
- obecna kolejność i sposób użycia `pam_faillock.so` nie gwarantują poprawnego zliczania nieudanych prób logowania dla wszystkich mechanizmów uwierzytelniania, ponieważ moduł nie jest użyty w pełnym schemacie (preauth / authfail / authsucc). 
- `pam_faillock.so` - powinien być  umieszczony zgodnie z rolą zgodnie z jego rolą (np. `preauth` przed uwierzytelnianiem, `authfail` po nieudanej próbie)
- `pam_permit.so` - moduł nie powinien być używany w systemach produkcyjnych w typie auth, ponieważ może prowadzić do obejścia uwierzytelnienia. 
- kolejność modułów ma wpływ na to, co zostanie policzone i zapisane w logach.   
  
Sugerowane działania:
- umieścić `pam_faillock.so` zgodnie z jego rolą (np. `preauth` przed uwierzytelnianiem, `authfail` po nieudanej próbie),
- dodać do modułu `pam_faillock.so` opcję `authfail` oraz zmienić jego flagę na `required`,
- usunąć z auth moduł `pam_permit.so` to może oznaczać potencjalny backdoor.
Po tych działaniach typ auth będzie się wykonywać w sposób poprawny i bezpieczny dla procesu uwierzytelniania. 

### Case study 2
```
auth [success=1 default=ignore] pam_unix.so  
account [success=ok user_unknown=die] pam_faillock.so  
auth [default=bad] pam_ldap.so   
session [ignore=ignore module_unknown=die] pam_limits.so   
  ```
Analiza:  
- kolejność typów jest nieprawidłowa, najpierw powinny wykonać się wszystkie moduły typu `auth` a dopiero potem przejść do typu `account`,
- kolejność modułów `pam_unix.so` i `pam_ldap.so` zależy od polityki, 
- flaga [success=1 default=ignore] w `pam_unix.so` powoduje pominięcie kolejnego modułu (`pam_ldap.so`) w przypadku udanego uwierzytelniania lokalnego, co jest zamierzonym i poprawnym zachowaniem. W przypadku niepowodzenia wynik jest ignorowany (default=ignore), co umożliwia kontynuację uwierzytelnienia przez `pam_ldap.so`. Jest to poprawne w konfiguracjach hybrydowych (local + LDAP), jednak może utrudnić analizę błędów i logów.
- moduł `pam_faillock.so` ma prawidłowo ustawioną flagę, gdy moduł sukces, typ idzie dalej. Jeśli wystąpi user_unknown=die, proces PAM zostanie przerwany natychmiast. Jednak nie ma tu decyzji co PAM ma zrobić w pozostałych przypadkach (np. w przypadku niepowodzenia modułu z powodu przekroczenia limitu nieudanych logowań), a to już ryzyko przepuszczenia użytkownika pomimo porażki modułu. Należy dodać odpowiedni wynik i akcję do flagi [default=bad] lub użyć flagi `required` - co jest zwykle stosowane typie account, 
- moduł `pam_ldap.so` ma poprawną flagę - w przypadku nieprawidłowego hasła, moduł zostanie potraktowany jako błąd i logowanie się nie powiedzie, 
- moduł `pam_limits.so` - flaga tego modułu nie zabezpiecza w pełni sesji użytkownika, ponieważ wskazuje ona tylko instrukcje co należy zrobić dla wyników ignore oraz module_unknown. To niekompletne zabezpieczenie, stwarzające ryzyko, że sesja otworzy się nawet w przypadku niepowodzenia modułu, a co za tym idzie limity nie zostaną nałożone na sesję użytkownika. To może prowadzić do zabicia systemu np. dużą liczbą procesów. Dlatego należy zmienić tą flagę, na taką, która pozwoli modułowi realnie zabezpieczyć sesję -> [success=ok default=bad],
- dla wielu modułów flagi rozszerzone nie są zasadne, ponieważ w przedstawionych powyżej modułach sens ma tylko sucess=1 ale dla poprawnej kolejności ustawienia modułu `pam_ldap.so`.  
    
Sugerowane działania:
- zmienić kolejność typów na prawidłowy auth -> account -> session,
- w przypadku modułu `pam_unix.so` zależnie od polityki:
	- albo zostawić przed `pam_ldap.so` i pozostawić obecną flagę,
	- albo umieścić go za modułem `pam_ldap.so` i zmienić flagę na [success=ok default=bad],
- moduł `pam_faillock.so` - zmienić flagę dodając moduł [default=bad],
- moduł `pam_limits.so` - zmienić flagę na [success=ok default=bad].