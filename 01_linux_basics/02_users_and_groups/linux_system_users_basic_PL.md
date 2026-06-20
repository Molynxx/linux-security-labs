# System Users Basics

## Cel 
Celem laboratorium było zapoznanie się z rodzajami użytkowników systemu Linux w celu lepszego zrozumienia, jak analizować logi (np. `/var/log`) i identyfikować potencjalne zagrożenia związane z nieautoryzowanym dostępem do systemu.

## Wykonane kroki
- Przegląd pliku `/etc/passwd` w celu:
	- identyfikacji typów użytkowników,
	- zrozumienie znaczenia pól, takich jak nazwa użytkownika, UID, katalog domowy i powłoka (shell) oraz ich znaczenia z perspektywy bezpieczeństwa.
- Analiza polecenia last w celu sprawdzenia historii udanych logowań.

## Obserwacje
- w systemie występują trzy typy użytkowników:
	- root - użytkownik administracyjny o pełnych uprawnieniach,
	- użytkownicy systemowi - konta używane przez usługi i demony,
	- użytkownicy interaktywni (ludzie) - konta przeznaczone do logowania przez użytkowników.
- Użytkownicy interaktywni:
	- zwykle posiadają katalog w /home,
	- mają przypisaną powłokę (/bin/bash, /bin/zsh).
- Użytkownicy systemowi:
	- często nie posiadają katalogu domowego lub mają katalog techniczny (np. /var/lib/...),
	- mają ustawione /usr/sbin/nologin lub /bin/false,
	- nie powinni mieć możliwości logowania interaktywnego. 
- Użytkownik root posiada powłokę systemową, ale jego użycie powinno być ograniczone i kontrolowane.

## Wnioski bezpieczeństwa
- Potencjalnym zagrożeniem jest użytkownik systemowy posiadający interaktywną powłokę (/bin/bash, /bin/zsh), ponieważ umożliwia to logowanie do systemu.
- Użytkownicy systemowi powinni mieć zablokowaną możliwość logowania (nologin lub false).
- Użytkownicy interaktywni stanowią naturalny wektor ataku, (np. przejęcie konta, brute force).
- W przypadku wykrycia podejrzanego użytkownika należy:
	- ustalić moment utworzenia konta (jeśli dostępny np. backupy, auditd, provisioning logs),
	- zweryfikować, czy konto było zgłoszone i autoryzowane,
	- sprawdzić, czy logowania odbywały się lokalnie czy zdalnie (SSH),
	- przeanalizować historię logowań.
- Do analizy:
	- udanych logowań służy polecenie last, 
	- nieudanych logowań standardowo służy lastb, jeśli narzędzie jest dostępne w danej dystrybucji.  

## Monitorowanie kont interaktywnych (SOC/IR)
Konta interaktywne stanowią jeden z głównych wektorów ataku w systemach linux. Z perspektywy SOC/IR kluczowe jest nie tylko istnienie kont, ale monitorowanie ich aktywności.
W środowiskach produkcyjnych SOC tworzy alerty na podstawie:
- logowań o nietypowych godzinach, 
- logowań z nowych lub nietypowych adresów IP,
- logowań na konta, które zwykle nie są używane,
- sekwencji wielu nieudanych prób logowania zakończonych sukcesem.  
Do lokalnej analizy zdarzeń wykorzystywane są m.in.:
- last - historia udanych logowań,
- journalctl -u ssh --since "czas" lub analiza logowań `/var/log/auth.log`,
- korelacja czasu, użytkownika i źródła logowania.

## Ograniczenia laba
- W mojej instalacji Kali Linux plik `/var/log/btmp` istnieje, jednak dostępne narzędzie last nie potrafi odczytać jego zawartości (błąd formatu). Polecenie lastb nie jest dostępne jako osobne narzędzie. Ogranicza to możliwość analizy nieudanych logowań przy użyciu klasycznych narzędzi. 
- System był świeżo zainstalowany i nie zawierał danych symulujących rzeczywisty incydent bezpieczeństwa. 