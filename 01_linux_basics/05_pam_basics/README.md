# 05_pam_basics

## Cel
Ten katalog zawiera analizę PAM (Pluggable Authentication Modules)  w systemach Linux, ze szczególnym naciskiem na logikę uwierzytelniania, kontrolę dostępu oraz aspekty bezpieczeństwa istotne z perspektywy SOC/Blue Team.
Jest to część większego repozytorium 01_linux_basics, obejmującego podstawy działania systemu Linux w kontekście bezpieczeństwa. 

## Zakres
W tym module skupiam się na:
- strukturze i działaniu PAM,
- analizie plików konfiguracyjnych (`/etc/pam.d`),
- typach PAM (`auth`, `account`, `session`, `password`),
- flagach kontrolnych (standardowych i rozszerzonych),
- modułach PAM i ich wpływie na bezpieczeństwo, 
- politykach bezpieczeństwa (`/etc/security`).

## Struktura katalogu 

- 05_pam_basics/
	|-common-auth_PL.md
	|-common-account_PL.md
	|-common-session_PL.md
	|-common-password_PL.md
	|-login_PL.md
	|-sshd_PL.md
	|-sudo_PL.md
	|-su_PL.md
	|-lightdm_PL.md
	|-gdm-password_PL.md
	|-pam_control_flags_PL.md
	|-pam_modules_overview_PL.md
	|-pam_types_overview_PL.md
	|-security/
		|-faillock_PL.md
		|-pwquality_PL.md
		|-limits_PL.md
		|-access_PL.md
		|-time_PL.md

## Opis zawartości
- common-* - analiza współdzielonych konfiguracji PAM używanych przez różne usługi,
- login_PL.md / sshd_PL.md - konfiguracja logowania lokalnego i zdalnego,
- sudo_PL.md / su_PL.md - mechanizmy eskalacji uprawnień,
- lightdm_PL.md / gdm-password_PL.md - logowanie graficzne,
- pam_control_flags_PL.md - szczegółowa analiza flag kontrolnych i ich wpływ na przepływ PAM,
- pam_modules_overview_PL.md - przegląd modułów PAM wraz z ich rolą, 
- pam_types_overview_PL.md - wyjaśnienie typów PAM i ich kolejności,
- security/ - konfiguracja polityk bezpieczeństwa (np. blokady kont, limity, dostęp czasowy).

## Uwaga dotycząca przykładów
Niektóre przykłady wykorzystują uproszczone konfiguracje i nie zawierają pełnych opcji specyficznych dla modułów.
Celem jest przedstawienie:
- logiki działania PAM,
- wpływu flag kontrolnych,
- kolejności wykonywania modułów,
a nie tworzenie kompletnej konfiguracji produkcyjnej. 

## Znaczenie dla bezpieczeństwa
PAM jest jednym z kluczowych mechanizmów bezpieczeństwa w Linuxie. Nieprawidłowa konfiguracja może prowadzić do:
- obejścia uwierzytelnienia,
- braku rejestrowania prób logowania, 
- niepoprawnego zarządzania sesją, 
- eskalacji uprawnień.  
Zrozumienie działania PAM jest istotne w pracy SOC, szczególnie przy:
- analizie logów,
- wykrywaniu brute-force,
- badaniu nieautoryzowanego dostępu. 

## Powiązania z innymi modułami
Ten katalog łączy się bezpośrednio z:
- 02_users_and_groups/ - tożsamość użytkowników, 
- 03_permissions_and_ownership/ - kontrola dostępu, 
- 04_sudo_and_privilege_escalation_basics/ - eskalacja uprawnień,
- 07_logging_basics/ - analiza logów uwierzytelniania.

## Status
Repozytorium jest rozwijane etapami - część plików może być rozszerzona o dodatkowe scenariusze i przypadki bezpieczeństwa. 

