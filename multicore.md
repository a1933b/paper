[TOC]
# Multi core
## 開機狀態
atomic 開機，用timing 去跑checkpoints 可以開
![image](https://hackmd.io/_uploads/rkP-Vx0v6.png)

script
![image](https://hackmd.io/_uploads/rkBJNgCvp.png)

timing 下的cpu和 stack狀態，看起來是有成功

![image](https://hackmd.io/_uploads/SJAqmJ5vp.png)
![image](https://hackmd.io/_uploads/BJ4NOKRDp.png =75%x)

## 跑multithread程式1

最近我有嘗試用multithread的filebench seqwrite在root跑，在atomic是可以跑完的，在timing的話會卡在開檔的步驟，然後filebench會顯示下面的錯誤


```clike
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

// 子執行緒函數
void* child(void* data) {
  char *str = (char*) data; // 取得輸入資料
  for(int i = 0;i < 3;++i) {
    printf("%s\n", str); // 每秒輸出文字
    sleep(1);
  }
  pthread_exit(NULL); // 離開子執行緒
}

// 主程式
int main() {
  pthread_t t; // 宣告 pthread 變數
  pthread_create(&t, NULL, child, "Child"); // 建立子執行緒

  // 主執行緒工作
  for(int i = 0;i < 3;++i) {
    printf("Master\n"); // 每秒輸出文字
    sleep(1);
  }

  pthread_join(t, NULL); // 等待子執行緒執行完成
  return 0;
}
```

* make
    * gcc hello.c -lpthread -o hello


./hello

* qemu可以跑
    * 結果:
     ```
        Master
        Child
        Master
        Child
        Master
        Child
    ```
* in gem5 atomic mode
    * ok
    * ![image](https://hackmd.io/_uploads/Hkm8SsfOa.png)
* in gem5 timing mode
    * ![image](https://hackmd.io/_uploads/HJh-immdT.png)

## filebench multithread
* **filebench 使用pthread**
* 應該以pthread 角度去查原因?

### script
```bash
set $dir=/root
set $iosize=1k
set $bytes=4 # 1 ~ 1024 (m)

debug 0

define file name=bigfile,path=$dir,size=0,prealloc

define process name=filewriter,instances=1
{
  thread name=filewriterthread,memsize=10m,instances=2
  {
      flowop finishoncount name=finish,value=1,target=write-file
    flowop write name=write-file,filename=bigfile,iosize=$iosize
   
  }
}

create files

run
```

### code
* parser會通過script的cmd去啟動對應的process執行
```c=
parser_run(cmd_t *cmd)
{
	int runtime;
	int timeslept;

	runtime = cmd->cmd_qty;
	parser_fileset_create(cmd);
	parser_proc_create(cmd);
	/* check for startup errors */
	if (filebench_shm->shm_f_abort)
		return;
	filebench_log(LOG_INFO, "Running...");
	stats_clear();
	timeslept = parser_pause(runtime);
	filebench_log(LOG_INFO, "Run took %d seconds...", timeslept);
	parser_statssnap(cmd);
	parser_proc_shutdown(cmd);
	parser_filebench_shutdown((cmd_t *)0);
}
```
first time 卡住
![image](https://hackmd.io/_uploads/S1kdirHF6.png)

### atomic
 
![image](https://hackmd.io/_uploads/rJLFsCEYa.png)

### timing
* kernel paging request
![image](https://hackmd.io/_uploads/BkEvZpsnp.png)
![image](https://hackmd.io/_uploads/H1pdbai3p.png)

* kernel paging request
![image](https://hackmd.io/_uploads/BJSoAfphp.png)
![image](https://hackmd.io/_uploads/r1ahCza36.png)

* bad page map in process systemd-udevd
![image](https://hackmd.io/_uploads/Hyd1iTi2p.png)
* bad rss
![image](https://hackmd.io/_uploads/BJPOpCih6.png)

* mutex unlock fail 然後繼續執行
![image](https://hackmd.io/_uploads/SyQgIEyap.png)
![image](https://hackmd.io/_uploads/SJfZI4ypa.png)

* stuck in read lock release
![image](https://hackmd.io/_uploads/r1CZjCgTa.png)

* 卡在取時間的地方(parser_pause) +1
    * parser_pause中有問題
* 卡在thread真正執行時(in procflow_allstarted) +2
    * 主要是 threadflow_allstarted(procflow->pf_pid, procflow->pf_threads);
    *  (in procflow_allstarted)介紹
        *  此function會先設置procflow的pf_running flag
            *  如果30秒內沒有設置好，將會log並跳過然後直接去設置下一個(但我們都只有一個process，所以只會執行一次)
        *  設置好之後會call threadflow_allstarted啟動所有Thread 執行   --**\*卡在這\***

* 以下可能是童情況
    * 有做完，但卡在15行filebench_log +2
        * 在15和16行中間+printf，但並沒有印出東西
        * 在15和16行中間+printf，有印出東西
        * 等很久可能就變下面的情況了(這裡只等了4~6小時)
    * 有做完，但最後有bug +1
        * ![image](https://hackmd.io/_uploads/Hyjea9NT6.png)

![image](https://hackmd.io/_uploads/B1ZXUZPpp.png)


* 有做完，但卡在 parser_proc_shutdown
    * ![image](https://hackmd.io/_uploads/ryngNNUCT.png)

* 有做完(沒錯誤也沒卡住)+3
    * ![image](https://hackmd.io/_uploads/rkYY4lcp6.png)


* 有做完(有一些錯誤但沒卡住)
    * ![image](https://hackmd.io/_uploads/BJ9v1iipp.png)


----
### debug
先找錯在哪
![image](https://hackmd.io/_uploads/BJwnwucnp.png)
timing mode gdb試試
* 

## 使用&command
### test 1
```
#! /bin/bash
echo "1" 
time sleep 1s &
time sleep 0.5s
echo "2" &

```
#### atomic
![image](https://hackmd.io/_uploads/H19fRI2Y6.png)

#### timing
![image](https://hackmd.io/_uploads/HJ_tjPpFT.png)
卡住
![image](https://hackmd.io/_uploads/rJuIRzxq6.png)
第二次也是


---
real：表示的是實際經過時間，說白了，其實就是從程序運行開始到結束所經歷的時間；

user：表示程序運行期間，cpu 在用戶態所花費的時間；

sys：表示程序運行期間，cpu 在內核態所花費的時間；


### test 2
```clike=
#include <pthread.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <stdio.h>
 void *thread_function(void *arg) {
  int i;
  for ( i=0; i<8; i++) {
    printf("Thread working...! %d \n",i);
    sleep(1);
  }
  return NULL;
}
int main(void) {
  pthread_t mythread;
  
  if ( pthread_create( &mythread, NULL, thread_function, NULL) ) {
    printf("error creating thread.");
    abort();
  }
  if ( pthread_join ( mythread, NULL ) ) {
    printf("error join thread.");
    abort();
  }
  printf("thread done! \n");
  exit(0);
}
```
#### atomic
![image](https://hackmd.io/_uploads/Sy2aI1v9T.png)
#### timing
![image](https://hackmd.io/_uploads/Hkx8PYn_9p.png)

## 跑multithread程式2 (pthread with mutex)
### reason
* 因為filebench是開檔卡住並出現mutex lock eroor，所以針對mutex進行測試

### code
```clike
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

// 計數器
int counter = 0;

// 加入 Mutex
pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;

// 子執行緒函數
void* child() {
  for(int i = 0;i < 3;++i) {
    pthread_mutex_lock( &mutex1 ); // 上鎖
    int tmp = counter;
    sleep(1);
    counter = tmp + 1;
    pthread_mutex_unlock( &mutex1 ); // 解鎖
    printf("Counter = %d\n", counter);
  }
  pthread_exit(NULL);
}

// 主程式
int main() {
  pthread_t t1, t2;
  pthread_create(&t1, NULL, child, NULL);
  pthread_create(&t2, NULL, child, NULL);
  pthread_join(t1, NULL);
  pthread_join(t2, NULL);
  return 0;
}
```
### compile
gcc ctest.c -lpthread -o ctest

### atomic 
work
### timing
![image](https://hackmd.io/_uploads/rJM0TgcnT.png)
work

只出現過一次?
![image](https://hackmd.io/_uploads/BJym0e52p.png)
## 跑multithread程式3 (pthread with mutex and fopen and file write)
### 介紹
* 相較test2，多跑了file相關的operation。
* pthread t2會晚執行，但是由於t1會先sleep 5ms，所以t2會先寫檔
### code
```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
struct fun_para
{
       int a;
};
int counter = 0;
FILE *fptr;
pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;

// 子執行緒函數
void* child(void *arg) {
    struct fun_para *para;
    para = (struct fun_para*) arg;
    if(para->a ==1){
      usleep(5000);
    }
    pthread_mutex_lock( &mutex1 ); // 上鎖
    counter++;
    printf("para->a = %d\n", para->a);
    printf("counter = %d\n", counter);
    fprintf(fptr,"%d",counter);
    pthread_mutex_unlock( &mutex1 ); // 解鎖
  pthread_exit(NULL);
}

// 主程式
int main() {
  struct fun_para para;
  para.a=1;
  struct fun_para para1;
  para1.a=2;
  pthread_t t1, t2;
  fptr = fopen("TheTXT.txt","w");
  pthread_create(&t1, NULL, child,&para); //will sleep 1 sec
  pthread_create(&t2, NULL, child,&para1);
  pthread_join(t1, NULL);
  pthread_join(t2, NULL);
  fclose(fptr);
  return 0;
}  
```
### atomic 
![image](https://hackmd.io/_uploads/ByWqft6aa.png)
* txt內會有"12"
### timing
![image](https://hackmd.io/_uploads/BJChGF6Ta.png)
* work

## 跑multithread程式4(兩thread同時寫file)
### code
```clike=
#define FILE_SIZE_LIMIT 1048576 // 1MB
#define BUFFER_SIZE 4096         // 4KB

// 共享的文件指针和互斥锁
FILE *file;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *writeToFile(void *threadId) {
    long tid = (long)threadId;
    char buffer[BUFFER_SIZE];
    memset(buffer, 'A' + tid, BUFFER_SIZE); // 生成用于写入的数据
    while (1) {
        pthread_mutex_lock(&mutex);
        if (ftell(file) >= FILE_SIZE_LIMIT) {
            pthread_mutex_unlock(&mutex);
            break;
        }
        long position = ftell(file);
        fwrite(buffer, 1, BUFFER_SIZE, file);
        fflush(file);
        pthread_mutex_unlock(&mutex);
        printf("Thread %ld wrote %ldKB~%ldKB\n", tid, position / 1024, (position + BUFFER_SIZE) / 1024);
    }
    pthread_exit(NULL);
}
int main() {
    file = fopen("output.txt", "w+");
    if (file == NULL) {
        perror("Error opening file");
        return -1;
    }
    pthread_t threads[2];
    int rc;
    long t;
    // 创建两个线程
    for (t = 0; t < 2; t++) {
        rc = pthread_create(&threads[t], NULL, writeToFile, (void *)t);
        if (rc) {
            fprintf(stderr, "Error creating thread: %d\n", rc);
            return -1;
        }
    }
    // 等待线程结束
    for (t = 0; t < 2; t++) {
        pthread_join(threads[t], NULL);
    }
    fclose(file);
    pthread_mutex_destroy(&mutex);
    return 0;
}
```
* 輸出會如以下看哪個thread先搶到lock，就會append寫4KB，並印出他寫哪裡，之後free lock，一直循環直到寫了512KB
* ![image](https://hackmd.io/_uploads/B1NSY0CA6.png)
### atomic
可以
![image](https://hackmd.io/_uploads/rJY2tZky0.png)
### timing
做完後會有error
* error and stuck
    *    ![image](https://hackmd.io/_uploads/BJWLgMkk0.png)
* erroe 2
    * ![image](https://hackmd.io/_uploads/ByhN4fJkR.png)
    * fs/file.c:605為__fd_install BUG_ON的地方
    * :::spoiler __fd_install註釋
        **Install a file pointer in the fd array.**

        The VFS is full of places where we drop the files lock between setting the open_fds bitmap and installing the file in the file array. At any such point, we are vulnerable to a dup2() race installing a file in the array before us. We need to detect this and fput() the struct file we are about to overwrite in this case.

        It should never happen - if we allow dup2() do it, really bad things will follow.

        NOTE: **fd_install() variant is really, really low-level**; **don't use it unless you are forced to by truly lousy API shoved down your throat**. 'files' MUST be either current->files or obtained by get_files_struct(current) done by whoever had given it to you, or really bad things will happen. Normally you want to use fd_install() instead.
        :::
    * ![image](https://hackmd.io/_uploads/rkD8VMy1A.png)

* error3
    * ![image](https://hackmd.io/_uploads/BJClSVkJ0.png)
    * ![image](https://hackmd.io/_uploads/rJU7rEJJR.png)
    * 目的是將一個 VMA 插入到進程的內存區域（memory map）的列表和紅黑樹中，並更新相關的計數器

* stuck +1
    * ![image](https://hackmd.io/_uploads/By_xpHkyR.png)

* error4+1
    * ![image](https://hackmd.io/_uploads/BkZATDk10.png)

* work 


# filebench trace
* 由於filebench program會斷在許多不同地方
* gdb好像連不到gem5裡跑的filebench
    * 先用printf看看
## 最上層parser_run code 以及介紹
* 介紹(根據其comment)
    * Do a file bench run. Calls routines to create file sets, files, and processes. It resets the statistics counters, then sleeps for the runtime passed as an argument to it on the command line in 1 second increments.
    * When it is finished sleeping, it collects a snapshot of the statistics and ends the run.
``` c=
static void
parser_run(cmd_t *cmd)
{
	int runtime;
	int timeslept;

	runtime = cmd->cmd_qty;
	parser_fileset_create(cmd);
	parser_proc_create(cmd);
	/* check for startup errors */
	if (filebench_shm->shm_f_abort)
		return;
	filebench_log(LOG_INFO, "Running...");
	stats_clear();
	timeslept = parser_pause(runtime);
	filebench_log(LOG_INFO, "Run took %d seconds...", timeslept);
	parser_statssnap(cmd);
	parser_proc_shutdown(cmd);
	parser_filebench_shutdown((cmd_t *)0);
}
```


* filebench 會執行此function裡的程式
* parser_statssnap
    * Kills off background statistics collection processes, then takes a snapshot  of the filebench run's collected statistics using stats_snap() from stats.c

## pohao with multithread
### ctest
使用[code 4](#跑multithread程式4兩thread同時寫file)
#### single cpu
* atomic ok
* timming ok
    * ![image](https://hackmd.io/_uploads/r1mhvW1J0.png)

#### multi cpu
* atomic pk

### filebench
#### single cpu
* 首先先對single cpu跑multithread 試試
    * 可以
    * ![image](https://hackmd.io/_uploads/HyRluap06.png)
    * ![image](https://hackmd.io/_uploads/HyaGDfJJC.png)

#### multi cpu

* atomic ok

* ![image](https://hackmd.io/_uploads/BJk0JUxkA.png)
    *    有做完且沒error(旁邊的message是cpu stall太久的訊息) +1
* 看起來有做完
![image](https://hackmd.io/_uploads/HkXn6zCR6.png)
* error
![image](https://hackmd.io/_uploads/HkNX8a0Rp.png)
* 開機後有機率出現
![image](https://hackmd.io/_uploads/rJsMORAAa.png)
![image](https://hackmd.io/_uploads/HyK4_00A6.png)

