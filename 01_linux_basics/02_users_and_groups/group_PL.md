## /etc/group

### Czym jest /etc/group
Plik /etc/group zawiera informacje o grupach w systemie:
- nazwa grupy, 
- hash hasła lub marker 'x',
- GID grupy,
- członkowie grupy.

### Wykonane kroki
- Przejrzano plik poleceniem cat /etc/group,
- Użyto grep sudo dla pliku, aby sprawdzić którzy użytkownicy należą do grupy sudo,
- Użyto grep -E 'sudo|adm|docker' /etc/group, aby sprawdzić użytkowników należących do uprzywilejowanych grup,
- Celem powyższych działań było upewnienie się, że żaden nieautoryzowany użytkownik nie uzyskał dostępu,
- Utworzono katalog na skrypty i ustawiono jego uprawnienia wyłącznie dla właściciela,
- Stworzono skrypt wyświetlający wszystkich użytkowników z /etc/passwd oraz listę grup, do których należą,
- Dodano skrypt do zmiennej środowiskowej $PATH, aby można go było uruchomić z dowolnego miejsca w systemie. (Kontekst poboczny (shell/środowisko).

### Napotkane problemy
- Polecenie source ~/.bashrc zgłaszało wiele błędów, a skrypt .sh w $PATH nie działał.

### Rozwiązanie problemu
- Zrozumiano, jak działają powłoki w systemie,
- Kali Linux używa powłoki zsh, a nie bash, co było przyczyną problemów, 
- Aby naprawić problem, skrypt .sh przeniesiono do pliku konfiguracyjnego powłoki zsh, tj. ~/.zshrc,
- Po tych krokach problem został rozwiązany i skrypt działał poprawnie.

### Obserwacje
- Podczas modyfikacji zmiennych środowiskowych należy zwracać uwagę na używaną powłokę systemu,
- W pliku /etc/group wyróżniono dwa rodzaje grup:
	- Grupy systemowe - zawierają użytkowników systemowych, GID <1000, często nie pokazują użytkowników,
	- Grupy użytkowników - zawierają użytkowników interaktywnych, GID >= 1000, członkowie są wymienieni w pliku.

### Wnioski bezpieczeństwa
- Jeśli grupa systemowa nie zawiera żadnego użytkownika, jest to normalne i można zignorować,
- Jeśli grupa systemowa zawiera użytkownika interaktywnego, należy sprawdzić, czy powinien mieć w niej dostęp - szczególnie w grupach sudo, adm, docker, 
- Grupy użytkowników interaktywnych należy zawsze dokładnie sprawdzić, aby upewnić się, że nie ma nieautoryzowanych członków. 
	- Oprócz sudo, adm, docker warto sprawdzać grupy takie jak whell (w niektórych dystrybucjach) i shadow pod kątem nietypowych członków.
	- Każda obecność interaktywnego użytkownika w grupach systemowych wymaga weryfikacji.