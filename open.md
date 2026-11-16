
# SOP — open() i obsługa errno

## 1. Potrzebne biblioteki

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <sys/stat.h>

#define ERR(source) (perror(source), exit(EXIT_FAILURE))
```



```c
int fd = open("plik.txt", O_RDONLY);

if (fd == -1) {
    if (errno == ENOENT) {
        // Plik nie istnieje
    }
    else if (errno == EACCES) {
        // Brak uprawnien
    }
    else {
        ERR("open");
    }
}
```

## 3. Najczęstsze błędy open()

| Błąd | Znaczenie | Typowe zachowanie |
|---|---|---|
| `ENOENT` | Plik lub katalog nie istnieje | Utwórz plik lub zgłoś błąd |
| `EEXIST` | Plik już istnieje | Otwórz istniejący plik |
| `EACCES` | Brak uprawnień | Zgłoś błąd lub zmień tryb |
| `EINTR` | Operacja przerwana sygnałem | Ponów operację |
| `ENXIO` | Brak czytelnika FIFO | Obsłuż brak odbiorcy |
| `ENOTDIR` | Element ścieżki nie jest katalogiem | Zgłoś błąd |
| `EISDIR` | Próba otwarcia katalogu do zapisu | Zgłoś błąd |
| `EMFILE` | Za dużo deskryptorów w procesie | Zamknij niepotrzebne |
| `ENFILE` | Systemowy limit otwartych plików | Zgłoś błąd |
| `ELOOP` | Za dużo przekierowań symbolicznych | Zgłoś błąd |
| `EROFS` | System plików tylko do odczytu | Zgłoś błąd |

---

# 4. Gotowe przypadki z kodem

## A. EEXIST — utwórz plik albo otwórz istniejący

**Sytuacja:** chcemy utworzyć plik, ale jeśli istnieje, nie chcemy usuwać jego zawartości.

```c
int fd = open(argv[1], O_RDWR | O_CREAT | O_EXCL, 0644);

if (fd >= 0) {
    // Nowy plik
    // Tutaj inicjalizacja
}
else if (errno == EEXIST) {
    // Plik juz istnieje
    fd = open(argv[1], O_RDWR);

    if (fd == -1)
        ERR("open");

    // Tutaj sprawdzanie starego pliku
}
else {
    ERR("open");
}
```

**Zapamiętaj:**
- `O_CREAT` — tworzy, jeśli brak pliku.
- `O_EXCL` — razem z `O_CREAT` powoduje błąd, gdy plik istnieje.
- `EEXIST` — pozwala wykonać inną logikę dla starego pliku.

**Zastosowanie:** zadanie SOP z przepisem 2048 bajtów.

## B. ENOENT — jeśli plik nie istnieje, utwórz go

```c
int fd = open(argv[1], O_RDWR);

if (fd == -1) {
    if (errno == ENOENT) {
        fd = open(argv[1],
                  O_RDWR | O_CREAT | O_EXCL, 0644);

        if (fd == -1)
            ERR("open create");

        // Utworzony nowy plik
    }
    else {
        ERR("open");
    }
}
else {
    // Plik istnial
}
```

**Zapamiętaj:**
- `ENOENT` oznacza brak pliku lub części ścieżki.
- Jeśli brakuje katalogu, samo `O_CREAT` nie pomoże.
- Między próbami otwarcia inny proces może utworzyć plik.

**Zastosowanie:** program tworzący bazę danych, jeśli jej nie ma.

## C. EACCES — spróbuj otworzyć tylko do odczytu

**Sytuacja:** jeśli nie możemy modyfikować pliku, dopuszczamy tryb tylko do odczytu.

```c
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
    // Nie wolno wykonywac zapisu
}
else {
    // Mozemy czytac i zapisywac
}
```

**Zapamiętaj:**
- Nie każdy błąd `EACCES` da się rozwiązać przez `O_RDONLY`.
- Zmiana trybu ma sens tylko, jeśli zadanie na to pozwala.
- Przy wymaganym zapisie należy zgłosić błąd.

## D. EINTR — ponawianie przerwanego open()

**Sytuacja:** blokujące otwarcie FIFO zostało przerwane sygnałem.

```c
int fd;

do {
    fd = open(argv[2], O_RDONLY);
} while (fd == -1 && errno == EINTR);

if (fd == -1)
    ERR("open FIFO");
