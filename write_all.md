
# SOP — funkcje pomocnicze write_all()

## 1. Biblioteki

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <poll.h>

#define ERR(source) (perror(source), exit(EXIT_FAILURE))
```

---

## 2. BLOKUJĄCE — write_all()

**Zastosowanie:** zwykłe pliki, blokujące FIFO, pipe.

Funkcja próbuje zapisać wszystkie `size` bajtów.
Obsługuje częściowy zapis i `EINTR`.

```c
int write_all(int fd, const void *buf, size_t size)
{
    const char *p = buf;
    size_t written = 0;

    while (written < size)
    {
        ssize_t n = write(fd, p + written, size - written);

        if (n > 0)
        {
            written += (size_t)n;
        }
        else if (n == -1 && errno == EINTR)
        {
            // Przerwanie sygnalem - ponawiamy
            continue;
        }
        else
        {
            if (n == 0)
                errno = EIO;

            return -1;
        }
    }

    return 0;
}
```

### Użycie

```c
int fd = open("plik.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
if (fd == -1)
    ERR("open");

char buf[] = "Hello\n";

if (write_all(fd, buf, sizeof(buf) - 1) == -1)
    ERR("write_all");

if (close(fd) == -1)
    ERR("close");
```

### Wartości zwracane

- `0` — wszystkie bajty zostały zapisane.
- `-1` — błąd, sprawdzamy `errno`.
- Przy błędzie część danych może już być zapisana.

**Uwaga:** `write_all()` zwraca 0 przy sukcesie,
ale zwykłe `write()` zwraca liczbę zapisanych bajtów.

---

## 3. NIEBLOKUJĄCE — write_all_nb() z poll()

**Zastosowanie:** FIFO, pipe, socket w trybie O_NONBLOCK.

Funkcja:
1. Próbuje zapisywać dane.
2. Obsługuje częściowy zapis.
3. Przy `EAGAIN` czeka przez `poll()`.
4. Ponawia zapis, aż zapisze całość.

Nie wykonuje busy waiting.

```c
int write_all_nb(int fd, const void *buf, size_t size)
{
    const char *p = buf;
    size_t written = 0;

    while (written < size)
    {
        ssize_t n = write(fd, p + written, size - written);

        if (n > 0)
        {
            written += (size_t)n;
        }
        else if (n == -1 && errno == EINTR)
        {
            continue;
        }
        else if (n == -1 &&
                 (errno == EAGAIN || errno == EWOULDBLOCK))
        {
            struct pollfd pfd = {
                .fd = fd,
                .events = POLLOUT,
                .revents = 0
            };

            int ret;

            do {
                ret = poll(&pfd, 1, -1);
            } while (ret == -1 && errno == EINTR);

            if (ret == -1)
                return -1;

            if (pfd.revents & POLLNVAL) {
                errno = EBADF;
                return -1;
            }

            if (pfd.revents & (POLLERR | POLLHUP)) {
                errno = EPIPE;
                return -1;
            }

            // Gotowosc do zapisu - wracamy do write()
        }
        else
        {
            if (n == 0)
                errno = EIO;

            return -1;
        }
    }

    return 0;
}
```

### Użycie

```c
int fd = open("fifo1", O_WRONLY | O_NONBLOCK);

if (fd == -1)
    ERR("open");

char msg[] = "Hello\n";

if (write_all_nb(fd, msg, sizeof(msg) - 1) == -1)
    ERR("write_all_nb");

close(fd);
```

**Uwaga na FIFO:** jeśli nie ma czytelnika,
`open(O_WRONLY | O_NONBLOCK)` może zwrócić
`-1` z `errno == ENXIO`.

`poll()` czeka bez aktywnego odpytywania.

Ta wersja funkcji nadal czeka na zapisanie całego bufora,
ale czeka poprzez `poll()`, a nie przez blokujące `write()`.

Przy zapisie do zamkniętego FIFO/pipe proces może dostać
SIGPIPE. Aby obsługiwać EPIPE zamiast zakończenia procesu,
trzeba odpowiednio obsłużyć ten sygnał.

---

## 4. NIEBLOKUJĄCE — write_some_nb() bez czekania

**Zastosowanie:** kiedy mamy wiele deskryptorów
i sami sterujemy oczekiwaniem przez poll().

Ta funkcja:
- zapisuje tyle, ile może w tej chwili;
- nie czeka na możliwość zapisu;
- zapamiętuje postęp w `*written`;
- może zostać wywołana ponownie później.

### Kod

```c
// Zwraca:
//  1 - zapisano caly bufor
//  0 - na razie brak mozliwosci dalszego zapisu
// -1 - blad

int write_some_nb(int fd, const void *buf,
                  size_t size, size_t *written)
{
    const char *p = buf;

    while (*written < size)
    {
        ssize_t n = write(fd, p + *written,
                          size - *written);

        if (n > 0)
        {
            *written += (size_t)n;
        }
        else if (n == -1 && errno == EINTR)
        {
            continue;
        }
        else if (n == -1 &&
                 (errno == EAGAIN || errno == EWOULDBLOCK))
        {
            return 0;
        }
        else
        {
            if (n == 0)
                errno = EIO;

            return -1;
        }
    }

    return 1;
}
```

### Użycie z poll()

```c
char msg[] = "Hello world\n";
size_t written = 0;
size_t size = sizeof(msg) - 1;

while (written < size)
{
    struct pollfd pfd = {
        .fd = fd,
        .events = POLLOUT,
        .revents = 0
    };

    int ret;

    do {
        ret = poll(&pfd, 1, -1);
    } while (ret == -1 && errno == EINTR);

    if (ret == -1)
        ERR("poll");

    if (pfd.revents & POLLNVAL) {
        errno = EBADF;
        ERR("poll");
    }

    if (pfd.revents & (POLLERR | POLLHUP)) {
        errno = EPIPE;
        ERR("poll");
    }

    if (pfd.revents & POLLOUT)
    {
        int result = write_some_nb(fd, msg, size, &written);

        if (result == -1)
            ERR("write_some_nb");

        if (result == 1)
            break;
    }
}
```

### Dlaczego potrzebujemy written?

Przykład:

Chcemy zapisać 100 bajtów.

- Pierwsze wywołanie zapisuje 40 bajtów.
- Kolejne `write()` zwraca EAGAIN.
- Funkcja zwraca 0, a `written == 40`.
- Czekamy przez poll().
- Ponawiamy zapis od bajtu 40.

Dzięki temu nie zapisujemy tych samych danych dwa razy.

**Ważne:** `written` trzeba zainicjalizować na 0
przed rozpoczęciem wysyłania nowego bufora.

Bufor musi istnieć i nie może zostać zmodyfikowany,
dopóki nie zakończymy jego wysyłania.

---

## 5. Porównanie funkcji

| Funkcja | Tryb | Czeka? | Częściowy zapis |
|---|---|---|---|
| `write_all()` | Blokujący | Tak, przez write | Obsługuje |
| `write_all_nb()` | Nieblokujący | Tak, przez poll | Obsługuje |
| `write_some_nb()` | Nieblokujący | Nie | Zapamiętuje postęp |

## 6. Najważniejsze zasady

1. `write()` może zapisać mniej bajtów, niż żądamy.
2. `write() == 0` oznacza brak postępu.
3. `write_all() == 0` oznacza sukces naszej funkcji.
4. `EINTR` — operację można ponowić.
5. `EAGAIN` — operacja teraz nie może zostać wykonana.
6. Przy `O_NONBLOCK` używamy poll(), aby uniknąć busy waiting.
7. `write_all()` nie sprawdza wielkości tablicy!
8. Jeśli bufor ma 100 bajtów, nie wolno żądać zapisu 200.
9. Przy błędzie część danych może już zostać zapisana.
10. Dla wielu FIFO wygodna jest wersja zapamiętująca postęp.

## 7. Co wybrać na laboratoria?

**Zwykły plik przepisu:**
`write_all()`

**Jedno nieblokujące FIFO, chcemy wysłać całość:**
`write_all_nb()`

**Kilka FIFO obsługiwanych przez jeden poll():**
`write_some_nb()` i przechowywanie `written`
osobno dla każdego deskryptora.
