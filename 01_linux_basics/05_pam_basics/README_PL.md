# PAM Basic Lab

## Czym jest PAM
PAM (Pluggable Authentication Modules) to warstwa pośrednicząca między aplikacjami )np. sshd, login, sudo, su) a mechanizmami uwierzytelniania w systemach Linux. Umożliwia centralne zarządzanie polityka bezpieczeństwa, kontrolę dostępu i zasady dotyczące haseł. 

## Struktura Folderu
- `common-auth.md` - faza auth,
- `common-account.md` - faza account,
- `common-session.md` - faza sesji,
- `common-password.md` - faza password,
- `login.md`, `sshd.md`, `lightdm.md`, `gdm-password.md` - konfiguracja usług, 
- `pam_control_flags.md` - opis flag kontrolnych,
- `pam_modules_overview.md` - przegląd modułów PAM,
- ` pam_types_overview.md` - typy PAM,
- `security/.md` - dodatkowe polityki bezpieczeństwa (`faillock`, `pwquality`, `limits`, `access`, `time`).

## Znaczenie dla SOC/IR
- PAM centralizuje kontrolę dostępu i uwierzytelniania,
- błędna konfiguracja lub nieautoryzowane zmiany mogą umożliwić obejście polityki bezpieczeństwa,
- monitorowanie plików `/etc/pam.d/` i `/etc/security/` pomaga wykrywać potencjalne incydenty. 