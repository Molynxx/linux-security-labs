# /etc/security/pwhistory.conf

## Cel
Zrozumienie pliku konfiguracyjnego modułu `pam_pwhistory.so`. 

## Czym jest pwhistory.conf
To plik konfiguracyjny modułu `pam_pwhistory.so`, znajdujący się w `/etc/security/pwhistory.conf`. Określa czy i jakie hasła są zapamiętywane i ile ich przechowywać. Moduł `pam_pwhistory.so` zapamiętuje poprzednie hasła i blokuje możliwość ustawienia haseł, które znajdują się w historii haseł. Plik `pwhistory.conf` mówi modułowi jak ma to robić.

## Parametry pwhistory.conf
- `remember` - wartość tego parametru określa ile ostatnich haseł zapamiętać, 
- `enforce_for_root` - jeśli istnieje - historia haseł dotyczy też roota,
- określona jest też tu lokalizacja pliku z historycznymi hasłami zapisanymi w postaci hashy. Zwykle hasła są zapisywane w pliku `/etc/security/opasswd`. 

## Zagrożenia
Atakujący ma tutaj kilka opcji, może:
- zmniejszyć parametr `remember` na 1 lub 2 - dzięki czemu ochrona jest słaba, hasła mogą się często powtarzać, 
- wyczyścić plik `/etc/security/opasswd` - co pozwala na natychmiastowy powrót do starego hasła, 
- usunąć moduł z konfiguracji `/etc/pam.d/` co sprawi, że zabezpieczenie przed powtarzaniem hasła nie będzie aktywne. 

## Wnioski bezpieczeństwa
Istotne jest by sprawdzać:
- czy moduł `pam_pwhistory.so` jest włączony dla typu passwd za pomocą polecenia `grep -r "pam_pwhistory.so" /etc/pam.d/`, 
- czy istnieje plik konfiguracyjny modułu `pwhistory.conf` i jakie ma ustawione parametry, za pomocą polecenia `cat /etc/security/pwhistory.conf`,
- czy plik `opasswd` nie jest pusty lub zmieniony, za pomocą polecenia `sudo ls -la /etc/security/opasswd`. 