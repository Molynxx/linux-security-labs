# 04_sudo_and_privilege_escalation_basics

## Cel
Zrozumienie, jak działa `sudo`, jakie są najważniejsze pliki konfiguracyjne (`/etc/sudoers`, `/etc/sudoers.d/`) oraz jakie zagrożenia wynikają z nieprawidłowej konfiguracji. 

## Zakres
- `sudoers_analysis` - analiza pliku `/etc/sudoers`, składnia, grupy (`%sudo`, `%wheel`), uprawnienia dla użytkowników,
- `visudo_usage` - dlaczego `visudo` jest bezpieczniejszy od zwykłego edytora, jak używać (`visudo`, `visudo -f`), co robić przy błędzie składni,
- `sudo_security_risks` - główne ryzyka: `NOPASSWD`, szerokie uprawnienia (`ALL`), `sudo` na edytorach/pagerach (`vim`, `less`, `more`), sudo na narzędziach (`find`, `tar`, `awk`, `git`), `sudo -s`, `sudo -i`, środowisko (`env_reset`, `secure_path`).

## Dlaczego to ważne (SOC/IR)
- nieprawidłowa konfiguracja `sudo` to częsty wektor eskalacji uprawnień, 
- `NOPASSWD` - jeśli atakujący uzyska dostęp do sesji, może wykonywać polecenia jako `root` bez hasła,
- `sudo` na edytorach / narzędziach - umożliwia ucieczkę do powłoki roota,
- monitoring - należy sprawdzać regularnie `sudo -l`, `grep` po `/etc/sudoers`, logi `/var/log/auth.log`.