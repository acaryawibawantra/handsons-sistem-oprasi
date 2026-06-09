# Hands-On 5: Mutex and Threads

**Nama:** I Dewa Nyoman Acarya Wibawantra
**NRP:** 5025251222
**Mata Kuliah:** Sistem Operasi

---

## Lab 1 - Creating Multiple Threads and Observing Concurrent Execution

### Tujuan
Memahami konsep thread sebagai unit eksekusi independen di dalam satu proses, serta mengamati bahwa beberapa thread dapat berjalan secara bersamaan (concurrent) dengan urutan output yang tidak selalu sama.

### Script - `lab1_basic_threads.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* worker(void* arg) {
    int id = *(int*)arg;

    for (int i = 1; i <= 5; i++) {
        printf("Thread %d is running: step %d\n", id, i);
        usleep(100000);
    }

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("All threads have finished.\n");
    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab1_basic_threads.c -o lab1_basic_threads -pthread
./lab1_basic_threads
```

### Hasil Eksekusi

```
Thread 1 is running: step 1
Thread 3 is running: step 1
Thread 2 is running: step 1
Thread 1 is running: step 2
Thread 2 is running: step 2
Thread 3 is running: step 2
Thread 1 is running: step 3
Thread 3 is running: step 3
Thread 2 is running: step 3
Thread 1 is running: step 4
Thread 2 is running: step 4
Thread 3 is running: step 4
Thread 1 is running: step 5
Thread 3 is running: step 5
Thread 2 is running: step 5
All threads have finished.
```

### Analisis

Output dari ketiga thread saling bercampur (interleaved) karena OS menjadwalkan thread secara independen. Urutan bisa berbeda setiap kali dijalankan — ini menunjukkan sifat nondeterministik dari concurrency. Semua thread berada dalam satu proses yang sama, berbagi address space, tapi masing-masing punya alur eksekusi sendiri. `usleep(100000)` memberikan jeda 100ms agar interleaving lebih terlihat jelas di terminal.

---

## Lab 2 - Race Condition on Shared Data

### Tujuan
Mendemonstrasikan race condition ketika dua thread meng-increment variabel global yang sama tanpa sinkronisasi.

### Script - `lab2_race_condition.c`

```c
#include <stdio.h>
#include <pthread.h>

#define ITERATIONS 1000000

int counter = 0;

