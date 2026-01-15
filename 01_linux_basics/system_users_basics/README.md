## Cel laba
Celem Laba było zapoznananie się z rodzajami użytkowników systemu Linux w celu lepszego zrozumienia, jak analizować logi (np. /var/log ) i identyfikować potencjalne zagrożenia związane z nieautoryzowanym dostępem do systemu.

## Wykonane kroki
- Przegląd pliku /etc/passwd w celu:
	- identyfikacji typów użytkowników,
	- zrozumienie pól takich jak nazwa użytkownika, UID, katalog domowy i powłoka (shell).
- Analiza polecenia last w celu sprawdzenia historii udanych logowań.

## Obserwacje
- w systemie występują trzy typy użytkowników:
	a. root - użytkownik administracyjny o pełnych uprawnieniach,
	b. użytkownicy systemowi - konta używane przez usługi i demony,
	c. użytkownicy interaktywni (ludzie) - konta przeznaczone do logowania przez użytkowników.
- Użytkownicy interaktywni:
	- posiadają katalog /home,
	- mają przypisaną powłokę (/bin/bash, /bin/zsh).
- Użytkownicy systemowi:
	- zazwyczaj nie posiadają katalogu domowego,
	- mają ustawione /usr/sbin/nologin lub /bin/false,
	- nie powinni mieć możliwości logowania interaktywnego. 
- Użytkownik root posiada powłokę systemową, ale jego użycie powinno być ograniczone i kontrolowane.

## Wnioski bezpieczeństwa
- Potencjalnym zagrożeniem jest użytkownik systemowy posiadający interaktywną powłokę (/bin/bash, /bin/zsh), ponieważ umożliwia to logowanie do systemu.
- Użytkownicy systemowi powinni mieć zablokowaną możliwość logowania (nologin lub false).
- Uźytkownicy interaktwni stanowią naturalny wektor ataku, (np. przejęcie konta, brute force).
- W przypadku wykrycia podejrzanego użytkownika należy:
	- sprawdzić datę utworzenia konta,
	- zweryfikować, czy konto było zgłoszone i autoryzowane,
	- sprawdzić, czy logowania odbywały się lokalnie czy zdalnie (SSH),
	- przeanalizować historię logowań.
-Do analizy:
	- udanych logowań służy polecenie last, 
	- nieudanych logowań standardowo służy lastb.

## Ograniczenia laba
- W mojej instalacji Kali Linux plik /var/log/btmp istnieje, jednak dostępne narzędzie last nie potrafi odczytać jego zawartości (błąd formatu). Polecenie lastb nie jest dostępne jako osobne narzędzie. Ogrnicza to możliwość analizy nieudanych logowań przy użyciu klasycznych narzędzi. 
- System był świeżo zainstalowany i nie zawirał danych symulujących rzeczywisty incydnet bezpieczeństwa. 