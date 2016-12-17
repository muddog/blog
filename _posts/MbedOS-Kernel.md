---
published: true
data: {}
title: mbedOS内核分析
Tags:
  - mbedOS
  - kernel
---
# 内核分析 #

mbedOS很有趣，从编写一个应用程序开始你会发现它和其他的RTOS有些不同。为啥它的main()函数里不需要做任何操作系统初始化，甚至板级初始化，亦或是建立其他进程？mbdOS的内核从CMSIS-RTOS衍生而来，为客户提供了最大程度上的Out of Box Experience的使用体验。它把操作系统、硬件初始化的一切都在main()之前做完了，而且main()其实就是一个单独的进程！我们来看看mbedOS到底是怎么做的，这里还是以K64F的硬件平台及GCC编译器为例。
我们都知道芯片上电都会从Reset_Handler向量开始，源代码在startup_MK64F12.S里。Reset_Handler首先会关闭Watchdog，然后将data数据段从Flash里拷贝到RAM中，之后调用\_start，讲控制权交给C运行时库CRT。crt1.s里会提供一个软件初始化的hook供用户实现

``` C
extern "C" void software_init_hook(void)
{                                                          
    mbed_sdk_init();                                                               
    software_init_hook_rtos();                                                     
}
```

mbed_sdk_init()会调用Kinetis SDK里的BOARD_BootXXX()来根据板子的时钟设计来配置时钟。software_init_hook_rtos()则是真正操作系统初始化的地方：

``` C
osThreadDef_t os_thread_def_main = {(os_pthread)pre_main, osPriorityNormal, 1U, sizeof(thread_stack_main), thread_stack_main};
...

void pre_main(void) {
    singleton_mutex_id = osMutexCreate(osMutex(singleton_mutex));
    malloc_mutex_id = osMutexCreate(osMutex(malloc_mutex));
    env_mutex_id = osMutexCreate(osMutex(env_mutex));
    __libc_init_array();
    main(0, NULL);
}

__attribute__((naked)) void software_init_hook_rtos (void) {
  __asm (
    "bl   osKernelInitialize\n"
#ifdef __MBED_CMSIS_RTOS_CM
    "bl   set_stack_heap\n"
#endif
    "ldr  r0,=os_thread_def_main\n"                                                
    "movs r1,#0\n"                                                                 
    "bl   osThreadCreate\n"                                                        
    "bl   osKernelStart\n"
    /* osKernelStart should not return */
    "B       .\n"
  );
}
```

software_init_hook_rtos首先调用osKernelInitialize()，它根据当前的CPU权限通过svc call或者直接调用svcKernelInitialize()：

``` C
osStatus svcKernelInitialize (void) {
  if (!os_initialized) {
    rt_sys_init();                              // RTX System Initialization
  } 

  os_tsk.run->prio = 255U;                      // Highest priority

  if (os_initialized == 0U) {
    // Create OS Timers resources (Message Queue & Thread)
    osMessageQId_osTimerMessageQ = svcMessageCreate (&os_messageQ_def_osTimerMessageQ, NULL);
    osThreadId_osTimerThread = svcThreadCreate(&os_thread_def_osTimerThread, NULL, NULL);
    // Initialize thread mutex
    osMutexId_osThreadMutex = osMutexCreate(osMutex(osThreadMutex));
  }

  sysThreadError(osOK);

  os_initialized = 1U;
  os_running = 0U;

  return osOK;
}

```
