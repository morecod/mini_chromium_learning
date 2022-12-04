# 线程局部存储(Thread Local Storage,TLS)
## 概述
&emsp;&emsp;为了在不同的函数中使用当前线程上下文的某个对象, 我们常常需要用到静态变量或者全局变量来读写对象，在单线程编程中这样操作还是很简单的，但是到了多线程后这个对象通常就要用数组的形式的来储存了,这样才能保证每个线程的对象数据都不互相影响, 为了在多线程中安全的读写同一个对象就有很大概率需要用到读写锁, 而锁是一个十分耗时的操作。  
&emsp;&emsp;举个Windows下的C语言例子来说, 如果我们想统计每个线程的内存使用情况:
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

  // 创建200个线程来运行, 前100个线程故意泄露内存
  for (int i = 0; i < 50; i++) {
    thread_handle[i] = CreateThread(
        NULL, NULL, thread_proc, (LPVOID)(i < 30 ? TRUE : FALSE), 0, NULL);
  }
  WaitForMultipleObjects(50, thread_handle, TRUE, INFINITE);
  DWORD used_time = timeGetTime() - start_time;

  delete_lock();

  printf("used time:%d\n", used_time);
  system("pause");
  return 0;
}

//-----------------------------------------------
// used time: 187
```
可以看到代码运行的时间大概是会在187毫秒左右(我电脑I7-8750H, 6核12线程)。get_thread_info_index是一个非常耗时的函数,而且时间基本消耗在重复性的调用get_thread_info_index(GetCurrentThreadId())这里;, 如果我们用TLS把这个索引给缓存一下, 就只需要调用一次get_thread_info_index就行了。