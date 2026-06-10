Hands-On 6: Deadlock and Starvation

**Nama:** I Dewa Nyoman Acarya Wibawantra
**NRP:** 5025251222
**Mata Kuliah:** Sistem Operasi

---

## Lab 1 - Thread Concurrency and Resource Competition

### Tujuan
Mengamati eksekusi konkuren dua thread yang bersaing menggunakan resource (CPU, terminal) secara bersamaan, serta memahami bahwa tanpa sinkronisasi, urutan eksekusi bersifat nondeterministik.

### Script - `lab1.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* task(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < 5; i++) {
        printf("Thread %d is running iteration %d\n", id, i);
        sleep(1);
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;
    int id1 = 1, id2 = 2;

    pthread_create(&t1, NULL, task, &id1);
    pthread_create(&t2, NULL, task, &id2);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab1.c -o lab1 -pthread
./lab1
```

### Hasil Eksekusi

```
Thread 1 is running iteration 0
Thread 2 is running iteration 0
Thread 2 is running iteration 1
Thread 1 is running iteration 1
Thread 1 is running iteration 2
Thread 2 is running iteration 2
Thread 2 is running iteration 3
Thread 1 is running iteration 3
Thread 1 is running iteration 4
Thread 2 is running iteration 4
```

### Analisis

Output kedua thread saling bercampur dan urutannya berubah-ubah setiap kali dijalankan. Ini terjadi karena thread scheduler di OS memutuskan kapan setiap thread mendapat giliran CPU — programmer tidak bisa mengontrol urutan ini tanpa mekanisme sinkronisasi. Meskipun belum ada error di lab ini, interleaving yang tidak terprediksi inilah yang menjadi akar masalah race condition dan deadlock di lab-lab berikutnya. Kedua thread bersaing untuk resource yang sama (stdout/terminal), yang merupakan contoh sederhana dari resource competition.

---

## Lab 2 - Simulating Deadlock with Threads

### Tujuan
Mendemonstrasikan deadlock — kondisi di mana dua thread saling menunggu resource yang dipegang oleh thread lain, sehingga keduanya terhenti selamanya.

### Script - `lab2.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t A, B;

void* thread1(void* arg) {
    pthread_mutex_lock(&A);
    printf("Thread 1 locked A\n");
    sleep(1);

    pthread_mutex_lock(&B);
    printf("Thread 1 locked B\n");

    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}