void* increment_counter(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        counter++;
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment_counter, NULL);
    pthread_create(&t2, NULL, increment_counter, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Expected counter value: %d\n", ITERATIONS * 2);
    printf("Actual counter value  : %d\n", counter);
    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab2_race_condition.c -o lab2_race_condition -pthread
./lab2_race_condition
```

### Hasil Eksekusi (3x percobaan)

```
$ ./lab2_race_condition
Expected counter value: 2000000
Actual counter value  : 1487263

$ ./lab2_race_condition
Expected counter value: 2000000
Actual counter value  : 1352891

$ ./lab2_race_condition
Expected counter value: 2000000
Actual counter value  : 1591044
```

### Analisis

Hasil selalu di bawah 2000000 dan berubah-ubah setiap eksekusi. Ini karena `counter++` bukan operasi atomik — secara internal terdiri dari read, add, dan write. Ketika dua thread membaca nilai yang sama sebelum salah satu menulis, satu increment hilang (lost update). Dengan 1 juta iterasi per thread, peluang overlap sangat tinggi. Ini membuktikan bahwa shared memory tanpa proteksi menghasilkan data yang tidak konsisten.

---

## Lab 3 - Solving the Race Condition with Mutual Exclusion

### Tujuan
Memperbaiki race condition dari Lab 2 menggunakan mutex untuk melindungi critical section.

### Script - `lab3_mutex_counter.c`

```c
#include <stdio.h>
#include <pthread.h>

#define ITERATIONS 1000000

int counter = 0;
pthread_mutex_t lock;

void* increment_counter(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_mutex_init(&lock, NULL);

    pthread_create(&t1, NULL, increment_counter, NULL);
    pthread_create(&t2, NULL, increment_counter, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    pthread_mutex_destroy(&lock);

    printf("Expected counter value: %d\n", ITERATIONS * 2);
    printf("Actual counter value  : %d\n", counter);
    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab3_mutex_counter.c -o lab3_mutex_counter -pthread
./lab3_mutex_counter
```

### Hasil Eksekusi

```
Expected counter value: 2000000
Actual counter value  : 2000000
```

### Analisis

Dengan mutex, hasilnya selalu tepat 2000000. `pthread_mutex_lock` memastikan hanya satu thread yang bisa mengeksekusi `counter++` dalam satu waktu — thread lain harus menunggu sampai lock dilepas. Ini menunjukkan prinsip mutual exclusion: critical section terlindungi dari akses bersamaan. Perlu dicatat bahwa sinkronisasi menambah overhead — program berjalan sedikit lebih lambat karena thread harus bergantian di critical section.

---

## Lab 4 - Producer-Consumer Using Semaphore Synchronization

### Tujuan
Mengimplementasikan pola producer-consumer menggunakan semaphore untuk koordinasi urutan eksekusi dan mutex untuk proteksi buffer.

### Script - `lab4_producer_consumer.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define ITEMS 10

int buffer;
sem_t empty;
sem_t full;
pthread_mutex_t lock;

void* producer(void* arg) {
    for (int item = 1; item <= ITEMS; item++) {
        sem_wait(&empty);

        pthread_mutex_lock(&lock);
        buffer = item;
        printf("Producer produced item %d\n", item);
        pthread_mutex_unlock(&lock);

        sem_post(&full);
        sleep(1);
    }

    return NULL;
}

void* consumer(void* arg) {
    for (int i = 1; i <= ITEMS; i++) {
        sem_wait(&full);

        pthread_mutex_lock(&lock);
        int item = buffer;
        printf("Consumer consumed item %d\n", item);
        pthread_mutex_unlock(&lock);

        sem_post(&empty);
        sleep(1);
    }

    return NULL;
}

int main() {
    pthread_t prod, cons;

    sem_init(&empty, 0, 1);
    sem_init(&full, 0, 0);
    pthread_mutex_init(&lock, NULL);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&lock);
    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab4_producer_consumer.c -o lab4_producer_consumer -pthread
./lab4_producer_consumer
```

### Hasil Eksekusi

```
Producer produced item 1
Consumer consumed item 1
Producer produced item 2
Consumer consumed item 2
Producer produced item 3
Consumer consumed item 3
Producer produced item 4
Consumer consumed item 4
Producer produced item 5
Consumer consumed item 5
Producer produced item 6
Consumer consumed item 6
Producer produced item 7
Consumer consumed item 7
Producer produced item 8
Consumer consumed item 8
Producer produced item 9
Consumer consumed item 9
Producer produced item 10
Consumer consumed item 10
```

### Analisis

Setiap item diproduksi lalu dikonsumsi secara bergantian — tidak ada data yang hilang atau di-overwrite. Semaphore `empty` (init=1) memastikan producer hanya bisa produce saat buffer kosong, sedangkan `full` (init=0) memastikan consumer hanya bisa consume saat buffer terisi. Mutex melindungi akses ke variabel `buffer`. Kombinasi semaphore + mutex ini menunjukkan dua peran sinkronisasi yang berbeda: semaphore mengatur ordering/blocking, sedangkan mutex mengatur mutual exclusion.

---

## Lab 5 - Blocking and Unblocking Threads with Condition Variables

### Tujuan
Mendemonstrasikan mekanisme blocking yang efisien menggunakan condition variable, di mana thread menunggu tanpa membuang CPU (berbeda dari busy waiting).

### Script - `lab5_blocking_condition.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int data_ready = 0;
int shared_data = 0;

pthread_mutex_t lock;
pthread_cond_t condition;

void* worker(void* arg) {
    pthread_mutex_lock(&lock);

    while (!data_ready) {
        printf("Worker: data is not ready, blocking now...\n");
        pthread_cond_wait(&condition, &lock);
    }

    printf("Worker: awakened, received shared data = %d\n", shared_data);

    pthread_mutex_unlock(&lock);
    return NULL;
}

void* controller(void* arg) {
    sleep(3);

    pthread_mutex_lock(&lock);

    shared_data = 42;
    data_ready = 1;
    printf("Controller: data prepared, signaling worker...\n");

    pthread_cond_signal(&condition);

    pthread_mutex_unlock(&lock);
    return NULL;
}

int main() {
    pthread_t t_worker, t_controller;

    pthread_mutex_init(&lock, NULL);
    pthread_cond_init(&condition, NULL);

    pthread_create(&t_worker, NULL, worker, NULL);
    pthread_create(&t_controller, NULL, controller, NULL);

    pthread_join(t_worker, NULL);
    pthread_join(t_controller, NULL);

    pthread_cond_destroy(&condition);
    pthread_mutex_destroy(&lock);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab5_blocking_condition.c -o lab5_blocking_condition -pthread
./lab5_blocking_condition
```

### Hasil Eksekusi

```
Worker: data is not ready, blocking now...
Controller: data prepared, signaling worker...
Worker: awakened, received shared data = 42
```

### Analisis

Worker langsung masuk ke kondisi blocking saat `data_ready` masih 0. Selama 3 detik, worker tidak melakukan apa-apa (tidak membuang CPU). Setelah controller menyiapkan data dan memanggil `pthread_cond_signal`, worker terbangun dan membaca `shared_data = 42`. Penggunaan `while (!data_ready)` (bukan `if`) penting untuk menghindari spurious wakeup — kondisi dimana thread terbangun tanpa signal yang valid. Pendekatan ini jauh lebih efisien dibanding busy-waiting karena thread yang di-block tidak mengonsumsi waktu CPU.

---

## Lab 6 - Implementing a Critical Section with Threads (Bank Account)

### Tujuan
Mengidentifikasi dan mengimplementasikan critical section pada kasus nyata: 5 thread melakukan deposit ke satu rekening bank yang sama.

### Script - `critical_section_bank.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

#define THREADS 5
#define DEPOSITS_PER_THREAD 3
#define DEPOSIT_AMOUNT 100

int balance = 0;
pthread_mutex_t balance_lock;

void* deposit_money(void* arg) {
    int thread_id = *(int*)arg;

    for (int i = 1; i <= DEPOSITS_PER_THREAD; i++) {
        pthread_mutex_lock(&balance_lock);

        int old_balance = balance;
        printf("Thread %d reads balance: %d\n", thread_id, old_balance);

        sleep(1);

        balance = old_balance + DEPOSIT_AMOUNT;
        printf("Thread %d updates balance to: %d\n", thread_id, balance);

        pthread_mutex_unlock(&balance_lock);

        sleep(1);
    }

    return NULL;
}

int main() {
    pthread_t threads[THREADS];
    int ids[THREADS];

    pthread_mutex_init(&balance_lock, NULL);

    for (int i = 0; i < THREADS; i++) {
        ids[i] = i + 1;
        pthread_create(&threads[i], NULL, deposit_money, &ids[i]);
    }

    for (int i = 0; i < THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_mutex_destroy(&balance_lock);

    printf("\nFinal balance: %d\n", balance);
    printf("Expected balance: %d\n",
           THREADS * DEPOSITS_PER_THREAD * DEPOSIT_AMOUNT);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc critical_section_bank.c -o critical_section_bank -pthread
./critical_section_bank
```

### Hasil Eksekusi (ringkasan)

```
Thread 1 reads balance: 0
Thread 1 updates balance to: 100
Thread 3 reads balance: 100
Thread 3 updates balance to: 200
Thread 2 reads balance: 200
Thread 2 updates balance to: 300
...
Thread 5 reads balance: 1400
Thread 5 updates balance to: 1500

Final balance: 1500
Expected balance: 1500
```

### Analisis

Hasil akhir selalu 1500 (5 thread × 3 deposit × 100). Mutex memastikan hanya satu thread yang bisa membaca dan mengupdate balance pada satu waktu. `sleep(1)` di dalam critical section sengaja ditambahkan agar window of vulnerability lebih terlihat — tanpa mutex, thread lain bisa membaca `old_balance` yang sudah stale selama jeda ini. Ini menggambarkan kasus dunia nyata seperti dua transaksi ATM yang mengakses saldo bersamaan.

---

## Lab 7 - Observing a Race Condition

### Tujuan
Mengamati efek race condition pada counter yang di-increment oleh dua thread tanpa proteksi.

### Script - `lab1_race.c`

```c
#include <stdio.h>
#include <pthread.h>

#define ITER 1000000

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < ITER; i++) {
        counter++;
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Expected: %d\n", ITER * 2);
    printf("Actual  : %d\n", counter);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab1_race.c -o lab1_race -pthread
./lab1_race
```

### Hasil Eksekusi (3x percobaan)

```
$ ./lab1_race
Expected: 2000000
Actual  : 1283746

$ ./lab1_race
Expected: 2000000
Actual  : 1456231

$ ./lab1_race
Expected: 2000000
Actual  : 1178503
```

### Analisis

Sama dengan Lab 2 — hasilnya tidak pernah mencapai 2000000 karena `counter++` bukan operasi atomik. Lab ini memperkuat pemahaman bahwa race condition bukan masalah sintaks, melainkan masalah timing. Program terlihat benar dari perspektif satu thread, tapi ketika dijalankan secara concurrent, hasilnya menjadi unpredictable. Perbedaan nilai antar eksekusi menunjukkan bahwa race condition sangat bergantung pada scheduling OS.

---

## Lab 8 - Implementing Mutual Exclusion with Mutex

### Tujuan
Mengeliminasi race condition dari Lab 7 dengan menambahkan mutex lock.

### Script - `lab2_mutex.c`

```c
#include <stdio.h>
#include <pthread.h>

#define ITER 1000000

int counter = 0;
pthread_mutex_t lock;

void* increment(void* arg) {
    for (int i = 0; i < ITER; i++) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_mutex_init(&lock, NULL);

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    pthread_mutex_destroy(&lock);

    printf("Expected: %d\n", ITER * 2);
    printf("Actual  : %d\n", counter);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab2_mutex.c -o lab2_mutex -pthread
./lab2_mutex
```

### Hasil Eksekusi

```
Expected: 2000000
Actual  : 2000000
```

### Analisis

Dengan mutex, hasil selalu konsisten 2000000. Ini mengonfirmasi bahwa mutual exclusion menyelesaikan race condition. Thread yang gagal mendapat lock akan di-block oleh OS sampai lock tersedia — ini lebih efisien daripada busy waiting. Trade-off-nya terlihat pada waktu eksekusi yang lebih lambat karena critical section hanya bisa diakses satu thread dalam satu waktu.

---

## Lab 9 - Semaphore-Based Mutual Exclusion

### Tujuan
Mengimplementasikan mutual exclusion menggunakan semaphore sebagai alternatif dari mutex.

### Script - `lab3_semaphore.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>

#define ITER 1000000

int counter = 0;
sem_t sem;

void* increment(void* arg) {
    for (int i = 0; i < ITER; i++) {
        sem_wait(&sem);
        counter++;
        sem_post(&sem);
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;

    sem_init(&sem, 0, 1);

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    sem_destroy(&sem);

    printf("Expected: %d\n", ITER * 2);
    printf("Actual  : %d\n", counter);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab3_semaphore.c -o lab3_semaphore -pthread
./lab3_semaphore
```

### Hasil Eksekusi

```
Expected: 2000000
Actual  : 2000000
```

### Analisis

Binary semaphore (init=1) berperilaku seperti mutex — `sem_wait` menurunkan nilai semaphore dan mem-block jika nilainya 0, sedangkan `sem_post` menaikkan nilai dan membangunkan thread yang menunggu. Perbedaan utama dengan mutex: semaphore lebih general karena bisa diinisialisasi dengan nilai > 1 untuk mengizinkan beberapa thread masuk bersamaan, sedangkan mutex selalu eksklusif untuk satu thread. Dalam konteks ini keduanya menghasilkan hasil yang identik.

---

## Lab 10 - Producer-Consumer Synchronization

### Tujuan
Mengkoordinasikan dua thread dalam pola producer-consumer menggunakan semaphore dan mutex.

### Script - `lab4_producer_consumer.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

int buffer;
sem_t empty, full;
pthread_mutex_t lock;

void* producer(void* arg) {
    for (int i = 1; i <= 5; i++) {
        sem_wait(&empty);

        pthread_mutex_lock(&lock);
        buffer = i;
        printf("Produced: %d\n", i);
        pthread_mutex_unlock(&lock);

        sem_post(&full);
        sleep(1);
    }

    return NULL;
}

void* consumer(void* arg) {
    for (int i = 1; i <= 5; i++) {
        sem_wait(&full);

        pthread_mutex_lock(&lock);
        printf("Consumed: %d\n", buffer);
        pthread_mutex_unlock(&lock);

        sem_post(&empty);
        sleep(1);
    }

    return NULL;
}

int main() {
    pthread_t p, c;

    sem_init(&empty, 0, 1);
    sem_init(&full, 0, 0);
    pthread_mutex_init(&lock, NULL);

    pthread_create(&p, NULL, producer, NULL);
    pthread_create(&c, NULL, consumer, NULL);

    pthread_join(p, NULL);
    pthread_join(c, NULL);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab4_producer_consumer.c -o lab4_producer_consumer -pthread
./lab4_producer_consumer
```

### Hasil Eksekusi

```
Produced: 1
Consumed: 1
Produced: 2
Consumed: 2
Produced: 3
Consumed: 3
Produced: 4
Consumed: 4
Produced: 5
Consumed: 5
```

### Analisis

Pola produksi dan konsumsi berjalan secara bergantian dan terurut — setiap item yang diproduksi langsung dikonsumsi sebelum item baru dibuat. Dengan buffer berukuran 1, semaphore `empty` dan `full` memastikan strict alternation. Consumer tidak bisa mengambil data sebelum producer mengisinya (dijaga oleh `sem_wait(&full)`), dan producer tidak bisa menimpa buffer sebelum consumer mengambilnya (dijaga oleh `sem_wait(&empty)`). Pola ini fondasi dari banyak sistem nyata: message queue, print spooler, dan data pipeline.

---

## Lab 11 - Readers-Writers Problem

### Tujuan
Mengimplementasikan skenario di mana beberapa reader bisa mengakses data secara bersamaan, tapi writer memerlukan akses eksklusif.

### Script - `lab5_readers_writers.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int read_count = 0;
pthread_mutex_t mutex;
pthread_mutex_t write_lock;

void* reader(void* arg) {
    pthread_mutex_lock(&mutex);
    read_count++;

    if (read_count == 1) {
        pthread_mutex_lock(&write_lock);
    }

    pthread_mutex_unlock(&mutex);

    printf("Reader reading\n");
    sleep(1);

    pthread_mutex_lock(&mutex);
    read_count--;

    if (read_count == 0) {
        pthread_mutex_unlock(&write_lock);
    }

    pthread_mutex_unlock(&mutex);

    return NULL;
}

void* writer(void* arg) {
    pthread_mutex_lock(&write_lock);

    printf("Writer writing\n");
    sleep(2);

    pthread_mutex_unlock(&write_lock);

    return NULL;
}

int main() {
    pthread_t r1, r2, w1;

    pthread_mutex_init(&mutex, NULL);
    pthread_mutex_init(&write_lock, NULL);

    pthread_create(&r1, NULL, reader, NULL);
    pthread_create(&r2, NULL, reader, NULL);
    pthread_create(&w1, NULL, writer, NULL);

    pthread_join(r1, NULL);
    pthread_join(r2, NULL);
    pthread_join(w1, NULL);

    return 0;
}
```

### Cara Menjalankan

```bash
gcc lab5_readers_writers.c -o lab5_readers_writers -pthread
./lab5_readers_writers
```

### Hasil Eksekusi

```
Reader reading
Reader reading
Writer writing
```

### Analisis

Kedua reader bisa membaca secara bersamaan (muncul hampir serentak), tapi writer harus menunggu sampai semua reader selesai. Mekanismenya: reader pertama yang masuk mengambil `write_lock` sehingga writer ter-block, reader-reader berikutnya cukup menaikkan `read_count` tanpa perlu mengambil `write_lock` lagi. Ketika reader terakhir keluar (`read_count == 0`), `write_lock` dilepas dan writer bisa masuk. Ini adalah implementasi "readers-preference" — reader diprioritaskan, yang bisa menyebabkan writer starvation jika reader terus datang tanpa henti.

---

## Kesimpulan

| Lab | Topik | Pelajaran Utama |
|-----|-------|-----------------|
| 1 | Multiple Threads | Thread berjalan concurrent, urutan output nondeterministik |
| 2 | Race Condition | Shared variable tanpa proteksi menghasilkan data salah |
| 3 | Mutex | Mutual exclusion mengembalikan correctness |
| 4 | Producer-Consumer (Semaphore) | Semaphore mengatur ordering, mutex mengatur akses |
| 5 | Condition Variable | Blocking efisien tanpa busy waiting |
| 6 | Critical Section (Bank) | Identifikasi dan proteksi critical section pada kasus nyata |
| 7 | Race Condition (ulang) | Memperkuat pemahaman tentang non-atomicity |
| 8 | Mutex (ulang) | Konfirmasi bahwa mutex menyelesaikan race condition |
| 9 | Semaphore Mutex | Binary semaphore sebagai alternatif mutex |
| 10 | Producer-Consumer | Koordinasi strict alternation dengan buffer tunggal |
| 11 | Readers-Writers | Multiple readers concurrent, writer eksklusif |

Intinya: thread memungkinkan concurrency yang efisien, tapi shared resource membutuhkan sinkronisasi (mutex, semaphore, condition variable) untuk menjaga correctness. Desain yang baik meminimalkan ukuran critical section agar concurrency tetap optimal.
