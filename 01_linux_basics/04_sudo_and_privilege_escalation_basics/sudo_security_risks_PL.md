# sudo security risks

## Cel 
Poznanie i zrozumienie ryzyka jakie niosą ze sobą zmiany w plikach `sudoers`.

## Główne ryzyka związane z sudo 
- `NOPASSWD` - jeśli taka opcja zostanie wprowadzona w plikach `sudoers`, użytkownik nie musi podawać hasła.  
	- przykład: `jan ALL=(ALL) NOPASSWD: ALL`
- zbyt szerokie uprawnienia - użytkownik może uruchomić dowolne polecenie jako root:
	- przykład: `jan ALL=(ALL) ALL`
- `sudo` na edytorach - z poziomu edytora (`vim`, `less`, `more`) można uruchomić powłokę, choć trzeba uruchomić edytor z uprawnieniami sudo. 
- `sudo` na narzędziach do uruchamiania programów  - można wykonać dowolne polecenie przez `-exec` (np. `sudo find`, `sudo tar`, `sudo awk`, `sudo git`), 
- `sudo -s` / `sudo -i` - uruchomienie powłoki jako roota - omija ograniczenia poleceń (jeśli użytkownik ma `ALL`),
- nieograniczone środowisko - Niektóre programy używają zmiennych środowiskowych, które mogą być spreparowane. 

## Dlaczego NOPASSWD jest niebezpieczne
Opcja ta powoduje, że użytkownik nie musi podawać hasła. W przypadku uzyskania dostępu do sesji, atakujący może wykonywać polecenia jako root. Dlatego istotna jest zasada: NOPASSWD tylko dla specyficznych i bezpiecznych poleceń (np. `/bin/systemctl restart sshd`), nigdy dla `ALL`.

## sudo w edytorach 
Edytory takie jak `vim`, `less`, mają funkcję uruchamiania powłoki jeśli program został uruchomiony jako `sudo`. Wystarczy wpisać polecenie `sudo vim`, `sudo less /ścieżka/do pliku`, a po uruchomieniu edytora wpisać `!bash` i uruchomi się powłoka `root`. To bardzo niebezpieczna sytuacja, w której zwykły użytkownik w prosty sposób może eskalować swoje uprawnienia. 

## sudo na narzędziach do uruchamiania programów (find, tar, awk, git)
Te narzędzia mają wbudowane mechanizmy do wykonywania innych programów. 
- `find` - ma opcję `exec` która wykonuje dowolne polecenie dla każdego znalezionego pliku. Przykład:
	- `sudo find / -exec /bin/bash \; -quit`   
- `tar` - opcje `--checkpoint` i `--checkpoint-action` pozwalają wykonać dowolne polecenie podczas tworzenia archiwum. Przykład:
	- `sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash` 
- `awk`, `git` i inne:
	- `awk` ma funkcję `system()` - może wykonać dowolne polecenie systemowe. 
	- `git` - może uruchamiać zewnętrzne programy przez `git help` lub `git --exec-path`.

## sudo -s i sudo -i
To także istotny problem z nadawaniem uprawnień sudo. Teoretycznie można dać `sudo` użytkownikowi ale zakazać mu wykonania czegoś. Jednak w przypadku zakazu zmiany powłoki, czyli wpisania ograniczenia `/bin/bash` w `sudoers`, użytkownik nadal ma możliwości zmiany powłoki za pomocą poleceń:
- `sudo -s` - uruchamia powłokę jako root, zachowując część środowiska użytkownika, 
- `sudo -i` - symuluje pełne logowanie jako root (ładuje profile roota).
Więc jeśli użytkownik ma `ALL` w `sudoers`, może omijając ograniczenia poleceń uruchamiając jedno z powyższych poleceń. UWAGA: zamiast wpisywać całą listę zakazów w `sudoers` lepiej dla użytkownika stworzyć plik w `sudoers.d` i tam wpisać pozwolenia zamiast zakazów. Przykładowo, chcąc zakazać zmiany powłoki, należałoby uwzględnić w zakazie wszystkie sposoby, a wpis musiał by wyglądać tak:  
- `jan ALL=(ALL) ALL, !/bin/bash, !/bin/sh, !/bin/zsh, !/usr/bin/su`   
Wpis się robi długi i łatwo przeczyć coś, dając podatność do wykorzystania celem zmiany powłoki. 

## Środowisko (env_reset, secure_path)
Ustawienia domyślne programu `sudo` odgrywają równie ważną rolę w bezpieczeństwie systemu:
- `env_reset` - gdy jest ustawione, czyści zmienne środowiskowe użytkownika przed wykonaniem polecenia, dzięki czemu zapobiega atakom przez LD_PRELOAD, PATH, itp.
- `secure_path` - gdy jest ustawione, ustawia bezpieczną `PATH` dla `sudo`, co zapobiega podmianie programów. 

## Wykrywanie i monitoring 
Co należy sprawdzać:
- `kto ma NOPASSWD` - za pomocą poleceń `sudo -l` (dla bieżącego użytkownika), `sudo -l -u nazwa` (dla innych użytkowników),
- `kto ma pełne ALL` - za pomocą polecenia `sudo -l` (należy szukać wpisów `(ALL) ALL`), 
- `kto ma sudo na edytorach/narzędziach` - poleceniem `grep -E "vim|less|more|find|tar|awk|git" /etc/sudoers /etc/sudoers.d/`,
- `czy secure_path jest ustawione` - poleceniem `sudo grep secure_path /etc/sudoers`,
- `logi sudo` - `/var/log/auth.log` (wpisy z sudo).

## Wnioski bezpieczeństwa 
- Należy ostrożnie i w przemyślany sposób nadawać uprawnienia sudo, aby nie spowodować możliwości eskalacji uprawnień, 
- opcja NOPASSWD jest niebezpieczna, wykonywanie `sudo` bez hasła. Jeśli atakujący zdobędzie dostęp do sesji może uzyskać uprawnienia root,
- zamiast `ALL` i ograniczania praw, lepiej określić do jakich programów z `sudo` użytkownik powinien mieć dostęp, 
- należy dbać o zabezpieczenie środowiska, żeby unikać podmiany bibliotek i programów. 
- dobrą praktyką jest nieudzielanie użytkownikom uprawnień `sudo` do edytorów i programów mających możliwość wykonywania poleceń,
- należy zawsze używać programu `visudo` do edycji plików `sudoers` i `sudoers.d`.