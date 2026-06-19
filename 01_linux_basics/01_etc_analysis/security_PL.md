# /etc/security/

## Czym jest /etc/security
Jest to katalog zawierający pliki konfiguracyjne modułów PAM. Pliki te określają polityki bezpieczeństwa, takie jak jakość hasła, historia hasła, kontrola dostępu, blokady konta, ograniczenia zasobów, określenie środowiska i inne. 

## Kluczowe Pliki i ich lokalizacja w repo
Pliki konfiguracyjne dla modułów PAM zostały omówione szczegółowo w innej części tego repozytorium:
- `faillock.conf` zawierający konfigurację modułu `pam_faillock.so`, odpowiadającego za blokadę konta po nieudanych próbach logowania, znajduje się w `05_pam_basics/security/faillock_PL.md`,
- `pwquality.conf` zawierający konfigurację modułu `pam_pwquality.so`, odpowiadającego za politykę jakości hasła, znajduje się w `05_pam_basics/security/pwquality_PL.md`,
- `limits.conf` zawierający konfigurację modułu `pam_limits.so`, odpowiadającego za ograniczenie zasobów dla sesji, znajduje się w `05_pam_basics/security/limits_PL.md`,
- `access.conf` zawierający konfigurację modułu `pam_access.so`, odpowiadającego za kontrolę dostępu na podstawie użytkownika i źródła, znajduje się w  `05_pam_basics/security/access_PL.md`,
- `time.conf` zawierający konfigurację modułu `pam_time.so`, odpowiadającego za kontrolę dostępu na podstawie czasu, znajduje się w `05_pam_basics/security/time_PL.md`,
- `pwhistory.conf` zawierający konfigurację modułu `pam_pwhistory.so`, odpowiadającego za historię haseł, znajduje się w `05_pam_basics/security/pwhistory_PL.md`,
- `pam_env.conf` zawierający konfigurację modułu `pam_env.so`, odpowiadającego za zmienne środowiskowe, znajduje się w `05_pam_basics/security/pam_env_PL.md`.

## Wnioski bezpieczeństwa
- sam katalog `/etc/security` nie stanowi zagrożenia, jednak pliki w nim zawarte definiują polityki PAM, 
- ataki na PAM nie są codziennością, ale zdarzają się, szczególnie w środowiskach o słabej konfiguracji, np. pozostawienie domyślnych ustawień lub brak monitorowania, 
- większość ataków wymaga dostępu do konta (SSH, konsola),
- monitorowanie zmian w plikach `/etc/security` oraz w plikach PAM jest kluczowe, szczególnie w kontekście detekcji i audytu, 
- szczegółowe zagrożenia dotyczące plików w katalogu `/etc/security` są opisane w katalogu `05_pam_basics/security`.