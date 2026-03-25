# /etc/pam.d/common-session

## Cel 
Celem tego laboratorium jest zrozumienie co robi i jak działa plik common-session.

## Czym jest /etc/pam.d/common-session
To jest faza sesji. Od typu session zależy wszystko co się dzieje po potwierdzeniach od auth i account. Zadania związane z typem session:
- tworzenie i czyszczenie sesji użytkownika,
- ustawianie zmiennych środowiskowych, 
- ograniczenia zasobów,
- logowanie aktywności. 

## Moduły typu session i ich kolejność 
W tym module występują zupełnie inne moduły (nie licząc `pam_unix.so`) niż w typach auth i account. Wśród modułów typu session wyróżnić można:
- `pam_limits.so` -  nakłada limity na zasoby i procesy sesji użytkownika, 
- `pam_env.so` - inicjalizacja bezpiecznego środowiska użytkownika,
- `pam_umask.so` - instrukcja jakie uprawnienia mają być zabrane nowo tworzonym plikom i katalogom.
- `pam_unix.so` -  otworzenie, utrzymanie i zamknięcie sesji użytkownika,  
- `pam_systemd.so` - rejestruje sesję użytkownika w systemd - logind,
- `pam_loginuid.so` - przypisuje UID (AUID) użytkownika do wszystkich procesów w sesji. 
- `pam_exec.so` - pozwala uruchomić zewnętrzny program lub skrypt na określonym etapie PAM. 
- `pam_motd.so` - wyświetla Message Of The Day użytkownikowi po otwarciu sesji.
- `pam_mail.so` - sprawdza czy istnieje plik poczty w `/var/mail/username` lub `/var/spool/mail/username` i czy nie jest pusty. 
- `pam_lastlog.so` - zapisuje i aktualizuje w pliku `/var/log/lastlog` informacje o ostatnim logowaniu użytkownika. 

## Flagi kontrolne modułów
Istotne dla bezpieczeństwa sesji i logowania moduły powinny występować z flagą `required`, dla mniej istotnych modułów, jak np. `pam_exec.so`, `pam_mail.so`, `pam_motd.so` często stosuje się flagę `optional`.

## Zagrożenia PAM common-session
Faza session w PAM odpowiada za szereg powiązanych zdarzeń:
- otwarcie i zamknięcie sesji użytkownika, 
- poprawne zainicjalizowanie kontekstu wykonawczego procesów, 
- rejestrację sesji, 
- zapewnienie widoczności i możliwości pełnego audytowania działań użytkownika. 
Ponieważ faza session odpowiada za szereg powiązanych zdarzeń, zagrożenia w tej fazie nie mają charakteru binarnego, a ich skutki często ujawniają się pośrednio, w czasie lub dopiero na etapie analizy incydentu.  
Przykłady: 
- utrata spójności sesji i sprzątania procesów (`pam_unix.so`, `pam_systemd.so`),
- utrata audytu i tożsamości użytkownika (`pam_unix.so`, `pam_loginuid.so`),
- manipulacja środowiskiem wykonawczym i politykami (`pam_env.so`, `pam_exec.so`, `pam_limits.so`, `pam_umask.so`).
- problemy zarządcze i obserwowalności systemu (`pam_lastlog.so`, opcjonalnie `pam_motd.so`, `pam_mail.so`), moduły te wymagają sprawdzenia praw dostępu i właścicieli plików,
- ryzyko socjotechniki / manipulacji użytkownikiem (`pam_motd.so`, `pam_mail.so`).

## Wnioski 
- kluczowe moduły odpowiedzialne za audyt, środowisko i zarządzanie sesją powinny mieć flagę `required`,
- kolejność modułów w common-session jest krytyczna: środowisko -> sesja -> systemd -> audyt -> pozostałe, 
- moduły wykonujące skrypty lub pokazujące info (`pam_exec.so`, `pam_mail.so`, `pam_motd.so`) wymagają weryfikacji ścieżek i właścicieli. 

## Przykłady analizy konfiguracji pliku common-session (Case Study)

### Case Study 1

session required pam_env.so   
session required pam_limits.so   
session optional pam_umask.so   
session required pam_unix.so   
session required pam_systemd.so    
session required pam_loginuid.so    
session optional pam_motd.so  
session optional pam_mail.so   
session optional pam_exec.so    
session optional pam_lastlog.so  

Analiza:    
- to jest przykład prawidłowego ustawienia common-session. Moduły są w odpowiedniej kolejności a ich flagi są poprawne.    

Sugerowane działanie:  
-  sprawdzenie ścieżek modułów `pam_exec.so`, `pam_motd.so`, `pam_mail.so`.  

### Case Study 2

session required pam_env.so  
session optional pam_limits.so  
session optional pam_umask.so  
session required pam_unix.so  
session required pam_systemd.so   
session optional pam_exec.so  
session optional pam_motd.so   
session optional pam_mail.so  
session optional pam_lastlog.so  

Analiza:  
W tym modelu występują dwa istotne dla bezpieczeństwa błędy:
- moduł `pam_limits.so` ma flagę `optional`, a to oznacza, że moduł może się nie wykonać, a jeśli tak się stanie, nie zostaną nałożone na sesję żadne limity. 
- brak modułu `pam_loginuid.so` utrudni audyt, ponieważ w logach procesów nie będzie informacji o AUID.  
- `pam_env.so` powinien być ustawiony przed `pam_limits.so`, żeby środowisko było poprawnie ustawione przed nakładaniem limitów. 

Sugerowane działanie:   
- zmienić flagę modułu `pam_limits.so` na `required`, dodać moduł `pam_loginuid.so` za modułem `pam_systemd.so`.  

### Case Study 3

session optional pam_env.so  
session required pam_limits.so  
session optional pam_umask.so  
session required pam_unix.so  
session required pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so  
session optional pam_lastlog.so  

Analiza:   
W tym przykładzie występuje jeden ale bardzo poważny błąd:
- `pam_env.so` powinien być ustawiony przed `pam_limits.so`, żeby środowisko było poprawnie ustawione przed nakładaniem limitów, co jest poprane, jednak ma niepoprawną flagę `optional`. 
To oznacza, że moduł ten może się nie wykonać. Zagrożenia wynikające z niewykonania się tego modułu to bardzo szeroka klasa zagrożeń manipulacji zmiennymi środowiskowymi. Brak `pam_env.so` jako `required` otwiera drogę do manipulacji PATH, TMPDIR, LOCALE, IFS, co  może prowadzić do eskalacji uprawnień i persistence.  

Sugerowane działanie:   
- zmienić flagę modułu `pam_env.so` na `required`.

