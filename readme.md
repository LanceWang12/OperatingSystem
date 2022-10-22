# 作業系統 & 計算機組織

## 作業系統

### Somthing about Memory
![](https://i.imgur.com/IOlBHBS.png)

#### Memory layout (stack & heap)
1. Text/Code: 存放 program 的組語
2. **Data**: 存放已初始化之 static, global variable
3. Uninitialized data: 未使用之全局區域
4. **Heap**: 
    - 其實為一個 link list，故位址並**不連續** (頻繁 new/delete 會造成 memory leak)
    - 每個節點記錄空閒記憶體區塊的起始位址及大小
    - 存放 malloc, new 等**動態分配**產生之 variable
5. **Stack**:
    - **連續**記憶體
    - 存放 local variable
    - **靜態或動態**分配。靜態分配由編譯器完成 (local variable)，動態分配由 alloca 函式分配，但動態分配的資源由編譯器進行釋放，無需程式設計師實現
    ```c
    #include <alloca.h>
    // 很不安全，不受任何標準保護
    // C_version >= C99 皆可使用 VLA (Variable Length Array)
    // glibc version of stdlib.h include alloca.h by default

    void foo(int size) {
        // alloca version
        char *data = alloca(size);

        // VLA version
        char vla[size];

        // data is automatically freed
    }
    ```
    
#### Virtual memory



### Process & Thread
![](https://i.imgur.com/6ePotLB.png)
1. Program: 程式文本本身
2. Process: 
    - Process 是電腦中已執行 Program 的實體
    - 每一個 Process 是互相獨立的 (各自擁有記憶體資源)
    - Process 本身不是基本執行單位，而是 Thread (執行緒)的容器。
    - Process 需要一些資源才能完成工作，如 CPU、記憶體、檔案以及I/O裝置。
3. Thread:
    - 實際執行單位
    - 一個 Process 可以有多個 Thread
    - 共享 memory (global, static, heap)，但有 private stack

### Synchronization
#### For Process (IPC, Inter Process Communication)
1. pipe
    ```c
    #include<unistd.h>
    int pipe(int fd[2]); 
    // fd[0]: read, fd[1]: write
    // return 0: success
    // return -1: fail
    ```
    ![](https://i.imgur.com/7FDgv5W.jpg)
    - 特性
        1. Pipe 的所有 read 都關掉，則 Pipe 消失
        2. 將 write 關閉後才會送 EOF 給對面 (對面沒收到 EOF 就會被 read block 住)
        3. 寫入 read 關閉的 Pipe，會得到 SIGPIPE 並終止程式
        4. size: 16pages
    - 兩個 Process 使用一個 Pipe 單向傳輸
    - 兩個 Process 使用兩個 Pipe 雙向傳輸
2. message queue
    - 特性
        1. 寫入 queue 不用管對面行為，不像 pipe 若寫入時沒有 process 讀就會終止
        2. Pipe close 之後，存放資料即丟失，但 msg que 除非所有程式結束或呼叫 unlink 不然會一直存在
    ```c
    #include <mqueue.h>
    mqd_t mq_open(const char *name, int oflag, mode_t mode, mq_attr attr);
    // 打開 message queue

    mqd_t mq_close(mqd_t mqdes);
    // Process 中關閉 message queue (但 meg que 仍存在於記憶體)
    
    mqd_t mq_unlink(const char *name);
    // 真正刪除 msg que
    ```
3. Shared memory
    - 特性
        - 速度是所有 IPC method 裡面最快的，因為是直接存取記憶體
        - 與 Pipe 相比都是存取一段連續記憶體，Pipe buffer size 較小，且半雙工，不易發生 race condition，Share Memory 反之。
    - System V
    ```c
    #include <sys/ipc.h>
    #include <sys/shm.h>

    int shmget(key_t key, size_t size, int shmflg);
    void * shmat(int shmid, const void *shmaddr, int shmflg);
    int shmdt(const void *shmaddr)
    ```
    
    - POSIX
        - 特性：將 tmpfs 中的記憶體空間映射至 Process 本地記憶體，進而達到 IPC
        - tmpfs (temporary file system): 在 linux 裡通常位於: /dev/shm，該目錄位於 ram 上，容量上限通常為電腦記憶體的一半，藉由存取記憶體上的文件進行 process 間的溝通
    ```c
    int shm_open(const char *name, int oflag, mode_t mode);
    // 打開(or 創建) /dev/shm 內的文件
    // name: 文件名(/dev/shm/yourName)
    // oflag: 開檔模式(O_CREAT, O_RDWR, ...)
    // mode: 文件模式(chmod 那個)
    // return 文件描述符回來
    
    int ftruncate(int fd, off_t length);
    // 重置檔案大小
    // fd: 修改目標文件的檔案描述符
    // length: 重置的大小
    // if success: return 0; else return -1;
    
    void *mmap(void *addr, size_t length, int prot, int flags,int fd, off_t offset);
    // 將檔案映射至 Process 本地的記憶體
    // addr: 欲映射之位址(同常為 NULL，由 os 分配)
    // length: 映射大小
    // prot: 映射操作權限
    // flags: MAP_SHARED -> 進程都看得到內容
    //        MAP_PRIVATE -> 只有使用的進程可以看到內容
    // fd: 目標檔案
    // offset: 文件偏移量
    
    int munmap(void *addr, size_t length);
    // 取消映射
    
    int shm_unlink(const char *name);
    // 刪除檔案
    ```

4. UDS(Unix Domained Socket)

#### For Thread
1. Synchronization method
    - Spin Lock (Busy waiting)
        ```c
        while (flag != my_rank);
        sum += my_sum;
        flag = (flag+1) % thread_count;
        ```
        - 優點：當 critical section 很短，不用 context switch 會更快
        - 缺點：當 cpu core num > thread num，會發生不必要等待開銷
    - Mutex (Sleeping)
        ```c
        pthread_mutex_t mutex_p; // global var

        int pthread mutex init(
            pthread_mutex_t mutex_p, // out
            const pthread_mutexattr_t attr_p // in
        );

        int pthread_mutex_lock(pthread_mutex_t *mutex); // out

        int pthread_mutex_unlock(pthread_mutex_t *mutex); // out

        int pthread_mutex_destroy(pthread_mutex_t *mutex): // in
        ```
        - 缺點：只有進入房間的 thread 持有鑰匙，萬一進入 critical section 的 thread dead 了，會 block 住整支 process
    - Semaphore (Sleeping)
        ```c
        sem_t sem; // global var

        int sem_init(
            sem_t *sem, // out, 要初始化的 semaphore
            int pshared, // in, 要在多少隻 process 之間 share
            unsigned int value // in, 房間容許人數
        );

        int sem_post(sem_t sem); // in, out, 走出房間, 房間容量 +1

        int sem_wait(sem_t sem); // in, out, 想進入房間, 房間容量 -1

        int sem_destroy(sem_t sem); // in, out
        ```
        - 優點：即使房內的 thread dead, 外部仍可以透過控制房間容量進去

2. Some classical problem
    - Consumer-Producer
        ![](https://i.imgur.com/Mm9K8Gf.png)
        ```c
        #include <semaphore.h>
        int thread_count; char** messages; sem_t* semaphores;
        int main(int argc, char* argv[]) {
            long thread;
            pthread_t* thread_handles;
            ... // processing argv[]
            thread_handles = (pthread_t*) malloc (thread_count*sizeof(pthread_t)); messages = (char**) malloc(thread_count*sizeof(char*));
            semaphores = malloc(thread_count*sizeof(sem_t));
            for (thread = 0; thread < thread_count; thread++) {
                messages[thread] = NULL;
                sem_init(&semaphores[thread], 0, 0);
            }
            for (thread = 0; thread < thread_count; thread++) 
                pthread_create(&thread_handles[thread], (pthread_attr_t*) NULL,
            Send_msg, (void*) thread);
            for (thread = 0; thread < thread_count; thread++) 
                pthread_join(thread_handles[thread], NULL);
            for (thread = 0; thread < thread_count; thread++) { 
                free(messages[thread]);
                sem_destroy(&semaphores[thread]);
            }
            free(messages);
            free(semaphores);
            free(thread_handles);
            return 0;
        } /*main*/
        ```
    - Barrier
        ![](https://i.imgur.com/G9rEGZY.png)
    - Philosophers Problem
    - Sleeping Barber Problem


## 計算機組織