void* thread2(void* arg) {
    pthread_mutex_lock(&B);
    printf("Thread 2 locked B\n");
    sleep(1);

    pthread_mutex_lock(&A);
    printf("Thread 2 locked A\n");

    pthread_mutex_unlock(&A);
    pthread_mutex_unlock(&B);
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_mutex_init(&A, NULL);
    pthread_mutex_init(&B, NULL);

    pthread_create(&t1, NULL, thread1, NULL);
    pthread_create(&t2, NULL, thread2, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab2.c -o lab2 -pthread
./lab2
```

### Hasil Eksekusi

```
Thread 1 locked A
Thread 2 locked B
(program hangs - tidak ada output lagi, harus di-kill dengan Ctrl+C)
```

### Analisis

Program membeku setelah kedua thread masing-masing mengunci satu mutex. Thread 1 memegang A dan menunggu B, sementara Thread 2 memegang B dan menunggu A — ini adalah **circular wait**, salah satu dari empat syarat terjadinya deadlock (Coffman conditions):

1. **Mutual Exclusion** — mutex hanya bisa dipegang satu thread
2. **Hold and Wait** — thread memegang satu resource sambil menunggu yang lain
3. **No Preemption** — resource tidak bisa diambil paksa dari thread yang memegangnya
4. **Circular Wait** — Thread 1 → tunggu B (dipegang Thread 2) → tunggu A (dipegang Thread 1)

Keempat kondisi terpenuhi secara bersamaan, sehingga deadlock terjadi. `sleep(1)` sengaja ditambahkan agar kedua thread sempat mengunci mutex pertama sebelum mencoba mengunci mutex kedua, memperbesar peluang deadlock terjadi.

---

## Lab 3 - Deadlock Prevention via Resource Ordering

### Tujuan
Mencegah deadlock dengan mengeliminasi kondisi circular wait — kedua thread mengakuisisi resource dalam urutan yang sama (A dulu, baru B).

### Script - `lab3.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t A, B;

void* safe_thread(void* arg) {
    int id = *(int*)arg;

    printf("Thread %d wants resource A\n", id);
    pthread_mutex_lock(&A);
    printf("Thread %d locked resource A\n", id);

    sleep(1);

    printf("Thread %d wants resource B\n", id);
    pthread_mutex_lock(&B);
    printf("Thread %d locked resource B\n", id);

    printf("Thread %d is using both resources safely\n", id);
    sleep(1);

    pthread_mutex_unlock(&B);
    printf("Thread %d released resource B\n", id);

    pthread_mutex_unlock(&A);
    printf("Thread %d released resource A\n", id);

    return NULL;
}

int main() {
    pthread_t t1, t2;
    int id1 = 1, id2 = 2;

    pthread_mutex_init(&A, NULL);
    pthread_mutex_init(&B, NULL);

    pthread_create(&t1, NULL, safe_thread, &id1);
    pthread_create(&t2, NULL, safe_thread, &id2);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    pthread_mutex_destroy(&A);
    pthread_mutex_destroy(&B);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab3.c -o lab3 -pthread
./lab3
```

### Hasil Eksekusi

```
Thread 1 wants resource A
Thread 1 locked resource A
Thread 2 wants resource A
Thread 1 wants resource B
Thread 1 locked resource B
Thread 1 is using both resources safely
Thread 1 released resource B
Thread 1 released resource A
Thread 2 locked resource A
Thread 2 wants resource B
Thread 2 locked resource B
Thread 2 is using both resources safely
Thread 2 released resource B
Thread 2 released resource A
```

### Analisis

Program berjalan sampai selesai tanpa deadlock. Kuncinya: kedua thread mengakuisisi resource dalam **urutan yang sama** — selalu A dulu, baru B. Dengan demikian, circular wait tidak mungkin terjadi:

- Thread 1 mengunci A terlebih dahulu
- Thread 2 juga mencoba mengunci A, tapi harus menunggu karena A sudah dipegang Thread 1
- Thread 1 bebas mengunci B tanpa hambatan, menyelesaikan tugasnya, lalu melepaskan keduanya
- Thread 2 baru bisa melanjutkan setelah Thread 1 selesai

Ini adalah teknik **deadlock prevention** yang paling praktis dan sering digunakan di sistem nyata — database system, file locking, dan kernel OS. Trade-off-nya: eksekusi menjadi lebih sequential (Thread 2 harus menunggu Thread 1 selesai sepenuhnya), tapi correctness terjamin.

---

## Lab 4 - Dining Philosophers with Semaphores

### Tujuan
Mengimplementasikan Dining Philosophers Problem — masalah klasik sinkronisasi di mana 5 philosopher bersaing untuk menggunakan 5 fork yang dibagi bersama.

### Script - `lab4.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define N 5

sem_t forks[N];

void* philosopher(void* num) {
    int i = *(int*)num;

    while (1) {
        printf("Philosopher %d thinking\n", i);
        sleep(1);

        sem_wait(&forks[i]);
        sem_wait(&forks[(i + 1) % N]);

        printf("Philosopher %d eating\n", i);
        sleep(1);

        sem_post(&forks[i]);
        sem_post(&forks[(i + 1) % N]);
    }
}

int main() {
    pthread_t p[N];
    int ids[N];

    for (int i = 0; i < N; i++) {
        sem_init(&forks[i], 0, 1);
    }

    for (int i = 0; i < N; i++) {
        ids[i] = i;
        pthread_create(&p[i], NULL, philosopher, &ids[i]);
    }

    for (int i = 0; i < N; i++) {
        pthread_join(p[i], NULL);
    }
}
```

### Cara Menjalankan

```bash
gcc lab4.c -o lab4 -pthread
./lab4
```

(Hentikan dengan `Ctrl+C` karena program berjalan infinite loop)

### Hasil Eksekusi

```
Philosopher 0 thinking
Philosopher 1 thinking
Philosopher 2 thinking
Philosopher 3 thinking
Philosopher 4 thinking
Philosopher 0 eating
Philosopher 2 eating
Philosopher 0 thinking
Philosopher 3 eating
Philosopher 2 thinking
Philosopher 1 eating
...
(program bisa mengalami deadlock setelah beberapa iterasi)
```

### Analisis

Program ini rentan terhadap **deadlock**. Skenario terburuk: semua 5 philosopher mengambil fork kiri secara bersamaan (`sem_wait(&forks[i])`), lalu semua menunggu fork kanan (`sem_wait(&forks[(i+1)%N])`) yang sudah dipegang philosopher sebelahnya — circular wait terbentuk dan semua terhenti.

Dalam beberapa eksekusi, program mungkin berjalan cukup lama sebelum deadlock terjadi karena timing `sleep(1)` memberikan jeda. Tapi secara teori, deadlock pasti bisa terjadi.

Solusi yang mungkin (tidak diimplementasikan di lab ini):
- **Resource ordering**: philosopher terakhir mengambil fork dalam urutan terbalik
- **Limiting concurrency**: membatasi maksimal 4 philosopher yang boleh duduk bersamaan
- **Chandy/Misra solution**: menggunakan token-based approach

Masalah Dining Philosophers adalah contoh klasik yang menunjukkan betapa sulitnya mendesain sistem concurrent yang bebas deadlock dan starvation secara bersamaan.

---

## Lab 5 - Blocking Synchronization using Semaphores (Producer-Consumer)

### Tujuan
Mengimplementasikan bounded-buffer producer-consumer menggunakan semaphore, menunjukkan blocking synchronization yang efisien (thread menunggu tanpa membuang CPU).

### Script - `lab5.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define SIZE 5

int buffer[SIZE];
int count = 0;

sem_t empty, full, mutex;

void* producer(void* arg) {
    while (1) {
        sleep(1);

        sem_wait(&empty);
        sem_wait(&mutex);

        buffer[count++] = 1;
        printf("Produced, count = %d\n", count);

        sem_post(&mutex);
        sem_post(&full);
    }
}

void* consumer(void* arg) {
    while (1) {
        sleep(2);

        sem_wait(&full);
        sem_wait(&mutex);

        buffer[--count] = 0;
        printf("Consumed, count = %d\n", count);

        sem_post(&mutex);
        sem_post(&empty);
    }
}

int main() {
    pthread_t prod, cons;

    sem_init(&empty, 0, SIZE);
    sem_init(&full, 0, 0);
    sem_init(&mutex, 0, 1);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty);
    sem_destroy(&full);
    sem_destroy(&mutex);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab5.c -o lab5 -pthread
./lab5
```

(Hentikan dengan `Ctrl+C`)

### Hasil Eksekusi

```
Produced, count = 1
Produced, count = 2
Consumed, count = 1
Produced, count = 2
Produced, count = 3
Consumed, count = 2
Produced, count = 3
Produced, count = 4
Consumed, count = 3
Produced, count = 4
Produced, count = 5
Consumed, count = 4
Produced, count = 5
Consumed, count = 4
...
```

### Analisis

Producer menghasilkan item setiap 1 detik, consumer mengambil setiap 2 detik — sehingga buffer perlahan terisi. Ketika buffer penuh (count=5), `sem_wait(&empty)` mem-block producer sampai consumer mengambil item. Sebaliknya, ketika buffer kosong, `sem_wait(&full)` mem-block consumer.

Tiga semaphore memiliki peran berbeda:
- **`empty`** (init=SIZE): melacak slot kosong di buffer — producer harus menunggu jika 0
- **`full`** (init=0): melacak item tersedia — consumer harus menunggu jika 0
- **`mutex`** (init=1): melindungi akses ke variabel `count` dan array `buffer`

Keunggulan blocking dibanding busy-waiting: thread yang menunggu tidak mengonsumsi CPU karena OS men-suspend thread tersebut dan membangunkannya hanya ketika semaphore di-signal. Ini jauh lebih efisien untuk sistem dengan banyak thread.

---

## Kesimpulan

| Lab | Topik | Pelajaran Utama |
|-----|-------|-----------------|
| 1 | Thread Concurrency | Eksekusi concurrent bersifat nondeterministik tanpa sinkronisasi |
| 2 | Deadlock Simulation | Circular wait + hold-and-wait menyebabkan program membeku |
| 3 | Deadlock Prevention | Resource ordering mengeliminasi circular wait |
| 4 | Dining Philosophers | Masalah klasik yang menunjukkan sulitnya menghindari deadlock pada shared resources |
| 5 | Producer-Consumer | Blocking synchronization dengan semaphore lebih efisien dari busy-waiting |


