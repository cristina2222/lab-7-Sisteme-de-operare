# Problema Firelor Albe si Negre (Readers-Writers Variant)

## Enunt

Exista o aplicatie cu mai multe fire de executie. Un fir este fie alb, fie negru si firele de executie trebuie sa acceseze o resursa care:
- **NU poate fi utilizata simultan** de fire albe si negre
- **Poate fi folosita simultan** de mai multe fire de acelasi tip

### Cerinte:
1. Protejati impotriva starvation (infometare)
2. Poate exista orice numar de fire care solicita acces la resursa in orice secventa
3. Evitati situatiile in care accesul unui fir este refuzat chiar daca nu exista alt fir de alt tip
4. Utilizarea resursei nu este preemptibila - odata ce un fir obtine acces, va folosi resursa cat este necesar
5. Utilizarea resursei de catre un fir este finita
6. Protectia impotriva infometarii garanteaza ca toate firele vor avea acces la resursa, in cele din urma
7. Trebuie implementat atat pentru Linux, cat si pentru Windows

---

## Solutia

### Conceptul de baza

Am implementat un sistem de **ture** (turn-based) pentru a preveni starvation:
- Fiecare "tura" permite unui numar limitat de fire (implicit 3) de acelasi tip sa acceseze resursa
- Dupa ce limita este atinsa SI exista fire de celalalt tip in asteptare, tura se schimba
- Daca nu asteapta nimeni de tipul opus, firele pot continua fara restrictii

### Structura ResourceManager

```c
typedef struct {
    // Primitive de sincronizare (mutex + condition variable)

    int white_count;      // Fire albe care folosesc resursa activ
    int black_count;      // Fire negre care folosesc resursa activ
    int white_waiting;    // Fire albe in asteptare
    int black_waiting;    // Fire negre in asteptare

    int current_turn;     // 0 = alb, 1 = negru, -1 = liber
    int turn_count;       // Cate fire au folosit resursa in tura curenta
    int max_per_turn;     // Limita per tura (anti-starvation)
} ResourceManager;
```

### Algoritmul can_access()

Un fir poate accesa resursa daca:
1. **Tipul opus NU foloseste resursa** (black_count == 0 pentru albi, white_count == 0 pentru negri)
2. **Una din conditiile urmatoare**:
   - Resursa e complet libera SI (nu e tura nimanui SAU e tura mea SAU tipul opus nu asteapta)
   - Resursa e folosita de tipul meu SI (nu am depasit limita per tura SAU tipul opus nu asteapta)

### Fluxul acquire_resource()

1. Ia lock-ul
2. Incrementeaza contorul de asteptare (white_waiting sau black_waiting)
3. Asteapta in bucla while(!can_access()) pe condition variable
4. Decrementeaza contorul de asteptare
5. Incrementeaza contorul activ (white_count sau black_count)
6. Actualizeaza tura curenta si turn_count
7. Elibereaza lock-ul

### Fluxul release_resource()

1. Ia lock-ul
2. Decrementeaza contorul activ
3. Daca nu mai sunt fire de tipul meu activ:
   - Daca tipul opus asteapta -> schimba tura
   - Altfel -> seteaza current_turn = -1 (liber)
4. Broadcast la toate firele care asteapta
5. Elibereaza lock-ul

---

## Implementare Linux (pthreads)

**Fisier:** `linux_solution.cpp`

**Compilare:**
```bash
g++ -pthread -o linux_solution linux_solution.cpp
```

**Rulare:**
```bash
./linux_solution
```

### Primitive folosite:

| Primitiva | Descriere |
|-----------|-----------|
| `pthread_mutex_t` | Mutex pentru protejarea sectiunii critice |
| `pthread_cond_t` | Condition variable pentru asteptare/notificare |
| `pthread_mutex_lock()` | Obtine mutex-ul |
| `pthread_mutex_unlock()` | Elibereaza mutex-ul |
| `pthread_cond_wait()` | Asteapta pe condition variable (elibereaza mutex-ul automat) |
| `pthread_cond_broadcast()` | Trezeste TOATE firele care asteapta |
| `pthread_create()` | Creeaza un fir nou |
| `pthread_join()` | Asteapta terminarea unui fir |

---

## Implementare Windows (Win32 API)

**Fisier:** `windows_solution.cpp`

**Compilare cu MSVC:**
```cmd
cl /EHsc windows_solution.cpp
```

**Compilare cu MinGW:**
```cmd
g++ -o windows_solution.exe windows_solution.cpp
```

**Rulare:**
```cmd
windows_solution.exe
```

### Primitive folosite:

| Primitiva | Descriere |
|-----------|-----------|
| `CRITICAL_SECTION` | Echivalent mutex (mai rapid decat Mutex pe Windows) |
| `CONDITION_VARIABLE` | Condition variable pentru asteptare/notificare |
| `EnterCriticalSection()` | Obtine critical section |
| `LeaveCriticalSection()` | Elibereaza critical section |
| `SleepConditionVariableCS()` | Asteapta pe condition variable |
| `WakeAllConditionVariable()` | Trezeste TOATE firele care asteapta |
| `CreateThread()` | Creeaza un fir nou |
| `WaitForMultipleObjects()` | Asteapta terminarea mai multor fire |

---

## Corespondenta Linux - Windows

| Linux (pthreads) | Windows (Win32) |
|------------------|-----------------|
| `pthread_mutex_t` | `CRITICAL_SECTION` |
| `pthread_cond_t` | `CONDITION_VARIABLE` |
| `pthread_mutex_lock()` | `EnterCriticalSection()` |
| `pthread_mutex_unlock()` | `LeaveCriticalSection()` |
| `pthread_cond_wait()` | `SleepConditionVariableCS()` |
| `pthread_cond_broadcast()` | `WakeAllConditionVariable()` |
| `pthread_create()` | `CreateThread()` |
| `pthread_join()` | `WaitForMultipleObjects()` |
| `usleep(microsec)` | `Sleep(millisec)` |

---

## Exemplu de output

```
=== Pornire test cu 5 fire albe si 5 fire negre ===

[ALB 1] Pornit, incearca sa obtina resursa...
[ALB 1] A obtinut accesul la resursa (albe: 1, negre: 0)
[ALB 1] Foloseste resursa...
[NEGRU 1] Pornit, incearca sa obtina resursa...
[ALB 2] A obtinut accesul la resursa (albe: 2, negre: 0)
[ALB 2] Foloseste resursa...
[ALB 3] A obtinut accesul la resursa (albe: 3, negre: 0)
[ALB 3] Foloseste resursa...
[ALB 1] A eliberat resursa (albe: 2, negre: 0)
... (dupa 3 albi, tura se schimba la negri)
[NEGRU 1] A obtinut accesul la resursa (albe: 0, negre: 1)
[NEGRU 2] A obtinut accesul la resursa (albe: 0, negre: 2)
[NEGRU 3] A obtinut accesul la resursa (albe: 0, negre: 3)
...
=== Toate firele au terminat ===
```

---

## De ce functioneaza anti-starvation?

1. **Limita per tura**: Dupa `max_per_turn` fire de un tip, se da prioritate celuilalt tip
2. **Schimbarea turei**: Cand ultimul fir de un tip termina, tura trece automat la celalalt tip daca asteapta
3. **Garantie de progres**: Fiecare tip va primi tura eventual, deci toate firele vor fi servite
4. **Fara refuz inutil**: Daca tipul opus nu asteapta, firele pot continua liber
