# 线程局部存储(Thread Local Storage,TLS)
## 来源
&emsp;&emsp;为了在不同的函数中使用当前线程上下文的某个对象, 我们常常需要用到静态变量或者全局变量来读写对象，在单线程编程中这样操作还是很简单的，但是到了多线程后这个对象通常就要用数组的形式的来储存了,这样才能保证每个线程的对象数据都不互相影响, 为了在多线程中安全的读写同一个对象就有很大概率需要重复用到读写锁, 而锁是一个十分耗时的操作。  
&emsp;&emsp;举个Windows下的C语言例子来说, 如果我们想统计每个线程的内存使用情况, 普通代码可能这么写:
```c
// File: main.c
#include <stdio.h>
#include <windows.h>
#include <time.h>
#pragma comment(lib, "winmm.lib")

CRITICAL_SECTION g_cs;
void create_lock() { InitializeCriticalSection(&g_cs); }
void delete_lock() { DeleteCriticalSection(&g_cs); }
void lock() { EnterCriticalSection(&g_cs); }
void unlock() { LeaveCriticalSection(&g_cs); }

#define max_thread_size 1000
unsigned g_thread_id[max_thread_size];
unsigned g_thread_alloc_info[max_thread_size];
unsigned g_thread_free_info[max_thread_size];

// 寻找线程信息的索引
unsigned int get_thread_info_index(unsigned int thread_id) {
  int index = -1;
  int cur_tid = 0;
  lock();
  for (int i = 0; i < max_thread_size; i++) {
    cur_tid = g_thread_id[i];
    if (cur_tid == 0) {
      g_thread_id[i] = thread_id;
      index = i;
      break;
    }
    else if (cur_tid == thread_id) {
      index = i;
      break;
    }
  }
  unlock();
  return index;
}

// 读写当前线程上下文信息
void thread_proc_inner(BOOL leak_it) {
  unsigned int index = get_thread_info_index(GetCurrentThreadId());
  void* ptr = malloc(1);
  g_thread_alloc_info[index]++;
  if (!leak_it) {
    free(ptr);
    g_thread_free_info[index]++;
  }
}

// 线程入口
unsigned long __stdcall thread_proc(void* param) {
  // 在线程里运行10000次读写线程上下文的操作
  for (int i = 0; i < 10000; i++) {
    thread_proc_inner((BOOL)param);
  }
  return 0;
}

int main(int argc, char*  argv[]) {
  memset(&g_thread_id, 0, sizeof(g_thread_id));
  memset(&g_thread_alloc_info, 0, sizeof(g_thread_alloc_info));
  memset(&g_thread_free_info, 0, sizeof(g_thread_free_info));
  create_lock();

  HANDLE thread_handle[50];

  DWORD start_time = timeGetTime();

  // 创建50个线程来运行, 前30个线程故意泄露内存
  int i = 0;
  for (i = 0; i < 50; i++) {
    thread_handle[i] = CreateThread(
        NULL, NULL, thread_proc, (LPVOID)(i < 30 ? TRUE : FALSE), 0, NULL);
  }
  WaitForMultipleObjects(50, thread_handle, TRUE, INFINITE);
  DWORD used_time = timeGetTime() - start_time;

  for(i = 0; i < 50; i++) {
    CloseHandle(thread_handle[i]);
  }
  delete_lock();

  printf("used time:%d\n", used_time);
  system("pause");
  return 0;
}

//-----------------------------------------------
// used time: 187
```
&emsp;&emsp;可以看到代码运行的时间大概是会在187毫秒左右(我电脑I7-8750H, 6核12线程)。 在这里, get_thread_info_index是一个非常耗时并且堵塞性的函数, 重复性的调用它将会严重的影响了自身线程以及其他线程的代码运行效率（如果其他线程也有调用这个函数的话）。
&emsp;&emsp;我们也可以发现这类代码有个特征, 就是每个线程都需要一个的索引值在内存空间去访问它的线程私有数据, 并且私有数据的类型每个线程都要用到, 每个线程为了这个索引值都要花费不少的时间。很明显, 由于索引是固定的, 我们只要把获取索引的这个过程加速一下就可以剩下不少时间。
&emsp;&emsp;为此操作系统在进程中设计了一个TLS的理念, 它的原理也比较简单，大概就是以下几点:
- 在线程创建的时候为每个线程专门开辟一段数组空间来储存自定义数据, 如下图(每个索引都固定到指定的类型):  
&emsp;&emsp;&emsp;index1 index2  index3 index4....indexN  
  线程1: [类型1] [类型2] [类型3] [类型4]....[类型N]  
  线程2: [类型1] [类型2] [类型3] [类型4]....[类型N]
- 索引由进程统一分配
- 