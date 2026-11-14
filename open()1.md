open() 

1. Czym jest open()?
#include <fcntl.h>  // open, O_RDONLY, O_WRONLY, O_CREAT, ...
#include <unistd.h> // close, read, write

int open(const char *path, int oflag, ...);
- path — ścieżka do pliku, np. "dane.txt" lub "pliki/dane.txt".
- oflag — tryb dostępu i ewentualne flagi dodatkowe, łączone operatorem bitowym |.
- Trzeci argument (mode, np. 0644) jest wymagany, gdy używamy O_CREAT (także przy linuxowym O_TMPFILE). Określa żądane uprawnienia nowo tworzonego pliku.
- Sukces: zwraca nieujemny deskryptor (fd), np. 3; błąd: zwraca -1 i ustawia errno.
- Po zakończeniu używania deskryptora zamykamy go przez close(fd) i także kontrolujemy wynik close().
int fd = open("dane.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    return EXIT_FAILURE; // w main()
}
/* ... operacje na pliku ... */
if (close(fd) == -1) {
    perror("close");
    return EXIT_FAILURE;
}
Deskryptor ≠ nazwa pliku. Jest małą liczbą całkowitą identyfikującą otwarty zasób w danym procesie. Zwyczajowo 0 to stdin, 1 stdout, 2 stderr, a kolejny może mieć wartość 3 (ale nie musi). Po close(3) numer 3 może zostać wykorzystany ponownie. close() nie usuwa pliku z dysku.
Gdzie open("dane.txt", ...) szuka pliku?
Ścieżka względna jest liczona od bieżącego katalogu roboczego procesu, a nie od katalogu zawierającego program.c lub plik wykonywalny.
cd projekt
./program                    # "dane.txt" -> projekt/dane.txt
cd ..
./projekt/program             # "dane.txt" -> dane.txt w katalogu nadrzędnym
"podfolder/dane.txt" — ścieżka względna; "/home/ola/dane.txt" — bezwzględna. Jeśli brakuje pliku albo części ścieżki, często dostaniemy ENOENT.
2. Flagi open() — znaczenie i pułapki
2.1. Tryby dostępu (wybierasz jeden)
Flaga	Działanie	Ważne
O_RDONLY	tylko odczyt	nie tworzy i nie usuwa pliku
O_WRONLY	tylko zapis	nie oznacza dopisywania na koniec
O_RDWR	odczyt i zapis	nie oznacza automatycznie wyczyszczenia


Trzeba wybrać dokładnie jeden tryb dostępu; nie łączymy np. O_RDONLY | O_WRONLY zamiast O_RDWR.
2.2. Flagi dodatkowe — najważniejsze z tutorialu
Flaga	Efekt	Pułapka
O_CREAT	utwórz plik, jeśli nie istnieje	dodaj trzeci argument mode
O_EXCL + O_CREAT	tworzenie wyłącznie nowego pliku	jeśli plik już istnieje → -1, errno == EEXIST
O_APPEND	każde write() dopisuje na końcu	samo O_WRONLY nie dopisuje
O_TRUNC	przy poprawnym otwarciu istniejącego zwykłego pliku do zapisu zmienia jego długość na 0	nie czyści „po write()”, tylko już w open()
O_NONBLOCK	w odpowiednich typach zasobów operacje I/O nie czekają na gotowość	istotne głównie dla FIFO/terminali/gniazd; dla zwykłych plików na ogół nie zmienia zachowania I/O


Uwaga o O_TRUNC + O_RDONLY: nie używaj takiego połączenia. POSIX nie definiuje jednoznacznie tego zachowania; na Linuksie zwykły plik może zostać przycięty mimo O_RDONLY. Używaj O_TRUNC z O_WRONLY albo O_RDWR.

Różnica O_APPEND vs O_TRUNC:

