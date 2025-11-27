# Linux-IPC-Semaphores
Ex05-Linux IPC-Semaphores

# AIM:
To Write a C program that implements a producer-consumer system with two processes using Semaphores.

# DESIGN STEPS:

### Step 1:

Navigate to any Linux environment installed on the system or installed inside a virtual environment like virtual box/vmware or online linux JSLinux (https://bellard.org/jslinux/vm.html?url=alpine-x86.cfg&mem=192) or docker.

### Step 2:

Write the C Program using Linux Process API - Sempahores

### Step 3:

Execute the C Program for the desired output. 

# PROGRAM:
```
Developed By : kamalesh v
Reg No : 212222240042
```

## Write a C program that implements a producer-consumer system with two processes using Semaphores.

```
/**
 * producer_consumer.c
 *
 * This program implements the classic Producer-Consumer problem using
 * two separate processes (parent = producer, child = consumer).
 *
 * It uses:
 * - POSIX Shared Memory (shm_open, mmap, ftruncate) for the shared buffer.
 * - POSIX Named Semaphores (sem_open, sem_wait, sem_post) for synchronization.
 *
 * How to compile:
 * gcc producer_consumer.c -o producer_consumer -lpthread -lrt
 *
 * (You need -lpthread for semaphores and -lrt for shared memory)
 *
 * How to run:
 * ./producer_consumer
 */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>     // For fork(), sleep(), ftruncate()
#include <semaphore.h>  // For POSIX semaphores (sem_t)
#include <fcntl.h>      // For O_CREAT, O_RDWR
#include <sys/mman.h>   // For mmap(), shm_open()
#include <sys/stat.h>   // For mode constants (S_IRUSR)
#include <sys/wait.h>   // For wait()

// --- Constants ---
#define BUFFER_SIZE 10  // Size of the shared buffer
#define NUM_ITEMS 30    // Number of items to produce/consume

// Names for shared memory and semaphores
const char* SHM_NAME = "/pc_shared_memory";
const char* SEM_MUTEX = "/pc_sem_mutex"; // For mutual exclusion
const char* SEM_EMPTY = "/pc_sem_empty"; // Counts empty slots
const char* SEM_FULL = "/pc_sem_full";   // Counts full slots

// --- Shared Data Structure ---
// This structure will be placed in shared memory.
struct shared_data {
    int buffer[BUFFER_SIZE];
    int in;  // Index for producer to write
    int out; // Index for consumer to read
};

// --- Function Prototypes ---
void producer(sem_t* mutex, sem_t* empty, sem_t* full, struct shared_data* data);
void consumer(sem_t* mutex, sem_t* empty, sem_t* full, struct shared_data* data);
void cleanup_resources();

int main() {
    int shm_fd;
    struct shared_data* shared_mem_ptr;
    pid_t pid;

    // --- Clean up old resources (in case of a crash) ---
    // This is good practice.
    cleanup_resources();

    // --- 1. Create Shared Memory ---
    // shm_open creates or opens a POSIX shared memory object.
    shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (shm_fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    // Set the size of the shared memory object
    ftruncate(shm_fd, sizeof(struct shared_data));

    // Map the shared memory object into the process's address space
    shared_mem_ptr = (struct shared_data*) mmap(
        0, 
        sizeof(struct shared_data),
        PROT_READ | PROT_WRITE, 
        MAP_SHARED, 
        shm_fd, 
        0
    );

    if (shared_mem_ptr == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    // Initialize shared memory indices
    shared_mem_ptr->in = 0;
    shared_mem_ptr->out = 0;

    // --- 2. Create Semaphores ---
    // sem_open creates or opens a named POSIX semaphore.
    // O_CREAT: Create if it doesn't exist
    // 0666: Permissions
    // Initial values: mutex=1, empty=BUFFER_SIZE, full=0

    sem_t* mutex = sem_open(SEM_MUTEX, O_CREAT, 0666, 1);
    sem_t* empty = sem_open(SEM_EMPTY, O_CREAT, 0666, BUFFER_SIZE);
    sem_t* full = sem_open(SEM_FULL, O_CREAT, 0666, 0);

    if (mutex == SEM_FAILED || empty == SEM_FAILED || full == SEM_FAILED) {
        perror("sem_open");
        exit(EXIT_FAILURE);
    }

    // --- 3. Fork ---
    pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // --- Child Process (Consumer) ---
        printf("Consumer process created.\n");
        consumer(mutex, empty, full, shared_mem_ptr);
        exit(EXIT_SUCCESS);
    } else {
        // --- Parent Process (Producer) ---
        printf("Producer process created.\n");
        producer(mutex, empty, full, shared_mem_ptr);

        // Wait for the child (consumer) process to finish
        wait(NULL);
        printf("Consumer finished. Parent cleaning up.\n");

        // --- 4. Cleanup (in Parent) ---
        // Close semaphores
        sem_close(mutex);
        sem_close(empty);
        sem_close(full);

        // Unmap shared memory
        munmap(shared_mem_ptr, sizeof(struct shared_data));
        close(shm_fd);

        // Unlink semaphores and shared memory (removes them from the system)
        cleanup_resources();
    }

    return 0;
}

/**
 * Producer Function
 */
void producer(sem_t* mutex, sem_t* empty, sem_t* full, struct shared_data* data) {
    for (int i = 0; i < NUM_ITEMS; i++) {
        // Produce an item (just a number)
        int item = i * 10; 
        
        // Wait for an empty slot
        sem_wait(empty);
        
        // Lock the buffer
        sem_wait(mutex);
        
        // --- Critical Section ---
        data->buffer[data->in] = item;
        printf("Produced: %d \t(at index %d)\n", item, data->in);
        data->in = (data->in + 1) % BUFFER_SIZE;
        // --- End Critical Section ---

        // Unlock the buffer
        sem_post(mutex);

        // Signal that a slot is now full
        sem_post(full);

        // Sleep to make the output observable
        sleep(1); 
    }
}

/**
 * Consumer Function
 */
void consumer(sem_t* mutex, sem_t* empty, sem_t* full, struct shared_data* data) {
    for (int i = 0; i < NUM_ITEMS; i++) {
        // Wait for a full slot
        sem_wait(full);

        // Lock the buffer
        sem_wait(mutex);

        // --- Critical Section ---
        int item = data->buffer[data->out];
        printf("\tConsumed: %d \t(from index %d)\n", item, data->out);
        data->out = (data->out + 1) % BUFFER_SIZE;
        // --- End Critical Section ---
        
        // Unlock the buffer
        sem_post(mutex);

        // Signal that a slot is now empty
        sem_post(empty);

        // Sleep a bit longer than producer to show buffer filling up
        sleep(2); 
    }
}

/**
 * cleanup_resources
 * Unlinks semaphores and shared memory from the system.
 * This is important to call, especially if the program crashed,
 * to prevent resource leaks.
 */
void cleanup_resources() {
    sem_unlink(SEM_MUTEX);
    sem_unlink(SEM_EMPTY);
    sem_unlink(SEM_FULL);
    shm_unlink(SHM_NAME);
}
```


## OUTPUT
$ ./sem.o 

<img width="718" height="402" alt="1" src="https://github.com/user-attachments/assets/465f59ee-831b-43d4-833f-bb77b6bc57f7" />




$ ipcs

<img width="1185" height="319" alt="Screenshot from 2025-10-27 09-38-01" src="https://github.com/user-attachments/assets/de8c4c2f-1bee-44ed-8c3c-d45a643a1981" />





# RESULT:
The program is executed successfully.
