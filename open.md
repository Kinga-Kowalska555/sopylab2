SOP — open()
open() zwraca deskryptor >= 0 przy sukcesie albo -1 przy błędzie. errno sprawdzamy dopiero wtedy, gdy open() zwróci -1. Udane wywołanie nie musi wyzerować errno.
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

#define ERR(source) (perror(source), exit(EXIT_FAILURE))
int fd = open(path, O_RDONLY);
if (fd == -1) {
    if (errno == ENOENT) {
        // plik lub element sciezki nie istnieje
    } else {
        ERR("open");
    }
}
perror("open") wypisuje opis błędu odpowiadającego bieżącemu errno na stderr. Jeśli musisz zachować numer błędu na później, zapisz go od razu: int saved_errno = errno;.

2. Najczęstsze błędy open()
Kod	Znaczenie	Kiedy spotkasz?	Typowe działanie
ENOENT	Brak pliku lub składnika ścieżki	open("brak.txt", O_RDONLY)	Utwórz, jeśli wymagane; w przeciwnym razie błąd
EEXIST	Plik już istnieje	`O_CREAT	O_EXCL`
EACCES	Brak uprawnień	Brak prawa odczytu/zapisu lub przejścia przez katalog	Błąd albo tryb awaryjny tylko do odczytu, jeśli zadanie na to pozwala
EINTR	Otwarcie przerwane przez sygnał	Blokujące open() na FIFO	Ponów operację, o ile nie masz logiki anulowania
ENXIO	Przy FIFO: nie ma czytelnika	`open(fifo, O_WRONLY	O_NONBLOCK)`
ENOTDIR	Składnik ścieżki miał być katalogiem, ale nim nie jest	open("plik.txt/dane", ...)	Popraw ścieżkę
EISDIR	Próba otwarcia katalogu do zapisu	open("katalog", O_WRONLY)	Nie traktuj katalogu jak zwykłego pliku
EMFILE	Proces ma zbyt dużo otwartych deskryptorów	Zapomniane close()	Zamykaj niepotrzebne deskryptory
ENFILE	Systemowy limit otwartych plików	Dużo otwartych plików w całym systemie	Zgłoś problem z zasobami
ELOOP	Za dużo przekierowań przez dowiązania symboliczne	Zapętlone symlinki	Popraw ścieżkę
EROFS	System plików tylko do odczytu	Próba zapisu/tworzenia pliku	Zgłoś błąd albo wybierz inne miejsce


Uwaga: konkretne kody mogą zależeć od systemu i rodzaju otwieranego pliku. Nie każdy błąd wymaga osobnej gałęzi — najczęściej obsługujesz tylko te, które zmieniają zachowanie programu, a resztę przekazujesz do perror().
3. Gotowe przypadki z kodem
A. EEXIST: utwórz nowy plik, a jeśli już jest — otwórz istniejący
Idealne do zadania z przepisem: dla nowego pliku zapisujesz 32 linie, dla starego sprawdzasz rozmiar.
int fd = open(argv[1], O_RDWR | O_CREAT | O_EXCL, 0644);

if (fd >= 0) {
    // Nowy plik: wpisz dane poczatkowe
} else if (errno == EEXIST) {
    fd = open(argv[1], O_RDWR);
    if (fd == -1)
        ERR("open istniejącego");
    // Istniejacy plik: np. sprawdz rozmiar przez lseek
} else {
    ERR("open");
}
O_EXCL działa tu razem z O_CREAT: jeśli ścieżka już istnieje, open() się nie powiedzie i ustawi errno = EEXIST.
B. ENOENT: brak pliku — spróbuj utworzyć
int fd = open(argv[1], O_RDWR);

if (fd == -1) {
    if (errno == ENOENT) {
        fd = open(argv[1], O_RDWR | O_CREAT, 0644);
        if (fd == -1)
            ERR("open create");
        // Plik mogl zostac utworzony; tutaj dalsza logika
    } else {
        ERR("open");
    }
}
Pułapka: między próbami open() inny proces może utworzyć plik. Jeśli musisz niezawodnie odróżniać nowy od istniejącego, lepszy jest wariant A (O_CREAT | O_EXCL).
C. EACCES: nie udało się otworzyć do zapisu — spróbuj tylko do odczytu
int fd = open(argv[1], O_RDWR);
int read_only = 0;

if (fd == -1 && errno == EACCES) {
    fd = open(argv[1], O_RDONLY);
    if (fd >= 0)
        read_only = 1;
}
if (fd == -1)
    ERR("open");

if (read_only) {
    // Nie wykonuj operacji zapisu!
}
Tylko jeśli zadanie dopuszcza tryb awaryjny. W zadaniu z tortem przepis ma być otwarty jako O_RDWR, więc nie należy przechodzić na O_RDONLY.
D. EINTR: otwieranie FIFO przerwane sygnałem
int fd;
do {
    fd = open(argv[2], O_RDONLY); // FIFO: moze blokowac
} while (fd == -1 && errno == EINTR);

if (fd == -1)
    ERR("open FIFO");
Przy ponawianiu open() upewnij się, że program rzeczywiście ma kontynuować czekanie (a nie np. kończyć na sygnał).
E. ENXIO: nieblokujące otwarcie FIFO do zapisu bez czytelnika
int fd = open("tower1", O_WRONLY | O_NONBLOCK);

if (fd == -1) {
    if (errno == ENXIO) {
        // Brak odbiorcy - np. zgłoś to i zakończ
        const char msg[] = "Brak czytelnika FIFO\n";
        write(STDERR_FILENO, msg, sizeof(msg) - 1);
    } else {
        ERR("open FIFO");
    }
}
Nie rób while (open(...) == -1) {} — to busy waiting. Sam poll() nie rozwiązuje bezpośrednio oczekiwania na możliwość otwarcia jeszcze nieotwartego deskryptora FIFO do zapisu.
F. ENOENT + stat: najpierw sprawdź, czy wieża istnieje i jest FIFO
W zadaniu z tortem brak ścieżki i zły typ pliku mają dawać ten sam komunikat:
struct stat st;

if (stat(argv[2], &st) == -1 || !S_ISFIFO(st.st_mode)) {
    fprintf(stderr, "Nie ma babci na wieży %s\n", argv[2]);
    exit(EXIT_FAILURE);
}

int tower = open(argv[2], O_RDONLY);
if (tower == -1)
    ERR("open tower");
Tutaj nie musisz rozróżniać ENOENT i ENOTDIR, bo zadanie narzuca wspólny komunikat o braku babci. stat() sprawdza typ pliku, a open() otwiera FIFO. Między tymi wywołaniami ścieżka może się zmienić.