Jeśli plik zawiera ABCDE, a zapisujemy dwa bajty XY:
open("plik.txt", O_WRONLY);             // po write("XY", 2): XYCDE
open("plik.txt", O_WRONLY | O_APPEND);  // po write("XY", 2): ABCDEXY
open("plik.txt", O_WRONLY | O_TRUNC);   // od razu pusty, po write: XY
Powyższe to trzy oddzielne scenariusze, każdy startuje z zawartością ABCDE.
2.3. Gotowe kombinacje
open("plik.txt", O_RDONLY);                              // odczytaj istniejący
open("plik.txt", O_WRONLY);                              // zapisuj istniejący od początku
open("plik.txt", O_RDWR);                                // czytaj i pisz
open("plik.txt", O_WRONLY | O_CREAT, 0644);              // utwórz, jeśli nie ma; bez czyszczenia
open("plik.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);    // utwórz lub wyczyść
open("plik.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);   // utwórz lub dopisuj
open("plik.txt", O_WRONLY | O_CREAT | O_EXCL, 0644);     // utwórz wyłącznie nowy
open("plik.txt", O_RDONLY | O_NONBLOCK);                 // tryb nieblokujący (zależnie od zasobu)
2.4. Dwie dodatkowe flagi, które warto rozpoznawać
- O_CLOEXEC — zamknij deskryptor przy udanym exec uruchamiającym nowy program; przydaje się w bardziej rozbudowanych zadaniach.
- O_DIRECTORY (Linux) — zażądaj, aby ścieżka wskazywała katalog; inaczej błąd ENOTDIR.
3. Uprawnienia mode, umask, chmod i fchmod
Kto korzysta z których bitów? System sprawdza tożsamość i grupy procesu wykonującego open(). Jeśli proces jest właścicielem, decydują bity właściciela; w przeciwnym razie sprawdzana jest grupa, a potem pozostali. To nie jest wybór „najkorzystniejszych” bitów. Dostęp mogą dodatkowo zmieniać ACL, prawa do katalogów, przywileje administratora itd.

Ważne rozróżnienie: O_RDWR mówi, co program chce robić z otwartym plikiem; 0644 mówi, jakie prawa mają być nadane tworzonemu plikowi. Podanie 0644 nie nadaje automatycznie prawa zapisu do już istniejącego pliku.
3.2. Co robi umask?
umask to maska uprawnień odejmowanych bitowo przy tworzeniu zasobu. W typowym przypadku, gdy katalog nie ma domyślnych ACL:
rzeczywiste_prawa = mode & ~umask
Przykłady:
mode w open()	umask	Wynik
0777	0022	0755
0666	0022	0644
0644	0022	0644
0666	0077	0600
0600	0000	0600


umask nie może dodać prawa, którego nie było w mode. Jeśli katalog ma domyślne ACL, końcowy wynik może być ustalany według reguł ACL zamiast umask.
umask            # pokaż maskę powłoki
umask 0077       # zmień maskę w bieżącej powłoce
ls -l plik.txt   # zobacz realne uprawnienia
chmod 600 plik.txt
W C:
#include <sys/stat.h>
mode_t previous = umask(0022);  // zmień maskę procesu i zapamiętaj poprzednią
/* ... tworzenie ... */
umask(previous);               // opcjonalne przywrócenie
Maska jest ustawieniem procesu, nie parametrem pojedynczego pliku. Procesy potomne dziedziczą maskę; po exec pozostaje ona w mocy.
3.3. Zmiana uprawnień istniejącego pliku
#include <sys/stat.h>

chmod("plik.txt", 0600); // według ścieżki
fchmod(fd, 0600);        // według otwartego deskryptora
Obie funkcje zwracają 0 przy sukcesie, -1 przy błędzie. Nie zapomnij o kontroli wyniku i errno. Zmiana uprawnień wymaga odpowiedniego uprawnienia (zazwyczaj własności pliku lub odpowiedniego przywileju).
if (fchmod(fd, 0600) == -1) {
    perror("fchmod");
    /* obsłuż błąd */
}
chmod/`fchmod` działają na istniejącym pliku i umask nie przycina ustawianych przez nie praw. Nie używaj 0777 bez potrzeby — daje wszystkim bardzo szeroki dostęp.
4. Błędy open(): errno, perror, strerror
#include <errno.h>   // errno, ENOENT, EACCES, ...
#include <stdio.h>   // perror
#include <string.h>  // strerror
Gdy open() zwraca -1, ustawia errno na konkretny kod przyczyny błędu. Nie sprawdzaj errno bez sprawdzenia wyniku: udane open() nie musi zerować errno.
int fd = open("plik.txt", O_RDONLY);
if (fd == -1) {
    if (errno == ENOENT) {
        puts("Pliku brak");
    } else if (errno == EACCES) {
        puts("Brak uprawnien");
    } else {
        perror("open"); // np. "open: ..."
    }
}
- perror("open") wypisuje etykietę i opis aktualnego errno na stderr.
- strerror(errno) zwraca tekstowy opis błędu, np. do fprintf(stderr, "Blad: %s\n", strerror(errno));.
- Kod błędu (ENOENT, EACCES itd.) jest stałą symboliczną. Nie wpisuj na sztywno wartości liczbowych, bo mogą różnić się między systemami.
- Jeżeli przed obsługą błędu wykonasz inne operacje mogące zmienić errno, zapisz go: int err = errno;.
4.1. Tabela przydatnych kodów błędów open()
errno	Znaczenie / kiedy występuje	Przykładowe zachowanie na labach
ENOENT	Brak pliku lub składnika ścieżki; bywa też przy wiszącym dowiązaniu symbolicznym	utwórz plik tylko jeśli zadanie na to pozwala; pamiętaj, że może brakować katalogu
EEXIST	Przy `O_CREAT	O_EXCL` wskazany plik już istnieje
EACCES	Brak prawa dostępu do pliku albo przeszukiwania któregoś katalogu na ścieżce	zgłoś brak uprawnień; zwykle nie ponawiaj bez zmiany warunków
EPERM	Operacja zabroniona ze względów uprawnień/polityki, w określonych sytuacjach	zgłoś odmowę; nie traktuj jak ENOENT
EINTR	Wywołanie przerwane przez sygnał	często można ponowić wywołanie
EMFILE	Proces wykorzystał swój limit deskryptorów	zamknij niepotrzebne fd / ogranicz liczbę otwarć
ENFILE	Systemowy limit otwartych plików	zgłoś brak zasobów systemu
EISDIR	Katalog otwierany z niedozwolonym trybem zapisu	zgłoś złą ścieżkę / rodzaj zasobu
ENOTDIR	Składnik ścieżki powinien być katalogiem, ale nim nie jest; także O_DIRECTORY dla nie-katalogu	popraw ścieżkę / rodzaj pliku
EINVAL	Nieprawidłowa kombinacja flag lub argumentów (szczegóły zależne od systemu)	popraw wywołanie, nie zapętlaj
ENOSPC	Brak miejsca na utworzenie nowego pliku	zgłoś brak miejsca; nie ponawiaj bez rozwiązania problemu
EDQUOT	Przekroczony limit miejsca/inodów dla użytkownika lub grupy	zgłoś limit przydziału
EROFS	Próba utworzenia lub zapisu na systemie plików tylko do odczytu	zakończ albo wybierz inny katalog
ELOOP	Zbyt wiele dowiązań symbolicznych w ścieżce	zgłoś problem ze ścieżką
ENAMETOOLONG	Nazwa lub ścieżka za długa	popraw ścieżkę
ENOMEM	Niewystarczające zasoby pamięci jądra	zgłoś brak zasobów
EIO	Błąd wejścia/wyjścia	zgłoś błąd I/O
ENXIO	W niektórych szczególnych zasobach (np. FIFO otwarte nieblokująco do zapisu bez czytelnika)	obsłuż specyfikę zasobu, nie zakładaj zwykłego pliku
EAGAIN / EWOULDBLOCK	Przy niektórych operacjach nieblokujących (np. specyficzne blokady i zasoby), zasób chwilowo niedostępny	czekaj na gotowość / użyj poll, nie rób busy waiting


4.2. Jak zachować pierwotny kod błędu?
int fd = open("plik.txt", O_RDONLY);
if (fd == -1) {
    int saved_errno = errno;  // od razu zapamiętaj przyczynę
    fprintf(stderr, "open nie wyszedl: %s\n", strerror(saved_errno));
    if (saved_errno == ENOENT) {
        /* inna reakcja */
    }
}
4.3. ERR() z tutorialu — kiedy wystarczy, a kiedy nie?
#include <stdio.h>
#include <stdlib.h>
#define ERR(source) (perror(source), fprintf(stderr, "%s:%d\n", __FILE__, __LINE__), exit(EXIT_FAILURE))
if (fd == -1)
    ERR("open");
ERR() jest makrem autora tutorialu, nie funkcją wbudowaną. Wypisuje błąd, nazwę pliku źródłowego i numer linii, a następnie kończy cały proces. Jest dobre dla błędów krytycznych, lecz nie używaj go bezwarunkowo, gdy polecenie wymaga różnych reakcji na ENOENT, EEXIST, EINTR itd.
5. Gotowe wzorce do kopiowania i modyfikowania
Wszystkie przykłady zakładają Linux/POSIX. W programie sprawdzaj także wynik close(); pominięcie obsługi wyjątków w krótkich fragmentach oznaczono komentarzem.

Wzorzec A — zwykłe otwarcie i zakończenie na błędzie
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd = open("dane.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return EXIT_FAILURE;
    }

    puts("Otwarto plik");
    if (close(fd) == -1) {
        perror("close");
        return EXIT_FAILURE;
    }
    return EXIT_SUCCESS;
}
Wzorzec B — brak pliku (ENOENT) to osobny przypadek
int fd = open("dane.txt", O_RDWR);
if (fd == -1) {
    if (errno == ENOENT) {
        /* Uwaga: może brakować katalogu, a nie samego pliku. */
        fd = open("dane.txt", O_RDWR | O_CREAT, 0644);
        if (fd == -1) {
            perror("open - tworzenie");
            return EXIT_FAILURE;
        }
    } else if (errno == EACCES) {
        fprintf(stderr, "Brak dostepu\n");
        return EXIT_FAILURE;
    } else {
        perror("open");
        return EXIT_FAILURE;
    }
}
/* ... uzyj fd ... i close(fd) ... */
Prostszy wariant, gdy nie musisz odróżniać „był już plik” od „utworzono nowy”:
int fd = open("dane.txt", O_RDWR | O_CREAT, 0644);
Wzorzec C — twórz tylko nowy (EEXIST to normalny konflikt)
int fd = open("wynik.txt", O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd == -1) {
    if (errno == EEXIST) {
        puts("Plik juz istnieje — pomijam tworzenie");
        /* Kontynuuj lub zakonc — zgodnie z trescia zadania. */
    } else {
        perror("open");
        return EXIT_FAILURE;
    }
} else {
    /* Nowy plik zostal utworzony: ... */
    if (close(fd) == -1) {
        perror("close");
        return EXIT_FAILURE;
    }
}
Wzorzec D — ponów open() tylko przy EINTR
int fd;
do {
    fd = open("dane.txt", O_RDONLY);
} while (fd == -1 && errno == EINTR);

if (fd == -1) {
    perror("open");
    return EXIT_FAILURE;
}
Wzorzec E — switch(errno) i odrębne ścieżki
int fd = open("dane.txt", O_RDONLY);
if (fd == -1) {
    int err = errno;
    switch (err) {
        case ENOENT:
            fprintf(stderr, "Brak pliku lub czesci sciezki\n");
            break;
        case EACCES:
            fprintf(stderr, "Brak wymaganych uprawnien\n");
            break;
        case EMFILE:
            fprintf(stderr, "Limit deskryptorow procesu\n");
            break;
        default:
            fprintf(stderr, "open: %s\n", strerror(err));
            break;
    }
    return EXIT_FAILURE;
}
Wzorce B–E to fragmenty do wstawienia wewnątrz main(); wymagają odpowiednich nagłówków (<fcntl.h>, <unistd.h>, <errno.h>, <stdio.h>, <stdlib.h>, a dla strerror także <string.h>).
Wzorzec F — utworzenie pliku, sprawdzenie rzeczywistych praw, fchmod
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

int main(void) {
    umask(0022);
    int fd = open("nowy_test.txt", O_WRONLY | O_CREAT | O_EXCL, 0777);
    if (fd == -1) {
        perror("open");
        return EXIT_FAILURE;
    }
    struct stat st;
    if (fstat(fd, &st) == -1) {
        perror("fstat");
        close(fd);
        return EXIT_FAILURE;
    }
    printf("Po utworzeniu: %03o\n", (unsigned)(st.st_mode & 0777)); // typowo 755

    if (fchmod(fd, 0600) == -1) {
        perror("fchmod");
        close(fd);
        return EXIT_FAILURE;
    }
    if (fstat(fd, &st) == -1) {
        perror("fstat");
        close(fd);
        return EXIT_FAILURE;
    }
    printf("Po fchmod: %03o\n", (unsigned)(st.st_mode & 0777));   // 600

    if (close(fd) == -1) {
        perror("close");
        return EXIT_FAILURE;
    }
    return EXIT_SUCCESS;
}
Test:
gcc -std=c11 -Wall -Wextra -pedantic program.c -o program
rm -f nowy_test.txt
./program
ls -l nowy_test.txt

8. Dokumentacja przydatna na laboratorium
man 3p open       # POSIX: składnia, flagi, błędy
man 2 open        # szczegóły Linuksa
man 2 umask
man 2 chmod
man 2 fchmod
man 2 close
man 3 perror
