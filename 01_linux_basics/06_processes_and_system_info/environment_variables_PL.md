# environment_variables

## Cel
Zrozumienie, czym są zmienne środowiskowe, jak je sprawdzać, ustawiać (tymczasowo i trwale) oraz usuwać.

## Czym są zmienne środowiskowe 
To pary `NAZWA=wartość`, które wpływają na zachowanie programów i powłoki. Przykładowo: `PATH` mówi gdzie szukać programów, `HOME` wskazuje katalog domowy, `LANG` określa język, `LD_PRELOAD` pokazuje ścieżkę do biblioteki wstrzykniętej przed innymi.  Zmienne środowiskowe mogą być zmienione przez atakującego, żeby podmienić programy, wstrzyknąć biblioteki, zmienić zachowanie skryptów. 

## Jak sprawdzić zmienne 
- `env` - polecenie wyświetla wszystkie zmienne środowiskowe, dziedziczone przez procesy potomne. To szybki podgląd, 
- `set` - wszystkie zmienne oraz funkcje powłoki. Gdy jest potrzeba uzyskania większej ilości szczegółów, 
- `echo $NAZWA` -  wyświetla wartość konkretnej zmiennej.

## Ustawianie zmiennych środowiskowych (tymczasowo)
W bieżącej sesji oraz dla jej procesów potomnych wystarczy wpisać polecenie:
- `export NAZWA=wartość`  
Przykład: `export LD_PRELOAD=/home/user/biblioteka.so`   
UWAGA: zmienna obowiązuje do końca sesji lub do zmiany tej zmiennej, to nie jest zmiana trwała.

## Ustawianie zmiennych środowiskowych (trwale)
Zmienne można trwale zmienić w kilki miejscach, edytując pliki:
- `~/.bashrc` - zmiana tylko dla danego użytkownika (sesja powłoki),
- `~/.profile` - tylko dla danego użytkownika (sesje logowania), 
- `/etc/environment` - zmiana globalna, dotyczy zmiennych dla wszystkich użytkowników,
- `/etc/security/pam_env.conf` - zmiana globalna przez PAM.  
Należy sprawdzać te pliki pod kątem niebezpiecznych zmiennych, zwłaszcza `LD_PRELOAD`, `PATH`, `IFS`, `LANG`, `TMPDIR`.

## Jak usuwać zmienne 
Służy do tego polecenie `unset NAZWA`. Przykład:
- `unset LD_PRELOAD` - usuwa zmienną z sesji.

## Zagrożenia
Szczegółowa analiza zagrożeń związanych z ze zmiennymi środowiskowymi (`LD_PRELOAD`, `LD_LIBRARY_PATH`, `PATH`, `IFS`, `TMPDIR`, `LANG`) znajduje się w pliki `05_pam_basics/security/pam_env_PL.md`. 
