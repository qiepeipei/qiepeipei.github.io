## ThreadSwitch.h
```c++
  #pragma once
  //最大支持的线程数
  #define MAXGMTHREAD 100
  //线程信息的结构(仿EHREAD)
  typedef struct 
  {
    char* name;                        //线程名 
    int Flags;                        //线程状态
    int SleepMillsecondDot;            //休眠时间

    void* initialStack;                //线程堆栈起始位置
    void* StackLimit;                //线程堆栈界限
    void* KernelStack;                //线程堆栈当前位置，也就是ESP

    void* lpParameter;                //线程函数的参数
    void(*func)(void* lpParameter);    //线程函数
  }GMThread_t;
  
  void GMSleep(int MilliSeconds);
  int RegisterGMThread(char* name, void(*func)(void*lpParameter), void* lpParameter);
  void Scheduling(void); 
  
```
## ThreadSwitch.cpp
```c++
  #include "stdafx.h"
  #include "ThreadSwitch.h"
  //定义线程堆栈的大小
  #define GMTHREADSTACKSIZE 0x80000
  //当前线程的索引
  int CurrentThreadIndex = 0;
  //线程的列表 第一个元素是给main函数使用的
  GMThread_t GMThreadList[MAXGMTHREAD] = {NULL, 0};
  //线程状态的标志
  enum FLAGS
  {			
    //线程创建状态
    GMTHREAD_CREATE = 0x1,
    //线程准备状态
    GMTHREAD_READY = 0x2,
    //线程延迟状态
    GMTHREAD_SLEEP = 0x4,
    //线程退出状态
    GMTHREAD_EXIT = 0x8,
  };
  //启动线程的函数
  void GMThreadStartup(GMThread_t* GMThreadp)
  {					
    GMThreadp->func(GMThreadp->lpParameter);
    GMThreadp->Flags = GMTHREAD_EXIT;
    Scheduling();
    return;
  }
  //空闲线程的函数
  void IdleGMThread(void* lpParameter)
  {
    printf("IdleGMThread---------------\n");
    Scheduling();
    return;
  }
  //向栈中压入一个uint值
  void PushStack(unsigned int** Stackpp, unsigned int v) 
  {
    //设 &StackDWordParam = 0x0012feb0
    //设 StackDWordParam = 0x004afff0 
    //*Stackpp  = 0x0012feb0的值 = 0x004afff0 - sizeOf(int)
    //**Stackpp = 0x004afff0的值 = v
    *Stackpp -= 1;
    **Stackpp = v;
    return;
  }
  //初始化线程的信息
  void initGMThread(GMThread_t* GMThreadp, char* name, void (*func)(void* lpParameter), void* lpParameter)
  {
    unsigned char* StackPages;
    unsigned int* StackDWordParam;
    //初始化赋值
    GMThreadp->Flags = GMTHREAD_CREATE;
    GMThreadp->name = name;
    GMThreadp->func = func;
    GMThreadp->lpParameter = lpParameter;
    //申请堆栈空间 指向栈底
    StackPages = (unsigned char*)VirtualAlloc(NULL, GMTHREADSTACKSIZE, MEM_COMMIT, PAGE_READWRITE);
    //清零堆栈空间
    ZeroMemory(StackPages, GMTHREADSTACKSIZE);
    //初始化堆栈地址 指向栈顶 相当于ebp
    GMThreadp->initialStack = StackPages + GMTHREADSTACKSIZE;
    //堆栈限制边界
    GMThreadp->StackLimit = StackPages
    //堆栈地址 相当于是esp
    StackDWordParam = (unsigned int*)GMThreadp->initialStack;
    //入栈
    //通过这个指针来找到：线程函数，函数参数
    PushStack(&StackDWordParam, (unsigned int)GMThreadp); 
    //平衡堆栈 
    PushStack(&StackDWordParam, (unsigned int)0);
    //线程入口函数 这个函数负责调用线程函数
    PushStack(&StackDWordParam, (unsigned int)GMThreadStartup);
    // push ebp
    PushStack(&StackDWordParam, (unsigned int)5);
    // push edi
    PushStack(&StackDWordParam, (unsigned int)7);
    // push esi
    PushStack(&StackDWordParam, (unsigned int)6);
    // push ebx
    PushStack(&StackDWordParam, (unsigned int)3);
    // push ecx
    PushStack(&StackDWordParam, (unsigned int)2);
    // push edx
    PushStack(&StackDWordParam, (unsigned int)1);
    // push eax
    PushStack(&StackDWordParam, (unsigned int)0);
    //当前线程的栈顶
    GMThreadp->KernelStack = StackDWordParam;
    //设置线程状态
    GMThreadp->Flags = GMTHREAD_READY;
    return;
  }
  //将一个函数注册为单独线程执行
  int RegisterGMThread(char* name, void(*func)(void*lpParameter), void* lpParameter)
  {
    int i;
    for (i = 1; GMThreadList[i].name; i++) {
      if (0 == _stricmp(GMThreadList[i].name, name)) {
          break;
      }
    }
    initGMThread(&GMThreadList[i], name, func, lpParameter);
    return (i & 0x55AA0000);
  }
  //切换线程
  __declspec(naked) void SwitchContext(GMThread_t* SrcGMThreadp, GMThread_t* DstGMThreadp)
  {
    __asm {
          //提升堆栈
          push ebp
          mov ebp, esp
          //把当前线程的寄存器压入堆栈
          push edi
          push esi
          push ebx
          push ecx
          push edx
          push eax
          //当前线程结构指针     		
          mov esi, SrcGMThreadp
          //要切换的线程结构体指针
          mov edi, DstGMThreadp
          //把当前线程的esp存储到KernelStack
          mov [esi+GMThread_t.KernelStack], esp
          //经典线程切换，另外一个线程复活。
          //上面代码是保存当前线程堆栈
          ------------------------------
          //下面是另一个线程
          //修改esp的值为目标线程的 KernelStack
          mov esp, [edi+GMThread_t.KernelStack]
          //下面pop是弹出目标线程的堆栈值
          pop eax
          pop edx
          pop ecx
          pop ebx
          pop esi
          pop edi
          pop ebp
          //设
          //GMThreadp	0x00428c70    =  0040132A   mov         eax,dword ptr [ebp+8]
          //	name	0x00423070 "Thread1"
          //	Flags	2
          //	SleepMillsecondDot	0
          //	initialStack	0x004b0000
          //	StackLimit	0x00000000
          //	KernelStack	0x004affd8
              //	lpParameter	0x00000000  =  0040132D   mov         ecx,dword ptr [eax+18h]
            //	func	0x0040103c Thread1(void *)  = 00401334   call        dword ptr [edx+1Ch]
          //00401328   mov         esi,esp
          //0040132A   mov         eax,dword ptr [ebp+8]
          //0040132D   mov         ecx,dword ptr [eax+18h]
          //00401330   push        ecx
          //00401331   mov         edx,dword ptr [ebp+8]
          //00401334   call        dword ptr [edx+1Ch]

          ret 
      }
  }
  //这个函数会让出cpu，从队列里重新选择一个线程执行
  void Scheduling(void)
  {
    int i;
    int TickCount;
    GMThread_t* SrcGMThreadp;
    GMThread_t* DstGMThreadp;
    //操作系统到当前所经的计时周期数
    TickCount = GetTickCount();
    SrcGMThreadp = &GMThreadList[CurrentThreadIndex];
    DstGMThreadp = &GMThreadList[0];
    //找到第一个处于READY状态的线程
    for (i = 1; GMThreadList[i].name; i++) {
      //如果该线程对象处于休眠
      if (GMThreadList[i].Flags & GMTHREAD_SLEEP) {
      //如果运行时间大于休眠时间
        if (TickCount > GMThreadList[i].SleepMillsecondDot) {
            GMThreadList[i].Flags = GMTHREAD_READY;
        }
      }
      if (GMThreadList[i].Flags & GMTHREAD_READY) {
          DstGMThreadp = &GMThreadList[i];
          break;
      }
    }
    //保存当前线程索引
    CurrentThreadIndex = DstGMThreadp - GMThreadList;
    //线程切换
    SwitchContext(SrcGMThreadp, DstGMThreadp);
    return;
  }
  void GMSleep(int MilliSeconds)
  {
    GMThread_t* GMThreadp;
    //获取当前线程对象
    GMThreadp = &GMThreadList[CurrentThreadIndex];
    //如果线程状态不等于创建
    if (GMThreadp->Flags != 0) {
      //设置线程状态为 休眠
      GMThreadp->Flags = GMTHREAD_SLEEP;
      //休眠时间
      GMThreadp->SleepMillsecondDot = GetTickCount() + MilliSeconds;
    }
    Scheduling();
    return;
  }
```
## main.cpp
```c++
#include "stdafx.h"
#include "ThreadSwitch.h"
extern int CurrentThreadIndex;
extern GMThread_t GMThreadList[MAXGMTHREAD];

void Thread1(void*) {
	while(1){
		printf("Thread1");
		GMSleep(500);
	}
}

void Thread2(void*) {
	while (1) {
		printf("Thread2");
		GMSleep(200);
	}
}

void Thread3(void*) {
	while (1) {
		printf("Thread3");
		GMSleep(10);
	}
}

void Thread4(void*) {
	while (1) {
		printf("Thread4");
		GMSleep(1000);
	}
}

int main()
{
  //参数1 = 线程名
  //参数2 = 线程要执行的函数
  //参数3 = 线程参数
	RegisterGMThread("Thread1", Thread1, NULL);
	RegisterGMThread("Thread2", Thread2, NULL);
	RegisterGMThread("Thread3", Thread3, NULL);
	RegisterGMThread("Thread4", Thread4, NULL);

  //模拟windows线程切换
	while(TRUE) {
		Sleep(20);
		Scheduling();
	}
  
  return 0;
}
```

# 模拟线程总结
1. 线程不是被动切换的，而是主动让出CPU。
2. 线程切换并没有使用TSS来保持寄存器，而是使用堆栈。
3. 线程切换的过程就是切换堆栈的过程。