```

**Zapamiętaj:**
- `EINTR` oznacza przerwanie operacji sygnałem.
- Można spróbować ponownie.
- Nie jest to busy waiting — blokujące `open()` czeka na nadawcę.

**Zastosowanie:** otwieranie łącza nazwanego FIFO.

## E. ENXIO — brak czytelnika FIFO

**Sytuacja:** chcemy otworzyć FIFO do zapisu bez blokowania.

```c
int fd = open("tower1", O_WRONLY | O_NONBLOCK);

if (fd == -1) {
    if (errno == ENXIO) {
        // Nikt nie otworzyl FIFO do odczytu
        // Nie ma teraz odbiorcy
    }
    else {
        ERR("open FIFO");
    }
}
else {
    // FIFO otwarte do zapisu
}
```

**Zapamiętaj:**
- `O_WRONLY` — otwieramy do zapisu.
- `O_NONBLOCK` — nie czekamy podczas otwierania.
- `ENXIO` — przy FIFO nie ma czytelnika.

**Uwaga:** nie ponawiaj `open()` w pustej pętli, bo to busy waiting.

## F. EMFILE — za dużo otwartych deskryptorów

**Sytuacja:** program otwiera wiele plików i osiąga limit deskryptorów.

```c
int fd = open(argv[1], O_RDONLY);

if (fd == -1) {
    if (errno == EMFILE) {
        // Za duzo otwartych deskryptorow
        // Zamknij niepotrzebne deskryptory
        // i ewentualnie ponow operacje
    }
    else {
        ERR("open");
    }
}
```

**Zapamiętaj:**
- Każdy `open()` przy sukcesie przydziela deskryptor.
- Po zakończeniu pracy wykonujemy `close(fd)`.
- Nie należy bez końca ponawiać `open()` przy `EMFILE`.

---

# 5. Przydatne flagi open()

| Flaga | Znaczenie |
|---|---|
| `O_RDONLY` | Tylko odczyt |
| `O_WRONLY` | Tylko zapis |
| `O_RDWR` | Odczyt i zapis |
| `O_CREAT` | Utwórz plik, jeśli nie istnieje |
| `O_EXCL` | Z O_CREAT: błąd, jeśli plik istnieje |
| `O_TRUNC` | Wyczyść zawartość pliku |
| `O_APPEND` | Dopisuj na końcu |
| `O_NONBLOCK` | Tryb nieblokujący |

Przykłady:

```c
// Odczyt istniejacego pliku
open("plik.txt", O_RDONLY);

// Odczyt i zapis
open("plik.txt", O_RDWR);

// Utworz, jesli nie istnieje
open("plik.txt", O_RDWR | O_CREAT, 0644);

// Utworz tylko nowy plik
open("plik.txt", O_RDWR | O_CREAT | O_EXCL, 0644);

// Utworz lub wyczysc istniejacy
open("plik.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

// Dopisuj na koncu
open("plik.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);

// FIFO - odczyt blokujacy
open("tower1", O_RDONLY);

// FIFO - odczyt nieblokujacy
open("tower1", O_RDONLY | O_NONBLOCK);
```


1. Nie sprawdzaj `errno` bez wcześniejszego błędu funkcji.
2. `open()` zwraca `-1` przy błędzie, a nie `NULL`.
3. `O_CREAT` nie usuwa zawartości istniejącego pliku.
4. `O_TRUNC` usuwa zawartość istniejącego pliku.
5. `O_EXCL` używamy razem z `O_CREAT`.
6. `O_NONBLOCK` zmienia sposób działania FIFO.
7. `EINTR` często oznacza, że można ponowić operację.
8. `ENOENT` może oznaczać brak katalogu, a nie tylko pliku.
9. Nie ponawiaj błędnej operacji w nieskończonej pętli.
10. Po użyciu deskryptora wykonuj `close(fd)`.


**Najważniejsze kombinacje:**

- `O_CREAT | O_EXCL` + `EEXIST`
  - Utwórz nowy plik albo otwórz istniejący.

- `O_RDWR` + `EACCES`
  - Obsłuż brak uprawnień.

- `O_RDONLY` na FIFO + `EINTR`
  - Ponów przerwane otwieranie FIFO.

- `O_WRONLY | O_NONBLOCK` na FIFO + `ENXIO`
  - Obsłuż brak czytelnika.

- `open()` + `EMFILE`
  - Sprawdź, czy nie zapomniano o `close()`.

